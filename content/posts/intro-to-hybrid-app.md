---
title: "Intro to Hybrid App"
date: 2020-07-04T15:45:14+08:00
keywords: ["hybrid"]
description: "Android Hybrid Development Introduction"
tags: ["hybrid"]
categories: ["android"]
draft: false
---

## Hybrid App 的几种方案

+ **PhoneGap/Cordova**
  
  WebView 作为用户界面，以 Javascript 作为基本逻辑，以及和中间件通信，再由中间件访问底层 API 的方式，进行 App 开发。
+ **React Native/Weex**
  
  使用前端技术开发，通过 JSBridge 将 Javascript 代码解析的 Virtual DOM 传递到 Native 并使用原生进行渲染。
+ **Native/H5 混合开发**
  
  在原生的架构基础上，嵌入 WebView 业务逻辑，一般有 Native 和 Web 前端开发人员组成。Native 写好基本的架构以及 API 让 Web 开发人员开发页面以及大部分渲染。

## PhoneGap/Cordova

PhoneGap 由 Apache 接管后改名为 Cordova，实际上是一个项目。Cordova 其实不应该称为 Hybrid 方案。因为，它的目标是全面使用前端技术开发移动应用，而不是前端和原生混合使用。不过，这些框架和 React Native 的目标是一致的：使用前端技术开发移动应用，提高工程效率。

Cordova 让前端技术尽可能多的完成开发工作，只有在前端无法直接调用的原生系统功能方面提供了前端可用的接口。主流的 Cordova 项目都将业务逻辑实现在一个 WebView 中，目标是让开发者使用前端技术就能完成一个应用开发。这种做法需要一个前提——前端技术可以解决移动开发的所有需求。

### Cordova 架构图

![cordova arch](https://user-gold-cdn.xitu.io/2018/7/23/164c7814e247d497?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

+ Web App 是开发人员编写代码的地方，应用以网页的形式呈现，在一个 index.html 的本地页面文件中引用所需要的各种 Web 资源，如 CSS、Javascript、图片和影音文件等。App 的配置保存在 config.xml 文件中。
+ WebView 层用来呈现 UI。
+ Plugin 主要用于在 Javascript 代码中调用各平台的 native 功能。Cordova 项目已包含一些核心的插件，如电池、摄像头、通讯录等。开发人员也可以自定义插件，其原理就是 Native 获取 Javascript 环境上下文，并直接在上面挂在对象或者方法，

### Cordova 的优缺点

#### 优点

+ 快速构建UI，全 Web 开发一定程度上有利于 Web 前端人员快速开发。
+ 样式统一，在不同平台可以获得相同的交互和展示。
+ 便于调试，可以在浏览器上进行调试，工具丰富。

#### 缺点

+ 兼容性，部分前端框架或语法在低版本 Android 手机上无法执行。
+ 功能有局限性，只适用于富UI场景，与底层交互需要做复杂的处理。

## Native/H5 混合开发

### 原生基础架构 Infra

+ Hybrid-Navigator，使用 URL 来标识每个页面，在 App 中指明 URL 跳转到此页面。所以需要一个路由表，根据 URL 找到。
+ Hybrid-Router，可以传递复杂对象，CRUD 功能。
+ Hybrid-Scheme，使用 Scheme 标识 web 调用原生的功能，比如修改 WebView 标题栏和状态栏颜色，获取 user_code 等。
+ Hybrid-Container，js 代码运行容器。它其实是一个内嵌浏览器（WebView），Container 需要提供一些必要的原生支持，比如 Cookie，OAuth 授权，图片缓存等。

### Native/H5 技术原理——JsBridge

#### H5 如何调原生方法

基于 WebView 的机制和开发的 API，主要有三种方式：

+ API 注入，原理就是 Native 获取 Javascript 环境的上下文，并在上面挂载对象或者方法，使 js 可以直接调用。
+ WebView 响应 js prompt/console/alert 方法，对应 WebChromeClient 中的 `onJsPrompt()`/`onJsConsole()`/`onJsAlert()` 方法。
+ WebView 拦截 URL Scheme，对应 WebViewClient 中的 `shouldOverrideUrlLoading()` 方法。

#### Native 如何调 H5 函数

原生 WebView 可以直接执行 js 代码

+ iOS:stringByEvaluatingJavaScriptFromString
  
  `webview.stringByEvaluatingJavaScriptFromString("alert('NativeCall')")`

+ Android 4.4 以下
  
  `mWebView.loadUrl("javascript:JSBridge.trigger('NativeCall')")`

+ Android 4.4+
  
  ``` java
  mWebView.evaluateJavascript("JSBridge.trigger('NativeCall')", new ValueCallback<String>() {
      @Override
      public void onReceiveValue(String value) {
          // js callback
      }
  });
  ```

#### 定义拦截规则

我们根据公司业务会制定一套 URL scheme 规则，比较类似常用的 `https://` 或 `file:///`，不过我们自定义的 scheme 可以是任意样式，比如：

`zapp://action?param1=xxx&param2=xxx`

这样定义好的 scheme，h5 通过创建 iframe 方式发送请求。

#### Scheme 的拦截

+ iOS 上：shouldStartLoadWithRequest
+ Android: `WebViewClient.shouldOverrideUrlLoading()`

#### Scheme 的回调

一般 Native 拦截 scheme ，处理完成后，调用 Bridge 的 callback 方法。

``` java
mWebView.evaluateJavascript("JSBridge.trigger('NativeCall')", new ValueCallback<String>() {
    @Override
    public void onReceiveValue(String value) {
        // js callback
    }
});
```

## React Native 实现原理

React Native 并没有运行在浏览器引擎（WebView/V8）中。不像大部分前端框架是简化 DOM 树的处理，React Native 和 DOM 没有关系。React Native 重写了大部分系统 UI 组件，并在框架内处理好了 Javascript 和原生组件之间的调用和通信。开发者可以通过 React 风格的 Javascript 代码使用这些 React Native UI 组件来构建应用的UI。可以认为 React Native 实现了一个渲染引擎，使用 React 作为渲染引擎的编程语言。

## 页面调用方案

简单的页面跳转，从 MainScreen 跳转到 SecondaryScreen

Android 中的实现：

``` kotlin
startActivity(Intent(this, SecondaryScreen.class))
```

iOS 中的实现：

``` oc
SecondaryViewController *secondaryViewController = [[SecondaryViewController alloc] init];
[self.navigationController pushViewController:secondViewController animated:YES];
```

随着项目逐渐增大，我们需要做一些模块化的工作。模块化基本的要求是解除模块间的依赖，我们需要一个间接的调用目标页面的方案。

### openURL 方案

在这些间接方案中，有很多是基于 iOS 提供的 UIApplicationDelegate 中的用于响应 URL 调用的钩子方法。

在 iOS 8 中，该方法为：

``` oc
application:openURL:sourceApplication:annotation:
```

iOS 9 中该方法被标记为 Deprecated，并提供了一个替代方法：

``` oc
application:openURL:options:
```

我们将这两个方法称为 `openURL` 钩子方法。

如果应用注册了一个 URL scheme。任何第三方应用，以及应用自身对该 URL 的调用，都会调起以上 `openURL` 钩子方法。这就提供了一个在应用间通信的方法。同时这也是目前为止，iOS 系统中，应用间通信的唯一方式。

## React Native

React Native 需要的技术栈比较特别，比较适合 Web 经验特别丰富，前端工程师特别优秀，但又缺乏优秀的客户端工程师的情况，React Native 是一个快速切入移动应用市场的技术选择。

## 常见问题

### 内存管理

根据 adb shell 查看 app 占用内存信息

``` shell
# 查看 app 基本信息
Zac:Sample Zac$ adb shell ps | grep com.guotai.dazhihui
u0_a674       9858   669 2205104 268448 0                   0 S com.guotai.dazhihui
u0_a674      10039   669 1973628 101852 0                   0 S com.guotai.dazhihui:remote
u0_a674      10129   669 1988412 113528 0                   0 S com.guotai.dazhihui:pushservice

# 未开启 WebView 时的内存占用信息
Zac:Sample Zac$ adb shell dumpsys meminfo com.guotai.dazhihui
Applications Memory Usage (in Kilobytes):
...
App Summary
                       Pss(KB)
                        ------
           Java Heap:    21264
         Native Heap:    35988
                Code:    15648
               Stack:       52
            Graphics:    72324
       Private Other:     4528
              System:    18578

               TOTAL:   168382       TOTAL SWAP PSS:      420
...

# 开启 WebView 之后的内存占用
...
App Summary
                       Pss(KB)
                        ------
           Java Heap:    25136
         Native Heap:    55420
                Code:    51480
               Stack:       52
            Graphics:    74704
       Private Other:     6244
              System:    29544

               TOTAL:   242580       TOTAL SWAP PSS:      376

内存
```

[rexxar]:https://lincode.github.io/Hybrid-Rexxar
[hybrid]:https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2650993147&idx=3&sn=685217d04dec453448e2cf53e5e0e75d&chksm=bdbf09a88ac880beab79ac59fa6a558a97bc8fd16b6da03b1e70a722a4f53420597e1e4a9684&scene=27#wechat_redirect
[hybrid]:https://github.com/xd-tayde/blog/blob/master/hybrid-1.md
[mpaas]:https://zhuanlan.zhihu.com/p/102645754
