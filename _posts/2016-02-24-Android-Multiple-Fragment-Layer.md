---
layout: post
title: 完美解决安卓上层输入控件随键盘弹起，下层 UI 变形问题
tags:
    - 安卓开发
    - 奇技淫巧
---

在YOLO的直播间内，可以发送文字评论，要求文字输入框随键盘弹起，而底下的视频又不会变形，也不会被顶上去，这个问题曾一度困扰我们很久，适逢大师兄公司安卓工程师也遇到了同样的问题，所以简单整理成一篇文章，供更多开发者参考。[本文源码地址](https://github.com/Piasy/AndroidPlayground/tree/master/effect/MultipleFragmentLayerDemo)。

## 面临的问题
主要还是activity的`windowSoftInputMode`选项只能设置一种值，如果希望输入框随着键盘弹起而顶上去，那底下的视图要么会顶上去，要么会变形（`adjustResize`），而希望底下的视图保持不动就成了一个伤脑筋的问题。

## 有图有真相

<img src="/img/11/multiple-fragment-demo.gif" alt="multiple-fragment-demo" style="height:400px">

## 解决思路
目前想到了三种思路：

1, 使用多层activity，底下用于播放视频，顶上用于其他交互信息以及文字输入框，顶上的activity设置背景透明，由于是两个单独的activity，它们响应键盘弹起的行为就可以独立开来了，但是两层activity的开销较大，所以并未采用过；

2, 一层activity，用于播放视频，其他交互信息以及文字输入框作为一个dialog fragment，由于dialog fragment响应键盘弹起的行为不受activity限制，所以同样可以达到想要的效果，YOLO曾一度使用过该方法，但是由于dialog fragment终究是一个dialog，它显示在窗口的最顶层，会带来其他UI层次的问题，最终废弃了这种方法；

3, 法2的改进版本，一层activity，但是使用多层fragment，最底层fragment播放视频，上层fragment放其他元素，而文字输入框单独放到一个dialog fragment里面，由于文字输入并非常态，它所在的dialog并不会长时间显示，所以不会带来负面效果，而activity的`windowSoftInputMode`值则设置为`adjustPan`，目前看来最完美了；
