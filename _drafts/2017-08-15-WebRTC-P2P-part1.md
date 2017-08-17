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

Offer Answer？SDP？Ice candidate？？？别急，我们先来一个简单的名词解释。

### SDP

SDP 全称 Session Description Protocol，顾名思义，它是一种描述会话（Session）的协议。一次电话会议，一次网络电话，一次视频流传输等等，都是一次会话。那会话需要哪些描述呢？最基础的有多媒体数据格式和网络传输地址，当然还包括很多其他的配置信息。

为什么需要描述会话？因为参与会话的各个成员能力不对等。大家可能会想到使用所有人都支持的媒体格式，我们暂且不考虑这样的格式是否存在，我们思考另一个问题：如果参与本次会话的成员都比较牛，可以支持更高质量的通话，那使用通用的、普通质量的格式，是不是很亏？既然无法使用固定的配置，那对会话的描述就很有必要了。

最后，一次会话用什么配置，也不是由某个人说了算，必须所有人的意见达成一致，这样才能保证所有人都能参与会话。那这就涉及到一个协商的过程了，会话发起者先提出一些建议（offer），其他人参与者再表示同意或者开始讨价还价（answer），最终意见达成一致后，才开始会话。

当然，上面只是对 SDP 一个极简的理解，SDP 的详细定义，还得查阅[IETF 的 RFC 文档](https://tools.ietf.org/html/draft-ietf-rtcweb-sdp-06)。

让我们回到 P2P 连接的建立过程，offer 和 answer 其实都是 SDP，而 local/remote 则是相对的，offer 是会话发起者的 local SDP，是会话加入者的 remote SDP，answer 则是会话发起者的 remote SDP，是会话加入者的 local SDP。

而 SDP 实际上就是一个字符串，它的具体格式定义，可以参考 [RFC 文档](https://tools.ietf.org/html/rfc4566)。它的拼接过程，native 和 Java 代码都有分布，native 代码调用栈还比较深，这里就不展开了，`createOffer` 主要逻辑就是根据创建 `PeerConnection` 对象时指定的 `MediaConstraints`，以及在 `createOffer` 调用前添加的 `VideoTrack`/`AudioTrack`/`DataChannel` 情况，拼出初始 SDP，最后在 `PeerConnectionClient.SDPObserver#onCreateSuccess` 中会添加 codec 相关的值。`createAnswer` 则还会参考 offer SDP 的值。

## prebuilt library

WebRTC 官方团队并没有发布正式版的包，但他们的 CI 系统会自动 build 每个 commit，并生成一个 aar，我们可以下载使用，[下载地址](https://build.chromium.org/p/client.webrtc.fyi/builders/Android%20Archive)，打开某次 build 后，搜索“gsutil.upload”，就能下载 aar 了。
