---
title: "Introduction to Custom View"
date: 2020-09-06T11:11:13+08:00
tags: ["custom view"]
description: "Custom view introduction"
categories: ["Custom View"]
draft: true
---

To create a custom view we can either extend an existing `View` subclass(like EditText), or create our own subclass of View. By extending View directly, we can create an interactive UI element of any size and shape by overriding the `onDraw()` method for the View to draw it.

<!--more-->

This article resource from official Android [custom view][cv] codelab. I think you'll get the most value if you work through this codelab in sequence.

## Understanding custom views

Views are the basic blocks of an app's UI. the View class provides many subclasses, referred to as *UI widgets*, that cover many of the needs of a typical Android app's user interface.

UI building blocks such as EditText or TextView are subclasses that extend the View class. To save time and development effort, we can extend one of these View subclasses.

To create our custom view from scratch, extend the View class itself. Our code overrides `View` methods to define the view's appearance and functionality. Key to create our custom view is that we are responsible for **drawing the entire UI element of any size and shape to the screen**. If we subclass an existing view such as `Button`, that class handles drawing for us.

### General steps to create a custom view

+ Create a custom view class that extends `View`, or extends a `View` subclass(such as `TextView` or `Button`)
+ If we extend a subclass of View(like Button), override only the behavior or aspect of the appearance methods that we want to change.
+ If we extend View class, draw the custom view's shape and control its appearance by overriding View methods such as `onDraw()` and `onMeasure()` in the new class.
+ Add code to respond to user interaction and, if necessary, redraw the custom view.
+ Use the custom view class as UI widget in activity's XML layout. We can also define custom attributes(in `attrs.xml` file) for the view, to provide customization for the view in different layouts.

## Drawing custom views

When we creating custom view from scratch(by extending `View`), we are responsible for drawing the entire view each time the screen refreshes, and for overriding the View methods that handle drawing. In order to properly **draw a custom view** that extends View, we need to follw this step:

+ Calculate the view's size when it first appears, and each time that view's size changes, by overriding the `onSizeChanged()` method.
+ Override the `onDraw()` method to draw the custom view, using a `Canvas` object styled by a `Paint` object.
+ Call the `invalidate()` method when responding to a user click that changes how the view is drawn to invalidate the entire view, thereby forcing a call to `onDraw()` to redraw the view.

The `onDraw()` method is called every time that screen refreshes, which can be many time a second. For performance reasons and to avoid visual glitches, we should do as little work as possible in `onDraw()`. In particular, don't place allocations in `onDraw()`, because allocations may lead to garbage collection that may cause a visual stutter.

The `Canvas` and `Paint` classes offer a number of useful **drawing shortcuts**:

+ Draw text using `drawText()`. Specify the typeface by calling `setTypeface()`, and the text color by calling `setColor()`.
+ Draw primitive shapes using `drawRect()`, `drawOval()` and `drawArc()`. Change whether that shapes are filled, outlined, or both by calling `setStyle()`.
+ Draw bitmaps using `drawBitmap()`.

> **Note**: in apps that have a deep view hierarchy, we can also override the `onMeasure()` method to accurately define how custom view fits into the layout. That way, the parent layout can properly align the custom view. The `onMeasure()` method provides a set of `measureSpecs` that we can use to determine out view's height and width. Learn more [how android draws][had].

## Change custom view behavior

### Add view interactivity

Normally, with a standard Android view, we implement `OnClickListener()` to perform an action when the user clicks that view. For a custom view, we should:

+ Set the view's isClickable property to true. This enables custom view to respond to clicks.
+ Implement the `View` class's `performClick()` to perform operations when the view is clicked.
+ Call the `invalidate()` method. This tells the Android system to call the `onDraw()` method to redraw the view.

### Use custom attributes with custom view

To use a custom attributes we should:

+ Create and open `res/values/attrs.xml`
+ Inside `<resource>`, add a `<declare-styleable>` resource element.
+ Inside the `<declare-styleable>` resource element, add three `attr` elements, one for each attribute, with a `name` and `format`. The `format` is the attribute's type, such as `color` or `string`.

The sample code is:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="CustomView">
        <attr name="customColor" format="color" />
        <attr name="customString" format="string" />
        <attr name="customInteger" format="integer" />
    </declare-styleable>
</resources>
```

> Note: the `attr` name is, by convention, the same name as the name ofthe class that defines the custom view. Although it's not strictly necessary to follow this convention, Android Studio depend on this naming convention to provide statement completion.

+ Open `fragment_custom_ui.xml` layout file, in `CustomView`, add attribute for `customColor`, `customString`, `customInteger`, and set their value. Use `app:` as preface for the custom attribute rather than `android:` because we custom attribute belong to the `schemas.android.com/apk/res/app_package_name` namespace rather than the android namespace.

``` xml
<com.zac4j.ui.widget.CustomView
    android:id="@+id/customView"
    android:layout_width="200dp"
    android:layout_height="200dp"
    android:layout_marginLeft="8dp"
    android:layout_marginTop="8dp"
    android:layout_marginRight="8dp"
    android:background="@android:color/darker_gray"
    app:customColor="#009688"
    app:customString="@string/hello"
    app:customInteger="0x101"
    app:layout_constraintLeft_toLeftOf="parent"
    app:layout_constraintRight_toRightOf="parent"
    app:layout_constraintTop_toBottomOf="@id/customViewLabel"/>
```

In order to use the attributes in `CustomView` class, we need to retrieve them. They are stored in an `AttributeSet`, which is handed to out class upon creation, if it exists. We retrieve the attributes in `init`, and assign the attribute values to local vaributes for caching.

+ Open the `CustomView.kt` class file.
+ Inside the `CustomView` declare vaributes to cache the attribute values.

``` kotlin
private var customColor = 0
private var customString = ""
private var customInt = 0
```

+ Inside the `init` block, add the following code using the `withStyledAttributes` extension function. We supply the attributes and view, and set the local vaributes. Importing `withStyledAttributes` will also import the right `getColor` function.

```kotlin
context.withStyledAttributes(attrs, R.styleable.CustomView) {
   customColor = getColor(R.styleable.CustomView_customColor, 0)
   customString = getString(R.styleable.CustomView_customString, "")
   customInt = getInt(R.styleable.CustomView_customInteger, 0)
}
```

> Android and the Kotlin extension library ([android-ktx][ktx]) do a lot of work for you here! The android-ktx library provides Kotlin extensions with a strong quality-of-life focus. For example, the [withStyledAttributes][wsa] extension replaces a significant number of lines of rather tedious boilerplate code. For more on this library, check out the documentation, and the [original announcement][oa] blog post!

+ Use the local variables in `onDraw()` to set the color and label to the current view.

[oa]:https://android-developers.googleblog.com/2018/02/introducing-android-ktx-even-sweeter.html
[wsa]:https://android.github.io/android-ktx/core-ktx/androidx.content/android.content.-context/index.html
[ktx]:https://android.github.io/android-ktx/core-ktx/index.html
[cv]:https://codelabs.developers.google.com/codelabs/advanced-andoid-kotlin-training-custom-views
[ref]:https://developer.android.com/guide/topics/ui/custom-components.html
[had]:https://developer.android.com/guide/topics/ui/how-android-draws.html
