---
title: "Coroutines Introduction"
date: 2020-04-25T11:40:03+08:00
description: "Coroutines Intro"
tags: ["coroutines"]
categories: ["kotlin"]
author: "Zac"
---

### 协程用来解决什么问题

Kotlin 的协程提供了一种全新的并发处理方式，我们可以使用它来简化安卓异步执行的代码。

在 Android 平台上协程主要用来解决两个问题：

+ 处理耗时任务(Long running tasks)
+ 保证主线程安全(Main-safety)

#### 处理耗时任务

我们常规处理耗时任务是通过异步回调的方式，比如：

```kotlin
fun fetchDoc() {
    get("https://doc.qq.com") { result ->
        show(result)
    }
}
```

通过协程的方式是这样：

```kotlin
// 主线程执行
suspend fun fetchDoc() {
    // 直接返回结果
    val result = get("https://doc.qq.com")
    show(result)
}

// 主线程执行
suspend fun get(url: String) = withContext(Dispatchers.IO) {
    /*IO 线程池执行*/
}
```

可以看到通过协程可以直接返回请求结果，而不用管理请求延迟和线程阻塞，这是如何实现的呢？

协程在常规的函数操作 —— invoke 和 return 之外，还新增了2项：

+ suspend —— 称为挂起或暂停，用于暂停执行当前协程，并保存所有局部变量
+ resume —— 让暂停的协程继续执行

那么 suspend 是如何实现的呢？

> Kotlin 使用堆栈帧来管理需要运行哪个函数及所有局部变量。suspend 协程时，系统会复制并保存当前的栈帧供后续使用，resume 协程时，系统会还原保存的栈帧，然后函数从暂停的位置继续执行。

在上面的协程示例中，`get()`函数在主线程上运行，但会在网络请求前暂停协程，请求完成时，`get()`函数恢复协程。

#### 保证线程安全

Kotlin 提供了三种调度器(Dispatcher)供协程执行任务。协程可以自行暂停，调度器(Dispatcher)负责恢复。

| Dispatcher         |   线程   |       任务设计      |                       用途                      |
|--------------------|:--------:|:-------------------:|:-----------------------------------------------:|
| Dispatcher.Main    | 主线程   | UI 交互和轻量级任务 | 调用 suspend 函数、调用 UI 函数、 更新 LiveData |
| Dispatcher.IO      | 非主线程 | 磁盘和网络 IO 任务  | 数据库、文件、网络处理                          |
| Dispatcher.Default | 非主线程 | CPU 密集型任务      | 数组排序、JSON 解析、处理 Diff 判断             |

上面的示例中，`get()`函数 通过 `withContext(Dispatcher.IO)` 创建一个在 IO 线程池内运行的代码块。

如果某个函数任务涉及到磁盘、网络或者占用过多的 CPU 资源，都应该使用 `withContext(Dispatcher)` 来确保可以在主线程安全地调用。

### 追踪协程

> 协程自身并不能追踪正在处理的任务，使用代码来手动追踪上千个协程是十分困难的，我们虽然可以尝试对所有协程进行追踪，手动确保它们都完成或取消任务，但这样代码会十分复杂和臃肿。如果没有追踪协程，则可能诱发任务泄漏(work leak)，即耗时任务会持续地占用资源执行下去。

为了能够避免这种情况，Kotlin 引入了 **结构化并发(structured concurrency)** 机制，遵循它可以帮助我们追踪所有运行的任务。我们可以使用结构化并发做到下面三件事：

+ 取消任务 —— 取消某项无用的任务
+ 追踪任务 —— 追踪某项正在执行的任务
+ 发出错误信号 —— 任务异常时发出错误信号

#### 使用 Scope 来追踪协程

Kotlin 中定义启动协程必须指定其 CoroutineScope, CoroutineScope 可以追踪或取消所有由它启动的协程。

有两种方式能够启动协程，分别用于不同的场景：

+ `launch` 方式启动新协程不会返回 result，适合不需要执行结果的场景
+ `async` 方式启动新协程并允许我们使用 await 的挂起函数返回 result.

通常我们应该使用 `launch` 方式启动新协程。

```kotlin
scope.launch {
    // launch 可以调用 suspend 函数
    fetchDoc()
}
```

#### 在 ViewModel 中启动协程

当协程和 Android Architecture Components 结合时，我们应该在哪个组件使用协程呢？

答案是 ViewModel，我们大部分任务都是在 ViewModel 中处理，而且 [AndroidX Lifecycle][vs] 从 2.1.0 版本开始已经引入扩展属性 `ViewModel.viewModelScope`，可以更方便地在 ViewModel 中使用协程：

```kotlin
class MainViewModel(): ViewModel() {
    fun setUserDoc() {
        // 启动新的协程
        viewModelScope.launch {
            fetchDoc()
        }
    }
}
```

当 viewModelScope 被清除时，它将自动取消所有它启动的协程，可以保证协程和 ViewModel 的生命周期是一致的。

#### 启动多个协程

可以使用 [`coroutineScope`][cs] 或 [`supervisorScope`][ss] 启动多个协程，结构化并发保证了当 `suspend` 函数返回时，它所处理的任务也都完成了。

```kotlin
suspend fun fetchDocs() {
    coroutineScope {
        async {fetchDoc()}
        async {fetchPdf()}
    }
}
```

Kotlin 确保使用 `coroutineScope` 构造方法不让 `fetchDocs()` 方法发生泄漏，会先将 `coroutineScope` 自身挂起，等待它内部的所有协程完成时，再返回结果。

#### 异常处理

`coroutineScope` 和 `supervisorScope` 两者都适合启动多个协程的场景，区别在于当启动的某一子协程出错时，coroutineScope 将会取消所有协程，而 supervisorScope 会继续执行剩余的协程。

协程中来自 suspend 函数的异常会通过 resume 重新抛给调用方 (Invoker) 来处理，跟函数一样，我们可以用 try/catch 处理异常。

```kotlin
suspend fun getError() {
    coroutineScope {
        async {
            throw IllegalStateException("...")
        }
    }
}
```

#### 参考

[在Android 开发中使用协程][ucia]
[Coroutines Discussion][cd]
[Introduction to Coroutines][itc]

[ucia]:https://mp.weixin.qq.com/s/kPvWOCkMjYRKJSTX4I5VKg
[itc]:https://www.youtube.com/watch?v=_hfBv0a09Jc
[vs]:https://developer.android.google.cn/topic/libraries/architecture/coroutines#lifecycle-aware
[cs]:https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[ss]:https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html
[cd]:https://www.reddit.com/r/androiddev/comments/ftqe6s/
