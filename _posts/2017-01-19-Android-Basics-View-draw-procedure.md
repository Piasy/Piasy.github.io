---
layout: post
title: 安卓基础：View 绘制
tags:
    - 安卓开发
    - 基础知识
---

## View 树的绘图流程

当 Activity 接收到焦点的时候，它会被请求绘制布局，该请求由 framework 处理。

整个 View 树的绘图流程在ViewRoot.java类的performTraversals()函数展开，该函数所做 的工作可简单概况为是否需要重新计算视图大小(measure)、是否需要重新安置视图的位置(layout)、以及是否需要重绘(draw)，流程图如下：

![](https://imgs.piasy.com/2018-03-23-view_mechanism_flow.png)

## View 绘制流程函数调用链

![](https://imgs.piasy.com/2018-03-23-Android_view_draw_procedure.jpg)

有几点注意：

+ invalidate/postInvalidate 只会触发 draw；
+ requestLayout，会触发 measure、layout 和 draw 的过程；
+ 它们都是走的 `scheduleTraversals -> performTraversals`，用不同的标记位来进行区分；
+ resume 会触发 invalidate；
+ dispatchDraw 是用来绘制 child 的，发生在自己的 onDraw 之后，child 的 draw 之前

## Measure 和 Layout 的具体过程

![](https://imgs.piasy.com/2018-03-23-measure_layout.png)

## 参考文章

+ [公共技术点之 View 绘制流程](http://a.codekk.com/detail/Android/lightSky/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20View%20%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B)
+ [测试代码](https://github.com/Piasy/AndroidPlayground/tree/master/try/ViewDrawDemo)
