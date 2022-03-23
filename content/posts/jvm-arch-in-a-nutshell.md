---
title: "JVM Basic"
date: 2020-07-17
description: "Java Virtual Machine Architecture"
tags: ["jvm", "jmm"]
categories: ["java"]
draft: false
---

## Run-Time Data Areas

![jvm-arch](/img/jvm-arch.png)

### The PC Register

JVM 可以支持多个线程同时运行，每个线程都有自己独立的 PC 寄存器(program counter register)。在任意时间点，任一线程在执行某个方法的代码，被称作该线程的[当前方法(current method)][currm]。

+ 如果该方法是 *native* 的，pc 寄存器的值是 *undefined*。
+ 如果该方法不是 *native* 的，pc 寄存器包含当前正在执行指令的地址。

JVM 的 pc 寄存器容量足够在特定平台上保存 *returnAddress* 或 *native* 指针。

### JVM Stacks

对每个线程，JVM 在创建线程时都会创建一个私有的堆栈。JVM 仅对堆栈进行两种操作：推入(push)和弹出(pop)栈帧(frames)。JVM 堆栈的内存**不必是连续的**。

线程执行的每个方法调用(method calls)都存储在对应的堆栈中，包括入参、局部变量、中间计算和其他数据。方法完成后，JVM 将从堆栈中删除对应的条目，完成所有的方法调用后，堆栈将变为空，并且在终止线程前，销毁该堆栈。

存储在堆栈中的数据仅适用于对应线程，对别的线程不适用。因此，可以说局部数据是线程安全的。堆栈中每个条目都称为 *堆栈帧(Stack Frame)* 或 *激活记录(Activation Record)*。

> 第一版的 Java 虚拟机规范 *The Java® Virtual Machine Specification*, *JVM 堆栈(Java Virtual Machine stack)* 被称为 *Java 栈(Java stack)*

JVM 规范允许Java虚拟机堆栈具有固定大小，或者根据计算要求动态伸缩。 如果Java虚拟机堆栈的大小固定，则在创建每个Java虚拟机堆栈时可以独立选择它们的大小。

关联的异常：

+ 如果线程中的计算需要比允许的 JVM 更大的堆栈，则 JVM 将抛出 **StackOverflowError**。

+ 如果可以动态扩展 JVM 堆栈容量，并且尝试进行动态扩展，但外部环境无法提供足够的内存资源，或没有足够的内存资源为新线程创建 JVM 堆栈，则 JVM 会抛出 **OutOfMemoryError**。

### Heap

堆是运行时数据区内为所有类实例和数组分配内存的部分，是共享资源。

堆在虚拟机启动时创建，*垃圾回收器(garbage collector)* 可以回收对象堆堆存储，对象永远不会显式释放内存。堆的大小同样可以是固定的，或者根据计算要求动态伸缩。堆内存**不必是连续的**。

用户可以控制堆固定初始大小，或动态伸缩场景控制堆的最大/最小值。

关联的异常：

+ 计算需要的堆内存多于系统可以提供的内存资源， JVM 会抛出 **OutOfMemoryError**。

### Method Area

方法区存储每个类的结构，如*运行时常量池(run-time constant pool)*，*字段(field)*，和*方法数据(method data)*，以及方法和构造函数的代码，是共享资源。

方法区同样是在 JVM 启动时创建。尽管方法区在逻辑上是堆的一部分，但 JVM 实现可以选择不对其进行垃圾回收或压缩。

方法区可以是固定大小，或根据计算需求进行动态伸缩。方法区内存同样**不必是连续的**。

关联的异常：

+ 方法区需要分配的内存多于系统可提供的内存资源，JVM 会抛出 **OutOfMemoryError**。

### Native Method Stacks

JVM 可以使用传统的堆栈，俗称 “C 栈”，来支持 *native* 方法。*native* 方法栈可以用于 JVM C语言指令集的解释器。JVM 不能加载 *native* 方法，并且它本身不依赖于传统堆栈，不需要提供 *native* 方法栈。如果提供，通常在创建线程时为每个线程提供 *native* 方法栈。

JVM 规范允许 *native* 方法堆栈具有固定大小，或者根据计算要求动态扩展和收缩。如果本机方法堆栈的大小固定，则在创建每个 *native* 方法栈的大小时可以独立选择。

*native* 方法栈关联的异常：

+ 如果线程中的计算所需的 *native* 方法栈容量超出允许的范围，则 JVM 会抛出 **StackOverflowError**.
+ 如果 *native* 方法栈容量尝试动态扩展，但外部环境无法提供足够的内存资源，或没有足够的内存资源为新线程创建 *native* 方法栈，则 JVM 会抛出 **OutOfMemoryError**.

### Run-Time Constant Pool

运行时常量池是每个类或接口的 *class* 文件中 *constant_pool* 表的运行时表示形式。它主要包含：

+ 编译时数字文字
+ 运行时方法和字段引用

运行时常量池功能类似于常规编程语言的符号表。尽管它包含的数据范围比典型的符号表更大。

每个运行时常量池都是从 JVM 方法去分配的，当 JVM 创建类或接口时，将为该类或结构构造运行时常量池。

运行时常量池会关联如下异常：

+ 创建类或缄口时，如果运行时常量池构造所需的内存超过 JVM 方法区的可用内存，则 JVM 会抛出 **OutOfMemoryError**.

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

JVM Run-Time Data Areas: <https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4>
JVM architecture: <https://www.geeksforgeeks.org/jvm-works-jvm-architecture/>
Class Loader: <https://www.baeldung.com/java-classloaders>

[currm]:https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6
