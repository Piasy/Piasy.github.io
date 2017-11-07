---
layout: post
title: WebRTC-Android 源码导读（六）：混音
tags:
    - 实时多媒体
    - WebRTC
---

基于 branch-heads/63

接口定义在 `api/audio/audio_mixer.h`

实现在 `modules/audio_mixer/audio_mixer_impl.h(cc)`

+ 创建：`const auto mixer = AudioMixerImpl::Create();`
+ 添加声源：`mixer->AddSource(&participants[i])`；
+ 混音并取得结果：`mixer->Mix(1, &audio_frame);`

实现原理：

`CalculateOutputFrequency`：

+ 收集所有声源的 `PreferredSampleRate`；
+ 找出 8K, 16K, 32K, 48K 中大于所有 `PreferredSampleRate` 的最小者，作为输出采样率；

`GetAudioFromSources`：

+ 遍历所有的 Source，如果 `GetAudioFrameWithInfo` 能取到声音数据，则添加到 `audio_source_mixing_data_list` 列表中；
+ 对 `audio_source_mixing_data_list` 列表排序，没有 mute 的在前面，active 的在前面，energy 大的在前面；
+ 从排序后的列表中，最多选出三路；
+ 需要混音的可能还要做 ramp（淡入淡出）和 gain（增益）处理，`RampAndUpdateGain`；
+ 混音路数太多，音质就会太差，janus gateway 的 audio mcu 是只混四路，其他的就丢掉了；webrtc 客户端则是只混三路，其他的也就丢掉了，`GetAudioFrameWithInfo` 就已经从 Source 那里把数据消费了，如果不用，那就没了；
+ 确实线下场景也是这样，多人同时说话，肯定闹成一团听不清，而 WebRTC 的排序以 PCM 样点的平方和作为能量，也确实符合“声音大更容易被听到”的常理；

`GetAudioFrameWithInfo`：

+ 要实现重采样功能；
+ 具体实现？

`FrameCombiner::Combine`：

+ 先 `RemixFrame`，单双声道切换；
+ 零路、单路其实没什么需要操作的，但还是过 limiter，防止增益曲线跳变；
+ 多路就是 PCM 数据直接相加，但要做些平滑处理；
