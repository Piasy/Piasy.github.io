---
layout: post
title: 手把手OkBuck教程：应用到AndroidTDDBootStrap项目
tags:
    - 安卓开发
    - RAD
    - BUCK
---

【注：本文信息已经过时，请看[续章](/AndroidTDDBootStrap-Use-OkBuck-2/index.html)】在本文中，我将一步一步手把手记录如何在[AndroidTDDBootStrap](https://github.com/Piasy/AndroidTDDBootStrap)项目中使用[OkBuck](https://github.com/Piasy/OkBuck)插件，使得AndroidTDDBootStrap支持BUCK构建，体验其畅快淋漓！

## 第一步，应用插件
按照[OkBuck中文文档](https://github.com/Piasy/OkBuck/blob/master/README-zh.md)的步骤，根据本工程的结构，在`/build.gradle`中加入11行配置，配置后`/build.gradle`是这样的：

``` groovy
...
buildscript {
    ...
    dependencies {
        ...
        classpath "com.github.piasy:okbuck-gradle-plugin:0.2.6"
    }
}
...
apply plugin: 'com.github.piasy.okbuck-gradle-plugin'
okbuck {
    target "android-23"
    overwrite true
    resPackages = [
            'common-android': 'com.github.piasy.common.android',
            'model': 'com.github.piasy.model',
            'presentation': 'com.github.piasy.template',
    ]
}
```

尝试运行`./gradlew okbuck`，哎哟，报错了：

``` bash
* What went wrong:
Execution failed for task ':okbuck'.
> Can not figure out sign config, please make sure you have only 
one sign config in your build.gradle, or set signConfigName in 
okbuck dsl.
```

## 第二步，指定签名配置，修改versionCode等的定义位置
根据报错信息，看来是OkBuck无法确定应该使用哪个签名配置，那么根据提示在`/build.gradle`中指定一个签名配置好了，修改后的`/build.gradle`内容如下：

``` groovy
...
apply plugin: 'com.github.piasy.okbuck-gradle-plugin'
okbuck {
    target "android-23"
    overwrite true
    signConfigName "develop"
    resPackages = [
            'common-android': 'com.github.piasy.common.android',
            'model': 'com.github.piasy.model',
            'presentation': 'com.github.piasy.template',
    ]
}
```

同时还注意到OkBuck文档中说明的“versionCode, versionName, targetSdkVersion, minSdkVersion的定义，需要放到AndroidManifest.xml文件中，而不是放在build.gradle文件里面”，所以改`/presentation/build.gradle`和`/presentation/src/main/AndroidManifest.xml`咯，修改后文件分别如下：

``` groovy
...
android {
  compileSdkVersion rootProject.ext.androidCompileSdkVersion
  buildToolsVersion rootProject.ext.androidBuildToolsVersion

  defaultConfig {
      // 删除了原来对targetSdkVersion, minSdkVersion的定义
      testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
      buildConfigField "boolean", "REPORT_CRASH", "true"
  }
  ...
  productFlavors.whenObjectAdded { flavor ->
      def flavorData = rootProject.ext[flavor.name]
      // 删除了原来对applicationId, versionCode, versionName的定义
  }
}
```

``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.github.piasy.template"
          android:versionCode="1"
          android:versionName="1.0.0"
        >

    <uses-sdk
            android:targetSdkVersion="23"
            android:minSdkVersion="15"
            />
    ...
```

然后再运行`./gradlew okbuck`，成功啦！

## 第三步，配置git ignore
这时候打开source tree一看，repo 有3000+改动，有点吓人啊，不过大都是buck和OkBuck搞的鬼，buck和OkBuck的相关目录（`/buck-out`, `/.buckd`, `/.okbuck`）都是程序生成的，还是不要被git管理为好，不然每次运行都会有大量改动，严重影响code review，所以在`/.gitignore`中加入以下几行：

``` bash
# buck & OkBuck
/buck-out/
/.buckd/
/.okbuck/
```

注意`/.buckconfig`以及各个module根目录下的BUCK文件我就没有ignore了，如果你想ignore也可以加到`/.gitignore`中。另外就是AndroidTDDBootStrap项目的签名文件本来就是公开的，所以也就没有必要ignore了，不过由于OkBuck默认是把生成的签名文件放到`/.okbuck`目录下，所以其实已经ignore了。

此时尝试运行`buck install presentation`，哎哟！果然没这么简单，依然报错：

``` java
/Users/piasy/src/AndroidTDDBootStrap/buck-out/annotation/
common-android/__src_gen__/com/github/piasy/common/
android/utils/AndroidUtilsModule_ProvideRxErrorPro
cessorFactory.java:9: error: 找不到符号
@Generated("dagger.internal.codegen.ComponentProcessor")
 ^
  符号: 类 Generated
/Users/piasy/src/AndroidTDDBootStrap/buck-out/annotation/
common-android/__src_gen__/com/github/piasy/common/
android/utils/AndroidUtilsModule_ProvideMiUIUtilFa
ctory.java:5: error: 程序包javax.annotation不存在
import javax.annotation.Generated;
                       ^
```

## 第四步，代码不兼容的修改
啊，原来是踩到OkBuck的已知坑了，`javajavax.annotation`的依赖需要声明为`compile`而不是`provided`，一看`/common-android/build.gradle`中的定义，还真是`provided`，改！

同时注意到OkBuck的已知坑里面还有RetroLambda, ButterKnife的不兼容问题（这都是BUCK的问题呀！），统统给改了！这个过程略有点漫长，代码就不演示了，但是有个建议，改一部分试一下，利用buck报错的信息快速定位需要改动的代码。

另外在此过程中还遇到了multi-product flavor不支持的问题（也是BUCK的问题呀！），所以放弃`developCompile`和`productCompile`的使用咯！修改后的`/presentation/build.gradle`dependencies部分如下：

``` groovy
    /*developCompile presentationDependencies.xLogAndroid
    productCompile presentationDependencies.xLogAndroidIdle*/
    compile presentationDependencies.xLogAndroid
    /*developCompile presentationDependencies.leakCanary
    productCompile presentationDependencies.leakCanaryIdle*/
    compile presentationDependencies.leakCanary
```

注意这里相当于都是使用了`developCompile`，真正产品开发时，如果要发布release版本，一定要使用相应的idle依赖，不然APP启动会先白屏5s，还时不时来一个blurrrr。

另外就是修改的过程中，如果改了build.gradle，或者添加了新的依赖（远程或者本地），都需要重新执行`./gradlew okbuck`以生成新的buck配置。

期间还遇到了一个buck对于build config支持的坑，buck原生只会暴露`DEBUG`这一个build config，而AndroidTDDBootStrap中使用了`BUILD_TYPE`这个变量来决定是否install leak canary，怎么办呢？使用buck暴露出来的`DEBUG`变量吗？但是打包发布的时候肯定不用buck呀，还是改个名字变成自定义的build config吧，但是就不能使用AS切换build variant来切换配置了，每次都需要改build.gradle，而且还需要重新执行`./gradlew okbuck`，这点有点不爽。

代码改完之后运行`buck install presentation`，结果报了一个奇怪的错误：

``` bash
/Users/piasy/src/AndroidTDDBootStrap/presentation/src/
main/java/com/github/piasy/template/features/splash/
GithubSearchFragment.java:-1: error: 不兼容的类型: 
java.util.List<java.lang.Object>无法转换为
java.util.List<com.github.piasy.model.entities.GithubUser>
```

报错行数居然是-1，不过根据强转为`List<GithubUser>`失败，大致定位到是下面这一行导致的：

``` java
showSearchUserResult(Collections.emptyList());
```

推测是buck编译时类型推到失败，改成下面这样后再试一次，果然可以了：

``` java
showSearchUserResult(Collections.<GithubUser>emptyList());
```

本以为是buck的bug，结果是因为在build.gradle中设置了`sourceCompatibility JavaVersion.VERSION_1_8`，所以AS能够进行类型推导成功，确实java 8在类型推导上是下了大工夫了的呀，所以别偷懒，既然不能使用RetroLambda了，那还是把jdk改为1.7吧，还省得AS成天提示可以替换为lambda表达式。

最后`buck install presentation`，成功咯！

## 闪退？？？
成功打包安装后，运行app，过了几秒之后闪退了！这可是之前没有的，但是目前buck打包的apk不支持调试，所以看不到log，先粗暴解决，在`/presentation/src/main/AndroidManifest.xml`中加入`android:debuggable="true"`，看看log再说。

``` java
This app relies on Crashlytics. Please sign up for access at https://fabric.io/sign_up,
install an Android build tool and ask a team member to invite you to this app's organization.
```

log看起来是fabric找不到key，所以注册失败了，看来新发现了一个OkBuck的坑呀，提issue去咯！至于这个问题怎么解决？先禁用呀！打发布包的时候，用gradle再恢复，等OkBuck解决这个问题后再彻底恢复。

## 小结
本来切换到BUCK至少需要一天的工作量的，可是使用OkBuck后，只需要1个小时就搞定了，这还包括写这篇博客的时间哟！OkBuck还是挺赞的 :)
