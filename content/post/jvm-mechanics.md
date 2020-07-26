---
title: "JVM Mechanics"
date: 2020-07-17
draft: true
---

## Class File

### ClassFile structure

``` class
ClassFile {
    u4 magic;
    u2 minor_version;
    u2 major_version;
    u2 constant_pool_count;
    cp_info constant_pool[constant_pool_count-1];
    u2 access_flags;
    u2 this_class;
    u2 super_class;
    u2 interfaces_count;
    u2 interfaces[interfaces_count];
    u2 fields_count;
    field_info fields[fields_count];
    u2 methods_count;
    method_info methods[methods_count];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

> The constant_pool is a table of structures representing various string constants, class and interface names, field names, and other constants that are referred to within the ClassFile structure and its substructures. The format of each constant_pool table entry is indicated by its first "tag" byte.

*constant_pool* 是一个结构表，表示字符串常量、class 和 interface 名称、field 名称以及 ClassFile 结构和其子结构中引用的常量。每个 *constant_pool* 表条目的格式由其 "tag" 字节标识。

#### constant pool

Jvm 指令不依赖于类，接口，类实例或数组，相反，指令引用了 constant_pool 表中的符号信息（symbolic info）。常量池中的条目有如下的结构：

``` class
cP_info {
    u1 tag;
    u1 info[];
}
```

tag 表示常量的类型，info 存储常量的值

tag 类型和对应的值：

|        Constant Type        | Value |
|:---------------------------:|:-----:|
| CONSTANT_Class              | 7     |
| CONSTANT_Fieldref           | 9     |
| CONSTANT_Methodref          | 10    |
| CONSTANT_InterfaceMethodref | 11    |
| CONSTANT_String             | 8     |
| CONSTANT_Integer            | 3     |
| CONSTANT_Float              | 4     |
| CONSTANT_Long               | 5     |
| CONSTANT_Double             | 6     |
| CONSTANT_NameAndType        | 12    |
| CONSTANT_Utf8               | 1     |

### Fields

*field_info* 的结构：

``` class
field_info {
    u2 access_flags;
    u2 name_index;
    u2 descriptor_index;
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

#### Field access_flags

| Flag Name     | Value  | Interpretation                                                          |
|---------------|--------|-------------------------------------------------------------------------|
| ACC_PUBLIC    | 0x0001 | Declared public; may be accessed from outside its package.              |
| ACC_PRIVATE   | 0x0002 | Declared private; usable only within the defining class.                |
| ACC_PROTECTED | 0x0004 | Declared protected; may be accessed within subclasses.                  |
| ACC_STATIC    | 0x0008 | Declared static.                                                        |
| ACC_FINAL     | 0x0010 | Declared final; no further assignment after initialization.             |
| ACC_VOLATILE  | 0x0040 | Declared volatile; cannot be cached.                                    |
| ACC_TRANSIENT | 0x0080 | Declared transient; not written or read by a persistent object manager. |

### Methods

*method_info* 的结构：

``` class
method_info {
    u2 access_flags;
    u2 name_index;
    u2 descriptor_index;
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

#### Method access_flags

| Flag Name        | Value  | Interpretation                                                  |
|------------------|--------|-----------------------------------------------------------------|
| ACC_PUBLIC       | 0x0001 | Declared public; may be accessed from outside its package.      |
| ACC_PRIVATE      | 0x0002 | Declared private; accessible only within the defining class.    |
| ACC_PROTECTED    | 0x0004 | Declared protected; may be accessed within subclasses.          |
| ACC_STATIC       | 0x0008 | Declared static.                                                |
| ACC_FINAL        | 0x0010 | Declared final; may not be overridden.                          |
| ACC_SYNCHRONIZED | 0x0020 | Declared synchronized; invocation is wrapped in a monitor lock. |
| ACC_NATIVE       | 0x0100 | Declared native; implemented in a language other than Java.     |
| ACC_ABSTRACT     | 0x0400 | Declared abstract; no implementation is provided.               |
| ACC_STRICT       | 0x0800 | Declared strictfp; floating-point mode is FP-strict             |

## Interpreter vs Compilers

### Interpreter

Fetch next bytecode, execute, repeat

### Compilers

HotSpot has two compilers: JIT compiler and server compiler

+ Convert entire methods into native code
+ Optimize(inlining, unboxing, loop optimization, etc)
+ Code calls back to "runtime" when necessary

### Compiling for JVM

编译是将 Java 源码编译成 Jvm 指令集（instruction set）的过程，编译器（compiler）是实现这一过程的工具，有些场景编译器指代将 Jvm 指令集转到到特定 CPU 指令集的工具。

[HotSpot-Internals]:https://www.youtube.com/watch?v=XjfhsJarQy0
[compiling]:https://docs.oracle.com/javase/specs/jvms/se6/html/Compiling.doc.html#6530
[class-file]:https://docs.oracle.com/javase/specs/jvms/se6/html/ClassFile.doc.html#20080
[thread]:https://docs.oracle.com/javase/specs/jvms/se6/html/Threads.doc.html#21294