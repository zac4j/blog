---
title: "Generics in Kotlin"
date: 2020-08-14T07:27:09+08:00
keywords: ["generic", "in and out"]
description: "Generics in Kotlin world"
tags: ["generic", "kotlin"]
draft: false
---

val 和 var 与 variables 的 VALUES 相关，val 保护变量的值不变。in 和 out 与 variables 的 TYPES 相关，in 和 out 保证在范型的使用中，只有安全的类型才能传入或传出函数。

<!--more-->

## Define an out type

``` kotlin
/**
 * Define a WaterSupply class and verify if water needs processing.
 */
open class WaterSupply(var needsProcessing: Boolean)

/**
 * Define a TapWater class extends [WaterSupply] class.
 */
class TapWater: WaterSupply(true) {
    // We do water processing by add chemical cleaners.
    fun addChemicalCleaners() {
        needsProcessing = false
  }
}

/**
 * Define a Aquarium class, it accept water supply.
 */
class Aquarium<out T: WaterSupply>(val waterSupply: T) {
    ...
}

/**
 * Define a funtion that could add Aquarium item.
 */
fun addItem(aquarium: Aquarium<WaterSupply>) = println("item added")

/**
 * Test function
 */
fun g1() {
    val aquarium = Aquarium(TapWater())
    addItem(aquarium)
}
```

如果我们移除了 `class Aquarium<out T: WaterSupply>` 定义中的 `out` 关键字，`addItem` 函数就会报 Type mismatch error:

``` error
Type mismatch
Required: Aquarium<WaterSupply>
Found: Aquarium<TapWater>
```

## Define an in type

``` kotlin
/**
 * Define a Cleaner interface that takes a generic T that's constrained to WaterSupply.Since it is only used as an argument to clean(), we can make it an in paramter.
 */
interface Cleaner<in T : WaterSupply> {
  fun clean(waterSupply: T)
}

/**
 * Create a class TapWaterCleaner that implements Cleaner for cleaning TapWater by adding chemicals
 */
class TapWaterCleaner : Cleaner<TapWater> {
  override fun clean(waterSupply: TapWater) {
    waterSupply.addChemicalCleaners()
  }
}

/**
 * In the Aquarium class, update addWater() to take a Cleaner of type T, and clean the water before adding it.
 */
class Aquarium<out T: WaterSupply>(val waterSupply: T) {
  fun addWater(cleaner: Cleaner<T>) {
    if (waterSupply.needsProcessing) {
      cleaner.clean(waterSupply)
    }
    println("Water added!")
  }
}

/**
 * Test function
 */
fun g2() {
    val cleaner = TapWaterCleaner()
    val aquarium = Aquarium(TapWater())
    aquarium.addWater(cleaner)
}
```

Kotlin 将使用 in 和 out 类型来确保我们可以安全地使用泛型。out 和 in 的使用场景：out 类型可以作为返回值向外传递，in 类型可以作为参数向内传递。

## Type erasure and generic type checks

Java 中泛型只在编译时使用，编译器确保所有类型相关的操作都可以安全的进行。在运行时，泛型类型的实例并不持有实际类型的信息（[类型擦除][erasure]），比如 `List<Foo>` 会擦除为 `List<*>`。通常无法在运行时检查实例是否属于某种泛型类型，不过在 Kotlin 中我们可以通过 `inline + reified` 的组合实现运行时类型判断。

``` kotlin
class Aquarium<T : WaterSupply>(val waterSupply: T) {
  fun addWater(cleaner: Cleaner<T>) {
    if (waterSupply.needsProcessing) {
      cleaner.clean(waterSupply)
    }
    println("Water added!")
  }

  /*
   * Declare the function inline, and make the type reified.
   */
  inline fun <reified R : WaterSupply> isOfType(): Boolean = waterSupply is R
}
```

我们在 Aquarium 类中添加 `isOfType` 方法，通过 `inline` 声明方法，将泛型类型标记为 `reified`后，`is` 关键字可以在运行时做泛型类型检查。

### Make extension functions

``` kotlin
inline fun <reified T: WaterSupply> Aquarium<*>.isOfType() = waterSupply is T
```

[Star-projections][sp] 很像 Java 编程中的 raw types，不过更加安全。

## inline functions

Lambdas 和高阶函数都很有用，但我们应该了解一个事实：Lambda 是对象。Lambda 表达式是 `Function` 接口的实例，`Function` 接口是 `Object` 的子类。

``` kotlin
/**
 * 自定义 with 函数，接收 [name] 字符串，执行 name.block() 函数
 */
fun customWith(name: String, block: String.() -> Unit) {
  name.block()
}

/**
 * 调用 customWith 函数，并将 fish.name 首字母大写
 */
customWith(fish.name) {
  capitalize()
}
```

上面调用 `customWith` 函数实际的实现像这样：

``` kotlin
customWith(fish.name, object: Function1<String, Unit>) {
  override fun invoke(name: String) {
    name.capitalize()
  }
}
```

通常这不是问题，因为创建对象和调用函数不会产生太多的开销。不过假如 `customWith` 函数在非常多的地方都有调用，那么开销就会多起来。

Kotlin 提供 `inline` 作为处理这种情况的一种方式，通过增加编译器的工作来减少运行时的开销。将函数标记为 `inline` 意味着每次调用该函数时，编译器实际上会将源码转换为内联函数。

比如我们在使用 `customWith` 时标记为 `inline`：

``` kotlin
inline customWith(fish.name) {
  capitalize()
}
```

它转换后像这样：

``` kotlin
fish.name.capitalize()
```

需要注意的是，内联大型函数会增加代码大小，因此 `inline` 适用于多次使用的简单的函数，如 `customWith`。

## SAM

*Single Abstract Method(SAM)* 即表示只有一个方法的接口。在 Java 编写 API 时非常常见，比如 `Runnable` 接口，只定义了一个 `run()` 方法，`Callable` 接口，定义了一个 `call()` 方法。

我们定义个 SAM：

``` java
class JavaRun {
  public static void runNow(Runnable runnable) {
    runnable.run();
  }
}
```

在 Kotlin 中调用该函数：

``` kotlin
val runnable = object: Runnable {
  override fun run() {
    println("I'm a Runnable")
  }
}

JavaNow.runNow(runnable)
```

Kotlin 提供的 SAM 机制会提示缩减代码：

``` kotlin
JavaNow.runNow {
  println("Lambda as a Runnable")
}
```

SAM 的模版像这样：

``` kotlin
Class.singleAbstractMethod {lambda_of_override}
```

[erasure]:https://kotlinlang.org/docs/reference/typecasts.html#type-erasure-and-generic-type-checks
[sp]:https://kotlinlang.org/docs/reference/generics.html#star-projections
[ie]:https://codelabs.developers.google.com/codelabs/kotlin-bootcamp-sams/#5
