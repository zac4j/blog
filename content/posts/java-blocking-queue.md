---
title: "Java Blocking Queue"
date: 2022-06-19T20:38:57+08:00
tags: ["java", "blocking", "queue"]
description: "Intro to Java Blocking Queue"
categories: ["java", "concurrent"]
author: "杨晓峰·geektime"
draft: false
---

ConcurrentLinkedQueue 和 LinkedBlockingQueue 有什么区别

+ Concurrent 类型基于 lock-free，在常见的多线程访问场景，一般可以提供较高吞吐量。
+ LinkedBlockingQueue 内部则是基于锁，并提供了 BlockingQueue 的等待性方法。

## 线程安全队列一览

常见的集合中 LinkedList 是个 Deque，只不过不是线程安全的。

下图展示了 Java 并发类库提供的各种线程安全队列实现：

![tsc](/img/thread-safe-collection.webp)

### Deque

Deque 侧重点是支持**队列头尾**都进行插入和删除，所以提供了特定的方法，如：

+ 尾部插入时需要的 addLast(e)、offerLast(e).
+ 尾部删除所需要的 removeLast()、pollLast().

JUC 包中绝大部分 Queue 都实现了 BlockingQueue 接口。在常规队列操作基础上，Blocking 意味着其提供了特定的等待性操作，获取时（take）等待元素进队，或者插入时（put）等待队列出现空位。

``` java
/**
 * 获取并移除队列头结点
 */
E take() throws InterruptedException;

/**
 * 插入元素，如果队列已满，则等待直到队列出现空闲空间
 */
void put(E e) throws InterruptedException;
```

另一个 BlockingQueue 经常被考察的点，就是是否有界（Bounded、Unbounded），这一点也往往会影响我们在应用开发中的选择。

+ ArrayBlockingQueue 是最典型的有界队列，其内部以 final 的数组保存数据，数组的大小就决定了队列的边界，所以我们在创建 ArrayBlockingQueue 时，都要指定容量，如：

``` java
public ArrayBlockingQueue(int capacity, boolean fair)
```

+ LinkedBlockingQueue 基于有界的逻辑实现，不过其默认初始容量为 `Integer.MAX_VALUE`，成为无界队列。
+ SynchronousQueue 每个删除操作都要等待插入操作，反之每个插入操作也都要等待删除操纵。那么这个队列的容量时多少呢？0.
+ PriorityBlockingQueue 是无边界的优先队列，其大小受系统资源影响。
+ DelayedQueue 和 LinkedTransferQueue 同样是无边界队列。对于无边界的队列，有一个自然的结果，就是 put 操作永远也不会发生其他 BlockingQueue 的那种等待情况。

我们看下 LinkedBlockingQueue 的实现：

``` java
/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Wait queue for waiting takes */
private final Condition notEmpty = takeLock.newCondition();

/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();

/** Wait queue for waiting puts */
private final Condition notFull = putLock.newCondition();
```

我们之前提到过 ArrayBlockingQueue 中的 notEmpty、notFull 都是同一个再入锁的条件变量，而 LinkedBlockingQueue 则改进了锁操作的粒度，头、尾操作使用不同的锁，所以在通用场景下，它的吞吐量相对要更好一些。

下面 take 方法与 ArrayBlockingQueue 中的实现，也是有不同的，由于其内部结构是链表，需要自己维护元素的数量值：

``` java
public E take() throws InterruptedException {
    final E x;
    final int c;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1) {
            notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    if (c == capacity) {
        signalNotFull();
    }
    return x;
}
```

类似 ConcurrentLinkedQueue 等，则是基于 CAS 的无锁技术，不需要在每个操作时使用锁，所以扩展性变现要更加优异。

相对比较另类的 SynchronousQueue，Java6 中，其实现发生了非常大的变化，利用 CAS 替换掉了原本基于锁的逻辑，同步开销比较小。它是 Executors.newCachedThreadPool() 的默认队列。

### 队列使用场景与典型用例

在实际开发中，Queue 被广泛用于生产者-消费者场景，比如利用 BlockingQueue 来实现，由于其提供的等待机制，我们可以少做很多协调性的工作，可以参考如下示例：

``` java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ConsumerProducer {
    private static final String EXIT = "Good bye!";
    public static void main(String[] args) {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);
        Producer producer = new Producer(queue);
        Consumer consumer = new Consumer(queue);
        new Thread(producer).start();
        new Thread(consumer).start();
    }

    static class Producer implements Runnable {
        private BlockingQueue<String> queue;
        public Producer(BlockingQueue<String> q) {
            this.queue = q;
        }

        @Override
        public void run() {
            for (int i = 0; i < 20; i++) {
                try {
                    Thread.sleep(5L);
                    String msg = "Message" + i;
                    System.out.println("Produced new item: " + msg);
                    queue.put(msg);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

            try {
                System.out.println("OK funs over!");
                queue.put(EXIT);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    static class Consumer implements Runnable {
        private BlockingQueue<String> queue;
        public Consumer(BlockingQueue<String> q) {
            this.queue = q;
        }

        @Override
        public void run() {
            try {
                String msg;
                while (!EXIT.equalsIgnoreCase((msg = queue.take()))) {
                    System.out.println("Consumed item: " + msg);
                    Thread.sleep(10L);
                }
                System.out.println("OK got exit msg!");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

上面是一个典型的生产者-消费者模型，如果使用非 Blocking 的队列，那么我们就要自己去实现轮询、条件判断（如检查 poll 返回值是否 null）等逻辑，如果没有特别的场景要求，Blocking 实现起来代码更加简单、直观。

### 各种 Blocking 的使用场景

以 LinkedBlockingQueue、ArrayBlockingQueue 和 SynchronousQueue 为例，我们根据需求可以从多方考量：

+ 从队列边界要求看，ArrayBlockingQueue 有明确的容量限制，而 LinkedBlockingQueue 则取决于我们是否在创建时指定，SynchronousQueue 则不能缓存任何元素。
+ 从空间利用效率看，数据结构的 ArrayBlockingQueue 要比 LinkedBlockingQueue 紧凑，因为不需要创建节点，但是其初始分配阶段就需要一段连续的空间，所以初始内存需求更大。
+ 通用场景中，LinkedBlockingQueue 的吞吐量一般优于 ArrayBlockingQueue，因为它实现了更加细粒度的锁操作。
+ ArrayBlockingQueue 实现比较简单，性能更好预测，属于表现稳定的结构。
+ 如果我们需要实现两个线程之间接力性（handoff）的场景，除了 CountDownLatch，SynchronousQueue 也是完美符合这种场景的，而且线程间协调和数据传输统一起来，代码更加规范。
+ 很多时候 SynchronousQueue 的性能表现，往往大大超过其他实现，尤其是在队列元素较小的场景。

#### Synchronous Handoffs Implementation

``` java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

public class HandoffSample {

    private static ExecutorService executor = Executors.newFixedThreadPool(2);
    private static SynchronousQueue<Integer> queue = new SynchronousQueue<>();

    private static Runnable producer = () -> {
        Integer producedElement = ThreadLocalRandom.current().nextInt();
        try {
            System.out.println("Saving an element: "+ producedElement +" to the exchange point");
            queue.put(producedElement);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    };

    private static Runnable consumer = () -> {
        try {
            Integer consumedElement = queue.take();
            System.out.println("Consumed an element: "+ consumedElement +" from the exchange point");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    };

    public static void main(String[] args) throws InterruptedException {
        executor.execute(producer);
        executor.execute(consumer);

        executor.awaitTermination(500L, TimeUnit.MILLISECONDS);
        executor.shutdown();
        assert(queue.size() == 0);
    }
}
```

打印出的 log:

``` log
Saving an element: 466241464 to the exchange point
Consumed an element: 466241464 from the exchange point
```
