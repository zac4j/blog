---
title: "Java Basic"
date: 2020-07-03T09:28:31+08:00
draft: false
---

🌟答案的组织策略：知道是什么，知道为什么，知道怎么用

0.JDK 和 JRE 有什么区别？

+ JDK 包含 JRE，同时还包含编译 Java 源码的编译器 javac，还包含了很多 Java 程序调试和分析的工具。
+ 加分项：除了 javac 还了解哪些命令行工具，它们的用途是什么？
  + jcmd：综合工具
  + jps：虚拟机进程状况工具
  + jinfo：虚拟机配置信息工具
  + jstat：虚拟机统计信息监视工具
  + jmap：Java 内存映像工具
  + jhat：虚拟机堆转储快照分析工具
  + jstack：Java 堆栈追踪工具
+ 加分项jstat 用过吗，有哪些参数？
  + -class：监视类装载、卸载数量、总空间和类装载所耗费的时间
  + -gc：监视 Java 堆的状况，包括 Eden 区，两个 survior 区，老年代、永久代等的容量，已用空间、GC 时间合计等统计

1.equals 和 == 的差别？
两者都是判断等价性，区别要从入参类型来看：

+ 对于基本类型，他们是比较值是否相等
+ 对于引用类型，他们是判断引用的是否为同一对象

+ 考察点：equals 的概念
+ 🌟实际要求：平时对源码的深挖意识即技术钻研和**批判性思维**
+ 考察目的：
  + 基础知识的扎实程度
  + 候选人对技术的热情
  
2.Java 中操作字符串有那些类？它们之间有什么区别？

+ 主要有：String，StringBuilder，StringBuffer
+ 区别：StringBuffer 和 StringBuilder 都继承自 AbstractStringBuilder，StringBuffer 具备线程安全性
  + **加分项**：String 源码分析
    + String 是 *final* 关键字修饰 -> 对象的值不可变 -> 每次操作都会生成新的对象
    + StringBuffer 和 StringBuilder 对象的值可变，但拼接字符串开销比较小
  + **加分项**：StringBuffer 线程安全性分析（查源码、找 synchronized、线程锁）
+ 使用场景：并发必选 StringBuffer，迭代必选 StringBuilder，普通场景选 String，避免中途不必要的类型转换开销
  
3.HashMap、HashTable、TreeMap 有什么区别？
是什么：

+ 上面3种数据结构都是最常见的 Map 接口的实现，是以 key-value 的形式存储和操作数据的容器类型。
+ HashTable 是 Java 类库提供的一个 hash 实现，本身是同步的，不支持 null key 和 null value，由于同步性能开销，所以已经很少推荐使用了。
+ HashMap 是应用广泛的哈希表实现，行为上大致与 HashTable 一致，主要区别在于 HashMap 不是同步的，支持 null key 和 null value 等。通常情况下，HashMap 进行 get 和 put 操作可以达到**常数时间的性能**。所以大部分利用键值对存取场景的首选。
+ TreeMap 则是基于红黑树的一种提供顺序访问的 Map，它的 get、put、remove 之类的操作都是 O(logn) 时间复杂度，具体顺序由可以指定的 Comparator 来决定，后者根据键的自然顺序来判断。
