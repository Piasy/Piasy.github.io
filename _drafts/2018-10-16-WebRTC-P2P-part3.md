---
layout: post
title: WebRTC Native 源码导读（十五）：P2P 连接的使用
tags:
    - 实时多媒体
    - WebRTC
    - 网络
---

> 连接建立成功后，就可以收发应用层的数据了，数据的收发将会通过选出来的 Connection 对象完成，Connection 则是调用 Port，Port 则是调用 AsyncPacketSocket，而这里用到的 Port 和 AsyncPacketSocket 对象，都是在收集本地 candidate 过程中创建的，并不会重新创建。

Unified Plan 模式下，transceiver 会在调用 AddTrack 时被创建，audio 和 video 各 add 一次，那就会创建两个 transceiver。

sender, receiver, channel, 这三者是什么关系？

transceiver 的 channel 会在 `PeerConnection::SetLocalDescription` 时被创建，transceiver 有个 media type 字段，用来表明是 audio 还是 video 类型（DataChannel 不走 transceiver 这套），创建 channel 时就会根据 media type 对应创建 `cricket::VoiceChannel` 或 `cricket::VideoChannel`。

在 `PeerConnection::SetRemoteDescription` 时，执行 transceiver 的 channel 的 `BaseChannel::SetRemoteContent` 函数，其中会创建 `VideoSendStream`。

视频数据：

+ 在 `RtpVideoSender::OnEncodedImage` 里，根据 `stream_index` 找到 `ModuleRtpRtcpImpl`，发送 RTP 报文；
  - `stream_index` 默认是 0，如果有 simulcast，那不同的层可能有不同的 index；
  - 不同的 RTP module、`RtpPayloadParams` 都用数组保存，通过 stream index 索引；
  - RTP module 和 payload params 都在 `RtpVideoSender` 构造时准备好，
+ 

PeerConnection 是全双工的，既可以发送数据，也可以接收数据，而且还有 bundle 机制，可以通过同一个 Socket 同时收发 audio/video/data 数据。

音视频、data channel 的数据包的流动过程。

socket/connection 的实际使用。

DTLS？SRTP？

使用 P2P 连接的各种 transport, channel 类结构、关系？

在通讯过程中，可能会选出新的 Connection，这种情况是如何处理的？

STUN binding, TURN allocate, STUN ping, 实际数据都是使用同一个 AsyncUdpSocket 收发的，如何区分收到的数据应该由谁处理？

DataChannel 的这个消息，最终没有消费者。

什么时候可以用？选出来了就可以用了？

FindNextPingableConnection

+ offer answer 里包含 P2P 连接（地址）信息？可以包含，也可以通过 candidate 单独发送给对方
+ DataChannel 的连接是怎么建立的？ICE P2P 的详细过程？
  + 收集本地 candidate、设置远端 candidate、ICE 连接、连接使用？
  + 两端同时 ICE 连接？还是先拿到对方 candidate 的先连接？一个 DataChannel 会建立几个连接？
+ 如何 bundle 的？
+ PeerConnection 类是否会维持 P2P 连接？
+ ICE P2P 和 usrsctp 如何关联？
+ ICE P2P 模块如何单独使用？

+ `transport_name` 就是 sdp 里的 mid；
+ IceTransportInternal/candidate 有个 component id 的概念，它的类型是 RTCIceComponent，1 表示 RTP，2 表示 RTCP，如果启用 RTCP mux，component id 也是 1；
+ remote candidate 有个 `origin_port` 的概念，啥意思？

刚创建出 PeerConnection 对象，就可以用其创建 DataChannel 对象了。就应该在刚创建 PC 的时候创建 DataChannel，这样后面生成的 sdp 里才会包含它。中途创建 DataChannel，是否需要重新 negotiate？

sdp 和 P2P 有没有关系？不收发音视频，也不建立 DataChannel，是否会建立 P2P 连接？没有 m section，setLocalDescription 会失败。
