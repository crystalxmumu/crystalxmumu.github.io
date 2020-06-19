---
layout: post
title: Spring WebFlux 学习笔记 - (二) 开篇：学习响应式编程 Reactor (3) - reactor 基础
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Reactor 学习系列, 响应式编程
keywords: Spring Web Flux, Reactive Streams, Reactor
---

# Reactor

Reactor 项目的主要 artifact 是 reactor-core，这是一个基于 Java 8 的实现了响应式流规范的响应式库。

Reactor 提供了实现 Publisher 的响应式类 Flux 和 Mono，以及丰富的操作符。一个 Flux 代表 0...N 个元素的响应式流；一个 Mono 代表 0\|1 个元素的响应式流。

Flux 和 Mono 之间可以转换，比如 Flux 的 count 操作（计算流中元素个数）返回 Mono，Mono 的 concatWith 操作（连接另一个响应式流）返回 Flux。

## Flux

Flux\<T\> 是一个能够发出 0 到 N 个元素的标准 Publisher\<T\>，它会被一个完成（completion）或错误（error）信号终止。因此，一个 Flux 的可能结果是 value、completion
 或 value，这三个分别会传递给订阅者中的 onNext、onComplete、onError 方法。
 
> 注意：所有的信号事件，包括代表终止的信号事件都是可选的。如果没有 onNext 事件，但是有 onComplete 事件，那么发出的就是空的有限流；如果去掉 onComplete
> 就得到一个 无限的空数据流。无限的数据流可以不是空的，比如 Flux.interval(Duration) 生成的是一个 Flux<Long>，这是一个无限周期性发出规律整数的时钟数据流。

下图展示的是 Flux 基于时间线的弹珠交互图，通过操作符转换 Flux 中元素：

<img src="/images/posts/spring_web_flux/04_reactor_flux_transform.png" alt="通过操作符转换 Flux 中元素" />

- 上面那条线表示的是 Flux 数据流时间线，时间从左至右
- 上面那条线中的弹珠代表示的是 Flux 发出的 数据元素
- 上面那条线最后的垂直线表示的是 Flux 已经完成成功事件 
- 中间的箭头虚线和框表示的是 Flux 中的元素正在被转换，框内的文字表示的是转换的方式（包含操作符）
- 下面那条线表示的是 FLux 经过转换后的新数据流
- 如果由于某种原因导致 Flux 的转换终止，将使用 X 来代替 垂直线

> 后续在学习操作符的过程中，我们将见到很多类似的弹珠图，请大家详细了解清楚该图各部分的含义。

## Mono

Mono\<T\> 是一种特殊的Publisher\<T\>，它最多只能发出一个元素，然后（可选的）终止于 onComplete 或 onError 信号。

Mono 中的操作符是 Flux 中操作符的子集，即 Flux 中只有部分操作符适用于 Mono，有些操作符是将 Mono 和另一个 Publisher 连接转换为 Flux。例如，Mono#concatWith(Publisher
) 转换为 Flux，Mono#then(Mono) 返回另一个 Mono。

> 注意：可以使用 Mono<Void> 来创建一个只有完成概念的空值异步处理过程（类似于 Runnable）。

下图展示的是 Mono 基于时间线的弹珠交互图：

<img src="/images/posts/spring_web_flux/05_reactor_mono_transform.png" alt="通过操作符转换 Mono 中元素" />

## 创建 Flux 和 Mono

如同创建 Java Stream 一样，Reactor 也为我们提供了 多个工厂方法用来创建 Flux 和 Mono，有了 Stream 的基础，创建的基本方法我们来快速过一下。

> 下面的创建方法，如果是 Flux 或 Mono 独有的，会在方法名前增加类名前缀。

> 下面的示例代码中都有用到 subscribe 方法，下面会讲到，大家先了解它是响应式流的订阅方法，用于触发流，类似于 Java Stream 中的终端操作。

### just

使用提供的元素发出数据然后结束的流。

<img src="/images/posts/spring_web_flux/06_reactor_flux_just.png" alt="通过 just 创建" />

```java
Mono.just("hello, world").subscribe(System.out::println);
Mono.justOrEmpty(str).subscribe(System.out::println);
Mono.justOrEmpty(optional).subscribe(System.out::println);

Flux.just("hello", "world").subscribe(System.out::println);
Flux.just("hello").subscribe(System.out::println);
```

### Flux#fromXxx

Flux 提供了 fromArray（从数组）、fromIterable（从迭代器）、fromStream（从 Java Stream 流） 的方式来创建 Flux。

```java
String[] array = new String[]{"hello", "reactor", "flux"};
List<String> iterable = Arrays.asList("foo", "bar", "foobar");

Flux.fromArray(array).subscribe(System.out::println);
Flux.fromIterable(iterable).subscribe(System.out::println);
Flux.fromStream(Arrays.stream(array)).subscribe(System.out::println);
```

### Flux#range

从 start 开始构建一个 Flux，该 Flux 仅发出一系列递增计数的整数。 也就是说，在 start（包括）和 start + count（排除）之间发出整数，然后完成。见图识意：

<img src="/images/posts/spring_web_flux/07_reactor_flux_range.png" alt="通过 range 创建" />

```java
Flux.range(3, 5).subscribe(System.out::println);
```

### Flux#interval

在全局计时器上创建一个 Flux，该 Flux 在初始延迟后，发出从0开始并以指定的时间间隔递增的长整数。 如果未及时产生，则会通过溢出 IllegalStateException 发出 onError 
信号，详细说明无法发出的原因。 在正常情况下，Flux 将永远不会完成。interval 提供了 3 个重载方法，三者的区别主要在于是否延迟发出、以及使用的调度器。

> interval 生成的是一个无限数据流。

```java
Flux<Long> interval(Duration period)
Flux<Long> interval(Duration delay, Duration period)
Flux<Long> interval(Duration delay, Duration period, Scheduler timer)
```

- 第 1 个方法，没有延迟，按照 period 的周期立即发出，默认使用 Schedulers.parallel() 调度器
- 第 2 个方法，以 delay 延迟，按照 period 的周期发出，默认使用 Schedulers.parallel() 调度器
- 第 3 个方法，以 delay 延迟，按照 period 的周期发出，使用指定的调度器

见图识意：

<img src="/images/posts/spring_web_flux/10_reactor_flux_interval.png" alt="通过 interval 创建" />

```java
Flux.interval(Duration.ofMillis(30), Duration.ofMillis(500)).subscribe(System.out::println);
```

### empty

生成一个空的有限流。见图识意：

<img src="/images/posts/spring_web_flux/08_reactor_mono_empty.png" alt="通过 empty 创建" />

```java
Flux.empty().subscribe(System.out::println, System.out::println, () -> System.out.println("结束"));
```

### never

生成一个空的无限流。见图识意：

<img src="/images/posts/spring_web_flux/09_reactor_flux_never.png" alt="通过 never 创建" />

```java
Flux.never().subscribe(System.out::println, System.out::println, () -> System.out.println("结束"));
```

### 其它

Flux 和 Mono 还提供了编程式的创建数据流的方法，诸如 create、generate、push、handle 等的方式，这些内容暂时不是我们的重点，这里我们不细展开，感兴趣的可看 Api 进行研究下。

<img src="/images/posts/spring_web_flux/11_reactor_flux_generate.png" alt="通过 generate 创建" />

## 订阅 Flux 和 Mono

在上面创建 Flux 和 Mono 笔记的示例代码中，我们已经提到了 subscribe 订阅，在 subscribe 订阅中，Flux 和 Mono 支持 Java 8 Lambda 表达式。下面我们来看看 Reactor
 为我们提供了哪些订阅方法。

```java
subscribe(); // ①

subscribe(Consumer<? super T> consumer);  // ②

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer); // ③

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer); // ④

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer,
          Consumer<? super Subscription> subscriptionConsumer); // ⑤

subscribe(Subscriber<? super T> actual); // ⑥
```

1. 序号① 订阅并触发响应式流。
2. 序号② 对每个生成的元素进行消费。
3. 序号③ 对正常元素进行消费，对错误进行响应处理。
4. 序号④ 对正常元素和错误均有响应，还定义了响应流正常完成后的回调。
5. 序号⑤ 对正常元素、错误信号和完成信号均有响应，同时也定义了 对该 subscribe 返回的 Subscription 的回调处理。
6. 序号⑥ 通过自定义实现 Subscriber 接口来订阅。

> 注意：序号⑤ 变量传递一个 Subscription 的引用，如果不再需要更多元素时，可以通过它来取消订阅。取消订阅时，源头会停止生成数据，并清理相关的资源。取消和清理的操作是在 Disposable 接口中定义的。

来看下序号 ⑤ 的 subscribe 的弹珠图：

<img src="/images/posts/spring_web_flux/11_reactor_flux_generate.png" alt="通过 generate 创建" />

```java
Flux.range(1, 4)
        .subscribe(System.out::println,
                error -> System.err.println("发生错误：" + error),
                () -> System.out.println("完成"),
                sub -> {
                    System.out.println("已订阅");
                    // 理解背压
                    // 尝试修改下 request 中的值，看看有啥变化
                    sub.request(10);
                });
```

> 注意：序号⑥ 的方式支持背压等操作，不在我们本次笔记的范畴，我们还是先略过，后期在学习。

# 补充

在上节我们讲解 Reactor 调试部分时，遗漏了记录数据流的日志方法，再此做下补充：除了基于 stack trace 的方式调试分析，我们还可以使用 log
 操作符，来跟踪响应式流并记录日志。将它添加到操作链上之后，它会读取每一个再其上游的 Flux 和 Mono 事件（包括 onNext、onError、onComplete、Subscribe、Cancel 和 Request）。

```java
// 尝试交换下 take 和 log 的顺序，看看有啥变化
Flux.range(1, 10)
        // .log()
        .take(3)
        .log()
        .subscribe();
```

# 总结

本篇我们介绍了 Reactor 的基础知识：先是了解了 Reactor 为我们提供的响应式流类 Flux 和 Mono，之后学习了如何创建他们和订阅他们，因为有之前 Stream
 的基础，想来大家对这些知识点都好理解和接受。

今天的内容就学到这里，我们下篇开始学习 Reactor 的操作符。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>{:target="_blank"} 下 02-reactor-core-learning
 模块下 ReactorBasicLearningTest 测试类。

# 参考
1. [Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference/){:target="_blank"}
2. [Reactor 3 中文指南](https://github.com/get-set/reactor-core/blob/master-zh/src/docs/index.html){:target="_blank"}