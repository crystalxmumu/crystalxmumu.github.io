---
layout: post
title: Spring WebFlux 学习笔记 - (二) 开篇：学习响应式编程 Reactor (4) - reactor 转换类操作符
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Reactor 学习系列, 响应式编程, reactor 操作符
keywords: Spring Web Flux, Reactive Streams, Reactor, reactor operators
---

# Reactor 操作符

数据在响应式流中的处理，就像流过一条装配流水线。Reactor 既是传送带，又是一个个的装配工或机器人。原材料从源头（最初的 Publisher ）流出，经过一个个的装配线中装配工或机器人的工位加工（operator 操作），最终被加工成成品，等待被推送到消费者（ subscribe 操作）。

在 Reactor 中，每个操作符对 Publisher 进行处理，然后将 Publisher 包装为另一个新的 Publisher 。就像一个链条，数据源自第一个 Publisher ，然后顺链条而下，在每个环节进行相应的处理。最终，订阅者（Subscriber ）终结这个过程。所以， 响应式编程按照链式方式进行开发。

> 注意，如同 Java Stream 的终端操作，订阅者（ Subscriber ）在没有订阅（ subscribe ）到一个发布者（ Publisher ）之前，什么也不会发生。

如同 Java Stream 的中间操作一样，Reactor 的 Flux 和 Mono 也为我们提供了多种操作符（远多于 Stream ），我们将它们分类如下：

序号 | 类型 | 操作符
---|---|---
1 | 转换 | as, collect, collectList, collectMap, collectMultimap, collectSortedList, concatMap, concatMapDelayError, concatMapIterable, elapsed, expand, expandDeep, flatMap, flatMapDelayError, flatMapIterable, flatMapSequential, flatMapSequentialDelayError, groupJoin, handle, index, join, map, switchMap, switchOnFirst, then, thenEmpty, thenMany, timestamp, transform, transformDeferred
2 | 筛选 | blockFirst, blockLast, distinct, distinctUntilChanged, elementAt, filter, filterWhen, ignoreElements, last, next, ofType, or,  repeat, retry, single, singleOrEmpty, sort, take, takeLast, takeUntil, takeUntilOther, takeWhile
3 | 组合 | concatWith, concatWithValues, mergeOrderWith, mergeWith, startWith, withLatestFrom, zipWith, zipWithIterable
4 | 条件 | defaultIfEmpty, delayUntil, retryWhen, switchIfEmpty
5 | 时间 | delayElements, delaySequence, delaySubscription, sample, sampleFirst, sampleTimeout, skip, skipLast, skipUntil, skipUntilOther, skipWhile, timeout
6 | 统计 | count, reduce, reduceWith, scan, scanWith
7 | 匹配 | all, any, hasElement, hasElements
8 | 分组 | buffer, bufferTimeout, bufferUntil, bufferUntilChanged, bufferWhen, groupBy, window, windowTimeout, windowUntil, windowUntilChanged, windowWhen, windowWhile
9 | 事件 | doAfterTerminate, doFinally, doFirst, doOnCancel, doOnComplete, doOnDiscard, doOnEach, doOnError, doOnNext, doOnRequest, doOnSubscribe, doOnTerminate, onBackpressureBuffer, onBackpressureDrop, onBackpressureError, onBackpressureLatest, onErrorContinue, onErrorMap, onErrorResume, onErrorReturn, onErrorStop
10 | 调试 | checkpoint, hide, log
11 | 其它 | cache, dematerialize, limitRate, limitRequest, materialize, metrics, name, onTerminateDetach, parallel, publish, publishNext, publishOn, replay, share, subscribeOn, subscriberContext, subscribeWith, tag

## 转换类操作符



# 总结

本篇我们介绍了 Reactor 的基础知识：先是了解了 Reactor 为我们提供的响应式流类 Flux 和 Mono，之后学习了如何创建他们和订阅他们，因为有之前 Stream
 的基础，想来大家对这些知识点都好理解和接受。

今天的内容就学到这里，我们下篇开始学习 Reactor 的操作符。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>{:target="_blank"} 下 02-reactor-core-learning
 模块下 ReactorBasicLearningTest 测试类。

# 参考
1. [Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference/){:target="_blank"}
2. [Reactor 3 中文指南](https://github.com/get-set/reactor-core/blob/master-zh/src/docs/index.html){:target="_blank"}