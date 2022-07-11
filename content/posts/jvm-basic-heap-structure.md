---
title: "JVM Heap Structure"
date: 2022-07-08T10:09:33+08:00
description: "Intro to JVM's Heap Structure"
tags: ["jmm", "heap area"]
categories: ["java", "jvm"]
draft: true
---

## 堆内部空间

堆内部结构随着 JVM 的发展和新 GC 方式的引入，可以有不同角度的理解。下图是**年代视角**的堆结构图：

![heap](/img/heap-struct.webp)

按照 GC 年代方式划分，Java 堆内部分为：

### 新生代（Young Generation）

新生代是大部分对象创建和销毁的区域，通常 Java 应用中，绝大部分对象生命周期都是很短暂的。其内部又分为 Eden 区域，作为对象初始分配的区域；两个 Survivor，有时候也叫 from、to 区域，用来放置从 Minor GC 中保留下来的对象。

JVM 会随意选取一个 Survior 区域作为 to，然后会在 GC 过程中进行区域间拷贝，也就是将 Eden 中存活下来的对象和 from 区域的对象，拷贝到这个 to 区域。这种设计主要是为了防止内存碎片化，并进一步清理无用对象。

从内存模型而不是垃圾收集的角度，对 Eden 区域继续进行划分，Hotspot JVM 还有一个概念叫做 Thread Local Allocation Buffer （TLAB）。这是 JVM 为每个线程分配的一个私有缓存区域，如果没有 TLAB 的话，多线程同时分配内存时，为避免操作同一地址，可能需要加锁等机制，进而影响分配速度。

### 老年代（Old Generation）

放置长生命周期的对象，通常都是从 Survior 区域拷贝过来的对象。当然，也有特殊情况，我们知道普通的对象会被分配在 TLAB 上；如果对象较大，JVM 会试图直接分配在 Eden 其他位置上；如果对象太大，完全无法在新生代找到足够长的连续空闲空间，JVM 就会直接分配到老年代上。

### 永久代（Permanent Generation）

早期的 Hotspot JVM 的方法区实现了永久代，用于储存 Java 类数据、常量池、Intern 字符串缓存，JDK 8 之后就不存在永久代了。

那么如何利用 JVM 参数，直接影响堆和内部区域的大小呢，我们来简单总结一下：

| 设定内存区域大小            | JVM 参数                | comment                                                                                                       |
|-----------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| 最大堆体积                  | -Xmx value              |                                                                                                               |
| 最小堆体积                  | -Xms value              |                                                                                                               |
| 老年代/新生代大小比例       | -XX:NewRatio=value      | 默认情况下，这个数值是 2，意味着老年代是新生代的 2 倍大；换句话说，新生代是堆大小的 1/3。                     |
| 新生代的大小                | -XX:NewSize=value       |                                                                                                               |
| Eden 和 Survivor 的大小比例 | -XX:SurvivorRatio=value | 如果这个值是 8，那么 Survivor 区域就是 Eden 的 1/8 大小，也就是新生代的 1/10，因为 YoungGen=Eden + 2*Survivor |

#### Virtual 区域

我们在年代视角堆结构图中标记了 Virtual 区域，那么什么是 Virtual 区呢？

在 JVM 内部，如果 Xms 小于 Xmx，堆的大小并不会直接扩展到其上限，也就是说保留的空间（reserved）大于实际能够使用的空间（committed）。当内存需求不断增长时，JVM 会逐渐扩展新生代等区域的大小，所以 Virtual 区域代表的就是暂时不可用（uncommitted）的空间。

## 堆外空间

``` log
Zac:java Zac$ java -XX:NativeMemoryTracking=summary -XX:+UnlockDiagnosticVMOptions -XX:+PrintNMTStatistics HelloWorld
Hello World!

Native Memory Tracking:

Total: reserved=5863529371, committed=364522395
-                 Java Heap (reserved=4294967296, committed=268435456)
                            (mmap: reserved=4294967296, committed=268435456) 
 
-                     Class (reserved=1082243435, committed=5093739)
                            (classes #495)
                            (  instance classes #430, array classes #65)
                            (malloc=113003 #548) 
                            (mmap: reserved=1082130432, committed=4980736) 
                            (  Metadata:   )
                            (    reserved=8388608, committed=4456448)
                            (    used=3253056)
                            (    free=1203392)
                            (    waste=0 =0.00%)
                            (  Class space:)
                            (    reserved=1073741824, committed=524288)
                            (    used=309872)
                            (    free=214416)
                            (    waste=0 =0.00%)
 
-                    Thread (reserved=16849528, committed=16849528)
                            (thread #16)
                            (stack: reserved=16777216, committed=16777216)
                            (malloc=55080 #98) 
                            (arena=17232 #30)
 
-                      Code (reserved=253667160, committed=7763800)
                            (malloc=34648 #399) 
                            (mmap: reserved=253632512, committed=7729152) 
 
-                        GC (reserved=212278175, committed=62856095)
                            (malloc=18254751 #2096) 
                            (mmap: reserved=194023424, committed=44601344) 
 
-                  Compiler (reserved=138338, committed=138338)
                            (malloc=2506 #43) 
                            (arena=135832 #5)
 
-                  Internal (reserved=567915, committed=567915)
                            (malloc=535147 #921) 
                            (mmap: reserved=32768, committed=32768) 
 
-                    Symbol (reserved=1731712, committed=1731712)
                            (malloc=1133976 #2317) 
                            (arena=597736 #1)
 
-    Native Memory Tracking (reserved=139176, committed=139176)
                            (malloc=4248 #51) 
                            (tracking overhead=134928)
 
-               Arena Chunk (reserved=834568, committed=834568)
                            (malloc=834568) 
 
-                   Tracing (reserved=97, committed=97)
                            (malloc=97 #5) 
 
-                   Logging (reserved=4428, committed=4428)
                            (malloc=4428 #186) 
 
-                 Arguments (reserved=18711, committed=18711)
                            (malloc=18711 #487) 
 
-                    Module (reserved=60728, committed=60728)
                            (malloc=60728 #1044) 
 
-              Synchronizer (reserved=19912, committed=19912)
                            (malloc=19912 #160) 
 
-                 Safepoint (reserved=8192, committed=8192)
                            (mmap: reserved=8192, committed=8192) 
 

```
