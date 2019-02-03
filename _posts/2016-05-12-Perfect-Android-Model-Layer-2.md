---
layout: post
title: 完美的安卓 model 层架构（下）
tags:
    - 安卓开发
    - 架构
---

在[完美的安卓 model 层架构（上）](/2016/05/06/Perfect-Android-Model-Layer/index.html){:target="_blank"}中，我主要介绍了网络请求、数据库持久化、Immutable/Value types、Json 序列化与反序列化这四部分内容，而剩下的关于 Parcelable，ZonedDateTime，null safety，rx error handling，config injection以及测试相关的内容，将在本篇中进行介绍。

声明：本文已独家授权微信公众号Android程序员（AndroidTrending）在微信公众号平台原创首发。

## 1. 回顾
先回顾一下上篇中的架构：

![架构图](/img/201605/perfect_android_model_layer.png)

总的来说，就是：

+ 利用 OkHttp 和 Retrofit 进行网络请求；
+ 利用 SqlDelight、AutoValue 及其系列扩展生成我们的 model 类，大大减少我们需要编写的代码；
+ 利用 Gson 进行 model 类的 Json 序列化和反序列化；
+ 利用 SqlBrite 提供对数据库访问的 reactive API；
+ 最后由于 Retrofit 和 SqlBrite 都提供了 reactive API，我们的业务逻辑代码就可以尽情享用响应式编程的强大了。

接下来的部分，就是对上面架构的各个部分的细化和完善。

+ model 类实现 Parcelable 以便于 Activity 和 Fragment 传参；
+ 使用 ZonedDateTime 表达时间；
+ Rx Retrofit error processor 统一处理网络错误；
+ model 的 ProGuard 配置注意事项；
+ Config Injection：把业务相关的配置注入到业务无关的 model 架构中；
+ 使用 delegate 接口层隔离安卓系统，便于单元测试，以及 UnMock Plugin 和 RestMock 工具的使用；

## 2. Parcelable
我们的数据类，很可能需要在 `Activity`，`Fragment` 以及 `View` 之间进行传递，一方面作为参数进行传递，另一方面，当发生屏幕旋转，以及其他情况下 `Activity` 被销毁时，我们需要保存和恢复状态。而这些使用场景中 `Parcelable` 都扮演着非常重要的角色。

### 2.1. Activity 传参
`Activity` 传参的原理就是在构造启动 `Activity` 的 `Intent` 时，调用 `intent.putExtra()` 函数，把参数保存到 `Intent` 对象中。而在目标 `Activity` 中，调用 `getIntent()` 函数获取到 `Intent` 对象，并从中读取参数。

允许的参数类型除了基本类型（primitive type）及其包装对象类型外，还支持 `Serializable` 和 `Parcelable`。所以如果我们的数据类型实现了 `Parcelable` 接口，那就可以直接使用了。

而实现 `Parcelable` 对于已经集成了 AutoValue 的我们来说，简直不能更简单了。我们只需要引入 [auto-value-parcel](https://github.com/rharter/auto-value-parcel){:target="_blank"}，并让我们的数据类型 `implements Parcelable` 即可：

``` java
@AutoValue
public abstract class GithubUser implements GithubUserModel, Parcelable {
    // ...
}
```

auto-value-parcel 和上篇中引入的 auto-value-gson 都是 AutoValue 的一个扩展，作者也是同一个人。经过上述修改之后，auto-value-parcel 会自动为我们生成实现 `Parcelable` 所需要的代码，在 `Activity` 之间传参的时候，我们直接调用 `putExtra` 和 `getParcelableExtra` 即可。

``` java
final class AutoValue_GithubUser extends $AutoValue_GithubUser {
    public static final Parcelable.Creator<AutoValue_GithubUser> CREATOR =
            new Parcelable.Creator<AutoValue_GithubUser>() {
                @Override
                public AutoValue_GithubUser createFromParcel(Parcel in) {
                    return new AutoValue_GithubUser(
                            in.readInt() == 0 ? (Long) in.readSerializable() : null,
                            in.readString(), in.readString(), in.readString(),
                            in.readInt() == 0 ? (ZonedDateTime) in.readSerializable() : null);
                }

                @Override
                public AutoValue_GithubUser[] newArray(int size) {
                    return new AutoValue_GithubUser[size];
                }
            };

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        if (id() == null) {
            dest.writeInt(1);
        } else {
            dest.writeInt(0);
            dest.writeSerializable(id());
        }
        dest.writeString(login());
        dest.writeString(avatar_url());
        dest.writeString(type());
        if (created_at() == null) {
            dest.writeInt(1);
        } else {
            dest.writeInt(0);
            dest.writeSerializable(created_at());
        }
    }

    @Override
    public int describeContents() {
        return 0;
    }
    
    // ...
}
```

写过 `Activity` 传参代码的朋友肯定知道，读写 `Intent` 中的 extra 都需要指定一个 `String` 作为 key，代码比较繁琐，因此这里我们可以引入 [IntentBuilder](https://github.com/emilsjolander/IntentBuilder){:target="_blank"}，我们只需要为参数添加注解，就可以完成参数传递，让代码瞬间简洁不少，例如这样：

``` java
@IntentBuilder
class DetailActivity extends Activity {
    @Extra
    String id;
    @Extra @Nullable
    String title;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        DetailActivityIntentBuilder.inject(getIntent(), this);
        // inject 之后，就可以使用 id 和 title 了
    }
}

// ...

startActivity(new DetailActivityIntentBuilder("12345")
    .title("MyTitle")
    .build(context))
```

### 2.2. Fragment 传参
`Fragment` 传参和 `Activity` 原理类似，`Fragment` 提供了 `setArguments()` 和 `getArguments` 方法，而这个 argument，就是 `Bundle` 对象，只要我们的数据类实现了 `Parcelable`，就可以通过 `Bundle` 进行传递了。

同样，为了保持代码的简洁，我们可以引入 [FragmentArgs](https://github.com/sockeqwe/fragmentargs){:target="_blank"}，其使用例子如下：

``` java
@FragmentWithArgs
public class MyFragment extends Fragment {
    @Arg
    int id;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        FragmentArgs.inject(this); 
        // inject 之后，就可以使用 id 了
    }
}

// ...

MyFragment fragment = MyFragmentBuilder.newMyFragment(101);
```

可能有朋友在想，Activity 传参和 Fragment 传参需要使用两个不同的库，能不能把它们统一起来？好消息是这个问题早就有人想到，并且已经造好轮子等着我们来用了：[AutoBundle](https://github.com/yatatsu/AutoBundle){:target="_blank"}。实际上 AutoBundle 所做的就是把上述两个库整合了起来，因此 AutoBundle 的用法也和它们类似，这里就不赘述了，大家可以自行查看其项目主页。

### 2.3. Activity，Fragment 和 View 的状态保存
`Activity`，`Fragment` 和 `View` 的状态保存使用的同样是 `Bundle`，而同样也有一个好用的轮子让我们保存状态变得异常简洁：[Icepick](https://github.com/frankiesardo/icepick){:target="_blank"}。

``` java
class ExampleActivity extends Activity {
  @State String username;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Icepick.restoreInstanceState(this, savedInstanceState);
  }

  @Override public void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    Icepick.saveInstanceState(this, outState);
  }
}
```

## 3. ZonedDateTime
细心地朋友可能已经注意到，在上篇中，我们 `GithubUser` 的 `created_at` 成员类型是个生面孔：`ZonedDateTime`。它是何方神圣？

简单来说，Java 8 引入了一套新的 API 来替代 `Date` 和 `Calendar` 等 API，代号 JSR-310，JSR-310 的实现者为其提供了一个向后兼容的版本 ThreeTenBP，而由于 ThreeTenBP 把时区数据保存在了 jar 包中，在安卓平台上存在严重的性能问题，因此 JakeWharton 又提供了一个改良版本：[ThreeTenABP](https://github.com/JakeWharton/ThreeTenABP){:target="_blank"}。ThreeTenABP 就是本节主角了。

为什么要使用 JSR-310 ？因为 `Date` 和 `Calendar` 太难用了！计算时间差，判断先后关系，根据已有时间对象创建新的时间对象，都非常麻烦而且极易出错。为什么不用 ThreeTenBP ？上面说了，存在性能问题，更多关于安卓平台读取 jar 包资源性能问题的内容，可以阅读[我翻译的 NimbleDroid 团队的博客：揭秘在安卓平台上奇慢无比的 ClassLoader.getResourceAsStream](http://blog.nimbledroid.com/2016/04/06/slow-ClassLoader.getResourceAsStream-zh.html){:target="_blank"}。

ZonedDateTime 的具体使用以及 ThreeTenABP 的设置，我就不赘述了，百分百推荐，值得拥有！

## 4. SqlDelight 的 null safety
细心地朋友可能已经注意到在上篇中，`GithubUserModel` 和 `ZonedDateTimeDelightAdapter` 的代码有些奇怪之处：

``` java
public interface GithubUserModel {
  // ...
  
  @Nullable
  ZonedDateTime created_at();

  // ...
}

public class ZonedDateTimeDelightAdapter implements ColumnAdapter<ZonedDateTime> {
    // ...

    @NonNull
    @Override
    public ZonedDateTime map(final Cursor cursor, final int columnIndex) {
        return mDateTimeFormatter.parse(cursor.getString(columnIndex), ZonedDateTime.FROM);
    }

    @Override
    public void marshal(final ContentValues values, final String key,
            @NonNull final ZonedDateTime value) {
        values.put(key, mDateTimeFormatter.format(value));
    }
}
```

在 `GithubUserModel` 中，`created_at` 是 `Nullable`，但是在 `ZonedDateTimeDelightAdapter` 中，我们却使用了 `NonNull`，虽然这两个注解并没有强制作用，但我们当然需要遵守它们的语义。

目前 SqlDelight 对 null safety 的处理并不是十分完善，详情可以参考[其 Github 项目主页的这个 issue](https://github.com/square/sqldelight/issues/145){:target="_blank"}。

生成的 `Mapper` 类已经很好地处理了 null safety 的问题了，但我们需要重写我们 `Marshal` 类的 `created_at` 方法，来实现处理 `created_at` 为空的逻辑：

``` java
final class Mapper<T extends GithubUserModel> implements RowMapper<T> {
    // ...

    @Override
    @NonNull
    public T map(@NonNull Cursor cursor) {
        return creator.create(cursor.isNull(cursor.getColumnIndex(ID)) ? null
                        : cursor.getLong(cursor.getColumnIndex(ID)),
                cursor.getString(cursor.getColumnIndex(LOGIN)),
                cursor.getString(cursor.getColumnIndex(AVATAR_URL)),
                cursor.getString(cursor.getColumnIndex(TYPE)),
                // 这里已经处理好了 null 的问题
                cursor.isNull(cursor.getColumnIndex(CREATED_AT)) ? null
                        : created_atAdapter.map(cursor, cursor.getColumnIndex(CREATED_AT)));
    }

    // ...
}

public static class Marshal extends GithubUserMarshal<Marshal> {
    // ...

    @Override
    public Marshal created_at(@Nullable final ZonedDateTime createdAt) {
        // 如果是 null，就什么也不做
        if (createdAt == null) {
            return this;
        }
        return super.created_at(createdAt);
    }
}
```

在上篇中我们提到，`Mapper` 负责把 `Cursor` 对象转换为 `GithubUser` 对象，`Marshal` 负责把一个 `GithubUser` 对象转化为一个 `ContentValues` 对象，用于保存到数据库中。`Mapper` 和 `Marshal` 处理好了 null safety 之后，数据库、转换过程、使用方，就都可以很好地处理 null 的问题了，让我们的代码从此少一些 `NullPointerException`。

## 5. Rx Retrofit error processor
这一节需要一点 RxJava 的基础，不熟悉的朋友可以先去[官网了解一下 RxJava](http://reactivex.io/intro.html){:target="_blank"}。

简而言之，因为网络请求出现错误的可能性很多，其中一部分是服务器返回的 API Error，不同的 API 调用可能需要根据 API Error 的具体内容进行不同处理，而其他的错误则可以统一的处理，例如提示网络异常。因此我们需要一个统一的 handler 来检查是否为 API Error，如果是则需要进行类型转换，以便于使用方进行处理。其次，RxJava 的 `Observable` subscribe 时需要处理 `onError` 事件，对于可以统一处理的错误，我们也可以复用一个 `RxErrorProcessor`。

在 Retrofit 1.x 中，`RestAdapter.Builder` 有一个 `setErrorHandler` 方法，可以在 Retrofit 返回的 `Observable` 调用 `onError` 之前，对错误进行一个集中的判断和转化。但是 Retrofit 2.x 移除了这个接口，我们在 `Observable` 调用 `onError` 之前没有机会对错误进行一次集中的转换了。

不过这并不是什么大问题，正好我们还可以把 handler 和 processor 统一起来。所以我们把错误判断与转换和统一处理逻辑都统一到 `Subscriber` 的 `onError` 方法中。我们的集中处理逻辑可以是这样的：

``` java
@Singleton
public class RxNetErrorProcessor implements Action1<Throwable> {

    private final Gson mGson;

    @Inject
    RxNetErrorProcessor(final Gson gson) {
        mGson = gson;
    }

    @Override
    public void call(final Throwable throwable) {
        Timber.e(throwable, "RxNetErrorProcessor");
    }

    public boolean tryWithApiError(final Throwable throwable, final Action1<ApiError> handler) {
        if (throwable instanceof HttpException) {
            final HttpException exception = (HttpException) throwable;
            try {
                final String errorBody = exception.response().errorBody().string();
                final ApiError apiError = mGson.fromJson(errorBody, ApiError.class);
                if (!TextUtils.isEmpty(apiError.message())) {
                    handler.call(apiError);
                    return true;
                }
            } catch (Exception e) {
                call(throwable);
            }
        } else {
            call(throwable);
        }
        return false;
    }
}
```

这里我们先判断异常是否为 `HttpException`，如果是则尝试把 error body 转化为 `ApiError` 对象，它是服务器定义的错误信息对象，如果是一个合法的 `ApiError` 对象，我们就使用传入的 `handler` 进行处理，否则我们就使用一个统一的处理方式处理。

结合 lambda 表达式，我们的使用方代码将异常简洁：

``` java
mGithubUserDao.searchUser(query)
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(users -> getView().showSearchResult(users),
                t -> mRxNetErrorProcessor.tryWithApiError(t,
                        e -> getView().showError(e.message())));
```

## 6. ProGuard
虽然项目是开源的，但商业项目 ProGuard 还是不可或缺的，所以还是需要保证在开启 ProGuard 之后能够正常运行。

在这套 model 层架构中，关于 ProGuard 有几点值得一提：

### 6.1. AutoParcel
安卓系统对 Parcelable 的序列化和反序列化，需要我们的类中有一个名为 `CREATOR` 的成员，并且它不能进行混淆，否则会报错。

```
# AutoParcel
-keep class **AutoValue_*$1 { }
-keepclassmembers class * implements android.os.Parcelable {
    static ** CREATOR;
}
```

## 7. Config Injection
在 AndroidTDDBootStrap 项目中，`Retrofit`，`EventBus`，`SqlBriteDatabase` 等对象的创建都在 base module 中，它们的创建需要配置 base url，是否 debug 等参数，但是这些参数都是和业务具体相关的，当然不应该出现在 base module 中，怎么办呢？依赖注入呀！由于注入的是配置参数，所以我称之为 Config Injection。

依赖注入目前应该算是广为人知了，依赖注入可以让我们的代码更加解耦，更 SOLID，更利于测试，依赖注入框架我使用的是 [dagger2](https://github.com/google/dagger){:target="_blank"}。关于 dagger2 的使用细节，不熟悉的朋友可以先看一下 dagger2 主页。

下面我以 Retrofit 的配置注入为例，其他的配置注入都与之类似，可以参见[项目源码](https://github.com/Piasy/AndroidTDDBootStrap-base/tree/master/src/main/java/com/github/piasy/base/model/provider){:target="_blank"}。

``` java
// RetrofitConfig.java，位于 base module 中：
@AutoValue
public abstract class RetrofitConfig {
    public static Builder builder() {
        return new AutoValue_RetrofitConfig.Builder();
    }

    public abstract String baseUrl();

    @AutoValue.Builder
    public abstract static class Builder {
        public abstract Builder baseUrl(final String baseUrl);

        public abstract RetrofitConfig build();
    }
}

// ProviderModule.java，位于 base module 中：
@Module
public class ProviderModule {
    // ...

    @Singleton
    @Provides
    Retrofit provideRetrofit(final RetrofitConfig config, final OkHttpClient okHttpClient,
            final Gson gson) {
        return new Retrofit.Builder().baseUrl(config.baseUrl())
                .client(okHttpClient)
                .addConverterFactory(GsonConverterFactory.create(gson))
                .addCallAdapterFactory(
                        RxJavaCallAdapterFactory.createWithScheduler(Schedulers.io()))
                .build();
    }

    // ...
}

// ProviderConfigModule.java，位于 app module 中：
@Module
public class ProviderConfigModule {

    private static final boolean DEBUG = "debug".equals(BuildConfig.BUILD_TYPE);

    // ...

    @Singleton
    @Provides
    RetrofitConfig provideRestConfig() {
        return RetrofitConfig.builder().baseUrl(BuildConfig.API_BASE_URL).build();
    }

    // ...
}
```

然后我们把 `ProviderModule` 和 `ProviderConfigModule` 都添加到目标 component 的 modules 列表中，就可以利用 dagger2 来进行依赖创建和依赖注入了。 是不是非常优雅？

## 8. 单元测试相关
终于到了最后一部分了：单元测试。为什么要进行单元测试这个问题讲得太多了，我这里就不再多说了。测试相关的内容，主要启发自 [Square 团队分享的单元测试系列文章](http://www.philosophicalhacker.com/2015/05/01/how-to-make-our-android-apps-unit-testable-pt-1){:target="_blank"}。

### 8.1. The Square Way
square way 的思路很简单，通过引入一层 delegate 接口，我们可以把我们的业务逻辑代码和安卓系统隔离开来，这样我们的业务逻辑代码就和安卓系统没有耦合了，delegate 接口我们可以随意 mock，因此我们就完全可以编写在 JVM 上运行的测试用例。当然也存在 Robolectric 这样的框架，可以在 JVM 上运行安卓测例，但我更倾向于 square way 这样的方式。因为它不仅能让我们更快的执行测试用例，还会让我们的代码更加解耦，更 SOLID。

这里我以 AndroidTDDBootStrap 项目的 `com.github.piasy.gh.model.users.dao` 包为例，分享 square way 的实践方式。

首先看看包结构：

```
com.github.piasy.gh.model.users.dao -
        - DbUserDelegate.java
        - DbUserDelegateImpl.java
        - GithubUserRepo.java
```

`DbUserDelegate` 就是负责代理数据库操作的，它的接口如下：

``` java
public interface DbUserDelegate {

    void deleteAllGithubUser();

    void putAllGithubUser(List<GithubUser> users);

    Observable<List<GithubUser>> getAllGithubUser();
}
```

`GithubUserRepo` 的方法如下：

``` java
@NonNull
public Observable<List<GithubUser>> searchUser(@NonNull final String query) {
    return mGithubApi.searchGithubUsers(query, GithubApi.GITHUB_API_PARAMS_SEARCH_SORT_JOINED,
            GithubApi.GITHUB_API_PARAMS_SEARCH_ORDER_DESC)
            .map(GithubUserSearchResult::items)
            .doOnNext(mDbUserDelegate::putAllGithubUser);
}
```

那么我们需要测试一下正常情况下，搜索到结果之后，会不会被保存到数据库中，如果发生了错误，会不会调用 `mDbUserDelegate.putAllGithubUser` 接口。由于 delegate 层的存在，我们完全可以编写普通的 JUnit 测试了，测例之一如下：

``` java
@Test
public void testSearchUserSuccess() {
    // given
    willReturn(Observable.create(new Observable.OnSubscribe<GithubUserSearchResult>() {
        @Override
        public void call(final Subscriber<? super GithubUserSearchResult> subscriber) {
            subscriber.onNext(mEmptyResult);
            subscriber.onCompleted();
        }
    })).given(mGithubApi).searchGithubUsers(anyString(), anyString(), anyString());

    // when
    final TestSubscriber<List<GithubUser>> subscriber = new TestSubscriber<>();
    mGithubUserDao.searchUser("Piasy").subscribe(subscriber);
    subscriber.awaitTerminalEvent();

    // then
    then(mDbUserDelegate).should(timeout(100))
            .putAllGithubUser(anyListOf(GithubUser.class));
    verifyNoMoreInteractions(mDbUserDelegate);
    subscriber.assertNoErrors();

    then(mGithubApi).should(timeout(100).only())
            .searchGithubUsers(anyString(), anyString(), anyString());
}
```

RxJava 也为我们提供了 `TestSubscriber` 供测试使用，让我们的测例非常简洁。

关于测试，还有两点值得一提。

### 8.2. UnMock Plugin
运行 JVM/Robolectric 测试的时候，我们经常会遇见 `*** not mocked` 的错误，这是由于我们在 JVM 上运行测例，如果调用到安卓系统的方法，而且又没有对这些方法进行 mock，就会报这样的错误。而 [UnMock Plugin](https://github.com/bjoernQ/unmock-plugin){:target="_blank"} 就是一个致力于解决这个麻烦的 gradle 插件，它可以指定安卓系统代码的实现版本 jar 包，能指定保留哪些类或者方法，这样这些类和方法就可以在测试中被正常调用了。

它的配置方式类似于这样（完整例子请参考其项目主页）：

``` gradle
unMock {
    downloadFrom 'https://oss.sonatype.org/content/groups/public/org/robolectric/android-all/6.0.0_r1-robolectric-0/android-all-6.0.0_r1-robolectric-0.jar'

    keep "android.os.Looper"
    keep "android.content.ContentValues"
    keepStartingWith "android.util."
}
```

首先指定了安卓代码的实现版本，我们使用了 Robolectric 提供的 jar 包，然后我们指定保留 `Looper` 和 `ContentValues` 类，以及 `android.util` 包及其子包中的类。

有了 UnMock Plugin，我们在 JVM 上编写安卓单元测试简直如虎添翼！

### 8.3. RestMock
在 Mockito 官网上有这样一句话：

> Don’t mock everything

在编写测试的时候，我们应该尽可能少地进行 mock，尤其是在编写集成测试的时候。例如，能在 OkHttp 层提供 mock 的数据，就不要 mock OkHttpClient，也不要 mock Retrofit，更不要 mock 定义的 RESTful API。

OkHttp 为我们提供了 [MockWebServer](https://github.com/square/okhttp/blob/master/mockwebserver){:target="_blank"}，让我们可以在 OkHttp 层提供 mock 数据。我曾在之前的文章 [（可能是）目前最全面的Android Espresso配置指南了](/2016/03/13/Android-Espresso-test-start/index.html){:target="_blank"} 中介绍过如何使用 MockWebServer 来返回 mock 的数据，但今天我们有了更加便捷的工具：[RestMock](https://github.com/andrzejchm/RESTMock){:target="_blank"}。

RestMock 是对 MockWebServer 的一层封装，让我们可以更加便捷地定义 网络请求要返回的 mock 数据。

一个简单地使用例子是这样（完整例子请参考其项目主页）：

``` java
RESTMockServer.whenGET(pathStartsWith("/search/users?"))
        .thenReturnString(200, MockProvider.provideSimplifiedGithubUserSearchResultStr());
```

是不是超级简洁？而 RestMock 还提供了网络请求路径匹配的强大 matcher，以及网络请求调用的 verifier，让我们 mock 网络请求和网络请求测试从此变得优雅而简洁。

## 9. 总结
引用一句[移动开发每周阅读清单第十一期](http://mobilefrontier.github.io/articles/weekly-11/#rd){:target="_blank"}对上篇文章的介绍：

> 无论是MVC、MVP还是MVVM，Model的角色都非常重要，合理的Model设计对整个项目的架构有着至关重要的作用。

这套近乎完美 model 层的架构，终于介绍完了，与其说是提出，说是 _发现_ 更合理。

+ 利用 OkHttp 和 Retrofit 进行网络请求；
+ 利用 SqlDelight、AutoValue 及其系列扩展生成我们的 model 类；
+ 利用 Gson 进行 model 类的 Json 序列化和反序列化；
+ 利用 SqlBrite 提供对数据库访问的 reactive API；
+ model 类实现 Parcelable 以便于 Activity 和 Fragment 传参；
+ 使用 ZonedDateTime 表达时间；
+ Rx Retrofit error processor 统一处理网络错误；
+ model 的 ProGuard 配置注意事项；
+ Config Injection：把业务相关的配置注入到业务无关的 model 架构中；
+ 使用 delegate 接口层隔离安卓系统，便于单元测试，以及 UnMock Plugin 和 RestMock 工具的使用；
