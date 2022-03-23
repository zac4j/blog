---
title: "Essential Design Pattern in Android"
date: 2020-06-20T20:22:10+08:00
description: "Design pattern in Android"
tags: ["design pattern"]
categories: ["android", "design pattern"]
draft: false
---

## 什么是设计模式

设计模式的[经典解释][ci]：软件工程中，设计模式是针对软件设计中给定上下文的常见问题的**通用、可复用的解决方案**。设计模式并非可以直接转换为源码或机器码的最终设计。相反，它是可以在不同场景下使用的解决问题的描述或模版。设计模式是程序员在设计 App 或系统时解决常见问题的最佳实践。

面向对象的设计模式通常展示了类和对象之间的关系和交互，而不是指定 App 的类或对象。

设计模式可以看作是一种编程的结构化方法，介于编程范式和具体算法之间。

设计模式根据其解决的问题分为三个类别：

+ 创造型模式（Creational Patterns）：提供创建类，对象的方法（Singleton, Dependency Inject, Builder等）。
+ 结构型模式（Structural Patterns）：提供对象和类之间组合的方法（Composite, Bridge, Adapter 等）。
+ 行为型模式（Bahavioral Patterns）：提供对象和类之间相互交流的方法（Command, Observer, Strategy 等）。

## 创造型模式

当我们用最基本的形式创建创建对象可能导致设计问题，或增加设计的复杂性。创造型设计模式通过**控制对象的创建方式**来解决这种问题。

Android 常用的创造型模式：

### Builder

Builder 模式主要解决创建复杂对象过程中遇到的问题：

+ 类如何创建复杂对象不同的表现形式(different presentations)
+ 如何简化包含创建复杂对象的类

Builder 模式在 Android 中常见的应用是 AlertDialog 的创建过程：

``` kt
AlertDialog.Builder(context)
 .setTitle("Builder Sample")
 .setMessage("Builder Nessage")
 .setNegativeButton("Cancel", {_,_ -> })
 .setPositiveButton("OK", {_, _ -> })
 .show()
```

Builder 模式通过这种方式解决问题：

+ 将复杂对象的各个部分的创建(creating)和组装(assembling)封装在一个独立的 Builder 对象中。
+ 类将对象的创建委托给 Builder 对象，而不是直接创建。

### Dependency injection

依赖注入（Dependency injection, 简称 Di）通常表示某个对象接收(receives)它依赖的对象(objects)的技术，这些依赖的对象称为依赖项(dependencies)。在典型的使用关系中，接收者被称为客户端（client），传递的对象被称为服务（service）。将服务传递给客户端的代码可能有很多种，被称为注入器（injector）, 注入器提供给客户端需要使用哪些服务。注入(injection)是指将依赖项（服务）传递给使用依赖项的对象（客户端）。

依赖注入的基本要求是将服务传递给客户端而不是让客户端创建或找到该服务。客户端不需要了解注入器，注入器实现构造服务然后将服务注入（传递）到客户端，客户端仅需要了解如何使用该服务。依赖注入的意图是实现对象构造和使用的分离，这可以提高代码的复用和可读性。

依赖注入是[控制反转][ioc]（inversion of control, 简称 IoC）的一种技术形式，其他遵循控制反转原则的还有回调（callbacks），调度器（schedulers）等。

[cdp]:https://www.raywenderlich.com/470-common-design-patterns-for-android-with-kotlin
[ci]:https://en.wikipedia.org/wiki/Software_design_pattern
[ioc]:https://en.wikipedia.org/wiki/Inversion_of_control