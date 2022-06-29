---
title: "Intro to Executor"
date: 2022-06-27T21:46:30+08:00
description: "Intro to Executor, Executors, ThreadPool"
tags: ["executor", "thread", "threadpool"]
categories: ["java", "concurrent"]
author: "杨晓峰·geektime"
draft: true
---

## Executor、ExecutorService 和 Executors

我们看看 Executor 框架的基本组成：

![executor](/img/executor.webp)

+ Executor 是一个基础的接口，其目的是将任务提交和任务执行解耦，我们可以看其定义的唯一方法：

``` java
void execute(Runnable cmd);
```

Executor 的设计是源于 Java 早期线程 API 使用的教训，开发者在实现应用逻辑时，被太多线程创建、调度等不相关细节所打扰。就像我们进行 HTTP 通信，如果还需要自己操作 TCP 握手，开发效率低下，质量也难以保证。

+ ExecutorService 则更加完善，不仅提供 service 的管理功能，比如 shutdown 等方法，也提供了更加全面的提交任务机制，如返回 Future 的 submit 方法。

``` java
// 入参是 callback，它解决了 Runnable 无法返回结果的问题。
<T> Future<T> submit(Callable<T> task);
```

+ Java 标准类库提供了几种基础实现，如 ThreadPoolExecutor、ScheduledThreadPoolExecutor、ForkJoinPool。这些线程池的设计特点在于其高度的可调节性和灵活性，以尽量满足复杂多变的实际应用场景。
+ Executors 则从简化使用的角度，提供了各种方便的静态方法。

## Executors 提供的几种线程池配置

+ newCachedThreadPool 它是一种用来处理大量短时间工作任务的线程池，具有几个特点：
  + 它会试图缓存线程并重用，当无缓存线程可用时，就会创建新的工作线程。
  + 如果线程闲置的时间超过 60s 则会被终止并移除缓存；长时间闲置时，这种线程池，不会消耗什么资源
  + 内部使用 SynchronousQueue 作为工作队列。
+ newFixedThreadPool(int nThreads) 用于指定数目（nThreads）的线程，其背后使用的是**无界的工作队列**，任何时候最多有 nThreads 个工作线程是活动的。这意味着，如果任务数量超过了活动线程数目，将在工作队列中等待空闲线程出现；如果有工作线程退出，将会有新的工作线程被创建，以不足指定数目的 nThreads。
+ newSingleThreadExecutor() 它的特点在于工作线程数目被限制为 1，操作一个**无界的工作队列**，所以它保证了所有任务都是被顺序执行，最多会有一个任务处于活动状态，并且不允许使用者改动线程池实例，因此可以避免其改变线程数目。
+ newSingleThreadScheduledExecutor() 和 newScheduledThreadPool(int corePoolSize)，创建的是个 ScheduledExecutorService，可以进行定时或周期性的工作调度，区别于单一工作线程还是多个工作线程。
+ newWorkStealingPool(int parallelism)，Java 8 加入的创建方法，其内部会构建 [ForkJoinPool][fjp]，利用 [Work-Stealing][ws] 算法，并行处理任务，不保证处理顺序。

## ExecutorService

下图描述线程池内部工作过程：

![es](/img/executorservice.webp)

+ 工作队列负责存储用户提交的各个任务，这个工作队列，可以是容量为 0 的 SynchronousQueue（使用 newCachedThreadPool），也可以是固定大小的线程池（newFixedThreadPool）那样使用 LinkedBlockingQueue。

``` java
private final BlockingQueue<Runnable> workQueue;
```

+ 内部的“线程池”，这是指保持工作线程的集合，线程池需要在运行过程中管理线程创建、销毁。例如，对于带缓存的线程池，当任务压力较大时，线程池会创建新的工作线程；当业务压力降低时，线程池会闲置一段时间（默认 60s）后结束线程。

``` java
private final HashSet<Worker> workers = new HashSet<>();
```

线程池的工作线程被抽象为静态内部类 ThreadPoolExecutor.Worker，基于 [AQS][aqs] 实现。

+ ThreadFactory 提供创建线程的逻辑。
+ 如果任务提交时被拒绝，比如线程池已经处于 SHUTDOWN 状态，需要为其提供处理逻辑，Java 提供了 ThreadPoolExecutor.AbortPolicy 等默认实现，可以按照实际需求自定义。

## ThreadPoolExecutor

上面我们了解了线程池的基本组成部分，我们再来看线程池的构建方法的参数：

``` java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

+ corePoolSize 核心线程数，可以理解为线程池中长期驻留的线程数目。对于不同的线程池，这个值的差别会很大，比如 newFixedThreadPool 会将其设置为 nThreads，而对于 newCachedThreadPool 则是 0.
+ maximumPoolSize 就是能创建的最大线程数。对于 newFixedThreadPool 就是 nThreads，而 newSingleThreadExecutor 就是 1.
+ keepAliveTime 和 TimeUnit，这两个参数指定了额外的线程能够闲置多久，显然有些线程池不需要它。
+ workQueue，工作队列，必须是 BlockingQueue。

### 线程池的状态

``` java
// 高位保存线程池状态，低位保存工作线程数
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 工作线程数的理论上限
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

ThreadPool 状态流转图：

![tps](/img/threadpoolstate.webp)

我们再来看看 execute 方法：

``` java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    
    int c = ctl.get();
    // 检查工作线程数目，低于 corePoolSize 则添加 worker
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // isRunning 检查线程池是否被 shutdown
    // 工作队列可能是有界的，offer 是比较友好的入列方式
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次进行防御性检查
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 尝试添加一个 worker，如果失败意味着已经饱和或者被 shutdown 了
    else if (!addWorker(command, false))
        reject(command);
}
```

### 线程池实践

线程池实际使用过程中

+ 避免任务堆积。我们前面说过 newFixedThreadPool 是创建指定数目的线程，但是其工作队列是无界的，如果工作线程数目太少，导致处理跟不上入列的速度，就很有可能占用大量系统内存，甚至出现 OOM。诊断时可以使用 jmap 诊断，查看是否有大量的任务对象入列。
+ 避免过度扩展线程。我们通常在处理大量短时任务时，使用缓存的线程池，比如在 HTTP/2 client API 中，默认实现就是如此。我们在创建线程池时，并不能准确预计任务压力有多大、数据特征是什么样子，所以很难明确设定一个线程数目。
+ 随着线程数目不断增长，我们需要警惕线程泄露的可能性，这种情况往往是因为任务逻辑有问题，导致工作线程迟迟不能被释放。
+ 避免死锁等同步问题，避免在使用线程池时操作 ThreadLocal（ThreadLocalMap.Entry 回收依赖显式的触发，否则就要等待线程结束）.

### 线程池大小的选择策略

我们之前介绍过，线程池大小不合适，太大或太小，都会导致麻烦，所以我们需要考虑一个合适的线程池大小。

+ 如果我们的任务主要是进行计算，那么就意味着 CPU 的处理能力是稀缺的资源，我们能够通过大量增加线程数提高计算能力吗？往往是不能的，如果线程太多，反而可能导致大量的上下文切换开销。所以，这种情况下，通常建议按照 CPU 核心数 N 或者 N+1.
+ 如果是需要较多等待的任务，例如 I/O 操作比较多，可以参考 Brain Goetz 推荐的计算方法：

``` log
线程数 = CPU 核心数 x 目标 CPU 利用率 x（1 + 平均等待时间/平均工作时间）
```

这些时间并不能准确预计，往往需要根据采样或者概要分析等方式进行计算，然后在实际中验证和调整。

+ 除了 CPU 资源的限制，我们可能受别的系统资源限制，比如文件描述符(Unix)、文件句柄(Windows)、内存等。
  > A file descriptor(Unit, Linux) or a file handle (Windows) is the connection id (generally to a file) from OS in order to perform IO oprations(Input/Output of Bytes).

另外在实际工作中，不能把解决问题的思路全部指望到调整线程池上，很多时候架构上的改变更方便解决问题，比如利用[Reactive Stream][rs]、合理的拆分等。

[fjp]:https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/ForkJoinPool.html
[ws]:https://en.wikipedia.org/wiki/Work_stealing
[aqs]:https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/locks/AbstractQueuedSynchronizer.html
[rs]:http://www.reactive-streams.org/
