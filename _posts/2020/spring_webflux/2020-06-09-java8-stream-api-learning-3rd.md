---
layout: post
title: Spring WebFlux 学习笔记 - (一) 前传：学习Java 8 Stream Api (3) - Stream的终端操作
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Stream, Stream Api
keywords: Spring Web Flux, Stream, Reactor
---

# Stream API

Java8中有两大最为重要的改变：第一个是 Lambda 表达式；另外一个则是 Stream API(java.util.stream.*)。

Stream 是 Java8 中处理集合的关键抽象概念，它可以指定你希望对集合进行的操作，可以执行非常复杂的查找、过滤和映射数据等操作。使用Stream API 对集合数据进行操作，就类似于使用 SQL 执行的数据库查询。也可以使用 Stream API 来并行执行操作。简而言之，Stream API 提供了一种高效且易于使用的处理数据的方式。

> 流(Stream)是数据渠道，用于操作数据源（集合、数组等）所生成的元素序列。"集合讲的是数据，流讲的是计算！ "

集合和流(Stream)，表面上有一些相似之处，他们有不同的目标。集合主要关注其元素的有效管理和访问，相比之下，流不提供直接访问或操纵元素的手段，而是关心声明性地描述其源和将在该源上进行聚合的计算操作。 

上篇内容我们学习了Stream的中间操作，接下来我们来看下Stream数据流的结果消费，即终端(终止)操作。以下用 **终端操作** 统称。

## Stream的终端操作

流管道通过源生成，经过零个或多个中间操作后，进行最后的终端操作，由此产生结果或副作用，如count()或forEach(Consumer)。

我们将终端操作的结果分为如下几类：
1. 匹配
2. 统计
3. 消费
4. 转换

`以下内容提到XxxStream代表IntStream、LongStream、DoubleStream。如无特殊说明Stream也包含IntStream、LongStream、DoubleStream。`

### 匹配

匹配类型的终端操作返回值为布尔值，根据使用语境调用不同的方法及传入谓语实现确定是否匹配。

序号 | 支持的类 | 方法定义 | 方法说明
---|---|---|---
1 | Stream<T> | boolean anyMatch(Predicate<? super T> predicate); | 部分匹配，返回此流的任何元素是否与提供的谓词匹配。
2 | Stream<T> | boolean allMatch(Predicate<? super T> predicate); | 全部匹配，返回此流的所有元素是否与提供的谓词匹配。
3 | Stream<T> | boolean noneMatch(Predicate<? super T> predicate); | 全不匹配，返回此流中是否没有元素与提供的谓词匹配。

`注意：如果流为空时，allMatch方法始终返回true；noneMatch方法始终返回true。`

> 以下代码见 StreamTerminalOperationMatchTest。

#### anyMatch的使用

```java
// 是否存在匹配的元素，true
log.info("[1, 2, 3, 4, 5, 6]存在偶数否：{}",
        Stream.of(1, 2, 3, 4, 5, 6).anyMatch(n -> n % 2 == 0));
```

#### allMatch的使用

```java
// 全部元素是否都匹配，false
log.info("[1, 2, 3, 4, 5, 6]全部都是偶数否：{}",
        Stream.of(1, 2, 3, 4, 5, 6).allMatch(n -> n % 2 == 0));
```

#### noneMatch的使用

```java
// 全部元素是否都不匹配，false
log.info("[1, 2, 3, 4, 5, 6]全部都不是偶数否：{}",
        Stream.of(1, 2, 3, 4, 5, 6).noneMatch(n -> n % 2 == 0));
```

#### 空流验证

```java
// 空流中不存在任何匹配元素，所以返回false
log.info("空流是否AnyMatch：{}", Stream.empty().anyMatch(Objects::isNull));
// 空流中不存在不匹配的，即全部匹配，所以返回true
log.info("空流是否AllMatch：{}", Stream.empty().allMatch(Objects::isNull));
// 空流中全部都不匹配，所以返回true
log.info("空流是否NoneMatch：{}", Stream.empty().noneMatch(Objects::isNull));
```

### 统计

统计类型的终端操作是对流元素的统计，如元素个数、最大值、最小值、统计对象等。

序号 | 支持的类 | 方法定义 | 方法说明
---|---|---|---
1 | Stream<T> | long count(); | 返回此流中的元素数。
2 | Stream<T> | Optional<T> min(Comparator<? super T> comparator); | 根据提供的 Comparator返回此流的最小元素。
3 | Stream<T> | Optional<T> max(Comparator<? super T> comparator); | 根据提供的 Comparator返回此流的最大元素。
4 | Stream<T> | OptionalXxx min(); | 返回 OptionalInt此流的最小元素的OptionalInt，如果此流为空，则返回一个空的可选项。 
5 | XxxStream<T> | OptionalXxx max(); | 返回 OptionalInt此流的最大元素的OptionalInt，如果此流为空，则返回一个空的可选项。
6 | XxxStream<T> | OptionalDouble average(); | 返回 OptionalDouble此流的元素的算术平均值的OptionalDouble，如果此流为空，则返回空的可选项。 
7 | XxxStream<T> | Xxx sum(); | 返回此流中元素的总和。 
8 | XxxStream<T> | XxxSummaryStatistics summaryStatistics(); | 返回一个 IntSummaryStatistics描述有关此流的元素的各种摘要数据。   

XxxSummaryStatistics类型的统计对象，如IntSummaryStatistics，除了提供最小值、最大值、平均值、元素个数、总和外，还提供了accept、combine两个方法，分别支持添加新的数据和连接另外的统计对象，并自动重新统计结果。

> 以下代码见 StreamTerminalOperationStatisticsTest。

#### count的使用

```java
log.info("[1, 2, 3, 4, 5, 6]元素个数：{}",
        Stream.of(1, 2, 3, 4, 5, 6).count());
```

#### min的使用(使用Comparator比较)

```java
log.info("[1, 2, 3, 4, 5, 6]的最大值：{}",
        Stream.of(1, 2, 3, 4, 5, 6).min(Comparator.comparingInt(n -> n)).get());
```

#### max的使用(使用Comparator比较)

```java
log.info("[1, 2, 3, 4, 5, 6]的最小值：{}",
        Stream.of(1, 2, 3, 4, 5, 6).max(Comparator.comparingInt(n -> n)).get());
```

#### min的使用

```java
log.info("[1, 2, 3, 4, 5, 6]的最小值：{}",
        IntStream.of(1, 2, 3, 4, 5, 6).min().getAsInt());
```

#### max的使用

```java
log.info("[1, 2, 3, 4, 5, 6]的最大值：{}",
        IntStream.of(1, 2, 3, 4, 5, 6).max().getAsInt());
```

#### average的使用

```java
log.info("[1, 2, 3, 4, 5, 6]的平均值：{}",
        IntStream.of(1, 2, 3, 4, 5, 6).average().getAsDouble());
```

#### sum的使用

```java
log.info("[1, 2, 3, 4, 5, 6]的求和：{}",
        IntStream.of(1, 2, 3, 4, 5, 6).sum());
```

#### summaryStatistics的使用

```java
IntSummaryStatistics summaryStatistics = IntStream.of(1, 2, 3, 4, 5, 6).summaryStatistics();
log.info("[1, 2, 3, 4, 5, 6]的统计对象：{}", summaryStatistics);
summaryStatistics.accept(7);
log.info("添加7后，统计对象变为：{}", summaryStatistics);
IntSummaryStatistics summaryStatistics2 = IntStream.of(8, 9).summaryStatistics();
summaryStatistics.combine(summaryStatistics2);
log.info("合并[8, 9]后，统计对象变为：{}", summaryStatistics);
```

### 消费

消费类型的终端操作是对流内元素的获取或循环消费。

序号 | 支持的类 | 方法定义 | 方法说明
---|---|---|---
1 | Stream<T> | Optional<T> findFirst(); | 返回描述此流的第一个元素的Optional，如果流为空，则返回一个空的Optional。
2 | Stream<T> | Optional<T> findAny(); | 返回描述流的一些元素的Optional如果流为空，则返回一个空的Optional。
3 | Stream<T> | void forEach(Consumer<? super T> action); | 对此流的每个元素执行操作。
4 | Stream<T> | void forEachOrdered(Consumer<? super T> action); | 如果流具有定义的顺序，则以流的顺序对该流的每个元素执行操作。

看了forEach和forEachOrdered的Api的说明，大家可能对这两个还是有点疑问，这里特别说明下，forEach在并行流(parallel后面会讲)中并不按照流内元素之前定义的顺序执行操作，是无序的，而forEachOrdered会按照流之前定义的顺序执行操作。除非必要，在并行流中不建议使用forEachOrdered对其进行排序执行操作，否则影响性能。

> 流有可能也可能没有定义顺序。流是否有顺序取决于源和中间操作。某些流源（如List或数组）本质上是有序的，而其他数据源（如HashSet）不是。一些中间操作（例如sorted()）可以在其他无序流上排序，而其他中间操作可以使有序流无序，例如BaseStream.unordered()。此外，一些终端操作可能会忽略顺序，如forEach()。 

> 如果一个流被命令，大多数操作被限制为在遇到的顺序中对元素进行操作; 如果流的源是List含有[1, 2, 3] ，然后执行的结果map(x -> x*2)必须是[2, 4, 6] 。 然而，如果源没有定义的顺序，则任何[2, 4, 6]排列组合值都将是有效结果。 
  
> 对于顺序流，遇到顺序的存在或不存在不影响性能，仅影响确定性。 如果流被排序，在相同的源上重复执行相同的流管线将产生相同的结果; 如果没有排序，重复执行可能会产生不同的结果。 
  
> 对于并行流，放宽排序约束有时可以实现更有效的执行。如果元素的排序不相关，某些聚合操作，例如过滤重复（distinct()）或组合减少（Collectors.groupingBy()）可以更有效地实现。 类似地，本质上与遇到顺序相关的操作，如limit()可能需要缓冲以确保正确排序，从而破坏并行性的好处。另外，当流遇到排序，但用户并不特别在意那次偶遇秩序的情况下，明确地去操作为无序流unordered()可以提高某些状态或终端操作的并行性能。 然而，大多数流管线，例如上面的例子的“权重之和”，仍然在有序的限制下有效地并行化。 

> 以下代码见 StreamTerminalOperationConsumeTest。

#### findFirst的使用

```java
log.info("[1, 2, 3, 4, 5, 6]的首个值：{}",
        Stream.of(1, 2, 3, 4, 5, 6).parallel().findFirst().get());
```

#### findAny的使用

```java
log.info("[1, 2, 3, 4, 5, 6]的任意值：{}",
        Stream.of(1, 2, 3, 4, 5, 6).parallel().findAny().get());
```

#### forEach的使用

```java
// 并行后顺序随机，输出不保证顺序
log.info("[1, 2, 3, 4, 5, 6]的并行后循环输出：");
Stream.of(1, 2, 3, 4, 5, 6).parallel().forEach(System.out::println);
```

#### forEachOrdered的使用

```java
// 无论是否并行，始终按照流定义的顺序或排序后的结果输出
log.info("[1, 2, 5, 6, 3, 4]的并行后顺序循环输出：");
Stream.of(1, 2, 5, 6, 3, 4).sorted().parallel().forEachOrdered(System.out::println);
```

### 转换

转换类型的终端操作是将流转换为另一种对象使用。

序号 | 支持的类 | 方法定义 | 方法说明
---|---|---|---
1 | Stream<T> | Optional<T> reduce(BinaryOperator<T> accumulator); | 使用associative累积函数对此流的元素执行reduction，并返回描述减小值（如果有的话）的Optional 。
2 | Stream<T> | T reduce(T identity, BinaryOperator<T> accumulator); | 使用提供的身份值和 associative累积功能对此流的元素执行 reduction ，并返回减小的值。 
3 | Stream<T> | U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator combiner); | 执行 reduction在此流中的元素，使用所提供的身份，积累和组合功能。 
4 | Stream<T> | <R, A> R collect(Collector<? super T, A, R> collector); | 使用 Collector对此流的元素执行 mutable reduction Collector。
5 | Stream<T> | <R> R collect(Supplier<R> supplier, BiConsumer<R, ? super T> accumulator, BiConsumer<R, R> combiner); | 对此流的元素执行 mutable reduction操作。

此处我们着重说下序号3，带有3个参数的reduce方法，该方法支持转换元素（结果）类型，即从类型T转换为类型U。第1个参数代表初始值；第2个参数是累加器函数式接口，输入类型U和类型T，返回类型U；第3个参数是组合器函数式接口，输入类型U和类型U，返回类型U。该方法的第3个参数在并行执行下有效。
同时需要注意，此方法有如下要求：
- u = combiner(identity, u);
- combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t);

该方法的代码示例见**reduce的使用3**和**reduce的使用4**，如果对该示例不了解，可以在后面的章节中讲解了并行执行之后再回过头来看该示例。

> 以下代码见 StreamTerminalOperationTransformTest。

#### reduce的使用1

```java
// 使用reduce方式实现查找最小值
log.info("[1, 2, 3, 4, 5, 6]的最小值：{}",
        Stream.of(1, 2, 3, 4, 5, 6).reduce(Integer::min).get());
```

#### reduce的使用2

```java
// 使用reduce方式实现求和
log.info("[1, 2, 3, 4, 5, 6]的求和：{}",
        Stream.of(1, 2, 3, 4, 5, 6).reduce(0, Integer::sum));
```

#### reduce的使用3

```java
// 求单词长度之和
Integer lengthSum = Stream.of("I", "love", "you", "too")
        .parallel()
        .reduce(0,// 初始值　// (1)
                (sum, str) -> sum + str.length(), // 累加器 // (2)
                Integer::sum);// 部分和拼接器，并行执行时才会用到 // (3)
// int lengthSum = stream.mapToInt(str -> str.length()).sum();
log.info("ILoveYouToo的长度为：{}", lengthSum);
```

#### reduce的使用4

```java
// 下方方法同步执行时，能出现正确结果
// 并行执行时，将出现意想不到的结果
// 多线程执行时，append导致初始值identity发生了变化，而多线程又导致了数据重复添加
StringBuffer word = Stream.of("I", "love", "you", "too")
        .parallel()                 // 同步执行注释该步骤
        .reduce(new StringBuffer(),// 初始值　// (1)
                StringBuffer::append, // 累加器 // (2)
                StringBuffer::append);// 部分和组合器，并行执行时才会用到 // (3)
log.info("拼接字符串为:{}", word);

// 此处如果使用字符串concat，导致性能降低，不停创建字符串常量
String word2 = Stream.of("I", "love", "you", "too")
        .parallel()                 // 同步执行注释该步骤
        .reduce("",// 初始值　// (1)
                String::concat, // 累加器 // (2)
                String::concat);// 部分和组合器，并行执行时才会用到 // (3)
log.info("拼接字符串为:{}", word2);

// 下面方法并行执行时，虽然能达到正确的结果，但是并未满足reduce的要求
List<Integer> accResult = Stream.of(1, 2, 3, 4)
        .parallel()
        .reduce(Collections.synchronizedList(new ArrayList<>()),
                (acc, item) -> {
                    List<Integer> list = new ArrayList<>();
                    list.add(item);
                    System.out.println("item BiFunction : " + item);
                    System.out.println("acc+ BiFunction: " + list);
                    return list;
                }, (accs, items) -> {
                    accs.addAll(items);
                    System.out.println("item BinaryOperator: " + items);
                    System.out.println("acc+ BinaryOperator: " + accs);
                    return accs;
                });
log.info("accResult: {}", accResult);
```

由于时间及版面的缘故，本期就先讲到这里，下期在着重将collect。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>

以上是本期笔记的内容，我们下期见。
