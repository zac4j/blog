---
title: "Java Basic"
date: 2020-07-03T09:28:31+08:00
description: "Java Fundamental"
tags: ["java"]
categories: ["java", "basic"]
draft: false
---

🌟答案的组织策略：知道是什么，知道为什么，知道怎么用

## 0.JDK 和 JRE 有什么区别？

+ JDK 包含 JRE，同时还包含编译 Java 源码的编译器 javac，还包含了很多 Java 程序调试和分析的工具。
+ 加分项：除了 javac 还了解哪些命令行工具，它们的用途是什么？
  + jcmd：综合工具
  + jps：虚拟机进程状况工具
  + jinfo：虚拟机配置信息工具
  + jstat：虚拟机统计信息监视工具
  + jmap：Java 内存映像工具
  + jhat：虚拟机堆转储快照分析工具
  + jstack：Java 堆栈追踪工具
+ 加分项jstat 用过吗，有哪些参数？
  + -class：监视类装载、卸载数量、总空间和类装载所耗费的时间
  + -gc：监视 Java 堆的状况，包括 Eden 区，两个 survior 区，老年代、永久代等的容量，已用空间、GC 时间合计等统计

## 1.equals 和 == 的差别？

两者都是判断等价性，区别要从入参类型来看：

+ 对于基本类型，他们是比较值是否相等
+ 对于引用类型，他们是判断引用的是否为同一对象

+ 考察点：equals 的概念
+ 🌟实际要求：平时对源码的深挖意识即技术钻研和**批判性思维**
+ 考察目的：
  + 基础知识的扎实程度
  + 候选人对技术的热情
  
## 2.Java 中操作字符串有那些类？它们之间有什么区别？

+ 主要有：String，StringBuilder，StringBuffer
+ 区别：StringBuffer 和 StringBuilder 都继承自 AbstractStringBuilder，StringBuffer 具备线程安全性
  + **加分项**：String 源码分析
    + String 是 *final* 关键字修饰 -> 对象的值不可变 -> 每次操作都会生成新的对象
    + StringBuffer 和 StringBuilder 对象的值可变，但拼接字符串开销比较小
  + **加分项**：StringBuffer 线程安全性分析（查源码、找 synchronized、线程锁）
+ 使用场景：并发必选 StringBuffer，迭代必选 StringBuilder，普通场景选 String，避免中途不必要的类型转换开销
  
## 3.HashMap、HashTable、TreeMap 有什么区别？
是什么：

+ 上面3种数据结构都是最常见的 Map 接口的实现，是以 key-value 的形式存储和操作数据的容器类型。
+ HashTable 是 Java 类库提供的一个 hash 实现，本身是同步的，不支持 null key 和 null value，由于同步性能开销，所以已经很少推荐使用了。
+ HashMap 是应用广泛的哈希表实现，行为上大致与 HashTable 一致，主要区别在于 HashMap 不是同步的，支持 null key 和 null value 等。通常情况下，HashMap 进行 get 和 put 操作可以达到**常数时间的性能**。所以大部分利用键值对存取场景的首选。
+ TreeMap 则是基于红黑树的一种提供顺序访问的 Map，它的 get、put、remove 之类的操作都是 O(logn) 时间复杂度，具体顺序由可以指定的 Comparator 来决定，后者根据键的自然顺序来判断。

## 4.内部类和静态嵌套类

### Inner class vs. Static nested class

``` java
class OuterClass {
  ...
  static class StaticNestedClass {
    ...
  }
  class InnerClass {
    ...
  }
}
```

> 嵌套类（Nested class）可以分为两类：静态和非静态。声明为静态的嵌套类称为静态嵌套类（static nested classes）。非静态嵌套类称为内部类（inner classes）。

嵌套类是其封闭类（enclosing class）的成员。内部类可以访问封闭类的其他成员，即使它们被声明为私有的可以访问。静态嵌套类无权访问封闭类的其他成员。作为 OuterClass 的成员，嵌套类可以声明为 *private*, *public*, *protected* 或 *package private*.

### Why Use Nested Classes

+ 这是一种对仅在一个地方使用的类进行逻辑分组的方法
  
  如果一个类B仅对其他一个类A有用，那么将B嵌入类A将更合乎逻辑，嵌套将使 package 更精简。
+ 增加封装
  
  对于同一层级的类A和类B，如果类B需要访问类A的成员，通常情况需要实例化A再调用A的成员。通过将B封装到A内，A的成员可以声明为 *private*，同时B可以调用A的成员。此外B对外界完全隐藏。
+ 可以增加代码可读性，便于维护
  
  嵌套类可以使它更靠近使用的位置。

### 静态嵌套类

与类的方法和变量一样，静态嵌套类与外部类相关联。与静态方法一样，静态嵌套类不能直接引用封闭类中定义的实例变量或方法：它只能通过对象的引用来使用它们。

> 静态嵌套类和它外部类的实例成员进行交互，就像对其他 top-level 类一样。实际上，静态嵌套类在行为上是顶级类，为了包装方便（for packaging convenience），该顶级类已嵌套在另一个顶级类中。

创建静态嵌套类通过：

``` java
OuterClass.StaticNestedClass sKlass = new OuterClass.StaticNestedClass()
```

### 内部类

和实例方法及变量一样，内部类与其所在类的实例相关联，并且可以直接访问该对象的方法和字段。另外，由于内部类与实例相关联，因此它本身不能定义任何静态成员。

内部类的实例只能存在于外部类实例中，且可以直接访问其外部类实例的方法和字段。

``` java
OuterClass outer = new OuterClass();
// Instantiate inner class
OuterClass.InnerClass inner = outer.new InnerClass();
// Instantiate static nested class
OuterClass.NestedClass nested = new OuterClass.NestedClass();
```

#### 局部内部类和匿名内部类

局部内部类声明在别的类的实例方法内，匿名内部类与局部内部类相似，但写法是返回一次性对象的表达式。

``` java
public class AnonymousClasses {

    interface HelloWorld {
        public void greet();
        public void greetSomeone(String someone);
    }

    public void sayHello() {
      // Local inner class
        class EnglishGreeting implements HelloWorld {
            String name = "world";
            public void greet() {
                greetSomeone("world");
            }
            public void greetSomeone(String someone) {
                name = someone;
                System.out.println("Hello " + name);
            }
        }

        HelloWorld englishGreeting = new EnglishGreeting();

        // Anonymous class
        HelloWorld frenchGreeting = new HelloWorld() {
            String name = "tout le monde";
            public void greet() {
                greetSomeone("tout le monde");
            }
            public void greetSomeone(String someone) {
                name = someone;
                System.out.println("Salut " + name);
            }
        };
    }
}
```

### 序列化

强烈不建议对内部类，包括本地和匿名类进行序列化。当 Java 编译器编译某些构造时，它将创建[合成构造][sc]（synthetic constructs）。这些是类、方法、字段和其他在源码中没有对应构造方法的构造。合成构造可以使 Java 编译器无需更改 JVM 即可实现新的 Java 语言功能。但是，合成构造在不同的 Java 编译器中的实现可能有所不同，这意味着 .class 文件在不同的实现中也可能有所不同。因此，如果序列化一个内部类，然后使用其他 JRE 实现对其进行反序列化，则可能出现兼容性问题。

[sc]:https://docs.oracle.com/javase/tutorial/reflect/member/methodparameterreflection.html#implcit_and_synthetic

## 5.[Reference][ref]

> Package java.lang.ref 描述：
> 提供引用-对象类，该类支持与*垃圾回收器进行有限程度的交互*。程序可以使用引用对象来维护对某个对象的引用，这样垃圾回收器可以收集后者的对象。垃圾回收器在确定给定对象的可达性更改后会通知程序。

引用对象封装了对某个其他对象的引用，因此该引用本身可以像其他任何对象一样进行检查和操作。引用对象提供三种类型，引用依次减弱：soft, weak, 和 phantom，每种类型都对应不同级别的可达性。软引用用于实现内存敏感型缓存**（Android SDK 不建议这样做）**，弱引用用于实现规范化映射，这些映射不会阻止回收其键或值。虚引用用于安排比 Java 终止机制更灵活的事前清理行动。

### StrongReference

Strong 是描述它们如何与 garbage collector 进行交互的。如果一个对象通过一系列强引用访问（强可达性），那个该对象不符合垃圾回收的条件。

### SoftReference

+ 在虚拟机引发 OutOfMemoryError 之前，确保清除所有软引用的对象。
+ **不建议使用软引用做缓存**，运行时并没有足够的信息确定哪些引用需要保留，哪些需要清除，引用对象清除太早造成不必要的工作，引用清除太晚造成浪费内存。
+ 大多数 App 需要使用 LruCache 取代软引用。LruCache 具有有效的逐出策略，并允许用户调整分配的内存量。

### WeakReference

弱引用可以利用垃圾回收器的能力来确定可达性，因此我们不必自己做。

[ref]:https://community.oracle.com/blogs/enicholas/2006/05/04/understanding-weak-references
