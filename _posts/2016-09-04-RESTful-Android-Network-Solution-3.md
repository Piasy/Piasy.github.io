---
layout: post
title: RESTful 安卓网络层解决方案（三）：API model 与 Business model 分离
tags:
    - 安卓开发
    - 架构
---

在[拆轮子系列：拆 Okio](/2016/08/04/Understand-Okio/){:target="_blank"} 最后我曾说过会对 Retrofit、OkHttp、Okio 三者进行一个小结，并且整理一套网络层的“微架构”，今天终于得以完成，在这里一起奉送给大家 :)

+ [RESTful 安卓网络层解决方案（一）：概览与认证实现方案](/2016/08/29/RESTful-Android-Network-Solution-1/){:target="_blank"}
+ [RESTful 安卓网络层解决方案（二）：空 JSON 和 API Error 解析](/2016/09/04/RESTful-Android-Network-Solution-2/){:target="_blank"}
+ 🏁 RESTful 安卓网络层解决方案（三）：API model 与 Business model 分离

## 1，API model “碎片化”

当我们的服务端程序是用动态类型语言（例如 PHP）编写的时候，那我们得到的 API 响应就可能会比较杂乱了。

例如根据 id 获取用户信息的 API：

``` json
{
  "uid": 1905378617,
  "username": "hahaha",
  "avatar_url": "https://frontend-yoloyolo-tv.alikunlun.com/official/v3/img/pc/logo.png"
}
```

这是非好友的情况，如果是好友，情况又还不一样：

``` json
{
  "uid": 1905378617,
  "username": "hahaha",
  "avatar_url": "https://frontend-yoloyolo-tv.alikunlun.com/official/v3/img/pc/logo.png",
  "is_friend": true,
  "friend_remark": "Remarkable",
  "starred": 0
}
```

好友比非好友多了 `is_friend`，`friend_remark` 和 `starred` 这三个字段。

而如果获取自己的信息，又还不一样：

``` json
{
  "uid": 1905378617,
  "username": "hahaha",
  "avatar_url": "https://frontend-yoloyolo-tv.alikunlun.com/official/v3/img/pc/logo.png",
  "phone": "18812345678",
  "token": "wx:1905378617",
  "im_password": "6dbc987dffd33876"
}
```

相比于非好友，多了 `phone`、`token` 和 `im_password` 这三个字段。

一方面，服务端要践行信息隐藏的原则，不需要的数据就坚决不返回，这就造成即便返回的都是同样的东西（例如用户信息），但返回的字段组合却是多种多样的；另一方面，服务端使用动态类型，无需为每种字段组合创建一个类型，只需要返回时进行组装即可，这就进一步加剧了字段组合“碎片化”的问题。

如何解决这一问题呢？为每种组合创建一个类，还是把所有的字段都揉进一个类？

## 2，解决方案

上面最后提到的两种办法都有问题，但我们把它们可以结合起来。

首先，对于和 API 打交道的代码，我们把所有字段都装进一个类型，`ApiUser`。否则我们就需要定义三个 API 了，而这基本上是不可行的，当我们要获取一个用户的信息时，调用哪个接口，好友还是非好友？我们根本不知道是不是好友！

但紧接着，对于和上层业务打交道的代码，我们要分别定义不同的类型，`Self`、`Friend`、`NonFriend`，绝不包含无用的信息。并且我们把 API 隐藏起来，外部不可访问，对外暴露的接口都要把 `ApiUser` 转换为相应的 Business model。

## 3，具体实现

首先，我们把所有的用户信息字段拆分为多个接口，遵循接口隔离。之所以使用接口而不是抽象类，是为了后面进行组合时可以多实现。

### 3.1，用户信息接口

``` java
public interface UserInfoModel {
  long uid();
  @NonNull
  String username();
  @Nullable
  String avatar_url();
}

public interface FriendInfoModel {
  long uid();
  int starred();
  @Nullable
  String friend_remark();
}

interface RelationshipInfo {
    boolean is_friend();
}

interface CredentialInfo {
    @Nullable
    String phone();
    @Nullable
    String im_password();
    @Nullable
    String token();
}
```

其中 `UserInfoModel` 和 `FriendInfoModel` 是由 SqlDelight 生成，用于进行持久化，它们都需要靠 uid 进行查询，所以都包含一个 uid 字段。`RelationshipInfo` 用于区分是否是好友，`CredentialInfo` 则包含自己的信息。

接下来的内容会涉及到 SqlDelight、AutoValue 及其扩展相关的内容，对这些不熟悉的朋友，强烈建议先看一下这篇文章：[完美的安卓 model 层架构（上）](/2016/05/06/Perfect-Android-Model-Layer/){:target="_blank"}。

### 3.2，`ApiUser`

我们的 `ApiUser` 要把所有的字段都包含进来，所以要实现上面的所有接口：

``` java
@AutoValue
abstract class ApiUser implements UserInfoModel, 
    RelationshipInfo, FriendInfoModel, CredentialInfo {

    public static TypeAdapter<ApiUser> typeAdapter(final Gson gson) {
        return new AutoValue_ApiUser.GsonTypeAdapter(gson);
    }
}
```

尽管 `UserInfoModel` 和 `FriendInfoModel` 都包含 `uid()` 接口，但它们组合到一起的时候，`ApiUser` 只会获得一个 `uid()` 接口，所以没有问题。这边我们利用 auto-value 实现 immutable，利用 auto-value-gson 实现高效的 Gson 转换。

### 3.3，`NonFriend`

``` java
@AutoValue
public abstract class NonFriend implements UserInfoModel, Parcelable {

    public static NonFriend createFrom(ApiUser user) {
        // ...
    }
}
```

NonFriend 只包含了基本的用户信息，它实现了 `Parcelable`，以便在 Activity/Fragment 之间进行传递。它还提供了一个从 ApiUser 转换的工厂方法。

### 3.4，`Friend`

``` java
@AutoValue
public abstract class FriendInfo implements FriendInfoModel, Parcelable {

    public static Builder builder() {
        return new AutoValue_FriendInfo.Builder();
    }

    @AutoValue.Builder
    public abstract static class Builder {
        // ...
    }
}

@AutoValue
public abstract class Friend implements UserInfoModel, Parcelable {

    public abstract FriendInfo friendInfo();

    public static Friend createFrom(ApiUser user) {
        // ...
    }
    
    static Friend compose(UserInfoModel user, FriendInfo friendInfo) {
        // ...
    }
}
```

Friend 使用组合的方式加入 `FriendInfo`，因为 FriendInfo 是需要单独持久化的，所以它需要是一个单独的类型。

### 3.5，`Self`

``` java
@AutoValue
public abstract class Self implements UserInfoModel, 
    CredentialInfo, Parcelable {

    public static Self createFrom(ApiUser user) {
        // ...
    }
}
```

至此，API model 和 Business model 都已经定义好了，接下来我们需要把 API 的结果转化为对应的 model。

### 3.6，API model -> Business model

``` java
interface UserInfoApi {

    @GET("/users/{uid}")
    Observable<ApiUser> userInfo(@Path("uid") long uid);
}

public class UserRepo {
    // ...

    public Observable<UserInfoModel> otherUserInfo(long uid, boolean refresh) {
        Observable<UserInfoModel> local = Observable.defer(() -> {
            List<UserInfoModel> cached = mUserDbAccessor.get(uid);  // 1
            if (cached.isEmpty()) {
                return Observable.empty();
            } else {
                List<FriendInfo> friendInfoList = 
                        mFriendDbAccessor.get(uid);                 // 2
                if (friendInfoList.isEmpty()) {
                    return Observable.just(
                            NonFriend.wrap(cached.get(0)));         // 3
                }
                return Observable.just(Friend.compose(
                        cached.get(0), friendInfoList.get(0)));     // 4
            }
        });
        Observable<UserInfoModel> remote = mUserInfoApi
                .userInfo(uid)
                .map(API_USER_MAPPER)                               // 5
                .doOnNext(mUserSaver);
        Observable<UserInfoModel> combined = 
                Observable.concat(local, remote);                   // 6
        if (!refresh) {
            return combined.first();                                // 7
        }
        return combined;
    }
    
    static final Func1<ApiUser, UserInfoModel> API_USER_MAPPER = apiUser -> {
        if (apiUser.is_friend()) {                                  // 8
            return Friend.createFrom(apiUser);
        } else {
            return NonFriend.createFrom(apiUser);
        }
    };

    private final Action1<UserInfoModel> mUserSaver = user -> {
        if (user instanceof Friend) {                               // 9
            mFriendDbAccessor.put(((Friend) user).friendInfo());
        }
        mUserDbAccessor.put(user);
    };
}
```

API、DB、类型转换的逻辑也并不复杂：

1. mUserDbAccessor 负责封装数据库访问，我们先尝试从数据库读取缓存。
2. 如果缓存命中，我们就从 mFriendDbAccessor 中尝试获取好友信息。
3. 如果没有好友信息，那我们就认为这个用户是 NonFriend。这里我们有一个假设，所有好友都一定会保存好友信息。
4. 如果有好友信息，那我们就组合出 Friend 返回。
5. 调用 API 时，获取到的是 ApiUser，我们需要将其转换为 Friend/NonFriend。
6. 我们利用 `concat` 把缓存和网络连接起来。
7. 如果不需要刷新本地缓存，我们直接返回连接结果的第一个即可。
8. 利用 `is_friend`，我们可以确定 ApiUser 是否为 Friend。
9. 保存用户信息时，如果是好友，我们还需要保存 FriendInfo。

## 4，单元测试

``` java
public class UserRepoTest {
    // ...

    @Test
    public void otherUserInfoNotRefreshCacheMissNonFriend() {
        // 不刷新本地缓存、缓存缺失、对方不是好友的情形
    }

    @Test
    public void otherUserInfoNotRefreshCacheHitNonFriend() {
        // 不刷新本地缓存、缓存命中、对方不是好友的情形
    }

    @Test
    public void otherUserInfoNotRefreshCacheHitFriend() {
        // 不刷新本地缓存、缓存命中、对方是好友的情形
    }

    @Test
    public void otherUserInfoRefreshCacheMissFriend() {
        // 刷新本地缓存、缓存缺失、对方是好友的情形
    }

    @Test
    public void otherUserInfoRefreshCacheHitFriend() {
        // 刷新本地缓存、缓存命中、对方是好友的情形
    }
}
```

这边我们测例其实并没有做到覆盖所有情形，稍微偷了一下懒，但我们有信心，经过这样的测试，代码已经可靠了。万一真的出了错误，到时候再加上相应的测例，小概率事件到时再说嘛 :)

这里测试代码比较类似，只展示“刷新本地缓存、缓存命中、对方是好友的情形”：

``` java
@Test
public void otherUserInfoRefreshCacheHitFriend() {
    long uid = 1905378617;
    final ApiUser user = mGson.fromJson(MOCK_FRIEND, ApiUser.class);// 1
    final FriendInfo friendInfo = FriendInfo.builder()
            .uid(user.uid())
            .friend_remark(user.friend_remark())
            .starred(user.starred())
            .build();
    final Friend friend = Friend.compose(user, friendInfo);
    final NonFriend nonFriend = NonFriend.createFrom(user);
    when(mUserDbAccessor.get(uid))
            .thenReturn(Collections.singletonList(nonFriend));      // 2
    when(mFriendDbAccessor.get(uid))
            .thenReturn(Collections.singletonList(friendInfo));
    when(mUserInfoApi.userInfo(uid))
            .thenReturn(Observable.just(user));

    TestSubscriber<UserInfoModel> testSubscriber = new TestSubscriber<>();
    mUserRepo.otherUserInfo(uid, true).subscribe(testSubscriber);
    testSubscriber.awaitTerminalEvent();

    testSubscriber.assertNoErrors();
    testSubscriber.assertCompleted();
    testSubscriber.assertValues(friend, friend);                    // 3

    // 查询 db
    verify(mFriendDbAccessor, times(1)).get(uid);                   // 4
    verify(mUserDbAccessor, times(1)).get(uid);
    // 请求 api
    verify(mUserInfoApi, times(1)).userInfo(uid);                   // 5
    verifyNoMoreInteractions(mUserInfoApi);
    // 保存到 db
    verify(mFriendDbAccessor, times(1)).put(friendInfo);            // 6
    verifyNoMoreInteractions(mFriendDbAccessor);
    verify(mUserDbAccessor, times(1)).put(friend);
    verifyNoMoreInteractions(mUserDbAccessor);
}
```

1. 我们准备好要返回的 ApiUser、FriendInfo、Friend、NonFriend 信息。
2. 尽管让 mUserDbAccessor 返回 Friend 语法上没问题，但逻辑上是不会发生的，所以我们还是返回 NonFriend。
3. 因为我们刷新了本地缓存，而且缓存命中、API 返回数据了，所以我们最终会收到两个 Friend。
4. 我们对 mFriendDbAccessor 和 mUserDbAccessor 都进行了一次查询操作。
5. 我们对 API 进行了一次调用。
6. 我们对 mFriendDbAccessor 和 mUserDbAccessor 都进行了一次保存操作。

## 5，总结

网络层微架构的内容，本来是只打算写一篇文章的。但是后来发现内容太长，而且没有明确加上单元测试的内容，所以最终把单元测试内容完整加上，拆分为了三篇内容。希望大家能意识到测试代码的重要性：

> 既然无论手工还是测例，总归是要测试的，那我们何不稍微多花一点工夫，编写单元测试呢？

这套微架构主要包含三部分内容：

+ [认证实现方案](/2016/08/29/RESTful-Android-Network-Solution-1/){:target="_blank"}
+ [空 JSON 和 API Error 解析](/2016/09/04/RESTful-Android-Network-Solution-2/){:target="_blank"}
+ 🏁 API model 与 Business model 分离

而每一部分都包含了尽可能详尽的单元测试，目前看来是最好水平了，已经使出了洪荒之力 😄

希望大家喜欢，欢迎留言讨论！

## Bonus：拆轮子与 model 层架构推荐

前段时间拆轮子系列的前三篇，分别对 [Retrofit](https://github.com/square/retrofit){:target="_blank"}，[OkHttp](https://github.com/square/okhttp){:target="_blank"} 和 [Okio](https://github.com/square/okio){:target="_blank"} 源码进行了分析和源码导读，发布之后大家反馈还不错，其中拆 OkHttp 篇成功登上[开发者头条榜首](http://toutiao.io/top/2016-07-14){:target="_blank"}。没有看过的朋友建议大家可以看一看：

+ [拆轮子系列：拆 Retrofit](/2016/06/25/Understand-Retrofit/){:target="_blank"}
+ [拆轮子系列：拆 OkHttp](/2016/07/11/Understand-OkHttp/){:target="_blank"}
+ [拆轮子系列：拆 Okio](/2016/08/04/Understand-Okio/){:target="_blank"}

此外，之前整理的安卓 model 层架构，有幸还在 GDG 进行了一次分享，大家反响也还不错，在这里也推荐大家看一看：

+ [完美的安卓 model 层架构（上）](/2016/05/06/Perfect-Android-Model-Layer/){:target="_blank"}
+ [完美的安卓 model 层架构（下）](/2016/05/12/Perfect-Android-Model-Layer-2/){:target="_blank"}
+ [08/07 北京 GDG Android Meetup 活动回顾，讲义，照片](http://mp.weixin.qq.com/s?__biz=MzA5MDg3MjczMg==&mid=2652003543&idx=1&sn=849c06ac198cbfe9cdcfae90b2a17021&scene=1&srcid=0902QGgAZZKCpZbNNPD66mnu#rd){:target="_blank"}
