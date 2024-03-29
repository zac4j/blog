---
title: "How to Prepare Job Interview"
date: 2022-05-04T20:44:38+08:00
tags: ["job", "interview"]
categories: ["career"]
description: Let's prepare job interview
author: Windson Yang
draft: false
---

> 作者：Windson Yang
> 链接：https://zhuanlan.zhihu.com/p/38432342
> 来源：知乎
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

## 概述

工作了几年，当我有了面试官的经验之后，我发现认真准备简历和面试是非常重要的，因为毫无准备就来面试的求职者真的太多了。这篇文章我把这几年作为面试者和面试官身份的的经验給大家，希望大家可以从中学到一些面试的技巧，找到心仪的工作。大家也可以使用 求职课程 进行简历 Review 和模拟面试，这样既能节省请假面试的时间，也能根据我们的反馈改善自己面试的表现。在真实面试的时候就会更有把握。

## 分析阶段

不同的公司乃至部门，面试的流程和着重点都有颇大的差别。国内以腾讯为例，微信部门与深圳总部的面试流程和着重点就不一样。微信一面的时候需要五十分钟内手写 4 道偏简单的算法题，但是在深圳总部面试的时候一面却是没有算法题，两小时的考卷。更多的是与面试官聊技术与项目经验。国外的话，甲骨文五轮的面试可能四轮是系统设计，一轮是算法。亚马逊虽然注重算法，但是非技术问题在面试中占比非常高。你首先需要知道面试中考察什么内容之后才去开始准备，国内可以通过我们整理的高频题库，牛客网，看准网着手复习。国外可以通过一亩三分地，Blind，Leetcode discuss 等网站找到这些信息。

## 准备阶段

### 设定限期

面试准备不能无休止地进行下去，因为计算机知识永远都学习不完。可以给自己设立一个时间点，在时间点之后就开始投简历进行面试。例如你可以设立一个月的面试准备时间，然后再根据求职的岗位以及自己的实际情况去分配时间，把时间主要分给面试主要考察的地方。

### 技术准备

基础知识主要包括：编程语言基础，第三方工具基础（框架，中间件等），算法与数据结构，计算机网络，操作系统，数据库。我在程序员面试推荐书籍这篇文章中列出了面试常见的问题以及对应的解答书籍供大家参考，这里我列举一些面试常见的问题：

1.编程语言基础

+ 数据结构的实现细节以及比较：数组，链表，哈希表是如何实现的，底层内存分配是怎样的？插入与查找的时间复杂度是多少，分别有什么优缺点。
+ 编程语言特性： Java 的字符串池是怎么实现的，垃圾回收的流程以及原理。
+ 关键字特性：包括 Java 中的 static，final，Python 中的 __init__ 关键字的含义以及使用场景。
+ 面向对象的细节：类的封装，函数与变量继承，抽象类和接口有什么区别等。
+ 多线程与多进程：线程如何同步，进程如何同步，wait() 函数使用场景以及常用的并发编程模式。

2.第三方工具

整体架构：这个工具整体的架构是怎样的？主要由哪几个部分组成，它们之间是如何通信以及合作的。
实现原理：核心功能是如何实现的？对比另外一款工具做了哪些优化以及改进。

3.算法基础

算法题：链表操作，二分查找，动态规划，DFS，BFS 等（可以使用 Leetcode 来进行学习）。
算法复杂度的分析：时间复杂度，空间复杂度，平均时间复杂度。
数据结构的实现：实现二叉查找树，Trie 树。

这里说一点题外话，可能有的同学有疑问，觉得这些平常工作都用不到，为什么还要花那么多时间在上面。其实不是的，第一，平常工作都能用到，无论从二分查找到复杂一点的前缀树。开发的过程中如果你知道这些算法／数据结构，就能根据自己的业务来选择最适合的算法／数据结构，减少整个项目的复杂度。 第二，数据结构和算法锻炼的是思维，就像 NBA 球员除了训练投篮和战术之外，还需要训练敏捷度，体力，耐力等其他方面。刷算法题的时候，可以从优秀的前辈中学习到一些有趣的，巧妙的方法。它们能扩展你的编程时思考的范围。即使你不准备换工作，我也建议每天都刷一道算法题，日积月累，一年下来你的算法基础一定能比同龄人高出不少。而且当你真正理解算法题的知识之后，写程序 debug 和花在 Stackoverflow 的时间就会大大减少，往往知道哪里可能有问题并且能大幅地增加工作效率。

4.计算机网络

+ 协议的基础组成与用途：HTTP 协议中不同头部，方法，状态码的含义。
+ 协议的使用场景：DNS 协议，ARP 协议，SSH 命令的使用场景以及原理。
+ 不同协议的区别：TCP 与 UDP 的区别，HTTP 与 HTTPS 的区别。
+ 协议具体功能实现：TCP 三次握手原理，TCP 慢启动以及滑动窗口的原理与实现方式。

5.操作系统

+ 操作系统基础概念：进程，线程，虚拟内存，文件权限，信号量等概念考察。
+ Shell 的基础使用：ls, find, top, ps 等命令的应用与原理。
+ 常见功能的实现：进程调度，用户态与内核态的切换，系统调用的实现方式，select, epoll 的实现以及区别。
+ 常用函数的实现：memcpy，strcpy，strstr 等常用库函数的实现方式与优化。

6.数据库

+ 基本概念：事务，存储引擎，隔离级别，索引等概念考察。
+ 数据库基准测试与性能分析：基准测试的策略，性能剖析工具的使用和分析。
+ 数据库设计与优化：范式和反范式，索引的类型，如何选择合适的索引，如何使用硬件优化。
+ 数据库原理及高级特性：数据库分区分表，存储过程的使用，全文索引，主从热备等常用策略的使用与原理。

### 项目经验

#### 公司的项目

简历中提到的项目需要认真回顾一遍，常见的问题包括：

+ 这个项目遇到了什么技术难点，你是如何解决的
+ 这个项目你学习到了哪些技术知识点

即使你在项目做的只是一些看起来非常基础的工作，例如数据库的增删改查。你也可以通过在复习的时候深挖原理以及要点来展现能力，例如写入数据的时候如何防止丢失，如何保持最终一致性，要扩展的话如何分库分表？会增删改查的虽然多，但是懂得上面问题答案的求职者却很少，有了这些过硬的基础知识，面试官可能也会对你刮目相看。由于每个人对于每个系统的理解都不一样，涉及的业务以及遇到的问题也各不相同，所以面试中引导面试官向自己熟悉的技术点提问也非常重要了，具体的简历写法可以参考程序员如何写一份更好的简历。

#### 参与的开源项目或自己写的项目

如果有参与过一些著名的开源项目那么当然是很好，曾经阅读过它的源码，知道一些核心功能的实现原理也是加分项。自己的项目最主要是突显技术能力，能够说出自己遇到的一些问题是如何通过分析来解决以及优化的话，能够极大地提高印象分。

### 模拟面试

如果你准备去面试一家心仪的公司，那么面试之前，你应该先进行模拟面试，模拟面试的意思是让另外一名工程师充当面试官，按照目标公司的题库以及面试流程对你进行面试，最后把面试过程中的优点和缺点反馈给你。你既可以让有面试经验的朋友来进行模拟面试，也可以选择我们的求职课程或者以英文为沟通语言的 Pramp 来进行模拟。

## 面试阶段

无论你的履历如何出众，都不能对面试掉以轻心。像我一开始说的，根据不同的公司以及部门，面试的流程都不一样，这里我以最常见的流程举例：

+ HR 电话确认
+ 电话面试
+ 家庭作业
+ 现场面试
+ 非技术问题
+ 求职者提问

### HR 电话确认

HR 会和你聊下天，确保你了解这个岗位的基本信息。问几个关于你简历的问题，这轮只是考核下你的基础信息是否正确, 谈吐是否正常, 对自己简历是否了解, 这轮放轻松，按照简历实话实话就好。通过之后 HR 会通过邮件和你约定技术面试的时间，方式和使用的工具。

### 远程面试

这是技术面试的第一轮，公司会使用 Teams 进行视频面试，根据不同情况也会使用不同的在线写代码的平台。确保自己在面试前知道怎么用这些工具并测试网络的稳定是重要的一步。面试官可能会通过电话或者视频问一些技术问题，也可能把算法题目发在在线文档，然后让你去解决。如果遇到难的不会做也不需要太担心，尽量提供解题的思路，把自己知道的都说出来。即使最后不能 bug free，起码也能向面试官证明你的实力。

### 家庭作业

这轮并不常见，有的公司会让你实现一个小模块或者小工具。主要考核你实际情况下的开发能力。这点就要靠平时积累了，如何设计 API，使用什么设计模式，都有讲究。维护好的 commit messages 文档，简单的测试用例也非常重要。平时可以多看看开源项目源码，Python 的话我推荐看 Requests 源码，常用而且简单易懂。

### 现场面试

因为每位求职者的项目经验都不同，无法给出一致的答案。

### 非技术问题

接下来面试官可能会问一些非技术的问题：

+ 你曾经面临最大的专业挑战是什么？你是怎么战胜它的？
+ 是什么为什么你选择离开你现任公司？你从你上一家公司学到最重要的是什么？
+ 你的长期工作目标是什么？
  + 可以先考虑未来三年自己的定位，我遇过很多求职者没有长远规划以及目标，代表他们并没有认真思考自己的未来，这往往不是一位优秀工程师的兆头

### 求职者提问

这点非常重要，要预防你到了新公司之后，发现公司文化并不适合你。

+ 这个职位空缺的原因？
+ 公司如何保证人才不流失？
+ 如果我入职的话，会有入职培训吗？会被分到哪个项目组，项目组的成员构成是怎样？
+ 我入职的前三个月，要完成什么工作来证明我的能力呢，试用期的时间和通过标准？
+ 我今天面试的表现怎样，如果通过之后我还会经过多少轮，怎样的面试流程？

## 总结阶段

一次面试过来，可能筋疲力尽了。回想下自己哪里可以做得更好，简历哪里可以修改的。统计学告诉我们不要选择第一家面试的公司，多面试几家。不要欺骗自己，认真去思考每家的优点和缺点，和你的好朋友聊聊，寻求他们的建议。如果没有拿到 Offer 也没关系，重复上面的步骤，相信自己，努力和汗水总会能得到回报的。
