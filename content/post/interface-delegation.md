---
title: "Interface Delegation"
date: 2020-08-11T20:42:28+08:00
keywords: ["interface", "delegation"]
description: "Interface delegation introduction"
tags: ["interface", "delegation"]
draft: false
---

[接口委托（Interface delegation）][cl] 是一种"高级"技术，接口的方法由 helper 或 delegate 对象实现，然后给别的类使用。当我们在一系列不相关的类中使用接口时，这项技术会很有用：我们将需要使用的接口的函数添加到单独的 helper 类中，别的类使用 helper 类的实例来实现这个函数。

<!--more-->

## Define interface

``` kotlin
/**
 * Define fish action interface.
 */
interface FishAction {
    fun eat()
}

/**
 * Define fish color interface.
 */
interface FishColor {
    val color: String
}

/**
 * Plecostomus class implement [FishAction] and [FishColor] interface.
 */
class Plecostomus: FishAction, FishColor {
    override val color = "gold"
    override fun eat() {
        println("eat algae")
    }
}

/**
 * Shark class implement [FishAction] and [FishColor] interface.
 */
class Shark: FishAction, FishColor {
    override val color = "gray"
    override fun eat() {
        println("eat fish")
    }
}
```

## Make a singleton class

``` kotlin
/**
 * Creat a singleton for gold fish color.
 */
object GoldColor: FishColor {
    override val color = "gold"
}

/**
 * Create printing fish action.
 */
class PrintingFishAction(val food: String): FishAction {
    override fun eat() {
        println(food)
    }
}

/**
 * Plecostomus use interface delegation mechanism, and all overrides
 * are handled by interface delegation.
 */
class Plecostomus(fishColor: FishColor = GoldColor): FishAction by PrintingFishAction("eat algae"), FishColor by fishColor
```

## Summary

Interface delegation is powerful, and you should generally consider how to use it whenever you might use an abstract class in another language. It lets you use composition to plug in behaviors, instead of requiring lots of subclasses, each specialized in a different way.

接口委托很有用，通常每当我们在使用另一种语言的抽象类时都应该考虑使用它。它使我们可以使用组合来插入 behavior，而不是需要大量各有特色的子类。

[cl]:https://codelabs.developers.google.com/codelabs/kotlin-bootcamp-classes/#7
