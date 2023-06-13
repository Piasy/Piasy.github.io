---
layout: post
title: 这个国庆假期我做了些什么？
---

今年国庆假期，我依然没有出去。那我在家做了些什么？

大抵就是三件事：；一口气翻译了六篇 Advanced RxJava 博客，完成了六周的量，目前这个系列还有 18 篇文章，预计 5 个月之后就能翻译完；花了 50 个小时，验证了 [AndroidTDDBootStrap](https://github.com/Piasy/AndroidTDDBootStrap){:target="_blank"} 在开启安卓新项目时的强大优势，怎么验证的？花了 50 个小时，就开发出了一个复杂的信息中心 APP 1.0 版本，架构先进，可维护性与可扩展性极佳。

> [https://github.com/Piasy/AndroidTDDBootStrap](https://github.com/Piasy/AndroidTDDBootStrap){:target="_blank"}

就在我以为这七天也就能做这么三件事的时候，世界又给了我一个大大的惊喜，具体细节暂且不表，接下来的几个月里，我将更努力地为之拼搏，我相信，来年国庆的时候，故事将会进入一个崭新的篇章。

_好了，自我感觉良好阶段结束，下面具体分享几个东西 :)_

## 博客

从去年国庆假期搭建起技术博客，转眼一年已经过去了。在这一年里，我也算勤恳，一共完成了 35 篇原创文章，平均每月 3 篇。

前期的文章比较简单，大抵都是“解决了什么问题 -> 解决问题的过程”这样的思路，但也不用觉得羞愧，圈内有一位热爱分享的前辈，Chiu-Ki Chan，

> [http://chiuki.github.io/](http://chiuki.github.io){:target="_blank"}

我看过她很多文章和演讲，她就鼓励大家多写写，多讲讲，即便内容不高深，分享的精神以及解决问题的思路，都是很有价值的。

前期的文章中，我觉得还是有几篇大家看了不会骂我的：

+ [（可能是）目前最全面的Android Espresso配置指南了](/2016/03/13/Android-Espresso-test-start/index.html){:target="_blank"}
+ [从 A/Looper: Could not create epoll instance. errno=24 错误浅谈解决各种 bug 的思路](/2016/03/16/Looper-crash/index.html){:target="_blank"}
+ [深入理解 RecyclerView 系列之一：ItemDecoration](/2016/03/26/Insight-Android-RecyclerView-ItemDecoration/index.html){:target="_blank"}
+ [深入理解 RecyclerView 系列之二：ItemAnimator](/2016/04/04/Insight-Android-RecyclerView-ItemAnimator/index.html){:target="_blank"}
+ [Dagger2 Scope 注解能保证依赖在 component 生命周期内的单例性吗？](/2016/04/11/Dagger2-Scope-Instance/index.html){:target="_blank"}
+ [安卓 OpenGL ES 2.0 完全入门（一）：基本概念和 hello world](/2016/06/07/Open-gl-es-android-2-part-1/index.html){:target="_blank"}
+ [安卓 OpenGL ES 2.0 完全入门（二）：矩形、图片、读取显存等](/2016/06/14/Open-gl-es-android-2-part-2/index.html){:target="_blank"}

后来，我花了更多的时间在 APP 架构方面的思考，以及必备开源库的源码导读这两件事情上面，产出的文章我还是觉得值得一看的。

架构系列：

+ [完美的安卓 model 层架构（上）](/2016/05/06/Perfect-Android-Model-Layer/index.html){:target="_blank"}
+ [完美的安卓 model 层架构（下）](/2016/05/12/Perfect-Android-Model-Layer-2/index.html){:target="_blank"}
+ [RESTful 安卓网络层解决方案（一）：概览与认证实现方案](/2016/08/29/RESTful-Android-Network-Solution-1/index.html){:target="_blank"}
+ [RESTful 安卓网络层解决方案（二）：空 JSON 和 API Error 解析](/2016/09/04/RESTful-Android-Network-Solution-2/index.html){:target="_blank"}
+ [RESTful 安卓网络层解决方案（三）：API model 与 Business model 分离](/2016/09/04/RESTful-Android-Network-Solution-3/index.html){:target="_blank"}

拆轮子系列：

+ [拆轮子系列：拆 Retrofit](/2016/06/25/Understand-Retrofit/index.html){:target="_blank"}
+ [拆轮子系列：拆 OkHttp](/2016/07/11/Understand-OkHttp/index.html){:target="_blank"}
+ [拆轮子系列：拆 Okio](/2016/08/04/Understand-Okio/index.html){:target="_blank"}
+ [拆轮子系列：拆 RxJava](/2016/09/15/Understand-RxJava/index.html){:target="_blank"}

很多前辈都说过，教是最好的学，经过这一年多的实践，我也深感如此。**当然，纸上得来终觉浅，别人教的都不是自己的，只能作为一个参考，知识最终都还得经过自己的思考、实践，才能沉淀为自己的能力**。

最后，披露两张 Analytics 的数据：

![blog_google_analytics_201510_2016_10.png](/img/201610/blog_google_analytics_201510_2016_10.png)

![blog_baidu_analytics_201603_2016_10.png](/img/201610/blog_baidu_analytics_201603_2016_10.png)

访问最多的一天是在 7 月 14 号，“拆轮子系列：拆 OkHttp”。

Analytics 我同时集成了 Google 和百度，百度各项指标基本是 Google 的两倍，看来还有一半的朋友在墙内，希望 GitHub 能坚持住 :)

过去一年里，大约有五万多位朋友阅读了我的文章近十二万次，谢谢大家的支持！

## Advanced RxJava

翻译这个系列博客的初衷，是为了让自己能把这些文章的内容看进去，顺带也让更多的朋友可以尽量避免语言上导致的困难，当然第二点我只能说尽量。

这个系列一共四十多篇，目前已经翻译了 28 篇，还剩 18 篇，预计还要 5 个月。

在这个过程中，我自己的收获还是很大的，RxJava 非常庞大，先以这样的方式从细微之处开始窥探它的奥妙之处，也不失为一种办法。在这个基础上，中秋假期期间，我一口气捋了一遍 RxJava 主要流程的代码，写出了“拆轮子系列：拆 RxJava”。

RxJava 在国内的普及率依然不高，我想原因主要有以下几点：

+ 学习成本较高，很多人有抵触情绪；
+ 比较庞大，如果团队中只有一个人、一小部分代码使用 RxJava，感觉不值当；
+ 完全迁移到使用 RxJava 又不太现实；

在我的工作经验中，安装包体积、multidex 这些我都没有深入搞过，因为这两件事对我们的产品并不重要，只有到了上亿用户的量级，才需要处处严谨。如果有些朋友的项目也和我们类似，就完全可以先引入进来，再逐步扩散。

当然，我们需要明确 RxJava 的应用场景：复杂的数据流、异步。这一点至关重要。

## AndroidTDDBootStrap

这个项目我从去年一直在维护，目标是作为开启新项目时的一个良好开端，其中包含了一些我在实际工作中的常用框架、最佳实践、解决方案，包括两个实际商业项目的验证和反馈。

今后只要我还在安卓开发领域，都会持续改进，欢迎大家关注！

## 写在最后

天道酬勤。共勉。
