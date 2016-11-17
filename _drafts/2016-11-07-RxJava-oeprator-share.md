---
layout: post
title: RxJava share 时，不 subscribeOn，第二个收不到数据
tags:
    - 安卓开发
    - 拆轮子
    - 原理剖析
---

share 也是利用的 publish，只不过加了个 refCount

先搞清楚 publish 的原理：

+ `OperatorPublish` 会保存上游 `source`、当前的 `PublishSubscriber` `current`
+ 这里先不考虑并发相关的问题，全都当成单线程
+ 它的 `onSubscribe` 会在 `call` 中新建一个 `PublishSubscriber`，它是 parent，会被设置给 current，传入的（实际 subscribe 的）subscriber 是 child，它们有一个转发关系，在 `OperatorPublish.InnerProducer` 中实现；转发的代码实现比较复杂，简单认为 source -> parent -> child 即可；`PublishSubscriber` 有一个数据队列，如果 child 还没来，会把数据放到队列中，来了之后会发给 child；
+ `OperatorPublish#connect` 中，有一个对并发调用 connect 的处理逻辑，

akarnokd.blogspot.com/2015/10/connectableobservables-part-1.html
