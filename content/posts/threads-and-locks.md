---
title: "Process, Threads and Locks"
date: 2020-07-27
description: "Process, Threads and locks"
tags: ["thread", "process", "lock"]
categories: ["java", "concurrent"]
draft: false
---

Threads are created and managed by the classes Thread and ThreadGroup. Creating a Thread object creates a thread, and that is the only way to create a thread. When the thread is created, it is not yet active; it begins to run when its start method is called.

<!--more-->

## 8.13 Locks and Synchronization

There is a lock associated with every object. The Java programming language does not provide a way to perform separate lock and unlock operations; instead, they are implicitly performed by high-level constructs that always arrange to pair such operations correctly. (The Java virtual machine, however, provides separate monitorenter and monitorexit instructions that implement the lock and unlock operations.)

每个对象都有一个锁。Java 没有单独提供锁定和解锁的方法；相反它们由始终安排准确配对操作的高阶构造隐式执行。Jvm 提供了单独的 *monitorenter* 和 *monitorexit* 指令，用于实现锁定和解锁操作。

The synchronized statement computes a reference to an object; it then attempts to perform a lock operation on that object and does not proceed further until the lock operation has successfully completed. (A lock operation may be delayed because the rules about locks can prevent the main memory from participating until some other thread is ready to perform one or more unlock operations.) After the lock operation has been performed, the body of the synchronized statement is executed. Normally, a compiler for the Java programming language ensures that the lock operation implemented by a monitorenter instruction executed prior to the execution of the body of the synchronized statement is matched by an unlock operation implemented by a monitorexit instruction whenever the synchronized statement completes, whether completion is normal or abrupt.

同步语句计算对象的引用，它尝试对该对象执行锁定操作，并且在锁定操作完成前不会进一步的操作。（锁定操作可能延迟，因为有关锁定的规则可能阻止主内存参与，直到线程准备一个或多个解锁操作为止。）指定锁定操作后，将执行同步语句的主体。通常，Java 编译器可以确保，由 *monitorenter* 指令（在执行同步语句的主体之前执行）实现的锁定操作和由 *monitorexit* 指令实现的解锁操作相匹配，而不管同步语句是否执行完成或打断。

A synchronized method automatically performs a lock operation when it is invoked; its body is not executed until the lock operation has successfully completed. If the method is an instance method, it locks the lock associated with the instance for which it was invoked (that is, the object that will be known as this during execution of the method's body). If the method is static, it locks the lock associated with the Class object that represents the class in which the method is defined. If execution of the method's body is ever completed, either normally or abruptly, an unlock operation is automatically performed on that same lock.

同步方法在被调用时会自动执行锁定操作。在锁定操作完成前不会执行方法主体。如果该方法是实例方法，则它将与调用该方法的实例锁定。如果该方法是静态的，则它将与定义该方法的 Class 对象相关联的类锁定。如果方法主体正常执行或突然中断，则与该锁配对解锁操作将自动执行。

Best practice is that if a variable is ever to be assigned by one thread and used or assigned by another, then all accesses to that variable should be enclosed in synchronized methods or synchronized statements.

最佳实践是对于某个变量由一个线程 *assign* 并由另一个线程 *use* 或 *assign*，则对该变量的所有访问都应该放在 *synchronized* 方法或 *synchronized* 语句中。

Although a compiler for the Java programming language normally guarantees structured use of locks (see Section 7.14, "Synchronization"), there is no assurance that all code submitted to the Java virtual machine will obey this property. Implementations of the Java virtual machine are permitted but not required to enforce both of the following two rules guaranteeing structured locking.

Let T be a thread and L be a lock. Then:

The number of lock operations performed by T on L during a method invocation must equal the number of unlock operations performed by T on L during the method invocation whether the method invocation completes normally or abruptly.

At no point during a method invocation may the number of unlock operations performed by T on L since the method invocation exceed the number of lock operations performed by T on L since the method invocation.
In less formal terms, during a method invocation every unlock operation on L must match some preceding lock operation on L.
Note that the locking and unlocking automatically performed by the Java virtual machine when invoking a synchronized method are considered to occur during the calling method's invocation.

## 7.14 [Synchronization][synchronized]

The Java virtual machine provides explicit support for synchronization through its monitorenter and monitorexit instructions. For code written in the Java programming language, however, perhaps the most common form of synchronization is the synchronized method.

Jvm 通过 *monitorenter* 和 *monitorexit* 指令为同步操作提供支持。但对于 Java 编写的代码，最常见的同步形式可能是 *synchronized* 方法。

A synchronized method is not normally implemented using monitorenter and monitorexit. Rather, it is simply distinguished in the runtime constant pool by the ACC_SYNCHRONIZED flag, which is checked by the method invocation instructions. When invoking a method for which ACC_SYNCHRONIZED is set, the current thread acquires a monitor, invokes the method itself, and releases the monitor whether the method invocation completes normally or abruptly. During the time the executing thread owns the monitor, no other thread may acquire it. If an exception is thrown during invocation of the synchronized method and the synchronized method does not handle the exception, the monitor for the method is automatically released before the exception is rethrown out of the synchronized method.

Jvm 通常不使用 *monitorenter* 和 *monitorexit* 实现同步方法，而是在运行时常量池通过 *ACC_SYNCHRONIZED* 标记加以区分，该标记由方法调用指令校验。当调用标记为 *ACC_SYNCHRONIZED* 的方法时，当前线程获取 monitor，调用这个方法自身，并释放 monitor，不管方法调用是正常完成或突然中断。

正在执行的线程拥有 monitor 时，别的线程无法获取它。如果在调用同步方法时抛出异常，且同步方法无法处理该异常，则在异常从同步方法抛出执行，自动释放该方法的 monitor.

The *monitorenter* and *monitorexit* instructions exist to support synchronized statements. For example:

``` java
void onlyMe(Foo f) {
    synchronized(f) {
        doSomething();
    }
}
```

is compiled to

``` assembly
Method void onlyMe(Foo)
   0 	aload_1				// Push f	
   1 	astore_2			// Store it in local variable 2
   2 	aload_2				// Push local variable 2 (f)
   3 	monitorenter		// Enter the monitor associated with f
   4 	aload_0				// Holding the monitor, pass this and...
   5 	invokevirtual #5 	// ...call Example.doSomething()V
   8	aload_2				// Push local variable 2 (f)
   9	monitorexit			// Exit the monitor associated with f
  10	return				// Return normally
  11 	aload_2				// In case of any throw, end up here
  12 	monitorexit			// Be sure to exit monitor...
  13 	athrow				// ...then rethrow the value to the invoker
Exception table:
   	From	To 	Target 		Type
    4     	8   11   		any
```

## Volatile

Entry Level:

The rules for volatile variables effectively require that main memory be touched exactly once for each use or assign of a volatile variable by a thread, and that main memory be touched in exactly the order dictated by the thread execution semantics. However, such memory operations are not ordered with respect to read and write operations on nonvolatile variables.

High Level:

+ volatile 有2个作用：
  + 可以保证在多线程环境下共享变量的可见性
  + 通过增加内存屏障防止多个指令之间的重排序
+ 原理：
  + 可见性原理
  + 我理解的可见性是指当一个线程对于共享变量的修改，其他线程可以立即看到修改后的一个值，其实可见性本质上是由几个方面来造成的：
    + 1.CPU 层面的高速缓存，CPU 设计三级缓存来解决 CPU 运算效率和内存 IO 效率不同步的问题，但是它也带来的就是缓存一致性的问题，而在多线程并行执行的情况下，缓存一致性问题就会导致可见性问题，所以对于增加了 volatile 关键字修饰的共享变量，JVM 虚拟机会自动增加 #Lock 汇编指令，那么这个指令会根据不同的 CPU 型号，去自动添加 CPU 总线锁，或者缓存锁
      + 总线锁：锁定 CPU 的前端总线，从而保证在同一时刻，只能有一个线程和内存通信，这样就避免了多线程并发造成的可见性问题
      + 缓存锁：缓存锁是对总线锁的优化，因为总线锁导致 CPU 的使用效率大幅度下降，所以缓存锁只针对 CPU 的三级缓存中的目标数据去加锁，而缓存锁是使用 MESI 缓存一致性协议来实现的
  + 指令的编写顺序和执行顺序是不一致的，从而在多线程环境下导致可见性问题，指令重排序本质是一种性能优化的手段，它来自于几个方面：
    + CPU 层面，针对于 MESI 协议的更进一步的优化，去提升 CPU 的利用率，它引入了一种叫 StoreBuffer 机制，而这种优化机制会导致 CPU 的乱序执行，那么为了避免这种问题，CPU 提供了内存屏障指令，上层应用可以在合适的地方去插入内存屏障，去避免 CPU 指令重排序的问题
    + 编译器层面的优化，编译器在编译过程中，在不改变单线程语义和程序正确性的前提下，对指令进行合理的重排序，从而去优化整体的特性
  + 所以对于共享变量增加了 volatile 关键字，那么编译器层面就不会去触发编译器优化，同时在 JVM 层面，它会插入内存屏障指令，来避免指令重排序的问题
  + 当然除了使用 volatile 关键字以外，从 JDK 5 开始，JMM 就使用了一种 Happens-Before 的模型去描述多线程之间的可见性的一个关系，也就是说如果两个操作之间具备 Happens-Before 的关系，那么意味着这两个操作具备可见性的一个关系，不需要在额外去考虑增加 volatile 关键字来提供可见性的保障

## Thread Pool

Entry level:

Most of the executor implementations in *java.util.concurrent* use thread pools, which consist of *worker threads*. This kind of thread exists separately from the *Runnable* and *Callable* tasks it executes and is often used to execute multiple tasks.

*java.util.concurrent* 包中大部分 executor 实现都使用线程池，这些线程池由 *work threads* 组成。这些线程与它执行的 *Runnable* 和 *Callable* 任务分开，通常用于执行多个任务。

Using *worker threads* minimizes the overhead due to thread creation. Thread objects use a significant amount of memory, and in a large-scale application, allocating and deallocating many thread objects creates a significant memory management overhead.

使用 *worker threads* 可以最大限度减少线程创建所带来的开销。线程对象占用大量内存，并且在大规模 App 中，分配和取消分配许多线程会产生大量内存管理的开销。

One common type of thread pool is the *fixed thread pool*. This type of pool always has a specified number of threads running; if a thread is somehow terminated while it is still in use, it is automatically replaced with a new thread. Tasks are submitted to the pool via an *internal queue*, which holds extra tasks whenever there are more active tasks than threads.

线程池的一种常见类型是 *fixed thread pool*。这种类型的线程池始终有指定数量的线程在运行；如果一个线程在使用时被某种方式突然终止，则线程池会自动创建新的线程替换终止的线程。任务通过内部队列提交到线程池中，*该内部队列在活动任务多于线程数时容纳额外的任务。*

An important advantage of the fixed thread pool is that applications using it degrade gracefully. To understand this, consider a web server application where each HTTP request is handled by a separate thread. If the application simply creates a new thread for every new HTTP request, and the system receives more requests than it can handle immediately, the application will suddenly stop responding to all requests when the overhead of all those threads exceed the capacity of the system. With a limit on the number of the threads that can be created, the application will not be servicing HTTP requests as quickly as they come in, but it will be servicing them as quickly as the system can sustain.

固定线程池的一个重要优势是使用该线程池的 App 可以正常降级。考虑一个 Web 服务应用，每个 HTTP 请求均由单独的线程处理。如果该应用仅简单针对每个 HTTP 请求创建新的线程，
并且系统收到的请求超出了其立即处理的数量，当这些线程的所有开销超出系统的容量时，应用会突然停止响应所有请求。由于可以创建的线程数量受到限制，因此应用可以不尽快的处理 HTTP 请求，但
可以根据系统能力尽快服务这些请求。

A simple way to create an executor that uses a fixed thread pool is to invoke the [newFixedThreadPool][nft] factory method in *java.util.concurrent.Executors* This class also provides the following factory methods:

调用 *java.util.concurrent.Executors* 中的 newFixedThreadPool 工厂方法可以创建固定线程池的 executor。此类还提供如下工厂方法：

+ The [newCachedThreadPool][ncp] method creates an executor with an expandable thread pool. This executor is suitable for applications that launch many short-lived tasks.
  
  *newCachedThreadPool* 使用可扩展的线程池创建 executor。此类线程池适用于启动许多短期任务的应用程序。

+ The [newSingleThreadExecutor][nst] method creates an executor that executes a single task at a time.
  
  *newSingleThreadExecutor* 创建的 executor 每次只执行一个任务。

+ Several factory methods are *ScheduledExecutorService* versions of the above executors.
  
  上述 executor 的 *ScheduledExecutorService* 版本有几种工厂方法。

If none of the executors provided by the above factory methods meet your needs, constructing instances of *java.util.concurrent.ThreadPoolExecutor* or *java.util.concurrent.ScheduledThreadPoolExecutor* will give you additional options.

除了上面这些创建 executor 的方法，*java.util.concurrent.ThreadPoolExecutor* 和 *java.util.concurrent.ScheduledThreadPoolExecutor* 也会提供额外的方法。

High level:

### 如何获取线程池中线程执行完成的状态

+ 从线程池的内部获取
  + 当我们把任务交给线程池处理的时候，线程池会调度工作线程来执行这个任务的 run 方法，当 run 方法正常结束以后，也意味着这个任务完成了，所以线程池中的工作线程是通过同步调用任务的 run 方法，并且等待任务的 run 方法返回后，再去统计任务的完成数量
+ 从线程池外部获取
  + 线程池提供了一个 `isTerminated()` 方法，可以判断线程池的运行状态，一旦 `isTerminated()` 方法返回的状态是 `TERMINATED` 意味着线程池中的所有任务都已经执行完成了，但是这个方法使用的前提，是程序中需要主动调用线程池的 `shutdown()` 方法，在实际业务中，一般不会去主动关闭线程池，因此这个方法在实用性和灵活性都不是很好
  + 线程池中有一个 `submit()` 方法，它有一个 Future 的返回值，我们可以通过 `Future.get()` 方法，去获得任务的执行结果，当线程池中的任务没有执行完成之前，`Future.get()` 方法会一直阻塞，直到任务执行结束，因此，只要 `Future.get()` 方法正常返回，就意味着传入线程池中的任务已经执行完成。
  + 引入 CountDownLatch 计数器，它可以通过初始化指定的一个计数器，去进行倒计时，它提供了2个方法，await() 阻塞线程 和 countDown() 倒计时，我们可以通过组合使用来获取线程执行状态
+ 总结：想要知道线程是否执行结束，我们必须要获取线程执行结束后的状态，由于线程执行是没有返回值的，所以只能通过阻塞-唤醒的方式来实现，`Future.get()` 和 `CountDownLatch` 都是这样的原理

### 线程池拒绝策略怎么自定义

| 任务拒绝策略        | Description                                          |
|---------------------|------------------------------------------------------|
| DiscardPolicy       | 直接丢弃任务                                         |
| CallerRunsPolicy    | 使用调用者线程直接执行被拒绝的任务                   |
| AbortPolicy         | 默认的拒绝策略，抛出 RejectedExecutionException 异常 |
| DiscardOldestPolicy | 丢弃处于任务队列头部的任务，添加被拒绝的任务         |

## Processes and Threads

In concurrent programming, there are two basic units of execution: *processes* and *threads*. In the Java programming language, concurrent programming is mostly concerned with threads. However, processes are also important.

在并发编程中，有两个基本的执行单元：进程和线程。Java 并发编程主要和线程有关，不过进程也很重要。

A computer system normally has many active processes and threads. This is true even in systems that only have a single execution core, and thus only have one thread actually executing at any given moment. Processing time for a single core is shared among processes and threads through an OS feature called *time slicing*.

计算机系统通常有很多活跃的进程和线程。这在只有一个执行核心以至于任何时刻都只有一个线程实际执行的系统也是如此。通过称为 *时间分片* 的 OS 功能，进程和线程可以共享单个内核的处理时间。

It's becoming more and more common for computer systems to have multiple processors or processors with multiple execution cores. This greatly enhances a system's capacity for concurrent execution of processes and threads — but concurrency is possible even on simple systems, without multiple processors or execution cores.

具有多个处理器或多个执行核心的处理器的计算机系统正在变得越来越普遍。这极大增强了并发执行多进程和多线程的能力——即使是在没有多核处理器的简单系统上，并发也是可能的。

### Processes

A process has a self-contained execution environment. A process generally has a complete, private set of basic run-time resources; in particular, each process has its own memory space.

进程具有独立的执行环境。进程通常具有一组完整的、私有的基本运行时资源，每个进程有自己的内存空间。

Processes are often seen as synonymous with programs or applications. However, what the user sees as a single application may in fact be a set of cooperating processes. To facilitate communication between processes, most operating systems support *Inter Process Communication* (IPC) resources, such as pipes and sockets. IPC is used not just for communication between processes on the same system, but processes on different systems.

进程通常被视为程序或应用的代名词。但实际上用户看见的单个应用可能一组协作进程。为了促进进程间的通信，大多数操作系统都支持 *进程间通信*（IPC）资源，比如管道（pipes）和sockets. IPC 不仅可以用在同一系统的进程间通信，还可用于不同系统上的进程。

Most implementations of the Java virtual machine run as a single process. A Java application can create additional processes using a [ProcessBuilder][pb] object. Multiprocess applications are beyond the scope of this lesson.

大多数的 Jvm 实现都是以单个进程运行的。Java 应用可以使用 [ProcessBuilder][pb] 创建新的进程，不过多进程应用不在这里的讨论范围。

### Threads

从操作系统角度，线程是系统任务调度的最小单元，一个进程可以包含多个线程，作为任务的真正运作者，有自己的栈（Stack）、寄存器（Register）、本地存储（Thread Local）等，但是会和进程内其他线程共享文件描述符、虚拟地址空间等。

在具体实现中，线程还分为内核线程、用户线程，Java 的线程实现其实是与虚拟机相关的。对于 Oracle JDK，其线程也经历了一个演进的过程，基本上在 Java 1.2 之后，JDK 已经抛弃了早期的[Green Thread][gt]，也就是用户调度的线程，现在的模型是一对一映射到操作系统内核线程。

通过 Thread 的源码，我们可以发现其操作逻辑大部分是以 JNI 形式调用的本地代码：

``` java

private native void start0();
private native void setPriority0(int newPriority);
private native void interrupt0();
```

这种实现有利有弊，总体上来说，Java 语言得益于精细粒度的线程和相关的并发操作，其构建高扩展性的大型应用的能力已经毋庸置疑。但是，其复杂性也提高了并发编程的门槛，近几年 Go 语言等提供了协程（coroutine），大大提高了构建并发应用的效率。于此同时，Java 也在 Loom 项目中，孕育新的轻量级用户线程（Fiber）等机制。

我们以线程最基本使用的例子开始：

``` java
Runnable task = () -> {System.out.print("new created task");};
Thread t1 = new Thread(task);
t1.start();
t1.join();
```

使用 Runnable 的好处是，不会受 Java 不支持多继承的限制，重用代码实现，当我们需要重复执行相应逻辑时优点明显。而且可以方便的和 Executor 之类的框架结合使用，比如上面例子的逻辑可以完全写成下面的结构：

``` java
Future future = Executors.newsingleThreadExecutor()
  .submit(task);
  .get();
```

这样我们就不用手动管理线程的创建和结束，也能利用 Future 等机制更好地处理执行结果。线程生命周期通常和业务之间没有本质联系，混淆实现需求和业务需求，就会降低开发的效率。

从线程生命周期的状态开始展开，那么在 Java 编程中，有哪些因素可能影响线程的状态呢？主要有：

+ 线程自身的方法，除了 start，还有多个 join 方法，等待线程结束；yield 是告诉调度器，主动让出 CPU；另外，就是一些已经被标记为 deprecated 的 resume、stop、suspend 之类的方法，比如在最新的 JDK 实现中，destroy/stop 方法已被移除
+ 基类 Object 提供了一些基础的 wait/notify/notifyAll 方法。如果我们持有某个对象的 Monitor 锁，调用 wait 会让当前线程处于等待状态，直到其他线程 notify 或者 notifyAll。所以，本质上是提供了 Monitor 的获取和释放的能力，是基本的线程间通信方式。

#### Thread Objects

每个线程都与一个 [Thread][td] 相关。有两种策略使用 *Thread* 对象创建并发应用。

+ 直接控制线程的创建和管理，每次应用需要启动异步任务时，只需实例化 *Thread*。
+ 要从应用的其他部分抽象线程管理，将应用的任务传递给 *executor*。

### DeadLock

High level：

死锁是指有两个或两个以上的线程在执行过程中去争夺同样一个共享资源造成的相互等待的一个现象。如果没有外部的干预，线程会一直阻塞，无法往下去执行，这样一直处于相互等待资源的线程，我们称为死锁线程。
导致死锁的条件有4个，这4个条件同时满足就会产生死锁：

+ 互斥条件：共享资源 A 和 B 只能被一个线程占用
+ 请求和保持条件：线程 t1 已经取得共享资源 X，在等待共享资源 Y 的时候，不释放共享资源 X
+ 不可抢占条件：其他线程不能强行抢占线程 t1 占有的资源
+ 循环等待条件：线程 t1 等待线程 t2 占有的资源，线程 t2 等待线程 t1 占有的资源就是循环等待

这些条件导致死锁之后，只能通过人工干预来解决，比如说重启服务或 kill 掉这个线程，而按照死锁发生的 4 个条件，我们只需要破坏其中任意一种就能解决死锁问题，不过互斥条件是没办法破坏的，因为它的互斥锁的基本约束，而其他的三个条件都有办法来破坏：

+ 破坏请求和保持条件：
  + 我们可以一次性申请所有的资源，这样就不存在锁要等待了
+ 破坏不可抢占条件：
  + 占用部分资源的线程在进一步申请其他资源的时候，如果申请不到，我们可以主动去释放它占有的资源
+ 破坏循环等待条件：
  + 可以按序申请资源来预防，按序申请是指资源是有线性顺序的，申请的时候，可以先申请资源序号小的，然后再去申请资源序号大的，这样线性化之后，自然就不存在循环了

[threads]:https://docs.oracle.com/javase/specs/jvms/se6/html/Threads.doc.html#21294
[synchronized]:https://docs.oracle.com/javase/specs/jvms/se6/html/Compiling.doc.html#6530
[CCY]:https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html
[nft]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newFixedThreadPool-int-
[ncp]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newCachedThreadPool-int-
[nst]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newSingleThreadExecutor-int-
[pb]:https://docs.oracle.com/javase/8/docs/api/java/lang/ProcessBuilder.html
[td]:https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html
[gt]:https://en.wikipedia.org/wiki/Green_threads
