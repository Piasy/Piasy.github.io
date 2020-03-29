---
layout: post
title: WebRTC H.264 编码的 Profile 和 Level
tags:
    - WebRTC
    - 实时多媒体
---

今天这篇文章的起因是在整理 iOS 屏幕共享代码时，H.264 编码失败，日志里一直在报错 `H264 encode failed with code: -12902`，去 [OSStatus.com](https://www.osstatus.com/search/results?platform=all&framework=all&search=-12902) 搜索发现这个错误是 `kVTParameterErr`，即参数错误。

对参数做了一番检查，以及对比了新老版本代码后发现，可能是分辨率的问题，把视频帧的分辨率变小后果然就没问题了。

原来是我测试时使用的 H.264 Level 是 3.1，[3.1 最高只支持 1280x720](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Levels)，而 iPhone 11 录屏采集到的分辨率是 1472x828，超出了 3.1 支持的最大分辨率，因此报错。

趁此机会，我对 WebRTC H.264 编码的 Profile 和 Level 相关代码再做了查验，在这里分享给大家。

## SDP 里的 profile-level-id

首先我们需要了解一下 SDP 里 profile-level-id 的含义，比如 SDP 里会有如下两行：

```
a=fmtp:96 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=640c1f
...
a=fmtp:98 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
```

`profile-level-id=640c1f` 和 `profile-level-id=42e01f` 就是两种 H.264 的 Profile 和 Level 组合，它可以分为三部分，每部分为两个十六进制数字，从左至右依次为 `profile_idc`, `profile_iop`, `level_idc`。

通常我们只需要关注 `profile_idc` 和 `level_idc`，它们都是十六进制数字，其十进制值有明确定义。

比如 `profile_idc` 0x64 = 100，[100 就是 High Profile 的编号值](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Profiles)，0x42 = 66，[66 就是 (Constrained) Baseline Profile 的编号值](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Profiles)。再看 `level_idc`，0x1f = 31，即 Level 3.1。

更多关于 `profile-level-id` 的解释，可以查看 [SDP Profile-level-id 解析](https://blog.csdn.net/liang12360640/article/details/52096499)这篇博客。

## WebRTC iOS SDP 的 profile-level-id 来源

创建 SDP 时，最终会调用到 `sdk/objc/components/video_codec/RTCDefaultVideoEncoderFactory.m` 的 `supportedCodecs` 函数：

```objective-c
+ (NSArray<RTCVideoCodecInfo *> *)supportedCodecs {
  NSDictionary<NSString *, NSString *> *constrainedHighParams = @{
    @"profile-level-id" : kRTCMaxSupportedH264ProfileLevelConstrainedHigh,
    @"level-asymmetry-allowed" : @"1",
    @"packetization-mode" : @"1",
  };
  RTCVideoCodecInfo *constrainedHighInfo =
      [[RTCVideoCodecInfo alloc] initWithName:kRTCVideoCodecH264Name
                                   parameters:constrainedHighParams];
  NSDictionary<NSString *, NSString *> *constrainedBaselineParams = @{
    @"profile-level-id" : kRTCMaxSupportedH264ProfileLevelConstrainedBaseline,
    @"level-asymmetry-allowed" : @"1",
    @"packetization-mode" : @"1",
  };
  RTCVideoCodecInfo *constrainedBaselineInfo =
      [[RTCVideoCodecInfo alloc] initWithName:kRTCVideoCodecH264Name
                                   parameters:constrainedBaselineParams];
  RTCVideoCodecInfo *vp8Info = [[RTCVideoCodecInfo alloc] initWithName:kRTCVideoCodecVp8Name];
#if defined(RTC_ENABLE_VP9)
  RTCVideoCodecInfo *vp9Info = [[RTCVideoCodecInfo alloc] initWithName:kRTCVideoCodecVp9Name];
#endif
  return @[
    constrainedHighInfo,
    constrainedBaselineInfo,
    vp8Info,
#if defined(RTC_ENABLE_VP9)
    vp9Info,
#endif
  ];
}
```

可以看到，profile-level-id 设置的都是 `kRTCMaxSupportedH264ProfileLevelXXX`……

_写到这里，我才发现是我 fork 的 WebRTC 分支里，把 `MaxSupported` 改为了 `Level31`，应该是在对接 Licode 时改的，后来放弃 Licode 了，但一直没改回来。_

改回去之后，测试发现 iPhone 11 最高支持的 Level 是 5.2，4K 都能支持，1472x828 当然不在话下了。

## WebRTC Android SDP 的 profile-level-id 来源

创建 SDP 时，最终会调用到 `sdk/android/api/org/webrtc/DefaultVideoEncoderFactory.java` 的 `getSupportedCodecs` 函数，而里面最终会调用到 `sdk/android/src/java/org/webrtc/H264Utils.java` 的 `getDefaultH264Params` 函数：

```java
public static final String H264_PROFILE_CONSTRAINED_BASELINE = "42e0";
public static final String H264_PROFILE_CONSTRAINED_HIGH = "640c";
public static final String H264_LEVEL_3_1 = "1f"; // 31 in hex.
public static final String H264_CONSTRAINED_HIGH_3_1 =
    H264_PROFILE_CONSTRAINED_HIGH + H264_LEVEL_3_1;
public static final String H264_CONSTRAINED_BASELINE_3_1 =
    H264_PROFILE_CONSTRAINED_BASELINE + H264_LEVEL_3_1;

public static Map<String, String> getDefaultH264Params(boolean isHighProfile) {
  final Map<String, String> params = new HashMap<>();
  params.put(VideoCodecInfo.H264_FMTP_LEVEL_ASYMMETRY_ALLOWED, "1");
  params.put(VideoCodecInfo.H264_FMTP_PACKETIZATION_MODE, "1");
  params.put(VideoCodecInfo.H264_FMTP_PROFILE_LEVEL_ID,
      isHighProfile ? VideoCodecInfo.H264_CONSTRAINED_HIGH_3_1
                    : VideoCodecInfo.H264_CONSTRAINED_BASELINE_3_1);
  return params;
}
```

可以看到，Android 是写死的 3.1。

## Android 1080p 编码测试

出于好奇，我想试试 Android 编超出 Level 3.1 上限的分辨率是否会出错，这里我同样使用了录屏作为数据来源。

初步测试发现成功编出了 1080p 的流，但仔细看了下日志，发现用的是 Constrained Baseline，而 Android 设置编码器 Profile Level 参数的代码长这样：

```java
if (codecType == VideoCodecType.H264) {
  String profileLevelId = params.get(VideoCodecInfo.H264_FMTP_PROFILE_LEVEL_ID);
  if (profileLevelId == null) {
    profileLevelId = VideoCodecInfo.H264_CONSTRAINED_BASELINE_3_1;
  }
  switch (profileLevelId) {
    case VideoCodecInfo.H264_CONSTRAINED_HIGH_3_1:
      format.setInteger("profile", VIDEO_AVC_PROFILE_HIGH);
      format.setInteger("level", VIDEO_AVC_LEVEL_3);
      break;
    case VideoCodecInfo.H264_CONSTRAINED_BASELINE_3_1:
      break;
    default:
      Logging.w(TAG, "Unknown profile level id: " + profileLevelId);
  }
}
```

也就是说 Constrained Baseline 时是不会设置 Profile Level 参数的。

为了能成功使用 High，需要修改 `sdk/android/api/org/webrtc/HardwareVideoEncoderFactory.java` 的 `isH264HighProfileSupported` 函数，它原本长这样：

```java
private boolean isH264HighProfileSupported(MediaCodecInfo info) {
  return enableH264HighProfile && Build.VERSION.SDK_INT > Build.VERSION_CODES.M
      && info.getName().startsWith(EXYNOS_PREFIX);
}
```

也就是说只有三星的芯片才认为支持 High，所以我们需要把 `&& info.getName().startsWith(EXYNOS_PREFIX)` 这个条件去掉。

修改之后发现确实启用了 High，而且也确实设置了 Profile Level 参数，且 Level 设置的确实是 3，但是依然可以成功编出 1080p 的流。

所以 Android 的 H.264 编码器，并没有完全遵循 H.264 规范，超出 Level 限定的分辨率也能正常编（我在华为 P10 plus 和 Nexus 5X 上测试，均如此）。

## iOS 屏幕共享额外修改

如果对比 Android 和 iOS PC Factory 的 createVideoSource 接口（iOS 叫 `videoSource`），我们会发现 Android 比 iOS 多一个 `isScreencast` 参数。实际上这个参数还比较重要，因为 WebRTC 内部会根据网络情况调整视频分辨率或帧率，而如果是屏幕共享（`isScreencast` 为 `true`）则不调分辨率。

所以如果希望屏幕共享时能保持分辨率不变，那就需要对 iOS PC Factory 的接口稍作修改，增加一个 `isScreencast` 参数，并在 VideoSource 的 `is_screencast` 接口里返回。
