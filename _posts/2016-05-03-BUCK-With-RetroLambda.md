---
layout: post
title: BUCK 与 RetroLambda 兼容性解决方案
tags:
    - 安卓开发
    - BUCK
---

从最初 OkBuck 发布时宣称 BUCK 与 RetroLambda 不兼容只能忍痛割爱（lambda），到 BUCK 维护者之一联系我声称 BUCK 可以编译 Java 8 结果遇到编译错误未解，到昨晚终于成功让 BUCK 与 RetroLambda 出双入对，时隔大半年终于臻至完美，怎一个爽字了得！如果你还不了解什么是 BUCK，可以参考我的两篇文章 [OkBuck, underneath the hood](/2016/02/01/OkBuck-Underneath-the-hood/index.html){:target="_blank"}，[手把手OkBuck教程：应用到AndroidTDDBootStrap项目（续）](/2016/03/10/AndroidTDDBootStrap-Use-OkBuck-2/index.html){:target="_blank"}，以及 [BUCK 官方文档](https://buckbuild.com){:target="_blank"}。

## BUCK 编译 Java 8
这一点 BUCK 确实已经支持了，只需要在 `java_library` 和 `android_library` 这两种 rule 中加入以下配置即可：

``` python
	source = '8',
	target = '8',
```

然而就是这一步中出现的拦路虎，挡住了我们前进的脚步半年之久，简而言之，纯 Java library module 这样做没有任何问题，但是 Android library module 的编译却报告了错误：

``` java
com.sun.tools.javac.code.Symbol$CompletionFailure: class file for 
java.lang.invoke.MethodType not found.
```

错误日志很明显对不对？然而，搜索出来的结果绝大部分都是指向了 [gradle-retrolambda 的一个 issue](https://github.com/evant/gradle-retrolambda/issues/126){:target="_blank"}，而这个 issue 的解决方案外部看来和这个问题没有任何联系。无奈之下只能再次向 BUCK 维护者求助：

<img src="/img/201605/Shawn_Wilsher.jpeg" alt="Shawn Wilsher">

但这个帅小伙却从此失去了音信 :(
    
好了帅小伙的故事先暂时到这里，我们继续。

对于 Java library module，BUCK 能够成功使用 javac 编译出 java 8 的字节码，但是我们怎么把 RetroLambda 集成到 BUCK 的构建过程中呢？还是这个帅小伙出的主意（其实是 BUCK 的有些文档严重缺乏，只能自己看源码或者他们出主意）：`postprocess_classes_commands`。

## BUCK 调用 RetroLambda
RetroLambda 只是一个命令行工具，大家通常使用的可能是另一个 gradle 插件：[gradle-retrolambda](https://github.com/evant/gradle-retrolambda){:target="_blank"}，利用上面提到的 `postprocess_classes_commands` 参数，我们可以在 `java_library` 和 `android_library` 这两种 rule 中加一个 class 编译完成之后的 hook，BUCK 会执行 `postprocess_classes_commands` 参数的命令，并把本次编译的 class 路径作为参数传入。所以我们就可以在这里执行 RetroLambda 程序把 java 8 的字节码编译为 java 6 的字节码了。这里因为需要为 shell 脚本传入参数，所以我们需要把命令封装到一个脚本文件中，脚本文件的内容如下：

``` bash
java \
-Dretrolambda.inputDir=$1 \
-Dretrolambda.classpath=$1 \
-jar ./retrolambda-2.3.0.jar
```

这里 RetroLambda 会直接覆盖 BUCK 编译生成的 class 文件，命令执行完毕之后，BUCK 会继续打包的后续步骤。

这个版本的脚本文件编译 Java library module 时成功了，但是编译 Android library module 时却失败了，报了上节提到的错误。

也就是这个错误，困扰了我们半年之久。当初的困境感兴趣的朋友可以查看这个 [Github issue](https://github.com/Piasy/OkBuck/issues/32){:target="_blank"}。

## class file for java.lang.invoke.MethodType not found 问题的解决
这个问题之前 RetroLambda 也遇见过，虽然他们的解决方法看上去和我们遇见的问题没有关系，但它毕竟解决了，所以深挖肯定有门路。[我在之前的文章中](/2016/03/16/Looper-crash/index.html){:target="_blank"}曾总结过各种问题的解决思路，这次的问题还是通过“差异分析法”解决的。

RetroLambda 可以把同样的代码先编译为 java 8 的字节码，BUCK 的却不可以，但他们用的至少都是相同的 javac 程序吧？那问题肯定就出现了**编译选项**上。通过给两种方式加上日志输出选项，我拿到了它们各自的编译选项，这里，我以我发布到 [Github 的 demo 工程](https://github.com/Piasy/BuckJava8RetroLambdaDemo){:target="_blank"}为例。

执行 `buck build -v 10 app/:src_release`，在控制台看到了一段红色的信息，就是出错的命令，错误就是上面提到的 `class file for java.lang.invoke.MethodType not found`，编译选项如下：

```bash
javac \
-source 8 -target 8 \
-sourcepath  -g \
-bootclasspath \
/Users/piasy/tools/android-sdk/platforms/android-23/android.jar:\
/Users/piasy/tools/android-sdk/platforms/android-23/optional/org.apache.http.legacy.jar \
-verbose \
-d /Users/piasy/src/BuckJava8RetroLambdaDemo/buck-out/bin/app/lib__src_release__classes \
-classpath /Users/piasy/src/BuckJava8RetroLambdaDemo/buck-out/bin/app/__src_release#dummy_r_dot_java_rdotjava_bin__:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/aar__animated-vector-drawable-23.3.0.aar#aar_prebuilt_jar.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/aar__appcompat-v7-23.3.0.aar#aar_prebuilt_jar.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/aar__support-v4-23.3.0.aar#aar_prebuilt_jar.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/aar__support-vector-drawable-23.3.0.aar#aar_prebuilt_jar.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/jar__support-annotations-23.3.0.jar.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/buck-out/gen/app/lib__build_config_release__output/build_config_release.jar \
@buck-out/gen/app/__src_release__srcs
```

执行 `./gradlew :app:compileDebugJavaWithJavac --debug`，查看输出找到编译选项：

``` bash
javac \
-d /Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/retrolambda/debug \
-g -encoding UTF-8 \
-bootclasspath /Users/piasy/tools/android-sdk/platforms/android-23/android.jar \
-sourcepath /Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/tmp/compileDebugJavaWithJavac/emptySourcePathRef \
-classpath /Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/intermediates/exploded-aar/com.android.support/support-v4/23.3.0/jars/classes.jar:\
/Users/piasy/tools/android-sdk/extras/android/m2repository/com/android/support/support-annotations/23.3.0/support-annotations-23.3.0.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/intermediates/exploded-aar/com.android.support/support-vector-drawable/23.3.0/jars/classes.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/intermediates/exploded-aar/com.android.support/appcompat-v7/23.3.0/jars/classes.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/intermediates/exploded-aar/com.android.support/animated-vector-drawable/23.3.0/jars/classes.jar:\
/Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/intermediates/exploded-aar/com.android.support/support-v4/23.3.0/jars/libs/internal_impl-23.3.0.jar:\
/Library/Java/JavaVirtualMachines/jdk1.8.0_92.jdk/Contents/Home/jre/lib/rt.jar \
/Users/piasy/src/BuckJava8RetroLambdaDemo/app/src/main/java/com/github/piasy/buck/retrolambda/demo/MainActivity.java \
/Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/generated/source/r/debug/android/support/v7/appcompat/R.java \
/Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/generated/source/r/debug/com/github/piasy/buck/retrolambda/demo/R.java \
/Users/piasy/src/BuckJava8RetroLambdaDemo/app/build/generated/source/buildConfig/debug/com/github/piasy/buck/retrolambda/demo/BuildConfig.java \
-XDuseUnsharedTable=true
```

差异最大的是两部分，一个是 `classpath` 选项，一个是最后一部分，BUCK 是一个 `@` 加一个文件路径，gradle 则是多个文件名，而查看 BUCK 命令中的那个文件内容，_基本上_ 就是 gradle 命令中传入的那几个文件名。那么很可能就是 `classpath` 部分了（当然，真实情况是我逐一替换了所有不同的部分，不幸的是最后才轮到 `classpath`）。

有趣的是，gradle 和 BUCK 都会先把 aar 文件解压开来，然后把其中的 jar 包加入到 classpath 中，而 gradle 的 classpath 相较于 BUCK 的多了一个非常可疑的对象：`jre/lib/rt.jar`，gotcha！不用试了，一看就是它了。

当然我还是 google 了一下它的渊源，详见[这篇文章](http://javarevisited.blogspot.jp/2015/01/what-is-rtjar-in-javajdkjre-why-its-important.html){:target="_blank"}，惭愧的是我正如这篇文章开头所描述的那帮程序员一样，连 rt.jar 为何方神圣都不知道，罪过罪过。解压开来之后确实发现 `java.lang.invoke.MethodType` 这个 class 端坐其中。

通过修改 BUCK 的源码，把 rt.jar 加入到 javac 的默认 classpath 中，进一步的测试也验证了问题的症结所在，就是 classpath 少了 rt.jar！

## 完整的示例
示例工程代码可以在 [Github 获取](https://github.com/Piasy/BuckJava8RetroLambdaDemo){:target="_blank"}。

上上节中的 RetroLambda 脚本对 Java library module 的 BUCK 与 RetroLambda 联编可以通过，但对 Android library module 却不行，因为还有许多 jar 包需要加入到 classpath 中，包括 android.jar。而这一点很好解决，把我们在上节中拿到的 BUCK 编译的 javac classpath 加入到 RetroLambda 执行的 classpath 中即可，完整的 RetroLambda 脚本如下：

``` bash
java \
-Dretrolambda.inputDir=$1 \
-Dretrolambda.classpath=$1:\
/Users/piasy/tools/android-sdk/platforms/android-23/android.jar:\
\
/Users/piasy/tools/android-sdk/platforms/android-23/optional/org.apache.http.legacy.jar:\
./buck-out/bin/app/__src_release#dummy_r_dot_java_rdotjava_bin__:\
./buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/aar__animated-vector-drawable-23.3.0.aar#aar_prebuilt_jar.jar:\
./buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/aar__appcompat-v7-23.3.0.aar#aar_prebuilt_jar.jar:\
./buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/aar__support-v4-23.3.0.aar#aar_prebuilt_jar.jar:\
./buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/aar__support-vector-drawable-23.3.0.aar#aar_prebuilt_jar.jar:\
./buck-out/gen/.okbuck/AFFB34D18189F4D10144A341628B7C81/jar__support-annotations-23.3.0.jar.jar:\
./buck-out/gen/app/lib__build_config_release__output/build_config_release.jar \
-jar ./retrolambda-2.3.0.jar
```

运行修改过后的 BUCK 编译，成功！

## 后续工作
这次对 BUCK 的修改量很少，只有一行代码，但却解决了困扰我们长达半年的一个问题，这段心酸岁月终于过去了。修改内容已经提交 [pr](https://github.com/facebook/buck/pull/732){:target="_blank"} 到 BUCK repo，希望能够尽快合入主干，而这个 RetroLambda 的脚本也是有可能自动生成的，所以也加入到了 OkBuck 的开发日程中，~~近日就将完成~~ 已经在 1.0.0-beta9 发布。届时欢迎大家使用，尽情体验 lambda + BUCK 的畅快淋漓！
