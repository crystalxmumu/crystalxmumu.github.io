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
1 | 转换 | as, cast, collect, collectList, collectMap, collectMultimap, collectSortedList, concatMap, concatMapDelayError, concatMapIterable, elapsed, expand, expandDeep, flatMap, flatMapDelayError, flatMapIterable, flatMapSequential, flatMapSequentialDelayError, groupJoin, handle, index, join, map, switchMap, switchOnFirst, then, thenEmpty, thenMany, timestamp, transform, transformDeferred
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

接下来我们来挨个学习各类的操作符，如同前面学习响应式流创建一样，讲解操作符时，如果是 Flux 或 Mono 独有的，会在方法名前增加类名前缀。

## 转换类操作符

转换类的操作符数量最多，平常过程中也是使用最频繁的。

### as

将响应式流转换为目标类型，既可以是非响应式对象，也可以是 Flux 或 Mono。

```java
Flux.range(3, 8)
    .as(Mono::from)
    .subscribe(System.out::println);
```

### cast

将响应式流内的元素强转为目标类型，如果类型不匹配（非父类类型或当前类型），将抛出 ClassCastException ，见图知意：

<img src="/images/posts/spring_web_flux/reactor/01_operator_flux_cast.png" alt="cast 操作符" />

```java
Flux.range(1, 3)
    .cast(Number.class)
    .subscribe(System.out::println);
```

### Flux#collect

通过应用收集器，将 Flux 发出的所有元素收集到一个容器中。当此流完成时，发出收集的结果。 Flux 提供了 2 个重载方法，主要区别在于应用的收集器不同，一个是 Java Stream 的 Collector， 另一个是自定义收集方法（同 Java Stream 中 collect 方法）：

```java
<R,A> Mono<R> collect(Collector<? super T,A,? extends R> collector);
<E> Mono<E> collect(Supplier<E> containerSupplier,
                     BiConsumer<E,? super T> collector);
```

见图知意：

<img src="/images/posts/spring_web_flux/reactor/02_operator_flux_collect.png" alt="collect 操作符" />

```java
Flux.range(1, 5)
    .collect(Collectors.toList())
    .subscribe(System.out::println);
```

### Flux#collectList

当此 Flux 完成时，将此流发出的所有元素收集到一个列表中，该列表由生成的 Mono 发出。见图知意：

<img src="/images/posts/spring_web_flux/reactor/03_operator_flux_collectList.png" alt="collectList 操作符" />

```java
Flux.range(1, 5)
    .collectList()
    .subscribe(System.out::println);
```

### Flux#collectMap

将 Flux 发出的所有元素按照键生成器和值生成器收集到 Map 中，之后由 Mono 发出。Flux 提供了 3 个重载方法：

```java
<K> Mono<Map<K,T>> collectMap(Function<? super T,? extends K> keyExtractor);
<K,V> Mono<Map<K,V>> collectMap(Function<? super T,? extends K> keyExtractor,
                                 Function<? super T,? extends V> valueExtractor);
<K,V> Mono<Map<K,V>> collectMap(Function<? super T,? extends K> keyExtractor,
                                 Function<? super T,? extends V> valueExtractor,
                                 Supplier<Map<K,V>> mapSupplier);
```

它们的主要区别在于是否提供值生成器和初始的Map，意同 Java Stream 中的 Collectors#toMap 。见图知意：

<img src="/images/posts/spring_web_flux/reactor/04_operator_flux_collectMap.png" alt="collectMap 操作符" />

```java
Flux.just(1, 2, 3, 4, 5, 3, 1)
    .collectMap(n -> n, n -> n + 100)
    .subscribe(System.out::println);
```

### Flux#collectMultimap

collectMultimap 与 collectMap 的区别在于，map 中的 value 类型不同，一个是集合，一个是元素。 collectMultimap 对于流中出现重复的 key 的 value，加入到了集合中，而 collectMap 做了替换。在这点上，reactor 不如 Java Stream 中的 Collectors#toMap 方法，没有提供 key 重复时的合并函数。也提供了 3 个重载方法。

```java
<K> Mono<Map<K,Collection<T>>> collectMultimap(Function<? super T,? extends K> keyExtractor);
<K,V> Mono<Map<K,Collection<V>>> collectMultimap(Function<? super T,? extends K> keyExtractor,
                                                  Function<? super T,? extends V> valueExtractor);
<K,V> Mono<Map<K,Collection<V>>> collectMultimap(Function<? super T,? extends K> keyExtractor,
                                                  Function<? super T,? extends V> valueExtractor,
                                                  Supplier<Map<K,Collection<V>>> mapSupplier)
```

见图知意：

<img src="/images/posts/spring_web_flux/reactor/05_operator_flux_collectMultimap.png" alt="collectMultimap 操作符" />

```java
Flux.just(1, 2, 3, 4, 5, 3, 1)
    .collectMultimap(n -> n, n -> n + 100)
    .subscribe(System.out::println);
```

### Flux#collectSortedList

将 Flux 发出的元素在完成时进行排序，之后由 Mono 发出。Flux 提供了 2 个重载方法：

```java
Mono<List<T>> collectSortedList();
Mono<List<T>> collectSortedList(@Nullable Comparator<? super T> comparator);
```

见图知意：

<img src="/images/posts/spring_web_flux/reactor/06_operator_flux_collectSortedList.png" alt="collectSortedList 操作符" />

```java
Flux.just(1, 3, 5, 3, 2, 5, 1, 4)
    .collectSortedList()
    .subscribe(System.out::println);
```


# 总结

本篇我们介绍了 Reactor 操作符的分类，之后介绍了部分转换类操作符，讲解示例时都是单个操作符，相信大家都能理解。

今天的内容就学到这里，我们下篇继续学习 Reactor 的操作符。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>{:target="_blank"} 下 02-reactor-core-learning
 模块下 ReactorTransformOperatorTest 测试类。

# 参考
1. [Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference/){:target="_blank"}
2. [Reactor 3 中文指南](https://github.com/get-set/reactor-core/blob/master-zh/src/docs/index.html){:target="_blank"}