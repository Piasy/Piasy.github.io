---
layout: post
title: 手把手OkBuck教程：应用到AndroidTDDBootStrap项目（续）
tags:
    - 安卓开发
    - RAD
    - BUCK
---

5个多月过去了，[OkBuck](https://github.com/Piasy/OkBuck)和[AndroidTDDBootStrap](https://github.com/Piasy/AndroidTDDBootStrap)项目都发生了重大更新，[原文]({{ site.baseurl }}/AndroidTDDBootStrap-Use-OkBuck/)中的描述已经过时已久，今天趁着AndroidTDDBootStrap完成重构，更新AndroidTDDBootStrap的OkBuck配置过程，继续体验其畅快淋漓！

## 第一步，应用插件
按照[OkBuck文档](http://blog.piasy.com/OkBuck/)的步骤，根据本工程的结构，在`/build.gradle`中加入配置，配置后`/build.gradle`是这样的：

``` groovy
...
buildscript {
    ...
    dependencies {
        ...
        classpath "com.github.piasy:okbuck-gradle-plugin:1.0.0-beta8"
    }
}
...
apply plugin: 'com.github.piasy.okbuck-gradle-plugin'
okbuck {
    target "android-23"
    overwrite true
    resPackages = [
            'base'    : 'com.github.piasy.base',
            'model'   : 'com.github.piasy.model',
            'app'     : 'com.github.piasy.app',
            'testbase': 'com.github.piasy.base.test',
    ]
}
```

尝试运行`./gradlew okbuck`，哎哟，报错了：

``` bash
* What went wrong:
Execution failed for task ':okbuck'.
> AndroidBinaryRule primaryDexPatterns must be non-null.
```

## 第二步，配置multidex
根据报错信息，看来是因为我使用了multidex，这块高级设置需要额外配置，而根据[OkBuck wiki描述](https://github.com/Piasy/OkBuck/wiki/Multidex-Configuration-Guide)的描述，linearAllocHardLimit可以直接设置为一个很大的值（或者从小值开始试，BUCK build命令失败提示多少就设置为多少），而primaryDexPatterns通常从App类开始，逐渐尝试，运行之后提示找不到类再加到列表中，初始设置如下：

``` groovy
...
apply plugin: 'com.github.piasy.okbuck-gradle-plugin'
okbuck {
    target "android-23"
    overwrite true
    resPackages = [
            'base'    : 'com.github.piasy.base',
            'model'   : 'com.github.piasy.model',
            'app'     : 'com.github.piasy.app',
            'testbase': 'com.github.piasy.base.test',
    ]
    linearAllocHardLimit = [
            app: 7194304
    ]
    primaryDexPatterns = [
            app: [
                    '^com/github/piasy/app/BootstrapApp^',
            ]
    ]
}
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

此时尝试运行`buck install -r appProductRelease`，直接成功！

## fabric问题
成功打包安装后，运行app，发现app卡在了Splashing页，查看log：

``` java
This app relies on Crashlytics. Please sign up for access at https://fabric.io/sign_up,
install an Android build tool and ask a team member to invite you to this app's organization.
```

log看起来是fabric找不到key，所以注册失败了，看来新发现了一个OkBuck的坑呀，提issue去咯！至于这个问题怎么解决？先禁用呀！打发布包的时候，用gradle再恢复，等OkBuck解决这个问题后再彻底恢复。

## 小结
由于代码不兼容的地方早前就进行了修改，这次换用新版本的OkBuck之后，集成时间只花了半小时，这还包括写这篇博客的时间哟！OkBuck还是挺赞的 :)