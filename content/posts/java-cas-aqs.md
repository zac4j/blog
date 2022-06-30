---
title: "Java CAS and AQS"
date: 2022-06-29T21:10:50+08:00
tags: ["java", "cas", "aqs"]
description: "Intro to CAS & AQS in Java"
categories: ["java", "concurrent"]
author: "杨晓峰·geektime"
draft: false
---

## AtomicInteger 的底层原理

AtomicInteger 是对 int 类型的一种封装，提供原子性的访问和更新操作，其原子性操作的实现是基于 CAS（[compare-and-swap][cas]）技术。

所谓 CAS，表现的是一系列操作的集合，获取当前数值，进行一系列运算，利用 CAS 指令试图进行更新。如果当前数值未变，代表没有其他线程进行并发修改，则更新成功。如果当前数值变了，可能出现不同的选择，要么进行重试，要么就返回一个成功或失败的结果。

从 AtomicInteger 的内部属性来看，它依赖于 Unsafe 提供的一些底层能力，进行底层操作；以 volatile 的 value 字段，记录数值，以保证可见性。

``` java
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
private volatile int value;
```

具体的原子操作细节，可以参考任意一个原子更新方法，比如下面的 getAndIncrement.

Unsafe 会利用 value 字段的内存地址偏移，直接完成操作。

``` java
public final int getAndIncrement() {
    return U.getAndInt(this, VALUE, 1);
}
```

因为 getAndIncrement 需要返回数值，所以需要添加失败重试逻辑。

``` java

public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
```

而类似 compareAndSet 这种返回 boolean 类型的函数，因为其返回值表现的就是是否成功，所以不需要重试:

``` java
public final boolean compareAndSet(int expectedValue, int newValue);
```

CAS 是 Java 并发中所谓的 lock-free 机制的基础。

## CAS 的底层实现

CAS 的底层实现依赖于 CPU 提供的特定指令，不同的 CPU 体系结构还存在明显区别，比如，x86 CPU 提供 cmpxchg 指令；而在精简指令集的体系架构中，则通常是靠一对指令（如 ”load and reserve“ 和 ”store conditional“）实现的，在大多数 CPU 上 CAS 都是非常轻量级的操作，这也是其优势所在。

<!-- The author said: 大部分情况下，掌握到这个程度也就够用了，我认为没有必要让每个 Java 工程师都去了解到指令级别，我们进行抽象、分工就是为了让不同层面的开发者在开发中，可以尽量屏蔽不相关的细节。 -->

### CAS 的应用

考虑这样一个场景，在数据库产品中，为保证索引的一致性，一个常见的选择是，保证只有一个线程能够排他性地修改一个索引分区，如何在数据库抽象层面实现呢？

我们可以考虑为索引分区对象添加一个逻辑上的锁，例如，以当前独占的线程 ID 作为锁的数值，然后通过原子操作设置 lock 数值，来实现加锁和释放锁，伪代码如下：

``` java
public class AtomicBTreePartition {
    private volatile long lock;
    public void acquireLock();
    public void releaseLock();
}
```

那么在 Java 代码中，我们怎么实现锁操作？Cassandra 之前使用 Unsafe.monitorEnter()/monitorExit()，但 Java 9 移除了这对方法，导致其无法平滑升级 JDK 版本。目前 Java 提供了两种公共 API，可以实现 CAS 操作，比如 **AtomicLongFieldUpdater**, 它是基于反射机制创建，我们需要保证类型和字段名称正确。

``` java
private static final AtomicLongFieldUpdater<AtomicBTreePartition> UPDATER = AtomicLongFieldUpdater.newUpdater(AtomicBTreePartition.class, "lock");

private void acquireLock() {
    long t = Thread.currentThread().getId();
    while(!UPDATER.compareAndSet(this, 0L, t)){
        // 数据库操作可能比较慢
        ...
    }
}
```

[Atomic 包][ac]提供了最常用的原子性数据类型，甚至是引用、数组等相关原子类型和更新操作工具，是很多线程安全程序的首选。

AtomicLongFieldUpdater 创建更加紧凑的计数器实现，以替代 AtomicLong. 优化永远是针对特定需求、特定目的，例如如果是针对一两个对象，那么用 AtomicLong 即可，如果是成千上万甚至更多的对象，就要考虑紧凑性的影响了。而 atomic 包提供的 LongAdder，在高度竞争环境下，可能是比 AtomicLong 更佳的选择，尽管它的本质是空间换时间。

除了 AtomicLongFieldUpdater，另一个 CAS 实现是 Java 9 以后提供的 Handle API，源自 [JEP 193][jep193]，提供了各种粒度的原子或有序性的操作。我们将前面的例子修改为：

``` java
private static final VarHandle HANDLE = MethodHandles.lookup().findStaticVarHandle(AtomicBTreePartition.class, "lock");

private void acquireLock() {
    long t = Thread.currentThread().getId();
    while(!HANDLE.compareAndSet(this, 0L, t)) {
        // 数据库操作可能比较慢
        ...
    }
}
```

一般来说，我们进行类型 CAS 操作，推荐使用 Variable Handle API 去实现，其提供了精细粒度的公共底层 API。和内部的 Unsafe API 不同，公共 API 不会发生不可预测的修改，这一点提供了对于未来产品升级和维护的基础保障。以 Cassandra 升级为例，很多额外的工作量，都是源自其使用了 Hack 而非 Solution 的方式解决问题。

### CAS 的缺陷

CAS 并非没有副作用，其常用的失败重试机制，隐含一个假设，即竞争情况是短暂的。大多数应用场景中，确实大部分重试只发生一次就获得成功，但总有意外的情况，所以在有需要的时候，还是要考虑限制自旋的次数，避免过度消耗 CPU。

另一个就是 [ABA][aba] 问题，只在 lock-free 算法下暴露的问题。我前面说过 CAS 是在更新时比较前值，乳沟只是恰好相同，例如期间发生了 A -> B -> A 的更新，仅仅判断数值时 A，可能导致不合理的修改操作。针对这种情况，Java 提供了 [AtomicStampedReference][asr] 工具类，通过为引用建立版本号（stamp）的方式，来保证 CAS 的正确性。

## AQS

AbstractQueuedSynchronizer（AQS）是 Java 并发包中，实现各种同步结构和部分其他组成单元（如 ThreadPoolExecutor.Worker）的基础。

Doug Lea 曾经介绍过 AQS 的设计初衷。从原理上，一种同步结构往往是可以利用其他结构实现的，例如使用 Semaphore 实现互斥锁。不过，对某种同步结构的倾向，会导致复杂、晦涩的实现逻辑，所以，他选择将基础的同步相关操作抽象在 AbstractQueuedSynchronizer 中，利用 AQS 为我们构建同步结构提供了范本。

AQS 内部数据和方法，可以简单拆分为：

+ 一个 volatile 的整数成员状态，同时提供了 setState 和 getState 方法
+ 一个先入先出（FIFO）的等待线程队列，以实现多线程间竞争和等待，这是 AQS 机制的核心之一。
+ 各种基于 CAS 的基础操作方法，以及各种期望具体同步结构去实现的 acquire/release 方法。

利用 AQS 实现一个同步结构，至少要实现两个基本类型的方法，分别是 acquire 操作，获取资源的独占权；还有就是 release 操作，释放都某个资源的独占。

以 ReentrantLock 为例，它内部通过扩展 AQS 实现了 Sync 类型，以 AQS 的 state 来反映锁的持有情况。

``` java
private final Sync sync;
abstract static class Sync extends AbstractQueuedSynchronizer {

}
```

下面是 ReentrantLock 对应的 acquire 和 release 操作：

``` java
public void lock() {
    sync.acquire(1);
}

public void unlock() {
    sync.release(1);
}
```

整体分析 acquire 方法，其直接实现是在 AQS 内部，调用了 tryAcquire 和 acquireQueued，这是两个需要搞清楚的基本部分：

``` java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

在 ReentrantLock 中，tryAcquire 逻辑实现在 NonfairSync 和 FairSync 中，分别提供了进一步的非公平或公平性方法。在 AQS 内部 tryAcquire 仅仅是个未完全实现的方法（直接抛异常），这个是留给 AQS 实现者自定义的操作。

公平性在 ReentrantLock 构建时指定：

``` java
public ReentrantLock() {
    // 默认是非公平锁
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

以非公平版的 tryAcquire 为例，其内部实现了如何配合 state 与 CAS 获取锁，对比公平版的 tryAcquire，非公平版的在锁无人占有时，并不检查是否有其他等待者，这里体现了非公平的语义。

``` java
final boolean nonfairTryAcquire(int acquires) {
    final Thread thread = Thread.currentThread();
    // 获取当前 AQS 内部状态量
    int c = getState();
    // 0 表示无人占有，则直接用 CAS 修改状态位
    if (c == 0) {
        // 不检查排队情况，直接争抢
        if (compareAndSetState(0, acquires)) {
            // 设置当前线程独占锁
            setExclusiveOwnerThread(current);
            return true;
        }
        // 即使状态不为 0，也可能当前线程是锁持有者，因为这是再入锁
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        // overflow
        if (nextc < 0) {
            throw new Error("Maximum lock count exceeded");
        }
        setState(nextc);
        return true;
    }
    return false;
}
```

我们再来看 acquireQueued，如果前面的 tryAcquire 失败，代表着锁争抢失败，进入排队竞争阶段。这里利用 FIFO 队列，实现线程间对锁的竞争的部分，是 AQS 的核心逻辑。

当前线程会被包装成一个排他模式的节点（EXCLUSIVE），通过 addWaiter 方法添加到队列中。acquireQueued 的逻辑，简要来说，就是如果当前节点的前面是头节点，则试图获取锁，一切顺利则成为新的头节点；否则，有必要则等待：

``` java
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            // 获取前一个节点
            final Node p = node.predecessor();
            // 如果前一个节点是头节点，表示当前节点适合去 tryAcquire
            if (p == head && tryAcquire(arg)) {
                // acquire 成功，则设置新的头节点
                setHead(node);
                // 将前面节点对当前节点的引用清空
                p.next = null;
                return interrupted;
            }
            // 检查是否失败后需要 park
            if (shouldParkAfterFailedAcquire(p, node)) {
                interrupted |= parkAndCheckInterrupt();
            }
        }
    } catch (Throwable t) {
        // 出现异常，取消
        cancelAcquire(node);
        if (interrupted) {
            selfInterrupt();
        }
        throw t;
    }
}
```

到这里基本展现线程试图获取锁的过程，tryAcquire 是按照特定场景需要开发者去实现的部分，而线程间竞争则是 AQS 通过 Waiter 队列与 acquireQueued 提供的，在 release 方法中，同样会对队列进行对应操作。

[cas]:https://en.wikipedia.org/wiki/Compare-and-swap
[aqs]:https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/locks/AbstractQueuedSynchronizer.html
[aba]:https://en.wikipedia.org/wiki/ABA_problem
[asr]:https://jenkov.com/tutorials/java-util-concurrent/atomicstampedreference.html
[ac]:https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/atomic/package-summary.html
[jep193]:https://openjdk.org/jeps/193
