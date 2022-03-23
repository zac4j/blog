---
title: "Intro to ClassLoader"
date: 2022-03-23T08:44:28+08:00
description: "Introduction to ClassLoader"
tags: ["classloader"]
categories: ["java"]
author: "Zac"
draft: false
---

## 类加载器（ClassLoader）

### 什么是类加载器

类加载器是 JRE 的一部分，负责在运行时将 Java 类动态加载到 JVM，有了类加载器，JVM 无需了解底层文件或文件系统即可运行Java程序。

类加载器不会一次将全部类加载到内存里，而是在程序需要时加载。

### JVM 内置的类加载器类型

+ Bootstrap class loader
  
  每个 JVM 实现必须有一个 bootstrap 类加载器。它负责 JDK 内部类，通常是 *rt.jar* 和位于 *JAVA_HOME/jre/lib* 目录下的其他核心类，该路径通常被称为引导路径(bootstrap path)。该类加载器由 C/C++ 语言实现。
+ Extension class loader
  
  Bootstrap 的子类加载器，它加载扩展目录 *JAVA_HOME/jre/lib/ext* 或 **java.ext.dirs** 指定路径下的类。它在 `sun.misc.Launcher$ExtClassLoader` 类中由 Java 语言实现。
+ System/Application class loader
  
  Extension 的子类加载器。它负责加载 App 层级的类，它加载在 *classpath* 环境变量、*-classpath*、-cp 命令行选项中找到的文件。它在 `sun.misc.Launcher$AppClassLoader` 类中由 Java 语言实现。

### 自定义的类加载器

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

### ClassLoader 是如何工作的

类加载器是 JRE 的一部分。当 JVM 需要一个类时，类加载器尝试定位该类，并使用 **完全限定的类名** *(fully qualified class name)* 将 **类定义**(*class definition*) 加载到 Runtime 中。

`java.lang.ClassLoader.loadClass()` 方法负责将 *class definition* 加载到 Runtime 中。如果请求的类尚未加载，则它将请求委派给父类加载器，此过程递归进行。如果父类加载器找不到该类，则子类加载器调用 `java.net.URLClassLoader.findClass()` 方法在文件系统中查找类。最终，如果子类加载器未找到该类，则它将抛出 `java.lang.NoClassDefFoundError` 或 `java.lang.ClassNotFoundException`。

#### 使用代理模型

类加载器遵循代理模型，在该模型中，类加载器会根据请求查找的类，将搜索类的工作委托给父加载器处理。仅当 bootstrap class loader 和 extension class loader 都未能加载该类时，system class loader 才会尝试自己加载该类。

### 类加载器的功能

类加载器系统主要有[3种功能][lli]：

+ **加载（Loading）**
  
  类加载器读取 *.class* 文件，生成对应的二进制数据并保存到 JVM 方法区。对每个 *.class* 文件，JVM 在方法区保存这些信息：
  
  + 已加载类和其父类的完全限定名
  + *.class* 文件是否与 Class 或 Interface 或 Enum 相关
  + 修饰符（Modifier）、变量和方法信息等。

  加载 *.class* 文件后，JVM 在堆中创建该文件对应的 Class 类型对象。可以使用此 Class 对象来获取类级别（class level）的信息，如类名、父类名、方法（`Class.getdeclaredMethods()`）和变量信息（`Class.getDeclaredFields()`）等。
+ **连接（Linking）**
  
  执行 Verification、Preparation 和 Resolution。

  + Verification：通过检查 *.class* 文件是否正确的格式化和是否由有效的编译器生成，确保该文件的正确性。如果验证失败，将抛出运行时异常：*java.lang.VerifyError*。
  + Preparation：JVM 为类变量分配内存，并将内存初始化为默认值。
  + Resolution：这是用直接引用(direct references)替换类型中的符号引用(symbolic references)的过程。通过搜索方法区域以找到引用的实体来完成此操作。

+ **初始化（Initialization）**
  
  这个阶段将为所有静态变量分配在代码或静态代码块中定义的值。在类中从顶至底（from top to bottom）执行，在类层级中从父类到子类（from parent to child）执行。类加载器的初始化：Bootstrap class loader -> Extension class loader -> App class loader.

[lli]:https://docs.oracle.com/javase/specs/jvms/se6/html/ConstantPool.doc.html#67960
