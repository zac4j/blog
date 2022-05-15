---
title: "Apm Basic"
date: 2022-04-20T08:54:05+08:00
tags: ["apm", "performance", "optimization"]
description: "Intro to app performance management"
categories: ["android"]
author: "Zac"
draft: true
---

## 启动优化

如果要你负责一款 App 的启动性能优化，你会如何去规划整体的优化方法？你可能一个想到很多方面的细节点，比如：

+ 优化主线程耗时
+ 减少布局层级
+ 对某些启动任务做按需加载或预加载
+ 避免主线程 IO
+ 对线程进行优化
+ 分析工具帮助定位性能问题

+ Lock trace 锁信息
  + 多线程同步等锁
  + Java 层的锁，无论是同步方法还是同步块，最终都会走到虚拟机的 MonitorEnter 和 MonitorExit，在 MonitorEnter 中实现了多种状态的切换，包括从无锁到轻锁，轻锁中的偏向和重入，出现竞争并超过自旋的次数后升级成重锁分配 monitor 对象，其中 art 的自旋不是真的自旋，而是用 `sched_yield` 主动让出 CPU 等待下次调度。
+ IO trace IO耗时
  + IO 操作读写哪些文件
  + IO 读写执行时常
+ Binder trace
  + Sleep 带来的耗时，处于 sleep 状态，进程通常是在等锁或是 Binder 调用耗时导致，通常在线下，可以通过开启 `tracing/events/binder`
+ App atrace
+ Linux ftrace
+ Android Framework atrace

性能优化工具

+ Systrace Android 4.3(API level 18) and higher
  + Method
    + Trace.beginSection
    + Trace.endSection
  + Data from Android kernel
    + CPU scheduler
    + disk activity
    + app threads
+ Perfetto Android 10 and higher
+ TraceView
+ CPU Profiler
+ RheaTracer

Questions

### 相关技术

+ 文件描述符 file descriptor
  + Linux 系统中打开文件就会获得文件描述符，它是个很小的非负整数。每个进程在 PCB(Process Control Block) 中保存着一份文件描述符表，文件描述符就是这个表的索引，每个表项都有一个指向已打开文件的指针。
  + __fdget_pos -- linux 文件读操作
    + 原因是在开启 Systrace 后，由于所有线程的 trace 都会写入同一个文件，所有线程会同步竞争内核态的文件 pos 锁导致。
+ [字节码插桩技术][cz]
  + 在 Java 层通过记录方法首末位置时间戳、所在线程等信息。将数据异步记录到文件。
+ Android native hook
  + 有两种实现方式:
    + PLT hook
      + project: xHook
    + inline hook
      + project: cydia substrate
  + 实现原理
    + 涉及so动态链接过程与ELF文件格式，汇编指令等

+ 静态代码插桩技术自动添加 trace，分析 App 运行时耗时性能分析工具
+ Trace file
  + ThreadName
  + ThreadId
  + Time seconds
  + B|E(begin or end)
  + ProcessId
  + Tag

System-wide tracing on Android and Linux

+ `ftrace` [Linux's kernel tracing][ftrace], allows to record kernel events into trace.
+ `atrace` 收集服务和应用中的用户空间注释
  + Android system and app trace events
  + [Java/Kotlin apps(SDK)][sdk]: `android.os.Trace`
  + Native processes(NDK)[ndk]:
+ 使用 `heapprofd` 收集服务和应用的本地内存使用情况信息

atrace

![atrace](/img/atrace.png)

线程调度状态

+ Running：线程正常执行代码逻辑
+ Runnable：可执行状态，等待调度，如果长时间调度不到，CPU 繁忙
+ Sleeping：休眠，一般是在等待事件驱动
+ Uninterruptible Sleep：不可中断的休眠，需要看 args 的描述来确定当时的状态
+ Uninterruptible Sleep - Block I/O：IO 阻塞

IO 耗时

冷启动耗时，占比最长的是进程处于D状态（Uninterruptible Sleep，PS 查看进程状态显示 D，俗称D状态）的时间，这部分耗时通常占启动总耗时的 40% 左右，进程处于 D 状态通常在等待 IO，比如磁盘 IO，外设 IO，正常因为得不到 IO 的响应，进程才进入 Uninterruptible Sleep 状态，所以要想使得进程从 Uninterruptible Sleep 状态恢复，就得使进程等待 IO 恢复。

插桩性能优化

+ 支持自定义插桩作用域，减少 Trace 对于其他无关模块的运行损耗
+ 针对 Trace 数据出现不闭合的问题，*对 catch 代码块进行全插桩*
+ 针对高频调用函数，可以选择性的添加到黑名单中，提高运行时性能
+ 为支持生产使用，采用在 proguard 后进行插桩，由于函数内联优化，相较于混淆前插桩数量可以减少 2.6%
+ 在编译阶段通过分析字节码信息，过滤掉不耗时函数的插桩

优化 Trace 写入性能

开启 atrace 后，所有线程会将所有的 trace 都写入到 trace_marker 文件，会带来  IO 损耗剧增，掩盖了真实的性能问题，原因是所有线程都在短时间向 trace_marker 文件进行写入操作，同时竞争内核态 pos 锁，导致获取到的 trace 文件无法真实反映性能问题。

优化的方式是，将原本直接写入内核态文件的 Trace 在用户态进行拦截，缓存起来，再以异步 IO 的方式转储。既避免了大量用户态与内核态切换带来的上下文损耗，又避免了直接 IO 带来的 IO 损耗。

## 内存优化

OOM 归因分类：

![oom](/img/oom_reason.png)

其中 pthread_create 和 fd 数量不足均为 native 内存限制导致的 Java 层崩溃，可以对这部分内存问题做针对性优化：

+ 线程收敛、监控
+ 线程栈泄漏自动修复
+ FD 泄漏监控
+ 虚拟内存监控、优化
+ app 64位专项

### 堆内存治理思路

Java 堆内存超限主要分为两类：

+ 堆内存单次分配过大/多次分配累计过大

  触发这类问题的原因有数据异常导致单词内存分配过大超限，如一些是 StringBuilder 拼接累计大小过大等等，解决的思路也比较简单，问题就在当前的堆栈。
+ 堆内存累积分配触顶

  这类问题的问题堆栈比较分散，在任何内存分配的场景上都有可能会被触发，那些高频的内存分配节点发生的概率会更高，比如 Bitmap 分配内存。这类 OOM 的根本原因是内存累积占用过多，而当前的问题堆栈只是引发 OOM 最后的一环，并不是问题所在。所以这类问题我们需要分析整体的内存分配情况，从中找到不合理的内存使用（比如内存泄漏、大对象、过多小对象、大图等）。

### 工具建设

线下目前主流分析内存泄漏的工具有：

+ Android Studio Memory Profiler
+ LeakCanary
  + 缺点：
    + 检测出来内存泄漏过多，没有比较好的优先级排序，研发解决不及时，问题容易堆积
    + Android 端 HPROF 的获取依赖原生的 Debug.dumpHprof，dump 过程会挂起主线程导致明显卡顿，使用体验较差
    + LeakCanary 基于 Shark 引擎分析，分析速度较慢，分析过程还会影响进程内存占用
    + 分析结果较为单一，仅仅只能分析出 Fragment、Activity 的内存泄漏，像大对象、过多小对象导致的 OOM 无法分析。
+ Memory Analyzer

线上分析内存泄漏的思路

在发生 OOM 或者内存触顶等条件下，通过用户无感知 dump 内存的 HPROF 文件，为了减少对 App 运行时影响，通过裁剪 HPROF 回传，在服务端对 HPROF 文件进行分析，分析出内存泄漏、大对象、小对象、图片问题并按照泄漏链路进行归因，根据大数据分析按照用户泄漏发生次数、泄漏大小、总大小等维度排序，推进业务研发按照优先级顺序来建立消费流程。

#### HPROF 收集

为了解决 dump 挂起进程的问题，我们可以采用子进程 dump + fileObserver 的方式完成 dump 采集和监听。

在 fork 子进程之前先 `suspend`获取主进程中的线程拷贝，通过 fork 系统调用创建子进程让子进程拥有父进程的拷贝，然后 fork 出的子进程中调用 Hprof 的 `DumpHeap` 函数即可完成把耗时的 dump 操作在放在子进程。由于 `suspend` 和 `resume` 是系统函数，抖音**通过自研的 native hook 工具**对 libart.so [hook 获取系统][hook]调用。由于写入是在子进程完成的，我们通过 Android 提供的 fileObsever 文件写入进行监控获取 dump 完成时机。

[tiktok]:https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247491335&idx=1&sn=e3eabd9253ab2f83925af974db3f3072&scene=21#wechat_redirect
[blue]:https://juejin.cn/post/7074762489736478757
[cz]:https://mp.weixin.qq.com/s/dbseDMO3tqNPtSvBB5UL3Q
[ftrace]:https://www.kernel.org/doc/Documentation/trace/ftrace.txt
[hook]:https://mp.weixin.qq.com/s/LytCXxiX8e07N9MGI9dPkg
