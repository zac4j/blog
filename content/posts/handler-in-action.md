---
title: "Handler in Action"
date: 2020-04-14T21:53:54+08:00
tags: ["handler"]
description: "Handler 的实际应用"
categories: ["android"]
author: "Zac"
---

Handler 在使用时常见的问题

<!-- more -->

## 1.Memory Leaks

### 1.1 背景

Handler 在使用下面这种实现方式处理消息时:

```kotlin
class MainActivity: Activity() {

    private val mLeakedHandler = object: Handler() {
        override fun handleMessage(msg: Message) {
            super.handleMessage(msg)
        }
    }
```

[Android Lint][lint] 会发出这样的警告: `This Handler class should be static or leaks might occur(anonymous android.os.Handler)`

我们在 Handler 构造方法里可以看到：

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

这里会检测 Handler 的实现类是否为匿名类/成员类/局部类之一，以及是否使用 `static` 关键字修饰，未使用的话便会有内存泄漏的警告。

那么内存泄漏是如何产生的呢，为何加 static 就可以解决泄漏问题，我们接着看。

### 1.2 内存泄漏是如何产生

在继续讨论之前我们先达成如下共识：

+ 进入 MessageQueue 队列的 msg 对象，其 msg.target 便是对应的 Handler 对象，当 Looper 执行到该 msg 时，会调用 `Handler.handleMessage(Message)` 处理消息。
+ 在 Java 语言中，非静态（Non-Static）内部类（InnerClass）或匿名类（AnonymousClass）会隐式的持有外部类的引用，同理，静态内部类就不会。
+ 我们在堆中创建内存而忘记删除它，就会导致了[内存泄漏][leaks]。

可以看如下的实现：

```kotlin
class MainActivity : AppCompatActivity() {

    private val mHandler = object: Handler() {
        override fun handleMessage(msg: Message) {
            super.handleMessage(msg)
        }
    }

    private val mTask = Runnable { /*do something*/ }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        mHandler.postDelayed(mTask, 5 * 60 * 1000L)
  
        finish()
    }
}
```

上面的代码中，MainActivity 在 mHandler.postDelay 任务后便立即销毁，此时主线程中的 mHandler 仍然隐式的持有外部 MainActivity 的引用，该引用会直到 `Handler.handleMessage(Message)` 在 5min 后执行后消失，也就是 MainActivity 销毁后并不会被及时的回收，MainActivity 内的资源得不到释放，从而导致内存泄露。

### 1.3 如何避免内存泄露

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

    private val mTask = Runnable { /*do something*/ }

    private val mHandler = OkHandler(this)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        mHandler.postDelayed(mTask, 5 * 60 * 1000L)

        finish()
    }
}
```

## 2.async Handler

我们先看一下 HandlerCompat 的 createAsync(looper) 方法：

``` java
/**
 * Create a new Handler whose posted messages and runnables are not 
 * subject to synchronization barriers such as display vsync.
 */
public static Handler createAsync(@NonNull Looper looper) {
    Exception wrappedException;

    if (Build.VERSION.SDK_INT >= 28) {
        return Api28Impl.createAsync(looper);
    } else if (Build.VERSION.SDK_INT >= 17) {
        try {
            // This constructor was added as private in JB MR1:
            // https://android.googlesource.com/platform/frameworks/base/+/refs/heads/jb-mr1-release/core/java/android/os/Handler.java
            return Handler.class.getDeclaredConstructor(Looper.class, Handler.Callback.class,
                    boolean.class)
                    .newInstance(looper, null, true);
        } catch (Exception e) {
            throw ...
        }
        // This is a non-fatal failure, but it affects behavior and may be relevant when
        // investigating issue reports.
        Log.w(TAG, "Unable to invoke Handler(Looper, Callback, boolean) constructor",
                wrappedException);
    }
    return new Handler(looper);
}
```

通过方法的注释我们可知, async Handler 发出的 message 或 runnable 可以无视 **同步屏障 (synchronization barriers)**, 展示 VSYNC 是一种同步屏障。

那么什么是同步屏障呢，我们可以在 MessageQueue.postSyncBarrier() 看到：

``` java
/**
 * When the barrier is encountered, later synchronous messages in the queue are 
 * stalled (prevented from being executed) until the barrier is released by 
 * calling {@link #removeSyncBarrier} and specifying the token that identifies
 * the synchronization barrier.
 * @hide
 */
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}

/**
 * Removes a synchronization barrier.
 *
 * @param token The synchronization barrier token that was returned by
 * {@link #postSyncBarrier}.
 *
 * @throws IllegalStateException if the barrier was not found.
 *
 * @hide
 */
@UnsupportedAppUsage
@TestApi
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

同步屏障会阻塞当前队列，直到移除屏障队列才恢复运行。`postSyncBarrier()` 和 `removeSyncBarrier(token)` 都是隐藏的方法，只能通过反射的方式调用。

async Handler 在 enqueue Message 时会调用 Message.setAsynchronous(true) 方法。

``` java
// Handler.java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    // async Handler => mAsynchronous = true
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

那么为何 asyncchronos 的 Message 可以立即执行呢，我们可以在 `MessageQueue.enqueueMessage()` 的实现中看到:

``` java
// android/os/MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    ...
    msg.markInUse();
    msg.when = when;
    Message p = mMessages;
    boolean needWake;
    ...
    // Inserted within the middle of the queue.  Usually we don't have to wake
    // up the event queue unless there is a barrier at the head of the queue
    // and the message is the earliest asynchronous message in the queue.
    needWake = mBlocked && p.target == null && msg.isAsynchronous();
    Message prev;
    for (;;) {
        prev = p;
        p = p.next;
        if (p == null || when < p.when) {
            break;
        }
        if (needWake && p.isAsynchronous()) {
            needWake = false;
        }
    }
    msg.next = p; // invariant: p == prev.next
    prev.next = msg;


    // We can assume mPtr != 0 because mQuitting is false.
    if (needWake) {
        nativeWake(mPtr);
    }
}
```

当有同步屏障时，且入列的 msg 是 async 的会唤起休眠的队列，这就是为何 async Handler 可以无视同步屏障的原因。

既然同步屏障是系统隐藏的 API，那么什么地方用到了呢，我们可以在

[lint]:https://tools.android.com/tips/lint-checks
[hic]:http://localhost:1313/posts/handler-in-action/
[leaks]:https://www.geeksforgeeks.org/what-is-memory-leak-how-can-we-avoid/
