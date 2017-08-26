---
layout: post
title: RxComboDetector：Android view点击“连击”检测
tags:
    - 安卓开发
    - MISC
    - Reactive eXtention
---

今天迷迷糊糊听见iOS同事对PM说“连击检测”其实只需要传一个参数就行了，我大为震惊，iOS竟有如此炫酷的API，Android似乎没有呀！在google和github搜索两次之后无果，我决定自己实现一个连击检测的库。因为主要使用RxJava实现，所以命名为`RxComboDetector`，[github 源码地址](https://github.com/Piasy/RxComboDetector)。

## 效果有图有真相

<img src="/img/8/combo-demo.gif" alt="combo-demo" style="height:400px">

## 原理

基本思想非常简单，如果本次点击事件发生的时间，距离上次点击事件之间的时间差小于某个阈值，就判定为属于连击。

核心代码如下：

~~~ java
static Observable<Integer> detect(Observable<Void> clicks, final long maxIntervalMillis,
        final int minComboTimesCared) {
    return clicks.map(new Func1<Void, Integer>() {
            @Override
            public Integer call(Void aVoid) {
                return 1;
            }
        }).timestamp()
        .scan(new Func2<Timestamped<Integer>, Timestamped<Integer>, Timestamped<Integer>>() {
            @Override
            public Timestamped<Integer> call(Timestamped<Integer> lastOne,
                    Timestamped<Integer> thisOne) {
                if (thisOne.getTimestampMillis() - lastOne.getTimestampMillis() <=
                        maxIntervalMillis) {
                    return new Timestamped<>(thisOne.getTimestampMillis(),
                            lastOne.getValue() + 1);
                } else {
                    return new Timestamped<>(thisOne.getTimestampMillis(), 1);
                }
            }
        }).map(new Func1<Timestamped<Integer>, Integer>() {
            @Override
            public Integer call(Timestamped<Integer> timestamped) {
                return timestamped.getValue();
            }
        }).filter(new Func1<Integer, Boolean>() {
            @Override
            public Boolean call(Integer combo) {
                return combo >= minComboTimesCared;
            }
        });
}
~~~

具体实现上采用了多个RxJava的operator：

+  利用[RxBinding](https://github.com/JakeWharton/RxBinding)，把View的点击事件转化为`Void`事件流，这里并未直接依赖RxBinding库，而是把View点击事件相关的两个类摘了出来，以避免多余的依赖；当然源码中加入了相应的版权声明（Apache V2）；
+  利用[`map`](http://reactivex.io/documentation/operators/map.html)操作符，把`Void`转化为`1`，表示1次连击；
+  利用[`timestamp`](http://reactivex.io/documentation/operators/timestamp.html)操作符，为每次点击事件加上时间戳；
+  利用[`scan`](http://reactivex.io/documentation/operators/scan.html)操作符，检查本次点击事件和上次点击事件的时间戳，并决定是否属于连击，设置连击次数加1，或者重置为1；这里和Rx官方文档的例子用法不完全一样，文档中是以递归求和举的例子，但是scan操作符正好可以保留本次和上次发射的两个对象，完全符合这里的需求，所以采用；
+  再次利用[`map`](http://reactivex.io/documentation/operators/map.html)操作符，把`Timestamped<Integer>`转化为`Integer`；
+  利用[`filter`](http://reactivex.io/documentation/operators/filter.html)操作符，过滤掉连击次数小于目标值的事件；

## 实现细节

`RxComboDetector`的创建采用builder模式进行创建，完整使用代码如下：

~~~ java
new RxComboDetector.Builder()
    .maxIntervalMillis(500)
    .minComboTimesCared(3)
    .detectOn(mBtnCombo)
    .start()
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer combo) {
            // do something
        }
    });
~~~

使用builder来配置连击判断最大时间间隔，以及过滤掉低连击事件的阈值。

此外，上一小节中描述的operator操作代码并不在`start`函数中，而是封装在了一个static的`detect`方法中，这是为了便于进行测试，一方面和View的点击事件解耦，逻辑完全是Rx事件流的操作，只需mock事件源即可；另一方面也无需mock构造`RxComboDetector`时的View参数，简化测试代码。

最后，`RxComboDetector`只是一个事件的检测类，并没有炫酷的UI效果，所以我使用了[rebound](https://github.com/facebook/rebound)库来实现具有物理真实感的View动画效果。

最后的最后，demo中的小黄笑脸来自[YOLO](https://www.yoloyolo.tv/)的logo。
