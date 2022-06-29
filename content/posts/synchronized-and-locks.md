---
title: "Intro to Java synchronized and Locks"
date: 2022-05-09T19:53:23+08:00
tags: ["lock", "async", "thread"]
description: "Intro to Java synchronized and ReentrantLock"
categories: ["java", "concurrent"]
author: "杨晓峰·geektime"
draft: true
---

锁作为并发工具之一，我们至少需要掌握：

+ 理解什么是线程安全
+ synchronized、ReentrantLock 等机制的基本使用与案例。
+ synchronized、ReentrantLock 底层实现
+ 理解锁膨胀、降级；理解偏斜锁、自旋锁、轻量级锁、重量级锁等概念
+ 掌握 juc 并发包各种不同实现和案例分析。

## 谈谈什么是线程安全

按照《Java 并发编程实战》（Java Concurrency in Practice）书中的定义，线程安全是一个多线程环境下正确性的概念，也就是保证多线程环境下共享的、可修改的状态的正确性。

线程安全需要保证几个基本特性：

+ **原子性**：简单说就是相关操作不会中途被其他线程干扰，一般通过同步机制实现
+ **可见性**：是一个线程修改了某个共享变量，其状态能够立即被其他线程知晓，通常被解释为将线程本地状态反映到主内存上，volatile 就是负责保证可见性的。
+ **有序性**：是保证线程内串行语义，避免指令重排等。

我们先看一段会有线程安全问题的例子：

``` java
public class ThreadIssueSample {

  public int sharedState;
  public void doUnsafeAction() {
    while(sharedState < 10000) {
      int former = sharedState++;
      int latter = sharedState;
      if (former != latter - 1) {
          System.out.println("Observe race condition, former is " + former + ", latter is " + latter);
      }
    }
  }

  public static void main(String[] args) throws InterruptedException {
    ThreadIssueSample sample = new ThreadIssueSample();
    Thread t1 = new Thread() {
        public void run() {
          sample.doUnsafeAction();
        }
    };
    Thread t2 = new Thread() {
        public void run() {
          sample.doUnsafeAction();
        }
    };
    t1.start();
    t2.start();
    t1.join();
    t2.join();
  }
}
```

运行这段代码，打印出的 log 表示出现了线程安全问题：

> Observed race condition, former is 7655, latter is 7650

我们可以将赋值过程用 synchronized 保护起来，使用 this 作为互斥单元，就可以避免线程并发的去修改 sharedState.

``` java
synchronized(this) {
  // ...
  int former = sharedState++;
  int latter = sharedState;
  if (former != latter - 1) {
      System.out.println("Observe race condition, former is " + former + ", latter is " + latter);
  }
  // ...
}
```

使用 javap 反编译，可以看到 Java 利用了 **monitorenter/monitorexit** 实现了同步语义：

``` java
11: astore_1
12: monitorenter
13: aload_0
14: dup
15: getfield    #2                // Field sharedState:I
18: dup_x1
…
56: monitorexit
```

我们再看看 ReentrantLock，什么是再入呢？它表示当一个线程试图获取一个它已经获取的锁时，这个获取动作就自动成功，这个是对锁粒度的一个概念，也就是锁的持有是以线程为单位而不是基于调用次数。Java 锁实现强调再入性是为了和 pthread 的行为进行区分。

我们可以在创建再入锁时设置**公平性（fairness）**

``` java
ReentrantLock fairLock = new ReentrantLock(true);
fairLock.lock();
try {

} finally {
  fairLock.unlock();
}
```

这里的公平性是指，在竞争场景中，当公平性为真时，会倾向于将锁赋予等待时间最久的线程。公平性是减少线程“饥饿”（个别线程长期等待锁，却始终无法获取）情况发生的一个办法。

如果使用 synchronized，则无法进行公平性选择，其永远是不公平的，这也是主流操作系统调度的选择。通用场景中，公平性未必有想象的那么重要，Java 默认的调度策略很少会导致“饥饿“发生。与此同时，若要保证公平性则会引入额外开销，自然会导致一定的吞吐量下降。所以，只有程序确实有公平性需求的时候，才有必要指定它。

ReentrantLock 相比 synchronized 可以进行精细的同步操作，如：

+ 带超时的获取锁尝试
+ 可以判断是否有线程，或者某个特定线程，在排队等待获取锁
+ 可以响应中断请求等
+ ...

Juc 包中还有**条件变量**（java.util.concurrent.Condition），如果说 ReentrantLock 是 synchronized 的替代选择，Condition 则是将 wait、notify、notifyAll 等操作转化为相应的对象，将复杂而晦涩的同步操作转变为直观可控的对象行为。

我们可以通过 ArrayBlockingQueue 源码直观看到 ReentrantLock 和 Condition 的使用。

``` java

/**
 * Condition for waiting takes
 */ 
private final Condition notEmpty;

/**
 * Condition for waiting puts
 */ 
private final Condition notFull;

public ArrayBlockingQueue(int capacity, boolean fair) {
  if(capacity <= 0) {
    throw new IllegalArgumentException();
  }
  this.items = new Object[capacity];
  lock = new ReentrantLock(fair);
  notEmpty = lock.newCondition();
  notFull = lock.newCondition();
}
```

两个条件变量是从同一再入锁创建出来，然后使用在特定的操作中，如下 take 方法，判断和等待条件满足：

``` java
public E take() throws InterruptedException {
  final ReentrantLock lock = this.lock;
  lock.lockInterruptibly();
  try {
    while(count == 0) {
      notEmpty.await();
    }
    return dequeue();
  } finally {
    lock.unlock();
  }
}
```

当队列为空时，试图 take 的线程的正确行为应该是等待入列发生，而不是直接返回，这是 BlockingQueue 的语义，使用条件 notEmpty 就可以优雅的实现这一逻辑。

那么怎么保证入列触发后续 take 操作呢？我们可以看 enqueue 的实现：

``` java
private void enqueue(E e) {
  // assert lock.isHeldByCurrentThread();
  // assert lock.getHoldCount() == 1;
  // assert items[putIndex] == null;
  final Object[] items = this.items;
  items[putIndex] = e;
  if (++putIndex == items.length) putIndex = 0;
  count++;
  // 通知 await 的线程，非空条件已满足
  notEmpty.signal();
}
```

通过 **signal/await** 的组合，完成了条件判断和通知等待线程，非常顺畅就完成了状态流转。

从性能上看，synchronized **早期比较低效**，对比 ReentrantLock，大多数场景性能都相差较大。但在 Java6 中对其进行了非常多的改进，在高竞争情况下，ReentrantLock 仍然有一定优势。

synchronized 和 ReentrantLock 有什么区别？

synchronized 是 Java 内建的同步机制，所以有人称为 Intrinsic Locking，它提供了互斥的语义和可见性，当一个线程已经获取当前锁时，其他试图获取的线程只能等待或者阻塞在那里。

在 Java 5 以前，synchronized 是仅有的同步手段，在代码中 synchronized 可以修饰方法，也可以修饰代码块，本质上 synchronized 方法等同于把方法全部语句用 synchronized 块包起来。

ReentrantLock 通常称为再入锁，Java 5 开始提供的锁实现，它的语义和 synchronized 基本相同。再入锁通过调用 lock() 方法获取，需要注意调用 unlock() 方法释放，不然会一直持有该锁。

Java 对一个对象加锁，锁是什么东西？
什么是锁竞争？

+ 如果多个线程轮流获取一个锁，如果每次获取锁的时候都很顺利，没有发生阻塞，那么就不存在锁竞争。只有当某线程尝试获取锁的时候，发现该锁已经被占用，只能等待其释放，这才发生了锁竞争。

## 悲观锁与乐观锁

悲观锁阻塞事务，乐观锁回滚重试

+ 悲观锁
  + 每次拿数据时都认为别人会修改，所以每次拿数据都会上锁，这样别人想拿数据就会被挡住，直到悲观锁被释放。
+ 乐观锁
  + 每次拿数据都认为别人不会修改，所以不会上锁，但是想要更新数据时，会在更新前检查从读取到更新这段时间数据是否被别人修改过。如果修改过，则重新读取，再次尝试更新，循环上述步骤直到更新成功。
  + 伪代码：
  
  ``` java
  var data = 111;
  var flag = true;

  while(flag) {
    val oldValue = data;
    val newValue = modify(data);
    // CAS = Compare-and-Swap 比较并替换
    if(data == oldValue) {
      data = newValue;
      flag = false;
    } else {
      // do nothing
    }
  }
  ```

因为整个过程并没有加锁-解锁的操作，因此乐观锁策略也被称为无锁编程，它仅仅是一个循环重试CAS算法而已。

自旋锁

所谓自旋，就是一个 for(;;) 无限循环，当然和乐观锁不同哈

synchronized 锁升级：偏向锁 -> 轻量级锁 -> 重量级锁

初次执行 synchronized 代码块时，锁对象变成偏向锁（通过 CAS 修改对象头里的锁标志位），字面意思是“偏向于第一个获得它的线程”的锁。执行同步代码块后，线程并不会主动释放偏向锁。当第二次到达同步代码块时，线程会判断此时持有锁的线程是否为自己（对象头里也含有持有锁的线程 ID），如果是则正常往下执行。由于之前没有释放锁，这里也就不需要重新加锁。如果自始至终使用锁的线程只有一个，很明显偏向锁几乎没有额外开销，性能极高。

一旦有第二个线程加入锁竞争，偏向锁就升级为轻量级锁（自旋锁）。在轻量级锁状态下继续锁竞争，没有抢到锁的线程将自旋，即不停地循环判断锁是否能够被成功获取。获取锁的操作，其实就是通过 CAS 修改对象头里的锁标识位。先比较当前锁标识位是否为“释放”，如果是则将其设置为“锁定”，比较并设置是原子性发生的。这就算抢到锁了，然后线程将当前锁的持有者信息修改为自己。

长时间的自旋操作是非常消耗资源的，一个线程持有锁，其他线程就只能在原地空耗 CPU，执行不了任何有效的任务，这种现象叫忙等（busy-waiting）。如果多个线程用一个锁，但是没有发生锁竞争，或者发生很轻微的锁竞争，那么 synchronized 就用轻量级锁，允许短时间的忙等现象。这是一种折中的做法，短时间的忙等，换取线程在用户态（user mode）和内核态（kernel mode）之间切换的开销。

显然，忙等也是有限度的（有个计数器记录自旋次数，默认允许循环10次）。如果锁竞争情况严重，某个达到最大自旋次数的线程，会将轻量级锁升级为重量级锁（依然是 CAS 修改锁标志位，但不修改持有锁的线程 ID）。当后续线程尝试获取锁时，发现被占用的锁是重量级锁，则直接将自己挂起（而不是忙等），等待将来被唤醒。

可再入锁（递归锁）

可再入锁的字面意思是“可以重新进入的锁”，即允许同一个线程多次获取同一把锁。

Reference：

+ https://zhuanlan.zhihu.com/p/71156910
+ https://zhuanlan.zhihu.com/p/451061367
+ https://xie.infoq.cn/article/d9479ba8900bd7645c035d006
