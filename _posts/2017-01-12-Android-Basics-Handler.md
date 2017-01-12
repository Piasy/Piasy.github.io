---
layout: post
title: 安卓基础：Handler
tags:
    - 安卓开发
    - 基础知识
---

大家好，博客断更将近三个月之后，终于可以恢复了。近期将要带来的，是一系列安卓系统的基础知识。这些内容其实也算是我对自己知识的一次整理，虽然日常开发工作可能用得不多，但这些基本的东西还是应该扎实地掌握，毕竟如果要参加面试，免不了要回答这些问题的 :)   

水平有限，难免有失偏颇，欢迎大家指正！

今天先讲讲 Handler，如果被问到“讲讲你说 Handler 的理解”，怎么把这么一大串内容讲清楚？

下面这几个具体的问题可能更有助于我们展开思路：

+ 能干啥？
+ 线程间通信怎么实现的？Handler, Looper, MessageQueue, Thread，这几个东西是什么关系？
+ Looper.loop 为什么不会阻塞住主线程？
+ 绘制、点击事件、Activity 生命周期和 Handler 有什么关系？
+ handler.post, handler.handleMessage 分别执行在什么线程？
+ Handler 何时会导致 Activity 泄露？

## 0. Handler 能干啥？

异步，线程间通信，

## 1. Handler, Looper, MessageQueue, Thread，这几个东西是什么关系？

Handler 是一个通信的工具，可以是线程内，也可以是多线程之间；通信可以是发送一个消息，也可以是发送一个任务；我们利用 Handler 发送消息/任务，那么谁来处理消息/任务？处理过程是怎样的？

那这就引出了 MessageQueue，我们发消息只是向队列里面加一个消息，由处理者线程去循环从队列里面取出消息，并且处理；那么这个处理者线程就是调用 Looper.loop 的线程，我们应该都遇到过 Can't create handler inside thread that has not called Looper.prepare() 这个错误，这是因为我们 new Handler 的线程没有调用 Looper.prepare。

可不可以两个不同的线程分别调用 Looper.prepare 和 Looper.loop？这其实是个伪命题，这两个方法的调用，由 myLooper 联系起来，而 myLooper 是利用 ThreadLocal 来存储 Looper 实例的，所以，能正常运作的 Looper，一定是在同一个线程调用的这两个方法。也就是说，最终处理消息的线程，就是调用 Looper.prepare 和 Looper.loop 的线程。

所以，Looper, MessageQueue, Thread 是一一对应的关系，而这一套东西，可以对应多个 Handler。

## 2. Looper.loop 为什么不会阻塞住主线程？

主线程被阻塞的定义是什么？其实就是主线程的 MessageQueue 中，某个消息的处理时间过长，导致后面的消息不能及时处理。

那 Looper.loop 会不会阻塞主线程？看在哪里调用，如果在 Activity.onCreate 中或者其他任何在主线程中执行的代码中调用，当然会，因为主线程进入死循环了。但如果在 ActivityThread 的 main 函数中（这也是这个问题被提出的场景），当然不会。

正是这个无限循环不断地从 MessageQueue 中取出消息，并进行处理，安卓系统的事件循环模型才得以运行。它会导致某个消息的处理时间过长，后面的消息不能及时处理吗？当然不会，没有它都谈不上消息处理！

## 3. 绘制、点击事件、Activity 生命周期和 Handler 有什么关系？

普通 View 的绘制、点击事件的回调、Activity/Fragment 的生命周期函数，都在主线程执行，这一点大家都知道，但它们是怎么执行在主线程上的？所谓主线程，就是 ActivityThread.main 执行的那个线程，它其实一直在一个无限循环里，上面说过的，还记得吗？那这些代码的执行，怎么会发生在主线程呢，主线程不是在死循环里吗？

其实上述的各种代码，都是由事件触发的，各种事件被发送到了主线程的 Handler 上，然后相应的处理代码就在主线程上执行了。至于事件具体怎么触发的，就超出这个问题的范围了。

## 4. handler.post, handler.handleMessage 分别执行在什么线程？

handler.post 当然执行在调用这个函数的线程里，handler.handleMessage 则发生在 new Handler 时的线程，也就是这个 Handler 所对应（或者说绑定）的 Looper 所在的那个线程。

这个问题其实没多大价值，但很多人会搞不清对象、函数、代码、运行、线程这些概念的区别，会问出类似下面的问题：

+ Activity 运行在哪个线程？Presenter 运行在哪个线程？
+ handler.post, handler.handleMessage 运行在哪个线程？

记住一句话：代码会在某个线程运行。Activity 是一个类，某个 Activity 实例一是个对象，类和对象能运行吗？并不能。能运行的只是函数，main 函数也是函数，对吧？

那怎么理解 App 的运行，Activity 的运行？其实就是主进程的主线程一直在跑着事件循环，响应各种事件！

## 5. Handler 何时会导致 Activity 泄露？

如果使用了非静态的内部 Handler 子类、匿名 Handler 子类，或者把非静态的内部 Runnable 子类、匿名 Runnable 子类 post 到任意 Handler 上，就**很可能**发生内存泄漏，而如果这些类都在 Activity 内部，那就泄露了 Activity。

搞清楚概念最重要，什么是内存泄漏？只要一个对象应该被 gc 的时候，仍**不能**被 gc，它就被泄漏了。注意是 _不能_，而不是 _没有_。Activity 何时应该被 gc？onDestroy 函数执行之后，就应该被 gc。

内存泄漏是不是一定很严重？看情况，如果只是泄漏了很短的时间，例如 10ms，基本上不会导致问题，当然这也得看情况。但由于 Activity 对象涉及很大的内存空间，一旦泄漏且产生问题，通常都很严重。

Activity 出了上述情形会发生泄漏，还有各种情况，还包括安卓系统的 bug 导致的。引入 [LeakCanary](https://github.com/square/leakcanary) 吧，众生百态，总能让你意外，也不必强迫症，非要消除所有被发现的内存泄漏，it's a tradeoff :)

## 参考资料

+ The Android Event Loop：http://mattias.niklewski.com/2012/09/android_event_loop.html
+ How to Leak a Context: Handlers & Inner Classes：http://www.androiddesignpatterns.com/2013/01/inner-class-handler-memory-leak.html
+ A small leak will sink a great ship：https://medium.com/square-corner-blog/a-small-leak-will-sink-a-great-ship-efbae00f9a0f#.x0otirhoe
