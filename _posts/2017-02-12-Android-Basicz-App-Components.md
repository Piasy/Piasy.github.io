---
layout: post
title: 安卓基础：App 各大组件
tags:
    - 安卓开发
    - 基础知识
---

## Activity

Activity 是和用户交互的入口，几乎用户所有的交互操作都发生在 Activity 中，只有一个例外：通知栏，notification area (bar) 和 notification drawer 都由系统控制（SystemUI，一系列有界面的 Service，除了状态栏，还包括虚拟按键、最近任务、壁纸等，更多内容可以阅读[《深入理解Android 卷III》](https://book.douban.com/subject/26598458)）。桌面组件（widget）也是被 launcher 利用，显示在 launcher 的 Activity 中。

后来引入了 Fragment 以增强交互界面的复用，同时优化性能。Fragment 必须要有 host Activity。不需要单独入口的交互界面，都应该用 Fragment 实现，谷歌官方强调，Fragment over Activity。但也要按照业务逻辑把 Fragment 分成不同的组，每组一个 Activity，例如登录注册一个组，用户资料设置一个组，即便注册流程有一步资料设置，也不应该把它揉进登录中，可以通过 startActivities 实现返回导航。

Activity 的一些要点：

+ [安卓基础：task, launchMode, Intent flag](/2017/01/16/Android-Basics-Task-and-LaunchMode/index.html)
+ [安卓基础：Activity/Fragment 生命周期](/2017/01/14/Android-Basics-Activity-Fragment-Life-Cycle/index.html)
+ [安卓基础：Activity/Fragment 销毁与重建](/2017/01/15/Android-Basics-Activity-Fragment-Kill-and-Recreate/index.html)
+ 安卓基础：Activity 启动过程，TODO
+ 安卓基础：Activity/Fragment 导航，TODO
+ 安卓基础：Activity/Fragment 过场动画，TODO

## Service

Service 的标准使用场景：后台任务且即使用户离开本 app 仍要继续运行。如果后台任务限制在本 app 运行期间，那也不应该使用 Service，而应该使用其他异步框架。一个典型的例子，同样是播放音乐，播放器就应该在用户使用其他 app 时也继续播放，而微信语音的播放，就应该在 onPause 时停止播放，尽管两者都需要启动新的线程播放音乐。

线程、Service、RxJava 等异步框架，各自优劣、如何取舍？

线程、RxJava 都不是安卓平台特有的，用它们势必要自己额外做些工作。实际上，Handler、Service 要完成异步工作，靠的还是线程/进程，但是它们默认是没有异步能力的，Handler 在哪个线程创建，处理消息就在哪个线程，而 Service 固定运行在主线程。
根据场景我们就能做出取舍了，在不建议使用 Service 的场景中，我们当然应该用一些现代化、高级一点的方式，例如 RxJava。

Service 一些其他的要点：

+ 分类/形式：启动和绑定；
+ 启动，startService，onStartCommand；自己决定何时结束；无法直接和调用组件通信，可以用 EventBus/Toast/Notification；
+ 绑定，bindService，onBind；多个组件可以绑定到同一个 Service，所有与之绑定的组件都结束后，Service 也将结束；通过 Binder 和调用组件通信，支持 IPC；
+ 两种模式可以混用，结束时机仍由自己控制，绑定只是使得调用组件可以直接通信；
+ Service 有可能被系统杀死以回收资源，之后可能会被系统重启；前台 Service 优先级最高，会有一个通知栏消息，绑定到前台 Activity 的 Service 优先级较高，可以声明被杀死后是否重启；
+ 被系统杀死是以进程为单位的，进程优先级关系：前台进程，可见进程（Activity onPause 之后 onStop 之前仍显示在屏幕上），服务进程，后台进程（onStop 之后的不可见的 Activity），空进程；
+ 提供服务的进程优先级不低于被服务的进程；

## ContentProvider

提供类似数据库查询的功能，使用过获取通讯录的服务，但我尚未实现过。

## BroadcastReceiver

接收广播，有两种注册方式：manifest 声明，或利用 Context 对象动态注册。manifest 注册方式会在 app 安装之后就生效（实测进程被 force stop 之后不会收到，安装后未启动也收不到），动态注册的有效期和注册时的 Context 对象生命周期一致（Activity onDestroy 之前，Application 被杀死之前），但要注意取消注册，避免内存泄漏（Receiver）。
onReceive 函数在主线程被调用，返回后系统就会认为 Receiver 所在的进程可以杀死了，如果需要异步，要先调用 goAsync，并在工作完成后 finish。

最佳实践：

+ 如果不需要把广播发送给 app 外的组件，可以利用 LocalBroadcastManager，更高效、安全；
+ 优先使用 Context 注册方式，避免过多 app 注册同一广播，导致同时启动大量 app 带来性能和体验问题；
+ 不要在隐式广播的 Intent 中包含敏感信息，可能被其他 app 监听到；可以考虑为广播设置权限、包名，或者使用 LocalBroadcastManager；
+ 收到的广播可能来自恶意程序（恶意的内容，或者恶意的发广播）；可以考虑为广播设置权限，manifest 声明的广播可以设置 android:exported=false，也可以使用 LocalBroadcastManager；
+ 广播名字是全局的，注意避免冲突；
+ 不要在 onReceive 中启动 Activity，可能会导致卡顿，考虑发一个通知栏消息；

## 杂

### 应用和进程的关系？

一个虚拟机只能跑一个进程，一个进程里可以跑多个应用（[声明相同的 android:shareUserID 和 android:process，并用同一个 key 签名](http://stackoverflow.com/a/17664341)），一个应用也可以跑在多个进程中（会有多个 Application 实例）。

### 线程和上面这些东西的关系？

线程除了和进程说得上有关系，和其他概念都谈不上关系。至于线程和进程的关系，请看 安卓基础：线程与进程，TODO。
