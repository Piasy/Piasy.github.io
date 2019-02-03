---
layout: post
title: 从 A/Looper&#58; Could not create epoll instance. errno=24 错误浅谈解决各种 bug 的思路
tags:
    - 安卓开发
    - 踩坑之旅
---

今天代码写着写着就莫名闪退了，手机也没有“程序停止运行”的提示，logcat也没有看到蓝色的调用栈log，这样的闪退最是蛋疼了，还好必现。复现几次之后，终于从logcat中看到了一行可疑的log：`A/Looper: Could not create epoll instance. errno=24`，看起来又是在native层闪退了。本文就把这个问题的分析解决过程记录了下来。

## 方法论
遇见没填过的坑，第一反应就是Google之，果然前几个结果中一个[Stack Overflow的问答](http://stackoverflow.com/questions/15203271/could-not-create-epoll-instance-errno-24)就为这个问题的解决提供了很好的思路。当然搜索结果并不能直接解决问题，如果Google的结果直接就能解决问题，我也就没必要写这样一篇文章，徒增无用信息了。

除了能够直接搜到完全一样的问题，并且已经有了解决办法的情况，还有时候只能搜到相关的问题，但是没有解决办法，或者解决办法不work，或者压根儿搜不到相关信息，这时候就需要自己根据已有信息一步步分析了，甚至只能自己一步步趟坑了，通常有几种情形：

有log，但是搜索结果没有解决方案（经常遇到有人两年前在stack overflow上问了一个一模一样的问题，但是没有答案），或者解决方案不work。这时我们可以**根据搜到的信息，确定问题出在哪儿**。就比如这个问题，[stack overflow上的回答](http://stackoverflow.com/questions/15203271/could-not-create-epoll-instance-errno-24)就说可能是打开了太多文件，而采纳的回答说去掉`looper.prepare()`就解决了问题，那就可能是创建了太多的`lopper`。又比如[之前一篇文章](/2016/02/24/Robust-Android-Audio-encapsulation/index.html)中遇到的log `A/libc: Fatal signal 11 (SIGSEGV) at 0x00000010 (code=1), thread 9302 (RxComputationTh)`，搜索结果就说可能是对native代码的多线程执行导致的。

有log，但是搜索不到有意义的结果。有时候搜索无结果可能是因为搜索内容包含了太多本地计算机相关的信息，比如文件路径，很多时候**删掉文件路径、时间戳等信息，就能搜到结果了**。如果还是搜不到结果，而log又有较明确的意义，这时候我们就可以**直接分析log的含义，确定问题的原因**。比如[之前另一篇文章](/2015/10/06/AndroidTDDBootStrap-Use-OkBuck/index.html)中遇到的`error: 不兼容的类型: java.util.List<java.lang.Object>无法转换为 java.util.List<com.github.piasy.model.entities.GithubUser>`，拿着这个错误信息去搜肯定都是不相干的结果，因为去掉自己定义的类路径之后，太短而且太常见了（类型转换错误）。但是通过分析报错信息，可以知道确实是类型转换错误。通过在代码中找和这个类相关的发生了类型转换的地方，最终解决了问题。

最惨的情况就是没有log，或者log没有任何有用信息了。这时候如果问题是突然冒出来的，这时候就可以通过**代码差异分析**来确定引入bug的代码了，通常情况下二分查找是最高效的，而git则有一个无比牛逼的命令，`git bisect`，通过二分查找commit，来定位引入bug的坏commit。而如果只有一个commit，或者没有版本控制（啥？），那就只能新拉一个分支，通过逐步注释/删除代码，来定位bug代码了。

好吧上面不是最惨的，最惨的就是上面的情况都无法适用。这就真的没别的办法了，只能推倒重来了，其实这还是属于通过回滚来定位坏代码的类型，由此可见，回滚是定位bug的万精油，只不过效率可能是最低的。

## `A/Looper: Could not create epoll instance. errno=24`错误的分析解决过程
好了回到本次的具体问题上来，既然判断可能是由于创建了太多的looper导致的闪退，那就查看代码咯。但是我的代码并没有一处创建了looper呀！别急，looper还有个兄弟handler，handler可是经常使用的，看看是不是handler创建了太多。

通过为创建handler的代码加“非中断”断点（调试时不中断程序执行，只记录log），发现确实就是创建了太多的handler！

好了，找到了问题可能的原因，改掉会导致创建过多handler的代码，测试运行，闪退确实不再出现了，OK问题圆满解决 :)
