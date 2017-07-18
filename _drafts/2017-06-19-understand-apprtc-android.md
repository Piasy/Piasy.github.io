---
layout: post
title: AppRTC-Android 源码导读
tags:
    - 流媒体
    - WebRTC
---

前面分享了一套[开箱即用的 WebRTC 开发环境](/2017/06/17/out-of-the-box-webrtc-dev-env/)，希望能给对 WebRTC 感兴趣的朋友带来帮助。不过有了开发环境只是迈出了万里长征第一步，后面的事情还得仔细研读源码才行，所以这里给大家先带来 WebRTC 的安卓 demo 工程—— AppRTC-Android 的源码导读。（[九个月前说好的拆 Dagger2](/2016/09/15/Understand-RxJava/) 看来又要等等了，海涵海涵...）

## 概览

让我们先搞清楚 WebRTC 的几个核心类以及它们之间的关系，[首先是三大核心类](https://www.youtube.com/watch?v=p2HzZkd2A40)：

+ `MediaStream`，获取 Audio/Video 数据；
+ `PeerConnection`，交换 Audio/Video 数据；
+ `DataChannel`，收发任意数据；

MediaStream 中包含多个 Track（AudioTrack，VideoTrack），用于采集数据，PeerConnection 则包含多个 Stream。【本地数据叫采集，是 Stream 的事，那远端的数据呢？在本地也是一个 Stream 吗？】而数据则通过 DataChannel 收发。【音视频数据也通过 DataChannel 收发？】

WebRTC 的代码量不小，一次性看明白不太现实，在本文中，我将试图搞清楚三个问题：

1. 客户端之间如何建立连接？
2. 客户端之间如何协商音视频相关的配置？
3. 音视频数据的采集、预览、编码、传输、解码、渲染完整流程。

当然，限于篇幅和实际需求，细节之处我不会展开，但把握住了主线之后，日后再有具体需求，我们就可以快速找到相关代码了。

## 建立连接

+ 首先是房间创建者（`initiator`）向 Room Server 发起请求，获取房间配置信息，各种 URL 信息；
+ 创建者获取到相关信息之后，创建 offer sdp，发送给 Room Server；
+ 接着是房间加入者向 Room Server 发起请求，除了获取房间配置信息，还会获取到 offer sdp；
+ 加入者创建 answer sdp，利用 Signal Server（长连接）发送给创建者，同时开始建立 P2P 连接；
+ 创建者收到 answer sdp 之后，也开始建立 P2P 连接；

下面是一张简单的示意图（懒得画高大上的流程图了）：

![](https://imgs.piasy.com/2017-07-05-webrtc_connect.jpeg)

下面是创建者关键的代码位置：

+ 入口代码是 `CallActivity#startCall`，调用 `appRtcClient.connectToRoom`，这里我们配置了 server url，所以 `appRtcClient` 是 `WebSocketRTCClient`；
+ `WebSocketRTCClient#connectToRoomInternal`，我们利用 `RoomParametersFetcher` 获取配置信息；
+ `RoomParametersFetcher#makeRequest`，我们利用 `AsyncHttpURLConnection` 发出 POST 请求，请求返回后，我们在 `roomHttpResponseParse` 中解析数据，解析完毕后，回调到 `WebSocketRTCClient`；
+ `WebSocketRTCClient#signalingParametersReady`，其中保存了相关信息后，回调到 `CallActivity`；
+ `CallActivity#onConnectedToRoomInternal`，在这里我们就要区分自己的角色了，我们是创建者，则开始创建 offer sdp；

涉及到 sdp 之后，就从 app 进入到 WebRTC 的库里面了，下面是创建 offer 在库里面的调用流程：

+ `PeerConnection#createOffer`
+ 【native 调用】
+ `SdpObserver#onCreateSuccess`，也就是 `PeerConnectionClient.SDPObserver#onCreateSuccess`，其中我们会设置 local sdp（offer）；
+ `PeerConnection#setLocalDescription`
+ 【native 调用】
+ `SdpObserver#onSetSuccess`，也就是 `PeerConnectionClient.SDPObserver#onSetSuccess`，此时我们还没有 remote sdp（answer），则我们会回调到 `CallActivity`；
+ `CallActivity#onLocalDescription`，我们是创建者，则调用 `appRtcClient.sendOfferSdp` 发送 offer sdp，然后等待加入者的 answer；

接下来是加入者关键代码的位置：

+ 从入口代码到获取房间配置，代码路径都和创建者一致，但在 `CallActivity#onConnectedToRoomInternal` 中，我们已经获取到了 remote sdp（offer），所以我们先设置 remote sdp；
+ `PeerConnection#setRemoteDescription`
+ 【native 调用】
+ `SdpObserver#onSetSuccess`，也就是 `PeerConnectionClient.SDPObserver#onSetSuccess`，此时我们还没有 local sdp（answer），但我们会马上创建 answer sdp；
+ `PeerConnection#createAnswer`
+ 【native 调用】
+ `SdpObserver#onCreateSuccess`，也就是 `PeerConnectionClient.SDPObserver#onCreateSuccess`，其中我们会设置 local sdp（answer）；
+ `PeerConnection#setLocalDescription`
+ 【native 调用】
+ `SdpObserver#onSetSuccess`，也就是 `PeerConnectionClient.SDPObserver#onSetSuccess`，此时我们已经有了 local sdp（answer），则我们会回调到 `CallActivity`，并且开始添加 ICE candidate；
+ 最后在 `CallActivity#onLocalDescription` 中，我们调用 `appRtcClient.sendAnswerSdp` 把 answer sdp 发送给创建者；

接下来是创建者收到 answer 之后的处理流程：

+ `WebSocketRTCClient#onWebSocketMessage` 中，我们收到 answer 之后，会回调到 `CallActivity`；
+ `CallActivity#onRemoteDescription` 中我们设置 remote sdp；
+ `PeerConnection#setRemoteDescription`
+ 【native 调用】
+ `SdpObserver#onSetSuccess`，也就是 `PeerConnectionClient.SDPObserver#onSetSuccess`，此时我们已经有了 remote sdp（answer），则开始添加 ICE candidate；

上面 `setLocalDescription`/`setRemoteDescription` 有一段较长的调用链，它们会分别回调到 `onCreateSuccess`/`onSetSuccess`。

【最后，添加 ICE candidate 会开始建立 P2P 连接？】

+ `PeerConnection#addIceCandidate`

## SDP

## 音视频全流程


