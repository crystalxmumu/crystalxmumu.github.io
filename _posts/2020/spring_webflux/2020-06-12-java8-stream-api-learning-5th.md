---
layout: post
title: Spring WebFlux 学习笔记 - (一) 前传 : 学习Java 8 Stream Api (5) - Stream 周边及其他
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Stream, Stream Api
keywords: Spring Web Flux, Stream, Reactor
---

# Stream API

经过前面 4 篇内容的学习，我们已经掌握了 Stream 大部分的知识，本节我们针对之前 Stream 未涉及的内容及周边知识点做个补充。

## Fork/Join 框架

fork/join 框架是 Java 7 中引入的 ，它是一个工具，通过 「 分而治之 」 的方法尝试将所有可用的处理器内核使用起来帮助加速并行处理。

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

```

> 以上代码见 StreamOtherTest 。

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

Fork/Join 最大的优点是提供了工作窃取算法，可以在多核CPU处理器上加速并行处理，他并非多线程开发替代品。在实践中，ExecutorService通常用于同时（并行）处理许多独立请求（又称为事务），Fork/Join通常用于加速一项连贯的工作任务。

> 以上代码见 StreamOtherTest 。



源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>

以上是本期笔记的内容，我们下期见。
