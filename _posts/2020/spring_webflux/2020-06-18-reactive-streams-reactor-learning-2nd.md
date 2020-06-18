---
layout: post
title: Spring WebFlux 学习笔记 - (二) 开篇：学习响应式编程 Reactor (2) - 初识 reactor
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Reactor 学习系列, 响应式编程
keywords: Spring Web Flux, Reactive Streams, Reactor
---

# Reactor

Reactor 是用于 Java 的异步非阻塞响应式编程框架，同时具备背压控制的能力。它与 Java 8 函数式 Api 直接集成，比如 分为CompletableFuture、Stream、以及 Duration
。它提供了异步 Api 响应流 Flux （用于 [0 - N] 个元素）和 Mono （用于 [0或1] 个元素），并完全遵守和实现了响应式规范。

## 引入 reactor

reactor 自 3.0.4 版本之后，采用了 BOM （Bill Of Materials）的方式，使用 BOM 可以管理一组良好集成的 maven artifacts，而无需担心不同版本组件之间的相互依赖问题，在 maven 项目中在 dependencyManagement 中 加入 reactor 的 bom 定义即可。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-bom</artifactId>
            <version>Dysprosium-SR8</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

在需要使用 reactor 的项目中，依赖对应 reactor 模块即可。

```xml
<dependencies>
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-core</artifactId>
    </dependency>
</dependencies>
```

在我学习的过程中，reactor 的最新版本是 Dysprosium-SR8 ，它的命名来自元素周期表，按顺序递增。通过 <https://github.com/reactor/reactor/releases> 获取最新版本。

reactor bom 中 包含了如下组件：

序号 | 模块 | artifactId | 说明
---|---|---|---
1 | Reactor Core | reactor-core | 基于 Java 8 的响应式流规范实现
2 | Reactor Test | reactor-test | reactor 的测试工具集
3 | Reactor Extra | reactor-extra | 为 Flux 额外提供的操作符
4 | Reactor Netty | reactor-netty | 基于 Netty 实现的 TCP、UDP、HTTP 的客户端和服务端
5 | Reactor Adapter | reactor-adapter | 和其他响应式库（如RxJava2、SWT Scheduler、 Akka Scheduler）的适配器
6 | Reactor Kafka | reactor-kafka | Apache Kafka 的响应式桥接实现
7 | Reactor Kotlin Extensions | reactor-kotlin-extensions | 在 Dysprosium 版本后额外提供的 Kotlin 扩展
8 | Reactor RabbitMQ | reactor-rabbitmq | RabbitMQ 的响应式桥接实现
9 | Reactor Pool | reactor-pool | 响应式应用程序的通用对象池
10 | Reactor Tools | reactor-tools | 一组用于改善 Project Reactor 调试和开发经验的工具。

序号 [1 - 3] 为我们学习 Reactor 过程中主要涉及的模块，序号 [4 - 9] 在我们学习 Spring WebFlux 的过程中会有所涉及，序号 [10] 是用于 Reactor 调试的，下面会讲到。

> 使用 gradle 的同学请自行百度。
> 如果需要尝鲜 Reactor 里程碑版或开发者预览版的同学，添加  Spring Milestones repository 的仓库即可。 

## Reactor 之 初体验

上面说了那么多，我们先来体验下 Reactor。

> 在学习 Java Stream 的环节中，不知是否有同学有提出这样的疑问：在进行中间操作或终端消费操纵时，如何获取流中元素的序号值呢？
> 假如有如下单词 [ the, quick, brown, fox, jumped, over, the, lazy, dog ] ，使用 Stream 可否实现输出时并打印每个单词的序号呢？
> 仔细想想，似乎没有直接的办法可以获取，我们只能通过外部创建变量获取并递增来实现。

来看下 Stream 的实现：

```java
AtomicInteger index = new AtomicInteger(1);
Arrays.stream(WORDS)
        .map(word -> StrUtil.format("{}. {}", index.getAndIncrement(), word))
        .forEach(System.out::println);
```

来看下 Reactor 的实现：
 
```java
Flux.fromArray(WORDS)                                                   // ①
        .zipWith(Flux.range(1, Integer.MAX_VALUE),                      // ②
                (word, index) -> StrUtil.format("{}. {}", index, word)) // ③
        .subscribe(System.out::println);                                // ④
```

先不看 Reactor 代码的含义，感觉怎么样，Reactor 的代码看起来是不是更清新一点，没有定义任何三方变量解决了这个问题。

有了 Stream 的基础，Reactor 的代码很容易理解了，我们稍微来解释下 Reactor 上段的代码：
1. 序号① 的代码 Flux 是我们之前提到的 一个能够发出 0 到 N 个元素的响应流发布者，fromArray 是它的静态方法，用来创建 Flux 响应流
2. 序号② 的代码 Flux 的 range 操作符和 Stream 的 range 相同，用来生成 整数 Flux 响应流；zipWith 操作符用来合并两个 Flux，并将响应流中的元素一一对应，当其中一个响应流完成时，合并结束，之前未结束的响应流剩下的元素将被忽略 
3. 序号③ 的代码 zipWith 操作符 支持传递一个 BiFunction 的函数式接口实现，定义如何来合并两个数据流中的元素，本例中我们将索引和单词连接起来
4. 序号④ 的代码 subscribe 即为订阅方法，此处我们做了数据流中元素输出至控制台

## Reactor 之 测试 & 调试

### 测试

Reactor 的测试需要依赖测试模块：

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
```

编写测试代码如下：

```java
// 创建 Flux 响应流
Flux<String> source = Flux.just("foo", "bar");
// 使用 concatWith 操作符连接 2 个响应流
Flux<String> boom = source.concatWith(Mono.error(new IllegalArgumentException("boom")));
// 创建一个 StepVerifier 构造器来包装和校验一个 Flux。
StepVerifier.create(boom)
        .expectNext("foo")          // 第一个我们期望的信号是 onNext，它的值为 foo
        .expectNext("bar")          // 第二个我们期望的信号是 onNext，它的值为 bar
        .expectErrorMessage("boom") // 最后我们期望的是一个终止信号 onError，异常内容应该为 boom
        .verify();                  // 使用 verify() 触发测试。
```

除了正常测试外，Reactor 还提供了诸如：
1. 测试基于时间操作符相关的方法，使用 StepVerifier.withVirtualTime 来进行
2. 使用 StepVerifier 的 expectAccessibleContext 和 expectNoAccessibleContext 来测试 Context
3. 用 TestPublisher 手动发出元素
4. 用 PublisherProbe 检查执行路径

测试方面暂时不是我们学习的重点，这块内容，我们快速跳过，等到学习到相关场景，需要的时候，我们回过头来再弥补。

### 调试

响应式编程的调试令人生畏，因为它不像命令式编程，我们可以从异常的堆栈信息中看到发生错误代码的位置及具体错误信息，这也是响应式编程学习曲线比较陡峭的原因。

有如下代码：

```java
Flux.range(1, 3)
    .flatMap(n -> Mono.just(n + 100))
    .single()
    .map(n -> n * 2)
    .subscribe(System.out::println);
```

执行测试时，打印错误日志如下：

```java
reactor.core.Exceptions$ErrorCallbackNotImplemented: java.lang.IndexOutOfBoundsException: Source emitted more than one item

Caused by: java.lang.IndexOutOfBoundsException: Source emitted more than one item
	at reactor.core.publisher.MonoSingle$SingleSubscriber.onNext(MonoSingle.java:129)
	at reactor.core.publisher.FluxFlatMap$FlatMapMain.tryEmitScalar(FluxFlatMap.java:480)
	at reactor.core.publisher.FluxFlatMap$FlatMapMain.onNext(FluxFlatMap.java:413)
	at reactor.core.publisher.FluxRange$RangeSubscription.slowPath(FluxRange.java:154)
	at reactor.core.publisher.FluxRange$RangeSubscription.request(FluxRange.java:109)
	at reactor.core.publisher.FluxFlatMap$FlatMapMain.onSubscribe(FluxFlatMap.java:363)
	at reactor.core.publisher.FluxRange.subscribe(FluxRange.java:68)
	at reactor.core.publisher.Mono.subscribe(Mono.java:4219)
	at reactor.core.publisher.Mono.subscribeWith(Mono.java:4330)
	at reactor.core.publisher.Mono.subscribe(Mono.java:4190)
	at reactor.core.publisher.Mono.subscribe(Mono.java:4126)
	at reactor.core.publisher.Mono.subscribe(Mono.java:4073)
	at top.todev.note.web.flux.reactor.ReactorFirstExperienceTest.testReactorDebug(ReactorFirstExperienceTest.java:79)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	...
	at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:68)
```

我们从上述的错误中获取到发生了 IndexOutOfBoundsException 数据越界异常，从上往下看，应该是 MonoSingle 响应式流发出了不止一个元素，查看 Mono#singe 操作符描述，我们看到 single 有一个规定： 源必须只能发出一个元素。看来是有一个源发出了多于一个元素，从而违反了这一规定。

粗略过一下这些行，我们可以大概勾画出一个大致的出问题的链：涉及 MonoSingle、FluxFlatMap、FluxRange（每一个都对应 trace 中的几行，但总体涉及这三个类）。 所以难道是 range().flatMap().single() 这样的链？

但是如果在我们的应用中多处都用到这一模式，那怎么办？通过这些还是不能确定为什么会抛除这个异常， 搜索 single 也找不到问题所在。直到最后几行指向了我们的代码，查看代码和我们之前的预测的调用链一样。

但是最终我们怎么快速确定代码的问题在哪里呢？

**方案1：** 开启调试模式

> 使用 Hooks.onOperatorDebug(); 在程序初始的地方开启调试模式

错误日志如下：

```java
reactor.core.Exceptions$ErrorCallbackNotImplemented: java.lang.IndexOutOfBoundsException: Source emitted more than one item

Caused by: java.lang.IndexOutOfBoundsException: Source emitted more than one item
	at reactor.core.publisher.MonoSingle$SingleSubscriber.onNext(MonoSingle.java:129)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.MonoSingle] :
	reactor.core.publisher.Flux.single(Flux.java:7851)
	top.todev.note.web.flux.reactor.ReactorFirstExperienceTest.testReactorDebug(ReactorFirstExperienceTest.java:83)
Error has been observed at the following site(s):
	|_ Flux.single ⇢ at top.todev.note.web.flux.reactor.ReactorFirstExperienceTest.testReactorDebug(ReactorFirstExperienceTest.java:83)
	|_    Mono.map ⇢ at top.todev.note.web.flux.reactor.ReactorFirstExperienceTest.testReactorDebug(ReactorFirstExperienceTest.java:84)
Stack trace:
		at reactor.core.publisher.MonoSingle$SingleSubscriber.onNext(MonoSingle.java:129)
		at reactor.core.publisher.FluxFlatMap$FlatMapMain.tryEmitScalar(FluxFlatMap.java:480)
		at reactor.core.publisher.FluxFlatMap$FlatMapMain.onNext(FluxFlatMap.java:413)
		at reactor.core.publisher.FluxRange$RangeSubscription.slowPath(FluxRange.java:154)
		at reactor.core.publisher.FluxRange$RangeSubscription.request(FluxRange.java:109)
		at reactor.core.publisher.FluxFlatMap$FlatMapMain.onSubscribe(FluxFlatMap.java:363)
		at reactor.core.publisher.FluxRange.subscribe(FluxRange.java:68)
		at reactor.core.publisher.Mono.subscribe(Mono.java:4219)
		at reactor.core.publisher.Mono.subscribeWith(Mono.java:4330)
		at reactor.core.publisher.Mono.subscribe(Mono.java:4190)
		at reactor.core.publisher.Mono.subscribe(Mono.java:4126)
		at reactor.core.publisher.Mono.subscribe(Mono.java:4073)
		at top.todev.note.web.flux.reactor.ReactorFirstExperienceTest.testReactorDebug(ReactorFirstExperienceTest.java:85)
```

我们从 Error has been observed at the following site(s) 这行错误起，可以看到错误沿着操作链传播的轨迹（从错误点到订阅点），我们从 Assembly trace from
 producer 这行开始的错误中也看到了源代码 83 行开始报错，也确定了上一行的 flatMap 操作符发出了不止一个元素导致。
 
**方案2：** 使用 checkpoint 操作符埋点调试

> 使用方案1开启全局调试有较高的成本即影响性能，我们可以在可能发生错误的代码中加入操作符 checkpoint 来检测本段响应式流的问题，而不影响其他数据流的执行。
> checkpoint 通常用在明确的操作符的错误检查，类似于埋点检查的概念。同时该操作符支持 3个重载方法：checkpoint(); checkpoint(String description); checkpoint
> (String description, boolean forceStackTrace);
> description 为埋点描述，forceStackTrace 为是否打印堆栈

**方案3：** 启用调试代理

**1. 在项目中引入 reactor-tools 依赖**

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-tools</artifactId>
</dependency>
```

**2. 使用 ReactorDebugAgent.init(); 初始化代理**

> 由于该代理是在加载类时对其进行检测，因此放置它的最佳位置是在main（String []）方法中的所有其他项之前

**3. 如果是测试类，使用如下代码处理现有的类**

> 注意，在测试类中需要提前运行，比如在 @Before 中

```java
ReactorDebugAgent.init();
ReactorDebugAgent.processExistingClasses();
```

# 总结

本篇我们了解了如何引入 Reactor ；初步体验了 Reactor 的 Hello World 代码；最后我们了解了如何测试及调试 Reactor，这些内容为我们后面学习 Reactor 的基础，希望大家都能掌握。

今天的内容就学到这里，我们下篇开始 Reactor 的基础和特性学习。

源码详见：<https://github.com/crystalxmumu/spring-web-flux-study-note>{:target="_blank"} 下 02-reactor-core-learning 模块。

# 参考
1. [Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference/){:target="_blank"}
2. [Reactor 3 中文指南](https://github.com/get-set/reactor-core/blob/master-zh/src/docs/index.html){:target="_blank"}