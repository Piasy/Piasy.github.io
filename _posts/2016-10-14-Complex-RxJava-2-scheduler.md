---
layout: post
title: RxJava 复杂场景（二）：调度
tags:
    - Reactive eXtention
---

RxJava 最大的两个特点：事件流操作，异步。

组合利用各种操作符，我们可以实现复杂的事件流处理需求，例如[前文中提到的缓存](/2016/08/24/Complex-RxJava-1-cache/){:target="_blank"}：根据一组 id，先从本地查询，本地缺失的部分再从服务器获取，再把两者合并起来返回，最后服务器获取的部分还要保存到本地。

而利用 `subscribeOn` 和 `observeOn` 这两个操作符，我们可以轻松地实现代码执行的异步调度。

但当我们的需求变得越来越复杂时，我们还能“轻松地”完成异步调度吗？

## `subscribeOn` 和 `observeOn` 的调度原理

磨刀不误砍柴工，我们先要搞清楚调度的原理。在[拆轮子系列：拆 RxJava](/2016/09/15/Understand-RxJava/#section-4){:target="_blank"} 中，我们分析过这俩好搭档的实现原理，这里摘录如下：

> …… 连接上游（可能会触发请求）、向上游发请求，都是在 `worker` 的线程上执行的，所以如果上游处理请求的代码没有进行异步操作，那上游的代码就是在 `subscribeOn` 指定的线程上执行的。这就解释了网上随处可见的一个结论：`subscribeOn` 影响它上面的调用执行时所在的线程。

> 另外关于使用多次调用 `subscribeOn` 的效果，我们这里也就很清楚了，后面的 `subscribeOn` 只会改变前面的 `subscribeOn` 调度操作所在的线程，并不能改变最终被调度的代码执行的线程，但对于中途的代码执行的线程，还是会影响到的。

> 这里 `observeOn` 调度了每个单独的 `subscriber.onXXX()` 调用，使得数据向下游传递的时候可以切换到指定的线程。这也同样解释了网上随处可见的另一个结论：`observeOn` 影响它下面的调用执行时所在的线程。

> 这时我们也就清楚了多次调用 `observeOn` 的效果，每次调用都会改变数据向下传递时所在的线程。

当然，上面都是结论性的片段，对此比较陌生的朋友，建议先好好看看[拆轮子系列：拆 RxJava](/2016/09/15/Understand-RxJava/#section-4){:target="_blank"}。

## 复杂场景一：zip

假设我们用 `create` 创建了两个 `Observable`，其中都不包含异步代码。我们需要把它们组合起来，这里我们就用 `zip`。然后对于合并之后的 `Observable`，我们还需要进行一个 `map` 操作。最后我们订阅之（`subscribe`）。

这里我们对调度的需求是：`create` 里的代码在 io 线程执行，`zip` 合并的代码在主线程执行，`map` 的操作在 io 线程执行，最后 `subscriber` 的代码在主线程执行。

我们先不质疑需求的合理性。怎么样，是不是有点蒙？

别怕，一步一步来。

_下面的代码都是在 JUnit 测试中运行，所以我把主线程都替换为 computation 线程。_

### `subscriber` 在 computation 线程

我们先保证 `subscriber` 在 computation 线程执行，这大家应该都会：

``` java
observable
    .observeOn(Schedulers.computation())
    .subscribe(this::print);
```

### `create` 在 io 线程

我们再看怎么让 `create` 的代码在 io 线程执行。如果没有 `zip`，我想大家也都会：

``` java
observable
    .subscribeOn(Schedulers.io())
    .observeOn(Schedulers.computation())
    .subscribe(this::print);
```

但有了 `zip` 之后会有什么不一样？我们也不必一行行看 `zip` 的代码，我们只需要知道它最终会通过 `lift(OperatorZip)` 来实现合并功能即可。而 lift 和 Operator 的流程，我们在“拆 RxJava”中都是了解过的，就是**内部搞一个 subscriber 订阅上游，收到上游的数据之后，实现自己的逻辑，再转发给下游**。zip 有什么逻辑？当然是从每个上游都收集到一个数据之后，调用我们的 `FuncX` 进行合并，再发给下游。

那这个过程本身是不会有线程切换的，也就是说我们的 `subscribeOn` 的作用将会一直向上传递，所以两个 `create` 都会在 io 线程执行。

### `zip` 合并在 computation 线程

上面我们提到：

> …… 内部搞一个 subscriber 订阅上游，收到上游的数据之后，实现自己的逻辑，再转发给下游。

`zip` 操作符会调用我们的 `FuncX` 执行合并操作，这已经开始数据向下游传递的过程了。那怎么改变这一过程的线程，相信大家也有了答案：`observeOn`。

但我们用在哪里呢？我们希望 `FuncX` 的执行在 computation 线程，所以我们需要数据在传递到 `zip` 的时候就已经切换到了 computation 线程。所以我们要用在前面两个 `create` 的 `Observable` 之后，`zip` 之前。

但这里有两个 `Observable`，用在哪一个呢，还是两个都需要？

对于这个问题，我们就需要更细致地看 `zip` 的源码才能回答了，不过看代码是不是最高效的方式呢？不是，我们实验一下就可以知道了。另外有一点我们也可以确定，如果我们对两个 `Observable` 都运用 `observeOn(Schedulers.computation())`，那 `FuncX` 肯定是在 computation 线程。

_这里我也没有细看 zip 的源码，没必要。通过实验我发现，只有对最后一个 Observable 使用 observeOn，才能起到调度效果，对其他 Observable 使用 observeOn，如果最后一个 Observable 没有使用 observeOn，就会被 subscribeOn 的效果所覆盖（如果没有 subscribeOn，那就是 subscribe 所在线程），如果最后一个 Observable 用了 observeOn，就会被它覆盖。_

所以我们的代码是这样的：

``` java
Observable<Integer> odd = Observable
        .<Integer>create(subscriber -> {
            logThread("create 1");
            subscriber.onNext(1);
            subscriber.onCompleted();
        });
Observable<Integer> even = Observable
        .<Integer>create(subscriber -> {
            logThread("create 2");
            subscriber.onNext(2);
            subscriber.onCompleted();
        });
Observable.zip(odd.observeOn(Schedulers.computation()),
        even.observeOn(Schedulers.computation()),
        this::add)
        // ...
```

### `map` 在 io 线程

数据经过了 `zip` 之后到达了 `map`，这同样是数据向下传递的过程，所以我们依然用 `observeOn` 改变线程：

``` java
// ...
Observable.zip(odd.observeOn(Schedulers.computation()),
        even.observeOn(Schedulers.computation()),
        this::add)
        .observeOn(Schedulers.io())
        .map(this::triple)
        // ...
```

### 完整例子

所以最后完整代码就是这样：

``` java
@Test
public void testZip4() {
    Observable<Integer> odd = Observable
            .<Integer>create(subscriber -> {
                logThread("create 1");
                subscriber.onNext(1);
                subscriber.onCompleted();
            });
    Observable<Integer> even = Observable
            .<Integer>create(subscriber -> {
                logThread("create 2");
                subscriber.onNext(2);
                subscriber.onCompleted();
            });
    Observable.zip(odd, // 只需要对最后一个 Observable 使用 observeOn
            even.observeOn(Schedulers.computation()),
            this::add)
            .observeOn(Schedulers.io())
            .map(this::triple)
            .subscribeOn(Schedulers.io())
            .observeOn(Schedulers.computation())
            .subscribe(this::print);
    Utils.sleep(2000);
}
```

最终运行的输出如下：

``` bash
create 1 from RxIoScheduler-2
create 2 from RxIoScheduler-2
add 1 and 2 from RxComputationScheduler-2
triple 3 from RxIoScheduler-3
print 9 from RxComputationScheduler-1
```

可以看到，符合预期。

## 复杂场景之二：`Observable` 创建的地方就有异步

前面我们反复提到一个前提：如果上游的代码没有进行异步操作。那如果有异步代码会怎么样？

在回答这个问题之前，我们先问一个问题，什么情况下创建 Observable 时会有异步代码？

在我们把老的非 Rx 异步 API 包装为 Rx API 时。很多第三方库都会提供异步的 API，它们利用 Thread/Handler/Executor 来实现异步。

这里我举一个具体的例子，友盟的社会化组件，它的第三方登录、获取用户第三方信息之类的 API 都是异步的，我们给它一个 callback，它在执行完毕之后利用 callback 把数据传给我们。

例如我们需要获取第三方头像，下载到本地，再上传到我们的服务器。我们可能会这样写代码：

``` java
Observable.create(subscriber -> {
    umSocialService.getPlatformInfo(context, platform, 
            new SocializeListeners.UMDataListener() {
                @Override
                public void onStart() {
                }

                @Override
                public void onComplete(int status, Map<String, Object> info) {
                    if (status == HTTP_OK && info != null) {
                        subscriber.onNext(info.get(UM_AVATAR_KEY).toString());
                    } else {
                        subscriber.onError(new IllegalStateException());
                    }
                }
            });
        })
        .flatMap(url -> downloader.download(url))
        .flatMap(file -> uploader.upload(file))
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(/** success */);
```

遗憾的是，上面的代码会抛出 `NetworkOnMainThreadException`，获取到第三方头像 url 之后，我们的下载、上传操作都是在主线程执行的，并不是 io 线程。

所以我们最初的问题，答案也很明显了：如果创建 Observable 时有异步代码，那调度的结果就不是我们预期的那样了。

### 问题出在哪儿？

别慌，也别急着 google，NetworkOnMainThreadException 太宽泛了，很难有和我们具体场景相关的结果，而且错误也很明显不是？所以我们直接结合调度的原理理解问题出在哪儿。

`subscribeOn(Schedulers.io())` 确实能让我们 `create` 里面的代码在 io 线程执行，但是我们传给友盟的回调，代码是否还会在 io 线程执行呢？这当然取决于友盟的具体实现了。

如果它内部的所有代码都是阻塞的（_这里不要纠结“阻塞”、“非阻塞”、“同步”、“异步”这几个概念，我们在这个具体场景理解具体含义就好_），也就是说如果它同步执行 HTTP 请求获取用户第三方信息，再同步调用我们的回调，没有发生线程切换，那很好，一切都没有发生线程切换，都在 io 线程。

如果它内部的代码是非阻塞的，例如它新启动了一个线程发起 HTTP 请求，再在其中同步调用我们的回调，那回调的代码就是在新的线程执行的。

而如果它新启动了一个线程发起 HTTP 请求，再利用主线程的 Handler 在主线程调用我们的回调，那回调的代码当然就是在主线程执行的了。我们现在应该就是这种情况。

为什么回调的代码在主线程执行，就会抛出 NetworkOnMainThreadException？因为从回调这里开始，我们数据发往下游的路上，就都是在主线程上了，回调后面我们改变数据发往下游的线程了吗（`observeOn`）？没有，所以我们的下载、上传操作都是在主线程，所以我们当然会遇到 NetworkOnMainThreadException 了。

好了，简单总结一句：

> 创建 Observable 时的异步代码，有可能打断我们调度的效果，引发意想不到的错误。

### 执行过程流程图

看过“拆 RxJava”之后，我们对整个过程应该有了一个比较清晰的认识，这里我们把其中的流程图稍作修改，体现出 `create` 的异步（_请把中间的 `map` 脑补成 `flatMap`_）：

![RxJava_call_stack_create_async.png](/img/201610/RxJava_call_stack_create_async.png)

我们可以看到，`subscribeOn` 的效果在我们创建的 `OnSubscribe` 那里就终止了，从那之后代码的执行线程都是 `create` 中的回调被执行的线程，直到遇到 `observeOn`。

### 解决问题

理解了问题出在哪儿，也理清了执行流程，解决办法就很简单了，我们在 `create` 之后、`flatMap` 之前改变一下数据发往下游的线程即可！

``` java
Observable.create(subscriber -> {
            // ...
        })
        .observeOn(Schedulers.io())
        .flatMap(url -> downloader.download(url))
        .flatMap(file -> uploader.upload(file))
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(/** success */);
```

## 总结

首先感谢 [Stay 兄](http://notes.stay4it.com/){:target="_blank"}对第二个例子提出了宝贵的改进建议。

在本文中，我举了两个复杂的调度场景，结合这两个场景，以及前面讲到的原理，大家对调度的原理应该有了更深刻的理解，后面面对更复杂的调度需求，相信也能轻松地解决了。

另外，距离“拆 RxJava”发布已经过去了一个月，今天终于抽空写了这篇调度的文章，抱歉来晚了。另外之前承诺的“拆 Dagger2”，因为我接下来几个月都会非常忙，所以可能得延期了 :(
