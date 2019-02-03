---
layout: post
title: RTC 产品在弱网下的性能对比
tags:
    - 实时多媒体
    - WebRTC
---

## 弱网环境模拟

有公司专门卖网损仪（制造网络丢包、延迟抖动，模拟各种网络环境），这种设备功能强大，支持各种丢包延迟策略，但也非常贵，一台设备得十几万人民币。其实利用 mac 电脑也可以搭建一个基本的网络模拟环境：使用 Mac Mini 或者 MacBook 加网卡转换器，使 mac 电脑通过以太网卡连接互联网，通过无线网卡创建一个共享 WiFi，再利用 Xcode Tools 里的 Network Link Conditioner 配置网络延迟和丢包。具体教程网上非常多，大家可以自行搜索，这里就不赘述了。

不过这里我可以分享一下几种配置，和实际测出来的 ping 结果，测试环境为 Nexus 5X 和 iPhone 6 ping 一台阿里云华北服务器（ping 的结果每次都会有一定的差异，比如丢包 1~2% 的波动，延迟 20~30ms 的波动，应该都属于正常情况）。

|                                                                         | Nexus 5X                                         | iPhone 6                                        |
| ----------------------------------------------------------------------- | :----------------------------------------------: | :---------------------------------------------: |
| 强网（关闭 Network Link Conditioner）                                     | 0% loss, RTT min/avg/max 7.51/13.21/311.51       | 0% loss, RTT min/avg/max 7.8/11.15/161.12       |
| 中网（Downlink/Uplink/DNS delay 14ms, Downlink/Uplink packet drop 1%）   | 3.79% loss, RTT min/avg/max 69.92/106.02/931.58   | 3.4% loss, RTT min/avg/max 74.39/109.45/299.29 |
| 弱网（Downlink/Uplink/DNS delay 45ms, Downlink/Uplink packet drop 2.8%） | 11.66% loss, RTT min/avg/max 234.52/263.53/731.27 | 10.7% loss, RTT min/avg/max 192.73/269.9/405.06 |

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
