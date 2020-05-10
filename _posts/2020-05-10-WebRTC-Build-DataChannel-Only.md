---
layout: post
title: WebRTC 只编译 DataChannel
tags:
    - WebRTC
    - 实时多媒体
---

上周一位朋友在知识星球里提问：

> 博主你好，我们是做 P2P 流媒体的，简单的说就是用 DataChannel 来传输数据，不需要用到音视频那些功能，但发布的 SDK 却不得不打包整个 WebRTC，太大了而且感觉没必要。请问有没有什么办法把 P2P 连接和 DataChannel 模块单独弄出来呢？

看到这个问题，我第一反应就是我曾在 WebRTC 代码库里看到过相关的 gn 选项或者宏的，可是在 m83 的代码和 gn 里，却没看到相关的宏/配置。不过想来这个问题应该是比较常见，所以 Google 一下应该会有结果。搜索一番后，我做了如下解答：

> 只编译 data channel，我以前在代码/gn 文件里是看到过相关的关键字的，但是刚才在 m83 的代码/gn 里，没看到相关的宏/配置了。
> 不过 iOS 现在还有个 `peerconnectionfactory_no_media_objc` target，首先可以看看这个 target 怎么用，以及是否能小很多。如果好使，那就可以参考着看看安卓怎么搞。
> 另外 [M63 release note 里有 Build WebRTC with DataChannel only](https://groups.google.com/forum/?__s=obdrvn8g53z98ugdgkuz#!msg/discuss-webrtc/qDtSDxoNSII/69b6fAkxAQAJ)，实在不行也可以看看当时是怎么搞的，参考你们现在用的代码，搞一搞。

几天之后，这位朋友再次向我求助，确实，编译 WebRTC 这件事如果没搞过，看着都会很头大的，而且这次的任务显然需要对 gn 和 WebRTC 代码有一定的了解。

今天我花了将近六个小时，成功把 iOS WebRTC arm64 架构的库文件从 5.3MB 缩减到了 2.3MB，体积缩减 56.6%，还是非常可观的，现在我把这个过程分享给大家。

思路就是前面回答的，先试试现有的 `peerconnectionfactory_no_media_objc` 是否好使，不行再看 M63 的修改能否参考。

## `peerconnectionfactory_no_media_objc`

由于发布版本是通过 `tools_webrtc/ios/build_ios_libs.py` 编译的，为了便于验证修改，裁剪的过程我是编译的调试版本，所以我先编了一个未裁剪的调试版本看看大小：66986352 B。

然后我把 `sdk/BUILD.gn` 里的 `framework_objc` 和 `ios_framework_bundle` 复制了一份，得到了 `framework_objc_dc_only` 和 `ios_framework_bundle_dc_only`，并修改了库名、`Info.plist` 里的 Bundle Id。这里之所以没有直接修改已有的 target，是为了便于出错后对比分析。

`framework_objc_dc_only` 和 `framework_objc` 最主要的区别就是要依赖 `peerconnectionfactory_no_media_objc` 了，其关键部位如下：

```python
common_objc_headers = [
    "objc/base/RTCLogging.h",
    "objc/base/RTCMacros.h",
    "objc/helpers/RTCDispatcher.h",
    "objc/helpers/UIDevice+RTCDevice.h",
    "objc/api/peerconnection/RTCConfiguration.h",
    "objc/api/peerconnection/RTCDataChannel.h",
    "objc/api/peerconnection/RTCDataChannelConfiguration.h",
    "objc/api/peerconnection/RTCFieldTrials.h",
    "objc/api/peerconnection/RTCIceCandidate.h",
    "objc/api/peerconnection/RTCIceServer.h",
    "objc/api/peerconnection/RTCIntervalRange.h",
    "objc/api/peerconnection/RTCLegacyStatsReport.h",
    "objc/api/peerconnection/RTCMediaConstraints.h",
    "objc/api/peerconnection/RTCMetrics.h",
    "objc/api/peerconnection/RTCMetricsSampleInfo.h",
    "objc/api/peerconnection/RTCPeerConnection.h",
    "objc/api/peerconnection/RTCPeerConnectionFactory.h",
    "objc/api/peerconnection/RTCPeerConnectionFactoryOptions.h",
    "objc/api/peerconnection/RTCSSLAdapter.h",
    "objc/api/peerconnection/RTCSessionDescription.h",
    "objc/api/peerconnection/RTCTracing.h",
    "objc/api/peerconnection/RTCCertificate.h",
    "objc/api/peerconnection/RTCCryptoOptions.h",
    "objc/api/logging/RTCCallbackLogger.h",
    "objc/api/peerconnection/RTCFileLogger.h",
]

sources = common_objc_headers
public_headers = common_objc_headers

deps = [
    ":peerconnectionfactory_no_media_objc",
    "../rtc_base:rtc_base_approved",
    ":callback_logger_objc",
    ":file_logger_objc",
]
```

去掉了很多音视频相关的头文件，以及依赖 target。

不过编译的时候发现，编译目标数只是从 2650 降到了 2391，那这体积肯定不会减小太多，编完后果然如此：63074224 B。

所以还得 get hands dirty :)

## `peerconnectionfactory_base_objc`

虽然 `peerconnectionfactory_no_media_objc` target 的注释里提到：

```python
# Build the PeerConnectionFactory without audio/video support.
# This target depends on the objc_peeerconnectionfactory_base which still
# includes some audio/video related objects such as RTCAudioSource because
# these objects are just thin wrappers of native C++ interfaces required
# when implementing webrtc::PeerConnectionFactoryInterface and
# webrtc::PeerConnectionInterface.
# The applications which only use WebRTC DataChannel can depend on this.
```

说得好听，`objc_peeerconnectionfactory_base` 只是一层很瘦的包装，但实际并非如此（注释里居然还有个 typo），所以下一步就是干掉 `peerconnectionfactory_base_objc` 里的各种音视频相关依赖了。

在尝试的过程中，我发现 `peerconnectionfactory_no_media_objc` 里定义的 `defines = [ "HAVE_NO_MEDIA" ]` 不管用，于是直接在根目录的 `BUILD.gn` 里加上这个定义，然后修改了 PC 和 PC Factory 的代码，对引用了 `RTCMediaStream`/`RTCMediaStreamTrack`/`RTCRtpReceiver`/`RTCRtpSender`/`RTCRtpTransceiver`/`RTCRtpTransceiverInit` 的地方，都用 `#ifndef HAVE_NO_MEDIA` 括起来，并把 `peerconnectionfactory_base_objc` 编译的上述文件，以及依赖，通通删掉。再次编译，果然有不小的收获：库文件变为了 45686288 B。

这个过程最主要的还是干掉了 `videotoolbox_objc`, `source`, `track`, `transceiver`, `rtp`, `stats`。

至此，`peerconnectionfactory_base_objc` 已经看不见多余的内容了，正向分析到头了。

## 逆向分析

正向分析到头了，那咱们就换个方向：根据编出来的文件，进一步干掉一些模块，从 obj 目录的大小来找目标。modules 排第二，其他的估计也不好剥，所以就以 modules 为主了。

modules 里面最显眼的就是 `audio_processing` 了，通过在所有的 `BUILD.gn` 里搜索 `audio_processing`，找到依赖它的 target，删掉依赖、尝试编译、查看大小。这个过程没啥难点，就是能搜到很多结果，因为不确定到底删掉哪个管用，所以就全删了。最后果然不失所望，库文件变为了 42114048 B。

然后再继续寻找目标，干掉了 `rtp_rtcp` 这个 target，库文件变为了 36949584 B。

本来打算到此为止的，因为在上述过程中想到了另一种办法：编一个完整的静态库出来，对 C++ 的 PC 接口做个包装，只引用 DataChannel 相关的接口，这样让链接器自动帮我们裁剪，效果应该要好不少。但这种办法需要写一些代码，由于时间和预算有限，所以就放弃这个思路了。

不过由于前面删的太多，也不清楚到底哪些是关键，所以之后又花了一些时间，通过恢复、编译、对比，删除、编译、对比，如此循环数十次，总算让修改变得最少了。

而且在这个过程中还有新的发现，干掉了更多的依赖，库文件变为了 30937488 B，而这个过程只在 `pc/peer_connection.cc` 里加了一个 `#ifndef HAVE_NO_MEDIA`。这个过程很重要的一个修改是把 `webrtc.gni` 里的 `rtc_enable_protobuf` 改为 `false`。

## push the limit

既然都到这里了，何不把 `rtp_rtcp` 模块的所有代码都干掉呢？因为 DataChannel 是用的 SCTP，所以 RTP/RTCP 是完全用不上的。不过为此对就需要在不少代码文件里加 `#ifndef HAVE_NO_MEDIA` 了。

最后在 10 个文件里加了 `#ifndef HAVE_NO_MEDIA`，成功干掉了 `rtp_rtcp` 模块，而此时 `video`, `audio`, `common_video`, `common_audio`, 以及 `modules` 目录下的绝大部分子目录，都没有被编译了，编译目标数只有 866 了，而库文件则变为了 27690304 B。

这时我把原有的 `framework_objc` 删掉，通过脚本编译发布版，得到了 2.3MB 的库文件。

---

安卓的编译，因为代码还没同步完，所以还得再等等，不过估计也只需要修改 sdk 下的 BUILD.gn 以及 JNI 代码了，Java 代码要动也可以，更简单。

完整代码，可以从 [GitHub 获取](https://github.com/cdnbye/DataChannel)。
