---
layout: post
title: OkBuck，一周年啦！
tags:
    - 安卓开发
    - BUCK
---

## 渊源

我从 15 年 9 月份开始了解到快速打包相关的技术，此时已经饱受 Gradle 打包龟速的痛苦，一次 one line edit build 就要一分半钟。

首先了解到的是 LayoutCast，但由于它只支持 Android 5.0 以上（ART）的手机，虽然 5.0 的测试机肯定有，但还有大多数测试机不是 5.0，还是有很多时候会比较慢，所以没有采用。

这时候 BUCK 进入了视野。

国庆期间尝试了一下 BUCK，深深觉得下载依赖的 jar/aar 文件、编写 BUCK 脚本特别麻烦，尤其是一旦加了新的依赖，还需要维护 BUCK 脚本以及依赖文件，是一件持续的麻烦事。恰好当时想要趁着国庆期间做点东西，而 BUCK 配置的编写与维护也有可能自动化，所以萌生了 OkBuck 的想法。

OkBuck 的命名受到了 Okio 和 OkHttp 的启发。

OkBuck 的目标，是通过读取工程的 Gradle 配置，自动生成 BUCK 脚本，免去开发者下载依赖的 jar/aar 文件，编写、维护 BUCK 脚本、处理依赖之间的冲突等繁琐又容易出错的工作。

## 发展

整个项目的发展还是比较顺利的。

从最开始的 PoC 阶段，手动编写 BUCK 示例工程、读取 Gradle 相关配置、生成简单的 BUCK 配置，到第一个可用版本 `0.0.1` 发布，经过了 20 多个小时的开发。

从 `0.0.1` 到能够满足公司项目的 `0.2.7`，经过了 11 天 近 50 个 commit。这时我们公司的项目 one line edit build & install **从 100s 降低到了 10s，10 倍速！**但是这期间公司的项目经历了痛苦的转变：移除 ButterKnife 的使用、移除 lambda 表达式的使用、依赖的梳理、manifest 等问题。花了将近 3 天的时间，但构建性能的提升现在看来非常值得。

而此时 OkBuck 的代码还非常原始，到处都是 magic code，代码结构非常不合理，对 Gradle API 的使用也基本都是试出来的，在完成了第二轮重构之后，在 10 月 18 号发布了 `0.3.0`，一定程度上优化了代码。

后续经过了三个多月的持续改进，功能终于逐步完善，也陆陆续续有了更多人了解到了 OkBuck，开始使用了 OkBuck。

正好在 16 年农历新年之前，公司项目进度没那么吃紧，我就趁此机会好好总结了一下 OkBuck 的工作原理，分析解决问题的方式、架构、代码的不优雅之处，进行了第三轮重构，另外完善部分功能，整理文档之后，发布了 1.0 版本。在公司年会期间，我都还抱着电脑在进行 OkBuck 的重构。

后来又经过了三个多月的维护，项目出现了一个缓慢期，但很快随着来自 Uber infrastructure 组的 leader，[Gautam Korlam](https://github.com/kageiit){:target="_blank"} 的加入，OkBuck 迎来了新生。

![kageiit_avatar.jpeg](/img/201609/kageiit_avatar.jpeg)

Gautam Korlam 非常看好 OkBuck 项目，并为此投入了大量时间（包括他们组的其他工程师），而我由于工作业务比较忙，就基本没有再参与 OkBuck 的开发了。

Gautam Korlam 非常猛，码力超群，他住在三藩市，经常搞到凌晨两点，在初期阶段我基本每天早晚都会和他在 Hangouts 上进行讨论，甚至有一天他还表示要和我视频聊天，但被我婉拒，我还是太害羞。

随着 Gautam Korlam 把 OkBuck 改造得越来越好，在 16 年 5 月份，我们创建了 OkBuilds 组织，并把相关 repo 进行了转移，然后重新基于新的插件名和版本号开始了 `0.x` 的开发。现在基本是他们全组都在对这个项目进行维护。

从 4 月下旬至今，项目一共提交了一百多个 commit，到今天发布了 `0.6.0`，主要进行了以下几大改进：

+ 项目代码从纯 Java 改成 Groovy 语言，利用了更多 Groovy 函数式编程以及其他方便的特性；
+ 实现方式进一步优化，代码结构进一步调整，使得项目的可扩展性大大提高；
+ 解决了资源引用的问题；
+ 实现了 ButterKnife 的兼容方案（为 ButterKnife 实现了 library module 的支持）；
+ 实现了 aidl 的支持；
+ 实现了 RetroLambda 的兼容处理，终于又可以使用 lambda 了！
+ 实现了 JUnit 测试、Espresso 测试的支持；
+ 引入 buck wrapper：buckw；
+ ……

总的来说，基于 Gradle 项目引入 BUCK 的成本极大地降低了，BUCK 的使用体验也极大地提升了，对很多项目真的可以做到两行代码开始使用 BUCK！

当然，项目现在依然有一些限制，例如由于 BUCK 产生的资源值都不是 `final`，所以无法应用在注解中，像 AndroidAnnotation 这样的库就依然是不兼容的，而对 ButterKnife 的兼容处理、aidl 的支持，都需要按照特定的方式编写代码。但是相比于 BUCK 带来的急速 build 体验，这一丁点代价几乎不值一提了。

当然，在这期间也出现了很多 build 加速的方案，JRebel、Instant Run 以及阿里开源的 freeline。对于我们公司的项目来说，JRebel 需要付费；Instant Run 不甚稳定，经常出现 swap 之后代码更改并未生效的情况，我们根本没有信心确定代码是否写得正确；而 freeline 刚开源时引入没有成功，遇到了一些问题，而它目前对注解处理仍然不支持。所以目前来说，BUCK 依然是最佳解决方案，而且随着它的快速完善，我相信较长一段时间内，BUCK 依然会是不可替代的存在。

最后有一点值得一提，BUCK 相比于 Gradle，确实有些功能限制，但对于日常开发的 build 来说，BUCK 的功能是完备的，而且速度极快，对于 release build 来说，由于 OkBuck 保持了 Gradle 和 BUCK 的共存，使用 Gradle 进行 release build 是完全没有问题的。这样 BUCK debug build 加上 Gradle release build，let's build better life :)

## 披露一些数据

首先是 OkBuck 项目的 commit graph：

![okbuck_commit_graph.png](/img/201609/okbuck_commit_graph.png)

可以看到，有两个高峰期，分别是 15 年 10 月我的初期开发，以及 16 年 4 月 Gautam Korlam 的二次开发。

然后是 OkBuck 插件的下载量：

![okbuck_bintray_download_number.png](/img/201609/okbuck_bintray_download_number.png)

可以看到，`0.5.3` 版本的下载量有 151 次，这也基本表明，至少有数十个团队在使用 OkBuck。据 Gautam Korlam 透露，硅谷 Lyft、Airbnb、Lookout（当然包括 Uber 了）等团队都在使用 OkBuck。此次参加 MDCC 期间，微信等大团队的诸多前辈也都了解过 OkBuck 项目，当然他们有没有在使用我就没有细问了。

最后我们看看 OkBuck 插件的下载 ip 分布：

![okbuck_bintray_download_country.png](/img/201609/okbuck_bintray_download_country.png)

可以看到，美国的下载量最多，其次是中国，可以说 OkBuck 的用户基本遍布全球。

而在 16 年 9 月 29 即将举办的 [DROIDCON NYC 2016 上](http://sched.droidcon.nyc/showSession/72039){:target="_blank"}，Gautam Korlam 将做一次关于 OkBuck 的演讲，届时 OkBuck 将会进入更多团队的视野，也能给更多开发者带来帮助。

## 几点感想

维护一个开源项目将近一年时间，遇到最大的挑战就是和工作时间的冲突，而 Gautam Korlam 最大的好处就是他的工作内容就是开发维护这些工具，所以他是更合适的维护者。

而关于时间，最近还有另一件事情让我感触颇深：去年 9 月份，我给 OkHttp 的 web-socket 实现提了一个 issue，当时由于信息不足，没能立即解决，但 Square 团队（JW 为主）一直对这个 issue 保持关注，最后终于在一年之后、经过了一个大版本的 release 之后，解决了这个问题，堪称业界良心。

一旦造出了轮子，就不要轻易放弃维护，不然对用户来说真的是很大的伤害。

（_2016.09.29 更新_）就在今天，OkBuck 从 OkBuilds 正式 transfer 到了 Uber，开始全方位迎来 Uber 正式的开发和维护，我相信这会是 OkBuck 发展中的一个重大里程碑。（不过说实话，自己做了一年的项目，就这样嫁入豪门，多少还是有点失落的，但和广大父母一样，哭过之后，更多的是喜悦和自豪吧！）

最后，OkBuck 依然处在高度活跃的开发中，欢迎大家进行尝试、体验，并提出遇到的问题，我们将在第一时间进行解答，以及进行完善。请大家和我们一起，打造更好的 Android build 体验。
