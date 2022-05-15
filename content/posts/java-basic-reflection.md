---
title: "Java Reflection"
date: 2022-05-15T16:26:15+08:00
description: "Java Reflection a simple intro"
tags: ["reflect"]
categories: ["java"]
draft: true
---

反射机制是 Java 语言提供的一种基础功能，赋予程序在运行时自省（introspect，官方用语）的能力。通过反射我们可以直接操作类或者对象，比如获取某个对象的类定义，获取类声明的属性和方法，调用方法或者构造对象，甚至可以运行时修改类定义。

<!-- more -->

## Reflection Essential

### 创建对象的两种途径

常规方式：

import 包类名称 -> 通过 new 实例化 -> 取得实例化对象

反射方式：

通过包类名称获取 Class 对象 -> 通过 Class 对象创建新的实例

看看反射利用创建 *Date* 对象：

``` java
public Date createDate() {
    Class clazz = Class.forName("java.util.Date");
    return (Date)class.newInstance();
} 
```

### 理解 Class 类

``` java
public Date createDate() {
    return new Date();
}
```
