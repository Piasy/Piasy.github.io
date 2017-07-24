---
layout: post
title: 记一次 WebRTC 安卓编码流程中的内存优化
tags:
    - 性能优化
    - 流媒体
    - WebRTC
---

一开始我发现内存抖动非常严重，CPU 占用也很高：

![](https://imgs.piasy.com/2017-07-23-monitor_1.png)

Allocation Tracker 的结果如下：

![](https://imgs.piasy.com/2017-07-23-allocation_track_result1.png)

这里我们可以看到，60% 的内存分配都发生在 BufferInfo 对象上，但这个对象非常小，只有几个 primitive 数据成员，怎么会出现这么多分配呢？我们看次数，15s 内发生了 2.6 万次，每毫秒分配了 1.7 次。看代码发现，是我在单独的线程调用 `dequeueOutputBuffer` 时传入的 timeout 为 0，所以在疯狂的创建 BufferInfo 对象。

单独的线程设置 timeout 为 0 显然不合理，除了这里的内存分配，CPU 占用也会更高，所以我们可以设置一个合适的值，这里我换成 3000，也就是 3ms，结果如下：

![](https://imgs.piasy.com/2017-07-23-monitor_2.png)

我们可以看到，内存抖动减缓了很多，但仍比较明显。CPU 占用率倒是下降很多了。

这时 Allocation Tracker 的结果如下：

![](https://imgs.piasy.com/2017-07-23-allocation_track_result2_2.png)
![](https://imgs.piasy.com/2017-07-23-allocation_track_result2_3.png)

优化性能切忌盲目，要找准瓶颈，并且测量对比成效。

我们发现最大的分配竟是由一条日志代码引起的！所以不要以为在日志工具函数内通过变量控制是否打日志就够了，即便日志最终没有打印出来，但拼接日志字符串就可能已经成为瓶颈。

除了日志，还存在两处很高的分配：allocateDirect 和 BufferInfo。

其实 BufferInfo 没必要每次创建，我们消费 MediaCodec 输出是单线程行为，只需要分配一次即可。同理，容纳输出数据的 buffer 也没必要每次分配，只有需要扩容时创建即可。

经过上述优化，内存抖动再次减弱：

![](https://imgs.piasy.com/2017-07-23-monitor_3.png)

分析 Allocation Tracker，较高的内存分配分别为：

+ `Display#getRotation`：19.58%；
+ 取到 MediaCodec 输出后，构造 ByteBuffer 对象的拷贝：17.31%；
+ 帧数据传递过程中 matrix 数组分配：16.32%；

上面这几点都改了之后，再剩下的就是 I420Frame 的创建、日志字符串拼接、Runnable 对象创建了。此外还发现了一个注意点：使用 ExecutorService 时，每次 submit 任务，还会创建一个链表节点对象，而 Handler 会复用 Message 对象，所以我把 ExecutorService 换成了 HandlerThread + Handler 的组合。

I420Frame 也许可以用对象池来优化，Runnable 则可以把局部变量成员化，但现在其实已经优化了很多，而这些做法需要比较复杂的处理才能确保不会发生消费者-生产者的竞争问题，所以就先告一段落啦！

最后在 Nexus 5X 安卓 7.1.2 上测试发现，Camera2 采集时，存在大量的 Binder 通信，内存抖动严重得多，而其中 48.86% 都是由 Binder 通信导致的。

+ Camera1 采集：一分钟内存增长 0.32MB；
+ Camera2 采集：一分钟内存增长 3.13MB；
