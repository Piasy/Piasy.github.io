---
layout: post
title: WebRTC-Android 源码导读（五）：P2P 连接过程和 DataChannel 使用
tags:
    - P2P
    - 流媒体
    - WebRTC
---

前面四篇里，我们分别分析了 WebRTC Android 的视频采集、视频渲染和视频硬编码，最后把相关代码剥离出来形成了一个独立的模块：VideoCRE，并对其进行了极大地内存抖动优化。从本篇起，我们将迈入新的领域：网络传输。首先我们看看 P2P 连接的建立过程，以及 DataChannel 的使用，最终我们会利用 DataChannel 实现一个 P2P 的文字聊天功能。

前面四篇没看过的朋友，下面是链接：

+ [相机采集实现分析](/2017/07/24/WebRTC-Android-Camera-Capture/)
+ [预览实现分析](/2017/07/26/WebRTC-Android-Render-Video/)
+ [视频硬编码实现分析](/2017/08/08/WebRTC-Android-HW-Encode-Video/)
+ [VideoCRE 与内存抖动优化](/2017/08/11/VideoCRE-and-Memory-Churn-Opt/)

## P2P 连接过程

首先总结一下 WebRTC 建立 P2P 连接的过程（就是喜欢手稿）：

![](https://imgs.piasy.com/2017-08-16-p2p_connect_procedure2.jpg)

Offer Answer？SDP？ICE candidate？？？别急，我们先来一个简单的名词解释。

### SDP

SDP 全称 Session Description Protocol，顾名思义，它是一种描述会话（Session）的协议。一次电话会议，一次网络电话，一次视频流传输等等，都是一次会话。那会话需要哪些描述呢？最基础的有多媒体数据格式和网络传输地址，当然还包括很多其他的配置信息。[^sdp]

为什么需要描述会话？因为参与会话的各个成员能力不对等。大家可能会想到使用所有人都支持的媒体格式，我们暂且不考虑这样的格式是否存在，我们思考另一个问题：如果参与本次会话的成员都比较牛，可以支持更高质量的通话，那使用通用的、普通质量的格式，是不是很亏？既然无法使用固定的配置，那对会话的描述就很有必要了。

最后，一次会话用什么配置，也不是由某一个人说了算，必须所有人的意见达成一致，这样才能保证所有人都能参与会话。那这就涉及到一个协商的过程了，会话发起者先提出一些建议（offer），其他人参与者再根据 offer 给出自己的选择（answer），最终意见达成一致后，才能开始会话。[^offer-answer]

当然，上面只是对 SDP 以及协商过程的一个极简理解，详细的定义还得查阅[相关的 RFC 文档](#section)。

让我们回到 P2P 连接的建立过程，offer 和 answer 其实都是 SDP，而 local/remote 则是相对的，offer 是会话发起者的 local SDP，是会话加入者的 remote SDP，answer 则是会话发起者的 remote SDP，是会话加入者的 local SDP。

SDP 实际上就是一个字符串，它的具体格式定义，可以参考 [RFC 文档](https://tools.ietf.org/html/rfc4566)。它的拼接过程，native 和 Java 代码都有分布，native 代码调用栈还比较深，这里就不展开了，`createOffer` 主要逻辑就是根据创建 `PeerConnection` 对象时指定的 `MediaConstraints`，以及在 `createOffer` 调用前添加的 `VideoTrack`/`AudioTrack`/`DataChannel` 情况，拼出初始 SDP，最后在 `PeerConnectionClient.SDPObserver#onCreateSuccess` 中会添加 codec 相关的值。`createAnswer` 则还会参考 offer SDP 的值。

### ICE

ICE 是用于 UDP 媒体传输的 NAT 穿透协议（适当扩展也能支持 TCP 协议），是对 Offer/Answer 模型的扩展，它会利用 STUN、TURN 协议完成工作。ICE 会在 SDP 中增加传输地址记录值（IP + port + 协议），然后对其进行连通性测试，测试通过之后就可以用于发送媒体数据了。[^ice]

#### candidate

每个传输地址记录值都叫做一个 candidate，candidate 可能有三种：

+ 客户端从本机网络接口上获取的地址（host）；
+ STUN server 看到的该客户端的地址（server reflexive）；
+ TURN server 为该客户端分配的中继地址（relayed）；

两个客户端上述 candidate 的任意组合也许都能连通，但实际上很多组合都不可用，例如 L R 两个客户端处于两个不同的 NAT 网络后面时，网络接口地址都是内网地址，显然无法连通。而 ICE 的任务，就是找出哪些组合可以连通。怎么找？也没有什么黑科技，就是逐个尝试，只不过是有条理地、按照某种顺序去尝试，而不是一通乱搞。

网络接口地址对应的端口号是客户端自己分配的，如果有多个网络接口地址，那就都要带着（看，这里就不是瞎猜哪个地址可用了）。TURN server 可以同时取得 reflexive 和 relayed candidate，而 STUN server 则只能取得 reflexive candidate（这下我就清楚 [coturn](https://github.com/coturn/coturn) 到底是 STUN server 还是 TURN server 了）。

三种 candidate 的关系如下图（_RFC 画图的技术也是比较高超的_）：

![](https://imgs.piasy.com/2017-08-20-ice_candidates_relationship.png)

#### 连通性检查

candidate 收集完毕后，双方的 candidate 两两配对，然后分三步对 candidate 组合进行连通性检查：

+ 把 candidates 组合按优先级排序；
+ 按顺序发送检查请求（STUN Binding request），源地址是 candidate 组合的本地 candidate，目的地址是对方 candidate；
+ 收到对方的检查请求后发出响应（STUN Binding response）；

每次检查实际上是一个四步握手的过程：

![](https://imgs.piasy.com/2017-08-20-ice_4_way_handshake.png)

STUN 请求和 RTP/RTCP 传输数据使用的是完全一样的地址和端口，解多路复用并不是 ICE 的任务，而是 RTP/RTCP 的任务。

客户端收到的 STUN Binding respose 中也会携带对方的公网地址，如果这个地址和发送请求的 request 地址不一致，那 response 里的地址也会作为一个新的 candidate（peer reflexive），参与到连通性检查中。

如果客户端收到了对方的检查请求，除了发送响应外，也会立即对这个 candidate 组合进行检查，以加快完成一次成功的连通性检查。

#### candidate 排序

每个客户端会为自己的 candidate 设置权值，双方 candidate 权值之和将作为组合的权值，用于排序。求和的方式确保了双方排序结果的一致性，这个一致性至关重要，因为通常 NAT 都不会允许外部主机的数据包从某个端口进入内网，除非这个端口有数据包发往过这个主机，因此只有双方都发送了检查请求，数据包才可能通过 NAT。

权值的确定，RFC 里面只说明了基本原则：直接的连接比间接的连接要好。但具体如何设置，并没有具体说明。

## candidate 收集代码

+ `Port::AddAddress` -> `BasicPortAllocatorSession::OnCandidateReady`
+ `AllocationSequence::EnableProtocol` -> `BasicPortAllocatorSession::OnProtocolEnabled`

+ `P2PTransportChannel::MaybeStartGathering`
+ `P2PTransportChannel::OnCandidatesReady@webrtc/p2p/base/p2ptransportchannel.cc`，发 signal
+ `IceTransportInternal->SignalCandidateGathered@webrtc/p2p/base/icetransportinternal.h`，signal
+ `TransportController::OnChannelCandidateGathered_n@webrtc/p2p/base/transportcontroller.cc`，slot，收 signal，发 event
+ `TransportController::OnMessage@webrtc/p2p/base/transportcontroller.cc`，处理 event，发 signal
+ `TransportController->SignalCandidatesGathered@webrtc/p2p/base/transportcontroller.h`，signal
+ `WebRtcSession::OnTransportControllerCandidatesGathered@webrtc/pc/webrtcsession.cc`，slot，收 signal
+ `PeerConnection::OnIceCandidate@webrtc/pc/peerconnection.cc`
+ `PeerConnectionObserverJni::OnIceCandidate@webrtc/sdk/android/src/jni/pc/peerconnectionobserver_jni.cc`
+ `PeerConnectionClient.PCObserver#onIceCandidate`

## prebuilt library

WebRTC 官方团队并没有发布正式版的包，但他们的 CI 系统会自动 build 每个 commit，并生成一个 aar，我们可以下载使用，[下载地址](https://build.chromium.org/p/client.webrtc.fyi/builders/Android%20Archive)，打开某次 build 后，搜索“gsutil.upload”，就能下载 aar 了。

## 脚注

[^sdp]: [SDP: Session Description Protocol](https://tools.ietf.org/html/rfc4566)
[^offer-answer]: [An Offer/Answer Model with the Session Description Protocol (SDP)](https://tools.ietf.org/html/rfc3264)
[^ice]: [Interactive Connectivity Establishment (ICE): A Protocol for Network Address Translator (NAT) Traversal for Offer/Answer Protocols](https://tools.ietf.org/html/rfc5245)
