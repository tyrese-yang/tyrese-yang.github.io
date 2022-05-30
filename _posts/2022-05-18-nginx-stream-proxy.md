---
layout: post
title: "nginx-stream-proxy模块源码分析——数据转发部分"
description:
categories: Nginx
tags: [nginx]
photos:
description:
---

nginx-stream-proxy模块可以在nginx实现tcp、udp、UNIX-domain sockets流量的代理功能。在分析模块之前先复习下epoll

## epoll
epoll是linux中IO多路复用的一种机制，nginx借助epoll和事件驱动模型实现了高并发，当epoll检测到IO事件发生时通知nginx处理，nginx读取或写入IO时会一次调用各个模块的相关阶段的处理函数进行处理。  
epoll有两种工作模式LT（level trigger）和ET（edge trigger），对应电路中水平触发和边缘触发两种模式。 
LT：当epoll_wait检测到描述符事件发生并将此事件通知应用程序处理，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次通知此事件。  
ET：epoll_wait触发后如果没有及时处理不会继续通知，所以用ET模式需要及时处理IO事件  
**nginx使用的是ET模式**

## ngx_stream_proxy_process
数据转发是在ngx_stream_proxy_process函数中实现，也是本文主要分析的函数
```
static void ngx_stream_proxy_process(ngx_stream_session_t *s,
    ngx_uint_t from_upstream, ngx_uint_t do_write)

有三个参数
s: 连接信息
from_upstream: 是否来自上游的数据
do_write: 是否为写事件
```
from_upstream表示是否为“上游”给的数据，这个上游比较有意思，nginx作为proxy服务器，维护了client到nginx和nginx到后端server两段连接，如果是client到nginx这段，当连接**可写**时from_upstream为1表示上游数据到来，数据从nginx写到client；如果是nginx到server这段，当连接**可读**时from_upstream为1，数据从server写到nginx。
nginx前后两段的连接使用的都是同一个session，client到nginx（下游）连接是`s->connection`，nginx到server（上游）连接是`s->upstream`。这个函数里有两个变量`src`和`dst`，这两个代表两个连接，src是读取数据的连接，dst是发送数据的目的连接，从src读取数据再写入dst就实现了proxy的功能。如果from_upstream参数置位，那么src就是s->upstream->peer.connection，dst就是c->connection；相反from_upstream为0时，src = c->connection，dst = s->upstream->peer.connection。使用stream proxy时可以把nginx看做数据的搬运工，从一个连接产生的数据搬运到另一个连接，前面提到nginx使用的是ET工作模式的epoll，所以需要及时处理数据，如果没法及时处理数据也需要设法重新触发读取或写入操作，这里nginx使用了两个buffer来存储从连接中读取的数据（因为传输是双向的，所以需要两个）。  

下面从nginx源码中抄出来的数据转发相关的函数，加了一些自己理解的注释：
{% highlight c %}
static void
ngx_stream_proxy_process(ngx_stream_session_t *s, ngx_uint_t from_upstream,
    ngx_uint_t do_write)
{
    off_t                        *received, limit;
    size_t                        size, limit_rate;
    ssize_t                       n;
    ngx_buf_t                    *b;
    ngx_int_t                     rc;
    ngx_uint_t                    flags;
    ngx_msec_t                    delay;
    ngx_chain_t                  *cl, **ll, **out, **busy;
    ngx_connection_t             *c, *pc, *src, *dst;
    ngx_log_handler_pt            handler;
    ngx_stream_upstream_t        *u;
    ngx_stream_proxy_srv_conf_t  *pscf;

    // 客户端到nginx和nginx到server共用一个server，
    // nginx到server的信息都保存在s->upstream中
    u = s->upstream;

    // 客户端到nginx的连接
    c = s->connection;
    // nginx到后端server的连接
    pc = u->connected ? u->peer.connection : NULL;

    if (c->type == SOCK_DGRAM && (ngx_terminate || ngx_exiting)) {
        // 如果是UDP，在进程退出时主动关闭连接
        /* socket is already closed on worker shutdown */

        handler = c->log->handler;
        c->log->handler = NULL;

        ngx_log_error(NGX_LOG_INFO, c->log, 0, "disconnected on shutdown");

        c->log->handler = handler;

        ngx_stream_proxy_finalize(s, NGX_STREAM_OK);
        return;
    }

    pscf = ngx_stream_get_module_srv_conf(s, ngx_stream_proxy_module);

    if (from_upstream) {
        src = pc;
        dst = c;
        b = &u->upstream_buf;
        limit_rate = pscf->download_rate;
        received = &u->received;
        out = &u->downstream_out;
        busy = &u->downstream_busy;

    } else {
        src = c;
        dst = pc;
        b = &u->downstream_buf;
        limit_rate = pscf->upload_rate;
        received = &s->received;
        out = &u->upstream_out;
        busy = &u->upstream_busy;
    }

    for ( ;; ) {
        // 处理写事件或者从另一端连接读到数据时do_write置为1
        if (do_write && dst) {

            if (*out || *busy || dst->buffered) {
                // 开始写数据，数据都存放在out中
                rc = ngx_stream_top_filter(s, *out, from_upstream);

                if (rc == NGX_ERROR) {
                    if (c->type == SOCK_DGRAM && !from_upstream) {
                        ngx_stream_proxy_next_upstream(s);
                        return;
                    }

                    ngx_stream_proxy_finalize(s, NGX_STREAM_OK);
                    return;
                }

                ngx_chain_update_chains(c->pool, &u->free, busy, out,
                                      (ngx_buf_tag_t) &ngx_stream_proxy_module);

                if (*busy == NULL) { // 相当于b->pos == b->last
                    // 如果buffer中的数据都写完了，那么重置buffer
                    // 读写数据真正使用的是upstream_buf或者downstream_buf
                    // out链表中的节点只是一个个指向buf内存不同位置的指针而已
                    b->pos = b->start;
                    b->last = b->start;
                }
            }
        }
        // size表示buf还能存多少数据
        size = b->end - b->last;

        // src->read->ready表示connection中还有数据可读
        // src->read->delayed为1时表示受到速率限制还不可读
        if (size && src->read->ready && !src->read->delayed
            && !src->read->error)
        {
            if (limit_rate) {
                // 这里可以控制读取的速率
                limit = (off_t) limit_rate * (ngx_time() - u->start_sec + 1)
                        - *received;

                if (limit <= 0) {
                    src->read->delayed = 1;
                    delay = (ngx_msec_t) (- limit * 1000 / limit_rate + 1);
                    ngx_add_timer(src->read, delay);
                    break;
                }

                if ((off_t) size > limit) {
                    size = (size_t) limit;
                }
            }

            n = src->recv(src, b->last, size);

            if (n == NGX_AGAIN) {
                break;
            }

            if (n == NGX_ERROR) {
                if (c->type == SOCK_DGRAM && u->received == 0) {
                    ngx_stream_proxy_next_upstream(s);
                    return;
                }

                src->read->eof = 1;
                n = 0;
            }

            if (n >= 0) {
                if (limit_rate) {
                    delay = (ngx_msec_t) (n * 1000 / limit_rate);

                    if (delay > 0) {
                        src->read->delayed = 1;
                        ngx_add_timer(src->read, delay);
                    }
                }

                if (from_upstream) {
                    if (u->state->first_byte_time == (ngx_msec_t) -1) {
                        u->state->first_byte_time = ngx_current_msec
                                                    - u->state->response_time;
                    }
                }

                if (c->type == SOCK_DGRAM && ++u->responses == pscf->responses)
                {
                    src->read->ready = 0;
                    src->read->eof = 1;
                }

                for (ll = out; *ll; ll = &(*ll)->next) { /* void */ }

                cl = ngx_chain_get_free_buf(c->pool, &u->free);
                if (cl == NULL) {
                    ngx_stream_proxy_finalize(s,
                                              NGX_STREAM_INTERNAL_SERVER_ERROR);
                    return;
                }

                *ll = cl;

                cl->buf->pos = b->last;
                cl->buf->last = b->last + n;
                cl->buf->tag = (ngx_buf_tag_t) &ngx_stream_proxy_module;

                cl->buf->temporary = (n ? 1 : 0);
                cl->buf->last_buf = src->read->eof;
                cl->buf->flush = 1;

                *received += n;
                b->last += n;
                do_write = 1;

                // 如果接收到数据就继续向另一端连接发送数据，实现proxy
                continue;
            }
        }

        break;
    }

    if (src->read->eof && dst && (dst->read->eof || !dst->buffered)) {
        handler = c->log->handler;
        c->log->handler = NULL;

        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                      "%s%s disconnected"
                      ", bytes from/to client:%O/%O"
                      ", bytes from/to upstream:%O/%O",
                      src->type == SOCK_DGRAM ? "udp " : "",
                      from_upstream ? "upstream" : "client",
                      s->received, c->sent, u->received, pc ? pc->sent : 0);

        c->log->handler = handler;

        ngx_stream_proxy_finalize(s, NGX_STREAM_OK);
        return;
    }

    flags = src->read->eof ? NGX_CLOSE_EVENT : 0;

    if (!src->shared && ngx_handle_read_event(src->read, flags) != NGX_OK) {
        ngx_stream_proxy_finalize(s, NGX_STREAM_INTERNAL_SERVER_ERROR);
        return;
    }

    if (dst) {
        if (!dst->shared && ngx_handle_write_event(dst->write, 0) != NGX_OK) {
            ngx_stream_proxy_finalize(s, NGX_STREAM_INTERNAL_SERVER_ERROR);
            return;
        }

        if (!c->read->delayed && !pc->read->delayed) {
            // 如果没有触发速率限制则设置连接超时时间
            ngx_add_timer(c->write, pscf->timeout);

        } else if (c->write->timer_set) {
            ngx_del_timer(c->write);
        }
    }
}
{% endhighlight %}