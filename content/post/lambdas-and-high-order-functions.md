---
title: "Lambdas and High-order Functions"
date: 2020-08-09
keywords: ["lambda", "high-order function"]
description: "Lambdas and high-order functions introduction"
tags: ["lambda", "high-order function"]
draft: false
--- 

除了传统命名的函数外，Kotlin 还支持 [lambdas][la]. *lambda* 是组成函数的表达式，一个没有名称的函数。lambda 表达式可以作为数据传递。在其他语言中，lambda 被称为 *匿名函数（anonymous function）*，*函数字面量（function literals）* 或类似名称。

<!--more-->

### High-order functions

我们可以通过传递 lambda 到另一个函数，来创建 *高阶函数（high-order function）*。`map` 是一个高阶函数，你传入的 lambda 表达式是需要 *应用的转换（transformation to apply）*。

### Create lambdas

与命名函数一样，lambda 也可以有入参。例如：

``` kotlin
val waterFilter = { dirty: Int -> dirty / 2 }
```

位于 `->` 左侧的为入参及入参类型，`->` 右侧即为需要执行的代码。将 lambda 表达式赋值给变量后，我们就可以像函数一样调用它。

``` kotlin
println(waterFilter(20))

// print => 10
```

Kotlin 的函数语法与 lambdas 语法紧密相关，我们可以使用这种语法明确声明一个包含函数的变量：

``` kotlin
val waterFilter: (Int) -> Int = {dirty -> dirty / 2}
```

### Create high-order function

我们可以使用 lambdas 来创建高阶函数，即函数的入参是另一个函数：

``` kotlin
fun updateDirty(dirty: Int, operation: (Int) -> Int): Int {
    return operation(dirty)
}
```

我们创建了一个 `updateDirty` 函数，这个函数的第二个参数是另一个函数 `operation`，`operation` 函数接收一个 `Int` 类型的入参并返回 `Int`类型的值。函数体将第一个参数作为入参，调用了传入的函数。

对于 `updateDirty` 函数的使用，我们可以 a).创建一个 lambda 传入，如：

``` kotlin
val waterFilter: (Int) -> Int = {dirty -> dirty / 2}
println(updateDirty(20. waterFilter))
```

除了创建 lambda 外，我们还可以 b).传入常规的命名函数，只是与直接 lambda 变量不同，我们需要使用 `::` 操作符来传入命名函数的引用。

``` kotlin
fun waterFilter(dirty: Int): Int = dirty / 2
println(updateDirty(40, ::waterFilter))
```

这种单行的函数也称为 *compat functions* 或 *single-expression functions*，可以增加代码可读性。

### Last parameter call syntax

Kotlin 倾向于将带有函数的任意参数作为最后一个入参。在使用高阶函数时，Kotlin 有一种特殊的语法，称为 [last parameter call syntax][lpcs] 可以使代码更加简洁。这种情况下，我们可以传入一个 lambda 而无需放入括号内：

``` kotlin
var dirty = 60
dirty = updateDirty(dirty) { dirty -> dirty / 2 }
println(dirty)
```

[la]:https://kotlinlang.org/docs/reference/lambdas.html
[cl]:https://codelabs.developers.google.com/codelabs/kotlin-bootcamp-functions/#6
[lpcs]:https://kotlinlang.org/docs/reference/lambdas.html#passing-a-lambda-to-the-last-parameter
