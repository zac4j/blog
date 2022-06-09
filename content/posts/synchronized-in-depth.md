---
title: "Java synchronized in Depth"
date: 2022-06-07T21:50:12+08:00
tags: ["lock", "async", "thread"]
description: "The JVM implementation of synchronized keyword"
categories: ["java", "concurrent"]
author: "杨晓峰·geektime"
draft: true
---

根据上篇 [Intro to Java synchronized and Locks]({{< relref "/posts/synchronized-and-locks.md" >}}) 的介绍，我们知道 synchronized 代码块是由一对 monitorenter/monitorexit 指令实现，Monitor 对象是同步的[基本实现单元][monitor]。

<!-- more -->

## 基本表现

在 Java6 以前，Monitor 的实现完全依赖操作系统内部的互斥锁，因为需要进行用户态到内核态的切换，所以同步操作是一个无差别的重量级操作。现代 JDK 中，JVM 对此进行了大刀阔斧的改进，提供了三种不同的 Monitor 实现，也就是常说的三种不同的锁：偏斜锁（Biased Locking）、轻量级锁和重量级锁，大大改进了 Monitor 的性能。

所谓锁的升级、降级，就是 JVM 优化 synchronized 运行的机制，当 JVM 检测到不同的竞争状态时，会自动切换到适合的锁实现，这种切换就是锁的升级、降级。

当没有竞争出现时，默认会使用偏斜锁。JVM 会利用 CAS（Compare and Swap）操作，在对象头上的 Mark Word 部分设置线程 ID，以表示这个对象偏向于当前线程，所以并不涉及真正的互斥锁。这样做的假设是基于在很多场景中，大部分对象生命周期中最多会被一个现场锁定，使用偏斜锁可以降低无竞争开销。

如果有另外的线程试图锁定某个已被偏斜过的对象，JVM 就需要撤销（revoke）偏斜锁，并切换到轻量级锁实现。轻量级锁依赖 CAS 操作 Mark Word 来试图获取锁，如果重试成功，就使用普通的轻量级锁；否则，进一步升级为重量级锁。

## 内部实现

synchronized 是 JVM 内部 Intrinsic Lock，所以偏斜锁、轻量级锁、重量级锁的代码实现，并不在核心类库部分，而是在 JVM 代码中。

Java 代码运行可能是解释模式也可能是编译模式，所以对应的同步逻辑实现，也会分散在不同模块下，比如解释器的版本就是[interpreterRuntime][ir]

首先，synchronized 的行为是 JVM runtime 的一部分，所以我们先找到 Runtime 相关的功能实现。通过搜索 "monitor_enter" 或 "Monitor Enter" 可以很直观的定位到：

+ [sharedRuntime.cpp][sr]/hpp, 是解释器和编译器运行时的基类。
+ [synchronizer.cpp][sy]/hpp, JVM 同步相关的各种基础逻辑。

在 sharedRuntime.cpp 中，下面代码体现了 synchronized 的主要逻辑：

``` cpp
Handle h_obj(THREAD, obj);
// UseBiasedLocking 在 JVM 启动时可以指定是否开启偏斜锁
if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, lock, true, CHECK);
} else {
    ObjectSynchronizer::slow_enter(h_obj, lock, CHECK);
}
```

偏斜锁并不适合所有应用场景，撤销操作（revoke）是比较重的行为，只有当存在较多不会真正竞争的 synchronized 块时，才能体现出明显改善。偏斜锁会延缓 JIT 预热的过程，所以很多性能测试中会显式地关闭偏斜锁，命令如下：

``` terminal
-xx:-UseBiasedLocking
```

fast_enter 是偏斜锁获取路径，slow_enter 则是绕过偏斜锁，直接进入轻量级锁获取逻辑。

fast_enter 的实现在 [synchronizer.cpp][sy] 中：

``` cpp
// -----------------------------------------------------------------------------
//  Fast Monitor Enter/Exit
// This the fast monitor enter. The interpreter and compiler use
// some assembly copies of this code. Make sure update those code
// if the following function is changed. The implementation is
// extremely sensitive to race condition. Be careful.

void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock,
                                    bool attempt_rebias, TRAPS) {
  if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }

  slow_enter(obj, lock, THREAD);
}
```

我们分析下上面的逻辑：

+ [BiasedLocking][bl] 定义了偏斜锁相关操作，revoke_and_rebias 是获取偏斜锁的入口方法，revoke_at_safepoint 则定义了当检测到安全点时的处理逻辑。
+ BiasedLocking 通过 CAS 设置 Mark Word，对象头中 Mark Word 的结构如图：
![markword](/img/markword.webp)
+ 如果偏斜锁获取失败，进入 slow_enter，偏斜锁升级为轻量级锁。

``` cpp
// -----------------------------------------------------------------------------
// Interpreter/Compiler Slow Case
// This routine is used to handle interpreter/compiler slow case
// We don't need to use fast path here, because it must have been
// failed in the interpreter/compiler code.
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  if (mark->is_neutral()) {
    // Anticipate successful CAS -- the ST of the displaced mark must
    // be visible <= the ST performed by the CAS.
    // 将目前的 mark word 复制到 displaced header 上
    lock->set_displaced_header(mark);
    // 利用 CAS 设置对象的 mark word
    if (mark == obj()->cas_set_mark((markOop) lock, mark)) {
      return;
    }
    // Fall through to inflate() ...
    // 检查存在竞争
  } else if (mark->has_locker() &&
             THREAD->is_lock_owned((address)mark->locker())) {
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    // 清除 mark
    lock->set_displaced_header(NULL);
    return;
  }

  // The object header will never be displaced to this lock,
  // so it does not matter what the value is, except that it
  // must be non-zero to avoid looking like a re-entrant lock,
  // and must not look locked either.
  // 重置 displaced header
  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD,
                              obj(),
                              inflate_cause_monitor_enter)->enter(THREAD);
}
```

+ 设置 displaced header，然后利用 cas_set_mark 设置对象 mark word
+ 否则重置 displaced header，进入 inflate() 锁膨胀阶段

``` cpp
ObjectMonitor* ObjectSynchronizer::inflate(Thread * Self,
                                                     oop object,
                                                     const InflateCause cause) {

  // Inflate mutates the heap ...
  // Relaxing assertion for bug 6320749.
  assert(Universe::verify_in_progress() ||
         !SafepointSynchronize::is_at_safepoint(), "invariant");

  EventJavaMonitorInflate event;

  // 自旋锁的由来
  for (;;) {
    const markOop mark = object->mark();
    assert(!mark->has_bias_pattern(), "invariant");

    // The mark can be in one of the following states:
    // *  Inflated     - just return
    // *  Stack-locked - coerce it to inflated
    // *  INFLATING    - busy wait for conversion to complete
    // *  Neutral      - aggressively inflate the object.
    // *  BIASED       - Illegal.  We should never see this

    // CASE: inflated
    if (mark->has_monitor()) {
      ObjectMonitor * inf = mark->monitor();
      assert(inf->header()->is_neutral(), "invariant");
      assert(oopDesc::equals((oop) inf->object(), object), "invariant");
      assert(ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
      return inf;
    }

    // CASE: inflation in progress - inflating over a stack-lock.
    // Some other thread is converting from stack-locked to inflated.
    // Only that thread can complete inflation -- other threads must wait.
    // The INFLATING value is transient.
    // Currently, we spin/yield/park and poll the markword, waiting for inflation to finish.
    // We could always eliminate polling by parking the thread on some auxiliary list.
    if (mark == markOopDesc::INFLATING()) {
      ReadStableMark(object);
      continue;
    }

    // CASE: stack-locked
    // Could be stack-locked either by this thread or by some other thread.
    //
    // Note that we allocate the objectmonitor speculatively, _before_ attempting
    // to install INFLATING into the mark word.  We originally installed INFLATING,
    // allocated the objectmonitor, and then finally STed the address of the
    // objectmonitor into the mark.  This was correct, but artificially lengthened
    // the interval in which INFLATED appeared in the mark, thus increasing
    // the odds of inflation contention.
    //
    // We now use per-thread private objectmonitor free lists.
    // These list are reprovisioned from the global free list outside the
    // critical INFLATING...ST interval.  A thread can transfer
    // multiple objectmonitors en-mass from the global free list to its local free list.
    // This reduces coherency traffic and lock contention on the global free list.
    // Using such local free lists, it doesn't matter if the omAlloc() call appears
    // before or after the CAS(INFLATING) operation.
    // See the comments in omAlloc().

    if (mark->has_locker()) {
      ObjectMonitor * m = omAlloc(Self);
      // Optimistically prepare the objectmonitor - anticipate successful CAS
      // We do this before the CAS in order to minimize the length of time
      // in which INFLATING appears in the mark.
      m->Recycle();
      m->_Responsible  = NULL;
      m->_recursions   = 0;
      m->_SpinDuration = ObjectMonitor::Knob_SpinLimit;   // Consider: maintain by type/class

      markOop cmp = object->cas_set_mark(markOopDesc::INFLATING(), mark);
      if (cmp != mark) {
        omRelease(Self, m, true);
        continue;       // Interference -- just retry
      }

      // We've successfully installed INFLATING (0) into the mark-word.
      // This is the only case where 0 will appear in a mark-word.
      // Only the singular thread that successfully swings the mark-word
      // to 0 can perform (or more precisely, complete) inflation.
      //
      // Why do we CAS a 0 into the mark-word instead of just CASing the
      // mark-word from the stack-locked value directly to the new inflated state?
      // Consider what happens when a thread unlocks a stack-locked object.
      // It attempts to use CAS to swing the displaced header value from the
      // on-stack basiclock back into the object header.  Recall also that the
      // header value (hashcode, etc) can reside in (a) the object header, or
      // (b) a displaced header associated with the stack-lock, or (c) a displaced
      // header in an objectMonitor.  The inflate() routine must copy the header
      // value from the basiclock on the owner's stack to the objectMonitor, all
      // the while preserving the hashCode stability invariants.  If the owner
      // decides to release the lock while the value is 0, the unlock will fail
      // and control will eventually pass from slow_exit() to inflate.  The owner
      // will then spin, waiting for the 0 value to disappear.   Put another way,
      // the 0 causes the owner to stall if the owner happens to try to
      // drop the lock (restoring the header from the basiclock to the object)
      // while inflation is in-progress.  This protocol avoids races that might
      // would otherwise permit hashCode values to change or "flicker" for an object.
      // Critically, while object->mark is 0 mark->displaced_mark_helper() is stable.
      // 0 serves as a "BUSY" inflate-in-progress indicator.


      // fetch the displaced mark from the owner's stack.
      // The owner can't die or unwind past the lock while our INFLATING
      // object is in the mark.  Furthermore the owner can't complete
      // an unlock on the object, either.
      markOop dmw = mark->displaced_mark_helper();
      assert(dmw->is_neutral(), "invariant");

      // Setup monitor fields to proper values -- prepare the monitor
      m->set_header(dmw);

      // Optimization: if the mark->locker stack address is associated
      // with this thread we could simply set m->_owner = Self.
      // Note that a thread can inflate an object
      // that it has stack-locked -- as might happen in wait() -- directly
      // with CAS.  That is, we can avoid the xchg-NULL .... ST idiom.
      m->set_owner(mark->locker());
      m->set_object(object);
      // TODO-FIXME: assert BasicLock->dhw != 0.

      // Must preserve store ordering. The monitor state must
      // be stable at the time of publishing the monitor address.
      guarantee(object->mark() == markOopDesc::INFLATING(), "invariant");
      object->release_set_mark(markOopDesc::encode(m));

      // Hopefully the performance counters are allocated on distinct cache lines
      // to avoid false sharing on MP systems ...
      OM_PERFDATA_OP(Inflations, inc());
      if (log_is_enabled(Debug, monitorinflation)) {
        if (object->is_instance()) {
          ResourceMark rm;
          log_debug(monitorinflation)("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
                                      p2i(object), p2i(object->mark()),
                                      object->klass()->external_name());
        }
      }
      if (event.should_commit()) {
        post_monitor_inflate_event(&event, object, cause);
      }
      return m;
    }

    // CASE: neutral
    // TODO-FIXME: for entry we currently inflate and then try to CAS _owner.
    // If we know we're inflating for entry it's better to inflate by swinging a
    // pre-locked objectMonitor pointer into the object header.   A successful
    // CAS inflates the object *and* confers ownership to the inflating thread.
    // In the current implementation we use a 2-step mechanism where we CAS()
    // to inflate and then CAS() again to try to swing _owner from NULL to Self.
    // An inflateTry() method that we could call from fast_enter() and slow_enter()
    // would be useful.

    assert(mark->is_neutral(), "invariant");
    ObjectMonitor * m = omAlloc(Self);
    // prepare m for installation - set monitor to initial state
    m->Recycle();
    m->set_header(mark);
    m->set_owner(NULL);
    m->set_object(object);
    m->_recursions   = 0;
    m->_Responsible  = NULL;
    m->_SpinDuration = ObjectMonitor::Knob_SpinLimit;       // consider: keep metastats by type/class

    if (object->cas_set_mark(markOopDesc::encode(m), mark) != mark) {
      m->set_object(NULL);
      m->set_owner(NULL);
      m->Recycle();
      omRelease(Self, m, true);
      m = NULL;
      continue;
      // interference - the markword changed - just retry.
      // The state-transitions are one-way, so there's no chance of
      // live-lock -- "Inflated" is an absorbing state.
    }

    // Hopefully the performance counters are allocated on distinct
    // cache lines to avoid false sharing on MP systems ...
    OM_PERFDATA_OP(Inflations, inc());
    if (log_is_enabled(Debug, monitorinflation)) {
      if (object->is_instance()) {
        ResourceMark rm;
        log_debug(monitorinflation)("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
                                    p2i(object), p2i(object->mark()),
                                    object->klass()->external_name());
      }
    }
    if (event.should_commit()) {
      post_monitor_inflate_event(&event, object, cause);
    }
    return m;
  }
}
```

[monitor]:https://docs.oracle.com/javase/specs/jls/se10/html/jls-8.html#d5e13622
[ir]:http://hg.openjdk.java.net/jdk/jdk/file/6659a8f57d78/src/hotspot/share/interpreter/interpreterRuntime.cpp
[sr]:http://hg.openjdk.java.net/jdk/jdk/file/6659a8f57d78/src/hotspot/share/runtime/sharedRuntime.cpp
[sy]:https://hg.openjdk.java.net/jdk/jdk/file/896e80158d35/src/hotspot/share/runtime/synchronizer.cpp
[bl]:http://hg.openjdk.java.net/jdk/jdk/file/6659a8f57d78/src/hotspot/share/runtime/biasedLocking.cpp
