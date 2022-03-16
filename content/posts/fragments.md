---
title: "Intro to Fragments"
date: 2020-05-07T20:30:50+08:00
tags: ["Fragment", "Lifecycle"]
description: "Fragment 生命周期和使用"
categories: ["Fragment"]
author: "Zac"
---

### Activity 和 Fragment 生命周期的关联

| Activity 状态 | Fragment 生命周期方法回调                                   | Fragment 状态                           |
|---------------|-------------------------------------------------------------|-----------------------------------------|
| Created       | onAttach(), onCreate(), onCreateView(), onActivityCreated() | Fragment 添加到 Activity 且视图已初始化 |
| Started       | onStart()                                                   | Fragment 活跃并可见                     |
| Resumed       | onResume()                                                  | Fragment 活跃并获取焦点                 |
| Paused        | onPause()                                                   | Fragment 暂停                           |
| Stopped       | onStop()                                                    | Fragment 停止并不再可见                 |
| Destroyed     | onDestroyView(), onDestroy(), onDetach()                    | Fragment 销毁                           |

#### Fragment 重要的生命周期方法的使用

+ `onAttach()`: Fragment 在被 attach 到宿主 Activity 时回调，可以在该方法里检查宿主 Activity 是否实现了某个接口。
+ `onCreateView()`: Fragment 的 XML 布局在这个回调方法里初始化，系统调用这个方法来绘制 Fragment 的 UI。
+ `onPause()`: 可以在 Fragment 销毁前在该回调方法保存必要数据或状态。
+ `onActivityCreated()`: 在宿主 Activity 的 `onCreate()` 方法调用后回调该方法。可以在该方法做最终的初始化，如检索 View `getView().findViewById(id)`或恢复状态(restore state)。如果 Fragmnt 设置了 `setRetainInstance()`，这个方法同样会被调用。
+ `onDestroyView()`: Fragment 的 UI 布局被移除时调用，可以在该方法标记 Fragment 不再可见。

### Fragment 和 Activity 间的连接

+ Fragment 获取宿主 Activity：在 Fragment attach 到宿主 Activity 后，可以通过 `getActivity()` 获得宿主 Activity 实例。
+ Activity 获取添加的 Fragment：可以通过 `FragmentManager.findFragmentById(containerId)`获取 Fragment 的实例。

#### Fragment 的 BackStack

对于 Activity，系统管理着 Activity 的回退栈，而对于 Fragment，是宿主 Activity 管理着回退栈，对于需要响应返回键的 Fragment 页面，我们需要调用 `addToBackStack(name)` 显式地将其添加到回退栈:

```kotlin
fragmentTransaction.add(R.id.fragment_container, fragment)
fragmentTransaction.addToBackStack(null)
fragmentTransaction.commit()
```

上面的代码中我们给 `addToBackStack()` 传入的 `null` ，如果是需要调用 `FragmentManager.BackStackEntry` 接口时，则必须传入具体的名称标记。
对于添加到 Activity 回退栈的 Fragment，当按下返回键，某个 Fragment 实例出栈时，其视图重建回调 `onCreateView()` 方法。

#### Activity 向 Fragment 传递数据

通常从 Activity 向 Fragment 传递数据，推荐使用 Fragment 的 `setArguments(Bundle)` 方法，具体的流程是：

```kotlin
companion object {
    fun newInstance(name: String) : SimpleFragment {
        val fragment = SimpleFragment()
        val args = Bundle(ARGS_NAME, name)
        fragment.arguments = argument
        return fragment
    }
}
```

在 Activity 初始化该 Fragment 传入 `name` 参数:

```kotlin
val name = "SimpleApp"
val ft = supportFragmentManager.beginTransaction()
val fragment = SimpleFragment.newInstance(name)
ft.add(R.id.fragment_container, fragment).commit()
```

在 Fragment 的 `onCreate()` 或 `onCreateView()` 的回调内，获取该参数：

```kotlin
if (arguments.containsKey(ARGS_NAME)) {
    mName = arguments.getString(ARGS_NAME);
}
```

#### Fragment 向 Activity 传递数据

对于 Activity 内实现 Fragment 内定义的接口，在 `Fragment.onAttach()`方法获取宿主 Activity:

```kotlin
interface OnPanelClickListener {
    void onClick()
}

override fun onAttach(context: Context) {
    super.onAttach(context)
    if (context is OnPanelClickListener) {
        mListener = context as OnPanelClickListener
    }
}
```

[gaf]:https://developer.android.com/reference/android/app/Fragment
[gf]:https://google-developer-training.github.io/android-developer-advanced-course-concepts/unit-1-expand-the-user-experience/lesson-1-fragments/1-2-c-fragment-lifecycle-and-communications/1-2-c-fragment-lifecycle-and-communications.html
