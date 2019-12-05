---
layout: post
title: WebRTC Native 源码导读（十一）：混音
tags:
    - 实时多媒体
    - WebRTC
---

去年十一月份我就打上了 WebRTC 混音模块的主意，并且做了一些尝试，不过最终没有做完，半年多过去了，正好最近工作上对此有了需求，所以这次就趁热打铁，把它彻底吃掉 :)

_本文的分析基于 WebRTC 的 #23295 提交_。

## AudioMixer 的使用

AudioMixer 的使用比较简单，只需要三步：

+ 创建：`const auto mixer = AudioMixerImpl::Create();`
+ 添加声源：`mixer->AddSource(src);`
+ 混音并取得结果：`mixer->Mix(output_channel_num, mixed_frame);`

_不过我们很可能需要实现自定义的 source，关于 source 我们暂时按下不表，后面再展开_。

## AudioMixer 的实现原理

high level 来看，AudioMixer 实际做的事情也只需要三步：

+ 计算输出采样率：`CalculateOutputFrequency`;
+ 从 source 收集音频数据：`GetAudioFromSources`;
+ 执行混音操作：`FrameCombiner::Combine`;

接下来我们就把这三步分析个究竟。

**CalculateOutputFrequency**

+ 收集所有 source 的 `PreferredSampleRate`，存入 `preferred_rates`；
+ 找出 8K, 16K, 32K, 48K 中大于 `preferred_rates` 所有元素的最小者，作为输出采样率；

**GetAudioFromSources**

+ 遍历所有的 source，如果 `Source::GetAudioFrameWithInfo` 能取到声音数据（返回值不为 `Source::AudioFrameInfo::kError`），则把取到的 `AudioFrame` 添加到 `audio_source_mixing_data_list` 列表中；
+ 对 `audio_source_mixing_data_list` 列表排序，没有 mute 的在前面，active 的在前面，energy 大的在前面；
  - energy 的计算在 `AudioMixerCalculateEnergy` 函数中完成，就是计算 AudioFrame 的 `data_` 的前 `samples_per_channel_` 个样点值的平方和；
  - 首先这个计算可能溢出，源码里也有 TODO 注释；
  - 其次，多声道时，这个计算没有涵盖所有的样点；
+ 从排序后的列表中，最多选出三路没有 mute 的参与混音；
+ 参与混音的 frame 可能还要做 Ramp（淡入）处理 `RampAndUpdateGain`；
  - 一个 source 被添加到 mixer 里时，它的 `gain` 为 0，如果需要参与混音，它的 `target_gain` 就为 1，Ramp 就是把 `gain` 从 0 更新为 1；
  - 如果 frame 的 `gain` 和 `target_gain` 不同，且没有静音，那就需要进行 Ramp 操作：frame 的第一个样点值乘以 `gain`，最后一个样点值乘以 `target_gain`，中间样点值乘以的系数从 `gain` 到 `target_gain` 呈线性变化；
+ 混音路数太多，音质就会太差，Janus gateway 的 audio MCU 是只混四路，其他的就丢掉了；WebRTC 客户端则是只混三路，其他的也就丢掉了（`GetAudioFrameWithInfo` 函数里就已经从 source 那里把数据消费了，如果不用，其实就是丢掉了）；
+ 确实线下场景也是这样，多人同时说话，肯定闹成一团听不清，而 WebRTC 的排序以 PCM 样点的平方和作为能量，也确实符合“声音大更容易被听到”的常理；

**GetAudioFrameWithInfo**

+ 由 `Source` 的子类实现，需要实现重采样功能，根据传入的 `sample_rate_hz` 和自身实际数据的采样率，进行重采样；
+ WebRTC 里收流端的实际实现是 `webrtc::voe::Channel`（它没有继承 Source，而是被层层委托过来的）；
+ 重采样的逻辑实际实现在 `AcmReceiver::GetAudio` 函数里，通过调用 `ACMResampler::Resample10Msec` 实现，而 ACMResampler 则是调用 `common_audio/resampler` 目录中的代码实现，这个 resampler 比较基础，不支持变声道数，功能不如 FFmpeg 的强大；
+ Channel 还实现了音量控制的逻辑，在把数据交给 mixer 之前，它调用 `AudioFrameOperations::ScaleWithSat` 实现音量调整（就是给每个样点值乘以一个系数，但结果不超过 `int16_t` 的取值范围，这正是 Sat 的含义：`saturated_cast`），这个功能最终通过 `AudioRtpReceiver::SetOutputVolume` 暴露出来，不过这个函数是私有的，而且这个类也是内部类；

**FrameCombiner::Combine**

+ FrameCombiner 要求所有待合并的 frame 采样率都相同，且样点数都为 10ms 的样点，即采样率除以 100；
+ FrameCombiner 允许待合并的 frame 声道数不一致，所以需要先做 `RemixFrame` 操作，即单双声道切换，调用 `AudioFrameOperations::MonoToStereo` 或 `AudioFrameOperations::StereoToMono` 实现；（`RemixFrame`）
  - MonoToStereo 实际上就是把单声道的数据拷贝一份，两个声道的样点值完全一致；
  - StereoToMono 则是把两个声道的样点值取平均作为单声道的样点值；
+ 零路、单路其实没什么需要操作的，直接把数据拷贝一份即可；（`MixFewFramesWithNoLimiter`）
+ 多路就是把样点值直接相加，这个过程用到了一个固定大小二维 float 数组，第一个维度是样点数，第二个维度是声道数，大小为 `kMaximumChannelSize * kMaximumAmountOfChannels`，它们的取值分别为 480 和 2，即 `Stereo, 48 kHz, 10 ms`；（`MixToFloatFrame`）
+ 多路时接下来的操作就是限幅了，目前支持两种模式 `apm_agc_limiter_`（`AudioProcessing`）和 `apm_agc2_limiter_`（`FixedGainController`）可供选择，限幅的具体实现这里就不展开了，留待以后探究；
  - _在这里我深刻意识到了自己对 C++ 的不熟悉，FrameCombiner 的构造函数列表里写道 `apm_agc2_limiter_(data_dumper_.get())`，我还以为是把 ApmDataDumper 赋值给 FixedGainController，后来才意识到是用 ApmDataDumper 构造一个 FixedGainController_ :(
+ 限幅之后就是把 float 数据转换为 `int16_t` 并写入输出 frame 里了；

## AudioMixer 的产品化

这次我设计的功能需求其实比较简单，就是普通唱歌 APP 的伴奏混音，只需要把采集到的数据，和伴奏 mp3 文件混音即可。

通过前面的分析，我们发现 WebRTC AudioMixer 的工作模式是，由我们主动索要混音后的 frame，mixer 内部负责向各个 source 索要数据，并进行混音操作。所以我们需要实现两种 source：mp3 文件的 source；音频采集的 source。

另外我们还需要考虑一下调用 `Mix` 函数的时机问题。在我们定义的功能需求下，source 其实是有主辅之分的，音频采集为主，mp3 为辅，如果没有采集到数据，那我们是不用消费 mp3 数据的。但也存在平等 source 的场景，WebRTC 里的使用场景就是这种情况，各个播放器没有主辅之分，混音事件是定时触发的，有数据的播放器就参与混音，没数据的就不参与混音。

在我们的使用场景下，主 source 有了数据之后就可以触发混音操作，由于混音操作是同步执行的，所以我们的接口可以做成同步的，对于平等 source 的场景，就只能做成异步接口了。

具体的代码编写过程我这里就不详细展开了，感兴趣的朋友可以自行[阅读源码](https://github.com/Piasy/AudioMixer)，_其实主要是我自己懒得解析自己写的代码_ :)

## 工程化要点

虽然代码懒得展开讲，但有些工程化的要点还是值得一提的。

### FFmpeg

这里我用了 FFmpeg 的音频解码、AVAudioFifo、音频重采样这三个模块。

网上很多教程，以及开源项目里对 FFmpeg 的使用都是把 FFmpeg 可执行程序打包进 APK，然后通过命令行进行调用。确实命令行模式就已经可以实现很多功能了，如果可以用命令行调用方式解决，就不必在代码中使用，但这也导致这方面资料比较少，所以我这个项目也可以作为 FFmpeg 的一个代码集成使用示例。当然更详细的还是得看[官方文档了](https://www.ffmpeg.org/doxygen/trunk/index.html)。

这里也给大家分享个小窍门，我们可以利用 Google 的 site 语法对其进行搜索：`site:www.ffmpeg.org/doxygen/trunk/ <keywords>`。

另外我也总结了下 [FFmpeg 的编译过程](https://github.com/Piasy/Piasy.github.io/blob/master/_instructions/2018-06-04-build-ffmpeg.md)，感兴趣的朋友可以看看。

### 静态 vs 动态

WebRTC 和 FFmpeg 库都不小，如果以动态库的形式引入，会让 APP “变胖”不少。

其实我们用到的只是这两个框架的很少一部分功能，所以我们可以以静态库的形式引入，最后我们输出一个动态库，这样这两个框架里只有被实际使用到的代码才会打包进去，体积会小很多。

不过对于 SDK 提供方来说，又有一个问题需要考虑：客户使用的其他 SDK 也用了这两个框架怎么办？一是会打包重复代码，浪费空间，二也可能发生运行时冲突。

所以这个问题还是需要仔细考虑的，不过这里我们就偷懒一下，使用静态库就好了 :)

### 跨平台支持

这里我再次利用 [djinni](https://github.com/dropbox/djinni) 做了跨平台支持，不过这里它能发挥作用的地方不多，跨边界传递 binary 数据，会有一次 copy，这对音视频数据来说性能开销就太大了，所以音频数据的传递代码还是得手写。

### 音量控制

通过前面的分析，我们发现有现成的代码可以用：`AudioFrameOperations::ScaleWithSat`。

### 混音路数

如果我们宁愿音质差些，也不愿意丢弃数据，那我们可以修改 WebRTC 的源码，把这个限制改大一些，我初步测试没有发现问题。

### 混音数据单位长度

WebRTC 里的音频处理都是以 10ms 为单位的，所以我们混音操作也需要是 10ms 一次。

这个设定的原因我们暂不深究，我曾想取个巧：source 每次往 frame 里填充数据时，可以只填充一部分，比如 5ms，消费混音后数据时，也只消费 frame 里的一部分，同时我们提高混音操作的频率，这样其实就达到了以更小粒度进行混音的目的。但实际测试这样做效果很差，声音毛刺非常明显。

经过分析发现，在混音的场景下，这个限制主要起作用的是在 AGC 模块，在上面的 5ms 测试里，我也尝试过注释掉对 AGC 模块的调用，效果会好一些，但仍存在少量毛刺，应该是音量溢出导致的。

此外我也尝试把 WebRTC 代码里对这 10ms 的使用、断言都改为 5ms（其他都不变），仍存在少量毛刺，所以现在我就只得暂时放弃 :(

## 附录：部分类实现代码路径

``` bash
api/audio/audio_mixer.h
modules/audio_mixer/audio_mixer_impl.h(cc)
modules/audio_mixer/default_output_rate_calculator.h(cc)
modules/audio_mixer/audio_frame_manipulator.h(cc)
modules/audio_mixer/frame_combiner.h(cc)
modules/audio_coding/acm2/acm_receiver.h(cc)
modules/audio_processing/audio_processing_impl.h(cc)
modules/audio_processing/agc2/fixed_gain_controller.h(cc)
```

---

欢迎大家加入 Hack WebRTC 星球，和我一起钻研 WebRTC。

<img src="https://imgs.piasy.com/2019-11-14-piasy-knowladge-planet.jpeg" alt="piasy-knowladge-planet" style="height:400px">
