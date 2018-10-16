---
layout: post
title: WebRTC Native 源码导读（十四）：API 概览
tags:
    - 实时多媒体
    - WebRTC
---

第一次看到 WebRTC 这个词是 15 年[在一期 Android Weekly 中](http://androidweekly.net/issues/issue-155)，但当时完全看不懂它在讲什么，也就没有深究。两年后，我开始搞起 WebRTC，并整理出了一套[开箱即用的 WebRTC 开发环境](/2017/06/17/out-of-the-box-webrtc-dev-env/)，距今又过了一年多。

按常理，这篇文章要介绍的内容本应最先呈现，但我搞 WebRTC 的路线略微反常，由于没有实际使用的需求，所以我先是研究了不少模块的实现原理。但随着接触到更多内容，我越来越意识到，如果没有一个清晰的全局观，那效率将会越来越低，因此也就有了这篇文章。

_鉴于目前我对 WebRTC 的了解仍然有限，所以本文很可能仍有遗漏，或错误，但我会持续更新本文_。

## SDP 基本结构

首先我们搞清楚 SDP 的基本结构。

总体来说，WebRTC 的 SDP 分为几个部分：

+ session metadata: v=, o=, s=, t=
+ network description: c=, a=candidate
+ stream description: m=, a=rtpmap, a=fmtp, a=sendrecv ...
+ security description: a=crypto, a=ice-frag, a=ice-pwd, a=fingerprint
+ QoS, grouping description: a=rtcp-fb, a=group, a=rtcpmux

m= 开头的一段叫做 m section，这一行叫 m line，里面有很多 a line 来描述这种 media 的各种属性。我们称一种媒体数据为一种 media，每种 media 在 SDP 里都有 m section。

WebRTC 的 SDP 有三种类型：

+ offer: 发起方提供的自己对本次通话的描述；
+ answer: 其他方收到 offer 后，给出的回应；
+ pranswer: provisional answer，非最终 answer，之后可能被 pranswer 或 answer 更新；

## Plan B v.s. Unified Plan

说到 SDP，就不得不提它的两种 Plan，它们是表达传输多路媒体流时的两种 SDP 格式。多路媒体流的例子有：录屏 + 相机，或多个相机（视角）。

Plan B 是 SDP 里同类型的媒体流只有一个 m line，同类型的多个媒体流之间通过 msid 区分，而 Unified Plan 则是每个媒体流都有一个 m line，因此如果有两路视频，那就会有两个 video m line。

WebRTC 标准采纳的是 Unified Plan，WebRTC 代码也已支持，所以我们就只关注 Unified Plan 的 API。

参考：[Plan B](https://webrtcglossary.com/plan-b/), [Unified Plan](https://webrtcglossary.com/unified-plan/), [Unified Plan vs Plan B](https://www.douban.com/note/658626071/)。

Plan B 在 WebRTC 源码里对应的是 PC 的 Stream/Sender/Receiver API，Unified Plan 对应的是 Track/Transceiver API。

## Capturer/Source/Track/Sink/Transceiver

接着我们梳理一下媒体数据交换过程中的几个关键概念：

+ Capturer: 负责数据采集，只有视频才有这一层抽象，它有多种实现，相机采集（Android 还有 Camera1/Camera2 两套）、录屏采集、视频文件采集等；
+ Source: 数据源；
+ Track: 媒体数据交换的载体，发送端把本地的 Track 发送给远程的接收端，每个 Track 都有自己的 track id，多个关联的 Track 有一个相同的 stream id；
+ Sink: Track 数据的消费者，只有视频才有这一层封装，发送端视频的本地预览、接收端收到远程视频后的渲染，都由 Sink 负责；
+ Transceiver: 负责收发媒体数据；

以视频为例，数据由发送端的 Capturer 采集，交给 Source，再交给本地的 Track，然后兵分两路，一路由本地 Sink 进行预览，一路由 Transceiver 发送给接收端；接收端 Track 则把数据交给 Sink 渲染。

Capturer 的创建和销毁完全由 APP 层负责，只需要把它和 Source 关联起来即可；创建 Source 需要调用 PC Factory 接口，创建 Track 也是，并且需要提供 Source 参数；Sink 的创建和销毁也由 APP 层负责，只需要把它们添加到 Track 里即可；创建 Transceiver 则需要调用 PC 接口。

好了，接下来我们就看看 PC Factory 和 PC 的接口。

## PeerConnectionFactory 接口

### CreatePeerConnectionFactory

默认的编译选项里，`rtc_use_builtin_sw_codecs = false`，因此 `USE_BUILTIN_SW_CODECS` 未被定义，`CreatePeerConnectionFactory` 只有一个重载版本：接收三个 thread、adm、audio/video encoder/decoder factory、AudioMixer 和 AudioProcessing。

### CreatePeerConnection

创建 PC 对象，接收 RTCConfiguration 和 PeerConnectionDependencies，前者用来容纳各种配置，后者则用来容纳各种可定制的接口实现，例如 PortAllocator, AsyncResolverFactory, RTCCertificateGeneratorInterface, SSLCertificateVerifier。

目前 Android/iOS 对 dependencies 的支持还未跟上，虽然这种高级用法的用户也不怕在 native 层自己做封装，但就又得重新造一遍 WebRTC Java/ObjC 代码里的轮子了。

### CreateAudio/VideoSource/Track

这就是前面我们提到的创建 Audio/Video Source/Track 的接口了。

## PeerConnection 接口

准备工作相关：

+ AddTrack: 添加要发送的 track；
+ AddTransceiver: 添加 transceiver；
+ CreateDataChannel: 添加 DataChannel；
+ RemoveTrack: 移除 track；
+ GetTransceivers: 获取所有的 transceiver；

建立 P2P 连接相关：

+ CreateOffer: 创建 offer；
+ CreateAnswer: 创建 answer；
+ SetLocalDescription: 设置本地 SDP；
+ SetRemoteDescription: 设置远端 SDP；
+ AddIceCandidate: 添加 ICE candidate；
+ RemoveIceCandidates: 移除 ICE candidate；

注意：CreateOffer/CreateAnswer 时传入的 RTCOfferAnswerOptions 里，有 `offer_to_receive_X` 字段，它们是为了兼容 Plan B 语义的，一旦设置，即便没有 AddTrack，SDP 里也会包含 audio/video 的 m line。使用 Unified Plan 时，不应设置这两个字段，而应提前调用 AddTrack/AddTransceiver/CreateDataChannel，来表明自己是否需要 audio/video/data。

其他接口：

+ GetStats: 获取统计数据；
+ SetBitrate: 设置这个 PC 总的发送码率，包括初始码率、最小码率、最大码率；
+ SetBitrateAllocationStrategy: 设置自定义码率分配策略，可以通过这个接口实现针对每个 track 的码率分配策略；

注意：`SetBitrateAllocationStrategy` 在 Android 和 iOS 平台都没有暴露出来，Android 暴露了 SetBitrate 接口，iOS 则没有，不过可以通过 `RTCRtpSender setParameters` 限制编码器的输出码率。

回调接口 PeerConnectionObserver：

+ OnSignalingChange: 产生/设置 SDP 后，会触发 signaling state 变化，常见的变化是 `stable -> have-local-offer -> stable` 或 `stable -> have-remote-offer -> stable`，具体可以查看 [SPEC 4.3 State Definitions](https://w3c.github.io/webrtc-pc/#state-definitions)；
+ OnRenegotiationNeeded: 需要重新协商（重新建立 P2P 连接）时回调，例如 ICE restart 时会回调；
+ OnIceGatheringChange: ICE candidate 收集状态变化后回调；
+ OnIceConnectionChange: ICE 连接状态变化后回调；
+ OnIceCandidate: 收集到本地 ICE candidate 后回调；
+ OnIceCandidatesRemoved: 本地 ICE candidate 被移除后回调；
+ OnTrack: 调用 SetRemoteDescription 后，如果 SDP 表明将会创建接收用的 transceiver，就会回调这个接口；
+ OnRemoveTrack: 当确定一个 track 不再接收媒体数据后，会回调这个接口，track 不会移除，但 transceiver 的 recv 方向将会被去掉；

接下来我们重点看一下 transceiver。

## RtpTransceiver

SDP 的 m section 里有一行 `a=mid:`，定义了这种 media 的 id，叫 mid，例如下面这对 offer 和 answer:

```
# offer

...
a=group:BUNDLE 0 1 2
...
m=video 9 UDP/TLS/RTP/SAVPF 100 96 97 98 99 101 127 124 125
...
a=mid:0
...
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
...
a=mid:1
...
m=application 9 DTLS/SCTP 5000
...
a=mid:2
...


# answer

...
a=group:BUNDLE 0 1 2
...
m=video 9 UDP/TLS/RTP/SAVPF 100 96 97 98 99 101 127 124
...
a=mid:0
...
m=audio 9 UDP/TLS/RTP/SAVPF 103 104 9 102 0 8 106 105 13 110 112 113 126
...
a=mid:1
...
m=application 9 DTLS/SCTP 5000
...
a=mid:2
...
```

其中有三种 media: video, audio, application，mid 依次为 0, 1, 2。application 是 DataChannel 的 media type。

我们注意到，offer 和 answer 里同一种 media 的 mid 是相同的，也就是说，对某一端来说，他收发的同一种媒体数据，mid 是相同的。

在 WebRTC 标准里，transceiver 表示的就是收发相同 mid 的 sender 和 receiver 的一个组合体，其中会有 media type, mid, direction, sender, receiver 等字段。其中 direction 有几种取值：kSendRecv, kSendOnly, kRecvOnly, kInactive。

AddTrack 时我们 add 的是本地的 track，即要发送的数据流，首次 AddTrack 时，会创建 transceiver，默认其 direction 是 kSendRecv。尽管在 CreateOffer 时我们可以通过设置 `RTCOfferAnswerOptions` 的 `offer_to_receive_X` 字段来控制是否 receive，但这两个字段是 legacy 字段，我们应该尽量避免。那如何控制 transceiver 的方向呢？我们可以使用 AddTransceiver 接口。

如果想要创建 kSendOnly 的 transceiver，可以传入 track，并在 `RtpTransceiverInit` 中设置 direction 为 kSendOnly；或者只传入 media type 和 init 结构体，稍后再 AddTrack。如果想要创建 kRecvOnly 的 transceiver，可以只传入 media type 和 init 结构体，并且不 AddTrack。

transceiver 何时与 SDP 里的 m section 关联呢？offer 端在创建 offer 时，[会根据已有的 transceiver 创建 m section，并记下每个 transceiver 在 SDP 里对应的 m section 的 index 值，以便在 SetLocalDescription 时，可以为 transceiver 设置正确的 mid](https://webrtc.googlesource.com/src/+/c84cd950b788f22a68a31081f0c85173d260bd32/pc/peerconnection.cc#3835)；answer 端在 SetRemoteDescription（offer 端发来的 offer）时，[如果 offer 里的 m section 有 recv 方向，那就按 media type 来查找已有的 transceiver，如果能找到就可以将其关联起来，否则就创建一个 kRecvOnly 的 transceiver](https://webrtc.googlesource.com/src/+/c84cd950b788f22a68a31081f0c85173d260bd32/pc/peerconnection.cc#2748)（因为 offer 只有可能是 kSendOnly 了，不发也不收的 media，不会出现在 SDP 里，那对此 offer 的回应也就只能是 kRecvOnly 了）。

总结一下，无论是 offer 端还是 answer 端，需要发送的 media，才提前添加好有 send 方向的 transceiver，仅接收的 media，无需提前添加 transceiver（提前添加了也不会被使用）。

## 附录一：SDP 部分细节

+ m line 里会指明传输协议，例如 UDP/TLS/RTP/SAVPF，最后的 SAVPF 还有其他几种值：AVP, SAVP, AVPF, SAVPF
  - AVP 意为 AV profile
  - S 意为 secure
  - F 意为 feedback
+ rtpmap 是描述 codec 的，但有特殊的 rtx codec，其实不是 codec，例如 rtx；
+ fmtp 补充描述 codec 的参数，format parameters
  - max-fr: maximum framerate
  - profile-level-id: H.264 的 profile level id
+ rtx 描述重传策略，由 rtpmap 指明，它的参数由 fmtp 描述
  - apt: associated payload type，指明所描述的 stream；
  - rtx-time: rtp 包在缓冲区保留时间；
+ rtcp-fb: RTCP 反馈机制
  - offer 里面列出一些反馈机制，answer 里应移除不理解、不支持的机制，但不能修改；
  - ack rpsi/app
  - nack pli/sli/rpsi/app
  - rpsi: reference picture selection indication
  - app: app 层反馈机制
  - pli: picture loss indication，表明收流端丢失了一幅图像的一些数据，发送端可能会发送一个 I 帧（类似于 FIR），但要考虑拥塞控制
  - sli: slice loss indication
  - ccm fir: codec control message, full intra refresh
+ fec 类似于 rtx，也由 rtpmap 指明，它的参数由 fmtp 指明；
