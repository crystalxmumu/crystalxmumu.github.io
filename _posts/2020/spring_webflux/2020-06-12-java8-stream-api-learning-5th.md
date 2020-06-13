---
layout: post
title: Spring WebFlux 学习笔记 - (一) 前传：学习Java 8 Stream Api (5) - Stream 周边及其他
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Stream, Stream Api
keywords: Spring Web Flux, Stream, fork/join
---

# Stream API

经过前面 4 篇内容的学习，我们已经掌握了 Stream 大部分的知识，本节我们针对之前 Stream 未涉及的内容及周边知识点做个补充。

## Fork/Join 框架

fork/join 框架是 Java 7 中引入的新特性 ，它是一个工具，通过 「 分而治之 」 的方法尝试将所有可用的处理器内核使用起来帮助加速并行处理。

在实际使用过程中，这种 「 分而治之 」的方法意味着框架首先要 fork ，递归地将任务分解为较小的独立子任务，直到它们足够简单以便异步执行。然后，join 部分开始工作，将所有子任务的结果递归地连接成单个结果，或者在返回 void 的任务的情况下，程序只是等待每个子任务执行完毕。

<img src="/images/posts/spring_web_flux/02_fork_join_principle.jpg" width="100%" alt="Fork/Join 原理图" />

为了提供有效的并行执行，fork/join 框架使用了一个名为 ForkJoinPool 的线程池，用于管理 ForkJoinWorkerThread 类型的工作线程。

### Fork/Join 优点

Fork/Join 架构使用了一种名为工作窃取（ work-stealing ）算法来平衡线程的工作负载。

简单来说，工作窃取算法就是空闲的线程试图从繁忙线程的队列中窃取工作。

默认情况下，每个工作线程从其自己的双端队列中获取任务。但如果自己的双端队列中的任务已经执行完毕，双端队列为空时，工作线程就会从另一个忙线程的双端队列尾部或全局入口队列中获取任务，因为这是最大概率可能找到工作的地方。

这种方法最大限度地减少了线程竞争任务的可能性。它还减少了工作线程寻找任务的次数，因为它首先在最大可用的工作块上工作。

### Fork/Join 使用

ForkJoinTask 是 ForkJoinPool 线程之中执行的任务的基本类型。我们日常使用时，一般不直接使用 ForkJoinTask ，而是扩展它的两个子类中的任意一个

1. 任务不返回结果 ( 返回 void ） 的 RecursiveAction
2. 返回值的任务的 RecursiveTask <V>

这两个类都有一个抽象方法 compute() ，用于定义任务的逻辑。

我们所要做的，就是继承任意一个类，然后实现 compute() 方法，步骤如下：
1. 创建一个表示工作总量的对象
2. 选择合适的阈值
3. 定义分割工作的方法
4. 定义执行工作的方法

如下是使用 Fork/Join 方式实现的1至1000006587的 Fork/Join 方式累加，我们和单线程的循环累加做了下对比，在 Intel i5-4460 的 PC 机器下，单线程执行使用了 650 ms，使用了 Fork/Join 方式执行 210 ms，优化效果挺明显。 

```java

public class NumberAddTask extends RecursiveTask<Long> {

    private static final int THRESHOLD = 10_0000;
    private final int begin;
    private final int end;

    public NumberAddTask(int begin, int end) {
        super();
        this.begin = begin;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - begin <= THRESHOLD) {
            long sum = 0;
            for(int i = begin; i <= end; i++) {
                sum += i;
            }
            return sum;
        }
        int mid = (begin + end) /2;
        NumberAddTask t1 = new NumberAddTask(begin, mid);
        NumberAddTask t2 = new NumberAddTask(mid + 1,  end);
        ForkJoinTask.invokeAll(t1, t2);
        return t1.join() + t2.join();
    }
}

// 1至1000006587的Fork/Join方式累加
@Test
public void testAddForkJoin() {
    long begin = System.currentTimeMillis();
    int n = 10_0000_6587;
    ForkJoinPool pool = ForkJoinPool.commonPool();
    log.info("1 + 2 + ... {} = {}", n, pool.invoke(new NumberAddTask(1, n)));
    long end = System.currentTimeMillis();
    log.info("ForkJoin方式执行时间：{}ms", end - begin);
}

// 1至1000006587的单线程累加
@Test
public void testAddFunction() {
    long begin = System.currentTimeMillis();
    int n = 10_0000_6587;
    long sum = 0;
    for(int i = 1; i <= n; i++ ) {
        sum += i;
    }
    log.info("1 + 2 + ... {} = {}", n, sum);
    long end = System.currentTimeMillis();
    log.info("函数方式执行时间：{}ms", end - begin);
}
```

### Fork/Join 使用场景

我使用 Java 8 官方 Api 中 RecursiveTask 的示例，创建了一个计算斐波那契数列的 Fork/Join 实现，虽然官方也提到了这是愚蠢的实现斐波那契数列方法，甚至效果还不如单线程的递归计算，但是这也说明了 Fork/Join 并非万能的。

```java
@Test
public void testForkJoin() {
    // 执行f(40) = 102334155使用3411ms
    // 执行f(80) 2个多小时，无法计算出结果
    long begin = System.currentTimeMillis();
    int n = 40;
    ForkJoinPool pool = ForkJoinPool.commonPool();
    log.info("ForkJoinPool初始化时间：{}ms", System.currentTimeMillis() - begin);
    log.info("斐波那契数列f({}) = {}", n, pool.invoke(new FibonacciTask(n)));
    long end = System.currentTimeMillis();
    log.info("ForkJoin方式执行时间：{}ms", end - begin);
}

// 不用递归计算斐波那契数列反而更快
@Test
public void testFibonacci() {
    // 执行f(50000) 使用 110ms
    // 输出 f(50000) = 17438开头的10450位长的整数
    long begin = System.currentTimeMillis();
    int n = 50000;
    log.info("斐波那契数列f({}) = {}", n, FibonacciUtil.fibonacci(n));
    long end = System.currentTimeMillis();
    log.info("函数方式执行时间：{}ms", end - begin);
}
```

> 以上代码见 StreamOtherTest 。

Fork/Join 最大的优点是提供了工作窃取算法，可以在多核CPU处理器上加速并行处理，他并非多线程开发替代品。

那么他们之间有什么区别呢？

Fork/Join框架是从jdk7中新特性,它同ThreadPoolExecutor一样，也实现了Executor和ExecutorService接口。它使用了一个无限队列来保存需要执行的任务，而线程的数量则是通过构造函数传入，如果没有向构造函数中传入希望的线程数量，那么当前计算机可用的CPU数量会被设置为线程数量作为默认值。

ForkJoinPool主要用来使用分治法(Divide-and-Conquer Algorithm)来解决问题。典型的应用比如快速排序算法。这里的要点在于，ForkJoinPool需要使用相对少的线程来处理大量的任务。比如要对1000万个数据进行排序，那么会将这个任务分割成两个500万的排序任务和一个针对这两组500万数据的合并任务。以此类推，对于500万的数据也会做出同样的分割处理，到最后会设置一个阈值来规定当数据规模到多少时，停止这样的分割处理。比如，当元素的数量小于10时，会停止分割，转而使用插入排序对它们进行排序。那么到最后，所有的任务加起来会有大概2000000+个。问题的关键在于，对于一个任务而言，只有当它所有的子任务完成之后，它才能够被执行。

所以当使用ThreadPoolExecutor时，使用分治法会存在问题，因为ThreadPoolExecutor中的线程无法像任务队列中再添加一个任务并且在等待该任务完成之后再继续执行。而使用ForkJoinPool时，就能够让其中的线程创建新的任务，并挂起当前的任务，此时线程就能够从队列中选择子任务执行。

那么使用ThreadPoolExecutor或者ForkJoinPool，会有什么差异呢？
 
首先，使用ForkJoinPool能够使用数量有限的线程来完成非常多的具有父子关系的任务，比如使用4个线程来完成超过200万个任务。但是，使用ThreadPoolExecutor时，是不可能完成的，因为ThreadPoolExecutor中的Thread无法选择优先执行子任务，需要完成200万个具有父子关系的任务时，也需要200万个线程，显然这是不可行的。

在实践中，ThreadPoolExecutor通常用于同时（并行）处理许多独立请求（又称为事务），Fork/Join通常用于加速一项连贯的工作任务。

## parallelStream 并行化

parallelStream 其实就是一个并行执行的流.它通过默认的 ForkJoinPool ，可以提高你的多线程任务的速度。parallelStream 具有并行处理能力，处理的过程会分而治之，也就是将一个大任务切分成多个小任务，这表示每个任务都是一个操作，可以并行处理。

### parallelStream 的使用

使用方式：
1. 创建时返回并行流：如 Collection<T>.parallelStream() 
2. 过程中转换为并行流：如 Stream<T>.parallel() 
3. 如果需要，转换为顺序流：Stream<T>.sequential()

```java
// 并行流时，并非按照1,2,3...500的顺序输出
IntStream.range(1, 500).parallel().forEach(System.out::println);
```

### parallelStream 的陷阱

由于 parallelStream 使用的是 ForkJoinPool 中的 commonPool，该方法默认创建程序运行时所在计算机处理器内核数量的线程，当同时存在多个工作并行执行时，ForkJoinPool 中的线程将被消耗完，而当有的worker因为执行耗时操作，将导致其他工作也被阻塞，而此时我们也不清楚哪个任务导致了阻塞。这就是 parallelStream 的陷阱。

parallelStream 是无法预测的，而且想要正确地使用它有些棘手。几乎任何 parallelStream 的使用都会影响程序中其他部分的性能，而且是一种无法预测的方式。但是在调用stream.parallel() 或者 parallelStream() 时候在我的代码里之前我仍然会重新审视一遍他给我的程序究竟会带来什么问题，他能有多大的提升，是否有使用他的意义。

那么到底是使用 stream 还是 parallelStream 呢？通过下面3个标准来鉴定

**1. 是否需要并行？**  

> 在回答这个问题之前，你需要弄清楚你要解决的问题是什么，数据量有多大，计算的特点是什么？并不是所有的问题都适合使用并发程序来求解，比如当数据量不大时，顺序执行往往比并行执行更快。毕竟，准备线程池和其它相关资源也是需要时间的。但是，当任务涉及到I/O操作并且任务之间不互相依赖时，那么并行化就是一个不错的选择。通常而言，将这类程序并行化之后，执行速度会提升好几个等级。

**2. 任务之间是否是独立的？是否会引起任何竞态条件？**

> 如果任务之间是独立的，并且代码中不涉及到对同一个对象的某个状态或者某个变量的更新操作，那么就表明代码是可以被并行化的。

**3. 结果是否取决于任务的调用顺序？** 

> 由于在并行环境中任务的执行顺序是不确定的，因此对于依赖于顺序的任务而言，并行化也许不能给出正确的结果。

## 创建流的其他方式

我们在第1篇中记录了几种创建流的方式，但还是遗漏了一部分，再此稍作补充。

### 从I/O通道 

方式1：从缓存流中读取为Stream，详见如下代码：

```java
final String name = "明玉";
// 从网络上读取文字内容
new BufferedReader(
        new InputStreamReader(
                new URL("https://www.txtxzz.com/txt/download/NWJhZjI3YjIzYWQ3N2UwMTZiNDQwYWE3")
                // new URL("https://api.apiopen.top/getAllUrl")
                        .openStream()))
        .lines()
        .filter(str -> StrUtil.contains(str, name))
        .forEach(System.out::println);
``` 

方式2：从文件系统获取下级路径及文件，详见如下代码：

```java
// 获取文件系统的下级路径及其文件
Files.walk(FileSystems.getDefault().getPath("D:\\soft"))
        .forEach(System.out::println);
```

方式3：从文件系统获取文件内容，详见如下代码：

```java
Files.lines(FileSystems.getDefault().getPath("D:\\", "a.txt"))
    // .parallel()
    .limit(200)
    .forEach(System.out::println);
```

方式4：读取JarFile内的文件，详见如下代码：

```java
new JarFile("D:\\J2EE_Tools\\repository\\org\\springframework\\spring-core\\5.2.6.RELEASE\\spring-core-5.2.6.RELEASE.jar")
        .stream()
        .filter(entry -> StrUtil.contains(entry.getName(), "Method"))
        .forEach(System.out::println);
```

### 获取随机数字流

使用类Random的ints、longs、doubles的方法，根据传递不同的参数，可以产生无限数字流、有限数字流、以及指定范围的有限或无限数字流，示例如下：

```java
double v = new Random()
        .doubles(30, 2, 45)
        .peek(System.out::println)
        .max()
        .getAsDouble();
log.info("一串随机数的最大值为：{}", v);
```

### 位向量流

将BitSet中位向量为真的转换为Stream，示例如下：

```java
BitSet bitSet = new BitSet(8);
bitSet.set(1);
bitSet.set(6);
log.info("cardinality值{}", bitSet.cardinality());
bitSet.stream().forEach(System.out::println);
```

### 正则分割流

将字符串按照正则表达式分隔成子串流，示例如下：

```java
Pattern.compile(":")
        .splitAsStream("boo:and:foo")
        .map(String::toUpperCase)
        .forEach(System.out::println);
```

## Stream 的其他方法

### 转为无序流

使用 unordered() 方法可将 Stream 随时转为无序流。

### 转换为Spliterator

使用 spliterator()  方法可将 Stream 转为 Spliterator，Spliterator 介绍请看 <https://juejin.im/post/5cf2622de51d4550bf1ae7ff>。


## 综合示例

根据1962年第1届百花奖至2018年第34届百花奖数据，有以下数据，编写代码按照获得最佳男主角的演员次数排名，次数相同的按照参演年份正序排，并打印他所参演的电影。

序号 | 最佳男主角 | 电影
---|---|---
第1届1962年 | 崔嵬 | 《红旗谱》
第2届1963年 | 张良 | 《哥俩好
第3届1980年 | 李仁堂 | 《泪痕》
第4届1981年 | 达式常 | 《燕归来》
第5届1982年 | 王心刚 | 《知音》
第6届1983年 | 严顺开 | 《阿Q正传》
第7届1984年 | 杨在葆 | 《血，总是热的》
第8届1985年 | 吕晓禾 | 《高山下的花环》
第9届1986年 | 杨在葆 | 《代理市长》
第10届1987年 | 姜文 | 《芙蓉镇》
第11届1988年 | 张艺谋 | 《老井》
第12届1989年 | 姜文 | 《春桃》
第13届1990年 | 古月 | 《开国大典》
第14届1991年 | 李雪健 | 《焦裕禄》
第15届1992年 | 王铁成 | 《周恩来》
第16届1993年 | 古月 | 《毛泽东的故事》
第17届1994年 | 李保田 | 《凤凰琴》
第18届1995年 | 李仁堂 | 《被告山杠爷》
第19届1996年 | 张国立 | 《混在北京》
第20届1997年 | 高明 | 《孔繁森》
第21届1998年 | 葛优 | 《甲方乙方》
第22届1999年 | 赵本山 | 《男妇女主任》
第23届2000年 | 潘长江 | 《明天我爱你》
第24届2001年 | 王庆祥 | 《生死抉择》
第25届2002年 | 葛优 | 《大腕》
第26届2003年 | 卢奇 | 《邓小平》
第27届2004年 | 葛优 | 《手机》
第27届2004年 | 李幼斌 | 《惊心动魂》
第28届2006年 | 吴军 | 《张思德》
第29届2008年 | 张涵予 | 《集结号》
第30届2010年 | 陈坤 | 《画皮》
第31届2012年 | 文章 | 《失恋33天》
第32届2014年 | 黄晓明 | 《中国合伙人》
第33届2016年 | 冯绍峰 | 《狼图腾》
第34届2018年 | 吴京 | 《战狼2》

根据题目要求，创建 HundredFlowersAwards 实体用来存储上述数据，我们分析题目要求最终需要转换为以演员为主的信息，然后再根据演员的获奖次数及出演年份做排序。
所以创建 ActorInfo 实体，包含 演员姓名和出演电影的信息。出演电影也需创建实体 FilmInfo ，包含 出演年份和电影名称。

有了如上存储数据实体信息后，代码实现逻辑如下：
1. 将百花奖的集合数据转换为 Stream
2. 将该数据流转换为Map类型，Map 的 key 为演员名，Map 的 Value 为演员信息
3. 对于重复出现的演员，我们需要把电影信息追加到该演员出现的电影列表中
4. 对于处理完的 Map 数据，将该 Map 的 values 数据再次转换为 Stream
5. 将该 Stream 排序即可。

```java
list.stream()
    .collect(Collectors.toMap(HundredFlowersAwards::getActorName, ActorInfo::new, ActorInfo::addFilmInfos))
    .values()
    .stream()
    .sorted(new ActorComparator())
    .forEach(System.out::println);
```

> 本节代码见 StreamOtherTest 。

经过几天的学习和总结，以上就是 Java Stream Api 的全部内容了。从开始认识 Stream Api，我们逐渐了解了使用 Stream Api 的流程：创建 Stream 、中间操作、终端操作。
我们对创建 Stream 、中间操作、终端操作的各个 api 方法进行了介绍及案例演示，之后我们还单独抽出一节讲解了 Collector 接口的实现及使用。
上述内容虽然文字不多，大部分都在代码中给出了演示，希望大家能下载下来代码并运行，以加深印象。

以上是前传部分的学习内容了，接下来我们将进入到 Reactor 部分的学习。

源码下载：<https://github.com/crystalxmumu/spring-web-flux-study-note>
