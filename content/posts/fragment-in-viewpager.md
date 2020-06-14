---
title: "Fragment in ViewPager"
date: 2020-05-25T20:55:31+08:00
description: "Fragment 在 ViewPager 中的应用"
tags: ["fragment", "viewpager"]
categories: ["fragment"]
author: "Zac"
---

### 使用 ViewPager

通常我们使用 ViewPager + TabLayout 主要有这些步骤：

+ 页面的布局结构:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity"
    >

  <com.google.android.material.appbar.AppBarLayout
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:theme="@style/AppTheme.AppBarOverlay"
      >

    ...

    <com.google.android.material.tabs.TabLayout
        android:id="@+id/tabs"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="?attr/colorPrimary"
        />

    <androidx.viewpager.widget.ViewPager
      android:id="@+id/view_pager"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      app:layout_behavior="@string/appbar_scrolling_view_behavior"
      />
  </com.google.android.material.appbar.AppBarLayout>
</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

+ 在 PagerAdapter 中会指定我们使用的 Fragment:

```kotlin
class SectionsPagerAdapter(private val context: Context,
  fm: FragmentManager
) : FragmentPagerAdapter(fm, BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {

  override fun getItem(position: Int): Fragment {
    return SampleFragment.newInstance(position)
  }

  override fun getCount(): Int {
    return TAB_TITLES.size
  }
```

+ 给 ViewPager 设置 PagerAdapter:

```kotlin
viewPager.adapter = pagerAdapter
```

+ TabLayout 绑定 ViewPager:

```kotlin
tabs.setupWithViewPager(viewPager)
```

### PagerAdapter

目前官方提供的 PagerAdapter 有 2 种，在源码中已介绍各自的使用场景(假设 offscreenPageLimit=1)：

+ FragmentPagerAdapter
  
  适合有少量的几个 Fragment 页面，比如一组 tabs，用户访问的每个页面都会保存在内存中，内存占用可能比较大。

  对于满足 `offscreenPageLimit` 的 Fragment ，其回调的方法是：onStop -> onDestroyView.
  
+ FragmentStatePagerAdapter
  
  适合有大量 Fragment 页面，类似于 ListView 工作的场景，页面不可见时，整个 Fragment 都会被销毁，只有 SavedState 会被保存。相较 FragmentPagerAdapter 内存占用更小。

  对于满足 `offscreenPageLimit` 的 Fragment ，其回调的方法是：onStop -> onDestroyView -> onDestroy -> onDetach.

### 保存和恢复 State

#### 保存 Fragment 的 State

对于使用 `FragmentStatePagerAdapter` 的场景，我们可能需要在 Fragment 销毁时保存一些数据，通常的做法是在 `onSaveInstanceState(outState)` 中保存，在 `onCreate(savedInstanceState)`、`onCreateView(savedInstanceState)` 或 `onActivityCreated(savedInstanceState)` 中恢复：

```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    // save instance state here
  }

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    savedInstanceState?.let {
      // restore instance state
    }
}
```

#### 保存 CustomView 的 State

官方提供的 UI 控件内部都实现了 `onSaveInstanceState` 和 `onRestoreInstanceState` 方法，在页面销毁-重建时，这些控件可以保存-恢复 ViewState。对于自定义的 View，我们也需要实现这两个方法：

```java
  @Override
  public Parcelable onSaveInstanceState() {
    Parcelable superState = super.onSaveInstanceState();
    ...
    SavedState ss = new SavedState(superState);
    ...
    return ss;
  }

  @Override
  public void onRestoreInstanceState(Parcelable state) {
    if (!(state instanceof SavedState)) {
      super.onRestoreInstanceState(state);
      return;
    }

    SavedState ss = (SavedState) state;
    super.onRestoreInstanceState(ss.getSuperState());
    ...
  }
```

[vp]:https://developer.android.google.cn/training/animation/screen-slide
[state]:https://inthecheesefactory.com/blog/fragment-state-saving-best-practices/en
