---
layout: post
title: 在 WebRTC 项目外单步调试 WebRTC.framework 里的代码
tags:
    - 实时多媒体
    - WebRTC
---

两年半以前我分享了 [macOS 下单步调试 WebRTC Android & iOS](/2018/08/14/build-webrtc/index.html)，说的都是在 WebRTC 项目内进行单步调试，但实际情况中我们显然是要在自己的项目里使用 WebRTC 库，如果这时我们希望能单步调试 WebRTC 库的代码，应该怎么办呢？

今天我再分享下在自己的项目里调试 iOS WebRTC.framework 的方法。

_我这边的环境：macOS 10.15.7 (19H2), Xcode 11.7 (11E801a), WebRTC 是基于 [#30987](https://webrtc.googlesource.com/src/+/04c1b445019e10e54b96f70403d25cc54215faf3) 提交_。

## 项目说明

这里我以[基于 Kotlin multiplatform 的多平台 WebRTC SDK](/2019/12/05/Kmpp-WebRTC-SDK/index.html) 的实际项目为例，手把手教会大家在自己的项目里调试 iOS WebRTC.framework 的方法。

AvConf 的 Kotlin 代码，在 iOS 上是编译为了静态库，然后再加上一层包装代码，使得对外 API 是正宗的 Objective-C 风格，一起编译为 framework，iOS demo 项目直接引用编译好的 framework，不依赖库工程。所以这里我会同时示范调试 AvConf.framework 和 WebRTC.framework。

## 项目设置

首先 iOSExample 工程原始设置如下图示：

![](https://imgs.piasy.com/2021-03-07-app-deps-remove.png)

可以看到工程引用的是 .xcframework（Apple 新的多架构库的格式，使用时和 .framework 没太多区别）。

那么第一步就是从项目依赖中删除这俩 .xcframework 了。

接着把 AvConf 和 WebRTC 的 .xcodeproj 拖进项目里：

![](https://imgs.piasy.com/2021-03-07-app-project-drag-in.png)

然后在项目 Build Phases - Dependencies 里添加依赖 target：

![](https://imgs.piasy.com/2021-03-07-app-deps-add.png)

![](https://imgs.piasy.com/2021-03-07-app-deps-add-avconf.png)

![](https://imgs.piasy.com/2021-03-07-app-deps-add-webrtc.png)

添加完毕后的效果：

![](https://imgs.piasy.com/2021-03-07-app-deps-add-2.png)

接着在项目 General - Frameworks, Libraries, and Embedded content 里添加依赖 target 编译出来的库：

![](https://imgs.piasy.com/2021-03-07-app-deps-add-avconf-2.png)

_WebRTC 的添加方式一样，但因为不太好截图，这里就不截了_。

添加完毕后的效果：

![](https://imgs.piasy.com/2021-03-07-app-deps-add-3.png)

最后是添加断点：展开拖进去的 .xcodeproj，找到源码位置，添加断点即可，就和之前在自己项目的源码里添加断点一样，同样这里也只展示一下在 AvConf 里添加断点的操作截图。

![](https://imgs.piasy.com/2021-03-07-app-breakpoint-add-avconf.png)

## Let's roll

现在让我们运行项目，加入房间，就能触发断点啦！

![](https://imgs.piasy.com/2021-03-07-app-breakpoint-avconf.png)

![](https://imgs.piasy.com/2021-03-07-app-breakpoint-webrtc.png)

_WebRTC 库里我是在自己添加的一个 PC 封装类 CFPeerConnectionClient 里加的断点_。
