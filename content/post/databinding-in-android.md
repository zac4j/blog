---
title: "Data Binding in Android"
date: 2020-07-11T08:37:45+08:00
draft: true
---

## findViewById vs. DataBinding

DataBinding 在编译时(compile time)会检查引用是否为空，相反 findViewById 并不会，从而可能引发运行时崩溃。

## observable

通过 DataBinding，observable 的值改变时，绑定该值的 UI 元素也会自动更新，有很多方式实现这种 observability：

+ observable classes
+ observable fields
+ LiveData
