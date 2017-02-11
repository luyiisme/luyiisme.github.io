---
layout:     post
title:      "reactor 介绍(1)"
date:       2017-01-21
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - reactive streams
    - reactive
    - RxJava
    - reactor
    - spring5
---

## Reactor 是什么？

很多人应该都知道 RxJava 项目，在 Netflix 被大规模使用来实现响应式编程。而Spring 社区也打算在 spring5中支持和推广 reactive 编程，但是并没有采用 RxJava 作为基础库，而是使用 Reactor3 项目（早期该项目一直研究消息的高效派发，到第二版后开始尝试实现 [Reactive Streams](http://www.reactive-streams.org/) 规范，终于在第三版宣告成功了）。

从目前一些 spring 5 的资料可以看到,Reactor 项目的 api （尤其是 Mono/Flux 两个类型），将被 Spring MVC,Spring Data 等子项目广泛使用。Spring (Pivotal) 的 Reactor 3 项目进行了较大革新，是一个针对Java 8，并遵守Rx规范的API的 Reactive Streams 代码库.与 RxJava 库设计原理与哲学基本一致，尽管有一些API差异。Reactor3 属于[第四代响应式库](http://akarnokd.blogspot.jp/2016/03/operator-fusion-part-1.html)（RxJava 2同样也是），允许操作聚合。

因此，Reactor 项目是 Spring Framework 5 的 响应式（reactive）编程的核心基石。

## RxJava 回顾

Reactor，和 RxJava 2一样，是第四代反应库。它由Pivotal发行，建立在 Reactive Streams 规范、Java 8 和 ReactiveX 概念之上。它的设计是由来自Reactor 2（上一个主要版本）和RxJava结合发展而来。

Observable 类（RxJava 库）是（数据流）推送源，Observer 是通过订阅行为来消费这个源的一个简单接口类。请记住，Observable通过 onNext 通知对应的 Observer (产生的)0~N 个数据项，（可选地）可以继续 onError 或 onComplete 终止事件。

为了测试一个 Observable 对象，RxJava 提供了一个 TestSubscriber，它是一个特殊的 Observer 风格，允许断言你流里的事件。

## Reactor 的类型

Reactor 创造了两个主要类型：`Flux<T>`和`Mono<T>`。

- Flux 相当于一个 RxJava Observable，能够发出 0~N 个数据项，然后（可选地）completing 或 erroring。

- Mono ,是指最多只能触发(emit) (事件)一次。它对应于 RxJava 库的 Single 和 Maybe 类型。因此一个异步任务，如果只是想要在完成时给出完成信号，就可以使用 `Mono<Void>`。

同时在 reactive API 中提供这两类区分，使得按场景使用变得简单。通过查看返回的 reactive 类型，我们可以知道该方法是“fire-and-forget”,“request-response“（Mono）型,还是属于处理多个数据项作为stream（Flux）。

Flux 和 Mono 可以通过一些运算符实现强制类型转换。例如，调用 single() 一个 `Flux<T>` 将返回一个 `Mono<T>`，而连接两个 monos一起使用 concatWith 将产生一个 Flux。然而有一些运算符是没有意义的，比如 Mono 使用 take(n)，其产生 n> 1个结果。

Reactor 设计理念的一个方面是保持 API 精简，分离出两种 Reactive 类型是基于概念表达性和 api 易用性的综合考量。

### 基于 Rx 与 Reactive Streams

RxJava (由于产生较早的缘故,基于Java6),具有一些与 Java8 概念相似的 Streams API。Reactor项目看起来很像 RxJava，这当然不是巧合。目的是提供一个遵循 Rx 规范的 Reactive Streams 本地库，用于组合一些异步逻辑。因此，虽然 Reactor 根植于 Reactive Streams，但它尽可能地寻求与 RxJava 的一般 API 对齐。

## Reactive 库和 Reactive Streams 的采用

Reactive Streams（RS）是“提供用于具有非阻塞 back pressure(背压)的异步流处理的标准倡议”。它是一组文本规范，同时具有 TCK 和四个简单接口（Publisher，Subscriber，Subscription和Processor），它们将被集成在 Java9 里。

它主要处理 「reactive-pull」 「back-pressure」 的概念（以后更多），以及如何在几个实现 reactive 源之间进行互操作。它不包括运算符定义，只专注于流的生命周期。

Reactor 是 RS 第一个实现。Flux 和 Mono 为 RS 规范中 Publisher 实现，并符合 「reactive-pull」 「back-pressure」。

在RxJava 1中，只有一小部分运算符支持 back-pressure，即使 RxJava 1 具有 RS 类型的适配器，它的 Observable 也不会直接实现这些类型。毕竟 RxJava 1先于RS规范，并在规范的设计期间作为基础参考作品。

这意味着，每次你使用这些适配器， Publisher 也没有任何运算符，为了做任何有用的操作，你可能又想退回使用 Observable，这意味着使用另一个适配器。这种混乱的使用方式不具有可读性，特别是当像Spring 5这样的整个框架直接建立在 Publisher 上时。

从 RxJava 1 迁移到 Reactor 或 RxJava 2 时，记住与的另一个区别是，在 RS 规范中，null值不被允许。如果你的代码库 null 用来表示一些特殊情况，这点需要注意。

RxJava 2是在 Reactive Streams 规范之后开发的，因此(mono/flux)直接实现 Publisher 。RxJava 2 除了关注 RS 类型，还保持了“传统” RxJava 1类型（Observable，Completable，和Single），并引入了“RxJava Optional” Maybe 类型。但是这些类型还是没有实现 RS 接口。注意，与RxJava 1不同，Observable 在RxJava 2 中不支持 backpressure 协议（现在专门保留的功能 Flowable）。它被保留用于为诸如用户界面事件（这类场景 backpressure 是不切实际或不可能）处理提供丰富且流畅的API。Completable，Single 和 Maybe 通过设计不需要 backpressure 支持，它们将提供丰富的 API 以及推迟任何工作负载直到被订阅。

Reactor Mono 和 Flux 类型，实现了 Publisher 和 backpressure-ready。Mono 作为一个 Publisher 开销（大部分被其他优化抵消）相对较小。我们将在后面部分看到 backpressure 对于 Mono的意义。

## 一个类似于(但不等于) RxJava 的API

ReactiveX 和 RxJava 运算符的词汇表由于历史原因可能具有混淆的名称。Reactor 旨在有一个更紧凑的API(特殊场景有些背离了，例如为了选择更好的名字）。总体而言，两个 API 看起来是很相似。实际上，RxJava 2 中的最新迭代实际上也从Reactor中借用了一些词汇，这是两个项目之间持续密切合作的一个提示。一些操作符和概念首先出现在一个库中，但通常最终出现在两个库中。

例如，Flux 具有相同的熟悉的 just 工厂方法（虽然只有两个 just 变体：一个元素(T data)和一个可变参类型（T... data））。但是from，已经被几种明显的变体取代，最显着的变体 fromIterable。Flux 也有常见的运算符：map，merge，concat，flatMap，take...等等。

Reactor 也回避了 RxJava 的一些运算符名称，比如，令人困惑的 amb 运算符，它已被更适当命名 firstEmitting 给替换了。此外，为了在API上更加一致，toList 已经重命名 collectList。事实上，所有 collectXXX 运算符是聚合数据项为一个特定类型的集合的操作(以 Mono类型包装的，例如：Mono<List<T>>)，而保留 toXXX 方法是为了（在我们现在所讨论的 Reative 世界之外）类型转换，比如：toFuture()。

Reactor可以更精简的另一个方式是，在类实例化和资源使用方面。[融合]：Reactor能够合并多个顺序的运算符(比如，执行两次 concatWith 运算符)为一次使用，仅实例化运算符内部类一次（宏融合）。这过程包括一些基于数据源的优化可有助于抵消 Mono 实现 Publisher 的消耗。它还能够在几个兼容的运算符之间共享资源(比如，内部队列)，叫做（微融合）。这些功能使Reactor成为第四代反应库。但这是一个未来的文章的主题。

让我们仔细看看几个 Reactor 运算符。

### 一些运算符示例

（本节包含代码片段，我们建议您尝试进行进一步实验，为此，您应该打开您选择的IDE并创建一个以Reactor为依赖的测试项目。）

要在Maven中这样做，请将以下内容添加到pom.xml的dependencies部分：

```
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-core</artifactId>
    <version>3.0.3.RELEASE</version>
</dependency>
```

要在Gradle中做同样的事情，编辑依赖项部分以添加反应器，类似于：

```
dependencies {
    compile "io.projectreactor:reactor-core:3.0.3.RELEASE"
}
```

类似于如何创建您的第一个RxJava Observable ，您可以创建一个Flux使用just(T…)和fromIterable(Iterable<T>)Reactor工厂方法。请记住，对于一个给定的List，使用 just 方法时，将发出(emit)该列表作为一个整体，而fromIterable 将发出该迭代列表种每个元素：

```
public class ReactorSnippets {
  private static List<String> words = Arrays.asList(
        "the",
        "quick",
        "brown",
        "fox",
        "jumped",
        "over",
        "the",
        "lazy",
        "dog"
        );

  @Test
  public void simpleCreation() {
     Flux<String> fewWords = Flux.just("Hello", "World");
     Flux<String> manyWords = Flux.fromIterable(words);

     fewWords.subscribe(System.out::println);
     System.out.println();
     manyWords.subscribe(System.out::println);
  }
}
```

打印:

```
Hello
World

the
quick
brown
fox
jumped
over
the
lazy
dog
```

为了输出“fox”中的每个字母，我们还需要 flatMap（如我们通过实例RxJava所做的那样），但在 Reactor 中，我们用fromArray，而不是from。然后，我们要过滤掉重复的字母，并使用 distinct 和对它们进行排序。最后，我们要为每个不同的字母输出一个索引，可以使用zipWith和range：

```
@Test
public void findingMissingLetter() {
  Flux<String> manyLetters = Flux
        .fromIterable(words)
        .flatMap(word -> Flux.fromArray(word.split("")))
        .distinct()
        .sort()
        .zipWith(Flux.range(1, Integer.MAX_VALUE),
              (string, count) -> String.format("%2d. %s", count, string));

  manyLetters.subscribe(System.out::println);
}
```
这有助于我们注意到和预期一样没有`s`：

```
1. a
2. b
...
18. r
19. t
20. u
...
25. z
```

一种修复方法是修改原始单词数组，但我们也可以手动添加“s”值到字母的 Flux 中，使用 concat/concatWith 和一个 Mono：

```
@Test
public void restoringMissingLetter() {
  Mono<String> missing = Mono.just("s");
  Flux<String> allLetters = Flux
        .fromIterable(words)
        .flatMap(word -> Flux.fromArray(word.split("")))
        .concatWith(missing)
        .distinct()
        .sort()
        .zipWith(Flux.range(1, Integer.MAX_VALUE),
              (string, count) -> String.format("%2d. %s", count, string));

  allLetters.subscribe(System.out::println);
}
```

添加缺少的s 操作，发生在我们过滤掉重复项和排序/计数字母之前：

```
1. a
2. b
...
18. r
19. s
20. t
...
26. z
```

上一篇文章指出了 Rx 词汇和 Streams API 之间的相似之处，事实上当数据容易从内存中获得时，Reactor 像 Java Streams 一样，以简单的推送模式工作（参见下面的 backpressure 部分）。更复杂和真正的异步代码片段不适用于在主线程中订阅的模式，主要是因为控制将返回到主线程，然后一旦订阅完成就退出应用程序。例如：

```
@Test
public void shortCircuit() {
  Flux<String> helloPauseWorld =
              Mono.just("Hello")
                  .concatWith(Mono.just("world")
                  .delaySubscriptionMillis(500));

  helloPauseWorld.subscribe(System.out::println);
}
```
此代码段打印“Hello”，但无法打印延迟的“world”，因为测试终止太早。在片段和测试中，你只需要编写一个这样的主类，你通常会想要恢复到阻塞行为。为此，您可以创建CountDownLatch,并在您的订阅者中(onError 和 onCompelete 中)执行 countDown 。但是那不是 reactive（而且错误时，你忘记了 countDown 呢？）

你可以解决这个问题的第二种方法是使用一个可以恢复到非 reactive 世界的运算符。具体来说，toIterable 并且 toStream 都将产生阻塞实例。所以让我们使用 toStream ,来说明我们的例子：

```
@Test
public void blocks() {
  Flux<String> helloPauseWorld =
    Mono.just("Hello")
        .concatWith(Mono.just("world")
                        .delaySubscriptionMillis(500));

  helloPauseWorld.toStream()
                 .forEach(System.out::println);
}
```

正如你所期望的，这打印“Hello”，然后短暂的暂停，然后打印“world”随后终止。

正如我们上面提到的，RxJava amb()运算符已经被重命名 firstEmitting（这更清楚地暗示了运算符的意图：emit 第一个 Flux）。在下面的示例中，我们创建一个 Mono 其开始前被延迟了 450ms ,以及一个 Flux 在每个值emit之前暂停400ms。当 firstEmitting() 他们时，由于Flux的第一个元素优先发出（相比mono），故flux得到使用：

```
@Test
public void firstEmitting() {
  Mono<String> a = Mono.just("oops I'm late")
                       .delaySubscriptionMillis(450);
  Flux<String> b = Flux.just("let's get", "the party", "started")
                       .delayMillis(400);

  Flux.firstEmitting(a, b)
      .toIterable()
      .forEach(System.out::println);
}
```

这打印句子的每个部分之间是都会有400ms的暂停。

此时你可能会想，如果你正在写一个 Flux 的测试，引入延迟4000ms而不是400ms呢？你不想在单元测试中等待4s！幸运的是，我们将在后面的章节中看到，Reactor 带有强大的测试工具可以解决这个问题。

## 基于 Java 8
Reactor 针对 Java 8 而不是老的Java 版本。这再次符合减少API目标：RxJava 是针对Java 6，那时还没有 java.util.function 包，所以类似 Function 或 Consumer 不能利用。相反，他们不得不像添加特定的类 Func1，Func2，Action0，Action1等。RxJava 2 这些类也仍然存在需要支持老的JDK。

Reactor API 也拥抱 Java 8才引入的类型. 多数的时间相关的运算符，表达“持续时间”时（例如timeout，interval，delay等等），使用了Java 8 Duration 类。

Java 8 StreamAPI 和 CompletableFuture 也可以很容易地转换为 Flux/Mono，反之亦然。我们是否应该经常讲转换 Stream 转为 Flux 吗？不是。添加Flux或Mono是可以忽略不计的成本时，他们包装IO或内存绑定操作等更昂贵的操作，但大部分时间的Stream并没有上述延迟，因此完全可以使用StreamAPI 。请注意，这些用例在使用 RxJava2 时，，我们将使用Observable，它不是 back-pressure 的，因此在订阅后就成为一个简单的push的用例。但是Reactor基于Java 8，Stream API对于大多数用例都足够表达。还要注意，即使你可以看到的是一些字面量（String,Double?）或简单的对象 Flux 和 Mono工厂，它们大多服务于在更高层次的流组合的目的。因此，通常你不会想将现有代码访问器（如“ long getCount()”）转换为 Reactive 编程中的“ Mono<Long> getCount()”。

---

### 著作权声明

`首次发布于此，转载请保留以上链接`

参考文章TODO;
