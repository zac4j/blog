---
title: "Intro to ConcurrentHashMap"
date: 2022-05-23T15:11:27+08:00
description: "ConcurrentHashMap introduction"
tags: ["hashmap", "concurrenthashmap", "map"]
categories: ["data structure", "concurrent"]
author: "杨晓峰·geektime"
draft: true
---

Java 提供了不同层面的线程安全支持。在传统集合框架内部，除了 Hashtable 等容器，还提供了同步包装器（Synchronized Wrapper），我们可以通过 Collections 工具类提供的包装方法来获取同步包装器，如 Collections.synchronizedMap，但是它们利用的都是粗粒度的同步方式，在高并发情况下，性能比较底下。

<!--more-->

除了同步包装器，我们更加普遍的选择是使用并发包（java.util.concurrent）提供的线程容器类：

+ 并发容器，如 ConcurrentHashMap、CopyOnWriteArrayList
+ 线程安全队列，如 ArrayBlockingQueue、SynchronousQueue。
+ 各种有序容器的线程安全版本等。

## 为什么需要 ConcurrentHashMap

Hashtable 本身比较低效，因为它的实现基本就是将 put、get、size 等各种方法加上 *synchronized* 关键字。简单来说，这就导致了所有并发操作都要竞争同一把锁，一个线程在进行同步操作时，其他线程只能等待，大大降低了并发操作的效率。

前面提到的 HashMap 不是线程安全的，并发情况会导致类似 CPU 占用 100% 等一些问题，那么能不能利用 Collections 提供的 Synchronized Wrapper 来解决问题呢？

我们可以看下 SynchronizedMap 的实现，虽然所有操作不声明为 synchronized 方法，但还是利用了 this 作为互斥的 mutex，这种版本不适合高并发的场景。

``` java
private static class SynchronizedMap<K,V> implements Map<K,V>, Serializable {
    private final Map<K,V> m;     // Backing Map
    final Object mutex;        // Object on which to synchronize
    // …
    public int size() {
        synchronized (mutex) {return m.size();}
    }
 // … 
}
```

我们再来看看 ConcurrentHashMap 是如何设计实现的，为什么它能大大提高并发效率。

实际上 ConcurrentHashMap 的设计实现一直在演化，这里我们基于 JDK8 和之前的版本做比较。

### JDK7 ConcurrentHashMap 的[实现][older] 基于

+ 分离锁，也就是将内部进行分段（Segment），里面则是 HashEntry 的数组，和 HashMap 类似，hash 相同的条目也是以链表形式存放。
+ HashEntry 内部使用 volatile 的 value 字段来保证可见性，也利用了不可变的机制以改进利用 **Unsafe** 提供的底层能力，如 volatile access，去直接完成部分操作，以优化性能。

``` java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
    ...
}
```

JDK7 中 ConcurrentHashMap 的结构图

![old-impl](/img/concurrenthashmap-jdk7.webp)

其核心是利用分段设计，在进行并发操作的时候，只需要锁定相应段，这样就有效避免了类似 Hashtable 整体同步的问题，大大提高了性能。

Segment 由 concurrencyLevel，默认值（DEFAULT_CONCURRENCY_LEVEL）是 16，concurrencyLevel 也可以在构造函数里指定。

可以看看 JDK7 里 ConcurrentHashMap 的 get 方法。

``` java

public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key.hashCode());
    //利用位操作替换普通数学运算
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // 以Segment为单位，进行定位
    // 利用Unsafe直接进行volatile access
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        //省略
        }
    return null;
}
```

对于 put 操作，首先是通过二次 hash 避免哈希冲突，然后以 Unsafe 调用方式，直接获取相应的 Segment，然后进行线程安全的 put 操作：

``` java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    // 二次哈希，以保证数据的分散性，避免哈希冲突
    int hash = hash(key.hashCode());
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
            (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    // 在 Segment 中进行 put 操作
    return s.put(key, hash, value, false);
}
```

其核心逻辑实现在下面的内部方法中：

``` java

final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // scanAndLockForPut 会去查找是否有 key 相同 Node
    // 无论如何，确保获取锁
    HashEntry<K,V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                // 更新已有 value...
            }
            else {
                // 放置 HashEntry 到特定位置，如果超过阈值，进行rehash
                // ...
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}

```

从上面的源码可以看出，在进行并发写操作时：

+ ConcurrentHashMap 会获取重入锁，以保证数据一致性，Segment 本身就扩展实现自 ReentrantLock，所以在并发修改期间，相应的 Segment 会被锁定。
+ 在最初阶段，进行重复性的扫描，以确定相应 key 值是否已经在数组里面，进而决定是更新还是放置操作。
+ 和 HashMap 扩容不同，ConcurrentHashMap 不是整体的扩容，而是单独对 Segment 进行扩容。

ConcurrentHashMap 还有个 size 方法，它的实现涉及分离锁的一个副作用。

试想，如果不进行同步，简单的计算所有 Segment 的总值，可能会因为并发 put，导致结果不准确，但是直接锁定所有 Segment 进行计算，就会变得非常昂贵。其实，分离锁也限制了 Map 的初始化等操作。

所以，ConcurrentHashMap 的实现是通过重试机制（RETRIES_BEFORE_LOCK，指定重试次数 2），来试图获得可靠值。如果没有监控到发生变化（通过对比 Segment.modCount），就直接返回，否则获取锁进行操作。

### Java8 版本中，ConcurrentHashMap 有哪些变化呢？

+ 总体结构上，它的内部存储和 HashMap 结构非常相似，同样是以 bucket 数组，然后内部也是一个个所谓的链表结构（bin），同步的粒度更细致一些。
+ 其内部仍有 Segment 定义，但仅为了保证序列化的兼容性，不再有任何结构上的用处。
+ 因为不再使用 Segment，初始化操作大大简化，修改为 lazy load 模式，这样可以有效避免初始化开销。
+ 数据存储利用 volatile 来保证可见性。
+ 使用 CAS 等操作，在特定场景进行无锁并发操作。
+ 使用 Unsafe、LongAdder 等底层手段，进行极端情况优化。

存储实现为 Node<K, V>，和 HashMap 比较类似，不同点是 ConcurrentHashMap 里 value 和 next Node 都声明为 volatile，保证可见性。

``` java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    // ...
}
```

我们看看 get 方法的实现

``` java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            // 初始化操作
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // Bin 为空时，利用 CAS 进行无锁线程安全操作
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                // 细粒度同步修改操作
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        // ...
                    }
                    else if (f instanceof TreeBin) {
                        // ...
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            // Bin 超过阀值，进行树化
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

这里同步使用了 synchronized 关键字，为何没用通常建议的 ReentrantLock 呢，这是因为现代 JDK 中，synchronized 已被不断优化，可以不再担心性能差异，而且相较于 ReentrantLock，synchronized 可以减少内存消耗。

tabAt 的实现直接利用了 Unsafe.getObjectAcquire 进行优化，避免间接调用的开销。

``` java
/*
 * Volatile access methods are used for table elements as well as
 * elements of in-progress next table while resizing.  All uses of
 * the tab arguments must be null checked by callers.  All callers
 * also paranoically precheck that tab's length is not zero (or an
 * equivalent check), thus ensuring that any index argument taking
 * the form of a hash value anded with (length - 1) is a valid
 * index.  Note that, to be correct wrt arbitrary concurrency
 * errors by users, these checks must operate on local variables,
 * which accounts for some odd-looking inline assignments below.
 * Note that calls to setTabAt always occur within locked regions,
 * and so in principle require only release ordering, not
 * full volatile semantics, but are currently coded as volatile
 * writes to be conservative.
 */
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

初始化操作在 initTable 中，这是一个典型的 CAS 使用场景，利用 volatile 的 sizeCtl 作为互斥手段：如果发现竞争性的初始化，就 spin 在那里，等待条件恢复；否则利用 CAS 设置排他标志。如果成功则进行初始化，否则重试。

``` java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 如果有冲突，进行 spin 等待
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // CAS 成功返回 true，进入初始化逻辑
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

再看看，size 的操作，最终的实现逻辑是在 sumCount 方法中：

``` java
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

而 CounterCell 的实现为：

``` java
static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

对于 CounterCell 的操作，是基于 java.util.concurrent.atomic.LongAdder 进行的，是一种 JVM 利用空间换取更高效率的方法，利用了Striped64内部的复杂逻辑。这个东西非常小众，大多数情况下，建议还是使用 AtomicLong，足以满足绝大部分应用的性能需求。

[older]:https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/classes/java/util/concurrent/ConcurrentHashMap.java
