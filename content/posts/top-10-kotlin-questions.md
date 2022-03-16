---
title: "Top 10 Kotlin Questions in StackOverflow"
date: 2020-06-19T22:03:29+08:00
keywords: ["Questions", "Kotlin"]
tags: ["kotlin"]
categories: ["kotlin"]
draft: false
---

#### IntArray vs. `Array<Int>`

+ In java code
  + IntArray: int[]
  + `Array<Int>`: Integer[]
+ Create method:
  + IntArray:
  
  ```kt
  val intArray : IntArray = intArrayOf(1, 2, 3, 4, 5)
  ```

  + `Array<Int>`:
  
  ```kt
  val arrayOfInts : Array<Int> = arrayOf(1, 2, 3, 4, 5)
  ```

#### Iterable vs. Sequence

+ Usage
  + Iterable:
  
  ```kt
  getPeople()
  .filter { it.age >= 18 }
  .map { it.name }
  .take(5)
  ```

  + Sequence: start with `asSequence()` and end with `toList()` or `toSet()`...
  
  ```kt
  getPeople().asSequence()
  .filter { it.age >= 18 }
  .map { it.name }
  .take(5)
  .toList()
  ```

+ Use for
  + Iterable: use for small collections by default, might faster than the overhead using a `Sequence`
  + Sequence: use for `very large` collections.

#### Iterate a collection

+ eg.1
  
    ```kt
    for(i in 0..args.size - 1) {
        println(args[i])
    }
    ```

+ eg.2, use Array's `lastIndex` extension property.
  
    ```kt
    for(i in 0..args.lastIndex) {
        println(args[i])
    }
    ```

+ eg., use `until` to create open ended range.
  
    ```kt
    for(i in 0 until args.size) {
        println(args[i])
    }
    ```

+ eg.4, use `indices` extension property.
  
    ```kt
    for(i in args.indices) {
        println(args[i])
    }
    ```

+ eg.5
  
    ```kt
    for (arg in args) {
        println(arg)
    }
    ```

+ eg.6, use `forEach` extension function.
  
    ```kt
    args.forEach { arg ->
        println(arg)
    }
    ```

+ eg.7, use `withIndex` function.
  
    ```kt
    for((index, arg) in args.withIndex()) {
        println("index: $index, arg: $arg")
    }
    ```

+ eg.8, use `forEachIndexed` functions.

    ```kt
    args.forEachIndexed { index, arg ->
        println("index: $index, arg: $arg")
    }
    ```

#### SAM conversion

> SAM conversions only work for interfaces, not for abstract classes

+ This code will work fine:
  
    ```kt
    button.setListener {
        println("Clicked!")
    }
    ```

+ The following will not work:

    ```kt
    /**
    * This gives the button a lambda that will create an anonymous instance of an OnClickListener
    * every time the button is clicked, which is then just thrown away (never assigned anywhere,
    * never invoked).
    */
    button.setListener {
        object: OnClickListener {
            override fun onClick(button: Button) {
                println("Clicked!")
            }
        }
    }

    /**
    * This one declares a useless local function in a very simliar fashion
    */
    button.setListener {
        fun onClick(button: Button) {
            println("Clicked!")
        }
    }
    ```

+ [SAM constructor][sam_constructor]

```kt
/**
 * If the interface is defined in Java, but the function that takes an instance of its defined in Kotlin,
 * try to use a SAM constructor.
 */
 button.setListener(OnClickListener {
    println("Clicked")
 })
```

+ Explanation
|               | Java interface  | Kotlin interface  |
|---------------|-----------------|-------------------|
| Java method   | SAM conversion  | Object expression |
| Kotlin method | SAM constructor | Object expression |

#### Static things

+ If your class has "non-static" parts as well, put the "static" parts in a companion object:

    ```kt
    class Foo {
        companion object {
            fun x() {...}
        }
        fun y() {...}
    }
    ```

+ If your class is completely static, replace `class` with a singleton `object` :

```kt
object Foo {
    fun x() {...}
}
```

+ And if you don't want to always scope your calls as `Foo.x()` but want to just use `x()` instead, use a top level function:
  
```kt
// file Foo.kt
fun x() {...}
```

+ For function, there has this table:
| Function declaration             | Kotlin usage | Java usage         |
|----------------------------------|--------------|--------------------|
| Companion object                 | Foo.f()      | Foo.Companion.f(); |
| Companion object with @JvmStatic | Foo.f()      | Foo.f();           |
| Object                           | Foo.f()      | Foo.INSTANCE.f();  |
| Object with @JvmStatic           | Foo.f()      | Foo.f();           |
| Top level function               | f()          | UtilKt.f();        |
| Top level function with @JvmName | f()          | Util.f();          |
+ For variable, here's the reference:
| Variable declaration                           | Kotlin usage | Java usage          |
|------------------------------------------------|--------------|---------------------|
| Companion object                               | X.x          | X.Companion.getX(); |
| Companion object with @JvmStatic               | X.x          | X.getX();           |
| Companion object with @JvmField                | X.x          | X.x;                |
| Companion object with const                    | X.x          | X.x();              |
| Object                                         | X.x          | X.INSTANCE.getX();  |
| Object with @JvmStatic                         | X.x          | X.getX();           |
| Object with @JvmField                          | X.x          | X.x;                |
| Object with const                              | X.x          | X.x;                |
| Top level variable                             | x            | ConstKt.getX();     |
| Top level variable with @JvmField              | x            | ConstKt.x;          |
| Top level variable with const                  | x            | ConstKt.x;          |
| Top level variable with @JvmName               | x            | Const.getX();       |
| Top level variable with @JvmName and @JvmField | x            | Const.x;            |
| Top level variable with @JvmName and const     | x            | Const.x;            |

#### 6.Smart cast on mutable properties

以下代码会报这种错：
> Kotlin: Smart cast to 'Toy' is impossible, because 'toy' is a mutable property that could have been changed by this time

```kt
class Dog(var toy: Toy? = null) {
    fun play() {
        if (toy != null) {
            toy.chew()
        }
    }
}
```

原因：
因为 mutable 的变量`Dog` 实例可能在`toy!=null`检查后在另一条线程上被修改，而在 `toy.chew()`执行时可能产生 NPE.

可以做以下修改：

+ 使用局部变量

```kt
class Dog(var toy: Toy? = null) {
    fun play() {
        val _toy = toy
        if (_toy != null) {
            _toy.chew()
        }
    }
}
```

+ 临时 let 操作符，使用范围更广泛：
  
```kt
class Dog(var toy: Toy? = null) {
    fun play() {
        toy?.let {
            it.chew()
        }
    }
}
```

+ ?. 操作符

```kt
class Dog(var toy: Toy? = null) {
    fun play() {
        toy?.chew()
    }
}
```

#### override Java 方法

假设 Java 中定义了接口 `OnClickListener`:

```java
public interface OnClickListener {
    void onClick(Button button)
}
```

在 Kotlin 中实现该接口，如果使用IDEA生成：

```kt
class KtListener: OnClickListener {
    override fun onClick(button: Button?): Unit {
        val name = button?.name ?: "Unknown button"
        println("Clicked $name")
    }
}
```

对于我们知道不可能为 `null` 的场景，可以这样写：

```kt
class KtListener: OnClickListener {
    override fun onClick(button: Button): Unit {
        val name = button.name
        println("Clicked $name")
    }
}
```

[article]:https://blog.autsoft.hu/top-10-kotlin-stack-overflow-questions/
[sam_constructor]:https://stackoverflow.com/questions/48284994/lambda-implementation-of-interface-in-kotlin
