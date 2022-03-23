---
title: "Activity Start"
date: 2020-04-22T08:10:27+08:00
description: "Activity Launch"
tags: ["activity", "launch"]
categories: ["android"]
---

### Task and Back Stack

Task 就是执行某项任务时开启的一系列 Activity 的集合，这些 Activity 会按照打开的顺序排列在回退栈 (Back Stack) 中。

#### Activity 的四种启动模式 (launchMode)

如通过 `startActivity(Intent)` 启动 Activity A

+ standard 模式：标准模式，创建 Activity A 实例并 push 到当前任务栈中
+ singleTop 模式：栈顶复用模式，如果当前栈顶是 Activity A ，直接复用并调用 `onNewIntent(Intent)`方法，不会创建新的实例
+ singleTask 模式：栈内复用模式，如果当前栈内含有 Activity A ，直接复用并调用 `onNewIntent(Intent)`方法，并清除栈内 Activity A  上方的所有 Activity 实例
+ singleInstance 模式：在新的任务栈开启 Activity A，如果新的任务栈和 Activity A 已创建，继续开启 Activity A 会调用其 `onNewIntent(Intent)`方法。

#### 三种 Intent 标记

`startActivity(Intent)` 方法可以添加 3 种 Intent 标记：

+ `FLAG_ACTIVITY_NEW_TASK`
  
  在新的任务栈中启动 Activity A，如果当前运行的任务栈中已含有 Activity A，则将任务栈内的 Activity A 恢复到前台并调用 `onNewInstent(Intent)` 方法，此 flag 的功能同 **singleTask** 启动模式相同。
+ `FLAG_ACTIVITY_SINGLE_TOP`
  
  如果待启动的 Activity A 已位于任务栈顶，则会调用该实例的 `onNewInstent(Intent)`，此 flag 的功能同 **singleTop** 启动模式。
+ `FLAG_ACTIVITY_CLEAR_TOP`
  
  如果待启动的 Activity A 已存在于当前任务栈内，则任务栈内该 Activity A 上方的所有 Activity 都会被销毁，Activity A 恢复前台并调用的 `onNewIntent(Intent)` 方法。*FLAG_ACTIVITY_CLEAR_TOP* 通常和 *FLAG_ACTIVITY_NEW_TASK* 结合使用，这是一种可以将另一个任务栈内已存在 Activity 响应 Intent 的方式。

#### taskAffinity 属性

taskAffinity 表示 Activity 倾向于归属哪个任务栈. 默认情况下，一个 App 内的所有 Activity 有相同的 affinity，即会被安排在同一任务栈下。我们可以通过 taskAffinity 修改 Activity 的归属任务栈。

使用场景：

+ 启动 Activity 的 intent 包含 `FLAG_ACTIVITY_NEW_TASK` 标记时。
  
  对于 Activity A 通过 `startActivity(Intent)` 方法启动 Activity B，在默认 standard 启动模式时，Activity B 会被 push 到 Activity A 所在的任务栈。但当传给 `startActivity(Intent)` 的 intent 使用 `FLAG_ACTIVITY_NEW_TASK` 标记时，系统会先寻找是否有和 Activity B 具有相同 affinity 的任务栈，如果有，则会将启动的 Activity B push 到该任务栈中，如果没有，创建新的任务栈。
  
  通过 `FLAG_ACTIVITY_NEW_TASK` 创建新的任务栈时，当用户按下 *HOME* 键离开该任务时，必须有方法供用户返回该任务。某些外部实体(如 Notification Manager) 通常在外部任务(external task)启动 Activity， 即在 `startActivity(Intent)` 的 intent 使用 `FLAG_ACTIVITY_NEW_TASK` 标记。如果你的 App 中某个 Activity 可能被外部实体(如 Notification Manager) 通过这种方式启动，需要确保该用户具有独立的方式(App Launcher)，返回到启动的任务。

+ 当 Activity 的 **allowTaskReparenting** 属性设为 `"true"`时。

#### 清除回退栈

默认情况下，App 置于后台一段时间后，系统会清除该 App 任务栈内除了 Root Activity 外所有的 Activity。当 App 再次回到前台时(回到该任务栈)时，只有 Root Activity 被恢复。Activity 有 3 种控制该特性的属性：

+ `alwaysRetainTaskState`
  
  当任务栈的 Root Activity 的该属性设为 `"true"` 时，系统不会再回收任务栈内的其他 Activity，所有 Activity 都会在栈内长时间保留。
+ `clearTaskOnLaunch`
  
  当任务栈的 Root Activity 的该属性设为 `"true"` 时，系统在会在离开该任务栈时就清除所有 Activity。
+ `finishOnTaskLaunch`
  
  与 `clearTaskOnLaunch` 相似，不过该属性仅适用于单一 Activity，不是整个任务栈。当某 Activity 的该属性设为 `"true"` 时，系统会在离开该任务栈时立即清除该 Activity。

#### 开启任务

一般来说 App 的入口 Activity 的 intent-filter 会设置成这样：

```xml
<activity ... >
    <intent-filter ... >
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    ...
</activity>
```

这种配置会在 Launcher App 中显示该 Activity 对应的的 icon 和 label，从而为用户提供启动 Activity 或从别的任务回到该任务的入口。

正如前面介绍通过 `FLAG_ACTIVITY_NEW_TASK` 创建新的任务，当用户按下 *HOME* 键离开该任务时，必须有方法供用户返回该任务，这种方法就是 activity launcher。因此对于可以启动新任务的2种启动模式, `singleTask` 和 `singleInstance`，应该仅在 Activity 的 intent filter 有 [ACTION_MAIN][am] 和 [CATEGORY_LAUNCHER][cl] 修饰时使用。

[as]:https://medium.com/martinomburajr/android-internals-1-how-android-starts-your-main-activity-8fcf80e65222
[ai]:https://android.jlelse.eu/android-application-launch-explained-from-zygote-to-your-activity-oncreate-8a8f036864b
[stack]:https://developer.android.com/guide/components/activities/tasks-and-back-stack
[am]:https://developer.android.com/reference/android/content/Intent#ACTION_MAIN
[cl]:https://developer.android.com/reference/android/content/Intent#CATEGORY_LAUNCHER
