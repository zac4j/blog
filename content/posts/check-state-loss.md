---
title: "Check State Loss"
date: 2020-02-11T16:30:26+08:00
description: "Fragment state loss"
tags: ["Fragment", "DialogFragment", "State Loss"]
categories: ["Fragment"]
---

#### 背景

最近做的一个 DialogFragment 在少数设备上会偶发闪退，Fabric 上的 Stacktrace 信息如下：

``` java
Fatal Exception: java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
       at android.support.v4.app.FragmentManagerImpl.checkStateLoss(SourceFile)
       at android.support.v4.app.FragmentManagerImpl.enqueueAction(SourceFile)
       at android.support.v4.app.BackStackRecord.commitInternal(SourceFile)
       at android.support.v4.app.BackStackRecord.commit(SourceFile)
       at android.support.v4.app.DialogFragment.show(SourceFile）
```

看起来是弹窗在 show 的时候，发生了 state loss，粗略 copy 了下 StackOverflow 上的回答，做了如下修改：

``` java
fun show(manager: FragmentManager?) {
 try {
   val ft = manager?.beginTransaction()
   ft?.add(this, "tag of dialog")
   ft?.commitAllowingStateLoss()
 } catch (e: Exception) {

 }
}
```

重写了 show 方法，允许 state loss，并加了异常捕捉，后续观察 Fabric， show 的时候的确没再出现异常情况，但 dismiss 的时候还是有闪退出现，异常信息如下：

``` java
Fatal Exception: java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
       at android.support.v4.app.FragmentManagerImpl.checkStateLoss(SourceFile)
       at android.support.v4.app.FragmentManagerImpl.enqueueAction(SourceFile)
       at android.support.v4.app.BackStackRecord.commitInternal(SourceFile)
       at android.support.v4.app.BackStackRecord.commit(SourceFile)
       at android.support.v4.app.DialogFragment.dismiss(SourceFile）  
```

可以看到与 show 出现的闪退如出一辙，只是闪退触发的位置换成了 dismiss，按照之前对 show 的修改，是不是也可以把 dismiss 方法也改下呢，实际上还真有 `dismissAllowingStateLoss()` 方法。。。但是真的可以这样写么？是什么原因导致的 state loss？如何才能有效避免这种情况呢？

仔细看了下 Stack Overflow 问题下的评论，发现 [Alex Lockwood][alex] 写的[一篇文章][article]分析的很全面，下面对重点内容翻译一下。

#### 为什么会抛异常

简单来说，就是你在 Activity state saved 后，仍旧 commit 一个 FragmentTransaction，从而导致出现了所谓的 `Activity state loss` 现象。所以 `onSaveInstanceState()` 时发生了什么？

##### onSaveInstanceState

Android 系统可以随时终止进程以释放内存，因此置于后台的 Activity 随时可能会被清理。Android framework 提供给每个 Activity 在被系统销毁前调用 `onSaveInstanceState` 以保存其状态(state)的机会，在随后恢复(restore)被保存的状态时，用户会感觉在无缝切换前/后台Activity，而不会察觉 Activity 会系统清理过。

当调用 `onSaveInstanceState` 方法时，Android Framework 将为 Acitvity 提供一个 Bundle 对象保存其状态，Activity 将记录 dialogs, fragments, views 的状态。在这个方法返回时，系统通过 Binder 接口将 Bundle 对象打包(parcel)到 System Server 进程。随后当系统决定重建(recreate) Activity 时，它将之前保存的 Bundle 发回 App，来恢复 Activity 旧的状态。

所以为什么回抛异常？原因就是在 `onSaveInstanceState` 调用后，你调用了 `FragmentTransaction#commit()`，因此 `onSaveInstanceState` 方法返回的 `Bundle` 对象并不包含该事务(transaction)。在用户的角度来看，事务被忽略，UI 状态丢失，Android 为了避免这种情况,只要发生状态丢失(state loss)便立即抛出 `IllegalStateException`。

#### 何时抛出异常

从 Honeycomb 开始，`onSaveInstanceState()` 在生命周期 `onStop()` 方法之前调用，而不是 `pre-Honeycomb`时的 `onPause()` 之前。

#### 如何避免异常

这里有一些关于在 App 中使用 `FragmentTransaction` 的建议：

+ **注意** 在 Activity 的生命周期方法 *提交事务(commit transaction)*：
当你在 `onCreate()` 方法 commit 事务可能从来不会碰到问题，但当你在 `onActivityResult()`、`onStart()`、`onResume` commit 时，事情变得有趣了起来。比如，你不应该在 `FragmentActivity#onResume()` 方法里 commit 事务，因为在[某些情况][p]下，Activity 会在恢复状态(restore state)之前调用 `onResume()` 方法。如果你的 App 需要在 `onCreate()` 方法以外 commit 事务，尝试在 `FragmentActivity#onResumeFragments()` 或 `Activity#onPostResume()` 方法内 commit，这两个方法会确保在 Activity restore state 后调用，因此避免了 state loss 的可能。对于这个例子的描述，可以参考 StackOverflow 上这个[回答][answer]。

+ **避免** 在异步回调方法中操作事务:
主要原因是，在异步回调方法执行时，并不了解 Activity 当前的生命周期状态，因此很可能在 `onStop()` 调用时 commit 事务，从而抛出异常。这种情况可以参考 StackOverflow 上[这个回答][answer1]和[这个回答][answer2]。

+ `commitAllowingStateLoss()` 仅作为最后的手段：
`commitAllowingStateLoss()` 与 `commit()` 方法唯一的区别便是该方法在状态丢失时不会抛异常。一般情况下不要使用该方法，一种更好的方式，便是确保在 Activity save state 之前 commit 事务。

#### 最后

回归到我的 DialogFragment 为何出现异常，原因是我 commit transaction 位置是在 `FragmentActivity#onResume()`方法内:-)

[alex]:https://twitter.com/alexjlockwood
[article]:https://www.androiddesignpatterns.com/2013/08/fragment-transaction-commit-state-loss.html
[p]:http://developer.android.com/reference/android/support/v4/app/FragmentActivity.html#onResume()
[answer]:http://stackoverflow.com/q/16265733/844882
[answer1]:http://stackoverflow.com/q/8040280/844882
[answer2]:http://stackoverflow.com/q/7992496/844882
