---
title: "Intro to Navigation"
date: 2020-08-23T11:11:13+08:00
description: "Introduction to Jetpack's Navigation Component"
tags: ["navigation", "jetpack"]
categories: ["android"]
draft: false
---

The Jetpack's Navigation component helps us implement navigation feature in a single MainActivity, it contains 3 core parts: Navigation graph, NavHost, NavController.

<!-- more -->

## NavHostFragment

`NavHostFragment` acts as a host for the fragments in the navigation graph.

As the user moves between destination defined in the navigation graph, the `NavHostFragment` swaps the fragments in and out and manages the Fragment back stack.

In the `activity_main.xml` layout file, the `NavHostFragment` is represented by a `fragment` element with the name `android:name="androidx.navigation.fragment.NavHostFragment"`.

## BackStack in Navigation

Each time user goes to a new destination on the device, Android adds that destination to the *back stack*.

When user presses the Back Button, the app goes to the destination that's at the top of the back stack. By default, the top of the back stack is the screen that the user last viewed.

### Set the pop behavior for the navigation actions

We can manage the back stack by setting the pop behavior for the actions that connect the fragments:

+ The `popUpTo` attribute of an action "pops up" the back stack to a given destination before navigating.
+ If the `popUpToInclusive` attribute is `false` or is not set, `popUpTo` removes destinations up to the specified destination, but leaves the specified destination in the back stack.
+ If `popUpToInclusive` is set to true, the `popUpTo` attribute removes all destinations up to and including the given destination from the back stack.
+ If `popUpToInclusive` is `true` and `popUpTo` is set to the app's starting location, the action removes all app destination from the back stack. The Back button takes the user all the way out of the app.

## The Up button

The Up button appears at the top left of the [app bar][ab](sometimes called action bar) The Up button navigates "upwards" within the app's screens, based on the hierarchical relationships between screens.

The navigation controller's [NavigationUI][ni] library integrates with the app bar to allow user tap Up button to get back to the app's home screen from anywhere in the app.

link step:

### NavHostFragment in FragmentContainerView element

+ In `onCreate()`, call:

``` kotlin
val navHostFragment = supportFragmentManager.findFragmentById(R.id.nav_host_fragment) as NavHostFragment
val navController = navHostFragment.navController
NavigationUI.setupActionBarWithNavController(this, navController)
```

+ Override `onSupportNavigateUp()` method to call `navigateUp()` in the navigatiohn controller:

``` kotlin
override fun onSupportNavigateUp(): Boolean {
    val navHostFragment = supportFragmentManager.findFragmentById(R.id.nav_host_fragment) as NavHostFragment
    val navController = navHostFragment.navController
    return navController.navigateUp()
}
```

### NavHostFragment in fragment element

+ In `onCreate()`, call:

``` kotlin
val navController = findNavController(R.id.nav_host_fragment)
NavigationUI.setupActionBarWithNavController(this, navController)
```

+ Override `onSupportNavigateUp()` method to call `navigateUp()` in the navigatiohn controller:

``` kotlin
override fun onSupportNavigateUp(): Boolean {
    val navController = findNavController(R.id.nav_host_fragment)
    return navController.navigateUp()
}
```

## Safe Args Plugin

When we needs to pass parameters from one fragment to another. To prevent bugs in these transactions and make them type-safe, we use a Gradle plugin called [Safe Args][sa]. The plugin generates `NavDirection` class, and we could use these classes to pass parameters.

We used a `Bundle` to pass data from fragment A to fragment B in the past. These lead two kinds of errors:

+ Type mis-match error. For example, if Fragment A sends a String but Fragment requests an Integer from `Bundle`, the request returns the default value of integer is 0.
+ Missing key errors. If fragment B requests an argument that isn't set in the bundle, the operation return `null`.

All above errors might make app misbehave or crash in runtime, but we can use *Safe Argu* to catch these errors in compile time, Safe Args is a Gradle plugin that generates code and classes that help to detect errors at compile-time that might not otherwise be surfaced until the app runs.

[links]:https://codelabs.developers.google.com/codelabs/kotlin-android-training-add-navigation/index.html#12
[ab]:https://developer.android.com/topic/libraries/architecture/navigation/navigation-ui#top_app_bar
[ni]:https://developer.android.com/topic/libraries/architecture/navigation/navigation-ui
[sa]:https://developer.android.com/topic/libraries/architecture/navigation/navigation-pass-data#Safe-args
[practice1]:https://medium.com/google-developer-experts/using-navigation-architecture-component-in-a-large-banking-app-ac84936a42c2
