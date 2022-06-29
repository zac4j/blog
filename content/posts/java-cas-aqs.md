---
title: "Java CAS AQS"
date: 2022-06-29T21:10:50+08:00
tags: ["java", "cas", "aqs"]
description: "Intro to CAS & AQS in Java"
categories: ["java", "concurrent"]
author: "杨晓峰·geektime"
draft: true
---

## AtomicInteger 的底层原理

AtomicInteger 是对 int 类型的一种封装，提供原子性的访问和更新操作，其原子性操作的实现是基于 CAS（[compare-and-swap][cas]）技术。

所谓 CAS，表征的是一系列操作的集合，获取当前数值，进行一系列运算，利用 CAS 指令试图进行更新。如果当前数值未变，代表没有其他线程进行并发修改，则更新成功。如果当前数值变了，可能出现不同的选择，要么进行重试，要么就返回一个成功或失败的结果。

从 AtomicInteger 的内部属性来看，它依赖于 Unsafe 提供的一些底层能力，进行底层操作；以 volatile 的 value 字段，记录数值，以保证可见性。

``` java
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
private volatile int value;
```

具体的原子操作细节，可以参考任意一个原子更新方法，比如下面的 getAndIncrement.

Unsafe 会利用 value 字段的内存地址偏移，直接完成操作。

``` java
public final int getAndIncrement() {
    return U.getAndInt(this, VALUE, 1);
}
```

因为 getAndIncrement 需要返回数值，所以需要添加失败重试逻辑。

``` java

public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
```

而类似 compareAndSet 这种返回 boolean 类型的函数，因为其返回值表现的就是是否成功，所以不需要重试:

``` java
public final boolean compareAndSet(int expectedValue, int newValue);
```

CAS 是 Java 并发中所谓的 lock-free 机制的基础。

[cas]:https://en.wikipedia.org/wiki/Compare-and-swap
[aqs]:https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/locks/AbstractQueuedSynchronizer.html
