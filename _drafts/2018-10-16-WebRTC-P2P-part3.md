---
layout: post
title: WebRTC Native 源码导读（十五）：P2P 连接的使用
tags:
    - 实时多媒体
    - WebRTC
    - 网络
---

## 视频数据 encoder => sender

+ `VideoStreamEncoder` 从 Capturer 接收视频数据，调用各平台的硬编实现，完成编码，再把数据交给 `VideoSendStreamImpl`，其过程如下图（在[视频数据 native 层之旅](/2018/05/24/WebRTC-Video-Native-Journey/#发送端视频数据的-native-层之旅)中有详细分析）：

![](https://imgs.piasy.com/2018-10-17-video_from_capturer_to_send_stream.jpg)

+ 有一点这里需要强调一下，已编码数据到达 `VideoStreamEncoder` 之前，会转换 NALU 存储格式、生成 RTP fragmentation header；
  - iOS 编出来的 NALU 是按 AVCC 格式存储的，会转换为 Annex-B 格式，转换过程中也会生成 RTP fragmentation header，这一逻辑封装在 `sdk/objc/components/video_codec/nalu_rewriter.cc` 的 `H264CMSampleBufferToAnnexBBuffer` 中；
  - 安卓编出来的就是 Annex-B 格式，因此无需做格式转换，但会生成 RTP fragmentation header，这一逻辑封装在 `sdk/android/src/jni/videoencoderwrapper.cc` 的 `OnEncodedFrame` 中；
+ `VideoStreamEncoder` 在把数据交出之前/之后，会向 `encoder_stats_observer_`, `overuse_detector_` 和 `quality_scaler_` 进行汇报，具体作用这里暂且跳过；
+ `VideoSendStreamImpl` 先把已编码数据回调给可能的 `post_encode_callback`，然后向 `check_encoder_activity_task_` 报告活动状态，再把数据交给 `rtp_video_sender_`，最后再在 `OnBitrateAllocationUpdated` 里向 `rtp_video_sender_` 更新码率；
  - 更新码率有个 throttle 的逻辑，防止更新太频繁；
+ 在 `RtpVideoSender::OnEncodedImage` 里，根据 `stream_index` 找到 `ModuleRtpRtcpImpl`，调用 `SendOutgoingData` 发送 RTP 报文；
  - `stream_index` 默认是 0，如果有 simulcast，那不同的层可能有不同的 index；
  - 不同的 RTP module（`RtpRtcp`，实现类为 `ModuleRtpRtcpImpl`）、`RtpPayloadParams` 都用数组保存，通过 stream index 索引；
  - RTP module 和 payload params 都在 `RtpVideoSender` 构造时准备好，
  - 发送报文所需的数据，除了 `EncodedImage` 的字段外，还有 payload type 和 `RTPVideoHeader`，payload type 保存在 `RtpConfig` 中，也在构造 `RtpVideoSender` 时传入，`RTPVideoHeader` 则保存在 payload params 中；
+ `ModuleRtpRtcpImpl` 先检查是否需要发送 RTCP 包，然后计算出重传超时时间，和传入的参数一并交给 `RTPSender`；
+ `RTPSender` 计算出 `rtp_timestamp`、根据 payload type 得到 video type 后、适时设置 RTP header 的 playout delay extension 后，把数据交给 `RTPSenderVideo` 进行 RTP 封包工作；
+ `RTPSenderVideo::SendVideo` 里实现了 RTP 封包的逻辑，包括设置 header 和 header extension、packetization/fragmentation、调用 FEC 等；
  - 如果不启用 FEC，则调用 `SendVideoPacket` 把 RTP packet 交给 `RTPSender`，并向 `video_bitrate_` 更新码率统计信息；
  - 如果启用 FLEX FEC，则调用 `SendVideoPacketWithFlexfec`，其中进行 FEC 操作，并把需要发送的 RTP packet 调用 `SendVideoPacket` 发送；
  - 如果启用 RED，则调用 `SendVideoPacketAsRedMaybeWithUlpfec`，其中也是进行 FEC 操作，最后把需要发送的 RTP packet 调用 `SendVideoPacket` 发送；
+ 再往下传递的都是 RTP packet 了，基本不再区分是音频还是视频，稍后我们再继续分析；

RTP 封包的逻辑，我们以后再展开分析，音频数据到达 `RTPSender` 之前的逻辑，这里也先跳过。

## sender 到 channel

![](https://imgs.piasy.com/2018-05-11-ios_rtp_sender_to_channel.png)

## channel 到 socket

![](https://imgs.piasy.com/2018-05-11-ios_rtp_channel_to_socket.png)

## P2P 连接建立时的 socket 如何使用？

## 各种类之间的关系？

使用 P2P 连接的各种 transport, channel 类结构、关系？

sender, receiver, channel, 这三者是什么关系？

Unified Plan 模式下，transceiver 会在调用 AddTrack 时被创建，audio 和 video 各 add 一次，那就会创建两个 transceiver。

transceiver 的 channel 会在 `PeerConnection::SetLocalDescription` 时被创建，transceiver 有个 media type 字段，用来表明是 audio 还是 video 类型（DataChannel 不走 transceiver 这套），创建 channel 时就会根据 media type 对应创建 `cricket::VoiceChannel` 或 `cricket::VideoChannel`。

在 `PeerConnection::SetRemoteDescription` 时，执行 transceiver 的 channel 的 `BaseChannel::SetRemoteContent` 函数，其中会创建 `VideoSendStream`。

---

+ `webrtc::internal::VideoSendStream` 的 `send_stream_` 成员是 `VideoSendStreamImpl`，`video_stream_encoder_` 成员是 `VideoStreamEncoder`；
+ `WebRtcVideoSendStream` 的 `stream_` 成员是 `webrtc::VideoSendStream`（实际是子类 `webrtc::internal::VideoSendStream`）；


> 连接建立成功后，就可以收发应用层的数据了，数据的收发将会通过选出来的 Connection 对象完成，Connection 则是调用 Port，Port 则是调用 AsyncPacketSocket，而这里用到的 Port 和 AsyncPacketSocket 对象，都是在收集本地 candidate 过程中创建的，并不会重新创建。

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
