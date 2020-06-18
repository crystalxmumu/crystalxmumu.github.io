---
layout: post
title: Spring WebFlux 学习笔记 - (二) 开篇：学习响应式编程 Reactor (1) - 响应式编程
categories: SpringWebFlux
description: Spring Web Flux 学习笔记, Reactor 学习系列, 响应式编程
keywords: Spring Web Flux, Reactive Streams, Reactor
---

# 响应式编程

> 命令式编程（Imperative Programing），是一种描述计算机所需做出的行为的编程范式。详细的命令机器怎么（How）去处理以达到想要的结果（What）。
> 声明式编程（Declarative Programing），是一种编程范式，与命令式编程相对立。它描述目标的性质，让计算机明白目标，而非流程。只告诉机器想要的结果（What），机器自己摸索过程（How）。

响应式编程（Reactive Programing）是一种关注数据流（data streams）和变化传递（propagation of charge）的异步编程方式，它属于声明式编程范式。

上面的文字理论性比较强，说的直白一点：

> 数据流就像一条河：它可以被观测，被过滤，被操作（和 Java Stream 一致），或者为新的消费者与另外一条流合并为一条新的流。
> 响应式编程的一个关键概念是事件。事件可以被等待，可以触发过程，也可以触发其它事件。事件是唯一以合适的方式将我们的现实世界映射到我们的软件中：如果屋里太热了，我们就打开一扇窗。
> 同样的，当我们的天气 app 从服务器端获取到新的天气数据后，我们需要更新 app 上展示天气的 UI ；汽车上的车道偏移系统探测到车辆偏移了正常路线就会提醒驾驶者纠正；这些就是响应事件。

响应式编程可以理解为面向对象中观察者模式（Observer Design Pattern）的一种扩展，它也与迭代器模式（Iterator Design Pattern）
有相同之处，响应式流（reactive streams）其中也有 Iterable - Iterator 这样的对应关系，主要的区别在于 Iterator 是基于拉取（pull）
方式的，而响应式流是基于推送（push）方式的。

Iterator 迭代器（模式）也是命令式编程方式，即使访问值的方法使用的是 iterable 方法，什么时候获取 next 是由开发者决定的。
在响应式流中，相对应的角色是发布者（publisher） - 订阅者（Subscriber），当有新值到来的时候，是由发布者通知订阅者，这种
典型的推送方式是响应式流的关键，同时对推送来的数据的操作是声明式的表达，而不是命令式的，开发者通过描述控制流程来定义对
数据流的处理逻辑。

除了数据推送，响应式流还提供了数据流完成的信号和发生错误的信号。一个发布者可以随时向订阅者推送数据（onNext），同时也可以
推送错误（onError）和完成信号（onComplete），错误和完成信号都将终止响应流。

说了这么多理论性描述，那么响应式编程到底牛叉在哪里呢？

## 阻塞是资源浪费

以现实中的双11秒杀为例，当大量用户同时在0点发起抢购某个商品时，假设在不做任何限流、架构优化等的情况下，大量的请求将同时
进入到后端，以tomcat容器为例，tomcat将为用户创建大量的线程（受线程池控制）来响应用户请求，后端的代码收到请求后，需要执行如下逻辑：判断
用户的下单请求是否合理有效，判断用户是否参与过当前商品的秒杀，判断当前商品库存是否充足，如果库存充足，执行下单锁库存，
随着并发数的加大，这一套代码逻辑执行下来将会花费很长时间，同时tomcat之前的线程一直阻塞着，等待servlet的结果来最终响应给用户。

现代应用面对大量的并发用户，即使硬件的处理能力高速发展，软件性能仍是关键因素。

广义来说，有两种方式来提高软件性能：
1. 并行化，使用更多的线程和硬件资源；
2. 优化现有（代码）资源，提高执行效率。

通常，开发者使用阻塞式编写代码，在出现性能瓶颈后，我们可以增加处理线程，线程中同样是阻塞的代码。但是这种使用资源的方式
会面临资源的竞争和并发问题。同时，阻塞会浪费资源。具体来说，当一个程序面临延迟（通常是 I/O 方面，比如数据库读写和网络请求），
所在线程进入 idle 状态等待返回，从而浪费资源。所以并行化并非解决问题的最佳方式，它是挖掘硬件潜力的方式，但是带来了复杂性，
并造成了对资源的浪费。

## 异步可以解决问题嘛？

第 2 个思路，提高执行效率，通过编写异步非阻塞的代码，任务发起异步调用后，执行过程会切换到另一个（使用同样底层资源）活跃任务，
等待异步任务返回结果后再去处理，这样可以解决资源的浪费问题。

Java 提供了2种方式实现异步代码：
1. 回调：异步方法没有返回值，而是采用一个 callback 作为参数，当有结果后，回调这个 callback。常见的例子，比如 Swing 中
的 EventListener。
2. Future：异步方法立即返回一个 Future<T> ，Future 的结果不是立马可以拿到，需要等待异步任务执行完成后，才可以使用。

但是这两个方法都有他们的局限性：
1. 如果在回调中嵌套回调时，多层嵌套的回调将导致代码难以理解和维护，即所谓的嵌套地狱。
2. Future 比回调要好一点，即使在 Java 8 中引入了 CompletableFuture，他对于多个处理的组合仍不友好。编排多个 Future 是可行的，
但却不易。此外，Future 还有一个问题，当对 Future 对象调用 get() 方法时，仍然会导致阻塞，并且缺乏对多个值以及更进一步对错误
的处理。

## 从命令式编程到响应式编程

基于上面提到的问题，开发牛人们提出了响应式流解决方案。在响应式编程方面，微软是先行者，他们率先在 .NET 中创建了响应式扩展库（
Reactive Extensions Library, Rx）。接着，RxJava 在 JVM 上实现了响应式编程。后来，在 JVM 平台出现了一套响应式编程规范，
它定义了一系列编程接口和交互规范，并整合到了 Java 9 中。

除了解决上述问题，响应式编程库还额外关注如下几个方面：
1. 可编排性（Composability）和可读性（Readability）
2. 提供丰富的操作符（operators）来处理形如流的数据
3. 在订阅（Subscribe）之前什么都不发生
4. 背压（BackPressure）支持，简单来说，订阅者能够反向告知发布者生产内容速度的能力
5. 高层次的抽象，从而达到并发无关的效果

# RxJava 和 Reactor

## RxJava

RxJava 是 Reactive Extensions 的 Java VM 实现，它用于通过使用可观察的序列来组成异步和基于事件的程序。

它扩展了观察者模式以支持数据/事件序列，并添加了运算符，使您可以声明性地将序列组合在一起，同时消除了对诸如多线程，
同步，线程安全和并发数据结构之类的问题的担忧。

RxJava 是 Java 界响应式编程的先行者，因为是先有的 RxJava，才后有的 Java 版本的响应式编程规范，同时该规范定义时参考了 
RxJava 的许多定义，RxJava 的早期版本最低支持 Java 6，官方版本直至 Java 9 中才被支持。

RxJava 最新版本 3.x，最低需要 Java 8+，官方请看 <https://github.com/ReactiveX/RxJava>。

## Reactor

Reactor 是一个用于 JVM 的完全非阻塞的响应式编程框架，具备高效的需求管理（即对 “背压（backpressure）”的控制）能力。它与 
Java 8 函数式 API 直接集成，比如 CompletableFuture， Stream， 以及 Duration。它提供了异步序列 API Flux（用于[N]个元素）
和 Mono（用于 [0|1]个元素），并完全遵循和实现了“响应式扩展规范”（Reactive Extensions Specification）。

Reactor 的 reactor-ipc 组件还支持非阻塞的进程间通信（inter-process communication, IPC）。 Reactor IPC 为 HTTP（包括 Websocket）、
TCP 和 UDP 提供了支持背压的网络引擎，从而适用于微服务架构。并且完整支持响应式编解码（reactive encoding and decoding）。

Reactor 是第四代响应式框架，跟 RxJava 2 有些相似。Reactor 项目由 Pivotal 启动，以响应式流规范、Java8 和 ReactiveX 
术语表为基础。它的设计是 Reactor 2（上一个主要版本）和 RxJava 核心贡献者共同努力的结果。

从设计概念方面来看，RxJava 有点类似 Java 8 Steams API。而 Reactor 看起来有点像 RxJava，不过这决不只是个巧合。这样的设计
是为了能够给复杂的异步逻辑提供一套原生的具有 Rx 操作风格的响应式流 API。所以说 Reactor 扎根于响应式流，同时在 API 方面尽
可能地与 RxJava 靠拢。

Reactor 最新版本3.x，最低需要 Java 8+，官方请看 <https://github.com/reactor/reactor-core>。

## Java Stream Vs RxJava Vs Reactor 

我们在前传中首先学习了Java Stream，通过上面笔记的介绍，发现 Java Stream 在很多方面与响应式编程方面相似，那么他们到底有
哪些区别呢，来看**徐靖峰**在【八个层面比较 Java 8, RxJava, Reactor】中的下面这张图：

<img src="/images/posts/spring_web_flux/03_stream_vs_rxjava_vs_reactor.png" width="100%" alt="八个层面比较 Java 8, RxJava, Reactor" />

由于上述文章已经讲解的很好了，请大家跳转过去学习一下。

# 总结

RxJava 在 Android 开发界可算是如火如荼，通过事件的响应配合界面的操作可谓如鱼得水。Reactor 直至 Spring 5 中引入了 Reactive
Stream 及 Spring WebFlux 才进入了我的视线，RxJava 的学习内容基本遍地都是了，而 Reactor 的内容却少之又少，这也是本片笔记的由来。

响应式编程的理论部分，总算是介绍完了。文字描述一直是我的弱势，上面的大部分内容都是源自 Reactor 的官方文档及下方各个参考
文档，感兴趣的朋友可到对应的文章详细了解下。

今天的内容就讲到这里，我们下篇开始 Reactor 的学习。

# 参考
1. [响应式规范](http://reactive-streams.org/){:target="_blank"}
2. [Reactor 3 中文指南](https://github.com/get-set/reactor-core/blob/master-zh/src/docs/index.html){:target="_blank"}
3. [这可能是最好的RxJava 2.x 教程（完结版）](https://www.jianshu.com/p/0cd258eecf60){:target="_blank"}
4. [重新理解响应式编程](https://www.jianshu.com/p/c95e29854cb1){:target="_blank"}
5. [八个层面比较 Java 8, RxJava, Reactor](https://www.cnkirito.moe/comparing-rxjava/){:target="_blank"}