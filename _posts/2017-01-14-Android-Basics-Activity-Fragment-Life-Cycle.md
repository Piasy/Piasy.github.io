---
layout: post
title: 安卓基础：Activity/Fragment 生命周期
tags:
    - 安卓开发
    - 基础知识
---

## 看图

developer 官方图：

![2017010899959activity_lifecycle.png](https://imgs.babits.top/2017010899959activity_lifecycle.png)  ![201701084431fragment_lifecycle.png](https://imgs.babits.top/201701084431fragment_lifecycle.png)

GitHub 神图：

![2017010890963complete_android_fragment_lifecycle.png](https://imgs.babits.top/2017010890963complete_android_fragment_lifecycle.png)

## 回答问题

以下过程均是针对 AppCompatActivity 和 support Fragment，测试代码可以在 [GitHub 获取](https://github.com/Piasy/android-lifecycle)。

### 单个 Activity + Fragment 的启动？

~~~ java
Activity onCreate
Fragment onAttach
Fragment onCreate
Fragment onCreateView
Fragment onViewCreated

Activity onStart
// 下面两个由 super.onStart 触发
Fragment onActivityCreated
Fragment onStart

Activity onResume

Activity onPostResume
// 下面一个由 super.onPostResume 触发
Activity onResumeFragments
// 下面一个由 super.onResumeFragments 触发
Fragment onResume
~~~

### A 启动 B，A 和 B 的生命周期函数调用顺序？

~~~ java
A onPause
B onCreate
B onStart
B onResume
A onStop
~~~

onPause 之后，Activity 依然是可见的，onStop 之后，Activity 就不可见了；onResume 之后，Activity 才是可见的；A 启动 B 的时候，先让 B onResume，再让 A onStop，这就避免了“无可见 Activity”的问题。

### Aa 的 Fa 启动 Ab 的 Fb，生命周期函数调用顺序？

~~~ java
// 启动 B
Aa onPause
// 下面一个由 super.OnPause 触发
Fa onPause

// 接下来同单个 Activity + Fragment 的启动流程
Ab onCreate
...
Fb onResume

Aa onStop
// 下面一个由 super. onStop 触发
Fa onStop

// B 返回
Ab onPause
Fb onPause

Aa onRestart
Aa onStart
Fa onStart
Aa onResume
Aa onPostResume
Aa onResumeFragments
Fa onResume

Ab onStop
Fb onStop
Ab onDestroy
Fb onDestroyView
Fb onDestroy
Fb onDetach
Ab onDestroy
~~~

### Fragment transaction 时，Fragment 的生命周期

+ add Fragment 并不会导致原本就在 id 指定的 ViewGroup 上的 Fragment onPause；
+ replace Fragment 会导致原本就在 id 指定的 ViewGroup 上的 Fragment onPause, onDestroy, onDetach，且这一切都发生在新 Fragment onResume 之后；
+ remove 的效果和 replace 一样，只不过不会新加 Fragment；

### 非 support 组件的过程

基本一致，但是不存在 Activity call super 的时候触发 Fragment 的生命周期函数，而是正向 Activity 在前，Fragment 在后，反向是 Fragment 在前，Activity 在后（并不完全这样，详见附录）。

## [附录：完整 logcat](/2017/01/14/Android-Basics-Activity-Fragment-Life-Cycle-Appendix/)
