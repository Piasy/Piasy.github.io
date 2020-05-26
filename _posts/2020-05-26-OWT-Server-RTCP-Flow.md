---
layout: post
title: OWT Server 进阶（三）：RTCP 流程
tags:
    - 实时多媒体
    - WebRTC
    - OWT
---

原定的「OWT Server 进阶（二）」主题是 OWT Server 各个模块的宏观结构和流程，不过这个主题在 Hack WebRTC 书中已经涵盖，所以就不在博客里发了，此外书中还详细总结了信令和媒体的 JS/C++ 代码调用流程，敬请期待 :)

正如 [RTP: Audio and video for the Internet 中文版阅读笔记](/2020/05/23/RTP/index.html) 中所言，梳理 OWT Server RTCP 流程的主要目的，是为了解决 Server 没有给发布端发送音频 RTCP RR 报文的问题，在准备好了理论知识和实用工具（[VSCode 远程调试 Docker 里的 OWT Server](/2020/05/05/VSCode-Remote-Debug-in-Docker/index.html)）后，现在是时候启动了。

## RTCP 补充

本文会涉及下面几种类型的 RTCP 包：

+ SR, Sender Report: 200;
+ RR, Receiver Report: 201;
+ RTPFB, Transport Layer Feedback: 205;
  + [FMT 1: NACK](https://tools.ietf.org/html/rfc4585#section-6.2.1);
  + [FMT 15: transport-cc](https://tools.ietf.org/html/draft-holmer-rmcat-transport-wide-cc-extensions-01#section-3.1)（draft 里 PT 有误，写成 206 了，实际应该是 205）;
+ PSFB, Payload Specific Feedback: 206;
  + [FMT 1: PLI](https://tools.ietf.org/html/rfc4585#section-6.3.1);
  + [FMT 4: FIR](https://tools.ietf.org/html/rfc5104#section-4.3.1);
  + [FMT 15: REMB](https://tools.ietf.org/html/draft-alvestrand-rmcat-remb-03#section-2.2);

## 核心类关系

Hack WebRTC 书中虽然总结了详细的 C++ 代码调用栈，但却缺少一个对 C++ 核心类的宏观介绍，所以这里先补充一下。

在 Licode v6 中，`WebRtcConnection` 是和 WebRTC 客户端打交道的核心类，信令和媒体数据都会经过它；`MediaStream` 则是对媒体流的一个抽象，概念上和 WebRTC API 里的 MediaStream 对应，信令里 publish 的结果和 subscribe 的目标也正是 `MediaStream` 的 id。由于 Licode 支持单 PC 模式（即同一个客户端推收流都走同一个 PC），所以 `WebRtcConnection` 和 `MediaStream` 是一对多的关系，一个 `WebRtcConnection` 可以有多个 `MediaStream`。

`MediaStream` 继承自 `MediaSink`、`MediaSource`、`FeedbackSink` 和 `FeedbackSource` 等类（定义在 `third_party/licode/erizo/src/erizo/MediaDefinitions.h` 中）：

+ `MediaSink`：定义了 `deliverAudioData`、`deliverVideoData` 和 `deliverEvent` 接口，用来接收音视频数据包、事件；也记录了数据源的 SSRC；此外，它还可能向数据源提供反馈，因此它有一个 `FeedbackSource* sink_fb_source_` 成员；
+ `MediaSource`：对数据源的抽象，它拥有 `video_sink_`、`audio_sink_` 和 `event_sink_` 这三个 `MediaSink*` 成员，以便子类向它们投递音视频数据包、事件；它也记录了自己的 SSRC；此外，它还可能接收来自反馈源（接收方）的反馈，因此它有一个 `FeedbackSink* source_fb_sink_` 成员；
+ `FeedbackSink`：定义了 `deliverFeedback` 接口，用来接收来自反馈源的反馈；
+ `FeedbackSource`：对反馈源的抽象，它拥有 `FeedbackSink* fb_sink_` 成员，以便子类向其投递反馈数据包；

在一个订阅关系中，A 发布的流被 B 订阅时，数据会从 A 的 `MediaStream` 传递到 B 的 `MediaStream`，同时 B 的 `MediaStream` 也会给 A 的 `MediaStream` 发送反馈，`MediaStream` 同时扮演了这四种角色，所以继承了这四个类。

在 `WebRtcConnection::onTransportData` 收到数据后，会通过 SSRC 找到正确的 `MediaStream`，比如：

+ 来自发布端 PC 的 RTP 包，应该交给 `MediaSource` SSRC 和 RTP 包头 SSRC（`head->getSSRC()`）相同的 `MediaStream`（`media_stream->isSourceSSRC(ssrc)`）（_注意这里并不是 source 到 sink 的逻辑，而是把数据交给正确的 `MediaStream`（source），以便执行之后的「source 到 sink 的逻辑」_）；
+ 来自发布端 PC 的 RTCP SR 包，应该交给 `MediaSource` SSRC 和 RTCP 包头 SSRC（`head->getSSRC()`）相同的 `MediaStream`（`media_stream->isSourceSSRC(ssrc)`）；
+ 来自订阅端 PC 的 RTCP 反馈包，应该交给 `MediaSink` SSRC 和反馈包 source SSRC（`head->getSourceSSRC()`）相同的 `MediaStream`（`media_stream->isSinkSSRC(ssrc)`）；

_但实际上在 `WebRtcConnection::onTransportData` 中处理 RTP 包、在 `WebRtcConnection::onRtcpFromTransport` 中处理 RTCP 包时，判断的都是 `media_stream->isSourceSSRC(ssrc) || media_stream->isSinkSSRC(ssrc)`，我推测是另一个条件在实际运行时一定不会满足，所以没有导致问题，这一推测也得到了 OWT 官方研发的确认。_

除了 Licode 的四个核心接口，OWT 本身也定义了两个核心接口，它们分别是（定义在 `source/core/owt_base/MediaFramePipeline.h` 中）：

+ `FrameDestination`：定义了 `onFrame` 接口，用来接收音视频帧；
+ `FrameSource`：对帧源的抽象，它拥有 `m_audio_dests` 和 `m_video_dests` 成员，以便子类向它们投递音视频帧；

然后有四个核心类，对这两套核心接口进行了组合，它们分别是：

+ `Audio/VideoFrameConstructor`：继承了 `MediaSink` 和 `FrameSource`，从 `MediaSource` 接收音/视频数据包，并组装出音/视频帧；
+ `Audio/VideoFramePacketizer`：继承了 `FrameDestination` 和 `MediaSource`，从 `FrameSource` 接收音/视频帧，并打包出音/视频数据包；

所以，不涉及流扩散时，A 发布的流被 B 订阅，详细的数据流程为：

```bash
# audio
                   A MediaStream           (MediaSource)  =>
(MediaSink)        A AudioFrameConstructor (FrameSource)  =>
(FrameDestination) B AudioFramePacketizer  (MediaSource)  =>
(MediaSink)        B MediaStream

# video
                   A MediaStream           (MediaSource)  =>
(MediaSink)        A VideoFrameConstructor (FrameSource)  =>
(FrameDestination) B VideoFramePacketizer  (MediaSource)  =>
(MediaSink)        B MediaStream
```

在 `wrtcConnection.js processSignalling` 函数中，收到 offer 后，会调用 `setupTransport`。对于发布端的 offer，其 `direction` 是 `in`，所以会创建 `Audio/VideoFrameConstructor`，并调用它们的 `bindTransport` 函数，最终把发布端从 `MediaStream` 到 `Audio/VideoFrameConstructor` 的数据通道建立起来。对于订阅端的 offer，其 `direction` 是 `out`，所以会创建 `Audio/VideoFramePacketizer`，并调用它们的 `bindTransport` 函数，最终把订阅端从 `Audio/VideoFramePacketizer` 到 `MediaStream` 的数据通道建立起来。

发布端的 `Audio/VideoFrameConstructor` 和订阅端的 `Audio/VideoFramePacketizer` 关联，则是在 `linkup` 时建立的：

```bash
Conference Agent  roomController.js  linkup（subscribe 的内部函数）  =>
webrtc/index.js  that.linkup  =>
webrtc/connections.js  that.linkupConnection  =>
webrtc/wrtcConnection.js  that.addDestination  =>
webrtc/wrtcConnection.js  WrtcStream 类的 addDestination：调用 C++ 接口关联发布端和订阅端
```

涉及到流扩散时，也只是在 `Audio/VideoFrameConstructor` 和 `Audio/VideoFramePacketizer` 之间增加一对 `InternalOut` 和 `InternalIn` 而已：

```bash
# audio
                   A MediaStream           (MediaSource)  =>
(MediaSink)        A AudioFrameConstructor (FrameSource)  =>
(FrameDestination) A InternalOut           (network)      =>
(network)          B InternalIn            (FrameSource)  =>
(FrameDestination) B AudioFramePacketizer  (MediaSource)  =>
(MediaSink)        B MediaStream

# video
                   A MediaStream           (MediaSource)  =>
(MediaSink)        A VideoFrameConstructor (FrameSource)  =>
(FrameDestination) A InternalOut           (network)      =>
(network)          B InternalIn            (FrameSource)  =>
(FrameDestination) B VideoFramePacketizer  (MediaSource)  =>
(MediaSink)        B MediaStream
```

建立关联的过程，这里就不赘述了。

## RTCP 流程

前面总结的都是数据流程，下面总结反馈流程。

`FrameSource` 也定义了 `onFeedback` 接口，以便接收来自接收方的反馈，这个接口只在 `FrameDestination::deliverFeedbackMsg` 中被调用，即接收方都将通过 `deliverFeedbackMsg` 函数发送反馈。_不过目前的实现这个接口应该只用来请求视频关键帧。_

`Audio/VideoFramePacketizer` 的 `bindTransport` 中，会把 `Audio/VideoFramePacketizer` 作为 `FeedbackSink` 设置给 `MediaStream` 的 `sink_fb_source_` 成员（实际上就是 `MediaStream` 自身），所以也就把订阅端从 `MediaStream` 到 `Audio/VideoFramePacketizer` 的反馈通道建立起来了。`Audio/VideoFrameConstructor` 的 `bindTransport` 中，`FeedbackSink` 是 `MediaStream` 的 `source_fb_sink_` 成员（实际上就是 `MediaStream` 自身），所以也就把发布端从 `Audio/VideoFrameConstructor` 到 `MediaStream` 的反馈通道建立起来了。

`Audio/VideoFramePacketizer` 都利用了 WebRTC 的 RtpRtcp 模块来实现音/视频的 RTP 封包逻辑，`VideoFrameConstructor` 则利用了 WebRTC 的 VCM（其中包括 RtpRtcp 模块）实现了视频 RTP 解包的逻辑。音频解包因为很简单，所以就直接在 `AudioFrameConstructor` 中实现了。

_WebRTC 的音视频的 RTP 解包有些区别，视频的 RTP 解包和 jitter buffer 结合紧密，共同构成了 VCM，和解码（其实解码也是 VCM 的一部分）、渲染分得很清楚；音频的 RTP 解包则是 NetEQ 的一部分，而 NetEQ 还包括了 jitter buffer、错误隐藏、播放控制等等逻辑，和解码、播放也紧密结合，比 VCM 复杂得多。_

WebRTC 的 RtpRtcp 模块除了实现 RTP 封包解包逻辑外，还有 RTCP 包处理逻辑，`Audio/VideoFramePacketizer` 的 `deliverFeedback_` 实现就是把 RTCP 报文交给了 WebRTC 的 RtpRtcp 模块。对于音频来说，`AudioFramePacketizer` 并未从 WebRTC 的 RtpRtcp 模块接收命令并反馈给发布端的 `AudioFrameConstructor`。对于视频来说，`VideoFramePacketizer` 会接收来自 WebRTC 的 RtpRtcp 模块的 `OnReceivedIntraFrameRequest` 回调（根源是订阅端的 `MediaStream`），并向发布端的 `VideoFrameConstructor` 进行反馈，这也正是前面提到的「`FrameDestination` 到 `FrameSource` 的反馈通道只用来请求视频关键帧」的一部分。

不过虽然目前的实现没有用来进行其他的反馈，但这条从订阅端的 `MediaStream` 到 `Audio/VideoFramePacketizer`，再到发布端的 `Audio/VideoFrameConstructor`，进而到发布端 `MediaStream` 的反馈链路是完整的，需要进行其他更多反馈时，增加调用即可。

综上所述，反馈的传递路径为：

```bash
# audio
               B MediaStream           (FeedbackSource)    =>
(FeedbackSink) B AudioFramePacketizer                      =>
               B WebRTC RtpRtcp module
               B AudioFramePacketizer  (FrameDestination)  =>
(FrameSource)  A AudioFrameConstructor (FeedbackSource)    =>
(FeedbackSink) A MediaStream

# video
               B MediaStream           (FeedbackSource)    =>
(FeedbackSink) B VideoFramePacketizer                      =>
               B WebRTC RtpRtcp module
               B VideoFramePacketizer  (FrameDestination)  =>
(FrameSource)  A VideoFrameConstructor                     =>
               A WebRTC RtpRtcp module                     =>
               A WebRTCTransport       (FeedbackSource)    =>
(FeedbackSink) A MediaStream
```

对于视频来说，除了来自订阅端的反馈，发布端 `VideoFrameConstructor` 持有的 WebRTC 的 RtpRtcp 模块应该也会发出一些反馈。

---

接下来我们看具体的代码流程。

在 `WebRtcConnection` 类里，RTCP 包的处理流程为：

```bash
WebRtcConnection::onTransportData  =>       // (1)
WebRtcConnection::onRtcpFromTransport  =>   // (2)
WebRtcConnection::onREMBFromTransport       // (3)
或
MediaStream::onTransportData
```

要点如下：

1. `onTransportData` 中收到数据后，如果是 RTCP 包，则交给 `onRtcpFromTransport` 处理；
1. 在 `onRtcpFromTransport` 中，如果 RTCP 包是 REMB 包，则交给 `onREMBFromTransport` 处理，否则直接交给 `MediaStream::onTransportData`；
1. OWT Server 注释掉了 REMB 包转发的逻辑，所以订阅端的 REMB 包不会被转发到发布端；不过即便转发，最终也是交给 `MediaStream::onTransportData`，只不过包的内容会重新生成；

包交到 `MediaStream::onTransportData` 后，最终会到达 `MediaStream::read` 函数。对于反馈包（RR, RTPFB, PSFB，它们只会来自于订阅端），将直接交给 `fb_sink_`，也就是订阅端的 `Audio/VideoFramePacketizer`，前面我们已经总结过了，这条反馈路径目前只用来请求了视频关键帧。对于 SR 包（只会来自于发布端），会被交给 `MediaSink::deliverAudio/VideoData`，也就是发布端的 `Audio/VideoFrameConstructor`，`AudioFrameConstructor` 会忽略 SR 包，`VideoFrameConstructor` 则会把 SR 包交给 VCM 进行处理。

处理来自客户端的 RTCP 包逻辑，就是上面这些，下面我们再看一下 OWT Server 内部给客户端发送 RTCP 包的情况。

_经过一番调试、日志分析，以及其他机缘巧合，我注意到 OWT 注释了不少 Licode 的代码，并且主要集中在 `MediaStream::initializePipeline` 中，去掉了 `pipeline_` 的很多 handler，而其中就有一个 `RtcpFeedbackGenerationHandler`，它是用来从 Server 内部向客户端主动发送 RTCP 反馈包的。实际上 [OWT 4.2.x 就有一个 patch 是恢复这个类的](https://github.com/open-webrtc-toolkit/owt-server/blob/4.2.x/scripts/patches/licode/0015-Add-FeedbackGenerator-for-audio-packets.patch)，而五月初我给 OWT 官方研发反馈了没有音频 RR 的问题之后，没过多久他就在 [4.3.x 上也加上了这个 patch](https://github.com/open-webrtc-toolkit/owt-server/blob/4.3.x/scripts/patches/licode/0014-Add-FeedbackGenerator-for-audio-packets.patch)。_

_Licode 的 pipeline 设计，我们留在下一篇中分析。_

这里简单总结一下 `RtcpFeedbackGenerationHandler` 的逻辑：

+ `RtcpFeedbackGenerationHandler::write` 不做任何处理，因为发包时不会触发任何 RTCP 反馈逻辑；
+ `RtcpFeedbackGenerationHandler::read` 中如果发现收到的是 SR 包，则交给 RR generator 处理；
+ 如果是 RTP 包，则交给 RR generator、NACK generator 处理，处理之后若需要发送 RR 或 NACK，则发送之；

## 客户端 RTCP 流程

现在再总结一下客户端处理 RTCP 包的流程：

+ 在 `PeerConnection::Initialize` 中创建了 `JsepTransportController`，创建时通过 `JsepTransportController::Config` 设置了 `rtcp_handler`，处理收到的 RTCP 包；
+ RTCP 包是在 `network_thread` 收到的，而处理 RTCP 包则切换到了 `worker_thread`，由 `Call::DeliverRtcp` 负责；
+ 由于在 `rtcp_handler` 中调用 `Call::DeliverRtcp` 时，`media_type` 是写死的 `MediaType::ANY`，所以 RTCP 包会交给每个 `VideoReceiveStream`/`AudioReceiveStream`/`VideoSendStream`/`AudioSendStream`，任其按需处理；
+ 这四种 stream 都是把 RTCP 包交给 `ModuleRtpRtcpImpl` 进行处理，并且是不同的实例；
+ 最终 RTCP 包会在 `RTCPReceiver::ParseCompoundPacket` 函数中，根据不同的 block 类型，进行不同的处理；

音频的 `remote-inbound-rtp` 在 `RTCStatsCollector::ProduceAudioRTPStreamStats_n` 中生成，源于 `voice_sender_info.report_block_datas`，而 `report_block_datas` 则最终来源于 `ModuleRtpRtcpImpl::GetLatestReportBlockData`，也就是 `RTCPReceiver::GetLatestReportBlockData`（`received_report_blocks_`），而 `received_report_blocks_` 则会在 `RTCPReceiver::HandleReportBlock` 中填充，`HandleReportBlock` 则会在收到 `SenderReport` 和 `ReceiverReport` 后被调用。

`registered_ssrcs_` audio 和 video 不一样，但收到的 RR 包，SSRC 都是 video 的，即只收到了 video 的 RR 包。

---

最后，再解答一下很早之前我记在这个草稿里的问题：DTLS 只有握手包才是 `DtlsTransport::isDtlsPacket`？RTP/RTCP 包不是？DTLS 和 SRTP 的关系是？

+ DTLS 只有握手包才是 `DtlsTransport::isDtlsPacket`，RTP/RTCP 包不是；
+ 根据 WebRTC Glossary [DTLS-SRTP](https://webrtcglossary.com/dtls-srtp/) 和 [SDES](https://webrtcglossary.com/sdes/) 词条，我们就可以知道（当然看 SRTP 和 DTLS RFC 也可以），SRTP 是 RTP 的一个安全的 profile，其中要求对 RTP 包做端到端加密（对称加密），但是加解密的密钥如何交换，RFC 中并未要求，最开始 WebRTC 是把密钥明文放在 SDP 中，假设 SDP 会通过安全通道交换，但后来被废弃，而是改为使用 DTLS 来做密钥交换，DTLS 会利用非对称加密算法，来实现认证、密钥交换；
