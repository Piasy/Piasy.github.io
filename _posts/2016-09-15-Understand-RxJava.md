---
layout: post
title: 拆轮子系列：拆 RxJava
tags:
    - 安卓开发
    - 拆轮子
    - 原理剖析
---

[RxJava](https://github.com/ReactiveX/RxJava/){:target="_blank"} 这个项目已经持续四年半了，第一个 commit 是在 2012 年 3 月 18 号。我从 14 年 11 月份开始使用 RxJava，应该算是比较早的，将近两年过去了，现在 RxJava 1.x 版本已经进入稳定期，2.0 版本也已经进入了 RC 阶段。

原本打算把 Advanced RxJava 系列博客翻译完之后再拆 RxJava 的，但是前两周看了一个 [JW 讲 RxJava 的视频](http://jakewharton.com/demystifying-rxjava-subscribers/){:target="_blank"}，突然有种隐隐打通任督二脉的感觉，索性趁着中秋佳节，一鼓作气把 RxJava 好好拆开看个究竟。本文的分析基于 [RxJava 截至 2016.9.16 的最新源码](https://github.com/ReactiveX/RxJava/tree/010256b0e19bdb18f136f80a4dc48c0a53c3da41){:target="_blank"}，非常建议大家下载 RxJava 源码之后，跟着本文，过一遍源码。

## 1，整体思路

拆轮子这也是第四回了，套路也算得到了很好的验证，顺着常用的场景/用例出发，理解整个过程、结构、原理，不要沉迷于细节，先对常用的内容有一个全局的概览，每一块的细节再按需深入。入手新项目也是这个思路。

对 RxJava 来说，基于目前已有的认识，我觉得主要应该抓住四个方面：

+ 事件流源头（observable）怎么发出数据
+ 响应者（subscriber）怎么收到数据
+ 怎么对事件流进行操作（operator/transformer）
+ 以及整个过程的调度（scheduler）

另外还有三点也值得一提：

+ backpressure
+ hook
+ 测试

## 2，`Hello world`

我们先看一个最简单的 Hello world 例子：

~~~ java
Observable.just("Hello world")
        .subscribe(word -> {
            System.out.println("got " + word + " @ " 
                    + Thread.currentThread().getName());
        });
~~~

### 2.1，`just`

逐行往下看显然是最自然的方式，那我们先看看 `just()`：

~~~ java
// Observable.java
public static <T> Observable<T> just(final T value) {
    return ScalarSynchronousObservable.create(value);
}

// ScalarSynchronousObservable.java
public static <T> ScalarSynchronousObservable<T> create(T t) {
    return new ScalarSynchronousObservable<T>(t);           // 1
}

protected ScalarSynchronousObservable(final T t) {
    super(RxJavaHooks.onCreate(new JustOnSubscribe<T>(t))); // 2
    this.t = t;
}
~~~

这里一定要注意，不要沉迷于细节，否则上万行的代码绝不是一两天能看出个大概来的。

1. 我们创建的是 `ScalarSynchronousObservable`，一个 `Observable` 的子类。
2. 我们先跳过 `RxJavaHooks`，从名字可以得知它是用来做一些 hook 的工作的，那我们就先认为它什么也不做。所以我们传给父类构造函数的就是 `JustOnSubscribe`，一个 `OnSubscribe` 的实现类。

`Observable` 的构造函数接受一个 `OnSubscribe`，它是一个回调，会在 `Observable#subscribe` 中使用，用于通知 observable 自己被订阅，它是怎么使用的，我们马上就能看到。

### 2.2，`subscribe`

我们接着看 `subscribe()`：

~~~ java
public final Subscription subscribe(final Action1<? super T> onNext) {
    // 省略参数检查代码
    Action1<Throwable> onError = 
        InternalObservableUtils.ERROR_NOT_IMPLEMENTED;
    Action0 onCompleted = Actions.empty();
    return subscribe(new ActionSubscriber<T>(onNext, 
        onError, onCompleted));                             // 1
}

public final Subscription subscribe(Subscriber<? super T> subscriber) {
    return Observable.subscribe(subscriber, this);
}

static <T> Subscription subscribe(Subscriber<? super T> subscriber, 
      Observable<T> observable) {
    // 省略参数检查代码
    subscriber.onStart();                                   // 2
    
    if (!(subscriber instanceof SafeSubscriber)) {
        subscriber = new SafeSubscriber<T>(subscriber);     // 3
    }

    try {
        RxJavaHooks.onObservableStart(observable, 
            observable.onSubscribe).call(subscriber);       // 4
        return RxJavaHooks.onObservableReturn(subscriber);  // 5
    } catch (Throwable e) {
        // 省略错误处理代码
    }
}
~~~

我们抓住主要逻辑：

1. 我们首先对传入的 Action 进行包装，包装为 `ActionSubscriber`，一个 `Subscriber` 的实现类。
2. 调用 `subscriber.onStart()` 通知 subscriber 它已经和 observable 连接起来了。这里我们就知道，`onStart()` 就是在我们调用 `subscribe()` 的线程执行的。
3. 如果传入的 subscriber 不是 `SafeSubscriber`，那就把它包装为一个 `SafeSubscriber`。
4. 我们再次跳过 hook，认为它什么也没做，那这里我们调用的其实就是 `observable.onSubscribe.call(subscriber)`。这里我们就看到了前面提到的 `onSubscribe` 的使用代码，在我们调用 `subscribe()` 的线程执行这个回调。
5. 跳过 hook，那么这里就是直接返回了 `subscriber`。`Subscriber` 继承了 `Subscription`，用于取消订阅。

关于 `SafeSubscriber`，我们跳过源码，直接看它的文档，其中说明了这个类的作用：保证 `Subscriber` 实例遵循 [Observable contract](http://reactivex.io/documentation/contract.html){:target="_blank"}。至于具体怎么保证的，以及 contract 的内容，大家直接看文档即可，这里不再赘述。

好了，我们已经看完了例子中两个调用的代码，但是 `Hello world` 是怎么被传递到打印的代码里的呢？别急，就在 `observable.onSubscribe.call(subscriber)` 中。

### 2.3，`OnSubscribe`

还记得 `just()` 的实现中，我们创建了一个 `JustOnSubscribe` 吗？这里我们执行的就是它实现的 `call()` 函数：

~~~ java
// ScalarSynchronousObservable.java
static final class JustOnSubscribe<T> implements OnSubscribe<T> {
    // ...

    @Override
    public void call(Subscriber<? super T> s) {
        s.setProducer(createProducer(s, value));
    }
}

static <T> Producer createProducer(Subscriber<? super T> s, T v) {
    // ...
    return new WeakSingleProducer<T>(s, v);
}
~~~

这里我们就是为 subscriber 设置了一个 `WeakSingleProducer`。

在 RxJava 1.x 中，数据都是从 observable push 到 subscriber 的，但要是 observable 发得太快，subscriber 处理不过来，该怎么办？一种办法是，把数据保存起来，但这显然可能导致内存耗尽；另一种办法是，多余的数据来了之后就丢掉，至于丢掉和保留的策略可以按需制定；还有一种办法就是让 subscriber 向 observable 主动请求数据，subscriber 不请求，observable 就不发出数据。它俩相互协调，避免出现过多的数据，而协调的桥梁，就是 producer。producer 的内容这里不展开，大家可以看 ReactiveIO 的文档，或者 [Advanced RxJava](/AdvancedRxJava/){:target="_blank"} 这个系列博客。

### 2.4，`setProducer`

我们接着看 `setProducer()` 的实现：

~~~ java
// Subscriber.java
public void setProducer(Producer p) {
    long toRequest;
    boolean passToSubscriber = false;
    synchronized (this) {
        toRequest = requested;
        producer = p;
        if (subscriber != null) {             // 1
            if (toRequest == NOT_SET) {
                passToSubscriber = true;
            }
        }
    }
    if (passToSubscriber) {                   // 2
        subscriber.setProducer(producer);
    } else {
        if (toRequest == NOT_SET) {           // 3
            producer.request(Long.MAX_VALUE);
        } else {
            producer.request(toRequest);
        }
    }
}
~~~

这里逻辑比较复杂，但是我们理清我们当前所处的状态，就简单了：

1. 我们这里确实有一层包装，`ActionSubscriber` -> `SafeSubscriber`。
2. 所以这里我们会发生一次 pass through，然后我们会进入 else 代码块。
3. 这里所有的 `requested` 初始值都是 `NOT_SET`，所以我们会请求 `Long.MAX_VALUE`，即无限个数据。

### 2.5，`request`

那我们再看 `WeakSingleProducer#request()` 的实现：

~~~ java
// ScalarSynchronousObservable.java
static final class WeakSingleProducer<T> implements Producer {
    // ...
    
    @Override
    public void request(long n) {
        // 省略状态检查代码
        Subscriber<? super T> a = actual;
        if (a.isUnsubscribed()) {
            return;
        }
        T v = value;
        try {
            a.onNext(v);
        } catch (Throwable e) {
            Exceptions.throwOrReport(e, a, v);
            return;
        }

        if (a.isUnsubscribed()) {
            return;
        }
        a.onCompleted();
    }
}
~~~

我们看到，在 `request()` 中，终于调用了 subscriber 的 `onNext()` 和 `onCompleted()`，那么，`Hello world` 就传递到了我们的 Action 中，并被打印出来了。

### 2.6，完整的过程

到这里我们就已经梳理出完整的调用过程了：

![RxJava_call_stack_just.png](/img/201609/RxJava_call_stack_just.png)

一切行为都由 subscribe 触发，而且都是直接的函数调用，所以都在调用 subscribe 的线程执行。

下面我们看一下调试时的调用栈：

![rxjava_sync_subscribe_call_stack.jpg](/img/201609/rxjava_sync_subscribe_call_stack.jpg)

确实和我们的分析结果一致。

## 3，操作符

我们把 Hello world 稍微变复杂一点，使用一个操作符：

~~~ java
Observable.just("Hello world")
        .map(String::length)
        .subscribe(word -> {
            System.out.println("got " + word + " @ "
                    + Thread.currentThread().getName());
        });
~~~

我们使用了一个 `map` 操作符，把字符串转换为它的长度。

### 3.1，`map`

~~~ java
// Observable.java
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return create(new OnSubscribeMap<T, R>(this, func));
}

public static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(RxJavaHooks.onCreate(f));
}
~~~

这里有两个小插曲，一是 `map` 的实现本来是利用 `lift` + `Operator` 实现的，但是后来改成了 `create` + `OnSubscribe`（[RxJava #4097](https://github.com/ReactiveX/RxJava/pull/4097){:target="_blank"}）；二是 `lift` 的实现本来是直接调用 observable 构造函数，后来改成了调用 `create`（[RxJava #4007](https://github.com/ReactiveX/RxJava/pull/4007){:target="_blank"}）。后者先发生，引入了新的 hook 机制，前者则是为了提升一点性能。

所以这里实际上是 `OnSubscribeMap` 干活了。

### 3.2，`OnSubscribeMap`

那我们看看 `OnSubscribeMap`：

~~~ java
public final class OnSubscribeMap<T, R> implements OnSubscribe<R> {

    final Observable<T> source;
    
    final Func1<? super T, ? extends R> transformer;

    public OnSubscribeMap(Observable<T> source, 
            Func1<? super T, ? extends R> transformer) {
        this.source = source;
        this.transformer = transformer;
    }
    
    @Override
    public void call(final Subscriber<? super R> o) {
        MapSubscriber<T, R> parent = 
            new MapSubscriber<T, R>(o, transformer);   // 1
        o.add(parent);                                 // 2
        source.unsafeSubscribe(parent);                // 3
    }
}
~~~

它的实现很直观：

1. 利用传入的 subscriber 以及我们进行转换的 Func1 构造一个 `MapSubscriber`。
2. 把一个 subscriber 加入到另一个 subscriber 中，是为了让它们可以一起取消订阅。
3. `unsafeSubscribe` 相较于前面的 `subscribe`，可想而知就是少了一层 `SafeSubscriber` 的包装。为什么不要包装？因为我们会在最后调用 `Observable#subscribe` 时进行包装，只需要包装一次即可。

转换的代码依然没有出现，它在 `MapSubscriber` 中。

### 3.3，`MapSubscriber`

~~~ java
static final class MapSubscriber<T, R> extends Subscriber<T> {
    
    final Subscriber<? super R> actual;
    
    final Func1<? super T, ? extends R> mapper;

    boolean done;
    
    public MapSubscriber(Subscriber<? super R> actual, 
            Func1<? super T, ? extends R> mapper) {
        this.actual = actual;
        this.mapper = mapper;
    }
    
    @Override
    public void onNext(T t) {
        R result;
        
        try {
            result = mapper.call(t);        // 1
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            unsubscribe();
            onError(OnErrorThrowable.addValueAsLastCause(ex, t));
            return;
        }
        
        actual.onNext(result);              // 2
    }
    
    // 省略 onError，onCompleted 和 setProducer
}
~~~

`MapSubscriber` 依然很直观：

1. 上游每新来一个数据，就用我们给的 mapper 进行数据转换。
2. 再把转换之后的数据发送给下游。

这里要解释一下“上游”和“下游”的概念：按照我们写的代码顺序，`just` 在 `map` 的上面，`Action1` 在 `map` 的下面，数据从 `just` 传递到 `map` 再传递到 `Action1`，所以对于 `map` 来说，`just` 就是上游，`Action1` 就是下游。数据是从上游（Observable）一路传递到下游（Subscriber）的，请求则相反，从下游传递到上游。

### 3.4，完整的过程

引入一个 `map` 操作符新增的内容并不多，而且由于责任分拆得好，每个部分的实现都很简单。结合第 2 部分基础的内容，这个例子的完整调用过程可以用下图表示：

![RxJava_call_stack_just_map.png](/img/201609/RxJava_call_stack_just_map.png)

上面的过程依然由 subscribe 触发，而且都是直接的函数调用，所以都在调用 subscribe 的线程执行。

## 4，线程调度

前面我们所有的过程都是通过函数调用完成的，都在 subscribe 所在的线程执行。RxJava 进行异步非常简单，只需要使用 `subscribeOn` 和 `observeOn` 这两个操作符即可。既然它俩都是操作符，那流程上就是和 `map` 差不多的，这里我们主要关注线程调度的实现原理。

我们先看看例子：

~~~ java
Observable.just("Hello world")
        .map(String::length)
        .subscribeOn(Schedulers.computation())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(len -> {
            System.out.println("got " + len + " @ " 
                    + Thread.currentThread().getName());
        });
~~~

### 4.1，`subscribeOn`

我们看看 `subscribeOn` 的实现：

~~~ java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    if (this instanceof ScalarSynchronousObservable) {
        return ((ScalarSynchronousObservable<T>)this)
                .scalarScheduleOn(scheduler);
    }
    return create(new OperatorSubscribeOn<T>(this, scheduler));
}
~~~

还记得上面的 `just` 吗？它创建的就是 `ScalarSynchronousObservable`，但是这个特殊情况我们先跳过，我们看普通的情况：通过 `create` + `OperatorSubscribeOn` 实现。

### 4.2，`OperatorSubscribeOn`

关于 scheduler/worker 的工作原理，不了解的朋友一定要先学习一下，不然下面的代码看起来估计会云里雾里，可以看 ReactiveIO 的文档，也可以看看下面几篇 Advanced RxJava 的译文：

+ [调度器 Scheduler（一）：实现自定义 Scheduler](/AdvancedRxJava/2016/08/05/schedulers-1/){:target="_blank"}
+ [调度器 Scheduler（二）：自定义 ExecutorService 使用逻辑的 Worker](/AdvancedRxJava/2016/08/19/schedulers-2/){:target="_blank"}
+ [调度器 Scheduler（三）：包装多线程 Executor](/AdvancedRxJava/2016/08/26/schedulers-3/){:target="_blank"}
+ [调度器 Scheduler（四，完结）：实现 GUI 系统的 Scheduler](/AdvancedRxJava/2016/09/02/schedulers-4/){:target="_blank"}

~~~ java
public final class OperatorSubscribeOn<T> 
      implements OnSubscribe<T> {

    final Scheduler scheduler;
    final Observable<T> source;

    public OperatorSubscribeOn(Observable<T> source, 
          Scheduler scheduler) {
        this.scheduler = scheduler;
        this.source = source;
    }

    @Override
    public void call(final Subscriber<? super T> subscriber) {
        final Worker inner = scheduler.createWorker();
        subscriber.add(inner);                            // 1

        inner.schedule(new Action0() {
            @Override
            public void call() {
                final Thread t = Thread.currentThread();

                Subscriber<T> s = new Subscriber<T>(subscriber) {
                    @Override
                    public void onNext(T t) {
                        subscriber.onNext(t);             // 2
                    }

                    // 省略 onError 和 onCompleted

                    @Override
                    public void setProducer(final Producer p) {
                        subscriber.setProducer(new Producer() {
                            @Override
                            public void request(final long n) {
                                if (t == Thread.currentThread()) {
                                    p.request(n);         // 3
                                } else {
                                    inner.schedule(new Action0() {
                                        @Override
                                        public void call() {
                                            p.request(n); // 4
                                        }
                                    });
                                }
                            }
                        });
                    }
                };

                source.unsafeSubscribe(s);                // 5
            }
        });
    }
}
~~~

1. `Worker` 也实现了 `Subscription`，所以可以加入到 `Subscriber` 中，用于集体取消订阅。
2. 在匿名 `Subscriber` 中，收到上游的数据后，转发给下游。
3. `Producer#request` 被调用时，如果调用线程就是 worker 的线程（`t`），就直接把请求转发给上游。
4. 否则还需要进行一次调度，确保调用上游的 `request` 一定是在 worker 的线程。
5. 在 worker 线程中，把自己（匿名 `Subscriber`）和上游连接起来。

这里我们就看到，连接上游（可能会触发请求）、向上游发请求，都是在 worker 的线程上执行的，所以如果上游处理请求的代码没有进行异步操作，那上游的代码就是在 `subscribeOn` 指定的线程上执行的。这就解释了网上随处可见的一个结论：

> `subscribeOn` 影响它上面的调用执行时所在的线程。

但如果仅仅是记住这么一句话，情况稍微一复杂，就必然蒙圈，所以一定要理解它的工作原理。

另外关于使用多次调用 `subscribeOn` 的效果，我们这里也就很清楚了，后面的 `subscribeOn` 只会改变前面的 `subscribeOn` 调度操作所在的线程，并不能改变最终被调度的代码执行的线程，但对于中途的代码执行的线程，还是会影响到的。

在上面的代码中，收到上游发来的数据之后，我们直接发给了下游，并没有进行线程切换，所以 `subscribeOn` 并不会改变数据向下游传递时的线程，这一工作由它的搭档 `observeOn` 完成。

### 4.3，`observeOn`

~~~ java
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, RxRingBuffer.SIZE);
}

public final Observable<T> observeOn(Scheduler scheduler, 
      int bufferSize) {
    return observeOn(scheduler, false, bufferSize);
}

public final Observable<T> observeOn(Scheduler scheduler, 
      boolean delayError, int bufferSize) {
    if (this instanceof ScalarSynchronousObservable) {
        return ((ScalarSynchronousObservable<T>)this)
            .scalarScheduleOn(scheduler);
    }
    return lift(new OperatorObserveOn<T>(scheduler, 
        delayError, bufferSize));
}

public final <R> Observable<R> lift(
      final Operator<? extends R, ? super T> operator) {
    return create(new OnSubscribeLift<T, R>(onSubscribe, 
        operator));
}
~~~

`observeOn` 有好几个重载版本，支持指定 buffer 大小、是否延迟 Error 事件，这个 `delayError` 是从 `v1.1.1` 引入的，关于它还有一个小插曲。

> 之前和 [Ryan Hoo](http://ryanhoo.github.io/){:target="_blank"} 碰到过一个问题，concat 两个 observable，第一个没有 error，第二个有 error，结果居然收不到第一个里面的数据，而是直接收到了第二个的 error。最后发现就是没有用 `delayError` 参数。

这里我们依然关注普遍情况，即利用 `lift` + `operator` 实现的情况。

所以我们先看 `OnSubscribeLift`。

### 4.4，`OnSubscribeLift`

~~~ java
public final class OnSubscribeLift<T, R> 
      implements OnSubscribe<R> {

    final OnSubscribe<T> parent;

    final Operator<? extends R, ? super T> operator;

    public OnSubscribeLift(OnSubscribe<T> parent, 
          Operator<? extends R, ? super T> operator) {
        this.parent = parent;
        this.operator = operator;
    }

    @Override
    public void call(Subscriber<? super R> o) {
        Subscriber<? super T> st = RxJavaHooks
            .onObservableLift(operator).call(o);
        st.onStart();
        parent.call(st);
        // 省略了异常处理代码
    }
}
~~~

我们先对下游 subscriber 用操作符进行处理（跳过 hook），然后通知处理后的 subscriber，它将要和 observable 连接起来了，最后把它和上游连接起来。

这里并没有线程调度的逻辑，所以我们看 `OperatorObserveOn`。

### 4.5，`OperatorObserveOn`

~~~ java
public final class OperatorObserveOn<T> implements Operator<T, T> {
    // ...

    @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            // avoid overhead, execute directly
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            // avoid overhead, execute directly
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(
                scheduler, child, delayError, bufferSize);
            parent.init();
            return parent;
        }
    }

    // ...
}
~~~

作为操作符的逻辑，还是很简单的，如果 scheduler 是 `ImmediateScheduler`/`TrampolineScheduler`，就什么也不做，否则就把 subscriber 包装为 `ObserveOnSubscriber`，看来脏活累活都是 `ObserveOnSubscriber` 干的了。

### 4.6，`ObserveOnSubscriber`

`ObserveOnSubscriber` 除了负责把向下游发送数据的操作调度到指定的线程，还负责 backpressure 支持，这导致它的实现比较复杂，所以这里只展示和分析最简单的调度功能。完整代码的分析大家可以自行阅读源码，其中还会涉及到串行访问相关的内容，建议大家先看看下面几篇 Advanced RxJava 译文：

+ [Operator 并发原语：串行访问（serialized access）（一），emitter-loop](/AdvancedRxJava/2016/05/06/operator-concurrency-primitives/){:target="_blank"}
+ [Operator 并发原语：串行访问（serialized access）（二），queue-drain](/AdvancedRxJava/2016/05/13/operator-concurrency-primitives-2/){:target="_blank"}
+ [SubscribeOn 和 ObserveOn](/AdvancedRxJava/2016/09/16/subscribeon-and-observeon/){:target="_blank"}

我们来看看简化版的调度实现（摘自上面的译文）：

~~~ java
Observable.create(subscriber -> {
    Worker worker = scheduler.createWorker();
    subscriber.add(worker);
 
    source.unsafeSubscribe(new Subscriber<T>(subscriber) {
        @Override
        public void onNext(T t) {
            worker.schedule(() -> subscriber.onNext(t));
        }
         
        @Override
        public void onError(Throwable e) {
            worker.schedule(() -> subscriber.onError(e));
        }
 
        @Override
        public void onCompleted() {
            worker.schedule(() -> subscriber.onCompleted());
        }            
    });
});
~~~

这里 `observeOn` 调度了每个单独的 `subscriber.onXXX()` 调用，使得数据向下游传递的时候可以切换到指定的线程。这也同样解释了网上随处可见的另一个结论：

> `observeOn` 影响它下面的调用执行时所在的线程。

这时我们也就清楚了多次调用 `observeOn` 的效果，每次调用都会改变数据向下传递时所在的线程。

### 4.7，完整的过程

最后我们看一下整个例子的调用过程：

![RxJava_call_stack_just_map_subscribeOn_observeOn.png](/img/201609/RxJava_call_stack_just_map_subscribeOn_observeOn.png)

## 5，backpressure

其实在前面我们讲 `just` 时，就已经讲过了 backpressure：

> 在 RxJava 1.x 中，数据都是从 observable push 到 subscriber 的，但要是 observable 发得太快，subscriber 处理不过来，该怎么办？一种办法是，把数据保存起来，但这显然可能导致内存耗尽；另一种办法是，多余的数据来了之后就丢掉，至于丢掉和保留的策略可以按需制定；还有一种办法就是让 subscriber 向 observable 主动请求数据，subscriber 不请求，observable 就不发出数据。它俩相互协调，避免出现过多的数据，而协调的桥梁，就是 producer。

为了章节完整性，这里保留一节，更多细节的内容，可以查看 [stackoverflow 上的这篇 wiki](http://stackoverflow.com/documentation/rx-java/2341/backpressure){:target="_blank"}，由 Advanced RxJava 系列博客原文作者编写，非常不错。不过其内容总结来说，也就是上面这一段 :)

## 6，hook

在上面的内容中，我们多次遇见了 hook，为了简化逻辑，也多次跳过了 hook，这里我们就看看 hook 有什么用，工作原理是什么。

利用 hook 我们可以站在“上帝视角”，多种重要的节点上，都有 hook。例如创建 Observable（create）时，有 `onCreate`，我们可以进行任意想要的操作，记录、修饰、甚至抛出异常；以及和 scheduler 相关的内容，获取 scheduler 时，我们都可以进行想要的操作，例如让 `Scheduler.io()` 返回立即执行的 scheduler。

这些内容让我们可以执行高度自定义的操作，其中就包括便于测试。

其实 hook 的原理并不复杂，在关心的节点（hook point）插桩，让我们可以操控（manipulate）程序在这些节点的行为，至于操控的策略，有一系列函数进行设置、以及清理。

目前和 hook 相关的内容主要在 `RxJavaPlugins` 和 `RxJavaHooks` 这两个类中，后者在 `v1.1.7` 引入，功能更加强大，使用更加方便。

## 7，测试

RxJava 项目本身测试覆盖率高达 84%，为了便于我们对使用 RxJava 的代码进行测试，它还专门提供了 `TestSubscriber`，我们可以用它来获取我们的事件流中的事件、进行验证、进行等待，使用起来非常简便。

此外，上面提到的 hook 机制也可以用来帮助我们进行测试，例如对线程调度进行一些操控。

## 8，总结

作为拆轮子系列第四篇，本文从基本用例出发，从四个角度对 RxJava 的原理进行了分析：

+ 事件流源头（observable）怎么发出数据
+ 响应者（subscriber）怎么收到数据
+ 怎么对事件流进行操作（operator/transformer）
+ 以及整个过程的调度（scheduler）

其中前两部分集中在第 2 节，看完这四部分之后，应该可以对 RxJava 的工作原理有一个比较清晰的认识。文中引用了好几篇 Advanced RxJava 系列博客的译文，还是非常值得看的，对于强化具体每一部分的理解非常有帮助。

文章最后还对另外三点内容简单提了一下，对于我们使用的高级需求，还是很有帮助的，但本文中仅仅作为一个导向，没有展开：

+ backpressure
+ hook
+ 测试

拆轮子系列也已经到了第四篇，希望大家看完之后能对打通任督二脉有所助益。对我来说，看过 OkHttp 和 Retrofit 的源码之后，面对相关的问题，都会很有底气，即便当时不清楚，也能够轻松通过阅读源码寻找到答案。这个系列我会继续坚持下去，接下来应该就是 dagger 2 了。但是在此之前，我会把 RxJava scheduler 的复杂使用场景进行一个总结，敬请期待！
