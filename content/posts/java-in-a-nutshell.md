---
title: "Java in a Nutshell"
date: 2022-06-08T17:23:59+08:00
tags: ["java"]
description: "Java in a Nutshell"
categories: ["java"]
author: "杨晓峰·geektime"
draft: true
---

Java 本身是一种面向对象的语言，最显著的特性有两个方面，一是 Write once, run anywhere，能够非常容易的获取跨平台能力；另外就是垃圾收集，Java 通过垃圾收集器（Garbage Collector）回收分配内存，大部分情况下，程序员不需要自己操心内存的分配和回收。

我们日常接触到的 JRE（Java Runtime Environment）和 JDK（Java Development Kit）。JRE 也就是 Java 运行环境，包含了 JVM 和 Java 类库，以及一些模块等。而 JDK 可以看作是 JRE 的一个超集，提供了很多工具，如编译器、各种诊断工具等。

## JIT 和 AOT

我们开发的 Java 程序源代码，首先通过 Javac 将源码编译成字节码（bytecode），然后，在运行时通过 JVM 内嵌的解释器（Interpreter）将字节码转换成最终的的机器码。

+ 文件类型的转换: ".java" => ".class" => ".so"
+ 文件内容的转换: "source code" => "bytecode" => "machine code"

我们通常把 Java 分为编译期和运行时，Java 的编译和 C/C++ 有着不同意义，Javac 的编译，将 Java 应用源码编译生成 ".class" 文件里面实际是字节码，而不是可以直接执行的机器码。

Oracle Hotspot JVM，提供了 JIT（Just-in-Time）编译器，也就是我们通常说的动态编译器，JIT 能够在运行时将热点代码编译成机器码，这种情况下部分热点代码就属于编译执行了。

在运行时，JVM 通过类加载器（Class Loader）加载字节码，解释（Interpreter）或编译（JIT）执行。Oracle JDK 8 的 Hotspot JVM 内置了两个不同的 JIT compiler，C1 对应的 client 模式，适用于对于启动速度敏感的应用，比如普通 Java 桌面应用；C2 对应 server 模式，它的优化是为了长时间运行的服务器端应用设计的。默认是采用分层编译（TieredCompilation）。

JVM 启动时，可以指定不同的参数对运行模式进行选择。比如，指定 "-Xint" 就是告诉 JVM 只进行解释执行，不对代码进行编译，这种模式抛弃了 JIT 可能带来的性能优势。与此相对的，是指定 "-Xcomp" 参数，告诉 JVM 关闭解释器，只进行编译执行，这种模式会导致 JVM 启动变慢很多，同时有些 JIT 编译器优化方式，比如分支预测，如果不进行 profiling，往往不能进行有效优化。Oracle JDK 8 默认是解释和编译混合的模式，即所谓的混合模式 "-Xmixed"。

除了常见的 JIT 编译模式，Oracle JDK 9 引入了新的编译方式，即 AOT（Ahead-of-Time Compilation），在编译时直接将字节码编译成机器码，这样就避免了 JIT 预热等方面的开销。
