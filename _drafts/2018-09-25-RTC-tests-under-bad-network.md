---
layout: post
title: RTC 产品在弱网下的性能对比
tags:
    - 实时多媒体
    - WebRTC
---

我们对卡顿进行如下分类：

* 大卡：卡顿大于 5s
* 中卡：卡顿 1~5s
* 小卡：卡顿小于 1s

测试结果对比：

|                           | 大卡 | 中卡 | 小卡 | 流畅播放时长 |
| ------------------------- | :-: | :--: | :-: | :--------: |
| Janus, iOS => Android     | --- | ---- | --- | 0s         |
| Janus, Android => iOS     | --- | ---- | --- | 7.3s       |
| P2P, iOS => Android       | 0   | 2    | 3   | ---------- |
| P2P, Android => iOS       | 0   | 2    | 7   | ---------- |
| Agora, iOS => Android     | 0   | 0    | 1   | ---------- |
| Agora, Android => iOS     | 0   | 0    | 0   | ---------- |
| Zego, iOS => Android      | 0   | 1    | 2   | ---------- |
| Zego, Android => iOS      | 0   | 0    | 2   | ---------- |

P2P 模式比 Janus 模式好太多了，那是由哪些因素决定的呢？我尝试修改 SDP，去掉 rtx, rtcp-fb, fec 相关参数，进行了对比测试，结果如下：

|                       |               |                                  |
| --------------------- | :-----------: | :------------------------------: |
| no fec                | rtcp-fb, rtx  | 略好于 rtcp-fb, fec               |
| no rtx                | rtcp-fb, fec  | 明显差于 P2P，但明显好于 Janus      |
| no rtx, fec           | rtcp-fb       | 和 rtcp-fb, fec 差不多            |
| no rtcp-fb            | rtx, fec      | 略好于 fec                        |
| no rtx, rtcp-fb       | fec           | 略好于 Janus                      |
| no rtcp-fb, fec       | rtx           | 和 Janus 效果差不多                |
| no rtx, rtcp-fb, fec  | ------------- | 比 Janus 效果更差，iOS 没有流畅播放过 |

由此可见，rtcp-fb, fec, rtx 三者的作用呈递减关系，并且 rtcp-fb 的作用明显比 fec 大很多，而 fec 和 rtx 的差别并不是很明显。

但在观察了 Janus 模式的 SDP 之后，我发现是有 rtcp-fb 相关选项的，但为什么效果却那么差呢？

分析 WebRTC 的日志，VideoSendStream stats 和 VideoReceiveStream stats：

+ 收发端的 stats，能否对上？
+ P2P 和 Janus 模式，有何区别？
