---
layout: post
title: Spring WebFlux 学习笔记 - (一) 前传：学习Java 8 Stream Api (1) - 创建 Stream
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Stream, Stream Api
keywords: Spring Web Flux, Stream, Reactor
---

# 影子
在学习Spring WebFlux之前，我们先来了解JDK的Stream，虽然他们之间没有直接的关系，有趣的是 Spring Web Flux 基于 Reactive Stream，他们中都带了 Stream。现有需求如下：筛选出一个数组中的偶数，每个增加 100 后输出到控制台，我们来看下使用JDK Stream和使用Reactor（Reactive Stream的一种实现，后面会讲）编写的代码：

```java
// JDK Stream实现
Arrays.stream(ARRAY)
        .filter(num -> num % 2 == 0)
        .map(num -> num + 100)
        .forEach(System.out::println);
```

```java
// Reactor实现
Flux.fromArray(ARRAY)
        .filter(num -> num % 2 == 0)
        .map(num -> num + 100)
        .subscribe(System.out::println);
```

> 以上代码见JdkStreamAndReactorTest。

我们发现他们的写法是相似的，都是采用[函数式编程](https://zh.wikipedia.org/wiki/%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BC%96%E7%A8%8B){:target="_blank"}，并且其中有很多函数(操作符)是一样的，抛除他们的异同点（后面会讲），我们先来了解下 Java8 的 Stream API，方便我们后面可以更快的了解 Reactor 中的各种操作符。

# Stream API
Java8中有两大最为重要的改变：第一个是 Lambda 表达式；另外一个则是 Stream API(java.util.stream.*)。

Stream Api 位于 java.util.stream 包下，Stream接口是该包下的关键抽象。Stream、IntStream、LongStream、DoubleStream 分别是Object（支持泛型)）、int（整数）、long（长整数）、double（双精度浮点数）类型的数据流抽象。

流在如下几个方面与集合不同：
- 不做存储。流不是存储元素的数据结构。相反，它从诸如数据结构、数组、生成函数或I/O通道之类的源中通过一系列计算操作（向流管道中）传递元素。
- 函数式的。对流的操作会产生结果，但不会修改源数据。例如，对从集合中获取的Stream进行过滤会生成一个不带过滤元素的新Stream，而不是从源集合中删除元素。
- 懒惰操作。许多流操作（例如过滤，映射或重复删除）总是延迟操作，从而找到优化的机会。例如，“查找以某三个字母开头的第一个字符串”不需要检查所有输入字符串。流操作分为中间（流产生）操作和终端（产生值或副作用）操作。中间操作总是惰性的。
- 可能无界。尽管集合的大小是有限的，但流不是必需的。诸如 limit(n) 或 findFirst() 之类的短路操作可以允许对无限流的计算在有限时间内完成。
- 消耗品。在流的生存期内，流的元素只能访问一次。像Iterator一样，必须生成新的流以重新访问源中的相同元素。

Stream 是 Java8 中处理集合的关键抽象概念，它可以指定你希望对集合进行的操作，可以执行非常复杂的查找、过滤和映射数据等操作。使用Stream API 对集合数据进行操作，就类似于使用 SQL 执行的数据库查询。也可以使用 Stream API 来并行执行操作。简而言之，Stream API 提供了一种高效且易于使用的处理数据的方式。

## Stream的使用流程
**1. 创建 Stream**
> 一个数据源（如：集合、数组）， 获取一个流。

**2. 中间操作**
> 一个中间操作链，对数据源的数据进行处理。

**3. 终端操作（终止操作）**
> 一个终止操作，执行中间操作链，并产生结果 。

## Stream的分类

序号 | 类名 | 说明
---|---|---
1 | BaseStream | Stream接口的父接口
2 | Stream<T> | 泛型类型的Stream
3 | IntStream | 整形Stream
4 | LongStream | 长整型Stream
5 | DoubleStream | 浮点型Stream

序号3-5的具体类型Stream提供了一些额外的方法，在下面章节【创建Stream】时有用到。它们之间的类图关系如下：

<img src="/images/posts/spring_web_flux/01_stream_diagram.png" width="100%" alt="Stream API Class Diagram" />

## 创建Stream
创建Stream主要分为如下几种方式：
1. 集合获取Stream
2. 数组创建Stream
3. 值创建Stream
4. 函数创建Stream
5. 其他方式创建Stream

由于篇幅问题及考虑到Stream提供的操作方法没有讲解，下面的示例代码中只是提供了创建Stream的示例，具体的使用示例请看代码。

`注意：以下创建Stream的方式仅为演示使用，因为Stream进行中间操作或终端操作后就会关闭，不可重复使用，因此你在使用的时候应该按函数式编程方式编写代码。`

**1. 集合获取Stream**

```java
// 返回以此集合作为源的顺序 Stream
Stream<Integer> stream = collection.stream();

// 返回以此集合作为其来源可能并行的 Stream
Stream<Integer> parallelStream = collection.parallelStream();
```

**2. 数组创建Stream**

```java
// 创建数组Stream
Stream<String> stream = Arrays.stream(strArray);
IntStream intStream = Arrays.stream(intArray);

// 数组Stream的重载
DoubleStream doubleStream = Arrays.stream(doubleArray);
IntStream intStream2 = Arrays.stream(intArray, 1, 3);
```

**3. 值创建Stream**

```java
// 构建Integer类型的Stream
IntStream stream = IntStream.of(14, 2, 31, 47, 5, 6, 9, 1, 33, 2, 6);

// 构建String类型的Stream
Stream<String> stringStream = Stream.of("Hello, Stream Api.");
```

**4. 函数创建Stream**

```java
// 方式1：使用generate方式创建一个新的无限无序Stream流
Stream<Integer> generateStream = Stream.generate(RandomUtil::randomInt);

// 方式2：使用iterate方式创建一个新的无限有序Stream流
IntStream iterateStream = IntStream.iterate(1, n -> n + 1);
``` 

**5. 其他方式创建Stream**

```java
// 方式1：创建空的顺序流
Stream<Object> emptyStream = Stream.empty();

// 方式2：使用两个Stream创建组合Stream
IntStream concatStream = IntStream.concat(intStream1, intStream2);

// 方式3：创建begin至end逐渐加1的整形Stream
IntStream rangeStream = IntStream.range(1, 501);

// 方式4：使用建造者模式创建Stream
Stream.Builder<Integer> builder = Stream.builder();
Stream<Integer> buildStream = builder.build();

// 方式5：使用StreamSupport创建Stream
// 代码忽略，具体使用请看API...
```

> 以上代码见CreateStreamTest。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>{:target="_blank"}

以上是本次笔记的内容，我们下次见。

# 参考
1. [【Java8新特性】关于Java8的Stream API，看这一篇就够了](https://www.cnblogs.com/binghe001/p/12940721.html){:target="_blank"}
