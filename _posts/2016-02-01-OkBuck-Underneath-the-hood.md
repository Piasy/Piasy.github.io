---
layout: post
title: OkBuck, underneath the hood
tags:
    - 安卓开发
    - 原理剖析
    - BUCK
---

本文对我目前在github上收获star最多的开源项目[OkBuck](https://github.com/Piasy/OkBuck)的工作原理进行了深度解析，并在本文写作过程中完成了对OkBuck的第三轮重构，作为OkBuck 1.0版本发布的基础。

## 渊源
15年9月份开始了解到快速打包相关的技术，此时已经饱受gradle打包龟速的痛苦。

首先了解到的是[LayoutCast](https://github.com/mmin18/LayoutCast)，但由于其只支持Android 5.0 以上（ART）的手机，虽然5.0的测试机肯定有，但还有大多数测试机不是5.0，所以还是有很多时候会比较慢，所以没有采用。这时候BUCK进入了视野。

国庆期间尝试了一下BUCK，深深觉得下载依赖的jar/aar文件，编写BUCK脚本特别麻烦，尤其是一旦加了新的依赖，还需要维护BUCK脚本以及依赖文件，是一件持续的麻烦事。恰好当时想要趁着国庆期间做点东西，而BUCK配置的编写与维护也有可能自动化，所以萌生了OkBuck的想法，OkBuck的命名受到了Okio和OkHttp的启发。

OkBuck的目标，是通过读取工程的gradle配置，自动生成BUCK脚本，免去开发者下载依赖的jar/aar文件，编写、维护BUCK脚本，处理依赖之间的冲突等繁琐又容易出错的工作。

## 面临的问题
+  获取每个module的依赖，包括从maven等服务器获取的依赖、本地libs目录下的jar/aar依赖、工程内的其他module；
+  获取apt依赖，并解析出annotation processor；
+  避免依赖冲突：同一个库，在同一个BUCK的rule中，不能出现多个不同版本；
+  获取工程的各种配置：build config，sign config，是否开启multidex，是否debuggable等；
+  BUCK rule的生成，BUCK文件的生成；
+  multi product flavor支持；
+  exopackage支持；

由于经验有限，很多问题都是通过摸索的方式解决的，没有查阅gradle的API文档，代码也比较原始。在0.3.0版本期间，进行了第二次代码的重构，一定程度上优化了代码，但是解决问题的方式，依然不太优雅，而且对于gradle版本，以及Android gradle插件的版本，兼容性也存在问题。

农历新年之前，正好公司项目进度没那么吃紧了（浮云 :joy:），趁此机会好好总结一下OkBuck的工作原理，分析解决问题的方式、架构、代码的不优雅之处，准备进行第三轮重构，另外完善部分功能，整理文档之后，发布1.0版本。

## 获取sub module的依赖
gradle api内部定义了`Dependency`系统，提供了接口获取，但它并不完整，主要是缺乏libs目录下的jar/aar依赖。

``` groovy
for (ResolvedDependency dependency : 
    project.configurations.getByName("compile").resolvedConfiguration
    .firstLevelModuleDependencies) {
    // dependency是gradle api定义的依赖，可以获取moduleGroup，moduleName，moduleVersion信息，
    // 包括maven依赖、本地子module依赖，间接依赖在dependency.children中，
    // 而这个依赖的本地文件则在dependency.moduleArtifacts中
}
```

下面的方式可以获取到最终所有的依赖的本地文件：

``` groovy
for (File dependency : project.configurations.getByName("compile").resolve()) {
    // compile 是依赖选项（configuration）， dependency 就是各类依赖解
    // 析完毕之后的本地文件，包括直接与间接
}
```

考虑到后续buck编译以及依赖冲突检测的需求，OkBuck依赖获取的方式同时使用了上面这两种方式。

依赖的分类参照gradle api，按照configuration分类，每个sub project都会有多个configuration，最终的flavorVariant组合的依赖，将合并该组合下所有configuration的依赖，这个合并的过程需要去除完全一样的依赖记录（本地依赖文件相同），但对冲突的依赖，应该进行检测与警告。

而对于本地的android library module，如果它有多个flavor，对于其依赖者的某种flavorVariant组合中，只可能依赖它的一种flavorVariant组合，所以如果一个library module有多flavor，那么它的不同flavorVariant组合将是不同的依赖。

最终获取到的所有依赖，使用自定义的`Dependency`类型进行封装，它需要提供的接口包括：

+  类型，用于区分后续buck的处理；
+  本地jar/aar文件，非sub module类，需要拷贝文件；
+  src/flavor/res ... rule对应的名字，用于声明buck依赖；
+  上述rule的存在性判断；

sub module依赖的分析工作分拆如下：

+  依赖获取分离为`DependencyExtractor`，专门负责获取各种configuration的依赖，以及annotation processor；
  +  输出的依赖就是每个sub module的每个flavor/variant下的依赖集合；
+  类型判断、重复检测逻辑，分离到`Dependency`抽象类中；
+  `DependencyAnalyzer`只负责分析工作，包括把所有间接依赖都集中起来，冲突检测，设置本地文件保存路径等；
  +  输出的依赖就是每个sub module在每种flavor + variant组合下的依赖集合，进行了去重与冲突检测，分配好了本地保存路径，这份输出将直接用于后续每个sub module的buck rule的依赖；
  +  组合flavor/variant；
  +  重复和冲突检测；
  +  分配本地保存路径；
+  `DependencyProcessor`进行依赖的处理工作，把依赖文件拷贝到okbuck目录，并为它们生成一个BUCK配置文件，以及解析Android application module的签名配置；


## annotation processor相关
annotation processor依赖和module的普通依赖类似，只不过configuration是`apt`和`provided`，这两种是目前通用的惯例，Android module涉及到注解处理的，基本都是用的apt插件，而Java module，常用的做法也是声明一个`provided` configuration，并添加到classpath中，就像这样：

``` groovy
sourceSets {
    main {
        compileClasspath += configurations.provided
    }
}
```

而annotation processor类的提取，则可以提取对应jar文件中的`META-INF/services/javax.annotation.processing.Processor`文件部分。需要注意的是，一个jar包里面可能会有多个annotation processor，需要全部解析出来。

## 避免依赖冲突
依赖冲突是引入BUCK过程中最痛苦的一件事，执行BUCK打包命令时，经常报形如`Multiple dex files define***`的错误，就是因为发生了依赖冲突。

OkBuck的做法是，任何rule都不要使用`exported_deps`参数，然后把整个工程所有的依赖集中进行分析，每个module rule的`deps`部分，就是该module的所有依赖，包括直接和间接。此外，除了annotation processor的依赖，其他的module rule的依赖，也没有使用BUCK官方样例中的`all_jars`方式，避免同一依赖文件同时存在多份造成依赖冲突。

同时OkBuck也采取了不同版本依赖冲突检测的机制，可以通过配置控制是否在这种情况下失败并提示。

+  遍历所有sub module(project)，利用第一种方法，获取到所有的maven依赖以及sub module，用于检测依赖冲突，对gradle的合理使用情况下，这些依赖将是整个工程依赖的大部分；
+  同时利用第二种方式，获取到所有的依赖本地文件，两者求差集，即可得到本地libs目录下的依赖，这部分依赖就只能使用文件名来进行重复检测了；
+  此外，也需要检测maven依赖和这部分依赖的重复，这一步检测也只能使用文件名进行；
+  以example工程为例，如果`javalibrary`在libs文件夹中引入了gson-2.4.jar，但是又在`app`中通过声明maven依赖引入了gson 2.3，此时就会提示：`in app, gson-2.4.jar is duplicated with gson-2.3.jar`；当然这个提示其实并不能直接找到根本原因，后续会改进；
+  如果提示依赖冲突，可以先通过`./gradlew dependencies`命令来查看整个工程的依赖列表，进行排查；

在此次重构过程中，避免依赖冲突的时候，遇到了一个问题：最初我是直接把每个module的每个flavor + variant组合下的依赖单独放在一个目录中，这就造成了同一个依赖文件会存在多份，导致了依赖冲突。因此需要把公共依赖集中起来，保证同一个依赖只会在okbuck目录下存在一份（apt依赖可以不考虑）。

解决办法很简单，只需要为每个dep分配好合适的dstDir即可，将依赖其的module名字按字典序排列拼接，然后计算hash之后，作为目标目录（之所以要hash，是为了避免出现目录名太长的问题）。这一个改变基本是在重构完成之后进行的，但是代码上的改动非常小，只需改变为每个依赖设置dstDir的策略即可，这也从侧面上反映出来此次重构的成功性。

## 获取工程的各种配置
以`minSdkVersion`为例：

``` groovy
project.extensions.getByName("android").metaPropertyValues.each { prop ->
    if ("defaultConfig".equals(prop.name)) {
        ProductFlavor defaultConfigs = (ProductFlavor) prop.value
        if (defaultConfigs.minSdkVersion != null) {
            minSdkVersion = defaultConfigs.minSdkVersion.apiLevel
        }
    }
    if ("productFlavors".equals(prop.name)) {
        if (!"default".equals(flavor)) {
            for (ProductFlavor productFlavor :
                    ((NamedDomainObjectContainer<ProductFlavor>) prop.value).asList()) {
                if (productFlavor.name.equals(flavor)) {
                    if (productFlavor.minSdkVersion != null) {
                        minSdkVersion = productFlavor.minSdkVersion.apiLevel
                    }
                }
            }
        }
    }
}
```

## multi product flavor支持
BUCK原生不支持multi product flavor，OkBuck通过解析每个flavor的配置，同时为每种flavor + variant组合生成一套BUCK配置，达到支持multi product flavor的效果。

对于flavor较多的情况，生成的BUCK文件会比较复杂，解析时间会达到1s以上，所以OkBuck也加上了一个flavor filter的选项，可以只生成指定flavor的BUCK文件。之所以不通过gradle task运行参数控制，是由于可能存在OkBuck无法满足的需求，仍需手动稍微修改生成的BUCK文件，这种情况下，每次切换flavor都需要重新编辑，会很麻烦。

## BUCK rule、BUCK文件的生成
OkBuck为buck的每种rule对应建立了一个类，`AndroidBinaryRule`, `AndroidLibraryRule`等，它们继承自公共基类`AbstractBuckRule`，提供`print`接口，把rule的内容打印到`PrintStream`中，并且按照它们的组成（参数），分为几大类型，将打印工作分摊开来，简化了每个类的大小，提高可读性与可扩展性。

上述依赖分析，以及工程配置的问题解决之后，就可以为每个sub module生成各类buck rule，并最终组成buck脚本了。步骤如下：

+  java library module
  +  目前gradle并未支持flavor与variant，也不支持build config，所以只需要考虑main目录下的代码，compile选项的依赖，以及注解处理
  +  只能依赖jar包，不能依赖aar包
  +  只需要生成java_library和project_config两个rule
+  android library module
  +  需要支持flavor和variant，包括：java, res, assets, build config, jniLibs, aidl
    +  java代码，每种flavor + variant组合将对应一个android_library rule，它包括了该种组合下所有源码目录内的代码
    +  res和assets，同属于android_resource rule，每个源码目录下均可能存在，每个源码目录将生成一个rule，而每种flavor + variant的组合，将需要使用该组合下所有源码目录对应的android_resource rule
    +  build config, aidl和java代码类似，每种flavor + variant组合对应一个相应的rule
    +  jniLib和res类似，每个源码目录需要一个rule
    +  上述差异主要是由buck rule参数的类型造成的
  +  由于manifest比较特殊，暂时不支持manifest的多flavor，只允许在main目录下存在一个manifest，它将作为android_library rule的参数被设置在每个android_library rule中
+  android application module
  +  需要支持flavor和variant，范围和android library module相同
  +  不同之处在于android_library rule不包括manifest，manifest将有单独的rule，供android_binary使用
  +  此外还需要为每种flavor + variant组合生成一个android_binary rule
    +  manifest比较特殊，先使用一个`genrule`，以main源码目录下的manifest文件生成一个skeleton，再合并依赖的manifest，最终为每种flavor + variant组合生成一个android_manifest rule
    +  key store也需要为每种flavor + variant组合（是否可以为每种flavor配置sign config？）生成一个keystore rule
+  其他buck文件
  +  .buckconfig，配置整个buck项目
  +  manifest生成有一个genrule，需要一个buck文件描述
  +  .okbuck目录下保存的是所有的依赖本地文件，每个目录都需要一个buck文件进行描述
+  整个过程
  +  遍历每个sub project
  +  判断类型，不同类型生成不同的buck文件（由BuckFile进行封装），且它们的生成逻辑分离到各个实现类中
  +  BuckFile生成后，统一进行打印、依赖文件处理
+  Windows兼容的部分暂时移除

## exopackage支持
支持exopackage时，multidex的`primary_dex_patterns`，以及app lib的依赖，目前都是交给用户配置的，目前还没有好的方案自动化获取。

而用户配置的具体值，只能依靠用户逐步测试，根据buck打包/app运行报错信息，逐步完善。不过这一步也还是值得的，一旦完成之后，后面基本不用修改。
