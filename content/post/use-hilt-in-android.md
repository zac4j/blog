---
title: "Use Hilt in Android"
date: 2020-08-06
description: "Activity Hilt"
tags: ["hilt", "dagger", "di"]
categories: ["hilt", "dagger", "di"]
draft: false
--- 

> If you want to use dagger in  your project, you may have 2 or 3 different sources, different samples, different blog posts. All of them use a different set-up, and all of them use Dagger in a different way, so it's very difficult to understand all the topics and relate them together. So that's why Google create a common guidance, a common ground, which is called Hilt.

<!--more-->

Hilt is an opinionated dependency injection library for Android that reduces the boilerplate of using manual DI in your project. Doing [manual dependency injection][mdi] requires constructing every class and its denpendencies by hand and using **containers** to reuse and manage **dependencies**.

## Dependencies Container

Annotating Android classes with `@AndroidEntryPoint` creates a **dependencies container** that follows the Android lifecycle.

``` kotlin
@AndroidEntryPoint
class MainFragment: Fragment() {
    ...
}
```

Hilt currently supports the following Android types: `Application` by using(`@HiltAndroidApp`), Activity, Fragment, Service and BroadcastReceiver.

Hilt only supports activities that extend [FragmentActivity][fa] and fragments that extend Jetpack library [Fragment][fx](Note:Hilt does not support retained fragments).

### Container

A *container* is a class which is in charge of providing dependencies in your codebase and knows how to create instances of other types of your app. It manages the graph of dependencies required to provide those instances by creating them and managing their lifecycle.

A container exposes methods to get instances of the types it provides. Those methods can always return a different instance or the same instance. If the method always provides the same instance, we say that the type is *scoped* to the container.

### @HiltAndroidApp

Annotating the Application class with `@HiltAndroidApp`, we add a *container* that is attached to the app's lifecycle:

``` kotlin
@HiltAndroidApp
class App:Application() {
    ...
}
```

`@HiltAndroidApp` triggers Hilt's code generation including a base class for our application that can use dependency injection. The application container is the **parent container**ƒ of the app, which means that other containers can access the dependencies that it provides.

## Provide dependencies

### Constructor Inject

To tell Hilt how to provide instance of type, add the `@Inject` annotation to the constructor of the class we want to be injected.

``` kotlin
class Logger @Inject constructor() {
    ...
}
```

#### Scoping instance

Hilt can produce different containers that have different lifecycles, there are different annotations that scope to those containers:

+ `@Singleton`: Application container
+ `@ActivityScoped`: Activity container
+ `@FragmentScoped`: Fragment container
+ `@ViewScoped`: View container

So we could annotating *Logger*.

``` kotlin
@Singleton
class Logger @Inject constructor() {
    ...
}
```

### Modules

For types that **cannot be constructor injected**, such as interface or classes are not contained in our project, we should use **Hilt modules**. An exemple of this is `OkHttpClient`, we need to use its builder to create an instance.

A Hilt module is a class annotated with `@Module` and `@InstallIn`. `@Module` tells Hilt this is a module and `@InstallIn` tells Hilt the bindings are specifying in which Hilt component.

For each Android class that can be injected by Hilt, there's an associated [Hilt Component][hiltc].

``` kotlin
@InstallIn(ApplicationComponent::class)
@Module
object DatabaseModule {
    ...
}
```

#### Providing instances with @Provides

We can annotate a function with `@Provides` in Hilt modules to tell Hilt how to provide types that cannot be constructor injected.

The function body of the `@Provides` annotated function will be executed every time Hilt needs to provide an instance of that type. The return type of the `@Provides` annotated function tells Hilt the bindings type or how to provide instances of that type.

``` kotlin
@InstallIn(ApplicationComponent::class)
@Module
object DatabaseModule {

    /**
     * We always want Hilt provide the same database instance, so annotate this
     * method with @Singleton.
     *
     * @param appContext: Each Hilt container comes with a set of default bindings that can be
     * injected as dependencies into our custom bindings, to access this appContext, we annotate
     * the filed with @ApplicationContext.
     */
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext appContext: Context): AppDatabase {
        return Room.databaseBuilder(
            appContext,
            AppDatabase::class.java,
            "loggind.db"
        ).build()
    }

    /**
     * provide AppDatabase dependency.
     */
    @Provides
    fun provideLogDao(database: AppDatabase): LogDao {
        return database.logDao()
    }

}
```

## Inject dependencies

We can make Hilt inject instances of different types with the `@Inject` annotation on the fields we want to be injected:

``` kotlin
@AndroidEntryPoint
class LoginFragment:Fragment() {
    // note that fields injected by Hilt cannot be private.
    @Inject lateinit var logger: Logger
    @Inject lateinit var formatter: Formatter
}
```


[mdi]:https://developer.android.com/training/dependency-injection/manual
[cl]:https://developer.android.com/codelabs/android-hilt?return=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fandroid-week6-jetpack%23codelab-https%3A%2F%2Fdeveloper.android.com%2Fcodelabs%2Fandroid-hilt#3
[bp]:https://github.com/google/dagger/issues/900
[fa]:https://developer.android.com/reference/androidx/fragment/app/FragmentActivity
[hiltc]:https://developer.android.com/training/dependency-injection/hilt-android#generated-components
[actx]:https://developer.android.com/training/dependency-injection/hilt-android#component-default
[labs]:https://codelabs.developers.google.com/codelabs/android-hilt#0
