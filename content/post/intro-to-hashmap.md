---
title: "Intro to HashMap"
date: 2020-07-15T15:11:27+08:00
description: "HashMap introduction"
tags: ["hashmap", "data structure"]
categories: ["hashmap", "data structure", "map"]
draft: true
---

Hash table based implementation of the Map interface.  This implementation provides all of the optional map operations, and permits null values and the null key.

<!--more-->

## HashMap 介绍

The HashMap class is roughly equivalent to Hashtable, except that it is unsynchronized and permits nulls. This class makes no guarantees as to the order of the map; in particular, it does not guarantee that the order will remain constant over time.

HashMap 和 HashTable 大致等效，除了 HashMap 是非同步的，以及允许空值。无法保证顺序随着时间的推移保持不变。

This implementation provides **constant-time** performance for the basic operations (get and put),assuming the hash function disperses the elements properly among the buckets.

假设 hash 函数将元素分散在正确的 bucket 中，HashMap 对于基本操作(get 或 put) 会提供恒定时间的性能。

### 何为 constant time

constant time 是表示时间复杂度的一个示例，常见时间复杂度的描述：

| Time complexity | Time description   |
|-----------------|--------------------|
| O(1)            | Constant time      |
| O(log n)        | Logarithmic time   |
| O(n)            | Linear time        |
| O(n log n)      | Pseudo-linear time |
| O(n^c)          | Polynomial time    |
| O(c^n)          | Exponential time   |
| O(n!)           | Factorial time     |

Iteration over collection views requires time proportional to the "capacity" of the HashMap instance (the number of buckets) plus its size (the number of key-value mappings).

从集合视角看，迭代所需的时间与 HashMap 实例的“容量”（ bucket 数量 ）及其大小（键-值映射数）成正比。

Thus, it's very important not to set the initial capacity too high (or the load factor too low) if iteration performance is important.

因此，如果迭代性能很重要，那么不要将初识容量设置过高（或负载因子设置过低），这很关键。

An instance of HashMap has two parameters that affect its performance：

有两个参数会影响 HashMap 实例的性能：

+ initial capacity 初始容量
+ load factor 负载因子

The capacity is the number of buckets in the hash table, and the initial capacity is simply the capacity at the time the hash table is created. The load factor is a measure of how full the hash table is allowed to get before its capacity is automatically increased.

容量是哈希表中 bucket 的数量，初始容量即是创建哈希表时的容量。负载因子用来评估哈希表填满的程度，以自动增长容量。

When the number of entries in the hash table exceeds the product of the load factor and the current capacity, the hash table is rehashed (that is, internal data structures are rebuilt) so that the hash table has approximately twice the number of buckets.

当哈希表中的条目数超过负载因子和当前容量的乘积时，哈希表将被重新映射（即内部数据结构将被重建），以使哈希表的 bucket 数大约为两倍。

As a general rule, the default load factor (0.75) offers a good tradeoff between time and space costs. Higher values decrease the space overhead but increase the lookup cost (reflected in most of the operations of the HashMap class, including get and put).  

通常，默认负载因子（0.75）在时间和空间成本之间提供了一个很好的权衡。较高的值会减少空间开销，但会增加查找成本（反应在 HashMap 类的大多数操作中，包括 get 和 put）。

The expected number of entries in the map and its load factor should be taken into account when setting its initial capacity, so as to minimize the number of rehash operations.  

设置其初始容量时，应考虑 map 中的预期条目数及其负载因子，以最大程度地减少重新映射操作的次数。

If the initial capacity is greater than the maximum number of entries divided by the load factor, no rehash operations will ever occur.

如果初始容量大于最大条目数除以负载因子，则将不会发生任何映射操作。

If many mappings are to be stored in a HashMap instance, creating it with a sufficiently large capacity will allow the mappings to be stored more efficiently than letting it perform automatic rehashing as needed to grow the table.

如果将许多映射关系存储在 HashMap 实例中，创建足够大容量的 map 将比让其按需自动增长哈希表重新映射，更有效的存储映射。

Note that using many keys with the same {@code hashCode()} is a sure way to slow down performance of any hash table. To ameliorate impact, when keys are {@link Comparable}, this class may use comparison order among keys to help break ties.

注意，使用许多具有相同 hashCode 的键是降低任何哈希表性能的方法。为了改善影响，当键是可比较时(Comparable)时，此类可以使用键之间的比较顺序来帮助打破平衡。

Note that this implementation is not synchronized. If multiple threads access a hash map concurrently, and at least one of the threads modifies the map structurally, it must be synchronized externally. (A structural modification is any operation that adds or deletes one or more mappings; merely changing the value associated with a key that an instance already contains is not a structural modification.) This is typically accomplished by synchronizing on some object that naturally encapsulates the map.

注意，这个实现并不是同步的。如果多线程同时访问 hash map，并且至少有一个线程在结构上修改了 map，则必须在外部进行同步。（结构型修改是指增删一个或多个映射关系，更新某个 key 对应的值并不是结构型修改）。通常是对封装 map 的某个对象加同步来实现。

If no such object exists, the map should be "wrapped" using the {@link Collections#synchronizedMap Collections.synchronizedMap} method.  This is best done at creation time, to prevent accidental unsynchronized access to the map: Map m = Collections.synchronizedMap(new HashMap(...));

如果没有这样的对象存在，则应该使用 Collections.synchronizedMap 方法来包装 map，最好是在创建时完成，以避免意外不同步地访问 map：`Map m = Collections.synchronizedMap(new HashMap(...));`

The iterators returned by all of this class's "collection view methods" are fail-fast: if the map is structurally modified at any time after the iterator is created, in any way except through the iterator's own remove method, the iterator will throw a {@link ConcurrentModificationException}.  Thus, in the face of concurrent modification, the iterator fails quickly and cleanly, rather than risking arbitrary, non-deterministic behavior at an undetermined time in the future.

此类的所有“集合视角方法”返回的迭代器都是 [fast fail][ff] 的：在创建迭代器之后任何时刻对 map 结构进行了修改，除了通过迭代器自已的 remove 方法以外，该迭代器都会抛出 ConcurrentModificationException。因此，面对并发修改，迭代器会快速干净的 fail，而不是在未来的不确定时间冒着任意，不确定行为的风险。

### 何为 fast fail

Note that the fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification.  Fail-fast iterators throw ConcurrentModificationException on a best-effort basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: the fail-fast behavior of iterators should be used only to detect bugs.

注意不能保证迭代器的 fast fail 行为，一般来说，在存在不同步的并发修改情况下，不可能作出任何严格的保证。因此，编写依赖于此异常的程序的正确性是错误的：迭代器的 fast fail 行为仅用于错误检测。

This class is a member of the Java Collections Framework[jcf]

[ff]:https://whatis.techtarget.com/definition/fail-fast
[jcf]:https://docs.oracle.com/javase/8/docs/technotes/guides/collections/index.html
