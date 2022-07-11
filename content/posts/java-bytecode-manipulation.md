---
title: "Java Bytecode Manipulation"
date: 2022-07-03T20:41:23+08:00
description: "How to Manipulate Java Bytecode"
tags: ["bytecode"]
categories: ["java"]
author: "Zac"
draft: true
---

## 从字节码到 Class 对象

在类加载过程中，类从字节码到 Class 对象的转换，通过下面的方法提供的功能：

``` java
// ClassLoaderImpl
protected final Class<?> defineClass(String name, byte[] b, int off, int len, ProtectionDomain domain)

protected final Class<?> defineClass(String name, java.nio.ByteBuffer buffer, ProtectionDomain domain)
```

可以看出，只要能够生成规范的字节码，不管是作为 byte 数组的形式，还是放到 ByteBuffer 里，都可以平滑地完成字节码到 Java 对象的转换过程。

JDK 提供了 defineClass 方法，最终都是本地代码实现的：

``` java
static native Class<?> defineClass1(ClassLoader loader, String name, byte[] b, int off, int len, ProtectionDomain pd, String source);

static native Class<?> defineClass2(ClassLoader loader, String name, java.nio.ByteBuffer b, int off, int len, ProtectionDomain pd, String source);
``

通过 JDK dynamic proxy 的源码，我们可以发现，在 ProxyBuilder 这个静态内部类中，[ProxyGenerator][pg] 生成字节码，并以 byte 数组的形式保存，然后通过调用 Unsafe 提供的 defineClass 入口。

``` java
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, interfaces.toArray(EMPTY_CLASS_ARRAY), accessFlag);
try {
    Class<?> pc = UNSAFE.defineClass(proxyName, proxyClassFile, 0, proxyClassFile.length, loader, null);
    reverseProxyCache.sub(pc).putIfAbsent(loader, Boolean.TRUE);
    return pc
} catch (ClassFormatError e) {

}
```

### 如何操纵字节码

### hard code

参考 [ProxyGenerator][pg] 的内部实现，我们可以了解到，其利用 DataOutputStream，配合 hard code 的各种 JVM 指令实现方法，生成所需的字节码数组。

``` java
private void codeLocalLoadStore(int lvar, int opcode, int opcode_0, DataOutputStream out) throws IOException {
    assert lvar >= 0 && lvar <= OxFFFF;
    // 根据变量数值，以不同格式，dump 操作码
    if (lvar <= 3) { 
        out.writeByte(opcode_0 + lvar);  
    } else if (lvar <= 0xFF) {      
        out.writeByte(opcode);      
        out.writeByte(lvar & 0xFF);  
    } else {
        // 使用宽指令修饰符，如果变量索引不能用无符号byte      
        out.writeByte(opc_wide);      
        out.writeByte(opcode);      
        out.writeShort(lvar & 0xFFFF);  
    }
}
```

这种实现方式的好处是没有太多依赖关系，简单实用，但是**前提**是需要懂各种 [JVM 指令][ji]，知道怎么处理那些偏移地址等，实际门槛非常高，所以不太适合大多数普通开发场景。

[pg]:http://hg.openjdk.java.net/jdk/jdk/file/29169633327c/src/java.base/share/classes/java/lang/reflect/ProxyGenerator.java
[ji]:https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5
[asm]:https://asm.ow2.io/
