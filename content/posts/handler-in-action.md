---
title: "Handler in Action"
date: 2020-04-14T21:53:54+08:00
tags: ["handler", "message queue", "looper"]
description: "Handler 消息机制的应用"
categories: ["android"]
author: "Zac"
---

### Memory Leaks

#### 1.1 背景

1.Handler 在使用下面这种实现方式处理消息时:

```kotlin
class MainActivity : Activity() {

    private val mLeakedHandler = object : Handler() {
        override fun handleMessage(msg: Message) {
            super.handleMessage(msg)
        }
    }
```

[Android Lint][lint] 会发出这样的警告: `This Handler class should be static or leaks might occur(anonymous android.os.Handler)`

2.我们在 Handler 构造方法里同样可以看到：

```java
public Handler(@Nullable Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    ...
}
```

这里会检测 Handler 的实现类是否为匿名类/成员类/局部类之一，然后再检测该实现类未使用 `static` 关键字修饰，未使用的话便会有内存泄漏的警告。

那么内存泄漏是如何产生的呢，为何加 static 就可以解决泄漏问题，实现是怎样的呢。

#### 1.2 Handler 如何产生内存泄漏

在继续讨论之前我们先达成如下共识：

+ 进入 MessageQueue 队列的 msg 对象，其 msg.target 便是对应的 Handler 对象，当 Looper 执行到该 msg 时，会调用 `Handler.handleMessage(Message)` 处理消息。
+ 在 Java 中，非静态（Non-Static）内部类（InnerClass）或匿名类（AnonymousClass）会隐式的持有外部类的引用，同理，静态内部类就不会。
+ 我们在堆中创建内存而忘记删除它，就会导致了[内存泄漏][leaks]。

可以看如下的实现：

```kotlin
class MainActivity : AppCompatActivity() {

    private val mHandler = object : Handler() {
        override fun handleMessage(msg: Message) {
            super.handleMessage(msg)
        }
    }

    private val mJob = Runnable { /*do something*/ }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        mHandler.postDelayed(mJob, 5 * 60 * 1000L)
  
        finish()
    }
}
```

上面的代码中，MainActivity 在 mHandler postDelay 任务后便立即销毁，此时主线程中的 mHandler 仍然隐式的持有外部 MainActivity 的引用，该引用会直到 `Handler.handleMessage(Message)` 在 5min 后执行后消失，也就是 MainActivity 销毁后并不会被及时的 GC，MainActivity 内的资源得不到释放，从而导致内存泄露。

#### 1.3 如何避免内存泄露

为了避免内存泄露我们可以使用弱引用：

```kotlin
class MainActivity : AppCompatActivity() {

    private class OkHandler(activity: Activity) : Handler() {
        private val mActivity: WeakReference<Activity> = WeakReference(activity)

        override fun handleMessage(msg: Message) {
            val activity = mActivity.get()
            if (activity != null && !activity.isFinishing) {
                // execute task
            }
        }
    }

    private val mJob = Runnable { /*do something*/ }

    private val mHandler = OkHandler(this)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        mHandler.postDelay(mJob, 5 * 60 * 1000L)

        finish()
    }
}
```

[lint]:http://tools.android.com/tips/lint-checks
[hic]:http://localhost:1313/posts/handler-in-action/
[leaks]:https://www.geeksforgeeks.org/what-is-memory-leak-how-can-we-avoid/
