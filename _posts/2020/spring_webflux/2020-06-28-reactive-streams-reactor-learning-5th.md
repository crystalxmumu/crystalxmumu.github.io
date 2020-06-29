---
layout: post
title: Spring WebFlux 学习笔记 - (二) 开篇：学习响应式编程 Reactor (5) - reactor 转换类操作符（2）
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Reactor 学习系列, 响应式编程, reactor 操作符
keywords: Spring Web Flux, Reactive Streams, Reactor, reactor operators
---

# Reactor 操作符

上篇文章我们将 Flux 和 Mono 的操作符分了 11 类，我们来继续学习转换类操作符的第 2 篇。 

## 转换类操作符

转换类的操作符数量最多，平常过程中也是使用最频繁的。

### Flux#concatMap

将响应式流中元素顺序转换为目标类型的响应式流，之后再将这些流连接起来。该方法提供了 2 个重载方法，传递的第 2 个参数为内部生成响应式流的预取数量。见图知意：

<img src="/images/posts/spring_web_flux/reactor/07_operator_flux_concatMap.png" alt="concatMap 操作符" />

```java
Flux.range(3, 8)
    .concatMap(n -> Flux.just(n - 10, n, n + 10), 0)
    .subscribe(System.out::println);
```

### Flux#concatMapDelayError

concatMapDelayError 和 concatMap 区别在于，当内部生成响应式流发出 error 时，是否延迟响应 error 。该方法提供了 3 个重载方法，支持传递参数：是否延迟发出错误和预取数量。

```java
Flux.range(3, 8)
    .concatMapDelayError(n -> {
        if (n == 4) {
            return Flux.error(new NullPointerException());
        }
        return Flux.just(n - 10, n, n + 10);
    })
    .subscribe(System.out::println, System.err::println);
```

### Flux#concatIterable

concatIterable 和 concatMap 的区别在于 内部返回的类型不同，一个为 Iterable， 一个为 响应式流。见图知意：

<img src="/images/posts/spring_web_flux/reactor/08_operator_flux_concatIterable.png" alt="concatIterable 操作符" />


```java
Flux.range(3, 8)
    .publishOn(Schedulers.single())
    .concatMapIterable(n -> {
        if (n == 4) {
            throw new NullPointerException();
        }
        return Arrays.asList(n - 10, n, n + 10);
    })
    .onErrorContinue((e, n) -> System.err.println("数据：" + n + ",发生错误：" + e))
    .subscribe(System.out::println);
```

### elapsed

收集响应式流中元素的间隔发出时间，转换为 时间间隔 和 旧元素 组成的 Tuple2 的响应式流。见图知意：

<img src="/images/posts/spring_web_flux/reactor/09_operator_flux_elapsed.png" alt="elapsed 操作符" />

```java
Flux.interval(Duration.ofMillis(300))
    .take(20)
    .elapsed(Schedulers.parallel())
    .subscribe(System.out::println);
Thread.sleep(7000);
```

### expand

从上层节点逐层展开方式递归展开树形节点。

```java
Flux.just(16, 18, 20)
    .expand(n -> {
        if (n % 2 == 0) {
            return Flux.just(n / 2);
        } else {
            return Flux.empty();
        }
    })
    .subscribe(System.out::println);
```

### expandDeep

从上层节点逐个展开方式递归展开树形节点。expand 和 expandDeep 的区别在于展开方式不同，另外它俩都提供了 capacityHint 指定递归时初始化容器的容量。

```java
Flux.just(16, 18, 20)
    .expandDeep(n -> {
        if (n % 2 == 0) {
            return Flux.just(n / 2);
        } else {
            return Flux.empty();
        }
    })
    .subscribe(System.out::println);
```

# 总结

本篇我们介绍了 Reactor 部分的转换类操作符，讲解示例时都是单个操作符，相信大家都能理解。

由于最近学习时间不确定，内容比较少。无论工作还是生活的困难，我们只要坚持，终将会被克服解决。今天的内容就学到这里，我们下篇继续学习 Reactor 的操作符。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>{:target="_blank"} 下 02-reactor-core-learning
 模块下 ReactorTransformOperator02Test 测试类。