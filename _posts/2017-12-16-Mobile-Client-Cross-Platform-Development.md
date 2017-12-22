---
layout: post
title: 移动客户端跨平台开发方案探索
tags:
    - 
---

跨平台开发想必很多朋友都听说过，甚至实践过，这里我就不过多介绍相关的背景了，Java 的 Slogan 完美诠释了这一愿景：**Write once, run anywhere！**

提到这个话题，大家首先想到的可能是 React Native，不过本文并不是 RN 教程。本文旨在探索我关注到的几种比较靠谱的移动客户端跨平台开发方案，当然 RN 是其中必不可少的一部分。

完美的跨平台开发方案自然是一行平台相关的代码都没有，但这个要求稍显苛刻，很多时候退一步海阔天空，在移动客户端跨平台开发这个场景下也是这样。所以我们的目标应该是尽可能地减少平台相关代码的开发，最大程度复用核心逻辑代码。

此外，客户端的开发可以分为两部分：业务逻辑与界面交互。由于不同的平台（Android、iOS、Web）GUI 系统差异较大，通常来说界面交互共享代码更加困难。这对于 APP 开发确实是个不小的问题，但对于 SDK 开发来说却影响较小。

## 可选方案

接下来我们先看看有哪些可选方案：

+ [React Native](https://facebook.github.io/react-native/) (Javascript)
+ [Flutter](https://flutter.io/) (Dart)
+ [J2ObjC](https://developers.google.com/j2objc/) + [GWT](http://www.gwtproject.org/) (Java)
+ [Djinni](https://github.com/dropbox/djinni) + [WebAssembly](http://webassembly.org/) (C++)

其实如果要把小程序和 Weex 列入其中，也未尝不可，毕竟能在不同平台的微信和支付宝里运行，也堪称跨平台了。

_注：上述各项技术的基本概念本文不会过多介绍，请大家自行查阅官方文档_。

### React Native

RN 由 Facebook 开源，Facebook 内部很多项目在用，工业界也有不少公司和团队都在使用。这样的项目，坑肯定有，但一定不会没人填。

RN 的主要优势有两大部分，一是 React 本身的优势，二是将「React」「Native 化」之后跨平台的优势。这里我主要总结下 React 本身的优势（或者说特性）：

+ 组件化：组件根据状态（state）和属性（props）确定 UI，各种组件组成 view tree；
+ 单向数据流：props 只允许自顶向下传递，父组件设置子组件的 props；交互事件则自底向上抛出，父组件收到子组件内的交互事件后更新 state，触发整个 view tree 的更新（重新设置子组件的 props）；
+ Virtual DOM：view tree 和实际 DOM tree 独立，通过计算差量，每次刷新界面只更新变化的 DOM 元素，提高效率；
+ JSX 语法、render 函数让代码更易于理解和维护；

我觉得下面这幅图形象地表达了 React 的设计精髓：**building large applications with data that changes over time**。

![](https://imgs.piasy.com/2017-12-17-react_ui_is_f_data.png)

内部原理：

+ 写的是 JS 代码，实际运行的也是 JS 代码，在 JavaScriptCore VM 中运行（而不是 WebView）；
+ JS 代码里的 View 实际会使用 native View（在 RN 的上下文中，native 指的是原生），映射关系是可以控制的；
  + 有一个 [native View 市场](https://react.parts/)，可以发现轮子、发布轮子；
+ 支持 JS 代码和 native 代码互相调用，三大特点：异步，批量，序列化；
  + 实际上 native 化靠的正是 JS 和 native 互相调用（RN 内部实现）；
  + 双向调用都是异步的；
  + 跨边界调用实际上是消息收发和处理，由于跨边界存在开销，所以批量处理减小开销；
  + 不跨边界共享可变对象，而是跨边界交换可序列化的消息；
+ 跨边界调用通过 bridge 层实现；
  + 把 native 模块对象注册到 bridge，之前 JS 代码里面的函数调用就可以执行了（JS 是动态语言，运行时有函数/属性，就能访问）；
  + 跨边界调用实际上是消息收发和处理，例如 JS 调用 native 的一个函数，就往一个特殊的队列里放入一条消息，bridge 取出消息后，利用反射，调用 native 函数，反之亦然；
  + bridge 可以替换 JS runtime，这样就变得很灵活了，例如可以在单独的 WebKit 进程中运行，可以在 Chrome 里面调试 RN APP；
  + 下图形象地展示了跨边界调用的过程：

  ![](https://imgs.piasy.com/2017-12-17-react_native_bridge.png)

总结：RN 利用 JavascriptCore VM 在 iOS/Android 系统上执行 JS 代码，实现了业务逻辑跨平台；通过 JS 和 native 互调完成 UI native 化，实现了界面交互跨平台。虽然 JS VM 解释执行、跨边界调用存在一定的性能开销，但大部分情况下它们都不会成为瓶颈，如果开发任务大多集中在界面交互、非计算密集的业务逻辑上，RN 是不错的选择。

### Flutter

Flutter 由 Google 开源，项目年龄比 RN 短大半年，目前仍处于 alpha 状态，没有正式发布版（RN 虽然更新频繁，而且还处于 `0.x` 版本，但好歹也算发布了）。

优势：

+ 自带 _beautiful UI_；
+ UI 代码的编写，借鉴了 React 的思想：组件化，声明式，集中的 build 函数（React 的 render）……
+ 利用 Flutter 自己的 GUI 引擎，高效实现所有的 2D GUI 开发；
+ 相机（拍照）、传感器、GPS、网络、持久化，都可以在 Flutter 里使用（利用 Dart 和 native 互调）；
+ [Dart package 市场](https://pub.dartlang.org/)，也可以发现轮子、发布轮子；

内部原理：

+ 架构图：

![](https://imgs.piasy.com/2017-12-17-Flutter_System_Architecture.png)

+ 核心引擎部分是纯 C++ 代码，iOS/Android 都直接支持，Dart 代码则会预编译为机器码，在各个系统里直接执行，没有解释器 _赚差价_；
+ UI 部分 Flutter 不使用任何系统 OEM 控件，只用在自己的 GUI 体系内打造的控件，因为 _Google 信不过 OEM_，Flutter 自己完全实现了一套跨平台的 GUI 系统，包括图层组合渲染、触摸事件、手势、动画、控件……这个 GUI 系统承载于一个 FlutterView（Android 继承自 SurfaceView，iOS 继承自 UIView）；
+ 支持 Dart 和 native 代码双向通讯（和 RN 一样，这里的 native 指的是原生）：通过 Channel 实现完全异步的消息收发；
  + Channel 是 Flutter 里 Dart 和 native 通讯的通道，类似于一个网络连接，通过名字进行区分，是全双工的通道，创建 Channel 时使用相同的名字即可互相收发消息；
  + MethodChannel/FlutterMethodChannel 是专门用来实现类似 RPC 的通道；
  + BasicMessageChannel/FlutterBasicMessageChannel 可以用来实现普通的消息收发，全双工使用示例（[Dart 端](https://github.com/flutter/flutter/blob/291602db92cfd80725c758da3139f0c193658258/examples/flutter_view/lib/main.dart#L32)，[Android 端](https://github.com/flutter/flutter/blob/291602db92cfd80725c758da3139f0c193658258/examples/flutter_view/android/app/src/main/java/com/example/view/MainActivity.java#L60)，[iOS 端](https://github.com/flutter/flutter/blob/291602db92cfd80725c758da3139f0c193658258/examples/flutter_view/ios/Runner/MainViewController.m#L38)）
  + EventChannel/FlutterEventChannel 用来实现双向 Event 通讯（看起来和 MessageChannel 功能重叠）；

总结：Flutter 利用 Dart 代码可以预编译为机器码这一特性，实现了业务逻辑的跨平台；通过自己实现一套跨平台的 GUI 系统，实现了界面交互的跨平台。由于最终执行的代码都是机器码（C++，Dart），而且跨平台的实现基本不涉及 Dart 代码与 native 代码互相调用，所以 Flutter 的效率肯定是有保障的。在涉及到动画、手势等强交互 UI 特性时，Flutter 的性能应该比 RN 要高不少。Flutter 还自带了一套 Material Design 的跨平台控件库，简直良心。

Web 端呢？[Dart 可是能运行在浏览器里的呢！](https://www.dartlang.org/)

> Dart runs fast in every modern browser, on the command line, on servers, and on mobile.

不过遗憾的是，虽然 Dart 代码可以在浏览器运行，但却无法和 Flutter APP 共用 UI，[Flutter 团队也不打算开发 Web 支持](https://flutter.io/faq/#does-flutter-run-on-the-web)。

### J2ObjC + GWT

J2ObjC 和 GWT 都由 Google 开源，分别用来把 Java 代码编译为 ObjC 代码和 JS 代码，用来在 Android、iOS、Web 端共用业务逻辑代码，界面交互代码则需要单独开发。

这个特性简直就是 SDK 开发者的福音，事实上这也正是我在最近两个月在公司项目中实践的方案。这套跨平台方案没有什么炫酷的技术，直接把代码编译为各个平台开发语言的代码，简单粗暴但行之有效。

这里我简单总结下我在实践中感受到的一些要点：

+ 利用 `interface` 隔离 UI 操作；
+ 利用 `interface` 隔离其他 SDK 调用；
+ JSON 解析作为一个特殊的 SDK，除了要隔离，还有别的坑：ObjC 中可能需要手动实现序列化与反序列化，我们项目使用的 MJExtension 就是如此；
+ 支持 try-catch，会被翻译为 ObjC 的异常机制；
+ 循环引用问题：Java 内存回收做得比 ObjC 好，即便有循环引用，但如果整体不被其他对象引用，也可以回收，但 ObjC 就不行，所以要么代码里得显式设置 null 切断循环引用，要么就得用 `WeakReference`（`@Weak` 注解会被翻译为 `__unsafe_unretained`，用起来还是有些担心的）；
+ J2ObjC 的静态库 `libjre_emu.a` 有好几百兆，如果发布的 SDK 是静态库，那让开发者用户也携带这个 .a 就会比较恼人；

### Djinni + WebAssembly

Djinni 由 Dropbox 开源，它的核心思想是用 C++ 开发业务逻辑，利用 JNI、Objective-C++ 实现 C++ 和 Java、ObjC 的互相调用。

那这是 JNI 和 Objective-C++ 实现的跨平台呀，和 Djinni 有啥关系呢？做过 NDK 开发的朋友肯定知道，要实现 C++ 调用 Java 非常麻烦，boilerplate code 很多，Djinni 的发力点就在于减少 boilerplate code，声明跨边界的数据结构和接口，自动生成 C++、Java、ObjC 跨边界调用相关的代码，让我们可以聚焦于真正的业务逻辑开发中。

Djinni 生成代码的思路和 J2ObjC 编译代码的思路异曲同工，也都是简单粗暴的办法。接触到 Djinni 是半年前，最近趁着总结这篇文章的机会，简单实践了一把，还谈不上有太深的感悟，唯一一点是 Djinni 生成的 record 在跨边界时，是 immutable 的，因为很难保证跨边界状态维护，设计接口时需要考虑这一点。

这个方案是在 Dropbox 内部大量实践后再开源的，而且生成的代码并不多，稳定性不存在太多的风险，靠谱程度还是很高的。如果 SDK 的业务逻辑想在 C/C++ 代码中实现，那这套方案就是良策了。

理论上来说，我们的 C++ 代码也是可以编译为可以被 WebAssembly 加载的代码的，那在 Web 端使用应该也是可行的，当然，一层 JS wrapper 必不可少。

## Demo time

光说不练假把式，demo 自然是需要的。但逼近年关，公司项目太忙，只来得及做完后两种方案实现 Android iOS 跨平台的 demo，剩下的部分，只能来年再补了。

毕竟「done is better then perfect」

+ [JavaUniverse: A demo project that showcase how to use Java to conquer the universe, with the help of J2ObjC and GWT :)](https://github.com/Piasy/JavaUniverse)
+ [CppUniverse: A demo project that showcase how to use C++ to conquer the universe, with the help of Djinni and WebAssembly :)](https://github.com/Piasy/CppUniverse)

## 参考文章

React Native:

+ [Why React](https://github.com/reactjs/reactjs.org/blob/3c25e091555e24681e4571641742597e7ca990d1/docs/01-why-react.md)
+ [The Good and the Bad of ReactJS and React Native](https://www.altexsoft.com/blog/engineering/the-good-and-the-bad-of-reactjs-and-react-native/)
+ [Removing User Interface Complexity, or Why React is Awesome](http://jlongster.com/Removing-User-Interface-Complexity,-or-Why-React-is-Awesome)
+ [Communication between native and React Native](https://facebook.github.io/react-native/docs/communication-ios.html)
+ [Under the hood of React Native, by Martin Konicek](https://speakerdeck.com/mkonicek/under-the-hood-of-react-native?slide=25)
+ [React Native: Under the Hood, by Alexander Kotliarskyi](https://speakerdeck.com/frantic/react-native-under-the-hood?slide=13)
+ [Bridging in React Native: An in-depth look into React Native's core](https://tadeuzagallo.com/blog/react-native-bridge/)
+ [React Native Architecture Overview](https://speakerdeck.com/reactamsterdam/tadeu-zagallo-facebook-london-react-native-architecture-overview)

Flutter:

+ [FAQ: Technology](https://flutter.io/faq/#technology)
+ [Writing custom platform-specific code with platform channels](https://flutter.io/platform-channels/)
+ [Does Dart have any useful features for web programmers?](https://softwareengineering.stackexchange.com/a/164304/291269)
