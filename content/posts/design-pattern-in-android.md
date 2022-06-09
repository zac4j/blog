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

+ 创造型模式（Creational Patterns）：提供创建类，对象的方法，包括Factory/Abstract Factory, Singleton, Builder，原型模式 ProtoType 等。
+ 结构型模式（Structural Patterns）：提供对象和类之间组合的方法，组合模式 Composite, 桥接模式 Bridge, Adapter，装饰者模式 Decorator，代理模式 Proxy，外观模式 Facade，享元模式 Flyweight 等。
+ 行为型模式（Bahavioral Patterns）：提供对象和类之间相互交流的方法（命令模式 Command, 解释器模式 Interpreter，观察者模式 Observer, 策略模式 Strategy，迭代器模式 Iterator，模版方法模式 Template Method，访问者模式 Visitor 等）。

可以结合主流开源框架使用的设计模式展开分析

[cdp]:https://www.raywenderlich.com/470-common-design-patterns-for-android-with-kotlin
[ci]:https://en.wikipedia.org/wiki/Software_design_pattern
[ioc]:https://en.wikipedia.org/wiki/Inversion_of_control