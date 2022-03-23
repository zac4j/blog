---
title: "Intro to Build System"
date: 2020-06-05T14:49:15+08:00
description: "Introduction to Android Build System"
tags: ["build", "gradle"]
categories: ["android", "gradle"]
author: "Zac"
---

### 构建流程（Build process）

简单来说构建流程就是涉及一系列工具和步骤将项目转换成 Android Application Package(APK)。典型的流程如下图所示：

![build-process][bp]

图中所示有4个步骤：

+ 编译器将源码转换为 DEX 文件(Dalvik 可执行文件，包括运行在 Android 设备的字节码)，并将其他所有内容转换成编译后的资源文件。
+ APK Packager 将 DEX 文件和编译后的资源文件合并到一个 APK 里。
+ APK Packager 使用 keystore 为 APK 签名。
+ 在生成最终的 APK 文件之前，Packager 会使用 [zipalign][zn] 工具

### 自定义构建配置

Gradle 和 Android 插件可以帮助我们修改 **build.gradle** 中的这些配置：

#### 构建类型 Build Types

构建类型定义 Gradle 在构建(building)和打包(packaging) App 时使用的某些属性，通常针对开发周期的不同阶段(different stages of development)进行配置。比如：

+ debug 构建类型支持 debug 选项，并使用 debug key 为 APK 签名。
+ release 构建类型则会压缩(shrink)和混淆(obfuscate) APK，并使用 release key 为 APK 签名。

构建 App 时必须指定一个构建类型。可以参考 [BuildType DSL][bt]，了解配置构建类型属性。

``` groovy
buildTypes {
  release {
    minifyEnabled false
    shrinkResources true
    proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
  }
  debug {
    applicationIdSuffix ".debug"
    debuggable true
  }
  staging {
    initWith debug
    manifestPlaceholders = [hostname:"internal.zacash.cn"]
    applicationIdSuffix ".debugStaging"
  }
}
```

#### 产品类型 Product Flavors

Product flavors 表示我们可以向用户发布 App 不同的版本，如免费版和付费版。我们可以自定义 product flavor 来使用不同的代码和资源。

Product flavors 支持与 `defaultConfig` 相同的属性，这是因为 `defaultConfig` 实际上属于 [ProductFlavor][pf] 类。

> Google Play 的[多 APK][multiple-apks] 分发应用，是将同一 applicationId 分配给所有的 flavor，并为每个 flavor 分配不同的 versionCode。

``` groovy
flavorDimensions "api", "mode"
productFlavors {
  demo {
    dimension "mode"
    applicationIdSuffix ".demo"
    versionNameSuffix "-demo"
  }
  full {
    dimension "mode"
    applicationIdSuffix ".full"
    versionNameSuffix "-full"
  }
  minApi21 {
    dimension "api"
    minSdkVersion 21
    versionCode 10000 + android.defaultConfig.versionCode
    versionNameSuffix "-minApi21"
  }
}
```

#### 构建变体 Build Variants

Build variants 是由 build type 和 product flavor 组成，也是 Gradle 用来构建应用的配置。配置 Build variants 就是指配置组成它的 build type 和 product flavor。

Gradle 创建的 build varuants 数量等于 productFlavors * buildTypes. Gradle 构建 APK 命名时，会先以高优先级的 flavorDimension 的 ProductFlavor 显示，而后是低优先级 flavorDimension 的 ProductFlavor，最后是 BuildType。比如上面的 ProductFlavor 和 BuildType 配置，在构建 APK 时对应的 Build Variants：

+ Build Variants: [minApi21][Demo, Full][Debug, Release]
+ APK 名称：app-[minApi21]-[demo, full]-[debug, release].apk

#### 清单条目 Manifest entries

我们可以在 build variants 中设置清单文件(AndroidManifest.xml)中的某些属性，这些构建值会替换清单文件中的现有值。如果我们要为模块生成多个 APK，这些 APK 有不同的应用名称、min SDK version 或 target SDK version，定义这些属性将会很有效。

#### 依赖项 Dependencies

构建系统会管理来自本地文件系统以及来自远程代码库的项目依赖项。

[bp]:https://developer.android.google.cn/images/tools/studio/build-process_2x.png
[zn]:https://developer.android.com/studio/command-line/zipalign
[bt]:https://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.BuildType.html
[pf]:http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.ProductFlavor.html
[multiple-apks]:https://developer.android.com/google/play/publishing/multiple-apks