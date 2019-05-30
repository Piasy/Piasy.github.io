---
layout: post
title: WebRTC Native 源码导读（十）：视频数据 native 层之旅
tags:
    - 实时多媒体
    - WebRTC
---

分析完应用上层的视频采集、渲染、编码之后，原本我是打算把完整的 WebRTC 带到 Flutter 的世界里，形成 FlutterRTC 的，但后来仔细想想，这件事没多大意思，做出来了也不能产生多大价值，所以我决定调头深入底层。

本篇算是真正深入底层的第一篇，让我们深究一下之前没有深究的话题：视频数据 native 层之旅，以及 WebRTC 对视频数据的处理。最近对 iOS 上层的分析也不算白费，毕竟在 iOS 平台深入底层，无论是编译还是调试都更方便。

_本文的分析基于 WebRTC 的 #23295 提交_。

## 基本概念

扎到代码里之前，让我们先理清一些概念，下面这些内容都是总结自 [WebRTC 标准](http://w3c.github.io/webrtc-pc)。

### PC stream track source sink

一个 track 是一个音频或视频数据流（发送或接收）。它有输入和输出，对于要发送的 track 来说，输入是本地的采集，输出是远端，对于要接收的 track 来说，输入是远端，输出是本地的渲染。

source 则是数据源，可以作为 track 的输入，它的消费者叫 sink，一个 source 可以有多个 sink。给 track 加 sink 实际上都是给 track 的 source 加 sink。

发送方发送的一个 track，在接收端也一定表现为一个 track。一个 track 可以归属于一个或多个 stream，通过 stream id 区分，如果接收端尚不存在对应 stream，则会被创建。

加一个 stream 是为了更灵活的控制本地和远端的 track 组合，比如本地有音视频，但只发送音频。其实直接指定发送哪些 track 也能达到这个效果，但通过 stream 这个概念整合一下，会更清晰一些，况且总得有个数据结构容纳多个 track，那就索性给他一个名字（stream）吧。

PC 是 PeerConnection 的简称，它表示了终端之间的 P2P 连接，用来传输数据。PC 是 WebRTC 的门面，我们直接使用的都是 PC 的接口。

总结下这四个概念的关系：PC - 一到多个 stream - 零到多个 track - 一个 source。

### sender receiver transceiver

sender 负责编码、发送，receiver 负责接收、解码，一个 sender 至多有一个要发送的 track，一个要接收的 track 刚好有一个 receiver。规范规定 transceiver 包含一个 sender 和一个 receiver，它们拥有相同的 mid，WebRTC 实现的 unified plan 遵循了这一规定。一个 PC 有多个 transceiver。

addTrack 时会为其分配 transceiver/sender（新建或复用已有 transceiver/sender），也可以通过 addTransceiver 来建立 track - sender/receiver - transceiver 的关联。

## 发送端视频数据的 native 层之旅

[前面我们已经知道](/2018/04/05/WebRTC-iOS-DataChannel/#server-%E4%BA%A4%E4%BA%92p2p-%E8%BF%9E%E6%8E%A5)，在 `AppRTCMobile/ARDAppClient.m` `startSignalingIfReady` 函数中，我们会创建 `RTCPeerConnection` 对象，并创建 source, track, stream。

根据上面理清的 source track stream 之间的关系，我们可以推断，视频数据会从 capturer 到 source，再到 sender（track），而 sender 里又包括编码和发送。那接下来我们就利用 iOS 系统的优势，一「栈」到底，直捣黄龙。

### capturer => source => encoder

![](https://imgs.piasy.com/2018-05-11-ios_video_capturer_to_encoder.png)

+ `RTCVideoSource` 实现了 `RTCVideoCapturerDelegate`，所以视频数据从 `RTCCameraVideoCapturer` 送到了 `RTCVideoSource`；
+ 紧接着数据被送到了 `ObjCVideoTrackSource`，它继承自 `AdaptedVideoTrackSource`，其主要责任是对视频数据做「适配」，包括裁剪缩放，旋转，甚至丢弃操作；
+ 此外，它还负责做一个时间戳的「翻译」工作：相机采集本身会提供一个时间戳，这个时间戳比系统时钟更精确（步调更稳），但它是相对时间，而后面实际使用的时间戳都是 unix 时间戳，所以需要「翻译」；这件「翻译」工作实际由 `rtc_base/timestampaligner.h` 完成，它背后还有一个数学模型，感兴趣的朋友可以仔细查看代码、注释、测例；
+ 经过上面的处理后，视频数据已经由 `RTCVideoFrame` 变成了 `rtc::VideoFrame`，并交给了 `AdaptedVideoTrackSource`，由此正式开始 native 层之旅；之后如果视频需要旋转，那就做一个旋转操作（其他平台里此前可能未做旋转处理），然后交给 `VideoBroadcaster`；
+ 顾名思义，`VideoBroadcaster` 负责把视频数据“广播”出去（交给它的每个 sink），这里 broadcaster 还会检查 sink 对视频方向的需求，如果不匹配则丢弃数据，如果需要黑屏数据则传黑屏数据；另外，`AdaptedVideoTrackSource` 作为一个 source，可以添加 sink，但其实最终都是添加到了 `VideoBroadcaster` 里；
+ iOS 里只有一个 sink，那就是 `VideoStreamEncoder`，如果视频时间戳超过了当前时间，它会将其重置为当前时间，然后它会统计采集帧数、丢弃帧数（编码过慢导致），并定期输出日志（60s 一次），最后如果视频编码速度跟得上，就会把数据交给编码器，当然丢帧或编码的操作都是在编码线程（`encoder_queue_`）完成的；（`VideoStreamEncoder::OnFrame`）
+ `VideoStreamEncoder` 支持每帧数据编码之前，先执行一个回调；开始编码之前我们要先启动编码，此外，如果视频尺寸发生变化，或者数据类型发生变化（是否 texture），那我们都要重启编码；最后如果视频数据量没有超标，编码器也没有暂停，就进入下一环节；（`VideoStreamEncoder::MaybeEncodeVideoFrame`）
+ 在进行最后一次裁剪缩放检查后（_裁剪、缩放、旋转操作可能发生的位置确实有点多……_），把数据送给 `VideoSender`；（`VideoStreamEncoder::EncodeVideoFrame`）
+ `VideoSender` 会根据帧率、分辨率和 buffer 类型决定是否需要丢弃视频帧，如果确定要编码，就再交给 `VCMGenericEncoder`；
+ `VCMGenericEncoder` 检查了帧类型，并发出编码前回调之后（_回调也有点多……_），就把数据交给了 `ObjCVideoEncoder`，并最终交到了 `RTCVideoEncoderH264` 的手里；

这里简单小结一下：

+ 视频数据采集到之后，会在采集线程做裁剪缩放、旋转、黑屏、丢弃等处理；
+ 随后视频数据提交到编码线程，可能做裁剪缩放操作，最后进行编码；

### encoder => sender

![](https://imgs.piasy.com/2018-05-11-ios_rtp_encoder_to_sender.png)

+ `RTCVideoEncoderH264` 在收到系统的编码完成回调后，除了转换 NALU 存储格式外，还会顺带产生每帧视频数据的所有 RTP header，每个 NALU 作为一个 RTP fragment；解码器实际使用的大多是 Annex-B 格式，但中途处理时 AVCC 更方便（不用统计 unit 长度），把 H.264 数据封装进 RTP 报文之后，我们可以从包头里得知 unit 大小，这样就可以让数据保持 Annex-B 格式了；
+ `ObjCVideoEncoder` 在 `RegisterEncodeCompleteCallback` 函数里为 `RTCVideoEncoderH264` 设置了编码回调，它收到编码后的数据之后，会把 ObjC 的数据结构转换为 C++ 的数据结构，然后交给 `VCMEncodedFrameCallback`；
+ `VCMEncodedFrameCallback` 是 `VCMGenericEncoder` 设置的编码回调，它会把传入的 `EncodedImage` 拷贝一份，因为传入的是 `const &`，而这里却需要对这个结构内容做修改（_为什么不传普通引用呢，是有点作……_），然后填充 timing info, experiment id, simulcast id，最后交给 `VideoStreamEncoder`；交出数据之后，会再调用 `media_optimization::MediaOptimization` 的相关接口，_应该是做帧率限制用的，但我并未在项目中搜到对 `drop_next_frame` 的使用_；
+ 好了，接收视频数据并发起编码的是 `VideoStreamEncoder`，接收编码完成回调的也是它，之后就该要封装 RTP 报文发送了，这个过程不涉及视频数据的处理，以后再做展开；

### 数据 pipeline 建立过程

上面我们了解了视频数据的流动过程，但这个数据流的 pipeline 是怎么建立起来的呢？现在就来揭晓。

**RTCCameraVideoCapturer => RTCVideoSource**:

在 `ARDAppClient createLocalVideoTrack` 函数中，我们把 `RTCVideoSource` 作为 `RTCCameraVideoCapturer` 的 delegate，用来构造 capturer。

**RTCVideoSource => ObjCVideoTrackSource**:

`RTCVideoSource` 的 native source 就是 `ObjCVideoTrackSource`，这一关联在前者的 init 函数中建立；`ObjCVideoTrackSource` 是 `AdaptedVideoTrackSource` 的子类，所以它俩对应同一个对象。

**AdaptedVideoTrackSource => VideoBroadcaster**:

VideoBroadcaster 是 AdaptedVideoTrackSource 自己构造的成员变量，它的作用就是把视频数据分发给多个 sink，给 AdaptedVideoTrackSource 添加 sink，实际上是给 VideoBroadcaster 添加 sink。

**VideoBroadcaster => VideoStreamEncoder**:

+ `PeerConnection::AddTrack`：创建 sender, receiver, transceiver，并关联 track 和 sender；
  - track 是 `VideoTrack`，它的 `video_source_` 是 `AdaptedVideoTrackSource`，此外它也实现了 `VideoSourceInterface` 接口；
+ `PeerConnection::setLocalDescription`：在 `VideoRtpSender::SetVideoSend` 里关联 `media_channel_` 和 `track_`，不过 `media_channel_` 把 track 当做 `VideoSourceInterface` 使用；
  - `WebRtcVideoChannel` 把 source 交给了它的 `send_streams_`，即 `WebRtcVideoSendStream`；
+ `PeerConnection::SetRemoteDescription`：经由 VidelChannel（BaseChannel）、WebRtcVideoChannel（MediaChannel）、WebRtcVideoSendStream，创建 VideoSendStream，并关联 stream 和 source，最后给 source 绑定了 sink（VideoSendStream 的 VideoStreamEncoder）；
  - 在这个过程中 WebRtcVideoChannel 当了个中间人，它也实现了 VideoSourceInterface，它利用中间人的身份，保证对 `VideoSourceInterface::AddOrUpdateSink` 的调用发生在 `worker_thread_`；
  - WebRtcVideoChannel 的 `source_` 就是 `setLocalDescription` 里被绑定的 VideoTrack，给它添加 sink，实际上是给它的 source 添加 sink；
  
那么我们终于把 VideoStreamEncoder 交给 AdaptedVideoTrackSource 了。

**VideoStreamEncoder => VideoSender**:

VideoSender 是 VideoStreamEncoder 自己构造的成员变量，构造 VideoSender 时，VideoStreamEncoder 还把自己作为编码数据的回调注册给了 VideoSender。

**VideoSender => VCMGenericEncoder => ObjCVideoEncoder => RTCVideoEncoderH264**:

VideoStreamEncoder 开始接收视频数据后，会在 `VideoStreamEncoder::ReconfigureEncoder` 里创建编码器。

+ 调用 `ObjCVideoEncoderFactory::CreateVideoEncoder` 创建编码器，创建出来的是包装了 `RTCVideoEncoderH264` 的 `ObjCVideoEncoder`；
+ 调用 `VideoSender::RegisterExternalEncoder` 把 ObjCVideoEncoder 注册给 VideoSender，进而注册给 VCMEncoderDataBase；
+ 调用 `VideoSender::RegisterSendCodec`，其中会调用 `VCMEncoderDataBase::SetSendCodec` 把 VideoSender 包装成 VCMGenericEncoder，并调用 `VCMEncoderDataBase::GetEncoder` 将其取出保存；

那么我们终于建立起了完整的编码数据通道了，不过还不能高兴得太早，编码后数据抛出的通道是怎么建立的？让我们再接再厉。

**RTCVideoEncoderH264 => VCMEncodedFrameCallback => VideoStreamEncoder**:

在 `VCMEncoderDataBase::SetSendCodec` 中，我们会调用 `InitEncode` 初始化编码，之后就会注册编码回调。

+ 在 `ObjCVideoEncoder::RegisterEncodeCompleteCallback` 里，我们调用 `RTCVideoEncoderH264 setCallback` 注册回调，这样编码数据就会被交给 ObjCVideoEncoder，进而交给 VCMEncodedFrameCallback；
+ 而 VCMEncodedFrameCallback 是在构造 VideoSender、VCMEncoderDataBase、VCMGenericEncoder 时，VideoStreamEncoder 把自己作为编码数据的回调一路传递过来的；

好了，到这里我们就搞清楚了发送端视频数据的流动过程，以及这个数据通道的建立过程了，确实不容易，让我们先歇一口气 :)

---

## 接收端视频数据的 native 层之旅

接收端的视频数据处理就很简单了，就是解码、渲染。在这里我们跳过 RTP 报文处理的逻辑，只看解码和渲染的过程。

待解码数据送入解码器：

![](https://imgs.piasy.com/2018-05-23-ios_rtp_stream_to_decoder.png)

从 VideoReceiveStream 到 VideoRecenver，再到 VCMGenericDecoder，最后到 ObjCVideoDecoder 和 RTCVideoDecoderH264，都没有视频数据的处理，只是简单的透传、解码，以及打上时间戳（VideoReceiveStream, VCMGenericDecoder）。

这条路径上的各个类名，我们都可以在发送端找到对应的类名：

+ VideoReceiveStream - VideoSendStream
+ VideoRecenver - VideoSender
+ VCMGenericDecoder - VCMGenericEncoder
+ ObjCVideoDecoder - ObjCVideoEncoder
+ RTCVideoDecoderH264 - RTCVideoEecoderH264

解码后数据进行渲染：

![](https://imgs.piasy.com/2018-05-02-ios_rtp_decoder_to_renderer.png)

在这个过程中最主要的逻辑就是时间戳的处理，限于篇幅我们这里不做展开，留待以后探究。

另外需要注意的是，发送端可能会选择不对视频做旋转操作，而是把旋转角度发过来，那么接收端就需要在渲染的时候进行旋转了。

## 跨平台视频处理模块的架构

通过这次的源码分析，我们可以看到 WebRTC 是如何设计 VCM (Video Coding Module) 这个跨平台视频处理模块的结构的。

首先这个模块包括采集、编码、jitter buffer、解码、渲染等功能，还有它们的线程、队列管理，其中采集、（硬件）编解码、渲染都是平台相关的，其他模块以及对平台相关模块的调用和管理，都是平台无关的。

那我们就通过接口 + 回调的形式，将平台相关的模块抽离出去，对它们的操作，VCM 就调用接口，而它们需要把数据交给 VCM，或者需要调用 VCM 的功能时，就通过回调的形式把控制权还给 VCM。

至于怎么把它们关联起来，就需要在构造 VCM 的时候，把接口实现注入进去了，或者利用工厂模式，把 factory 注入进去，VCM 调用 factory 来创建接口的平台相关实现，显然 WebRTC 是用了工厂模式。

这个套路其实我在[移动客户端跨平台开发方案探索](/2017/12/16/Mobile-Client-Cross-Platform-Development/index.html)中提到过，看，WebRTC 也是这样的套路 :)

## 总结

好了，WebRTC 真正 native 层的初体验到这里就结束了，其实并没有涵盖多少内容，主要是都要展开内容就太多了，别急，我们一步一步来。

另外，在这次的分析过程中，我还梳理了一下 WebRTC 一些核心类的关系，以及音视频数据、DataChannel 的流程，[画了一张超大的 svg](https://github.com/Piasy/HackWebRTC/blob/master/WebRTC_classes_23261.svg)，也算是为其他模块的分析打下了一个底子，欢迎大家 star 支持 :)

## 附录：部分类实现代码路径

``` bash
sdk/objc/Framework/Classes/PeerConnection/RTCVideoSource.mm
sdk/objc/Framework/Native/src/objc_video_track_source.mm
media/base/adaptedvideotracksource.cc
media/base/videobroadcaster.cc
media/engine/webrtcvideoengine.cc
call/video_send_stream.cc
video/video_stream_encoder.cc
modules/video_coding/video_coding_impl.cc
modules/video_coding/video_sender.cc
modules/video_coding/generic_encoder.cc
```
