---
title: "Use Hilt in Android"
date: 2020-08-06
draft: true
--- 

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

`@HiltAndroidApp` 会

[cl]:https://developer.android.com/codelabs/android-hilt?return=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fandroid-week6-jetpack%23codelab-https%3A%2F%2Fdeveloper.android.com%2Fcodelabs%2Fandroid-hilt#3