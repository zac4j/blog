---
title: "Generics in Kotlin"
date: 2020-08-14T07:27:09+08:00
keywords: ["generic", "in and out"]
description: "Generics in Kotlin world"
tags: ["generic"]
draft: true
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

[erasure]:https://kotlinlang.org/docs/reference/typecasts.html#type-erasure-and-generic-type-checks
[sp]:https://kotlinlang.org/docs/reference/generics.html#star-projections
