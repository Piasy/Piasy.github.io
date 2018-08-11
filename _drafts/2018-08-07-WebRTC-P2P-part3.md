---
layout: post
title: WebRTC Native 源码导读（十三）：P2P 连接的使用
tags:
    - 实时多媒体
    - WebRTC
    - 网络
---

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
