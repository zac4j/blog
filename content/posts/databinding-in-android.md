---
title: "Data Binding in Android"
date: 2020-08-23T08:37:45+08:00
description: "DataBinding"
tags: ["databinding"]
categories: ["android"]
draft: false
---

每次在 view 发生创建或重建，我们使用 `findViewById()` 来获取 view 的引用时，Android 系统在运行时会遍历整个视图层级结构（view hierarchy）来查找它。在我们 app 的视图结构比较简单时没有问题，但是生产上的 app 一个 layout 可能有几十个 views，即使是最佳设计，也会存在嵌套的 views.

<!--more-->

## Intro to DataBinding

对于庞大或较深的视图层级结构，查找 view 可能耗费很多时间，从而可能明显降低 app 的响应速度。我们可以在变量中缓存这些 view 的引用，但这需要我们在每个 Activity/Fragment 中对于每个 view 都创建变量来暂存引用。

另一种方案是创建一个对象包含每个 view 的引用。整个对象，称为 *绑定对象（binding object）*，可以被整个 app 使用。这项技术被称为 *数据绑定（data binding）*。当 app 的绑定对象创建后，我们可以通过绑定对象访问 views，或别的数据，而不用遍历整个 view hierarchy 来查找它们。

## findViewById vs. DataBinding

Data binding 有如下优势：

+ 与 `findView` 相比代码更简洁，更易于阅读和维护。
+ 数据和 views 明确分离，
+ Android 系统在 app 启动时仅遍历 view hierarchy 一次即可获取每个 view 的引用，而不是像 `findView` 在运行时获取 view 引用。
+ 我们可以 [type safety][ts] 访问 views。（Type safety 意味着编译器在编译时会验证类型，如果我们将错误的类型分配给变量，则会引发编译错误。）
  
## Databing expression

### Formatted string

+ String res:

``` xml
<string name="quote_format">\"%s\"</string>
<string name="score_format">Current Score: %d</string>
```

+ Layout res:

``` xml
<TextView
    android:id="@+id/word_text"
   ...
   android:text="@{@string/quote_format(gameViewModel.word)}"
/>
```

### Null coalescing operator

The null coalescing operator (??) chooses the left operand if isn't null or the right if the former is null.

``` xml
<TextView
    android:text="@{user.displayName ?? user.lastName}"
/>
```

This is functionally equivalent to:

``` xml
<TextView
    android:text="@{user.displayName != null ? user.displayName : user.lastName}"
/>
```

[es]:https://developer.android.com/topic/libraries/data-binding/expressions#string_literals
[ts]:https://en.wikipedia.org/wiki/Type_safety
