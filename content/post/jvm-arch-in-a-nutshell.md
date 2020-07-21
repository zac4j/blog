---
title: "JVM Architecture in A Nutshell"
date: 2020-07-17
draft: true
---

## 跨平台的本质

跨平台的本质是因为汇编指令的不同.

## ClassLoader

### What is it

类加载器负责在运行时将 Java 类动态加载到 JVM，是 JRE 的一部分，由于有了类加载器，JVM 无需了解底层文件或文件系统即可运行Java程序。

类加载器不会一次将全部类加载到内存里，而是在程序需要时加载。

### Built-in class loaders types

+ Bootstrap class loader
  
  每个 JVM 实现必须有一个 bootstrap 类加载器。它加载 *JAVA_HOME/jre/lib* 目录下的核心 Java API 类，该路径通常被称为引导路径(bootstrap path)。该类加载器由 C/C++ 语言实现。
+ Extension class loader
  
  Bootstrap 的子类加载器，它加载扩展目录 *JAVA_HOME/jre/lib/ext* 或 **java.ext.dirs** 指定路径下的类。它由 Java 在 `sun.misc.Launcher$ExtClassLoader` 类中实现。
+ System/Application class loader
  
  Extension 的子类加载器。它负责从 app 路径加载类，它加载在 *classpath* 环境变量、*-classpath*、-cp 命令行选项中找到的文件。它同样由 Java 在 `sun.misc.Launcher$AppClassLoader` 类中实现。

### How it works

类加载器是 JRE 的一部分。当 JVM 请求一个类时，类加载器尝试定位该类，并使用*完全限定的类名(fully qualified class name)* 将*类定义(class definition)* 加载到运行时环境中。

`java.lang.ClassLoader.loadClass()` 方法负责将类定义加载到运行时环境中。如果请求的类尚未加载，则它将请求委派给父类加载器，此过程递归进行。如果父类加载器找不到该类，则子类加载器调用 `java.net.URLClassLoader.findClass()` 方法在文件系统中查找类。最终，如果子类加载器未找到该类，则它将抛出 `java.lang.NoClassDefFoundError` 或 `java.lang.ClassNotFoundException`。

### Class loader features

#### 代理模型（Delegation Model）

类加载器遵循代理模型，在该模型中，ClassLoader 会根据请求查找的类或资源，将搜索类或资源的工作委托给父加载器处理。仅当 bootstrap class loader 和 extension class loader 都未能加载该类时，system class loader 才会尝试自己加载该类。

#### 唯一类（Unique classes）

作为代理模型的结果，很容易确保唯一的类，因为我们始终向上委托。

#### 可见性（Visibility）

子类加载器对其父类加载器加载的类完全可见，父类加载器则对子类加载器加载的类不可见。

### Custom ClassLoader

在需要从本地或网络加载类的场景，我们可能需要使用自定义类加载器。

#### 使用场景

自定义类加载器不仅对在运行时加载类有帮助，还包括：

+ 修改现有字节码，如织入代理
+ 动态创建适合用户需求的类。如 JDBC 中，通过动态类加载完成不同驱动实现(driver implementations)之间的切换。
+ 实现类版本控制机制，如为具有相同名称和包名的类加载不同的字节码（热修复）。这可以通过 URL 类加载器或自定义加载器完成。

我们可以自定义类加载器实现指定类的加载：

``` kotlin
package com.zac4j.system.util

import java.io.ByteArrayOutputStream
import java.io.File
import java.io.IOException

/**
 * Custom ClassLoader sample
 *
 * @author: zac
 * @date: 2020/7/19
 */
internal object Test {
  @JvmStatic fun main(args: Array<String>) {
    val classLoader = CustomClassLoader()
    val clazz = classLoader.loadClass("com.zac4j.system.util.Utils")
    println("custom classloader:${clazz.canonicalName}")
  }
}

/**
 * 自定义 ClassLoader，加载指定类文件。
 */
class CustomClassLoader : ClassLoader() {
  override fun findClass(name: String): Class<*> {
    val b = loadClassFromFile(name)
    return defineClass(name, b, 0, b.size)
  }

  /**
   * 根据类名加载类文件
   */
  private fun loadClassFromFile(filename: String): ByteArray {
    val inStream = javaClass.classLoader?.getResourceAsStream(
        filename.replace('.', File.separatorChar) + ".class"
    ) ?: return byteArrayOf()
    val outStream = ByteArrayOutputStream()
    try {
      while (inStream.read() != -1) {
        val nextVal = inStream.read()
        outStream.write(nextVal)
      }
    } catch (e: IOException) {

    }
    return outStream.toByteArray()
  }
}
```

### Responsibility

Class Loader 系统主要有3种功能：

+ Loading
+ Linking
+ Initialization

#### Loading

类加载器读取 *.class* 文件，生成对应的二进制数据并保存到 JVM 方法区。对每个 *.class* 文件，方法区保存这些信息：

+ 完全限定名称的加载类和其直属父类
+ *.class* 文件是否与 Class 或 Interface 或 Enum 有关
+ 修饰符（Modifier）、变量和方法信息等。

加载 *.class* 文件后，JVM 在堆中创建该文件对应的 Class 类型对象。开发者可以使用此 Class 对象来获取类级别（class level）的信息，如类名、父类名、方法（`Class.getdeclaredMethods()`）和变量信息（`Class.getDeclaredFields()`）等。

#### Linking

执行验证（Verification）、准备（Preparation）和解决（Resolution）。

+ 验证：通过检查 *.class* 文件是否正确的格式化和是否由有效的编译器生成，确保该文件的正确性。如果验证失败，将抛出运行时异常：*java.lang.VerifyError*。
+ 准备：JVM 为类变量分配内存，并将内存初始化为默认值。
+ 解决（可选）：这是用直接引用替换类型中的符号引用的过程。通过搜索方法区域以找到引用的实体来完成此操作。

#### Initialization

这个阶段将为所有静态变量分配在代码或静态代码块中定义的值。在类中从顶至底（from top to bottom）执行，在类层级中从父类到子类（from parent to child）执行。类加载器的初始化：Bootstrap class loader -> Extension class loader -> App class loader.

## Run-Time Data Areas

基本结构.jpg

### The pc Register

JVM 可以支持多个线程同时运行，每个线程都有自己独立的 PC 寄存器(program counter register)。在任意时间点，任一线程在执行某个方法的代码，被称作该线程的[当前方法(current method)][currm]。

+ 如果该方法是 *native* 的，pc 寄存器的值是 *undefined*。
+ 如果该方法不是 *native* 的，pc 寄存器包含当前正在执行指令的地址。

JVM 的 pc 寄存器容量足够在特定平台上保存 *returnAddress* 或 *native* 指针。

### JVM Stacks

对每个线程，JVM 在创建线程时都会创建一个私有的堆栈。JVM 仅对堆栈进行两种操作：推入(push)和弹出(pop)栈帧(frames)。JVM 堆栈的内存**不必是连续的**。

线程执行的每个方法调用(method calls)都存储在对应的堆栈中，包括入参、局部变量、中间计算和其他数据。方法完成后，JVM 将从堆栈中删除对应的条目，完成所有的方法调用后，堆栈将变为空，并且在终止线程前，销毁该堆栈。

存储在堆栈中的数据仅适用于对应线程，对别的线程不适用。因此，可以说局部数据是线程安全的。堆栈中每个条目都称为 *堆栈帧(Stack Frame)* 或 *激活记录(Activation Record)*。

> 第一版的 Java 虚拟机规范 *The Java® Virtual Machine Specification*, *JVM 堆栈(Java Virtual Machine stack)* 被称为 *Java 栈(Java stack)*

JVM 规范允许Java虚拟机堆栈具有固定大小，或者根据计算要求动态伸缩。 如果Java虚拟机堆栈的大小固定，则在创建每个Java虚拟机堆栈时可以独立选择它们的大小。

关联的异常：

+ 如果线程中的计算需要比允许的 JVM 更大的堆栈，则 JVM 将抛出 **StackOverflowError**。

+ 如果可以动态扩展 JVM 堆栈容量，并且尝试进行动态扩展，但外部环境无法提供足够的内存资源，或没有足够的内存资源为新线程创建 JVM 堆栈，则 JVM 会抛出 **OutOfMemoryError**。

### Heap

堆是运行时数据区内为所有类实例和数组分配内存的部分，是共享资源。

堆在虚拟机启动时创建，*垃圾回收器(garbage collector)* 可以回收对象堆堆存储，对象永远不会显式释放内存。堆的大小同样可以是固定的，或者根据计算要求动态伸缩。堆内存**不必是连续的**。

用户可以控制堆固定初始大小，或动态伸缩场景控制堆的最大/最小值。

关联的异常：

+ 计算需要的堆内存多于系统可以提供的内存资源， JVM 会抛出 **OutOfMemoryError**。

### Method Area

方法区存储每个类的结构，如*运行时常量池(run-time constant pool)*，*字段(field)*，和*方法数据(method data)*，以及方法和构造函数的代码，是共享资源。

方法区同样是在 JVM 启动时创建。尽管方法区在逻辑上是堆的一部分，但 JVM 实现可以选择不对其进行垃圾回收或压缩。

方法区可以是固定大小，或根据计算需求进行动态伸缩。方法区内存同样**不必是连续的**。

关联的异常：

+ 方法区需要分配的内存多于系统可提供的内存资源，JVM 会抛出 **OutOfMemoryError**。

### Native Method Stacks

JVM 可以使用传统的堆栈，俗称 “C 栈”，来支持 *native* 方法。*native* 方法栈可以用于 JVM C语言指令集的解释器。JVM 不能加载 *native* 方法，并且它本身不依赖于传统堆栈，不需要提供 *native* 方法栈。如果提供，通常在创建线程时为每个线程提供 *native* 方法栈。

JVM 规范允许 *native* 方法堆栈具有固定大小，或者根据计算要求动态扩展和收缩。如果本机方法堆栈的大小固定，则在创建每个 *native* 方法栈的大小时可以独立选择。

*native* 方法栈关联的异常：

+ 如果线程中的计算所需的 *native* 方法栈容量超出允许的范围，则 JVM 会抛出 **StackOverflowError**.
+ 如果 *native* 方法栈容量尝试动态扩展，但外部环境无法提供足够的内存资源，或没有足够的内存资源为新线程创建 *native* 方法栈，则 JVM 会抛出 **OutOfMemoryError**.

### Run-Time Constant Pool

运行时常量池是每个类或接口的 *class* 文件中 *constant_pool* 表的运行时表示形式。它主要包含：

+ 编译时数字文字
+ 运行时方法和字段引用

运行时常量池功能类似于常规编程语言的符号表。尽管它包含的数据范围比典型的符号表更大。

每个运行时常量池都是从 JVM 方法去分配的，当 JVM 创建类或接口时，将为该类或结构构造运行时常量池。

运行时常量池会关联如下异常：

+ 创建类或缄口时，如果运行时常量池构造所需的内存超过 JVM 方法区的可用内存，则 JVM 会抛出 **OutOfMemoryError**.

## Execution Engine

执行引擎执行 *.class(bytecode)*.它逐行读取字节码，使用各个存储区的数据和信息并执行指令，它可以分成三部分：

+ Interpreter：逐行解释字节码然后执行。缺点是当多处调用一个方法时，每次都需要解释步骤。
+ Just-in-Time Compiler(HotSpot)：用于提高解释器的效率，它编译整个字节码并将其转换为 *native* 代码，每当解释器遇见重复的方法调用时，JIT 都会为该部分提供 *native* 代码，因此不用解释器重复解释，从而提高了效率。
+ Garbage Collector：销毁未引用的对象，回收内存。

## Java Native Interface（JNI）

JNI 是与 *native* 方法库交互的接口，它使 JVM 可以调用 C/C ++ 库。

## Native Method Libraries

供执行引擎需要的 *native* 代码库。

## Reference

JVM Run-Time Data Areas: <https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4>
JVM architecture: <https://www.geeksforgeeks.org/jvm-works-jvm-architecture/>
Class Loader: <https://www.baeldung.com/java-classloaders>

[currm]:https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6
