---
title: "Handler in Details"
date: 2020-04-12T20:36:33+08:00
tags: ["handler"]
description: "Handler 消息机制的实现原理"
categories: ["android"]
author: "Zac"
---

### Handler 的创建

+ 在主线程(UI Thread)中：

``` kotlin
class MainActivity: Activity {
    private val mLeakedHandler = object: Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            super.handleMessage(msg)
        }
    }
}

```

+ 在子线程中：

``` kotlin
val thread = object: Thread() {
    override fun run() {
        val looper = Looper.prepare()
        val handler = object: Handler(Looper.myLooper()) {
            override fun handleMessage(msg: Message) {
                super.handleMessage(msg)
            }
        }
        Looper.loop()
    }
}
```

#### 为什么在子线程创建 Handler 需要准备 Looper，而主线程却不用

因为 ActivityThread 中的 `main()` 方法已经为我们初始化了 **Looper**

``` java
// android/app/ActivityThread.java
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

我们可以通过观察 **Handler** 的构造方法理解：

``` java
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
```

`Looper.myLooper()` 的实现则是：

``` java
// android/os/Looper.java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

那么 `sThreadLocal` 在哪里 set 的呢？答案是 `Looper.prepare()` 方法:

``` java
// android/os/Looper.java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

可以看到 **Handler** 和 **Looper** 是绑定的关系，每个 **Handler** 只会存在一个**线程**和 **Looper**，**Looper** 内的 `sThreadLocal` 标识对应的线程。

### Handler 的使用

根据官方文档的介绍，Handler 通常有2种用法：

+ Schedule to be executed at some point in the future:
  Schedule Message 和 Runnable 比较容易理解，Handler 提供了如 `sendMessageAtTime(Message, long)` 和 `postAtTime(Runnable, long)` 方法，可以设定在未来某个时刻执行任务。
+ Enqueue an action to be performed on a different thread than your own:
  在别的线程执行任务，我们通常的应用就是在子线程处理耗时任务后，我们可以通过 `Handler(getMainLooper())` 来执行更新 UI 的操作。
  
### Handler 和 MessageQueue

观察 Handler 的实现可以发现 `sendMessage(msg)`、`sendMessageDelayed(msg, timeMillis)`、`post(runnable)` 方法最后都是通过调用 `sendMessageAtTime(msg, uptimeMillis)` 实现：

``` java
// android/os/Handler.java
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

``` java
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

可以看到像官方文档描述的那样，每个 Handler 实例都关联着一条线程以及这条线程的消息队列(*Each Handler instance is associated with a single thread and that thread's message queue*)。

#### 消息如何入列

在 `sendMessageAtTime()` 方法最后返回了 `Handler.enqueueMessage()` 的值：

``` java
// android/os/Handler.java
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

``` java
// android/os/MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    synchronized (this) {
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

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

我们分开来看，这里 `msg.target` 即 Message 对应的 Handler 实例, 在 `Handler.enqueueMessage()` 里有赋值。

MessageQueue 并没有队列的数据结构，而是以单链表的形式存储 msg，`mMessages` 为当前链表头部的消息，我们称为 head_msg，当前传入的消息称为 current_msg，

``` java
if (p == null || when == 0 || when < p.when) {
    // New head, wake up the event queue if blocked.
    msg.next = p;
    mMessages = msg;
    needWake = mBlocked;
}
```

`if` 的判断是，如果:

+ 当前队列没有 head_msg
+ current_msg 是需要立即执行
+ current_msg 的执行时间在 head_msg 执行时间之前

则将 head_msg 标记为 current_msg 的下一条消息，current_msg 标记为头部消息，并且唤醒当前阻塞的队列。

``` java
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

``` java
// android/os/Looper.java

/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        if (!loopOnce(me, ident, thresholdOverride)) {
            return;
        }
    }
}

/**
 * Poll and deliver single message, return true if the outer loop should continue.
 */
private static boolean loopOnce(final Looper me, final long ident, final int thresholdOverride) {
    // 1.从消息队列中取出 msg
    Message msg = me.mQueue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        final Printer logging = me.mLogging;
        // 2.处理消息前的 logging，需要 call looper.setMessageLogging()
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " "
                    + msg.callback + ": " + msg.what);
        }
        ...
        try {
            // 3.处理消息
            msg.target.dispatchMessage(msg);
            ...
        } catch (Exception exception) {
            ...
            throw exception;
        } finally {
            ...
        }
        ...

        // 4.处理消息后的 logging
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        msg.recycleUnchecked();

        return true;
}
```

`Looper.loopOnce()` 通过 `MessageQueue.next()` 取出 Message:

``` java
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        // 1.nextPollTimeoutMillis 不为 0 则阻塞
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            // 2.判断是否为同步屏障消息
            if (msg != null && msg.target == null) {
                // 3.是同步屏障消息，遍历队列跳过同步消息，找出异步消息
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            // 4.正常消息处理，判断是否有延时
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有消息，那么循环走到 1 那里会一直阻塞
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // 我们可以 call looper.quit() 来结束
            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            ...
        }

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

`MessageQueue.next()` 的大致流程是这样的：

+ MessageQueue 是一个单链表的数据结构，首先会判断 MessageQueue 的头部是不是同步屏障
  + 如果头部 msg 是同步屏障，那么会遍历链表，跳过同步消息，获取队列里面的异步消息，
    + 如果遍历后没有异步消息，那么此时 msg 为空，会走到注释 5，`nextPollTimeoutMillis` 设置为 -1，然后循环到执行 `nativePollOnce(ptr, nextPollTimeoutMillis)` 开始阻塞队列
  + 如果头部 msg 不是同步屏障，那么会判断 msg 是否为空，不为空的话会正常处理消息，为空则同样走到注释 5
+ 如果有异步消息，那么会和处理同步消息一样，先判断是否有延时，有延时则 `nextPollTimeoutMillis` 会被赋值，没延时则会返回 msg。

`Looper.loop()` 的实现就是在无限循环中不断从消息队列中取出 msg，有消息就交给  `msg.target` 即  handler 处理，没消息则会调用 `nativePollOnce` 阻塞，`nativePollOnce` 底层的 Linux 的 epoll 机制.

Handler 中处理消息像这样 `Handler.dispatchMessage(msg)`:

``` java
// android/os/Handler.java
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

处理 msg 有三种方式，分别对应 handler 的三种使用方式：

+ `msg.callback` 对应的是 Handler.post(runnable) 中的 runnable 对象
+ `mCallback.handleMesage(msg)` 对应的是 `Handler(callback, async)` 构造方法传递的 `callback`
+ `handleMessage(msg)` 对应的是复写 `handleMessage(msg)` 的方法

`msg.callback` 对应的是 `Handler.post(runnable)` 传递的 Runnable 对象，而 `mCallback.handleMessage(msg)` 或 `handleMessage(msg)` 则是实例化 Handler 对象时复写的方法。

### 总结

Handler 帮助我们处理 Message 和 Runnable 对象，每个 Handler 实例都关联着一个线程以及这个线程的 MessageQueue。当我们创建一个新的 Handler 对象时，它将与当前所在的线程(UI/Work Thread)和 MessageQueue 绑定在一起，通过 Handler 发出的 Message 或者 Runnable(msg.callback) 会根据执行时间插入 MessageQueue，Looper 会持续不断的从 MessageQueue 取出可执行的 Message 通过 Handler.handleMessage(msg) 处理。

如下图所示：![handler](/img/handler_looper_messagequeue.jpeg)

### 面试常见问题

+ handler 里的 nativepollonce 为什么不会 anr
  + 对于线程即是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。真正会卡死主线程的操作是在回调方法onCreate/onStart/onResume等操作时间过长，会导致掉帧，甚至发生ANR，looper.loop本身不会导致应用卡死。
  
  主线程的死循环一直运行是不是特别消耗CPU资源呢？ 其实不然，这里就涉及到Linux pipe/epoll机制，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。 Gityuan–Handler(Native层)
+ activity有个内部类handler，描述下引用关系链路，并说明为何gcroots能访问到activity
+ 说下handler的流程，异步消息是什么？Android中哪些场景会发送异步消息？我们在代码中可以手动发异步消息吗
+ Handler的工作流程，源码要记牢，细节要理解透，比如怎么唤醒主线程的，while为啥不会阻塞主线程
+ handler用于线程间通信，怎么保证线程安全
+ handler原理，sendMessageDelayed是怎么实现的，为什么不卡主线程，底层是如何通知进程这边恢复阻塞的
  