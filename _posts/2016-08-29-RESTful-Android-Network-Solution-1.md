---
layout: post
title: RESTful 安卓网络层解决方案（一）：概览与认证实现方案
tags:
    - 安卓开发
    - 架构
---

在[拆轮子系列：拆 Okio](/2016/08/04/Understand-Okio/){:target="_blank"} 最后我曾说过会对 Retrofit、OkHttp、Okio 三者进行一个小结，并且整理一套网络层的“微架构”，今天终于得以完成，在这里一起奉送给大家 :)

+ 🏁 RESTful 安卓网络层解决方案（一）：概览与认证实现方案
+ [RESTful 安卓网络层解决方案（二）：空 JSON 和 API Error 解析](/2016/09/04/RESTful-Android-Network-Solution-2/){:target="_blank"}
+ [RESTful 安卓网络层解决方案（三）：API model 与 Business model 分离](/2016/09/04/RESTful-Android-Network-Solution-3/){:target="_blank"}

注：本来只打算写一篇文章，但篇幅太长，最后还是按照内容拆分为了三篇，也算是单一职责 :)

## 1，网络“三板斧”架构回顾

<img src="/img/201608/okio_okhttp_retrofit.png" alt="okio_okhttp_retrofit" style="height:500px">

今天还在和 iOS 同事讨论，iOS 开发中有没有可以和“三板斧”相对应的存在，得到的答案是 AFNetworking，不过它独自完成了“三板斧”的所有工作，既有底层的 API，也有高度的封装（不一定准确，如有错误，欢迎指出）。

相比之下，“三板斧”根据分工完全隔离，还是更加合理的，灵活而且干净，flexible and clean。我们完全可以只用其中一层，例如用 Okio 进行 IO 操作、二进制数据操作，只用 OkHttp 进行网络访问，或者用 Retrofit 定义 RESTful API 但使用其他 HttpClient。

在 [拆轮子系列：拆 OkHttp](/2016/07/11/Understand-OkHttp/#callserverinterceptor){:target="_blank"} 中，我们就曾提到：

> 分层的思想在 TCP/IP 协议中就体现得淋漓尽致，分层简化了每一层的逻辑，每层只需要关注自己的责任（单一原则思想也在此体现），而各层之间通过约定的接口/协议进行合作（面向接口编程思想），共同完成复杂的任务。

分层（分治）在软件开发中可以说无处不在，是一种非常有用的方法。在这里我们也可以看到，“三板斧”除了在细节之处践行了分层思想，它们之间的协作，也正是一种更全局的分层思想的体现。

## 2，安卓 RESTful 网络层“微架构”

基础的 API 定义、请求发起，这些内容就不在这里展开了，对 Retrofit、OkHttp 不熟悉的朋友一定要先看看官方教程和文档，不然后面可能会觉得云里雾里。当然也可以阅读我的两篇文章：

+ [拆轮子系列：拆 Retrofit](/2016/06/25/Understand-Retrofit/){:target="_blank"}
+ [拆轮子系列：拆 OkHttp](/2016/07/11/Understand-OkHttp/){:target="_blank"}

在这套“微架构”里面主要涉及三大部分内容：

1. 怎么做认证；
2. 怎么做 JSON 解析，空 JSON 以及 API Error 解析；
3. API model 和 Business model 分离；

在第一篇中，我们先讲一下认证功能的实现。

## 3，认证功能的实现

### 3.1，认证需求

身份认证其实是一个基本的需求，如果我们有用户系统，那登录之后发出的请求可能都是需要一个 token 的（query），而在登录之前发出的请求，我们可能会做一个 basic auth 认证（header）。而对于安全追求更高的团队，可能会有一些防止重放攻击、防止恶意构造请求的策略，例如每个请求加上时间戳，每个请求进行一次额外的校验（验证是合法的客户端，不验证具体是哪个用户）。

这里我先讲一下额外校验的一种方式，例如每个请求加上 `timestamp` 和 `mac` 这两个参数，`timestamp` 就是当前时间戳，而 `mac` 则是一个认证码，`mac` 的计算取决于 `timestamp` 以及另外一个 `mac_key`，它只在登录成功时会返回。也就是：

~~~ java
String mac = hash("timestamp=" + timestamp + "mac_key=" + macKey);
~~~

那这里其实有一个问题，如果用户还没有登录，我们怎么做 mac 校验？我们可以暂且用 basic auth 来代替 macKey。

### 3.2，方案设计

需求确定了，那我们怎么实现呢？我们的每个请求都需要加上额外的几个参数（timestamp，mac 以及可选的 token），每个 API 定义时都加上这些参数吗？

当然可以这样做，但这显然有点傻，而且这样会给 token 和 macKey 的管理带来麻烦：我们很多地方都需要维护它们，如何同步更新？当然可以通过全局变量的方式来实现，但这显然也不合理，它们只应该被需要的模块看到。

其实了解 OkHttp 的 Interceptor 链条的朋友应该能想到，我们可以利用一个 Interceptor 来集中实现我们的认证需求，请求发出去之前根据不同的情况添加不同的 query/header。

想到之后其实就比较简单了，但当我们真正去实现的时候会遇到一个问题：我们怎么知道哪些请求是需要 token 的，哪些请求是进行 basic auth 的呢？因为我们是在 OkHttp 层在做事了，Retrofit 定义 API 的信息已经完全丢失了。

怎么办？我们需要一个不会丢失的信息。Header！

我们可以给进行 basic auth 的 API 在定义时就加上一个特殊的 header，具体内容无所谓，只要它具有可识别性。那么在一个 API 调用到达 Interceptor 时，我们就有了可以进行判断的信息。

### 3.3，代码实现

好了，下面我们看一看简单实现的代码。

#### 3.3.1，定义 API

~~~ java
public interface Api {
    @POST("tokens")
    @FormUrlEncoded
    @Headers("Auth-Type:Basic")                                 // 1
    Observable<User> login(@Field("account") String account,
            @Field("password") String password);

    @GET("/users/{uid}")                                        // 2
    Observable<User> user(@Path("uid") long uid);
}
~~~

这边我们用一个特殊的 header 来标记是 basic auth（1），这里为了代码简洁，就没有定义在常量中，其实是需要定义常量的。而 auth 类型默认是 token auth，为了减少代码量，我们就不显式加上对应的 header 了（2）。

#### 3.3.2，YLAuthInterceptor 的结构

~~~ java
public class YLAuthInterceptor implements Interceptor {

    private final String mBasicAuthId;
    private final String mBasicAuthPass;

    private volatile String mToken;                             // 1
    private volatile String mMacKey;
    
    public YLAuthInterceptor(String basicAuthId,                // 2
            String basicAuthPass) {
        mBasicAuthId = basicAuthId;
        mBasicAuthPass = basicAuthPass;
    }

    public void setAuth(String token, String macKey) {          // 3
        mToken = token;
        mMacKey = macKey;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        // ...
    }

    @VisibleForTesting
    void tokenAuth(Request.Builder newRequest, HttpUrl url, 
            long timestamp) {                                   // 4
        // ...
    }

    @VisibleForTesting
    void basicAuth(Request.Builder newRequest, HttpUrl url, 
            long timestamp) {
        // ...
    }
}
~~~

1. 由于我们的 token 是会发生变化的（未登录 -> 登录 -> 退出登录 -> 重新登录），所以我们需要保证它的可见性，而由于 token 的更新不依赖旧的状态，`volatile` 关键字就足够了。
2. basic auth 的用户名密码是固定不变的，我们直接构造函数传入即可。
3. token，macKey 都是后面会变化的，所以我们需要一个 setter，而不是在构造函数中传入。
4. 这边有两个小技巧：方法声明为 package private，便于测试代码访问；时间作为参数传入，使得测试可控制。

#### 3.3.3，YLAuthInterceptor 的实现

先看 `intercept()` 的实现：

~~~ java
@Override
public Response intercept(Chain chain) throws IOException {
    Request origin = chain.request();
    Headers originHeaders = origin.headers();
    Headers.Builder newHeaders = new Headers.Builder();                     // 1
    String authType = "Token";
    for (int i = 0, size = originHeaders.size(); i < size; i++) {
        if (!TextUtils.equals(originHeaders.name(i), "Auth-Type")) {        // 2
            newHeaders.add(originHeaders.name(i), originHeaders.value(i));
        } else {
            authType = originHeaders.value(i);
        }
    }
    Request.Builder newRequest = origin.newBuilder()
            .headers(newHeaders.build());
    switch (authType) {                                                     // 3
        case "Basic":
            basicAuth(newRequest, origin.url(), System.currentTimeMillis());
            break;
        case "Token":
        default:
            tokenAuth(newRequest, origin.url(), System.currentTimeMillis());
            break;
    }
    return chain.proceed(newRequest.build());                               // 4
}
~~~

1. 我们需要移除这个标记 header，所以我们要构造一个新的 header 集合。
2. 对比 header name，来从中寻找 auth 类型，这里同样应该定义为常量。
3. 根据不同的类型应用不同的认证策略。
4. 我们利用 OkHttp 的 Interceptor API，发起修改过的请求，并返回响应。

再看 `tokenAuth()` 和 `basicAuth()` 的实现：

~~~ java
@VisibleForTesting
void tokenAuth(Request.Builder newRequest, HttpUrl url, long timestamp) {
    if (TextUtils.isEmpty(mToken) || TextUtils.isEmpty(mMacKey)) {
        throw new YLApiError(/**...*/);                             // 1
    }
    String text = "token=" + mToken + "timestamp=" + timestamp;
    String mac = hash(text + "mac_key=" + mMacKey);

    HttpUrl.Builder newUrl = url.newBuilder()
            .addQueryParameter("timestamp", String.valueOf(timestamp))
            .addQueryParameter("mac", mac)
            .addQueryParameter("token", mToken);

    newRequest.url(newUrl.build());
}

@VisibleForTesting
void basicAuth(Request.Builder newRequest, HttpUrl url, long timestamp) {
    String text = "timestamp=" + timestamp;

    String macKey = hash(mBasicAuthId + mBasicAuthPass);
    String mac = HashUtils.sha1(text + "mac_key=" + macKey);

    HttpUrl.Builder newUrl = url.newBuilder()
            .addQueryParameter("timestamp", String.valueOf(timestamp))
            .addQueryParameter("mac", mac);

    newRequest.url(newUrl.build());
    newRequest.addHeader("Authorization",
            basicAuthHeader(mBasicAuthId, mBasicAuthPass));
}

String basicAuthHeader(String username, String pwd) {
    final String userAndPassword = username + ":" + pwd;
    return "Basic " + Base64.encodeToString(
                    userAndPassword.getBytes("UTF-8"), Base64.NO_WRAP);
}
~~~

这段代码比较直观，主要是对 OkHttp 相关 API 的使用。

我们需要在 tokenAuth 时检查 token 和 macKey，如果为空我们就抛出一个异常（1）。但这其实只能处理我们初始化时存在问题的情况，如果我们被挤下线，导致 token 失效，我们应该怎么处理呢？而进一步抽象这个问题，其实就是 token/macKey 如何管理。

解决方案其实很简单，我们把 interceptor 作为一个单例依赖，首先注入到登录注册模块中，登陆成功之后，我们就为它更新 token/macKey，其次我们的 API Error 要有一个集中处理的地方，我们把 interceptor 也注入进去，在捕获到 token 失效的错误后，我们就清除 interceptor 的 token/macKey。至于 UI 上怎么给用户提示，我们可以在 BaseActivity/BaseFragment 中监听错误的发生，并弹出对话框。

#### 3.3.4，单元测试

从[前面一篇讲 RxJava 复杂场景](/2016/08/24/Complex-RxJava-1-cache/){:target="_blank"}的文章开始，我就在强调单元测试的重要性，上面的代码也不短，足有一百多行，不写几个测试用例，还真没有信心它一定能正确工作。

~~~ java
public class YLAuthInterceptorTest {

    private YLAuthInterceptor mYLAuthInterceptor;
    private Request mOriginRequest;
    private long mTimestamp;

    @Before
    public void setUp() {                                           // 1
        mYLAuthInterceptor = new YLAuthInterceptor(CLIENT_ID, CLIENT_PASS);
        mOriginRequest = new Request.Builder()
                .url(SERVER_ENDPOINT + "/users/1905378617")
                .build();
        mTimestamp = 1438141764;                                    // 2
    }

    @Test
    public void tokenAuth() throws Exception {
        mYLAuthInterceptor.setAuth("wx:1905378617",
                "975b56d640c0864a2c277dd0fe429b1dcbbf34a8");

        Request.Builder builder = mOriginRequest.newBuilder();
        mYLAuthInterceptor.tokenAuth(builder, mOriginRequest.url(), mTimestamp);
        Request newRequest = builder.build();

        HttpUrl expectedUrl = HttpUrl.parse(SERVER_ENDPOINT         // 3
                + "/users/1905378617"
                + "?timestamp=1438141764"
                + "&mac=1cadbea4e322d42fdabe3b8fed15f741b6be67f1"
                + "&token=wx:1905378617");
        assertThat(newRequest.url(), is(expectedUrl));              // 4
    }

    @Test
    public void basicAuth() throws Exception {
        Request.Builder builder = mOriginRequest.newBuilder();
        mYLAuthInterceptor.basicAuth(builder, mOriginRequest.url(), mTimestamp);
        Request newRequest = builder.build();
        
        HttpUrl expectedUrl = HttpUrl.parse(SERVER_ENDPOINT
                + "/users/1905378617"
                + "?timestamp=1438141764"
                + "&mac=883df97d47e60a51236c4b08e82b0aa4be0076b2");
        assertThat(newRequest.url(), is(expectedUrl));
        assertThat(newRequest.headers("Authorization"),             // 5
                is(Collections.singletonList("Basic dGVzdF9jbGllbnQ6dGVzdF9wYXNz")));
    }
}
~~~

测试代码里面常量有点多，不过总的来说代码还是挺漂亮的：

1. 我们把多个测例都需要的逻辑都放到 setUp 函数中。
2. 时间戳我们在编码实现的时候就考虑到了测试，所以这里我们的测试非常稳定。
3. 验证时预期结果是怎么来的？手动计算测试数据的输出，或者使用一个正确的标准测试用例数据。千万不要先跑一遍，把输出作为预期，否则测试的正确性就被“强制保证”了。
4. 这里我们用了一些 hamcrest 的 assertion 和 matcher，功能要比 junit 内置的更强大一些，测试结果的信息也更丰富一些。
5. basic auth 别忘了验证 header。

## 4，小结

好了，三板斧的回顾、网络层“微架构”的概览、以及认证功能的方案与实现就讲到这里。在接下来的第二篇中，我将讲讲 JSON 转换中的两点注意事项，欢迎继续阅读 [RESTful 安卓网络层解决方案（二）：空 JSON 和 API Error 解析](/2016/09/04/RESTful-Android-Network-Solution-2/){:target="_blank"}。

## Bonus：拆轮子与 model 层架构推荐

前段时间拆轮子系列的前三篇，分别对 [Retrofit](https://github.com/square/retrofit){:target="_blank"}，[OkHttp](https://github.com/square/okhttp){:target="_blank"} 和 [Okio](https://github.com/square/okio){:target="_blank"} 源码进行了分析和源码导读，发布之后大家反馈还不错，其中拆 OkHttp 篇成功登上[开发者头条榜首](http://toutiao.io/top/2016-07-14){:target="_blank"}。没有看过的朋友建议大家可以看一看：

+ [拆轮子系列：拆 Retrofit](/2016/06/25/Understand-Retrofit/){:target="_blank"}
+ [拆轮子系列：拆 OkHttp](/2016/07/11/Understand-OkHttp/){:target="_blank"}
+ [拆轮子系列：拆 Okio](/2016/08/04/Understand-Okio/){:target="_blank"}

此外，之前整理的安卓 model 层架构，有幸还在 GDG 进行了一次分享，大家反响也还不错，在这里也推荐大家看一看：

+ [完美的安卓 model 层架构（上）](/2016/05/06/Perfect-Android-Model-Layer/){:target="_blank"}
+ [完美的安卓 model 层架构（下）](/2016/05/12/Perfect-Android-Model-Layer-2/){:target="_blank"}
+ [08/07 北京 GDG Android Meetup 活动回顾，讲义，照片](http://mp.weixin.qq.com/s?__biz=MzA5MDg3MjczMg==&mid=2652003543&idx=1&sn=849c06ac198cbfe9cdcfae90b2a17021&scene=1&srcid=0902QGgAZZKCpZbNNPD66mnu#rd){:target="_blank"}
