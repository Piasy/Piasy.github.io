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

+ `GetAudioFromSources`，遍历所有的 Source，如果 `GetAudioFrameWithInfo` 能取到声音数据，则添加到 `audio_source_mixing_data_list` 列表中；
+ 对 `audio_source_mixing_data_list` 列表排序，没有 mute 的在前面，active 的在前面，energy 大的在前面；
+ 从排序后的列表中，最多选出三路；
+ 需要混音的可能还要做 ramp（淡入淡出）和 gain（增益）处理；
