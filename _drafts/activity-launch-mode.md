Android Activity 的 launch mode

首先还是要明确一下几个概念：

> task 是一个从用户角度出发的概念，它是一些 activity 的组合，它们组合起来是为了让用户完成某一件工作（或者说操作）。

task 这一概念在 framework 中对应的类是 `TaskRecord`，它有一个 List 成员，记录着其中的所有 activity。

backstack，在 [developer 上的文档中](https://developer.android.com/guide/components/tasks-and-back-stack.html)，其描述如下：

> A task is a collection of activities that users interact with when performing a certain job. The activities are arranged in a stack (the back stack), in the order in which each activity is opened.

据我看到的


## 目的

当 activity A 启动 B 时，B 和当前的 task 是什么关联？利用 launchMode 来定义这件事情。

A 和 B 都有权规定这件事情，A 通过 intent flag 来做，B 则通过在 manifest 里面声明 launchMode 来做。A 的优先级高于 B。

## 模式

![launchmode](/img/201612/activity-launch-mode.png)

几个要点：

+ standard，如果没有其他设置，B 会出现在 A 的 task 尾部（TaskRecord 内用一个 List 保存 Activity，相当于栈顶）
+ singleTop，如果没有其他设置，B 的目标 task 就是 A 所在的 task
