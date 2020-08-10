---
title: "Use Hilt in Android"
date: 2020-08-06
description: "Activity Hilt"
tags: ["hilt", "dagger", "di"]
categories: ["hilt", "dagger", "di"]
draft: true
--- 

> If you want to use dagger in  your project, you may have 2 or 3 different sources, different samples, different blog posts. All of them use a different set-up, and all of them use Dagger in a different way, so it's very difficult to understand all the topics and relate them together. So that's why Google create a common guidance, a common ground.

<!--more-->

## Hilt 中一些注解

### container

A *container* is a class which is in charge of providing dependencies in your codebase and knows how to create instances of other types of your app. It manages the graph of dependencies required to provide those instances by creating them and managing their lifecycle.

*container* 是一个类，负责在代码中提供依赖关系，并且知道如何创建其他类型的 App 实例。它通过创建依赖所需的实例并管理其生命周期来控制依赖关系图。

A container exposes methods to get instances of the types it provides. Those methods can always return a different instance or the same instance. If the method always provides the same instance, we say that the type is *scoped* to the container.

*container* 暴露提供这些实例的方法。这些方法始终可以返回不同或相同的实例。如果该方法始终提供相同的实例，则可以说这种类型属于 *container* 的*范围*。

### @HiltAndroidApp

为了添加一个依附 app 生命周期的 *container*，我们需要在 **Application** 类上添加 `@HiltAndroidApp` 注解：

``` kotlin
@HiltAndroidApp
class App:Application() {
    ...
}
```

`@HiltAndroidApp` 会触发 Hilt 生成 App 所使用的依赖注入的基类，*application container* 是 app 的父容器，这意味着别的 *containers* 可以访问它提供的依赖。

[cl]:https://developer.android.com/codelabs/android-hilt?return=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fandroid-week6-jetpack%23codelab-https%3A%2F%2Fdeveloper.android.com%2Fcodelabs%2Fandroid-hilt#3
[bp]:https://github.com/google/dagger/issues/900
