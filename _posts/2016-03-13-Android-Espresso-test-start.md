---
layout: post
title: （可能是）目前最全面的Android Espresso配置指南了
tags:
    - 安卓开发
    - TDD
---

安卓开发过程中测试的编写是一个公认的痛点，本文总结了我在[AndroidTDDBootStrap工程](https://github.com/Piasy/AndroidTDDBootStrap)中配置Espresso测试所遇到的坑，例如神秘报错`android.content.res.Resources$NotFoundException`和`java.util.zip.ZipException: duplicate entry`，以及对dagger，mock网络请求的实践，目测应该是目前最全面的指南了 :)  本文涉及的完整代码可以在[Github: AndroidTDDBootStrap](https://github.com/Piasy/AndroidTDDBootStrap/tree/1958be13bda22d62ccae1a44830d7dd5a07be98f)获取。

## 配置Gradle依赖

在`app/build.gradle`中加入以下配置：

``` groovy
androidTestCompile project(':testbase')
androidTestCompile (appTestDependencies.androidJUnitRunner) {
    exclude module: 'support-annotations'
}
androidTestCompile (appTestDependencies.mockito) {
    exclude module: 'hamcrest-core'
}
androidTestCompile appTestDependencies.dexmaker
androidTestCompile (appTestDependencies.dexmakerMockito) {
    exclude module: 'hamcrest-core'
    exclude module: 'mockito-core'
}
androidTestCompile (appTestDependencies.androidJUnit4Rules) {
    exclude module: 'support-annotations'
    exclude module: 'hamcrest-core'
}
androidTestApt (appDependencies.daggerCompiler) {
    exclude module: 'dagger'
}
```

`androidTestCompile`就是用来声明安卓instrumentation测试的依赖的，它对应的gradle测试命令是`./gradlew :app:connectedAndroidTest`或者`./gradlew :app:cAT`。而`androidTestApt`则是在instrumentation测试代码中使用apt插件生成代码的（dagger2用到）。

在这里我还遇到了两个特别诡异的错误，第一个是运行测例直接失败，logcat报错如下：

``` java
android.content.res.Resources$NotFoundException: Resource ID
```

Google几番也没有找到什么头绪，最后从[StackOverflow上一个回答的评论](http://stackoverflow.com/questions/34791907/inflating-activity-content-fails-in-instrumentation)中找到了原因：归根结底还是因为发生了重复依赖！通过`./gradlew :app:dependencies`分析`app`的依赖，发现`testbase`这个module依赖了`appcompat-v7`，而`app`则直接编译依赖了它，两者在instrumentation测试中发生了冲突，导致了这个诡异的闪退，移除`testbase`对`appcompat-v7`的依赖之后就解决了这个问题。

第二个是执行`./gradlew :model:cAT`失败，gradle报错如下：

``` java
java.util.zip.ZipException: duplicate entry:      javax/annotation/Generated.class
```

同样是几番Google无果，反复折腾了几天，无奈只能自行分析。从报错来看还是发生了依赖重复，存在两个`javax/annotation/Generated.class`类，通过`./gradlew :model:dependencies`分析`model`的依赖，最终发现`testbase`这个module通过`espresso`间接依赖了`javax.annotation-api`，而`model`则通过`base`编译依赖了`javax.annotation`，两者发生了重复，导致了这个问题。通过配置`testbase`，在依赖`espresso`的时候`exclude module: 'javax.annotation-api'`终于解决了这个问题。

## 依赖注入
如何为测试代码配置依赖注入这个问题已经纠缠我两年了。

最初听从了某大神的博客建议，debug和release两个variant用作不同用途，debug一套DI的代码，专门用于测试，release一套DI的代码，专门用于非测试（博客出处已经不可考了）。这种方式最大的问题就是需要维护两套代码，而且他们无法同时被IDE展示（必须通过build variant切换），在重构的时候无法同时重构，经常导致重构了业务代码，结果测试代码没有被重构，必须手动修改，非常蛋疼。

后来参考了另一位大神[Chiu-Ki的博客](http://blog.sqisland.com/2015/04/dagger-2-espresso-2-mockito.html)，采取了Application类暴露接口设置dagger component的方式，设置进去的component负责提供mock的对象，并且它可以把依赖注入到测试代码中。这种方式比较简洁，但是Application类却暴露了不应该暴露的接口，不是十分优雅。

最终[Chiu-Ki再次发力](http://blog.sqisland.com/2015/12/mock-application-in-espresso.html)，通过编写一个MockApplication类，并通过自定义Test Runner来启动mock application，完美解决了这一问题。mock application类继承自application类，在其中初始化component为提供mock的component，而mock component又把依赖注入到测试代码中，完美 :)

但是dagger在涉及到继承的时候有一个细节需要注意：如果component定义的`inject`接口接受的是父类型，那么当子类型实例调用`inject(this)`时，子类型中需要注入的依赖（`@Inject`注解的成员）将无法注入！需要编写一个component的子类，把`inject`接口的参数类型声明为子类型，并且声称component子类的实例进行依赖注入。这种情况下父类中需要注入的依赖是可以成功注入的，因为dagger可以搜索父类并把依赖注入到父类中，但反过来是行不通的，dagger是无法搜索子类并注入依赖到子类中的（因为编译期间可能根本就无法获取到子类的符号呀，怎么能搜索子类呢？）。

## Mock网络请求
可以说目前最流行的网络请求库就是[OkHttp](https://github.com/square/okhttp/)了，而且从安卓6.0开始它就是系统的默认实现了。而OkHttp还提供了另一个无比强大的工具：[MockWebServer](https://github.com/square/okhttp/blob/master/mockwebserver)。而OkHttp + [Retrofit](https://github.com/square/retrofit)应该是REST API请求的标配了，下面我就总结一下如何利用MockWebServer来进行网络请求mock，对APP进行集成测试。

Retrofit 2.0中没有了end point的概念，取而代之的是base url，在创建Retrofit对象的时候可以配置，在测试中我们配置base url为`http://localhost:9876/`，同时在测试用例中我们配置MockWebServer运行在`9876`端口。这样任何通过Retrofit发起的API调用都是请求的MockWebServer了。

MockWebServer可以设置`Dispatcher`，可以根据不同的请求返回不同的数据，在本工程的一个测例中，dispatcher的设置如下：

``` java
new Dispatcher() {
    @Override
    public MockResponse dispatch(final RecordedRequest request)
            throws InterruptedException {
        final String path = request.getPath();
        if (path.startsWith("/search/users?")) {
            return new MockResponse().setBody(
                    MockProvider.provideSimplifiedGithubUserSearchResultStr());
        } else {
            return new MockResponse();
        }
    }
};
```

如此通过Retrofit调用`/search/users`接口返回的就是mock的数据了。完美 :)
