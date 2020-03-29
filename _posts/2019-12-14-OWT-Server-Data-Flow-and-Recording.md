---
layout: post
title: OWT Server 进阶（一）：音视频数据转发和录制流程
tags:
    - 实时多媒体
    - WebRTC
    - OWT
---

今年上半年开始接触 OWT，当时用起来之后整理了 [OWT Server 快速入门](/2019/04/14/OWT-Server-Quick-Start/index.html)，其中初步涉及了音视频数据的转发流程。最近在考虑实现客户端的推收流录制，正好之前也遇见了 OWT 录制的一些问题，所以这次就先对 OWT 的音视频数据转发和录制流程探个究竟，一方面可以作为客户端实现的借鉴，另一方面也可以为向 OWT Server 提交 PR 做好准备。

_注：本文分析的代码，是 OWT Server 的 4.2.x 分支，4.3.x 更新了 Licode 的版本，有了较大的变化_。

## 音视频数据转发流程

Licode（OWT 使用了一些 Licode 的代码）定义了 `Transport` 接口，并提供了 `DtlsTransport` 实现；`WebRtcConnection` 类实现了 `TransportListener` 接口，故 `DtlsTransport::onIceData` 收到发布端的数据并解密后，会交给 `WebRtcConnection::onTransportData`。`DtlsTransport` 的数据从哪里来？来自 `LibNiceConnection::onData`，它是对 [libnice/libnice](https://github.com/libnice/libnice) 的封装。原本 Licode 使用的是 [resiprocate/nICEr](https://github.com/resiprocate/nICEr)，对应的函数为 `NicerConnection::onData`，但 OWT 用 libnice 替换了 nICEr。

`WebRtcConnection` 之后的数据传递流程为：

```
WebRtcConnection::onTransportData ->
Pipeline::read ->
PacketReader::read ->
WebRtcConnection::read ->
MediaSink::deliverAudio/VideoData
```

`WebRtcConnection` 的 `audio_sink_` 就是 `AudioFrameConstructor`，`video_sink_` 就是 `VideoFrameConstructor`。

所以音频数据接下来的流程就是：

```
MediaSink::deliverAudioData ->
AudioFrameConstructor::deliverAudioData_ ->
FrameSource::deliverFrame ->                    // (1)
AudioFramePacketizer::onFrame ->                // (2)
WebRtcConnection::deliverAudioData_ ->          // (3)
WebRtcConnection::sendPacket ->
WebRtcConnection::write ->
DtlsTransport::write                            // (4)
```

要点如下：

1. 有客户端订阅某个流时，就会给 `FrameSource` 增加 `FrameDestination`（具体流程可以查看 [OWT Server 快速入门：建立数据转发关系](/2019/04/14/OWT-Server-Quick-Start/index.html#subscribe-%E5%BB%BA%E7%AB%8B%E6%95%B0%E6%8D%AE%E8%BD%AC%E5%8F%91%E5%85%B3%E7%B3%BB)的分析），而 `FrameSource::deliverFrame` 里会把帧转发给所有的 `FrameDestination`；
1. 音频数据的 `FrameDestination` 实现类为 `AudioFramePacketizer`，故音频帧就送到了 `AudioFramePacketizer::onFrame`；
1. `AudioFramePacketizer` 的 `audio_sink_` 是订阅端的 `WebRtcConnection`，故欲发送的音频数据就到了 `WebRtcConnection::deliverAudioData_` 里；
1. 最后要转发的音频数据通过 `DtlsTransport` 发送给订阅端；

视频数据的流程为：

```
MediaSink::deliverVideoData ->
VideoFrameConstructor::deliverVideoData_ ->     // (1)
ViEReceiver::ReceivedRTPPacket ->               // (2)
RtpReceiverImpl::IncomingRtpPacket ->           // (3)
ViEReceiver::OnReceivedPayloadData ->           // (4)
VideoReceiver::IncomingPacket ->
VCMReceiver::InsertPacket ->                    // (5)

VideoReceiver::Decode ->                        // (6)
VideoFrameConstructor::Decode ->                // (7)
FrameSource::deliverFrame ->
VideoFramePacketizer::onFrame ->                // (8)
VideoFramePacketizer::receiveRtpData ->         // (9)
WebRtcConnection::deliverVideoData_ ->          // (10)
WebRtcConnection::sendPacket ->
WebRtcConnection::write ->
DtlsTransport::write                            // (11)
```

要点如下：

1. `VideoFrameConstructor` 有两个关键成员：`webrtc::ViEReceiver m_videoReceiver` 和 `webrtc::vcm::VideoReceiver m_video_receiver`；`webrtc::ViEReceiver` 有两个关键成员：`RtpReceiver rtp_receiver_` 和 `webrtc::vcm::VideoReceiver video_receiver_`；`m_video_receiver` 和 `video_receiver_` 指向同一个对象；
1. `VideoFrameConstructor::deliverVideoData_` 里调用 `ViEReceiver::ReceivedRTPPacket`，进而把 RTP 包交给 `RtpReceiver`，由其负责 RTP 解包逻辑；
1. `RtpReceiverImpl::IncomingRtpPacket` 调用 `RTPReceiverVideo::ParseRtpPacket`（`RTPReceiverStrategy` 接口的实现），其中会根据不同的 payload type，创建不同的 `RtpDepacketizer` 去提取载荷内容；
1. 提取到载荷数据后会回调到 `ViEReceiver::OnReceivedPayloadData`；
1. 之后载荷数据会交给 `VideoReceiver::IncomingPacket`，进而交给 `VCMReceiver::InsertPacket`，进而交给了 jitter buffer；
1. `VideoFrameConstructor::deliverVideoData_` 里调用完 `m_videoReceiver->ReceivedRTPPacket` 后，还会调用 `m_video_receiver->Decode`，以从 jitter buffer 中取出已完整收到的视频帧（未解码）；
1. `VideoFrameConstructor` 还实现了 `webrtc::VideoDecoder` 接口，在 `VideoFrameConstructor::OnInitializeDecoder` 中会把自己注册给 `webrtc::vcm::VideoReceiver`，所以调用 `VideoReceiver::Decode` 实际上会调用到 `VideoFrameConstructor::Decode`；当然，`VideoFrameConstructor` 只是做视频数据的转发，无需实现真正的解码逻辑；通过这一「伪解码」的过程，OWT 就利用 WebRTC 的 VCM 来实现了视频 RTP 的解包过程，拿到了完整的视频帧（未解码）；
1. 视频数据的 `FrameDestination` 实现类为 `VideoFramePacketizer`，故视频帧就送到了 `VideoFramePacketizer::onFrame`；其中需要实现视频帧的 RTP 封包逻辑，它是调用 `webrtc::RtpRtcp` 类来实现的，需要指出的是，调用 `m_rtpRtcp->SendOutgoingData` 时，第一个参数传的一律都是 `webrtc::kVideoFrameKey`，这是因为这个参数在 WebRTC H.264 封包的实现代码里并未使用，所以这里图省事就这么传了（_这一点是请教 OWT 研发人员后得到的解答_）；
1. 在 `VideoFramePacketizer::init` 函数中创建 `webrtc::RtpRtcp` 时，设置了 `outgoing_transport` 为 `WebRTCTransport` 对象（`core/owt_base/WebRTCTransport.h`），故 RTP 封包完成后，会调用到 `WebRTCTransport<dataType>::SendRtcp`，进而调用到 `RTPDataReceiver::receiveRtpData`（即 `VideoFramePacketizer::receiveRtpData`，`VideoFramePacketizer` 实现了 `RTPDataReceiver` 接口，并在构造函数里创建 `WebRTCTransport` 时传入了自己）；
1. `VideoFramePacketizer` 的 `video_sink_` 是订阅端的 `WebRtcConnection`，故欲发送的视频数据就到了 `WebRtcConnection::deliverVideoData_` 里；
1. 最后要转发的视频数据通过 `DtlsTransport` 发送给订阅端；

搞清楚了音视频数据的转发流程后，我们发现 OWT 对音视频 RTP 包做了解包和重新封包。其实音频倒也谈不上解包和封包，因为音频数据编码后都比较小，一个 RTP 包就能容纳，所以没有像视频那样复杂的封包和解包逻辑。

~~但实际上 SFU 是不需要做 RTP 解包的，收到发布端的 RTP 包后直接转发给订阅端即可，OWT 这里做解包，应该是给 MCU 功能用的。~~

经 OWT 官方指正，OWT 做 RTP 解包并非是为了 MCU 功能考虑，而是一个基础设计原则：

> 将传输层事务在接入节点终结掉，媒体在集群中流转以“媒体帧”为封装单元，所有操作均在“帧交互层”以上进行。

SFU 是否组帧，效果上并没有简单明确的好坏之分：

> 端到端的全程延迟取决于每一帧什么时候被完整拼出来，而不是第一个包什么时候到达。理论上看并不会因为中间组过一次帧而显著增加“最后一块拼图”的到达时间，增加的只是将收齐的rtp包序列拼装成帧及将帧打成包序列的CPU计算过程的时间，这个时间一般是毫秒以内。另一方面将传输拆成帧接力的两段的话，可以比盲转更早发现丢包并请求重传，实际上是帮助减小了“最后一块拼图”的延迟。但实际弱网环境时刻在变化，很难模拟，需要实际测试一下效果，并调整各阶段的对抗手段，来达到相对较好的抗丢包效果。

## 服务端录制

启用服务端录制需要调用 RESTful API，其处理流程为：

```
management_api/api.js ->                                                // (1)
management_api/resource/recordingsResource.js exports.add ->
management_api/requestHandler.js exports.addServerSideSubscription ->   // (2)
agent/conference/conference.js that.addServerSideSubscription ->
agent/conference/accessController.js that.initiate ->                   // (3)
agent/conference/rpcRequest.js that.initiate ->                         // (4)
agent/recording/index.js that.subscribe                                 // (5)
```

要点如下：

1. 定义 RESTful API 的处理函数；
1. `agent/conference/conference.js that.rpcAPI` 注册了 RPC server；
1. 根据 `sessionOptions.type` 找到 RPC server node 为 recording node；
1. direction 是 out，故 RPC 调用（recording node 的）subscribe；
1. 创建 AVStreamOut；

`AVStreamOut` 是 `agent/addons/avstreamLib` 扩展里的类型，其 C++ 实现为 `agent/addons/avstreamLib/AVStreamOutWrap.cc`，在 `AVStreamOutWrap::New` 中, type 是 `file`，故实际创建的是 `owt_base::MediaFileOut`。另一种 type 是 `streaming`，用于转推 `rtsp`/`rtmp`/`hls`/`dash`。

`agent/recording/index.js createFileOut` 函数里，创建 `AVStreamOut` 时传入了一个状态回调函数（_应该是由 `AVStreamOut::notifyAsyncEvent` 函数调用触发_），如果没有发生错误，就会调用 `notifyStatus` 抛出 `ready` 消息，之后的处理流程为（在 [OWT Server 快速入门：subscribe 建立数据转发关系](/2019/04/14/OWT-Server-Quick-Start/index.html#subscribe-%E5%BB%BA%E7%AB%8B%E6%95%B0%E6%8D%AE%E8%BD%AC%E5%8F%91%E5%85%B3%E7%B3%BB)曾分析过订阅端的流程，这里稍作更新即可得到录制的流程）：

```
agent/recording/index.js notifyStatus ->
agent/conference.js onSessionProgress ->
agent/conference/accessController.js onSessionStatus ->
agent/conference.js onSessionEstablished
```

之后，服务端建立推流端的数据到录制的转发关系：

```
agent/conference.js onSessionEstablished ->
agent/conference.js addSubscription ->
agent/conference/roomController.js subscribe ->
agent/conference/roomController.js spreadStream ->
agent/conference/roomController.js linkup ->
agent/webrtc/index.js linkup ->
agent/connections.js linkupConnection ->            // (1)
agent/webrtc/wrtcConnection.js addDestination ->
NAN_METHOD(AudioFrameConstructor::addDestination), NAN_METHOD(VideoFrameConstructor::addDestination) ->
FrameSource::addAudioDestination, FrameSource::addVideoDestination
```

要点如下：

1. `linkupConnection` 里调用 `conn.connection.receiver('audio')` 和 `conn.connection.receiver('video')` 分别获取音频和视频的 dest，然后调用 `connection.addDestination`；`connection.addDestination` 里的 `connection` 毫无疑问就是发布端的 `wrtcConnection`，但 dest 是什么呢？我们可以看到 `agent/recording/index.js` 里定义了 `connection.receiver` 函数，返回的就是 `this`，因此音视频的 dest 也就都是 `AVStreamOut` 了，所以音视频数据最终会送到 `AVStreamOut::onFrame` 中；

`owt_base::MediaFileOut` 继承自 `owt_base::AVStreamOut`。在 `AVStreamOut::onFrame` 中，会把帧放入 `m_frameQueue` 中；在构造函数里，会创建一个线程，线程函数为 `AVStreamOut::sendLoop`，其中不停从 `m_frameQueue` 里取帧，取到后调用 `AVStreamOut::writeFrame`；其中就是调用 ffmpeg 的 `av_interleaved_write_frame` 写入音视频帧。

通过前文音视频数据转发流程的分析，我们知道数据到达 `FrameDestination::onFrame` 之前已经处理好了乱序、重复等问题，所以这里直接写入文件即可。至于丢包的情况，如果最终视频包也没能按时收到，那也只能任其处于缺失状态了。

## 增加日志

最后分享一下如何给 OWT Server 增加日志，以便分析问题或分析调用流程。

如果原本类中已经有过日志代码了，那直接使用 `ELOG_INFO` 等宏即可。

如果原本类中没有任何日志代码，比如 `FrameSource` 类，则需要做以下几处修改：

+ `core/owt_base/MediaFramePipeline.h` 增加 `#include <logger.h>`；
+ `class FrameSource {` 后增加 `DECLARE_LOGGER();`；
+ `core/owt_base/MediaFramePipeline.cpp` 在所有 `FrameSource` 函数定义之前增加 `DEFINE_LOGGER(FrameSource, "FrameSource");`；
+ 在需要打日志的地方使用 `ELOG_INFO` 等宏即可；

编译时，如果报类似如下错误：

```bash
[FAIL] mediaFrameMulticaster.node Error:
/media/psf/Home/src/media/owt-server/source/agent/addons/mediaFrameMulticaster/
build/Release/mediaFrameMulticaster.node: undefined symbol: _ZN7log4cxx6Logger9getLoggerEPKc
```

这是因为该模块没有链接 `log4cxx`，在相应模块的 `binding.gyp` 的 `libraries` 中添加 `'-llog4cxx',` 即可。例如上面是 `agent/addons/mediaFrameMulticaster` 模块，那就修改 `agent/addons/mediaFrameMulticaster/binding.gyp` 即可。

此外，每个模块下都有 `log4cxx.properties` 和 `log4js_configuration.json`，用于控制日志输出的级别，需要注意如果开的日志级别不够，可能在日志文件里看不到日志。
