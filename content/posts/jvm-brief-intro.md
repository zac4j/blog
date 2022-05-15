---
title: "JVM a brief intro"
date: 2022-04-18T21:28:09+08:00
description: "A brief intro to JVM(Hotspot) architecture"
tags: ["basic", "hotspot"]
categories: ["java", "jvm"]
author: "Zac"
draft: true
---

JVM 全称是 Java Virtual Machine，JVM 有2个作用：第一是运行并管理 Java 源码文件所生成的 class 文件，第二是在不用的操作系统上，安装不同平台的 JVM，从而实现 Java 跨平台的保障。

<!--more-->

一般情况下，对于开发者而言，即时不熟悉 JVM 的运行机制，也不影响业务代码的开发，因为在安装 JRE 或 JDK 之后，其中就内置了 JVM，所以只需要将 class 文件交给 JVM 运行即可。但当程序运行过程中出现了问题，而这个问题发生在 JVM 层面的时候，我们就需要熟悉 JVM 的运行机制，才能够去迅速排查并解决 JVM 的性能问题。

![hotspot](/img/jvm-arch.png)

通过上图我们可以了解 JVM 的大致流程：JVM 将一个 class 文件通过类加载机制装载到 JVM 中，然后放到不同的运行时数据区，通过 JIT 编译器来执行。

第一部分是类加载系统，在 Java 中 class 文件由源码文件构成，class 文件加载到内存中需要借助 Java 的类加载机制，类加载机制分为装载，链接和初始化，它主要就是对类进行查找，验证和以及分配相关内存空间和赋值。

第二部分是运行时数据区，运行时数据区解决的问题是，class 文件在加载到内存后，该如何进行存储不同的类别数据，以及数据该如何进行流转，比如 Method Area 通常会存储 class 文件对应的运行时长量池、字段和方法的元数据信息、类的模版信息等，Heap 存储各种 Java 中对象的实例，Java Threads 通过线程以栈的方式运行加载各个方法，Native Internal Threads 加载运行 native 类型的方法，PC register 则是保存没个线程执行方法的实时地址，这样通过运行时数据区的 5 个部分就能很好的把数据存储和运行起来。

第三部分就是垃圾回收器，就是对运行时数据区中的数据进行管理和回收，回收机制可以基于不同的垃圾回收器，比如 Serial、Parallel、CMS、ZGC等，可以针对不同的业务场景去选择不同的垃圾回收器，只需通过 JVM 参数设置即可，同时这些垃圾收集器其实就是对不同垃圾收集算法的实现，核心的算法由3个：标记-清除算法、标记-整理算法、复制算法。

第四部分是 JIT Compiler 和 Interpreter，class 的字节码指令通过 JIT Compiler 和 Interpreter 翻译成对应操作系统的 CPU 指令，不过可以选择解释执行或编译执行，如图：

![jitcompiler](/img/jit-compiler.png)

第五部分是 JNI 部分，如果我们想找到 Java 中某个 native 方法如何通过 C/C++实现，那么可以通过 Native Method Interface 来进行查找，也就是所谓的 JNI 技术。

通过上面的解释我们可以了解 JVM 是如何运行的，当然在实际的操作过程中，我们可以借助 JVM 参数

![jvm-args](/img/jvm-args.png)

和一些常见的 JDK 命令，如 jps, jinfo, jhat, jmap, jstat, jstack 等，来分析 JVM 出现的常见问题，并对其进行调优。

[ref]:http://ksoong.org/troubleshooting/index.html
