---
title: "Intro to Executor"
date: 2022-03-29T21:46:30+08:00
description: "Introduction to Executor"
tags: ["executor", "thread", "threadpool"]
categories: ["java", "concurrent"]
draft: true
---

## Executor, ExecutorService, Executors

通常我们执行多个 Runnable task 会使用这种形式：

``` kotlin
val t1 = Thread { 
    // Runnable task1
}

val t2 = Thread {
    // Runnable task2
}

t1.start()
t2.start()
```

如果是执行 N 个 Runnable tasks，对每个 Runnable task 创建一个 Thread 执行显然不合适，我们可以通过实现 Executor 接口来编排 Runnable task 的执行：

``` kotlin
class SerialExecutor(private val executor: Executor): Executor {

  private val tasks: ArrayDeque<Runnable> = ArrayDeque()
  private var active: Runnable? = null

  @Synchronized
  override fun execute(r: Runnable?) {
    tasks.add {
      try {
        r?.run()
      } finally {
        scheduleNext()
      }
    }
    if (active == null) {
      scheduleNext()
    }
  }

  @Synchronized
  private fun scheduleNext() {
    tasks.removeFirstOrNull()?.let {
      active = it
      executor.execute(active)
    }
  }

}
```


