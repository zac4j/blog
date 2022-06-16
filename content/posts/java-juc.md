---
title: "Intro to Java Juc"
date: 2022-06-14T15:24:53+08:00
description: "Intro to classes in java.util.concurrent package"
tags: ["java", "juc"]
categories: ["java"]
author: "杨晓峰·geektime"
draft: true
---

juc 通常指 java.util.concurrent 并发包，这个包集中了 Java 并发的各种基础工具类，具体包括几个方面：

+ 提供了比 synchronized 更高级的各种同步结构，包括 CountDownLatch、CyclicBarrier、Semaphore 等，可以实现更加丰富的多线程操作，比如利用 Semaphore 作为资源控制器，限制同时急性工作的线程数量。
+ 各种线程安全的容器，如 ConcurrentHashMap、有序的 ConcurrentSkipListMap，或者类似快照机制，实现线程安全的动态数组 CopyOnWriteArrayList 等。
+ 各种并发队列的实现，如各种 BlockingQueue 实现，比较典型的 ArrayBlockingQueue、SynchronousQueue 或针对特定场景的 PriorityBlockingQueue 等。
+ Executor 框架，可以创建各种不同类型的线程池，调度任务运行等，绝大部分情况下，不再需要自己从头实现线程池和任务调度器。

<!-- more -->

我们进行多线程编程，无非是达到几个目的：

+ 利用多线程提高程序的扩展能力，以达到业务对吞吐量的要求。
+ 协调线程间调度、交互，以完成业务逻辑
+ 线程间传递数据和状态，这同样是实现业务逻辑的需要。

### Semaphore

Semaphore 提供了 Java 版本的信号量实现，它通过控制一定数量的允许（permit）的方式，来达到限制通用资源访问的目的。可以想象一种场景，我们在机场等出租车排队时，当很多空闲出租车就位时，为防止过度拥挤，调度员指挥排队等待上车的人群一次进来5人上车，等这5人坐车出发，再放进来下一批，这和 Semaphore 的工作原理类似。

``` java
import java.util.concurrent.Semaphore;

public class SemaphoreSample {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(0);
        for (int i = 0; i < 10; i++) {
            Thread t = new Thread(new AbnormalSemaphoreWorker(semaphore));
            t.start();
        }
        System.out.println("Action...Go!");
        semaphore.release(5);
        System.out.println("Wait for permits off");
        while (semaphore.availablePermits() != 0) {
            try {
                Thread.sleep(100L);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        System.out.println("Action...Go again!");
        semaphore.release(5);
    }
}

class AbnormalSemaphoreWorker implements Runnable {
    private String name;
    private Semaphore semaphore;
    public AbnormalSemaphoreWorker(Semaphore semaphore) {
        this.semaphore = semaphore;
    }

    @Override
    public void run() {
        try {
            semaphore.acquire();
            log("Executed!");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void log(String msg) {
        if (name == null) {
            name = Thread.currentThread().getName();
        }
        System.out.println(name + " " + msg);
    }
}
```

上面的代码侧重演示 Semaphore 的功能和局限性，包含了很多多线程编码中的反实践，比如使用了 sleep 来协调任务执行，而且使用轮询调用 avaliablePermits 来检测信号量获取情况，这都是很低效并且脆弱的，通常只在测试或者诊断场景。

Semaphore 的工作逻辑是，首先线程试图获取工作许可，得到许可则执行任务，然后释放许可，这时等待许可的其他线程，就可以获得许可进入工作状态，直到全部处理结束。编译运行，我们就能看到 Semaphore 的允许机制对工作线程的限制。

总的来说，Semaphore 就是个计数器，其基本逻辑基于 acquire/release，并没有太复杂的同步逻辑。

如果 Semaphore 的数值初始化为 1，那么一个线程就可以通过 acquire 进入互斥状态，本质上和互斥锁是非常相似的。但是区别也非常明显，比如互斥锁是持有者，而对于 Semaphore 这种计数器结构，虽然有类似功能，但其实不存在真正意义的持有者，除非对其进行扩展包装。

### CountDownLatch 和 CyclicBarrier

#### 基本区别

+ CountDownLatch 不可以重置，所以无法重用；而 CyclicBarrier 可以重用。
+ CountDownLatch 的基本操作组合是 countDown/await。调用 await 的线程阻塞等待 countDown 足够的次数，不管你是一个线程还是多个线程里 countDown，只要次数足够即可。**CountDownLatch 操作的重点是事件**。
+ CyclicBarrier 的基本操作组合，是 await，当所有的 parties 都调用了 await，才会继续进行任务，并自动进行重置。正常情况下，CyclicBarrier 的重置都是自动发生的，如果调用 reset 方法，但还有线程在等待，就会导致等待线程被打扰，抛出 BrokenBarrierException 异常。**CyclicBarrier 侧重点是线程**，而不是调用事件，它的典型应用场景是用来等待并发线程结束。

继续机场排队等车的场景，假设有 10 个人排队，我们将其分成 5 人一组，通过 CountDownLatch 来协调批次，我们可以这样实现：

``` java
import java.util.concurrent.CountDownLatch;

public class LatchSample {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(6);
        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(new FirstBatchWorker(latch));
            t.start();
        }

        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(new SecondBatchWorker(latch));
            t.start();
        }

        while (latch.getCount() != 1) {
            Thread.sleep(100L);
        }
        System.out.println("Wait for first batch finish");
        latch.countDown();
    }
}

class FirstBatchWorker implements Runnable {
    private CountDownLatch latch;
    public FirstBatchWorker(CountDownLatch latch) {
        this.latch = latch;
    }

    @Override
    public void run() {
        System.out.println("First batch executed!");
        latch.countDown();
    }
}

class SecondBatchWorker implements Runnable {
    private CountDownLatch latch;
    public SecondBatchWorker(CountDownLatch latch) {
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            latch.await();
            System.out.println("Second batch executed!");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

CountDownLatch 的调度方式相对简单，后一批次的线程进行 await，等待前一批 countDown 足够多次。这个例子也从侧面体现了它的局限性，虽然它也能支持10个人排队的情况，但是因为不能重用，如果要支持更多人排队，那么就不能依赖一个 CountDownLatch 进行了。其运行结果如下：

``` console
First batch executed!
First batch executed!
First batch executed!
First batch executed!
First batch executed!
Wait for first batch finish
Second batch executed!
Second batch executed!
Second batch executed!
Second batch executed!
Second batch executed!
```

如果用 CyclicBarrier 来实现：

``` java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierSample {
    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(5, new Runnable() {
            // 屏障执行调用该 action
            @Override
            public void run() {
                System.out.println("Action...Go again");
            }
        });

        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(new CyclicWorker(barrier));
            t.start();
        }
    }

    static class CyclicWorker implements Runnable {
        private CyclicBarrier barrier;
        public CyclicWorker(CyclicBarrier barrier) {
            this.barrier = barrier;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < 2; i++) {
                    System.out.println("Executed!");
                    barrier.await();
                }
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
