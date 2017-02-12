---
layout: post
title: 安卓基础：事件传递及滑动冲突的处理
tags:
    - 安卓开发
    - 基础知识
---

## 基本概念

+ 所有 Touch 事件都被封装成了 MotionEvent 对象，包括 Touch 的位置、时间、历史记录以及第几个手指（多点触控）等；
+ 事件有很多类型，`ACTION_DOWN`，`ACTION_UP`，`ACTION_MOVE` 等；
+ 对事件的处理包括三类：传递，dispatchTouchEvent()；拦截，onInterceptTouchEvent()；消费，onTouchEvent() OnTouchListener；

## 传递过程

+ 事件从 Activity.dispatchTouchEvent() 开始传递，只要没有被拦截，就会沿着最上层的 ViewGroup 开始一直往下（子 View/ViewGroup）传递，当然这个从上往下是指 View 树的结构上，从 View 在界面的层次上来说是从底层往顶层传递；
+ 父 ViewGroup 可以通过 onInterceptTouchEvent() 对事件做拦截，阻止其往下传递；
+ 如果未被拦截，则子 View 可以通过 onTouchEvent() 消费（处理）事件；
+ 如果事件从上往下传递过程中一直没有被拦截，且最底层子 View 没有消费事件，事件会反向往上传递，这时父 ViewGroup 可以在 onTouchEvent() 中消费该事件，如果还是没有被消费的话，最后会到 Activity 的 onTouchEvent() 函数；
+ 如果 View 没有对 `ACTION_DOWN` 进行消费，此次点击的后续事件不会传递过来（TODO：怎么实现的？）；
+ 如果 View 消费了 `ACTION_DOWN `，此次点击的后续事件会直接给这个 View（TODO：怎么实现的？），但其父 ViewGroup 的 onIntercept 函数仍会被调用，仍能进行拦截，但它自己的 onIntercept 不会被调用了；
+ 但是 View 可以在 onTouchEvent 中调用 `getParent().requestDisallowInterceptTouchEvent(true)`，这样父 ViewGroup 的 onIntercept 在后续的事件中就不会被调用了；
+ 但要搞清楚，第一个事件是先调用 ViewGroup 的 onIntercept 的，所以如果一开始 ViewGroup 就打算拦截，子 View 将没有任何机会；
+ OnTouchListener 优先于 onTouchEvent() 对事件进行消费；
+ 消费就是指相应的函数返回 true；

上面有两个“短路逻辑”，一旦子 View 对 DOWN 事件不感兴趣，后续事件就再也不会传过来，而一旦感兴趣，后续事件将直接传过来。这个逻辑是怎么实现的？我暂时不打算深入研究，欢迎了解的朋友指出来。

## 滑动冲突

什么是滑动冲突？就是父 View 和子 View 都需要处理滑动，例如父 View 需要左右滑动，子 View 需要上下滑动（ViewPager 嵌套 RecyclerView），一个点击事件，到底交给谁处理？

知晓了点击事件的传递和处理机制之后，处理滑动冲突其实就很简单了：首先我们需要定义好处理规则（如果我们都不清楚应该怎么处理，怎么能写好代码让程序处理呢，对吧？），然后我们在父 View 的 onIntercept、子 View 的 onTouchEvent 以及父 View 的 onTouchEvent 函数中实现我们定义的规则即可。例如父 View 的 onIntercept 中，如果发现是左右滑动，那就拦截，否则不拦截。

## 参考文章

+ [公共技术点之 View 事件传递](http://a.codekk.com/detail/Android/Trinea/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20View%20%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92)
+ [listview所带来的滑动冲突](http://blog.csdn.net/singwhatiwanna/article/details/8863232)
+ [测试代码](https://github.com/Piasy/AndroidPlayground/tree/master/try/ViewDrawDemo)
