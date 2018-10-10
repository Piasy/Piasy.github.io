---
layout: post
title: RESTful 安卓网络层解决方案（二）：空 JSON 和 API Error 解析
tags:
    - 安卓开发
    - 架构
---

在[拆轮子系列：拆 Okio](/2016/08/04/Understand-Okio/){:target="_blank"} 最后我曾说过会对 Retrofit、OkHttp、Okio 三者进行一个小结，并且整理一套网络层的“微架构”，今天终于得以完成，在这里一起奉送给大家 :)

+ [RESTful 安卓网络层解决方案（一）：概览与认证实现方案](/2016/08/29/RESTful-Android-Network-Solution-1/){:target="_blank"}
+ 🏁 RESTful 安卓网络层解决方案（二）：空 JSON 和 API Error 解析
+ [RESTful 安卓网络层解决方案（三）：API model 与 Business model 分离](/2016/09/04/RESTful-Android-Network-Solution-3/){:target="_blank"}

## 1，JSON 解析需求

JSON 应该是大部分项目 CS 通信的数据格式，相比于简单、调试友好的优势，它的性能不足几乎不足一提，毕竟绝大多数情况下，它都不会成为性能的瓶颈。在 Retrofit + Gson 的方案中，我们有两个问题需要特殊处理。

首先，如果一个 API 请求不需要返回数据，很可能我们的服务器也就不会返回数据（返回空的 response body），而空字符串并不是合法的 JSON，所以 Square 实现的 `GsonResponseBodyConverter` 会不认账，直接抛出 JSON 解析错误。关于这个问题更多的讨论，可以看一下 [Retrofit 的这个 issue：#1554 Handle Empty Body](https://github.com/square/retrofit/issues/1554){:target="_blank"}。

其次，很多公司的后端程序都会把 API Error 的 HTTP status code 设置为 200，这样我们就没法利用 OkHttp 的错误处理来解析 API Error 了，我们需要先尝试把响应数据解析为 API Error，如果不是 API Error，再解析为目标类型。

在 Retrofit 1.x 中，Gson 解析是通过设置一个自定义 Converter 来实现的，我们尝试解析为 API Error 的代码自然也在其中，但 Retrofit 在 2.x 中，单独实现了各种常用的 Converter，它们是没法实现我们这种解析需求的。

怎么办呢？其实如果顺着这个思路，答案也会很直接，自己实现一个 Converter 就好了嘛！

但遗憾的是，我在项目重构时没有想到这种方案，而是采用了 Interceptor 的方案，也许是思路被 [YLAuthInterceptor](/2016/08/29/RESTful-Android-Network-Solution-1/#ylauthinterceptor-){:target="_blank"} 给限制住了。这种方案其实也还说得过去，在拿到网络 response 之后，先拿到数据，再尝试转换为 API Error，如果成功，就抛出这个 API Error，否则返回 response。

但显然让 Converter 来做这件事更加合理，这完全是一件 response 转换的事情，如果说 API error 的响应会带着特殊的 header，那放在 interceptor 层来做就还是合理的。

所以加上这个需求，我们的 converter 需要实现三个功能：JSON 转换、空字符串处理、API Error 检查。

## 2，解决方案设计

明确了需求之后，有的朋友可能会把这三个功能都放到一起，用一个类来实现，至于 JSON 转换的功能，可以直接把 `retrofit-converter-gson` 的代码 copy 进来，还省得自己实现。

但这样真的好吗？

首先，copy 别人的代码，就意味着我们也需要对它进行维护，他们发布新版本之后，我们需要把最新的代码再次 copy 进来，这显然是在徒增成本。其次，一个类负责三件事情，一点都不“单一职责”。

那怎么办才好呢？

这里我们现学现卖，[Okio 不是很好的践行了“修饰模式”嘛](/2016/08/04/Understand-Okio/#source--sink){:target="_blank"}，我们这边也可以这么做，动态为 converter 增加功能。这边我们额外实现两个类：`EmptyJsonLenientConverterFactory` 和 `YLApiErrorAwareConverterFactory`，前者负责处理空 JSON 字符串，后者则用来捕获 API Error。

## 3，处理空 JSON 字符串

### 3.1，`EmptyJsonLenientConverterFactory`

``` java
public class EmptyJsonLenientConverterFactory extends Converter.Factory {

    private final GsonConverterFactory mGsonConverterFactory;       // 1

    public EmptyJsonLenientConverterFactory(
            GsonConverterFactory gsonConverterFactory) {
        mGsonConverterFactory = gsonConverterFactory;
    }

    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type,
            Annotation[] parameterAnnotations,
            Annotation[] methodAnnotations,
            Retrofit retrofit) {
        return mGsonConverterFactory.requestBodyConverter(type,     // 2
                parameterAnnotations, methodAnnotations, retrofit);
    }

    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type,
            Annotation[] annotations,
            Retrofit retrofit) {
        final Converter<ResponseBody, ?> delegateConverter =        // 3
                mGsonConverterFactory.responseBodyConverter(type,
                        annotations, retrofit);
        return value -> {
            try {
                return delegateConverter.convert(value);            // 4
            } catch (EOFException e) {
                // just return null
                return null;                                        // 5
            }
        };
    }
}
```

总的来说还是比较直观的：

1. 修饰模式要求我们实现同样的接口，并且进行一定程度的委托，我们这边明确就是对 `GsonConverterFactory` 的功能进行扩充，所以我们的委托类型就直接声明为它。
2. request body 我们无需特殊处理，直接返回 `GsonConverterFactory` 创建的 converter。
3. 我们返回的 converter 可能会被多次使用，所以不要在匿名 converter 实例中创建委托 converter，而是只在外面创建一次。
4. 尝试把请求转发给 `GsonConverterFactory` 创建的 converter。
5. 如果抛出了 `EOFException`，则说明遇到了空 JSON 字符串，那我们直接返回 `null`。

### 3.2，单元测试

同样，我们要编写单元测试，增加我们的信心。

**任何事情都不要极端，写完代码之后对着每个 `if-else` 分支编写测试用例是没必要的，这也会让我们抵触编写测试，因为这样做会让我们觉得测试代码都是重复的“废话”。合理的做法是我们首先就设计一些考察要点，用它们来验证我们的代码是否正确。其实如果不写测试，我们是怎么确保代码正确的呢？还是靠这些“潜在的”测例！所以何不先就把测例准备好呢？何不先就把测试代码写好呢？而这就是 TDD。**

我们先看一下测试要点：

``` java
public class EmptyJsonLenientConverterFactoryTest {

    private Retrofit mRetrofit;
    private EmptyJsonLenientConverterFactory mFactory;

    @Before
    public void setUp() {
        mRetrofit = new Retrofit.Builder()
                .baseUrl(SERVER_ENDPOINT)
                .build();
        mFactory = new EmptyJsonLenientConverterFactory(
                GsonConverterFactory.create());
    }

    @Test
    public void convertNormalJson() 
            throws IOException {
        // 验证正常 JSON 能正确解析
    }

    @Test(expected = EOFException.class)
    public void gsonConverterFailOnEmptyJson() 
            throws IOException {
        // 验证 GsonConverter 无法处理空字符串
    }

    @Test
    public void convertEmptyJson() 
            throws IOException {
        // 验证我们的 converter 可以处理空字符串
    }
}
```

再看 `convertNormalJson()`：

``` java
public static ResponseBody stringBody(String body) {        // 1
    return ResponseBody.create(
            MediaType.parse("application/json"), body);
}

@Test
public void convertNormalJson()
        throws IOException {
    String normalJson = "{\"request\":\"req\","
            + "\"errcode\":123,"
            + "\"errmsg\":\"qw\"}";
    Converter<ResponseBody, ?> converter =
            mFactory.responseBodyConverter(YLApiError.class,
                    EMPTY_ANNOTATIONS, mRetrofit);
    Object response = converter.convert(stringBody(normalJson));
    assertTrue(response instanceof YLApiError);
    YLApiError apiError = (YLApiError) response;
    assertEquals(123, apiError.getErrcode());
}
```

测试代码也要保持简洁优雅，否则我们也会对编写测试产生抵触，所以这里我把从 String 创建 `ResponseBody` 的代码封装了一个函数（1）。

再看 `gsonConverterFailOnEmptyJson()`：

``` java
@Test(expected = EOFException.class)                    // 1
public void gsonConverterFailOnEmptyJson()
        throws IOException {
    String emptyJson = "";
    Converter<ResponseBody, ?> converter =
            GsonConverterFactory.create().responseBodyConverter(
                    YLApiError.class, EMPTY_ANNOTATIONS, mRetrofit);
    converter.convert(stringBody(emptyJson));
}
```

这里我们利用 JUnit 的注解来验证测例抛出了 `EOFException`（1）。

最后我们看看 `convertEmptyJson()`，它就非常简单了：

``` java
@Test
public void convertEmptyJson()
        throws IOException {
    String emptyJson = "";
    Converter<ResponseBody, ?> converter =
            mFactory.responseBodyConverter(YLApiError.class,
                    EMPTY_ANNOTATIONS, mRetrofit);
    Object response = converter.convert(stringBody(emptyJson));
    assertNull(response);
}
```

## 4，解析 API Error

### 4.1，`YLApiErrorAwareConverterFactory`

``` java
public class YLApiErrorAwareConverterFactory extends Converter.Factory {

    private final Converter.Factory mDelegateFactory;           // 1

    public YLApiErrorAwareConverterFactory(
            Converter.Factory delegateFactory) {
        mDelegateFactory = delegateFactory;
    }

    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type,
            Annotation[] parameterAnnotations,
            Annotation[] methodAnnotations,
            Retrofit retrofit) {
        return mDelegateFactory
                .requestBodyConverter(type, parameterAnnotations,
                        methodAnnotations, retrofit);
    }

    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type,
            Annotation[] annotations,
            Retrofit retrofit) {
        final Converter<ResponseBody, ?> apiErrorConverter =    // 2
                mDelegateFactory.responseBodyConverter(YLApiError.class,
                        annotations, retrofit);
        final Converter<ResponseBody, ?> delegateConverter =
                mDelegateFactory.responseBodyConverter(type,
                        annotations, retrofit);
        return value -> {
          // read them all, then create a new ResponseBody for ApiError
          // because the response body is wrapped, 
          // we can't clone the ResponseBody correctly
          MediaType mediaType = value.contentType();
          String stringBody = value.string();                   // 3
          try {
              Object apiError = apiErrorConverter
                      .convert(ResponseBody.create(mediaType, stringBody));
              if (apiError instanceof YLApiError 
                      && ((YLApiError) apiError).isApiError()) {
                  throw (YLApiError) apiError;                  // 4
              }
          } catch (JsonSyntaxException notApiError) {
          }
          // then create a new ResponseBody for normal body
          return delegateConverter
                  .convert(ResponseBody.create(mediaType, stringBody));
    }
}
```

依然比较直观，不过有几点值得一提：

1. 我们的 `YLApiErrorAwareConverterFactory` 并不是明确针对哪个具体实现扩充功能的，所以我们把委托声明为接口。
2. 除了正常的 response body converter，我们还需要一个专门转化为 API Error 的 converter。
3. 这里我们必须对 `ResponseBody` 进行 clone，因为 Okio 的流都是只允许读一次的，如果我们直接对传入的参数进行操作，那后面我们尝试解析为正常 body 时就会出错了。
4. 如果确实是一个 API Error，那我们就抛出它，进入后面的错误处理流程。

**2016.09.27 更新**：由于 Retrofit 会对 OkHttp 返回的 ResponseBody 进行包装，会导致以前的 clone 无法奏效，所以这里我们直接把 body 作为 String 读出来，后面尝试解析为 ApiError 以及正常 body 时，都创建一个新的 ResponseBody。

### 4.2，单元测试

同样，先看测例结构：

``` java
public class YLApiErrorAwareConverterFactoryTest {

    private Retrofit mRetrofit;
    private YLApiErrorAwareConverterFactory mFactory;

    @Before
    public void setUp() {
        mRetrofit = new Retrofit.Builder()
                .baseUrl(SERVER_ENDPOINT)
                .build();
        EmptyJsonLenientConverterFactory delegate =
                new EmptyJsonLenientConverterFactory(
                        GsonConverterFactory.create());
        mFactory = new YLApiErrorAwareConverterFactory(delegate);
    }

    @Test
    public void nonApiError() throws IOException {
        // 验证解析正常数据
    }

    @Test
    public void apiError() throws IOException {
        // 验证解析 API Error
    }

    @Test
    public void emptyJson() throws IOException {
        // 验证空字符串不会被解析为 API Error
    }
}
```

测试代码比较简单，我就只贴一下 `apiError()` 了：

``` java
@Test
public void apiError() throws IOException {
    String errorString = "{\"request\":\"req\"," 
            + "\"errcode\":123," 
            + "\"errmsg\":\"qw\"}";
    Converter<ResponseBody, ?> converter = 
            mFactory.responseBodyConverter(
                    Dummy.class, EMPTY_ANNOTATIONS, mRetrofit);
    try {
        converter.convert(stringBody(errorString));
        assertTrue(false);
    } catch (YLApiError apiError) {
        assertEquals(123, apiError.getErrcode());       // 1
    }
}
```

这里我们没有利用 JUnit 注解来验证异常的抛出，而是手动编写了 `try-catch`，因为我们需要验证 API Error 对象的正确性（1）。

## 5，小结

好了，JSON 转换中的注意事项也就讲到这里。本文中 converter 对修饰模式的使用算是一大亮点，另外对于单元测试也进行了一定的思考和讨论。在接下来的第三篇中，我将讲讲 model 层中 API 和业务逻辑结合时的一个大问题，欢迎继续阅读 [RESTful 安卓网络层解决方案（三）：API model 与 Business model 分离](/2016/09/04/RESTful-Android-Network-Solution-3/){:target="_blank"}。

## Bonus：拆轮子与 model 层架构推荐

前段时间拆轮子系列的前三篇，分别对 [Retrofit](https://github.com/square/retrofit){:target="_blank"}，[OkHttp](https://github.com/square/okhttp){:target="_blank"} 和 [Okio](https://github.com/square/okio){:target="_blank"} 源码进行了分析和源码导读，发布之后大家反馈还不错，其中拆 OkHttp 篇成功登上[开发者头条榜首](http://toutiao.io/top/2016-07-14){:target="_blank"}。没有看过的朋友建议大家可以看一看：

+ [拆轮子系列：拆 Retrofit](/2016/06/25/Understand-Retrofit/){:target="_blank"}
+ [拆轮子系列：拆 OkHttp](/2016/07/11/Understand-OkHttp/){:target="_blank"}
+ [拆轮子系列：拆 Okio](/2016/08/04/Understand-Okio/){:target="_blank"}

此外，之前整理的安卓 model 层架构，有幸还在 GDG 进行了一次分享，大家反响也还不错，在这里也推荐大家看一看：

+ [完美的安卓 model 层架构（上）](/2016/05/06/Perfect-Android-Model-Layer/){:target="_blank"}
+ [完美的安卓 model 层架构（下）](/2016/05/12/Perfect-Android-Model-Layer-2/){:target="_blank"}
+ [08/07 北京 GDG Android Meetup 活动回顾，讲义，照片](http://mp.weixin.qq.com/s?__biz=MzA5MDg3MjczMg==&mid=2652003543&idx=1&sn=849c06ac198cbfe9cdcfae90b2a17021&scene=1&srcid=0902QGgAZZKCpZbNNPD66mnu#rd){:target="_blank"}
