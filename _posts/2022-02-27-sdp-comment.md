---
layout: post
title: "SDP注释"
description:
categories: LiveStream
tags: [live, webrtc]
photos:
description: sdp样例注释
---

```
// 版本号0
v=0
// 会话信息
// o=<username> <sess-id> <sess-version> <nettype> <addrtype> <unicast-address>
o=- 1 0 IN IP4 0.0.0.0
// 会话名称
s=webrtc_core
// 会话起始时间和结束时间，0代表不限制
t=0 0
// 端口复用，msid为0和1的数据流复用一个端口
a=group:BUNDLE 0 1
// WMS是WebRTC Media Stram的缩写，这里给Media Stream定义了一个唯一的标识符。
// 一个Media Stream可以有多个track（video track、audio track），这些track就是通过这个唯一标识符关联起来的，具体见下面的媒体行(m=)以及它对应的附加属性(a=ssrc:)
a=msid-semantic: WMS 0_5664_harchar1
// 媒体描述，一个媒体的开头行，描述了媒体类型端口（没用到）、传输协议、RTP type数值等等
// m=<media> <port> <proto> <fmt> ...
m=audio 1 RTP/AVPF 123 125
// 连接信息，在WebRTC中使用candidate表示地址信息，所以目前该字段没有用到
// c=<nettype> <addrtype> <connection-address>
c=IN IP4 0.0.0.0
// RTCP的端口和地址信息，没用到
a=rtcp:1 IN IP4 0.0.0.0
// 数据传输的地址，candidate是ICE中的概念，一个candidate表示一个地址，地址类型有：
// host：从网卡获取的地址
// srflx：向stun服务器发送binding请求时stun服务器看到的地址
// prflx：向对端发起binding request时，对端看到的地址
// relay：中继服务器地址
a=candidate:foundation 1 udp 100 125.78.248.71 8000 typ srflx raddr 125.78.248.71 rport 8000 generation 0
a=candidate:foundation 1 udp 100 125.78.248.71 14000 typ srflx raddr 125.78.248.71 rport 14000 generation 0
// ice-ufrag和ice-pwd用于ICE协商认证
a=ice-ufrag:0_5664_harchar1_3820b77a35f2471f
a=ice-pwd:dd486181aa907c3e565d3600
// DTLS协商过程中的指纹信息
a=fingerprint:sha-256 8A:BD:A6:61:75:AF:31:4C:02:81:2A:FA:12:92:4C:48:7B:9F:23:DD:BF:3D:51:30:E7:59:5C:9B:17:3D:92:34
// setup 指的是 DTLS 的角色，也就是谁是 DTLS Client(active)，谁是 DTLS Server(passive)，如果自己两个都可以那就是 actpass。这里我们是 actpass，那么就要由对方在 Answer 中确定最终的 DTLS 角色。这里passive表示对端是server
a=setup:passive
// 媒体流的方向有四种，分别是 sendonly、recvonly、sendrecv、inactive，它们既可以出现在会话级别描述中也可以出现在媒体级别的描述中。
// sendonly 表示只发送数据，比如客户端推流到 SFU，那么会在自己的 Offer(or Answer) 中携带 senonly 属性
// revonly 表示只接收数据，比如客户端向 SFU 订阅流，那么会在自己的 Offer(or Answer) 中携带 recvonly 属性
// sendrecv 表示可以双向传输，比如客户端加入到视频会议中，既要发布自己的流又要订阅别人的流，那么就需要在自己的 Offer(or Answer) 中携带 sendrecv 属性
// inactive 表示禁止发送数据，比如在基于 RTP 的视频会议中，主持人暂时禁掉用户 A 的语音，那么用户 A 的关于音频的媒体级别描述应该携带inactive 属性，表示不能再发送音频数据。
a=sendonly
// 扩展头ID字段的定义
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/meta-data-01
a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/meta-data-02
a=extmap:9 http://www.webrtc.org/experiments/rtp-hdrext/meta-data-03
a=extmap:10 http://www.webrtc.org/experiments/rtp-hdrext/decoding-timestamp
// 媒体描述的ID，一般和group bundle结合使用，上面的group bundle将mid为0和1的流数据复用一个端口
a=mid:0
// rtp和rtcp复用一个端口
a=rtcp-mux
// rtp类型对应的音视频编码信息
a=rtpmap:123 MP4A-ADTS/44100/2
// 支持的反馈机制
a=rtcp-fb:123 nack
a=rtcp-fb:123 transport-cc
// 对编码属性做进一步的描述
a=fmtp:123 PS-enabled=0;SBR-enabled=0;config=40002420adca1fe0;cpresent=0;object=2;profile-level-id=1;stereo=1
// 类型为125的rtp为fec数据
a=rtpmap:125 flexfec-03/44100/2
// 将ssrc为26353088和ssrc为91659898的rtp流通过FEC-FR绑定到一起
a=ssrc-group:FEC-FR 26353088 91659898
a=ssrc:26353088 cname:0_5664_harchar1
a=ssrc:26353088 msid:0_5664_harchar1 5664_harchar1_audio
a=ssrc:26353088 mslabel:0_5664_harchar1
a=ssrc:26353088 label:5664_harchar1_audio
a=ssrc:91659898 cname:0_5664_harchar1
a=ssrc:91659898 msid:0_5664_harchar1 5664_harchar1_audio
a=ssrc:91659898 mslabel:0_5664_harchar1
a=ssrc:91659898 label:5664_harchar1_audio
m=video 1 RTP/AVPF 96 98
c=IN IP4 0.0.0.0
a=rtcp:1 IN IP4 0.0.0.0
a=candidate:foundation 1 udp 100 125.78.248.71 8000 typ srflx raddr 125.78.248.71 rport 8000 generation 0
a=candidate:foundation 1 udp 100 125.78.248.71 14000 typ srflx raddr 125.78.248.71 rport 14000 generation 0
a=ice-ufrag:0_5664_harchar1_3820b77a35f2471f
a=ice-pwd:dd486181aa907c3e565d3600
a=extmap:14 urn:ietf:params:rtp-hdrext:toffset
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:13 urn:3gpp:video-orientation
a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:12 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/meta-data-01
a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/meta-data-02
a=extmap:9 http://www.webrtc.org/experiments/rtp-hdrext/meta-data-03
a=extmap:10 http://www.webrtc.org/experiments/rtp-hdrext/decoding-timestamp
a=extmap:18 http://www.webrtc.org/experiments/rtp-hdrext/video-composition-time
a=extmap:19 http://www.webrtc.org/experiments/rtp-hdrext/video-frame-type
a=fingerprint:sha-256 8A:BD:A6:61:75:AF:31:4C:02:81:2A:FA:12:92:4C:48:7B:9F:23:DD:BF:3D:51:30:E7:59:5C:9B:17:3D:92:34
a=setup:passive
a=sendonly
a=mid:1
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 H264/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 goog-remb
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 rrtr
a=rtcp-fb:96 transport-cc
a=fmtp:96 bframe-enabled=1;level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=640c1f
a=rtpmap:98 H264/90000
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 goog-remb
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=rtcp-fb:98 rrtr
a=rtcp-fb:98 transport-cc
a=fmtp:98 bframe-enabled=1;level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=ssrc:9575872 cname:0_5664_harchar1
a=ssrc:9575872 msid:0_5664_harchar1 5664_harchar1_video
a=ssrc:9575872 mslabel:0_5664_harchar1
a=ssrc:9575872 label:5664_harchar1_video
```