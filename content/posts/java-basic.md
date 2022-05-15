---
title: "Java Basic"
date: 2020-07-03T09:28:31+08:00
description: "Java Fundamental"
tags: ["java", "fundamental"]
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
  
## 2. String、StringBuffer、StringBuilder 有什么区别？

### 经典回答

+ String 是典型的 Immutable 类，被声明为 *final* class，所有属性也都是 final 的，因此类似拼接、裁剪字符串的操作都会产生新的 String 对象，过多的进行这种操作对应用性能有影响。
+ StringBuffer 为了解决 String 拼接产生对象而提供的类，我们可以用 append 或 add 方法，进行字符串的拼接，StringBuffer 本质是线程安全的序列，它保证了线程安全，也随之带来额外的性能开销，所以除非有线程安全的需要，不然推荐使用 StringBuilder。
+ StringBuilder 是 Java 1.5 中新增的，在能力上和 StringBuffer 没有本质区别，但是它去掉了线程安全的部分，有效减小了开销，是绝大部分情况下进行字符串拼接的首选。
+ 使用场景：并发必选 StringBuffer，迭代必选 StringBuilder，普通场景选 String，避免中途不必要的类型转换开销

### 引申问题

String 为什么设计成final的？

+ 安全性
  + 线程安全，Immutable 天生线程安全
  + String常被用作HashMap的key，如果可变会引有安全问题，如两个key相同 （3）
  + String常被用作数据库或接口的参数，可变的话也会有安全问题
+ 效率
  + 通过字符串池可以节省很多空间
  + 每个String对应一个hashcode，再次使用的话不用重新计算

### 知识扩展

+ String 是 Immutable 类的典型实现，原生的保证了基础线程安全，因为你无法对它内部数据进行任何修改，这种便利甚至体现在拷贝构造函数中，由于不可变，Immutable 对象在拷贝时不需要额外复制数据。
+ 为了实现修改字符串序列的目的，StringBuffer 和 StringBuilder 底层都是利用可修改的（char，JDK 9 以后是 byte）数组，两者都继承自 AbstractStringBuilder，实现的方法区别就是是否加了 synchronized.
  
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

## 5. 强引用、软引用、弱引用、虚引用有什么区别？具体使用场景是什么？

### 经典回答

不同的引用类型，主要体现的是**对象不同的可达性（reachable）状态**和**对垃圾收集的影响**。

Java 中引用类型分为 4 类：强引用、软引用、弱引用、虚引用

+ 强引用：通过 new 对象创建的引用，GC 即使内存不足也不会回收强引用
+ 软引用：软引用通过 SoftReference 实现，它的生命周期比强引用短，在内存不足，抛出 OOM 之前，GC 会回收软引用的对象。
  + **不建议使用软引用做缓存**，运行时并没有足够的信息确定哪些引用需要保留，哪些需要清除，引用对象清除太早造成不必要的工作，引用清除太晚造成浪费内存。
  + 大多数 App 需要使用 LruCache 取代软引用。LruCache 具有有效的逐出策略，并允许用户调整分配的内存量。
+ [弱引用][weak-ref]：弱引用通过 WeakReference 实现，它的生命周期比软引用短，GC 只要扫描到弱引用就会回收。App 中的用途是解除一些强引用的场景，如 Handler 对 Activity 的引用。
+ 虚引用：虚引用通过 PhantomReference 实现，它的生命周期最短，随时可能被回收。如果一个对象被虚引用引用，那么我们无法通过虚引用来访问这个对象的任何属性和方法。

## 6. Exception 和 Error 的异同

### 经典回答

Exception 和 Error 都是继承了 Throwable 类，在 Java 中只有 Throwable 类型的实例才可以被抛出（throw）或者捕获（catch），它是异常处理机制的基本组成类型。

Exception 和 Error 体现了 Java 平台设计者对不同异常情况的分类。Exception 是程序正常运行中，可以预料的意外情况，可能并且应该被捕获，进行相应处理。Error 是指在正常情况下，不大可能出现的情况，绝大部分的 Error 都会导致程序（比如 JVM 自身）**处于非正常的、不可恢复状态**。既然是非正常情况，所以不便于也不需要捕获，常见的比如 OutOfMemoryError 之类，都是 Error 的子类。

Exception 又分为可检查（checked）异常和不检查（unchecked）异常，可检查异常在源代码里必须显式地进行捕获处理，这是编译期检查的一部分。前面我介绍的不可查的 Error，是 Throwable 不是 Exception。不检查异常就是所谓的运行时异常，类似 NullPointerException、ArrayIndexOutOfBoundsException 之类，通常是可以编码避免的逻辑错误，具体根据需要来判断是否需要捕获，并不会在编译期强制要求。

### 一些常见的 Exception 和 Error

![ex-er](/img/exception-error.webp)

### 引申问题

+ NoClassDefFoundError 和 ClassNotFoundException 有什么区别？
  + NoClassDefFoundError是一种Error，Error在大多数情况下代表无法从程序中恢复的致命错误，产生的原因在于JVM或者ClassLoader在运行时类加载器在classpath下找不到需要的类定义（编译期是可以正常找到的，所以和ClassNotFoundException不同的是这是一个运行期的Error），这个时候虚拟机就会抛出NoClassDefFoundError，通常造成该ERROR的原因是打包过程中漏掉了部分类，或者 jar 包出现损坏或篡改，对应的 Class 在 classpath 中不可用等等原因。解决这个问题的办法是查找那些在开发期间存在于类路径下但在运行期间却不在类路径下的类。
  + ClassNotFoundException是属于Exception的运行时异常，因此是可检查异常；大多是可以从代码中恢复的异常类型，导致该异常的原因是因为java支持反射方式在运行时动态加载类，如使用Class.forName()/ClassLoader.loadClass()/ClassLoader.findSystemClass()方法动态的加载类信息，但是这个类在类路径中并没有被找到，那么就会在运行时抛出 ClassNotFoundException。解决该问题需要确保所需的类连同它依赖的包存在于类路径中，常见问题在于类名书写错误。另外还有一个导致ClassNotFoundException的原因就是：当一个类已经某个类加载器加载到内存中了，此时另一个类加载器又尝试着动态地从同一个包中加载这个类。通过控制动态类加载过程，可以避免上述情况发生。

## 7. 谈谈 final、finally、finalize 有什么不同

### 经典回答

final 可以用来修饰类、方法、变量，分别有不同的意义，final 修饰的 class 代表不可以继承扩展，final 的变量是不可以修改的，而 final 的方法也是不可以重写的（override）。

finally 则是 Java 保证重点代码一定要被执行的一种机制。我们可以使用 try-finally 或者 try-catch-finally 来进行类似关闭 JDBC 连接、保证 unlock 锁等动作。

finalize 是基础类 java.lang.Object 的一个方法，它的设计目的是保证对象在被垃圾收集前完成特定资源的回收。finalize 机制现在已经不推荐使用，并且在 JDK 9 开始被标记为 deprecated。

### 引申问题

+ 使用 final 有哪些好处？
  + 我们可以将方法或者类声明为 final，这样就可以明确告知别人，这些行为是不许修改的。
  + 使用 final 修饰参数或者变量，也可以清楚地避免意外赋值导致的编程错误，甚至，有人明确推荐将所有方法参数、本地变量、成员变量声明成 final。
  + final 变量产生了某种程度的不可变（immutable）的效果，所以，可以用于保护只读数据，尤其是在并发编程中，因为明确地不能再赋值 final 变量，有利于减少额外的同步开销，也可以省去一些防御性拷贝的必要。
  + 很多文章或者书籍中都介绍了 final 可在特定场景提高性能，比如，利用 final 可能有助于 JVM 将方法进行内联，可以改善编译器进行条件编译的能力等等

+ finally 推荐使用 try-with-resources 语句，try(java.lang.AutoCloseable 的实现类)中的资源会被自动关闭，并且将先关闭后声明的资源。

## 8. 谈谈 Java 反射机制，动态代理是基于什么原理

### 经典回答

反射机制是 Java 语言提供的一种基础功能，赋予程序在运行时自省（introspect）的能力。通过反射我们可以直接操作类或者对象，比如获取某个对象的类定义，获取类声明的属性和方法，调用方法或者构造对象，甚至可以运行时修改类定义。

动态代理是一种方便运行时动态构建代理、动态处理代理方法调用的机制，很多场景都是利用类似机制做到的，如用来包装 RPC 调用、面向切面的变成（AOP）。

实现动态代理的方式很多，比如 JDK 自身提供的动态代理，就是利用了反射机制。还有其他的实现方式，如据说高性能的字节码操作机制，类似 ASM、cglib、Javassist 等。

## volatile

一般提到 volatile，就不得不提内存模型相关的概念。

我们知道，在程序运行过程中，每条指令都是 CPU 执行的，而指令的执行过程中，势必涉及到数据的读取和写入。程序运行中的数据都存放在主存（Main Memory）中，这样就会出现一个问题，由于 CPU 的执行速度要远高于主存的读写速度，所以直接从主存中读取数据会降低 CPU 的效率。为了解决这个问题，就有了高速缓存（Cache Memory）的概念，

**Minimum Cache Configuration** ![memory-cache](/img/cpumemory.1.png "Minimum Cache Configuration")

### 一致性问题

引入 CPU 高速缓存后，提升了效率，但同时也会引发缓存与主存不一致的问题。对于单核 CPU 来说，处理比较简单，通常有一些两种方式：

+ 通写法（Write Through）：每次 CPU 修改了缓存内容，**立即更新**到内存，这也意味着每次 CPU 写共享数据，都会导致总线事务。
+ 回写法（Write Back）：每次 CPU 修改了缓存数据，**不会立即更新**到内存，而是等到某个合适的时机才会更新到内存中。

**Processor with Level 3 Cache** ![cpumemory](/img/cpumemory.2.png "Processor with Level 3 Cache")

多核 CPU 存在多个一级缓存，如何保证这些缓存以及内存之间的一致性呢？

**Multi processor, multi-core, multi-thread** ![cpumemory](/img/cpumemory.3.png "Multi processor, multi-core, multi-thread")

首先在通写法和回写法之外，又引入了两种操作：

+ 写失效：当一个 CPU 修改了数据，如果其他 CPU 有该数据，则通知其为无效。
+ 写更新：当一个 CPU 修改了数据，如果其他 CPU 有该数据，则通知其更新数据。

另外在 CPU 层面，提供了两种解决方案：

+ 总线锁：在多 CPU 情况下，某个 CPU 对共享变量操作时，在总线上发出一个 #LOCK 信号，总线把 CPU 和内存之间的通信锁住了，其他 CPU 不能操作该内存地址的数据。
+ 缓存锁：降低锁的粒度，基于缓存一致性协议来实现。

缓存一致性协议需要满足以下两种特性：

+ 写传播（Write propagation）：一个 CPU 对于某个内存位置所做的写操作，对于其他 CPU 是可见的，写传播可分为以下两种方式：
  + 嗅探（Snooping）：广播机制，即要监听总线上的所有活动
  + 基于目录（Directory-based）：点对点，总线事件只发送给相应的 CPU（利用 directory）
+ 写串行化（Write Serialization）：对同一内存单元的所有写操作都能串行化。即所有的处理器能以相同的次序看到这些写操作
  + 总线上任意时间只能出现一个 CPU 的写事件，多核并发的写事件会通过总线仲裁机制将其转换成串行化的写事件序列。

我们常提到的缓存一致性协议通常是指：MESI 协议

### MESI 协议

MESI 指缓存行的四种状态 —— Modified, Exclusive, Shared, Invalid

+ Modified: 当前 CPU 缓存有最新数据，其他 CPU 拥有失效数据，当前 CPU 数据与内存不一致，但以当前 CPU 数据为准
+ Exclusive: 只有当前 CPU 有数据，其他 CPU 没有该数据，当前 CPU 数据与内存数据一致
+ Shared: 当前 CPU 与其他 CPU 拥有相同的数据，并与内存中数据一致
+ Invalid: 当前 CPU 数据失效，其他 CPU 数据可能有可能无，数据应从内存中读取，且当前 CPU 与内存数据不一致。

MESI 中，依赖总线嗅探机制，整个过程是串行的，可能会发生阻塞。

### JMM

在 JVM 层面，定义了一种抽象的内存模型（JMM）用来解决可见性和有序性问题。

### Java 内存模型

JMM 属于语言级别的抽象内存模型，它定义了在共享内存中多线程读写的操作规范。JMM 将底层的问题抽象到 JVM 层面，基于 CPU 层面提供的**内存屏障指令**，以及**限制编译器重排序**，来解决 CPU 多级缓存、处理器优化、指令重排导致的并发问题。

JMM 决定了一个线程对共享变量的写入何时对另一个线程可见，如图：

这就导致了对共享变量的缓存一致性的问题，那么为了解决这个问题，提出了缓存一致性协议：多 CPU 情况下，某个 CPU 对共享变量操作时，在总线上发出一个 `#LOCK` 信号，总线把 CPU 和内存之间的通信锁住了，其他 CPU 不能操作该内存地址的数据。

在 Java 多线程开发中，有 3 个重要概念，原子性，可见性，有序性。

+ 原子性：一个或多个操作要么不执行，要么都执行。
+ 可见性：一个线程中对共享变量（类中的成员变量或静态变量）的修改，在其他线程立即可见。
+ 有序性：程序执行的顺序按照代码顺序执行。

把一个变量声明为 volatile，其实就保证了可见性和有序性。可见性我上面已经说过了，对于有序性可以展开说下，为了执行效率，有时候会发生指令重排序，这在单线程中指令重排序后的输出和我们代码逻辑输出还是一致的。但在多线程中就可能发生问题，volatile 在一定程度上可以避免指令冲排序。

volatile 的原理是在生成的汇编代码中多了一个 #LOCK 前缀指令，这个指令相当于一个内存屏障，这个内存屏障有3个作用：

+ 确保指令重排的时候不会把屏障后的指令排在屏障前，确保不会把屏障前的指令排在屏障后。
+ 修改缓存中的共享变量后立即刷新到主存中
+ 当执行写操作时会导致其他 CPU 中的缓存无效。

[weak-ref]:https://community.oracle.com/blogs/enicholas/2006/05/04/understanding-weak-references
[cache]:https://lwn.net/Articles/252125/
