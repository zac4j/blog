---
title: "Java Dynamic Proxy"
date: 2022-05-15T21:51:47+08:00
description: "Java Dynamic Proxy a simple intro"
tags: ["dynamic proxy"]
categories: ["java"]
draft: true
---

动态代理是一种代理机制，我们知道，代理可以看作是对调用目标的一个包装，这样我们对目标代码调用不是直接发生的，而是通过代理完成。很多动态代理场景，也可以看作是装饰器（Decorator）模式的应用。

<!-- more -->

## JDK Proxy

通过代理可以让调用者（Caller）和实现者（Implementation）之间解耦。比如进行 RPC 调用，框架内的寻址、序列化、反序列化等，对于调用者往往没有太大意义内容，通过代理，可以提供更加友善的接口。

可以看一个简单的例子，在生产环境中，我们可以轻松扩展类似逻辑进行诊断、限流等。

``` java
public class DynamicProxySample {
    public static void main(String[] args) {
        Bonjor bonjor = new Bonjor();
        SimpleInvocationHandler handler = SimpleInvocationHandler(bonjor);
        // 构造代码实例
        Hello proxyHello = (Hello)Proxy.newProxyInstance(Bonjor.class.getClassLoader(), Bonjor.class.getInterfaces(), handler);
        // 调用代理方法
        proxyHello.sayHi();
    }
}

interface Hello {
    void sayHi();
}

class Bonjor implements Hello {
    @Override
    public void sayHi() {
        System.out.println("Bonjor!");
    }
}

class SimpleInvocationHandler implements InvocationHandler {
    private Object target;
    public SimpleInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Calling sayHi method");
        Object result = method.invoke(target, args);
        return result;
    }
}
```

上面的 JDK Dynamic Proxy 的例子，简单的实现了动态代理的构建和代理操作。从 API 设计和实现的角度，这种实现有局限性，因为它是以接口为中心，相当于添加了一种对于被调用者没有太大意义的限制。我们实例化的是 Proxy 对象，而不是真正的被调用类型，这在实践中还是可能带来各种不便和能力退化。

## todo 其他动态代理方式

+ Javassist
+ ASM

## 动态代理应用

动态代理应用非常广泛，虽然最初多是因为 RPC 等使用进入我们视线，但是动态代理的使用场景远远不仅如此，它完美符合 Spring AOP 等切面编程。
简单来说它可以看作是对 OOP 的一个补充，因为 OOP 对于跨越不同对象或类的分散、纠缠逻辑表现力不够，比如在不同模块的特定阶段做一些事情，类似日志、用户鉴权、全局性异常处理、性能监控，甚至事务处理等，你可以参考下面这张图。

![aop](/img/aop-used.webp)

AOP 通过（动态）代理机制可以让开发者从这些繁琐事项中抽身出来，大幅度提高了代码的抽象程度和复用度。从逻辑上来说，我们在软件设计和实现中的类似代理，如 Facade、Observer 等很多设计目的，都可以通过动态代理优雅地实现。
