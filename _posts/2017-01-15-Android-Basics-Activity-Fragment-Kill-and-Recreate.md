---
layout: post
title: 安卓基础：Activity/Fragment 销毁与重建
tags:
    - 安卓开发
    - 基础知识
---

Activity 的销毁与重建有两种情况：App 处在后台，由于内存紧张而被杀死，当用户回到 我们的 App 时，被杀掉的 Activity 会被重新创建；当手机配置发生变化时，例如旋转屏幕方向，Activity 会被重新创建。

当 Activity 被销毁时，在第一种情况中，它加载的 Fragment 显然也要被销毁，因为此时通常整个 App 都被杀死了，但第二种情况中，我们可以通过设置 Fragment 的 `setRetainInstance` 来避免 Fragment 被重新创建，但相应的，我们在加载 Fragment 的时候，就需要检查 Fragment 是否已经加载（利用 FragmentManager 进行查找），以免重复添加 Fragment。

为了防止 Activity 在重新创建的过程中丢失状态，我们可以在 `onSaveInstanceState` 回调中保存数据，并在 `onCreate` 回调中进行恢复，利用 [Icepick](https://github.com/frankiesardo/icepick) 可以很方便地完成这件事。

## 具体例子

怎么处理 Activity 的重新创建，其实很大程度上取决于具体业务，下面我将分享一个 GitHub OAuth 客户端的例子。

OAuth 相关的内容这里不展开，可以看看 [阮一峰的理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)。OAuth 通常需要应用服务器参与，但在这个例子中并不涉及，我的需求就是用户在 GitHub 完成授权后，取得 GitHub 的 token，然后获取用户在 GitHub 上的信息，全部都在客户端完成。

GitHub OAuth 过程分为以下几步：

+ 打开授权页面，用户可能需要登录；
+ 用户授权后，跳转到指定的回调页；
+ 拿到跳到回调页时携带的 code 值；
+ 用 code 换取 token；
+ 用 token 读取用户的数据；

一开始我利用 WebView 拦截请求的 url，当检测到是要打开回调页的时候，就把 code 拿出来，后续的步骤通过 RESTful API 完成。但这需要用户在 App 内输入 GitHub 的账户密码，这样的话，是在 WebView 内还是在 EditText 内没有区别，我不希望这样。

所以我换成了在浏览器中完成授权，但我的 App 声明一个对回调 url 的 Intent filter，那用户完成授权之后就可以调起我的 App 了。当然，跳转到回调页时系统会弹出“打开方式”对话框，我们可以在打开浏览器之前给用户一个提醒，而在 6.0 之后，我们可以通过 App links 优化这一体验。

## 实现方案

整个过程会启动 OAuthActivity，它的 launchMode 为 singleTop，同时监听回调 url；它会用浏览器打开授权页，用户在浏览器完成授权后，会再次启动 OAuthActivity，但由于此时栈顶已有一个实例，所以会调用 onNewIntent；此时我就可以调用相关 API，成功之后利用 activity result 回传给调用方（MainActivity）。

这里有几个要点：

+ singleTop，为了让浏览器的 Intent 能被调用方启动的 OAuthActivity 实例处理，所以我们使用 singleTop；正常情况下，浏览器回调时，OAuthActivity 一定位于栈顶，所以一定会调用 onNewIntent；
+ GitHub 的 OAuth 要求 redirect_uri 必须使用 http/https 标准 scheme，不允许自定义 scheme，所以“打开方式”的问题只能优化，无法避免；
+ 用 onActivityResult 传递结果，而不是利用 listener 对象，因为一旦我们的 App 被杀死，我们就无法取得正确的 listener 对象了；

在这个过程中其实有很多细节需要妥善处理：用户返回了怎么办？浏览器位于前台时，我们的 App 被杀掉了怎么办？调用 GitHub API 的过程中如果退到后台，被杀掉了怎么办？为了处理这些问题，我们需要把整个过程分为几个状态，并考虑状态之间的转移关系，请看下面的状态机：

![](https://imgs.piasy.com/2018-03-23-2017011551325GitHubOAuth_state_machine_normal.jpg)

这里面并未包含 Activity 被杀掉的情况，一步步来。

+ `STATE_NOT_REQ` 是初始状态，onCreate 中会用浏览器打开授权页；
+ 打开浏览器后，我们的 App 会 onPause，进入 `STATE_SEND_REQ` 状态；
+ 授权完成后，回调 onNewIntent，我们调用 API，进入 `STATE_CALL_API` 状态；
+ API 返回后，进入 `STATE_SUCCESS` 或者 `STATE_FAIL`；
+ 如果用户没有完成授权就返回了我们的 App，那我们会在 `STATE_SEND_REQ` 状态下 onResume，那就进入 `STATE_FAIL`；

## 处理 Activity 重新创建

上述过程中，我们的 App 可能会在好几个环节中被杀掉，先来看看浏览器处于前台时：

![](https://imgs.piasy.com/2018-03-23-2017011567121GitHubOAuth_state_machine_killed_behind_browser.jpg)

利用开发者选项中的“不保留活动”可以触发 Activity 被销毁。

据我观察，打开浏览器之后，MainActivity 和 OAuthActivity 均被销毁，从浏览器返回时，无论是完成授权后浏览器调起我们的 App，还是用户直接返回退出浏览器，都会先打开一个新的 OAuthActivity，再恢复之前的 OAuthActivity。也就是说会出现两个 OAuthActivity 实例，都执行了 onCreate。

浏览器启动的 OAuthActivity 实例显然无法向 MainActivity 发送结果，而且它也不知道各种配置参数（client id, client secret 等）无法调用 API，所以我们需要把 code 从这个实例传递到由 MainActivity 启动的 OAuthActivity 实例中。

怎么传递？EventBus？那需要老的 OAuthActivity 实例先 onCreate（注册），但我观察到的现象并非如此；而且它们还需要取得同一个 EventBus 实例，`EventBus.getDefault` 倒是可以满足这一条件。

其实这里主要的问题是先有数据还是先有订阅者，我很自然地想到了 RxJava 的 ReplaySubject，但它们如何取得同一个 subject 实例？

### 多个实例间传递数据

这里我利用了 WeakReference：

~~~ java
private static WeakReference<ReplaySubject<~>> sOAuthResultSubject;
private ReplaySubject<~> mOAuthResultSubject;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // init reference
    if (sOAuthResultSubject == null || sOAuthResultSubject.get() == null) {
        mOAuthResultSubject = ReplaySubject.create();
        sOAuthResultSubject = new WeakReference<>(mOAuthResultSubject);
    } else {
        mOAuthResultSubject = sOAuthResultSubject.get();
    }
}

@Override
protected void onDestroy() {
    super.onDestroy();

    // reduce reference
    mOAuthResultSubject = null;
}
~~~

+ 利用 static 变量跨实例共享对象；
+ 利用 WeakReference 避免内存泄漏；
+ 每个实例会在 onCreate 中用一个强引用指向 subject，在 onDestroy 中取消这个引用，只要两个 OAuthActivity 实例的有效期（onCreate ~ onDestroy）存在重叠，那数据传递就可以完成；当两个实例都 onDestroy 之后，subject 对象也就可以被 gc 了；

在被恢复的 OAuthActivity 中，onCreate 时我们检测到当前处于 `STATE_SEND_REQ`，所以我们等待浏览器启动的实例发来 code，进入 `STATE_WAIT_CODE`。这一等待会在 3s 后超时，超时则说明是用户取消了授权。

## 调用 API 时被杀掉

请看下面的状态图：

![](https://imgs.piasy.com/2018-03-23-2017011557554GitHubOAuth_state_machine_killed_calling_api.jpg)

由于 code 只能用一次，而且这里也不涉及多个 OAuthActivity 实例，所以这里的处理就简单得多：直接判定授权失败。

## 手机屏幕旋转

现在所有的代码都在 OAuthActivity 中，我们可以把代码转移到 Fragment 中，并设置 retain instance，不过这会让我们处理浏览器的 Intent 以及向 MainActivity 发送结果的代码变得复杂，这里也就只是简单地判定为授权失败。

毕竟授权时手机屏幕发生旋转，也不是主要场景，我们的处理已经算得上“妥当”了。

## 其他问题

+ 从浏览器返回时，是否一定会有两个 OAuthActivity 实例都走 onCreate？它们之间是否有固定的先后顺序？有效期是否一定有重叠？这几个问题现在只有观测值，并没有理论值，如果有熟悉 framework 的朋友，欢迎指点；
+ 之前我的 OAuthActivity 没有设置 layout，而是显示了一个 ProgressDialog，dialog 后面能看到 MainActivity 的 View，当 dialog 关闭后，OAuthActivity 并未 onDestroy，而此时如果我点击屏幕，MainActivity 会收到 onActivityResult，而 OAuthActivity 却不会 onDestroy！这个问题我比较困惑，这是安卓系统的特性，还是 bug？
+ 调用 API 的时候我现在使用了 RxJava 2.x，它的错误处理机制发生了一些变化：使用 zip/concat 之类的操作符时，如果多个上游发生了 onError，只有一个会被传递到下游，其他的会被传递到 `RxJavaPlugins.errorHandler` 中；如果我们没有设置 errorHandler，那这些异常就属于 uncaught exception，在安卓系统上会导致 App 闪退！[详见这个 issue](https://github.com/ReactiveX/RxJava/issues/4996)。

这个例子的完整代码可以[在 GitHub 获取](https://github.com/Piasy/GitHubAndroidOAuth)。
