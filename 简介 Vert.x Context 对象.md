原文：[An Introduction to the Vert.x Context Object](http://www.millross-consultants.com/vertx_context_object.html)

原作者 [millross](http://github.com/millross)，更新于2017年10月29日

Vert.x 的 Context 对象在保证 verticles 的线程安全方面有着重要作用。大部分时候，vert.x 开发者不需要直接与 Context 打交道。本文将简要介绍vert.x Context 类的重要性，以及什么时候你可能会需要直接使用它。

## Vert.x 中的 Context 对象 - 简要介绍

### 介绍

最近我一直在思考，如何构建一个异步版本的 [pac4j](http://www.pac4j.org/) 库，并且想让 [vertx-pac4j](https://github.com/pac4j/vertx-pac4j) 作为 pac4j 的默认异步实现方式。

我本来希望 pac4j 的异步版本不需要和某个特定的异步/非阻塞框架有强耦合。所以一开始我决定使用 [CompletableFuture](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) 来暴露 API，用它来把结果包装在 future 对象里。后来我选用了 [vert.x](http://vertx.io/) 框架来测试API，这让我必须好好学习 vert.x [Context](http://vertx.io/docs/apidocs/io/vertx/core/Context.html) 。

##### 这篇文章的内容基于 vert.x 3.3.3，在vert.x 的后续版本中有可能将不再正确。

### 介绍 Context 类

当我们执行 vert.x 的 [Handler](http://vertx.io/docs/apidocs/io/vertx/core/Handler.html) ，或者调用 verticle 的 start 方法和内部方法的时候，这些操作都将与一个特定的 context 对象关联着。一般来说，一个 context 对象就是一个循环事件的上下文。所以必须和一个特定的 event loop 线程绑定（下面会提到一些异常情况）。Contexts 还是可传播的。当某段代码已经运行在一个context 中，若此时这段代码写了一个 handler，那这个 handler 会由相同的 context 来执行。这意味着，比如一个 verticle 对象的 start 方法中设置了一些event bus 的 handler（不论有多少handler），那么这些 handler 和这个 verticle 的 start 方法会在同一个 context 中被执行（所以说，对这个verticle对象来说，所有的 handler 共享一个 context）。

图一显示了 non-worker 的 verticles，contexts 和 eventloop 线程之间的关系。



![Vertx Context/Thread/Verticle Relationships](http://vertx.io/assets/blog/vertx3-intro-to-context-object/VertxContextRelationships.png)

注意，每个 verticle 对象只会在 start 方法的时候，创建一个 context，这个 verticle 对象的所有 handler 都绑定在这个 context 上。并且，每个 context 只绑定在一个 event loop 线程上。可是，一个 event loop 线程却可以和多个 context 相绑定。

### 什么时候contexts 不会传递？

当调用一个 verticle 对象的 start 方法时，会创建一个新的 context 对象。如果一个 verticle 在部署时，设置了拥有4个对象，那每个对象的 start 方法都将创建新的 context 对象。这样做是合理的，因为当有多个 eventloop 线程可用时，我们肯定不会想把所有的 non-worker 的 verticles 仅仅只与一个 eventloop 线程绑定在一起。

### 保证线程安全

上面提到了很多关于 contexts 的可传播性的概念。最重要的一点是，给定了 eventloop 线程的 verticle，它所有的 handler 都会在同一个 context 中执行（即执行 start 方法的context），而且这些 handler 都会在同一个 eventloop 中执行。所以只要某个状态值只能被一个 verticle 对象访问，那么这个状态值永远只能被一个线程访问。这样就保证了 vert.x 的线程安全，而不再需要手动进行同步。

### 异常处理

在 event loop 线程处理 context 的时候，每个 context 都有自己的异常处理机制。

#### 为什么有时候你可能不想要默认的异常处理机制？

比如，你可能运行一些 verticles 来监控另一些 verticles 的状态：假如被监控的 verticles 出现问题了，卸载或者重启它们。这是一个常见的 actor- or microservices - style 架构。所以，一个可选方案是，当被监控的 verticle 出现不可恢复的错误时，它可以通过 eventbus 发消息给 监控它的 verticle，这样，监控的 verticle 可以选择卸载或者重新部署这些被监控的 verticle（也可以选择在连续几次失败之后，放弃这些 verticle，或者把这些错误进一步上报给监控者的上级）。

### 脱离当前 context 和返回到特定的 context

有些时候，你的程序可能不在 context 中执行，但当程序执行结束时，又可能需要操作 vert.x context。比如下面这些场景：

#### 在另一个线程中执行代码

在刚接触 vert.x 的时候，你有可能会使用某个没有与 vert.x 融合的异步驱动库。这个库的代码没有运行在 eventloop 线程中，但你可能会需要使用这个库运行的结果，来更新 verticle 中的信息。这时，如果你不在正确的 context 中更新 verticle 的信息，你将无法保证线程安全。这种情况下，你就必须在正确的 eventloop 线程中进行操作。

#### 使用 Java 8 异步APIS

像 CompletableFuture 这样的APIs，是没有运行在 context 中的。比如下面的测试代码，我在 vert.x 的 event loop 线程中创建了一个已经结束的 future，然后我通过 thenRun 方法 执行接下来的操作:-

```java
@RunWith(VertxUnitRunner.class)
public class ImmediateCompletionTest {
    @Rule
    public final RunTestOnContext rule = new RunTestOnContext();

    @Test
    public void testImmediateCompletion(TestContext context) {

        final Async async = context.async();
        final Vertx vertx = rule.vertx();
        final CompletableFuture<Integer> toComplete = new CompletableFuture<>();
        // delay future completion by 500 ms
        final String threadName = Thread.currentThread().getName();
        toComplete.complete(100);
        toComplete.thenRun(() -> {
            assertThat(Thread.currentThread().getName(), is(threadName));
            async.complete();
        });
    }
}
```

有些人可能会天真的以为，这块代码会自动运行在对应的 context 上，因为它在 future 结束的时候，并没有离开 eventloop 线程，而且运行的结果也表明它确实是在正确的线程上。但是，它并不会运行在正确的 context 里面。这就意味着，比如说，它不会调用 context 的异常处理机制。

#### 返回到正确的 context

庆幸的是，即便我们脱离了的当前 context，也可以很方便的返回。比如说之前我们在 thenRun 方法中的代码，可以通过使用 Vertx.currentContext() 或者 vertx.getOrCreateContext() 来获得当前在 eventloop 中运行的 context 对象，然后我们可以通过调用 Context::runOnContext 方法来执行这段代码，类似于：

```java
final Context currentContext = vertx.getOrCreateContext();
toComplete.thenRun(() -> {
        currentContext.runOnContext(v -> {
        assertThat(Thread.currentThread().getName(), is(threadName));
        async.complete();
    }
});
```

### 扩展阅读

Vert.x 团队也提供了一篇优秀的关于 Vert.x eventloop 的博客，其中就有 context 的详细介绍  ：[Github](https://github.com/vietj/vertx-materials/blob/master/src/main/asciidoc/Demystifying_the_event_loop.adoc) 。

### 致谢

非常感谢 Vert.x 核心小组发表在 github上的关于 eventloop 的详细介绍，同时也十分感谢 [Alexander Lehmann](https://twitter.com/alexlehm?lang=en) 耐心的回答我在 [Vert.x google group](https://groups.google.com/forum/#!forum/vertx) 提的那些弱智问题。
