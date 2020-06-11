---
layout: post
title: Spring WebFlux 学习笔记 - (一) 前传 : 学习Java 8 Stream Api (2) - Stream的中间操作
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Stream, Stream Api
keywords: Spring Web Flux, Stream, Reactor
---

# Stream API
Java8中有两大最为重要的改变：第一个是 Lambda 表达式；另外一个则是 Stream API(java.util.stream.*)。

Stream 是 Java8 中处理集合的关键抽象概念，它可以指定你希望对集合进行的操作，可以执行非常复杂的查找、过滤和映射数据等操作。使用Stream API 对集合数据进行操作，就类似于使用 SQL 执行的数据库查询。也可以使用 Stream API 来并行执行操作。简而言之，Stream API 提供了一种高效且易于使用的处理数据的方式。

> 流(Stream)是数据渠道，用于操作数据源（集合、数组等）所生成的元素序列。"集合讲的是数据，流讲的是计算！ "

集合和流(Stream)，表面上有一些相似之处，他们有不同的目标。集合主要关注其元素的有效管理和访问，相比之下，流不提供直接访问或操纵元素的手段，而是关心声明性地描述其源和将在该源上进行聚合的计算操作。 

上篇内容我们学习了创建Stream，接下来我们来看下Stream数据流的消费处理，即中间操作。

## Stream的中间操作
为了执行计算，流组合操作被组合成流管道。流管道由源（其可以是数组、集合、生成函数、I/O通道等）组成，零个或多个中间操作（其将流转换成另一个流，例如 filter(Predicate) ）以及终端操作（产生结果或副作用，如count()或forEach(Consumer)）。

流是惰性的，源数据上的计算仅在终端操作启动时执行，源元素仅在需要时才被使用。

所谓中间操作，即为经过该操作后仍返回Stream的方法，只是(流)数据元素发生了变化。因为仍返回Stream，编程时可以通过连续的多个中间操作，逐渐变为想要的数据流，最后进行`终端(终止)操作`，变为最终的结果数据。

我们将中间操作分为如下几类：
1. 筛选
2. 映射
3. 排序

`以下内容提到XxxStream代表IntStream、LongStream、DoubleStream。如无特殊说明Stream也包含IntStream、LongStream、DoubleStream。`

### 筛选

筛选操作方法即源流经过筛选或过滤后，输出新的数据流。

序号 | 支持的类 | 方法定义 | 方法说明
---|---|---|---
1 | Stream<T> | Stream<T> distinct(); | 去重，返回由该流的不同元素（根据 Object.equals(Object) ）组成的流。
2 | Stream<T> | Stream<T> filter(Predicate<? super T> predicate); | 过滤，返回由与此给定谓词匹配的元素组成的流。
3 | Stream<T> | Stream<T> limit(long maxSize); | 限制元素数量，返回截短长度不超过 maxSize 的元素组成的流。
4 | Stream<T> | Stream<T> skip(long n); | 跳过头部元素，在丢弃流的第 n 个元素后，返回由该流的元素 n 之后组成的流。
5 | Stream<T> | Stream<T> peek(Consumer<? super T> action); | 窥探流内元素，返回由该流的元素组成的流，在从生成的流中消耗元素时对每个元素执行提供的操作。

> 以下代码见 StreamSearchOperationTest。

#### distinct的使用

```java
int[] array = new int[]{1, 2, 3, 4, 5, 6, 1, 2, 3};
// count为终端操作，代表统计流中元素个数
log.info("{}数据去重后元素个数为{}个", array, Arrays.stream(array).distinct().count());
```

#### filter的使用

```java
log.info("整数3-250中大于等于21,小于148的奇数有{}个",
        IntStream.range(3, 251).filter(n -> n >= 21 && n < 148 && n % 2 == 1).count());
```

#### limit的使用

```java
log.info("随机生成3-50之间的20个正整数，如下：");
// forEach为终端操作，代表循环消费流中元素
LongStream.generate(() -> RandomUtil.randomLong(3, 50)).limit(20).forEach(System.out::println);
```

#### skip的使用

```java
int[] array = new int[]{1, 2, 3, 4, 5, 6, 1, 2, 3};
log.info("{}数据去重头部3个元素后变为：", array);
Arrays.stream(array).skip(3).forEach(System.out::println);
```

#### peek的使用

```java
// sum为终端操作，代表流中元素的总和
log.info("以上一串随机浮点数和为：{}",
        DoubleStream.generate(RandomUtil::randomDouble).limit(30).peek(System.out::println).sum());
```

#### 综合示例1

统计如下英文名句的使用的字母个数，要求：忽略大小写，去重、不用重复统计。

> Cease to struggle and you cease to live.(Thomas Carlyle) 

> 生命不止，奋斗不息。(卡莱尔)

根据题目需求，分析实现如下：

1. 先将语句转换为小写
2. 将语句转换为字符 (Stream) 流
3. 过滤出字符流中的字母
4. 去除字符流中的重复字母
5. 打印字符流中的每个字母
6. 输出使用字母个数

最终代码实现如下：

```java
String sentence = "Cease to struggle and you cease to live.(Thomas Carlyle)";
// 正则 \p{Lower} 小写字母字符：[a-z]
// 正则 \p{Upper} 大写字母字符：[A-Z]
// 正则 \p{Alpha} 字母字符：[\p{Lower}\p{Upper}]
log.info("语句[{}]使用的字母个数为：{}", sentence,
        Arrays.stream(sentence.toLowerCase().split(""))
                .filter(str -> str.matches("\\p{Alpha}"))
                .distinct()
                .peek(System.out::println)
                .count()
);
```

#### 综合示例2

在1-3000的整数中，70个数为1组，统计第34组的质数个数，并输出他们。

根据题目需求，实际上该题目为分页模型的变体，取出每页70条第34页的质数，分析实现如下：

1. 创建1-3000的整数数字流
2. 忽略前33页的整数
3. 取出第34页的70个整数
4. 过滤为质数的整数
5. 打印他们
6. 输出使用字母个数

最终代码实现如下：

```java
int page = 34;
int pageNo = 70;
int total = 3000;
log.info("1-{}中每页显示{}条数据，第{}页数据中的质数如下:", total, 70, 34);
// isPrimes 方法判断数字是否是质数（素数）
log.info("质数个数为：{}",
        IntStream.range(1, total + 1)
                .skip(page * pageNo)
                .limit(pageNo)
                .filter(NumberUtil::isPrimes)
                .peek(System.out::println)
                .count()
);
```

### 映射

映射操作方法即源流经过转换后，元素类型发生了变化，生成了新的数据流。

序号 | 支持的类 | 方法定义 | 方法说明
---|---|---|---
1 | Stream<T> | <R> Stream<R> map(Function<? super T, ? extends R> mapper); | 返回由给定函数应用于此流的元素的结果组成的流。
2 | Stream<T> | IntStream mapToInt(ToIntFunction<? super T> mapper); | 返回一个 IntStream ，其中包含将给定函数应用于此流的元素的结果。 
3 | Stream<T> | LongStream mapToLong(ToLongFunction<? super T> mapper); | 返回一个 LongStream ，其中包含将给定函数应用于此流的元素的结果。 
4 | Stream<T> | DoubleStream mapToDouble(ToDoubleFunction<? super T> mapper); | 返回一个 DoubleStream ，其中包含将给定函数应用于此流的元素的结果。 
5 | XxxStream<T> | <U> Stream</U> mapToObj(XxxFunction<? extends U> mapper); | 返回一个对象值 Stream ，其中包含将给定函数应用于此流的元素的结果。
6 | Stream<T> | <R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper); | 返回由通过将提供的映射函数应用于每个元素而产生的映射流的内容来替换该流的每个元素的结果的流。 
7 | Stream<T> | IntStream flatMapToInt(Function<? super T, ? extends IntStream> mapper); | 返回一个 IntStream ，其中包含将该流的每个元素替换为通过将提供的映射函数应用于每个元素而产生的映射流的内容的结果。 
8 | Stream<T> | LongStream flatMapToLong(Function<? super T, ? extends LongStream> mapper); | 返回一个 LongStream ，其中包含将该流的每个元素替换为通过将提供的映射函数应用于每个元素而产生的映射流的内容的结果。 
9 | Stream<T> | DoubleStream flatMapToDouble(Function<? super T, ? extends DoubleStream> mapper); | 返回一个 DoubleStream ，其中包含将该流的每个元素替换为通过将提供的映射函数应用于每个元素而产生的映射流的内容的结果。 
10 | XxxStream<T> | XxxStream flatMap(XxxFunction<? extends IntStream> mapper); | 返回由通过将提供的映射函数应用于每个元素而产生的映射流的内容来替换该流的每个元素的结果的流。 

Stream的映射方法主要提供了map和flatMap两个方法，二者的区别为map方法的参数生成的是对象，而flatMap方法的参数生成的是Stream，并将这些生成的Stream连接(concat)起来。flatMap类似于对每个流内元素执行map后的结果执行concat方法。可以理解为map方法生成的新Stream流和之前的旧Stream流的元素比为 1:1，而flatMap方法每个元素生成的Stream元素个数为零到多个，最终连接 (concat) 后，新旧元素比为 n:1。

map和flatMap都提供了不同的重载方法，含义一致，下面测试代码不再复述。

> 以下代码见 StreamMapOperationTest。

#### map的使用

```java
log.info("1-20的整数每个增加1543后为:");
IntStream.range(1, 21)
        .map(n -> n + 1543)
        .forEach(System.out::println);
```

#### flatMap的使用

```java
// 方式1： 将一个元素变为零个或多个元素组成的Stream
// 需求1： 输出1-20中每个奇数的前后及其本身，并求和
log.info("1-20的中奇数前后元素及求和为:{}",
        IntStream.range(1, 21)
        .flatMap(n -> n % 2 == 0 ? IntStream.empty() : IntStream.of(n - 1, n, n +1))
        .peek(System.out::println)
        .sum()
);

// 方式2：将一个元素变为多个元素组成的Stream
// 需求2：
// 生命不止，奋斗不息。(卡莱尔)
String sentence1 = "Cease to struggle and you cease to live.(Thomas Carlyle).";
// 过一种高尚而诚实的生活。当你年老时回想起过去，你就能再一次享受人生。
String sentence2 = "Live a noble and honest life. Reviving past times in your old age will help you to enjoy your life again.";
// 充实今朝，昨日已成过去，明天充满神奇。
String sentence3 = "Enrich your life today. Yesterday is history，and tomorrow is mystery.";
log.info("语句[{}\n{}\n{}\n]使用的字母个数为：{}", sentence1, sentence2, sentence3,
        Stream.of(sentence1, sentence2, sentence3)
                .flatMap(str -> Arrays.stream(str.toLowerCase().split("")))
                .filter(str -> str.matches("\\p{Alpha}"))
                .distinct()
                .peek(System.out::println)
                .count()
);
```

### 排序
排序操作方法即源流内的元素经过排序后，输出新的数据流。

序号 | 支持的类 | 方法定义 | 方法说明
---|---|---|---
1 | Stream<T> | Stream<T> sorted(); | 正序排序，返回由此流的元素组成的流，根据自然顺序排序。 
2 | Stream<T> | Stream<T> sorted(Comparator<? super T> comparator); | 返回由该流的元素组成的流，根据提供的 Comparator进行排序。 

> 以下代码见 StreamSortedOperationTest。

#### sorted的使用

```java
log.info("随机生成3-50之间的20个正整数，排序如下：");
// forEach为终端操作，代表循环消费流中元素
LongStream.generate(() -> RandomUtil.randomLong(3, 50))
        .limit(20)
        .distinct()
        .sorted()
        .forEach(System.out::println);
```

#### sorted的使用(使用Comparator比较)

```java
log.info("随机生成3-50之间的20个正整数，倒序排序如下：");
// forEach为终端操作，代表循环消费流中元素
Stream.generate(() -> RandomUtil.randomInt(3, 50))
        .limit(20)
        .distinct()
        .sorted(Comparator.comparingInt(o -> -o))
        .forEach(System.out::println);
```

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>

以上是本期笔记的内容，我们下期见。
