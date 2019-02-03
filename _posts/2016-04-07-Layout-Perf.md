---
layout: post
title: RelativeLayout, FlexLayout 及其他 layout 性能对比
tags:
    - 安卓开发
    - 性能优化
---

一直以来，我们都在各种场合、各种文章中看到避免使用 RelativeLayout、避免使用过多的 layout 嵌套，因为它们存在很大的性能开销。开发的过程中也确实在留意这一点，然而每每编写 layout 文件时，都会怀疑，这样或者那样，到底会变快，还是变慢？本文就针对简单的 layout 和复杂的 layout，是否使用 RelativeLayout 的性能进行了测试，此外，还对最近很火的 [FlexLayout](https://github.com/mmin18/FlexLayout){:target="_blank"} 也进行了测试。测试代码和测试结果数据都可以在 [Github 获取](https://github.com/Piasy/AndroidPlayground/blob/master/perf/LayoutPerfDemo){:target="_blank"}。

## TL; DR
+ UI 简单时，RelativeLayout, FlexLayout 和其他 layout 的性能差异很小
+ UI 很复杂时，RelativeLayout 和 Flexlayout 可以避免嵌套，性能通常要优于其他 layout 的嵌套实现
+ 列表元素由于每个 item 都会反复渲染，所以即便性能差异微小，累计差异还是影响很大的
+ 小马过河，不同实现方式的性能，最好还是实际测试一下，不至于轻信了他人的谬论

## 测试内容及方法
这里主要测试三部分的性能：inflate、measure、layout。inflate 通过测量 fragment 在 onCreateView 中进行 inflate 的时间，而 measure 和 layout 则通过实现 RelativeLayout/FrameLayout/GridLayout/FlexLayout 的子类，在其 onMeasure 和 onLayout 中测试父类实现执行的时间，并把它们作为 layout 的根节点。

实验对比了两种复杂度不同的 UI 下使用单层 RelativeLayout，不使用 RelativeLayout（其他 layout 可能需要嵌套），使用单层 FlexLayout 这三者的性能。两种 UI 的效果如下图：

<img src="/img/201604/two-ui.png" alt="两种 UI 的效果">

其中左图是对 AndroidStudio 预览的截图，因为这个 layout 只有下半部分。左图很简单，只有4个 View，而右图则就非常复杂了，它们都是 YOLO APP 中的真实界面。实验时，反复重新加载每种不同的 fragment（2 * 3 一共6种 fragment）20次，统计 inflate, measure, layout 三种操作消耗的时间。测试手机为 Nexus 5X，安卓 6.0.1 系统。

## 测试结果

### 简单 UI 的测试结果

<img src="/img/201604/simple_layout_perf_result.png" alt="简单 UI 测试结果">

可以看到，第一组数据比较异常，明显超出其他组数据，去除这一组数据之后，我们看到，三种实现方式的耗时平均值为：

 -- | inflate (ns) | measure (ns) | layout (ns)
RelativeLayout | 3325842 | 947464 | 108585 
FrameLayout | 3159841 | 879161 | 112988
FlexLayout | 5278923 | 796837 | 111414

可以看到，RelativeLayout 和 FrameLayout 三种操作的耗时都差不多，差别都小于 1 ms，而 FlexLayout 的 inflate 明显耗时长于前两者，因为它的 layout 属性解析过程更复杂，理所当然需要更长的时间，不过差别也就是 2 ms，而且 measure 和 layout 耗时与前两者都差不多。在这个简单的 UI 中，FlexLayout 的强大功能并未体现出来，但是在下面的复杂 UI 中我们将发现它真的无比强大！

### 复杂 UI 的测试结果

<img src="/img/201604/complex_layout_perf_result.png" alt="复杂 UI 测试结果">

同样，我们也将异常的第一组数据剔除，然后看看平均值：

 -- | inflate (ns) | measure (ns) | layout (ns)
RelativeLayout | 17479435 | 2268045 | 822163 
GridLayout | 20350271 | 3270156 | 1177185
FlexLayout | 21698676 | 2703914 | 1001549

这里由于 UI 太复杂，三种情况 inflate 的时间都急剧增加，此时 FlexLayout 所需的时间依然最长，需要 21.7 ms，而 GridLayout 其次，需要 20.4 ms，RelativeLayout 反倒是最快的了，只需 17.5 ms。原因很简单，如此复杂的效果，以 GridLayout 为根节点的实现方式，存在大量的 layout 嵌套，最多存在 4 层嵌套！加载过程当然更久了。同样，由于存在深层嵌套，GridLayout 实现版本的 measure 和 layout 时间都比 Flexlayout 和 RelativeLayout 要长，但都不超过 1 ms。

inflate 的时间只需要花费在创建界面的时候，后续的更新操作主要是在 measure 和 layout 上，而从这里我们可以看出，在不嵌套的情况下，RelativeLayout 和 FlexLayout 的性能和其他 layout 几乎是没有差别的。当然前提是其他 layout 的实现存在嵌套，但这个前提是显然成立的，一旦 UI 变得复杂，其他 layout 不嵌套几乎是不可能实现想要的界面效果的。

而且我们可以看一下复杂 UI 时三种实现的 layout 代码，代码较长，我就不贴进来了，不过强烈建议看一看，尤其是 [FlexLayout](https://github.com/Piasy/AndroidPlayground/blob/master/perf/LayoutPerfDemo/src/main/res/layout/fragment_complex_flex.xml){:target="_blank"} 和 [GridLayout](https://github.com/Piasy/AndroidPlayground/blob/master/perf/LayoutPerfDemo/src/main/res/layout/fragment_complex_other.xml){:target="_blank"} 两种版本的实现。相比之下，FlexLayout 的实现非常优雅，对齐和定位功能非常强大，基本上可以说，有了 FlexLayout，动态调整 UI 之外的需求都不需要使用 Java 代码来设置布局了。

## 特殊情况
这里我对比的，都是非列表元素的界面，一旦涉及到列表（RecyclerView, ListView 等），由于每个 item 都会反复绘制，所以即便很小的差别，也会对界面的流畅性产生严重的影响。所以列表元素 item 的布局，就一定要小心谨慎了。

## 结论
+ UI 简单时，RelativeLayout, FlexLayout 和其他 layout 的性能差异很小
+ UI 很复杂时，RelativeLayout 和 Flexlayout 可以避免嵌套，性能通常要优于其他 layout 的嵌套实现
+ 列表元素由于每个 item 都会反复渲染，所以即便性能差异微小，累计差异还是影响很大的
+ 小马过河，不同实现方式的性能，最好还是实际测试一下，不至于轻信了他人的谬论
