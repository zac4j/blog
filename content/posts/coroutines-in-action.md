---
title: "Coroutines in Action"
date: 2020-05-16T10:55:23+08:00
draft: true
---

介绍完协程是什么，以及如何管理协程后，我们再看看如何使用协程。在实际任务中，有两种类型的任务非常适合使用协程来解决：

+ 一次性请求(one shot requests) 请求一次获取到结果后就结束执行。
+ 流式请求(streaming request) 请求发出后，还一直监听它的变化并处理返回结果

### 一次性请求
