---
layout: post
title: 打造鲁棒的安卓APP，从此告别 Activity Not Found 错误和 Activity State Loss 错误
tags:
    - 安卓开发
---

开发安卓APP的过程中，肯定有不少人遇见过 Activity Not Found 错误和 Activity State Loss 错误，前者是由于启动的目标 intent 对应的 activity 不存在，后者则是由于在 activity onSaveInstanceState 函数被调用之后进行了 fragment transaction，关于后者[有一篇文章](http://www.androiddesignpatterns.com/2013/08/fragment-transaction-commit-state-loss.html){:target="_blank"}总结得非常到位，[这一篇译文](http://jaredlam.github.io/blog/2015/12/23/android-fragment-transactions-and-activity-state-loss-yi){:target="_blank"}翻译得也还不错，建议看看。本文则主要介绍我的一个开源库 [SafelyAndroid](https://github.com/Piasy/SafelyAndroid){:target="_blank"}，其中整合了解决这两类问题的最佳实践，让我们一起利用它打造鲁棒的安卓APP吧！

## 安全地启动 Activity
在[ Developer 官方教程](http://developer.android.com/training/basics/intents/sending.html#StartActivity){:target="_blank"}里面有这样一段代码：

``` java
// Verify it resolves
PackageManager packageManager = getPackageManager();
List<ResolveInfo> activities = packageManager.queryIntentActivities(mapIntent, 0);
boolean isIntentSafe = activities.size() > 0;

// Start an activity if it's safe
if (isIntentSafe) {
    startActivity(mapIntent);
}
```

以此确保 intent 对应的 activity 存在，从而避免 ActivityNotFoundException，SafelyAndroid 根据此建议，封装了一系列安全启动 activity 的接口：

``` java
StartActivityDelegate.startActivitySafely(fragment, intent)

StartActivityDelegate.startActivitySafely(fragment, intent, options)

StartActivityDelegate.startActivityForResultSafely(fragment, intent, requestCode)

StartActivityDelegate.startActivityForResultSafely(fragment, intent, requestCode, options)

StartActivityDelegate.startActivitySafely(context, intent)

StartActivityDelegate.startActivitySafely(context, intent, options)

StartActivityDelegate.startActivityForResultSafely(activity, intent, requestCode)

StartActivityDelegate.startActivityForResultSafely(activity, intent, requestCode, options)
```

可以从 context, fragment, support v4 fragment 安全的调用 startActivity，或者 startActivityForResult。

## 安全地 dismiss dialog fragment
dismiss 一个 dialog fragment 实际上就是进行了一次 fragment transaction，本文开头提到的 Activity State Loss 就是在 activity onSaveInstanceState 函数被调用之后进行了 fragment transaction 导致的。SafelyAndroid 对 dismiss dialog fragment 进行了单独的封装，以保证安全地 dismiss ，而更一般的 fragment transaction，则请见下节。

``` java
dialogFragmentDismissDelegate.safeDismiss(dialogFragment)

dialogFragmentDismissDelegate.onResumed(dialogFragment)

supportDialogFragmentDismissDelegate.safeDismiss(dialogFragment)

supportDialogFragmentDismissDelegate.onResumed(dialogFragment)
```

同样，SafelyAndroid 提供了针对 fragment 和 support v4 fragment 的两套 API，调用 `safeDismiss` 时，如果当前 dialog fragment 处于 resumed 状态，则立即 dismiss，并返回 `false`，否则返回 `true`，并在 `onResumed` 函数被调用时再 dismiss。如此，就达到了既不会发生 Activity State Loss 错误，也不会存在 dialog fragment 无法隐藏的 bug 了。

使用时，需要让 dialog fragment 内部持有一个 `DialogFragmentDismissDelegate` 或者 `SupportDialogFragmentDismissDelegate` 对象，需要 dismiss 时，调用这个 delegate 的 `safeDismiss` 方法，并在 `onResume` 生命周期回调中调用 delegate 的 `onResumed` 方法，这样就能彻底避免 dismiss dialog fragment 而触发的 Activity State Loss 错误了。

## 安全地进行 fragment transaction
SafelyAndroid 提供了以下几个 API 用于安全地进行 fragment transaction：

``` java
fragmentTransactionDelegate.safeCommit(transactionCommitter, transaction)

fragmentTransactionDelegate.onResumed()

supportFragmentTransactionDelegate.safeCommit(transactionCommitter, transaction)

supportFragmentTransactionDelegate.onResumed()
```

使用时，进行 fragment transaction 的类（activity 或者 fragment），需要实现 `TransactionCommitter` 接口，用来检查其是否处于 resume 状态。commit 一个 fragment transaction 时，调用 delegate 对象的 `safeCommit` 方法，如果此时处于 resume 状态，则可以立即 commit，否则就会延迟到 `onResumed` 被调用时再 commit。

## 使用内置的 safely 组件
SafelyAndroid 提供了 `SafelyActivity`, `SafelyDialogFragment`,
`SafelyFragment`, `SafelyAppCompatActivity`, `SafelySupportDialogFragment`, 和
`SafelySupportFragment` 这6个内置的基础组件，它们封装了上述的安全行为，只需要让 activity, fragment, dialog fragment 继承自它们，就可以在子类中调用 `safeDismiss()` 和 `safeCommit(transaction)` 了。

## 实现自己的 base 组件
如果 activity 和 fragment 等需要继承自其他的基类，则可以利用 delegate 来实现上述安全行为，只需要参照[内置的安全组件实现方式](https://github.com/Piasy/SafelyAndroid/tree/master/safelyandroid/src/main/java/com/github/piasy/safelyandroid/component){:target="_blank"}即可。
