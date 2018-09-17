---
layout: post
title: WebRTC Native 源码导读（）：PC/Factory API
tags:
    - 实时多媒体
    - WebRTC
---

## Plan B v.s. Unified Plan

它们是表达传输多路媒体流时的两种 SDP 格式。多路媒体流的例子有：录屏 + 相机，或多个相机（视角）。

Plan B 是 SDP 里同类型的媒体流只有一个 m line，同类型的多个媒体流之间通过 msid 区分，而 Unified Plan 则是每个媒体流都有一个 m line，因此如果有两路视频，那就会有两个 video m line。

WebRTC 标准采纳的是 Unified Plan，WebRTC 代码也已支持，所以我们就只关注 Unified Plan 的 API。

参考：[Plan B](https://webrtcglossary.com/plan-b/), [Unified Plan](https://webrtcglossary.com/unified-plan/), [Unified Plan vs Plan B](https://www.douban.com/note/658626071/)。

Plan B 在 WebRTC 源码里对应的是 PC 的 Stream/Sender/Receiver API，Unified Plan 对应的是 Track/Transceiver API。

## PeerConnectionFactory

默认的编译选项里，`rtc_use_builtin_sw_codecs = false`，因此 `USE_BUILTIN_SW_CODECS` 未被定义，`CreatePeerConnectionFactory` 只有一个重载版本：接收三个 thread、adm、audio/video encoder/decoder factory、AudioMixer 和 AudioProcessing。

在 Android/iOS 平台，只用到了以下接口：

### CreatePeerConnection

创建 PC 对象，接收 RTCConfiguration 和 PeerConnectionDependencies，前者用来容纳各种配置，后者则用来容纳各种可定制的接口实现，例如 PortAllocator, AsyncResolverFactory, RTCCertificateGeneratorInterface, SSLCertificateVerifier。

目前 Android/iOS 对 dependencies 的支持还未跟上，虽然这种高级用法的用户也不怕在 native 层自己做封装，但就又得重新造一遍 WebRTC Java/ObjC 代码里的轮子了。

### CreateAudioSource

### CreateAudioTrack

### CreateVideoTrack

## PeerConnection

AddTrack, RemoveTrack, AddTransceiver, GetTransceivers, GetStats, CreateDataChannel

CreateOffer, CreateAnswer, SetLocalDescription, SetRemoteDescription, AddIceCandidate, RemoveIceCandidates

要想 SDP 里包含 DataChannel 的 m line，就必须在 CreateOffer 之前调用 CreateDataChannel。

CreateOffer/CreateAnswer 时传入的 RTCOfferAnswerOptions 里，有 `offer_to_receive_X` 字段，它们是为了兼容 Plan B 语义的，一旦设置，即便没有 AddTrack，SDP 里也会包含 audio/video 的 m line。使用 Unified Plan 时，不应设置这两个字段，而应提前调用 AddTransceiver，来表明自己是否需要 audio/video。

SetBitrate, SetBitrateAllocationStrategy