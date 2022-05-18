---
layout: post
title: "nginx-rtmp-module的子模块开发"
description:
categories: LiveStream
tags: [live]
photos:
description:
---

nginx-rtmp-module是nginx中的一个模块，在nginx上实现rtmp推拉流的功能。nginx-rtmp-module模块很大，可以基于nginx-rtmp-module开发子模块，如果说nginx提供了http模块的开发环境，那nginx-rtmp-module就是提供了rtmp模块的开发环境。

## 创建nginx-rtmp-module子模块
### 模块注册
模块注册是为了让nginx能够知道模块名称以及源码路径，在编译时加载模块。注册很简单，只要再config文件中增加几行代码就能实现注册，config文件中包含了需要编译的各个模块名称以及源码c文件名和头文件名，新加的模块需要在RTMP_CORE_MODULES变量中增加模块名，模块的调用顺序是从后到前的，所以如果要让一个模块在另一个模块之前调用那么就要把该模块写在后面。RTMP_CORE_SRCS变量包含模块的源码文件名，如果模块有头文件，需要加在RTMP_DEPS变量中。
### 模块结构体
模块注册完之后要在模块源码文件中实现模块结构体和模块上下文，模块结构体中包含模块的配置项和模块初始化、退出等阶段的处理函数。命名是config文件中注册模块名

{% highlight c %}
ngx_module_t  ngx_rtmp_relay_module = {
    // 版本号
    NGX_MODULE_V1,
    // 模块上下文，包含个阶段的配置项处理函数
    &ngx_rtmp_relay_module_ctx,             /* module context */
    // 该模块的配置项
    ngx_rtmp_relay_commands,                /* module directives */
    NGX_RTMP_MODULE,                        /* module type */
    // nginx程序启动时会产生一个master进程与多个worker进程，
    // master进程的作用是管理worker进程，比如worker进程挂掉可以由master进程重新拉起，
    // 以下函数指针可以在master、worker进程初始化和退出的各阶段进行处理
    NULL,                                   /* init master */
    NULL,                                   /* init module */
    ngx_rtmp_relay_init_process,            /* init process */
    NULL,                                   /* init thread */
    NULL,                                   /* exit thread */
    NULL,                                   /* exit process */
    NULL,                                   /* exit master */
    NGX_MODULE_V1_PADDING
};
{% endhighlight %}

### 模块上下文
模块上下文包含了模块配置解析各阶段函数，处理函数入口一般是在postconfiguration阶段加载

{% highlight c %}
static ngx_rtmp_module_t  ngx_rtmp_relay_module_ctx = {
    ngx_rtmp_relay_preconfiguration,        /* preconfiguration */
    ngx_rtmp_relay_postconfiguration,       /* postconfiguration */
    NULL,                                   /* create main configuration */
    NULL,                                   /* init main configuration */
    NULL,                                   /* create server configuration */
    NULL,                                   /* merge server configuration */
    ngx_rtmp_relay_create_app_conf,         /* create app configuration */
    ngx_rtmp_relay_merge_app_conf           /* merge app configuration */
};
{% endhighlight %}

## nginx-rtmp-module的处理阶段
和http类似，RTMP请求也被分为几个处理阶段，连接阶段、rtmp各种消息处理甚至amf中的不同字段也能针对处理，在postconfiguration中加载对应阶段的处理函数，所谓‘加载’就是把自己的函数添加到一个类似链表的结构里，每个消息对应一个链表，当对应消息出现时会遍历链表调用函数。以下是relay模块的postconfiguration实现：

{% highlight c %}
static ngx_int_t
ngx_rtmp_relay_postconfiguration(ngx_conf_t *cf)
{
    ngx_rtmp_handler_pt                *h;
    ngx_rtmp_amf_handler_t             *ch;
    ngx_rtmp_core_main_conf_t          *cmcf;

    cmcf = ngx_rtmp_conf_get_module_main_conf(cf, ngx_rtmp_core_module);

    // rtmp握手结束调用
    h = ngx_array_push(&cmcf->events[NGX_RTMP_HANDSHAKE_DONE]);
    *h = ngx_rtmp_relay_handshake_done;

    // rtmp控制消息触发的函数调用
    // publish消息调用
    next_publish = ngx_rtmp_publish;
    ngx_rtmp_publish = ngx_rtmp_relay_publish;

    // play消息调用
    next_play = ngx_rtmp_play;
    ngx_rtmp_play = ngx_rtmp_relay_play;

    next_delete_stream = ngx_rtmp_delete_stream;
    ngx_rtmp_delete_stream = ngx_rtmp_relay_delete_stream;

    next_close_stream = ngx_rtmp_close_stream;
    ngx_rtmp_close_stream = ngx_rtmp_relay_close_stream;

    // amf消息触发的调用
    // _result消息调用
    ch = ngx_array_push(&cmcf->amf);
    ngx_str_set(&ch->name, "_result");
    ch->handler = ngx_rtmp_relay_on_result;

    ch = ngx_array_push(&cmcf->amf);
    ngx_str_set(&ch->name, "_error");
    ch->handler = ngx_rtmp_relay_on_error;

    ch = ngx_array_push(&cmcf->amf);
    ngx_str_set(&ch->name, "onStatus");
    ch->handler = ngx_rtmp_relay_on_status;

    return NGX_OK;
}
{% endhighlight %}

分析rtmp控制消息的处理，可以看到模块会将ngx_rtmp_xxx赋值给next_xxx并且将自己的函数赋值给ngx_rtmp_xxx函数，这个操作相当于将自己的函数放到链表的头部。需要注意的是需要在自己的函数里面再调用next_xxx才能继续遍历链表上的其他函数，如果直接返回NGX_OK，那相当于链表到这边就停止了。

{% highlight c %}
static ngx_int_t
ngx_rtmp_relay_play(ngx_rtmp_session_t *s, ngx_rtmp_play_t *v)
{
    // do something...
next:
    return next_play(s, v);
}
{% endhighlight %}

relay模块的这个函数，在处理完之后再调用next_play进入下一阶段的处理，看起来像条流水线。
## 不同的处理阶段
### 连接处理

| 阶段 | 说明 |
| --- | --- |
| NGX_RTMP_CONNECT | 建连成功阶段 |
| NGX_RTMP_DISCONNECT | 连接断开阶段 |
| NGX_RTMP_HANDSHAKE_DONE | 握手结束阶段 |

### rtmp命令消息处理

| 阶段 | 说明 |
| --- | --- |
| ngx_rtmp_connect | 处理rtmp connect消息 |
| ngx_rtmp_disconnect | 与NGX_RTMP_DISCONNECT一样 |
| ngx_rtmp_create_stream | 处理rtmp create stream消息 |
| ngx_rtmp_close_stream |  |
| ngx_rtmp_delete_stream |  |
| ngx_rtmp_publish |  |
| ngx_rtmp_play |  |
| ngx_rtmp_seek |  |
| ngx_rtmp_pause |  |

### rtmp用户消息处理

| 阶段 | 说明 |
| --- | --- |
| ngx_rtmp_stream_begin |  |
| ngx_rtmp_stream_eof |  |
| ngx_rtmp_stream_dry |  |
| ngx_rtmp_recorded |  |
| ngx_rtmp_set_buflen |  |