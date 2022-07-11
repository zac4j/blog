---
title: "JVM Basic - Rumtime Data Area"
date: 2022-03-24T08:44:28+08:00
description: "Intro to JVM's Rumtime Data Area"
tags: ["jmm", "rumtime data area"]
categories: ["java", "jvm"]
draft: false
---

前面说了[类加载器]({{< ref "/posts/jvm-basic-classloader" >}})，我们接下来看看**运行时数据区(Run-Time Data Areas)**

## Run-Time Data Areas

![jvm-arch](/img/jvm-arch.png)

### 程序计数器（The pc Register）

JVM 可以支持多个线程同时运行，每个线程都有自己的程序计数器(program counter register)。在任意时间点，任一线程只有一个方法在执行，即该线程的[当前方法(current method)][currm]。程序计数器会存储当前方法的 JVM 指令地址；如果是在执行本地方法，则是未指定值（undefined)。

### Java 虚拟机栈 (JVM Stack)

每个线程在创建时都会创建一个虚拟机栈。虚拟机栈存储一个个 *栈帧(frames)*，对应着一次次 Java 方法调用。

前面在程序计数器中提到了当前方法，同理，在同一时间点，对应的只会有一个活动的栈帧，通常称作当前帧，方法所在的类叫做当前类。如果该方法中调用了其他方法，对应的新的栈帧会被创建出来，成为新的当前帧，一直到它返回结果或者执行结束。JVM 对于栈帧仅有两种操作：push 和 pop。JVM 栈的内存**不必是连续的**。

栈帧中存储着局部变量表、操作数（operand）栈、动态链接、方法正常退出或异常退出的定义等。

> 第一版的 Java 虚拟机规范 *The Java® Virtual Machine Specification*, *JVM 栈* 被称为 *Java 栈 (Java stack)*

在 JVM 启动时，我们可以通过 `-Xss` 来指定每个线程的堆栈大小。

关联的异常：

+ 如果线程中的计算需要比允许的 JVM 栈容量更大，则 JVM 将抛出 **StackOverflowError**。
  + 例如我们写的程序不断进行递归调用，而且没有退出条件，就会导致不断进行压栈，这种情况就会抛出 **StackOverflowError**.

+ 如果可以动态扩展 JVM 栈容量，并且尝试进行动态扩展，但外部环境无法提供足够的内存资源，或没有足够的内存资源为新线程创建 JVM 栈，则 JVM 会抛出 **OutOfMemoryError**。

### 堆 (Heap)

堆是 Java 内存管理的核心区域，用来放 Java 对象实例，几乎所有创建的 Java 对象实例都是被直接分配在堆上。堆对所有线程共享。

堆在虚拟机启动时创建，堆内空间会被不同的 *垃圾回收器(garbage collector)* 进行进一步细分，比较有名的就是新生代、老年代的划分。对象永远不会被显式释放内存 (deallocated)。堆的大小同样可以是固定的，或者根据计算要求动态伸缩。堆内存**不必是连续的**。

在 JVM 启动时，我们指定 `-Xmx`/`-Xms` 参数来指定最大/最小堆空间指标。

关联的异常：

+ 计算需要的堆内存多于系统可以提供的内存资源， JVM 会抛出 **OutOfMemoryError**。

### 方法区 (Method Area)

方法区也是所有线程共享的一块内存区域，用于存储元数据（Meta data），类结构信息，例如*运行时常量池(run-time constant pool)*，*字段(field)*，和*方法代码(method data)*等。

由于早期的 Hotspot JVM 实现，很多人习惯于将方法区成为永久代（Permanent Generation）。Oracle JDK 8 中已将永久代移除，同时增加了元数据区（Metaspace）。

方法区同样在虚拟机启动时创建。尽管方法区在**逻辑上是堆的一部分**，但一些简单的 JVM 实现可以选择不对其进行垃圾回收或压缩。

方法区可以是固定大小，或根据计算需求进行动态伸缩。方法区内存同样**不必是连续的**。

关联的异常：

+ 方法区需要分配的内存多于系统可提供的内存资源，JVM 会抛出 **OutOfMemoryError**。

#### 运行时长量池 (Rum-Time Constant Pool)

运行时常量池是方法区的一部分，是每个类或接口的 *class* 文件中 *constant_pool* table 的运行时的表现形式。它主要包含：

+ 编译期生成的字面量
+ 运行时方法和字段引用

运行时常量池功能类似于常规编程语言的符号表。尽管它包含的数据范围比典型的符号表更大。

每个运行时常量池都是从 JVM 方法去分配的，当 JVM 创建类或接口时，将为该类或结构构造运行时常量池。

运行时常量池会关联如下异常：

+ 创建类或缄口时，如果运行时常量池构造所需的内存超过 JVM 方法区的可用内存，则 JVM 会抛出 **OutOfMemoryError**.

### 本地方法栈（Native Method Stack）

本地方法栈和 Java 虚拟机栈相似，支持对本地方法的调用，也是每个线程都会创建一个。在 Oracle Hotspot JVM 中，本地方法栈和 Java 虚拟机栈是同一块区域，这种结构取决于技术实现，并未在 JVM 规范中强制要求。

+ 如果线程中的计算所需的本地方法栈容量超出允许的范围，则 JVM 会抛出 **StackOverflowError**.
+ 如果本地方法栈容量尝试动态扩展，但外部环境无法提供足够的内存资源，或没有足够的内存资源为新线程创建本地方法栈，则 JVM 会抛出 **OutOfMemoryError**.

## Execution Engine

执行引擎执行 *.class(bytecode)*.它逐行读取字节码，使用各个存储区的数据和信息并执行指令，它可以分成三部分：

+ Interpreter：逐行解释字节码然后执行。缺点是当多处调用一个方法时，每次都需要解释步骤。
+ Just-in-Time Compiler(HotSpot)：用于提高解释器的效率，它编译整个字节码并将其转换为 *native* 代码，每当解释器遇见重复的方法调用时，JIT 都会为该部分提供 *native* 代码，因此不用解释器重复解释，从而提高了效率。
+ Garbage Collector：销毁未引用的对象，回收内存。

## Java Native Interface（JNI）

JNI 是与 *native* 方法库交互的接口，它使 JVM 可以调用 C/C ++ 库。

## Native Method Libraries

供执行引擎需要的 *native* 代码库。

## Reference

+ JVM Run-Time Data Areas: <https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4>
+ JVM architecture: <https://www.geeksforgeeks.org/jvm-works-jvm-architecture/>
+ Class Loader: <https://www.baeldung.com/java-classloaders>

[currm]:https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6
