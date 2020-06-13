---
layout: post
title: Spring WebFlux 学习笔记 - (一) 前传：学习Java 8 Stream Api (4) - Stream 终端操作之 collect
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Stream, Stream Api
keywords: Spring Web Flux, Stream, Reactor
---

# Stream API

Java8中有两大最为重要的改变：第一个是 Lambda 表达式；另外一个则是 Stream API(java.util.stream.*)。

Stream 是 Java8 中处理集合的关键抽象概念，它可以指定你希望对集合进行的操作，可以执行非常复杂的查找、过滤和映射数据等操作。使用Stream API 对集合数据进行操作，就类似于使用 SQL 执行的数据库查询。也可以使用 Stream API 来并行执行操作。简而言之，Stream API 提供了一种高效且易于使用的处理数据的方式。

> 流(Stream)是数据渠道，用于操作数据源（集合、数组等）所生成的元素序列。"集合讲的是数据，流讲的是计算！ "

集合和流(Stream)，表面上有一些相似之处，他们有不同的目标。集合主要关注其元素的有效管理和访问，相比之下，流不提供直接访问或操纵元素的手段，而是关心声明性地描述其源和将在该源上进行聚合的计算操作。 

上篇内容我们学习了Stream的大部分终端操作，我们这篇着重了解下Stream中重要的终端操作：collect。

## collect 方法

序号 | 支持的类 | 方法定义 | 方法说明
---|---|---|---
1 | Stream<T> | <R> R collect(Supplier<R> supplier, BiConsumer<R, ? super T> accumulator, BiConsumer<R, R> combiner); | 对此流的元素执行 mutable reduction操作。
2 | Stream<T> | <R, A> R collect(Collector<? super T, A, R> collector); | 使用 Collector对此流的元素执行 mutable reduction Collector。

> 以下代码见 StreamTerminalOperationTransformTest。

### 实现3参数转换接口

序号1的方法，传递了3个参数，参数1为创建新结果容器的函数；参数2为累加器函数，将参数1和流内元素执行累加操作；参数3为组合器函数，并行执行时会使用该函数。

同步执行时，该方法相当于执行：

```java
R result = supplier.get();
for (T element : this stream) {
  accumulator.accept(result, element);
}
return result;
```

我们编写如下代码，看下实际效果

```java
// 使用collect方法实现字符串连接
log.info("拼接字符串为：{}",
        Stream.of("I", "love", "you", "too")
                .collect(StringBuilder::new, (b1, b2) -> {
                    log.info("累加执行：{} + {}", b1, b2);
                    b1.append(b2);
                }, (b1, b2) -> {
                    log.info("组合执行：{} ++ {}", b1, b2);
                    b1.append(b2);
                })
                .toString());
```

以上代码将输出如下日志：
> [main] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 累加执行： + I <br> 
  [main] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 累加执行：I + love <br> 
  [main] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 累加执行：Ilove + you <br> 
  [main] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 累加执行：Iloveyou + too <br> 
  [main] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 拼接字符串为：Iloveyoutoo <br> 


并行执行时，该方法相当于执行：

```java
R result1 = supplier.get();
R result2 = supplier.get();
R result3 = supplier.get();
R result4 = supplier.get();

// 累加执行，此处为并发（多线程）执行，每行代表一个线程
accumulator.accept(result1, element1);
accumulator.accept(result2, element2);
accumulator.accept(result3, element3);
accumulator.accept(result4, element4);
// ...
// accumulator.accept(resultN, elementN);

// 开始组合，此处为并发（多线程）执行，每行代表一个线程
combiner.accept(result1, result2);
combiner.accept(result3, result4);
combiner.accept(result1, result3);
// combiner.accept(result1, resultN);

return result1;
```

将上述的代码改为.parallel()方式调用，将输出如下日志：

> [main] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 累加执行： + you <br> 
  [ForkJoinPool.commonPool-worker-3] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 累加执行： + I <br> 
  [ForkJoinPool.commonPool-worker-2] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 累加执行： + too <br> 
  [ForkJoinPool.commonPool-worker-2] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 组合执行：you ++ too <br> 
  [ForkJoinPool.commonPool-worker-1] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 累加执行： + love <br> 
  [ForkJoinPool.commonPool-worker-1] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 组合执行：I ++ love <br> 
  [ForkJoinPool.commonPool-worker-1] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 组合执行：Ilove ++ youtoo <br> 
  [main] INFO top.todev.note.web.flux.stream.StreamTerminalOperationTransformTest - 拼接字符串为：Iloveyoutoo

`注意：上述日志中出现的ForkJoinPool.commonPool-worker-N为并发（多线程）执行时的线程名。`



### 实现Collector接口

实现Collector需要实现如下4个接口：

```java
// 一个创建并返回一个新的可变结果容器的函数。
Supplier<A> supplier();
// 将值折叠成可变结果容器的函数。
BiConsumer<A, T> accumulator();
// 一个接受两个部分结果并将其合并的函数。 
BinaryOperator<A> combiner();
// 执行从中间累积类型 A到最终结果类型 R的最终 R 。 
Function<A, R> finisher();
// 返回一个 Collector.Characteristics 类型的Set, 表示该收集容器的特征。
Set<Characteristics> characteristics();
```

collect方法执行时，他们的调用流程如下：
1. 创建新的结果容器（supplier()）
2. 将新的数据元素并入结果容器（accumulator()）
3. 将两个结果容器组合成一个（combiner()）
4. 在容器上执行可选的最终变换（finisher()）

简单来讲，生成容器A，通过accumulator针对A及流元素T执行累加，（如果并行存在的话）对多个A执行组合combiner，最终执行finisher后由A转换为R。对于使用者来说，A为中间变量，无关其实现细节。

我们实现一个计算整数流的平均数的Collector，代码如下：

```java
// 使用collector实现求ping均值
log.info("[1, 2, 3, 4, 5, 6]的平均值：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
                .parallel()
                .collect(new Collector<Integer, long[], Double>() {
                             @Override
                             public Supplier<long[]> supplier() {
                                 return () -> new long[2];
                             }

                             @Override
                             public BiConsumer<long[], Integer> accumulator() {
                                 return (a, t) -> {
                                     log.info("{}累加{}", a, t);
                                     a[0] += t;
                                     a[1]++;
                                 };
                             }

                             @Override
                             public BinaryOperator<long[]> combiner() {
                                 return (a, b) -> {
                                     log.info("{}组合{}", a, b);
                                     a[0] += b[0];
                                     a[1] += b[1];
                                     return a;
                                 };
                             }

                             @Override
                             public Function<long[], Double> finisher() {
                                 return (a) -> a[1] == 0 ? 0 : new Long(a[0]).doubleValue() / a[1];
                             }

                             @Override
                             public Set<Characteristics> characteristics() {
                                 Set<Characteristics> set = new HashSet<>();
                                 set.add(Characteristics.CONCURRENT);
                                 return set;
                             }
                         }
                )
);
```

### 常用Collector

通过上面的示例，我们实现了一个自定义的Collector，我们发现实现一个自定义的Collector还是比较麻烦的，需要实现5个接口。
Java 开发者们已经想到了这个问题，他们额外提供了一个 **of** 方法，可以通过lambda的方式创建 collector，类似 collect 中传递几个参数：提供者、累加器、组合器、完成器以及特征配置，此处我们就不细讲了。
Java 开发者们更为贴心的为我们创建了一些常用的 Collector ，让我们可以直接使用。这些常用的 Collector 实现放在 Collectors 类下，我们来了解下。

#### 统计平均值 averagingXxx 的使用

Collectors 提供了 averagingDouble、averagingLong、averagingInt 3种统计平均值的 Collector 实现类，以下代码以 averagingInt 为例，由于使用方式相似，我们就不举例了。

```java
// 使用collector实现求均值
log.info("[1, 2, 3, 4, 5, 6]的平均值：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
        .collect(Collectors.averagingInt(n -> n))
);
```

#### 统计元素个数 counting 的使用

该方法和 Stream 中的 count 方法一样。 

```java
// 使用collector获取元素数量
log.info("[1, 2, 3, 4, 5, 6]的个数：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
                .collect(Collectors.counting())
);
```

#### 统计总和 summingXxx 的使用

Collectors 提供了 summingDouble、summingLong、summingInt 3种统计求和值的 Collecto r实现类，同时还提供了 summarizingDouble 、 summarizingLong 、summarizingInt 3种统计对象的 Collector 实现类，以下代码以 summingInt 为例，由于使用方式相似，我们就不举例了。

```java
// 使用collector获取总和
log.info("[1, 2, 3, 4, 5, 6]的总和：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
                .collect(Collectors.summingInt(n -> n))
);
```

#### 统计最小元素 minBy 的使用

```java
// 使用collector获取最小元素
log.info("[1, 2, 3, 4, 5, 6]的最小值：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
                .collect(Collectors.minBy(Integer::min))
                .get()
);
```

#### 统计最大元素 maxBy 的使用

```java
// 使用collector获取最da元素
log.info("[1, 2, 3, 4, 5, 6]的最大值：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
                .collect(Collectors.maxBy(Integer::max))
                .get()
);
```

#### 统计累加处理 reducing 的使用

reducing 和 Stream 中的 reduce 操作方法类似，我们就不详述了。

```java
// 使用collector实现求均值
log.info("[1, 2, 3, 4, 5, 6]的求和：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
                .collect(Collectors.reducing(0, Integer::sum))
);
```

#### 转换映射 mapping 的使用

mapping 支持将 第一个参数的结果再次执行转换，即向下游传递。

```java
log.info("[1, 2, 3, 4, 5, 6]每个增加20后的平均值：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
                .collect(Collectors.mapping(n -> n + 20, Collectors.averagingInt(n -> n)))
);
```

#### 转换连接 joining 的使用

joining 提供了 3 种重载方法，支持传递 分隔符、前缀、后缀等。

```java
// 使用collector连接字符串
log.info("连接字符串为：{}",
        Stream.of("I", "love", "you", "too")
                .collect(Collectors.joining(" ", "Java, ", "!"))
);
```

#### 转换为集合 toList 的使用

```java
log.info("[1, 2, 3, 4, 5, 6, 5, 3, 6]转换为集合：{}",
        Stream.of(1, 2, 3, 4, 5, 6, 5, 3, 6)
                .collect(Collectors.toList())
);
```

#### 转换为Map toMap 的使用

toMap 提供了 3 种重载方法，除了指定 Key 和 Value 的生成器外，区别在于对于 Key 重复时， Value的处理方式；以及初始Map的生成器。

```java
log.info("[1, 2, 3, 4, 5, 6, 5, 3, 6]转换为Map：{}",
        Stream.of(1, 2, 3, 4, 5, 6, 5, 3, 6)
                .collect(Collectors.toMap(Object::toString, n -> n, Integer::sum))
);
```

#### 转换为Set toSet 的使用

```java
log.info("[1, 2, 3, 4, 5, 6, 5, 3, 6]的转换为Set：{}",
        Stream.of(1, 2, 3, 4, 5, 6, 5, 3, 6)
                .collect(Collectors.toSet())
);
```

#### 转换为分组 groupingBy 的使用

分组函数将流中元素按某种定义分组，也提供了 2 种重载方法，支持递归向下游分组。

```java
log.info("[1, 2, 3, 4, 5, 6, 5, 3, 6]的分组数据：{}",
        Stream.of(1, 2, 3, 4, 5, 6, 5, 3, 6)
                .collect(Collectors.groupingBy(n -> n))
);
```

#### 转换为分区 partitioningBy 的使用

分区函数将流中元素按条件分为2组，也提供了 2 种重载方法，支持递归向下游分组。

```java
log.info("[1, 2, 3, 4, 5, 6]的奇偶分区数据：{}",
        Stream.of(1, 2, 3, 4, 5, 6)
                .collect(Collectors.partitioningBy(n -> n %2 == 0))
);
```

#### 其他方法

Collectors 中还提供了 groupingByConcurrent 、 toCollection 、 toConcurrentMap 等几种支持并发的 Collector 实现，用法基本和非并发的相同，我们就不详述了。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>

以上是本期笔记的内容，我们下期见。
