---
title: "Threads and Locks"
date: 2020-07-27
draft: true
---

## 8.12 Threads

Threads are created and managed by the classes Thread and ThreadGroup. Creating a Thread object creates a thread, and that is the only way to create a thread. When the thread is created, it is not yet active; it begins to run when its start method is called.

线程由 Thread 和 ThreadGroup 创建和管理。创建一个线程对象会创建一个线程，这是创建线程的唯一方法。创建线程后，它尚未处于活动状态。当调用其 *start* 方法时，它开始运行。

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

The rules for volatile variables effectively require that main memory be touched exactly once for each use or assign of a volatile variable by a thread, and that main memory be touched in exactly the order dictated by the thread execution semantics. However, such memory operations are not ordered with respect to read and write operations on nonvolatile variables.

## Thread Pool

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

[threads]:https://docs.oracle.com/javase/specs/jvms/se6/html/Threads.doc.html#21294
[synchronized]:https://docs.oracle.com/javase/specs/jvms/se6/html/Compiling.doc.html#6530
[nft]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newFixedThreadPool-int-
[ncp]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newCachedThreadPool-int-
[nst]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newSingleThreadExecutor-int-
