---
layout: post
title: 安卓基础：task, launchMode, Intent flag
tags:
    - 安卓开发
    - 基础知识
---

Task 是一个从用户角度出发的概念，是一些 Activity 的组合，它们组合起来是为了让用户完成某一件工作（或者说操作）。Task 在 framework 中对应的类是 `com.android.server.am.TaskRecord`，它用一个列表记录着其中的所有 Activity，至于到底是怎么记录的，就要看源码了。

Task 内的 Activity 以栈的形式组织起来，这个 _栈_ 也就是 developer 里面说的 back stack（back stack 只是对 Task 组织 Activity 方式的一种形象描述，back stack 并不对应一个什么类，如果非要对应，那就是 `ArrayList<ActivityRecord>`，所以下文的 Task 和 _栈_ 是等同的）。

栈内的 Activity 不会重新排序，只能 push 或者 pop。栈内的 Activity 可以来自不同的 App，因此可以是运行在不同的进程，但是它们都属于同一个 Task 内，一个典型的例子就是调用系统相机拍照。Task 保证了用户在利用一系列（甚至不同 App 的）组件完成工作时的无缝体验，同时也实现了返回导航。

一个虚拟机只能跑一个进程，一个进程里可以跑多个应用（[声明相同的 android:shareUserID 和 android:process，并用同一个 key 签名](http://stackoverflow.com/a/17664341)），一个应用也可以跑在多个进程中（会有多个 Application 实例）。

新启动一个 Activity 会怎么安放到栈中？

默认情况下，新的 Activity 是不断往当前的栈里 push 的，但我们可以利用 launch mode 和 Intent flag 对这一行为进行控制。

## launch mode

`launchMode` 在 manifest 的 `<activity>` 标签中进行声明，有四种模式：

### standard

默认模式，新的 Activity 不断加入到当前栈顶。

### singleTop

和默认模式相比，新启动一个 Activity 时，如果当前栈顶就是这个 Activity 的实例，那就不会创建新的 Activity，而是调用该实例的 `onNewIntent()` 方法。其他情况行为和默认模式一致。

注意，如果 A 启动 B，在创建新的 B 实例过程中，如果用户按下返回键，A 是可以 resume 的，但如果 B 是 singleTop，且它再尝试启动 B，那么在 `onNewIntent()` 被调用之前，用户按下返回键，B 也不会 resume。更具体的过程，请参见[安卓基础：Activity/Fragment 生命周期](/2017/01/14/Android-Basics-Activity-Fragment-Life-Cycle/)。

### singleTask

这段编辑自[罗升阳的博客](http://blog.csdn.net/luoshengyang/article/details/6714543)：

> 在启动 singleTask 的 Activity时，系统会先检查是否有 Task 的 affinity 值与该 Activity 的 taskAffinity 相同，如果有，Activity 就会在这个 Task 中启动，否则就会在新的 Task 中启动。因此，如果我们想要设置了 singleTask 启动模式的 Activity 在新的 Task 中启动，就要为它设置一个独立的 taskAffinity 属性值。如果不是在新的 Task 中启动，它会在已有的 Task 中查看是否已经存在相应的 Activity 实例，如果存在，就会把位于这个 Activity 实例上面的 Activity 全部结束掉，即最终这个 Activity 实例会位于该 Task 的栈顶，并调用该实例的 `onNewIntent()` 方法。

### singleInstance

和 singleTask 相比，它所在的 Task 将只会包含这个 Activity，即其后启动的 Activity，都将在新的 Task 中启动。

还是有两个问题，启动 singleInstance，会不会在新的 Task？singleInstance 启动新的 Activity，新的 Activity 是不是在新的 Task？

~~答案都是否定的：第一个问题，需要 singleInstance Activity 设置不同的 taskAffinity；第二个问题，需要新的 Activity launchMode 为 singleTask 或 singleInstance，且设置不同的 taskAffinity。~~

~~那 singleInstance 和 singleTask 还有区别吗？现在看来是没有了！~~

**2017.01.17 更新：**上面的结论来自错误的测试结果，代码写错了，manifest 中 singleInstance 都写成了 singleTask。惭愧。

singleInstance 的 Activity，不需要设置 taskAffinity 就可以启动到新的 task；而它启动的新 Activity，只有 taskAffinity 为不同的值时，才会启动到新的 task，至于新 Activity 的 launchMode 是什么，无关紧要。

### back stack 合并？

[developer 文档](https://developer.android.com/guide/components/activities/tasks-and-back-stack.html)中有这么一个例子：

![2017010846558diagram_backstack_singletask_multiactivity.png](https://imgs.babits.top/2017010846558diagram_backstack_singletask_multiactivity.png)

> At this point, the back stack now includes all activities from the task brought forward, at the top of the stack.

developer 的描述很具有迷惑性，其实根本不存在 back stack 的合并，也不存在 Task 的合并（前文我们说过，这两者是一回事），这里只不过是两个 Task 交换了顺序，后台变前台，前台变后台，而返回操作都是现在 Task 内进行返回的，所以当然是 `Y -> X -> 2 -> 1`，而这个返回过程经历了两个 Task（back stack，再次强调，这两个概念是一回事）。

在 developer 中，这一幅图里面关于 Task、back stack 的描述，和这一幅图之前对 Task、back stack 的描述是矛盾的，我琢磨了很久才想透彻，我们统一以之前的概念为准，而对这一幅图所描述的行为，也很好解释。

当然，上述论断只是基于我的逻辑推理，我并没有阅读 framework 中这部分的源码，但通过测试，观察 dumpsys 的结果，以及返回导航的结果，我的推论得到了证实，如果有朋友能够在源码层面对这一结论进行证实，那就完美了。

### 小结

新启动的 Activity 是否在新的 Task，由两个东西一起控制：launchMode 和 taskAffinity，只有 launchMode 为 singleTask 或 singleInstance，且设置不同的 taskAffinity，新的 Activity 才会启动到新的 Task 中。

**2017.01.17 补充：**launchMode 相关的内容，不必太过纠结，我们只需要使用的时候避免复杂用法，并小心验证，而且被问到能把简单的情况答出来，就够了。oasisfeng 老师对这块内容的评价是：“设计的非常失败的部分”。 :)

## Intent flag

launch mode 除了可以在 manifest 中通过 launchMode 属性控制，还能由调用方在 Intent 中设置 Intent flag。

A 启动 B，B 的启动行为受 B 在 manifest 中的声明，以及 A 在 Intent 中的设置。两者冲突时，A 的设置优先。

有些选项只能在 manifest 中设置，有些选项只能在 Intent flag 中设置。

至于通过 flag 控制是否要在新的 Task 中启动，同样受到 taskAffinity 的影响。

## 其他

launch mode 相关的还有更多复杂的内容，比如 taskAffinity 完整的作用，allowTaskReparenting 属性，系统清除 Task 的行为控制……这些内容已经超出了我的面试和工作经验，就暂且到此为止。

## 附录1：launchMode 测试结果

测试代码可以从 [GitHub 获取](https://github.com/Piasy/AndroidPlayground/tree/master/showcase/TaskDemo)。

### singleTask 测试

singleTask 不设置 taskAffinity：

![2017010852activity_single_task_without_task_affinity.png](https://imgs.babits.top/2017010852activity_single_task_without_task_affinity.png)

singleTask 设置 taskAffinity 为包名：

![2017010884648activity_single_task_with_same_task_affinity.png](https://imgs.babits.top/2017010884648activity_single_task_with_same_task_affinity.png)

singleTask 设置 taskAffinity 为包名以外的值：

![2017010879551activity_single_task_with_different_task_affinity.png](https://imgs.babits.top/2017010879551activity_single_task_with_different_task_affinity.png)

### singleInstance 测试

singleInstance 不设置 taskAffinity：

![2017011786410activity_single_instance_without_task_affinity.png](https://imgs.babits.top/2017011786410activity_single_instance_without_task_affinity.png)

singleInstance 设置 taskAffinity 为包名：

![2017011752729activity_single_instance_with_same_task_affinity.png](https://imgs.babits.top/2017011752729activity_single_instance_with_same_task_affinity.png)

singleInstance 设置 taskAffinity 为包名以外的值：

![2017011751645activity_single_instance_with_different_task_affinity.png](https://imgs.babits.top/2017011751645activity_single_instance_with_different_task_affinity.png)

singleInstance（设置 taskAffinity 为包名以外的值）启动 standard（不设置 taskAffinity）：

![2017011751555single_instance_difftaskaff_launch_standard_notaskaff.png](https://imgs.babits.top/2017011751555single_instance_difftaskaff_launch_standard_notaskaff.png)

singleInstance（设置 taskAffinity 为包名以外的值）启动 standard（设置 taskAffinity 为包名以外的值）：

![2017011712232single_instance_difftaskaff_launch_standard_difftaskaff.png](https://imgs.babits.top/2017011712232single_instance_difftaskaff_launch_standard_difftaskaff.png)

singleInstance（设置 taskAffinity 为包名以外的值）启动 singleTask（不设置 taskAffinity）：

![201701178055single_instance_difftaskaff_launch_singletask_notaskaff.png](https://imgs.babits.top/201701178055single_instance_difftaskaff_launch_singletask_notaskaff.png)

singleInstance（设置 taskAffinity 为包名以外的值）启动 singleTask（设置 taskAffinity 为包名以外的值）：

![2017011789836single_instance_difftaskaff_launch_singletask_difftaskaff.png](https://imgs.babits.top/2017011789836single_instance_difftaskaff_launch_singletask_difftaskaff.png)

### Task 合并测试

先设置测试环境，Main 启动 SingleTaskWithDifferentTaskAffinity，后者再启动 SingleTaskWithDifferentTaskAffinity2，注意后两者的 taskAffinity 值相同：

![2017010884876task_merge_setup.png](https://imgs.babits.top/2017010884876task_merge_setup.png)

再通过最近任务，切换到 Main 所在的 Task：

![2017010899241task_merge_recent_switch.png](https://imgs.babits.top/2017010899241task_merge_recent_switch.png)

再从 Main 启动 SingleTaskWithDifferentTaskAffinity2：

![201701084051task_merge_launch.png](https://imgs.babits.top/201701084051task_merge_launch.png)

注意看清楚，上面有三个 TaskRecord 的条目，但实际只有两个 Task，169 和 170，而且这个排列顺序是 most recent first，真正逐步返回的时候，顺序是 `SingleTaskWithDifferentTaskAffinity2 -> SingleTaskWithDifferentTaskAffinity -> Main`。

## 附录2：一个由 taskAffinity 引发的 bug

之前遇到过一个 bug：我有三个 activity，splash、invite、home，splash 和 home 使用了 singleTop 的 launchMode，invite 没有设置 launchMode；首先我从 launcher 启动 app，打开的是 splash，然后 splash 启动 home 并 finish 自己；然后我按 HOME 键回到桌面，再打开浏览器，从网页跳入 app（通过 intent VIEW，url scheme），打开的也是 splash，但这里 splash 检查到是从网页跳入，会启动 invite，并 finish 自己；然后，在 invite 里面，我利用微信 sdk 拉起微信支付，支付完成之后，回到了我的 app（微信是通过启动 wxpay），但这时 resume 的不是 invite，而是 home！

我通过 dumpsys 查看了 stack，从 launcher 启动和从网页跳入是两个不同的 task，分别记为 task1 和 task2，从网页跳入启动到 invite 之后，stack 结构如下：

task2: invite
task1: home

task2 是在栈顶的；但是我支付完成之后，回到我的 app，stack 结构就变成了这样：

task1: home
task2: invite

也就是说，task1 被调到栈顶了。

但是经过一番测试，锤子手机 5.1.1 系统，不存在此问题，因为两次启动 app 是在同一个 task 里面；乐视手机 6.0 存在此问题，因为两次启动 app 不在同一个 task 里面。此外我还发现，如果利用 adb 发送 intent VIEW，在乐视手机上，两次启动也在同一个 task 里，也就不存在问题。华为 P8，6.0 也存在同样的问题。

另外我也还做了一些测试，例如在 invite 里面不是调用微信支付的 sdk，而是启动相机 app 拍照，或者只是利用微信分享的 sdk，在乐视手机上，虽然会有两个 task，但是返回的时候都不会出现 bug，都能正常返回到 invite。

这里看起来至少不是特定 rom 的问题，我怀疑是微信/微信 sdk/我们 app 的问题，但根本不知道从哪里着手分析。从上面的几个结果也能看出不同系统版本表现也不一致，但后来我用原生 6.0 系统试了一下没问题，也就排除了系统版本的问题。

由于这个问题发生在发版前夕，而我又不可能短时间内找到问题的根源进行解决，所以就用了猥琐的办法：当我检测到从浏览器启动时，不管三七二十一，先把所有的后台 activity 全部 finish 掉，再启动相应的 activities，这样就肯定不会有问题了。虽然猥琐，但也奏效。不过这也只能用在我们这种体量不大的 app 中，像微信/今日头条这样日活几亿的 app，这样的方式肯定是不行的 :)

其实问题的关键，在于从浏览器第二次启动 app，启动到了一个新的 task 中（当然，这是马后炮，时隔一个月后，在一位乐视技术专家的帮助下，才找到了问题的根源）。我用乐视手机利用 chrome 打开网页跳进 app，启动到了同一个 task 里面，确实没有问题。

微信支付和相机拍照的结果不一样，是因为微信支付的返回，并不是真的“返回”，而是微信启动了 wxpay 这个 activity，而相机拍照，就真的是 finish 了拍照的 activity，resume 了启动它的 activity。

乐视的浏览器，打开 invite 的时候，是和浏览器在一个 task 内，而微信支付完成之后，启动 wxpay 的时候，wxpay 启动到了 home 所在的 task1，所以把 task1 提到前台了，导致了出现的问题。为 wxpay 设置单独的 taskAffinity 就解决了这个问题，因为 task1 不会被提到前台，所以 wxpay 自动 finish 之后，会回到 invite 所在的 task2。

为什么 wxpay 会启动到 home 所在的 task1？我所有的 activity 都没有声明 taskAffinity（那就都是 app 的包名），微信启动 wxpay 的做法是正确的，wxpay 一定要启动到我的 app 的 task 里面，但由于乐视的浏览器错误的启动了 invite（invite 此时在浏览器的 task2 里），所以 wxpay 肯定不能启动到 task2（因为 task2 的 affinity 和我的 app 包名不一样），但 wxpay 会启动到新的 task 里面吗？不会，因为启动到新的 task 需要同时满足 singleTask/singleInstance 和单独的 taskAffinity，wxpay 都不满足，所以它只能启动到 task1 中（利用 dumpsys 观察，的确如此）。

据微信的朋友透露，微信使用的 flag 是 `Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_MULTIPLE_TASK`，经过试验，这两个 flag 组合起来，不需要单独的 taskAffinity，新的 activity 就会启动在新的 task 里面，但是“最近任务”界面中，却只能看到一个任务。

那这就有两个问题了：1. 为什么 wxpay 不是启动在一个新的 task 里面，而是启动到了 home 所在的 task1？2. 为什么“最近任务”界面中只有一个任务？希望对 framework 熟悉的朋友可以指点迷津。
