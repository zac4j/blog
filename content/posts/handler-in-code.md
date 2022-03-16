---
title: "Handler in Code"
date: 2020-04-12T20:36:33+08:00
tags: ["Handler", "MessageQueue", "Looper"]
description: "Handler 消息机制的实现"
categories: ["Handler"]
author: "Zac"
---

### Handler 的创建

+ 在主线程(UI Thread)中：

```kotlin
class MainActivity : Activity {
    private val mLeakedHandler = object : Handler() {
        override fun handleMessage(msg: Message) {
            super.handleMessage(msg)
        }
    }
}

```

+ 在子线程中：

```kotlin
val thread = object : Thread() {
    override fun run() {
        Looper.prepare()
        val handler = object : Handler() {
            override fun handleMessage(msg: Message) {
                super.handleMessage(msg)
            }
        }
        Looper.loop()
    }
}
```

#### 为什么在子线程创建 Handler 需要准备 Looper，而主线程却不用

因为 ActivityThread 中的 `main()` 方法已经为我们初始化了 Looper

```java
public static void main(String[] args) {

    // Install selective syscall interception
    AndroidOs.install();

    ...

    Looper.prepareMainLooper();

    ...
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

### Handler 和 Looper

#### 为什么使用 Handler 就需要显式的调用 `Looper.prepare()` 和 `Looper.loop()` 呢

我们可以通过观察 Handler 的构造方法理解：

```java
public Handler(@Nullable Callback callback, boolean async) {
    ...

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException("Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

`Looper.myLooper()` 的实现则是：

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

那么 sThreadLocal 在哪里 set 的呢？答案是 `Looper.prepare()` 方法:

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

可以看到 Handler 和 Looper 是绑定的关系，每个 Handler 只会存在一个线程和 Looper，Looper 内的 ThreadLocal 标识对应的线程。

### Handler 的使用

根据官方文档的介绍，Handler 通常有2种用法：

+ Schedule to be executed at some point in the future:
  Schedule Message 和 Runnable 比较容易理解，Handler 提供了如 `sendMessageAtTime(Message, long)` 和 `postAtTime(Runnable, long)` 方法，可以设定在未来某个时刻执行任务。
+ Enqueue an action to be performed on a different thread than your own:
  在别的线程执行任务，我们通常的应用就是在子线程处理耗时任务后，我们可以通过 `Handler(getMainLooper())` 来执行更新 UI 的操作。
  
### Handler 和 MessageQueue

观察源码可以发现 handler 的 `sendMessage(msg)`、`sendMessageDelayed(msg, timeMillis)`、`post(runnable)` 方法最后都是通过调用 `sendMessageAtTime(msg, uptimeMillis)` 实现：

```java
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

这里 `Handler.mQueue` 是在 Handler 构造方法里通过 `mLooper.mQueue` 获取，而 `Looper.mQueue` 是在 Looper 初始化时创建:

```java
// android/os/Handler.java
public Handler(@Nullable Callback callback, boolean async) {
    ...
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException("Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

// android/os/Looper.java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

可以看到像官方文档描述的那样，每个 Handler 实例都关联着一条线程以及这条线程的消息队列(Each Handler instance is associated with a single thread and that thread's message queue)。

#### 消息如何入列

在 `sendMessageAtTime()` 方法最后返回了 `Handler.enqueueMessage()` 的值：

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

`Handler.enqueueMessage()` 调用的 `MessageQueue.enqueueMessage()` 实现如下：

```java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
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
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

我们分开来看，这里 `msg.target` 即消息对应的 handler, 在 `Handler.enqueueMessage()` 方法有赋值。

MessageQueue 并没有队列的数据结构，而是以单链表的形式存储 msg，`mMessages` 为当前链表头部的消息，我们称为 head_msg，当前传入的消息称为 current_msg，

```java
if (p == null || when == 0 || when < p.when) {
    // New head, wake up the event queue if blocked.
    msg.next = p;
    mMessages = msg;
    needWake = mBlocked;
}
```

`if` 的判断是，如果:

+ 1).当前队列没有 head_msg
+ 2).current_msg 是需要立即执行
+ 3).current_msg 的执行时间在 head_msg 执行时间之前

则将 head_msg 标记为 current_msg 的下一条消息，current_msg 标记为头部消息，并且唤醒当前阻塞的队列。

```java
...
} else {
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
}
```

`else` 内的逻辑是，根据 current_msg 的执行时间在消息队列中找到对应的位置入列。

#### 何时执行入列的消息

答案就是在子线程使用 handler 时显式调用的 `Looper.loop()`:

```java
// android/os/Looper.java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        ...
        try {
            msg.target.dispatchMessage(msg);
            ...
        } catch (Exception exception) {
            ...
            throw exception;
        } finally {
            ...
        }
        ...
        msg.recycleUnchecked();
    }
}
```

`Looper.loop()` 的实现就是在无限循环中不断从消息队列中取出 msg，并在 `msg.target`即 handler 中 `dispatchMessage`:

```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

`msg.callback` 对应的是 `Handler.post(runnable)` 传递的 Runnable 对象，而 `mCallback.handleMessage(msg)` 或 `handleMessage(msg)` 则是实例话 Handler 对象时复写的方法。

### 总结

Handler 帮助我们处理 Message 和 Runnable 对象，每个 Handler 实例都关联着一个线程以及这个线程的 MessageQueue。当你创建一个新的 Handler 对象时，它将与当前所在的线程(UI/Work Thread)和 MessageQueue 绑定在一起，通过 Handler 发出的 Message 或者 Runnable(msg.callback) 会根据执行时间插入 MessageQueue，Looper 会持续不断的从 MessageQueue 取出可执行的 Message 通过 Handler.handleMessage(msg) 处理。

如下图所示：![handler](https://i1.wp.com/tutorial.eyehunts.com/wp-content/uploads/2018/08/Android-Handler-Background-Thread-Communicate-with-UI-thread-1.png?resize=1024%2C621&ssl=1)
