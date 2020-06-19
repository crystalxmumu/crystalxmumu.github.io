---
layout: post
title: Spring WebFlux 学习笔记 - (二) 开篇：学习响应式编程 Reactor (3) - reactor 基础
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Reactor 学习系列, 响应式编程
keywords: Spring Web Flux, Reactive Streams, Reactor
---

# Reactor

Reactor 项目的主要 artifact 是 reactor-core，这是一个基于 Java 8 的实现了响应式流规范的响应式库。

Reactor 提供了实现 Publisher 的响应式类 Flux 和 Mono，以及丰富的操作符。一个 Flux 代表 0...N 个元素的响应式流；一个 Mono 代表 0|1 个元素的响应式流。

Flux 和 Mono 之间可以转换，比如 Flux 的 count 操作（计算流中元素个数）返回 Mono，Mono 的 concatWith 操作（连接另一个响应式流）返回 Flux。

## Flux

Flux<T> 是一个能够发出 0 到 N 个元素的标准 Publisher<T>，它会被一个完成（completion）或错误（error）信号终止。因此，一个 Flux 的可能结果是 value、completion
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

Mono<T> 是一种特殊的Publisher<T>，它最多只能发出一个元素，然后（可选的）终止于 onComplete 或 onError 信号。

Mono 中的操作符是 Flux 中操作符的子集，即 Flux 中只有部分操作符适用于 Mono，有些操作符是将 Mono 和另一个 Publisher 连接转换为 Flux。例如，Mono#concatWith(Publisher
) 转换为 Flux，Mono#then(Mono) 返回另一个 Mono。

> 注意：可以使用 Mono<Void> 来创建一个只有完成概念的空值异步处理过程（类似于 Runnable）。

下图展示的是 Mono 基于时间线的弹珠交互图：

<img src="/images/posts/spring_web_flux/05_reactor_mono_transform.png" alt="通过操作符转换 Mono 中元素" />

# 补充

log 操作符。

# 总结

本篇我们了解了如何引入 Reactor ；初步体验了 Reactor 的 Hello World 代码；最后我们了解了如何测试及调试 Reactor，这些内容为我们后面学习 Reactor 的基础，希望大家都能掌握。

今天的内容就学到这里，我们下篇开始 Reactor 的基础和特性学习。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>{:target="_blank"} 下 02-reactor-core-learning 模块。

# 参考
1. [Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference/){:target="_blank"}
2. [Reactor 3 中文指南](https://github.com/get-set/reactor-core/blob/master-zh/src/docs/index.html){:target="_blank"}