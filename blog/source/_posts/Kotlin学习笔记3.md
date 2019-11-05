---
title: Kotlin学习笔记3
tags: kotlin
date: 2019-11-05 18:46:19

---

> 本文为笔记性质，尚未成文，待整理

# 异步流

1. lazy方式创建一个序列，只有在访问的时候才生产对应的项目
```kotlin
fun foo(): Sequence<Int> = sequence{
    for (i in 1..3) {
        Thread.sleep(1000)//会阻塞调用线程
        yield(i)//生产一个项目
    }
}
```

2. 使用Flow流在不阻塞主线程的情况下，延迟生产多个值并返回

    当流在一个可取消的挂起函数（例如 [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html)）中挂起的时候取消，否则不能取消。 
```kotlin
//flow 构建器中的代码直到流被收集的时候才运行，并且每次collect都会被启动
fun foo(): Flow<Int> = flow { // 流构建器
    for (i in 1..3) {
        delay(100) // 假装我们在这里做了一些有用的事情，这里可以被取消
        emit(i) // 发送下一个值
    }
}

fun main() = runBlocking<Unit> {
    // 启动并发的协程以验证主线程并未阻塞
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // 收集这个流
    foo().collect { value -> println(value) } 
}
```

[flowOf](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-of.html) 构建器定义了一个发射固定值集的流。

使用 `.asFlow()` 扩展函数，可以将各种集合与序列转换为流。

 可以使用操作符转换流，就像使用集合与序列一样。 过渡操作符应用于上游流，并返回下游流。 这些操作符也是冷操作符，就像流一样。这类操作符（`map`、`fliter`...）本身不是挂起函数，但是可以调用挂起函数`suspend`。它运行的速度很快，返回新的转换流的定义。 

 在流转换操作符中，最通用的一种称为 [transform](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/transform.html)。它可以用来模仿简单的转换，例如 [map](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html) 与 [filter](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/filter.html)，以及实施更复杂的转换。 使用 `transform` 操作符，我们可以 [发射](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) 任意值任意次 

 限长过渡操作符（例如 [take](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/take.html)）在流触及相应限制的时候会将它的执行取消。 

```kotlin
fun numbers(): Flow<Int> = flow {
    try {                          
        emit(1)
        emit(2) 
        println("This line will not execute")
        emit(3)    
    } finally {
        println("Finally in numbers")
    }
}

fun main() = runBlocking<Unit> {
    numbers() 
        .take(2) // 只获取前两个
        .collect { value -> println(value) }
}  
```

流构造器中的协程上下文默认和collect的协程上下文一致，如果强行转换上下文会出错。

而使用`flowOn()`则可以指定流创建的协程上下文：

```kotlin
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // 假装我们以消耗 CPU 的方式进行计算
        log("Emitting $i")
        emit(i) // 发射下一个值
    }
}.flowOn(Dispatchers.Default) // 在流构建器中改变消耗 CPU 代码上下文的正确方式
```

如果flow的生产和收集很消耗时间时，可以用[`buffer()`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/buffer.html )函数将`buffer()`之前的代码在一个单独的协程运行，collect则在调用协程运行，这样将flow的构建、收集由串行转化为并行可以节约时间（如果构建运行的快，则会挂起直到collect赶上来）。

>  It will use two coroutines for execution of the code. A coroutine `Q` that calls this code is going to execute `collect`, and the code before `buffer` will be executed in a separate new coroutine `P` concurrently with `Q`

```kotlin
 foo()
        .buffer() // buffer emissions, don't wait
        .collect { value -> 
                delay(300) // pretend we are processing it for 300 ms
            println(value) 
        } 
```

# 合并conflate

 [conflate](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/conflate.html) operator can be used to skip intermediate values when a collector is too slow to process them. 

当collect比构建慢的时候，就只会请求最新的值，而跳过中间生产的这些值。

比如，构建器生产了1，2，... ,100这些数，而collect读取的慢，第一次读的时候是1，等处理完再读取的时候构建器生产的是5，那么collect就读取5，中间的2，3，4都会被丢弃。

 Conflation is one way to speed up processing when both the emitter and collector are slow 。 The other way is to cancel a slow collector and restart it every time a new value is emitted.  

`collectLatest`可以保证每次都获取最新的值，如果collect比生产慢，那么当新的值生产出来时，collect会被取消，并且去处理最新的值。

```kotlin
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        foo()
            .collectLatest { value -> // cancel & restart on the latest value
                    println("Collecting $value") 
                delay(300) // pretend we are processing it for 300 ms
                println("Done $value") 
            } 
    }   
    println("Collected in $time ms")
}
//output
Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 694 ms
```

# 组合多个流

`zip`将两个流“压缩”为一个流：

```kotlin
    val number = (1..3).asFlow()
    val strs = flowOf("one", "two", "three")
    number.zip(strs) { a, b ->
        "$a -> $b"
    }.collect{
        println("$it")
    }
// output
1 -> one
2 -> two
3 -> three
```

 当 flow 表示变量或操作的最新值时(参见关于合并的相关章节) ，可能需要执行依赖于相应流的最新值的计算，并在任何上游流发出值时重新计算它。 相应的操作符族称为联合操作符 [`combine`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/combine.html)。即每个构建值发生变化时都会触发collect。

```kotlin
val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms          
val startTime = System.currentTimeMillis() // remember the start time 
nums.combine(strs) { a, b -> "$a -> $b" } // compose a single string with "combine"
    .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
} 
//output    
1 -> one at 452 ms from start
2 -> one at 651 ms from start
2 -> two at 854 ms from start
3 -> two at 952 ms from start
3 -> three at 1256 ms from start
```



[`flatMapConcat`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-concat.html ) 可以将flow的内容“抹平”（即假设原先为`Array<Array<Int>>`，则押平后为：`Array<Int>`）。串行执行，即先执行代码块，然后对其`flatMapConcat`，然后collect，之后再执行下一轮的Flow项目。

[` flatMapMerge `](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-merge.html ) 按顺序调用它的代码块 ，但是同时收集结果流，这相当于首先执行一个顺序映射 ，然后对结果调用 flattonmerge。并行执行，先依次对Flow项目调用代码块，然后哪个值先出来，就先对其调用 ` flatMapMerge `，然后collect。

[`flatMapLatest`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-latest.html ) 类似 [`collectLatest`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect-latest.html) ，每次新值出来就会取消还没有处理结束的旧流的操作。

# 流异常

流的异常有如下捕获方式：

1.  [`try/catch`](https://kotlinlang.org/docs/reference/exceptions.html) block 

2. 透明捕获  [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html)  ，只会捕获发送在他之前的异常

3. #### 声明式捕获 将collect的主要逻辑放到onEach中，保证onEach在catch之前

```kotlin
foo()
    .onEach { value ->
            check(value <= 1) { "Collected value" }                 
        println(value) 
    }
    .catch { e -> println("Caught e") }
    .collect()
```

# 流完成

1.   `try`/`finally` 

2.  `onCompletion()`

   ```kotlin
   foo()
       .onCompletion { println("Done") }
       .collect { value -> println(value) }
   ```

   而且他还可以判断是否是异常退出。但是只是判断，并不会处理、拦截异常，并且只能处理上游的异常。

   ```kotlin
       foo()
           .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
           .catch { cause -> println("Caught exception") }
           .collect { value -> println(value) }
   ```



# launchIn(this) 与collect

`collect`后的代码只有在collect执行完后才能执行，而`launchIn`可以指定其在单独的协程程序中启动流的集合，从而不会阻塞当前协程。



# Channel 

 [Channel](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html)  类似于BlockingQueue。但他的操作是挂起的。  通道提供了一种在流中传输值的方法 

`send` 发送 缓存区已满或不存在时调用方会被挂起

`channel.receive()` 接收

`channel.close()` 关闭通道，表示没有更多的元素进入通道

[`CoroutineScope.produce`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html )  Launches new coroutine to produce a stream of values by sending them to a channel and returns a reference to the coroutine as a [ReceiveChannel](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/index.html). This resulting object can be used to [receive](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html) elements produced by this coroutine. 在新的协程中生产并返回了一个`ReceiveChannel<T>`对象。

[`ReceiveChannel<E>.consumeEach`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/consume-each.html ) 遍历ReceiveChannel的item执行指定action，并在块执行完毕后消耗掉这个ReceiveChannel（调用cancel()）。

 [ReceiveChannel](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/index.html).[cancel()](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/cancel.html) 取消接收来自这个通道的剩余元素，关闭通道并从中删除所有缓存的元素。

# 管道

管道指：1.一个协程在流中开始生产无穷多个元素 2.另一个或多个协程消费这些流

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    val numbers = produceNumbers() // 从 1 开始生产整数
    val squares = square(numbers) // 对整数做平方
    for (i in 1..5) println(squares.receive()) // 打印前 5 个数字
    println("Done!") // 我们的操作已经结束了
    coroutineContext.cancelChildren() // 取消子协程
}

fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // 从 1 开始的无限的整数流
}

fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

没有缓冲的通道： 如果发送先被调用，则它将被挂起直到接收被调用， 如果接收先被调用，它将被挂起直到发送被调用。 

带缓冲的通道： [Channel()](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel.html) 工厂函数与 [produce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html) 建造器通过一个可选的参数 `capacity` 来指定 *缓冲区大小* 。缓冲允许发送者在被挂起前发送多个元素， 就像 `BlockingQueue` 有指定的容量一样，当缓冲区被占满的时候将会引起阻塞。 

```kotlin
val channel = Channel<Int>(4) // 启动带缓冲的通道
```



 发送和接收操作是 *公平的* 并且尊重调用它们的多个协程。它们遵守先进先出原则。



 计时器通道是一种特别的会合通道，每次经过特定的延迟都会从该通道进行消费并产生 `Unit` 。如果在间隔还没到的时候调用`tickerChannel.receive()`则会返回`null`。产生的间隔由[`TickerModel`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-ticker-mode/index.html )控制

```kotlin
val tickerChannel =  ticker(delayMillis = 100, initialDelayMillis = 0,mode = TickerMode.FIXED_PERIOD) //创建计时器通道 mode默认为TickerMode.FIXED_PERIOD
 tickerChannel.receive() //第一次调用立马返回Unit
```



`coroutineContext.cancelChildren()` // 取消所有的子协程来让主协程结束

# 异常

## 异常的传播

 协程构建器有两种风格：自动的传播异常（[launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) 以及 [actor](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html)） 或者将它们暴露给用户（[async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) 以及 [produce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html)）。 前者对待异常是不处理的，类似于 Java 的 `Thread.uncaughtExceptionHandler`， 而后者依赖用户来最终消耗异常，比如说，通过 [await](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html) 或 [receive](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html)  

[CoroutineExceptionHandler](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html) 仅在预计不会由用户处理的异常上调用， 所以在 [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) 构建器中注册它没有任何效果。

 协程内部使用 `CancellationException` 来进行取消，这个异常会被所有的处理者忽略，所以那些可以被 `catch` 代码块捕获的异常仅仅应该被用来作为额外调试信息的资源。 

 如果协程遇到除 `CancellationException` 以外的异常，它将取消具有该异常的父协程。 这种行为不能被覆盖，且它被用来提供一个稳定的协程层次结构来进行[结构化并发](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/composing-suspending-functions.md#structured-concurrency-with-async)而无需依赖 [CoroutineExceptionHandler](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html) 的实现。 且当所有的子协程被终止的时候，原本的异常被父协程所处理。 

应该将[CoroutineExceptionHandler](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html) 总是被设置在由 [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) 启动的协程中。将异常处理者设置在 [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) 主作用域内启动的协程中是没有意义的，尽管子协程已经设置了异常处理者， 但是主协程也总是会被取消的。 

异常被抛出后，所有同级的子协程都会被关闭，然后异常传递给父协程，直到异常被处理。

 一个协程的多个子协程抛出异常将会发生什么？ 通常的规则是“第一个异常赢得了胜利“。

## 监督

普通的取消 是一种双向机制，在协程的整个层次结构之间传播。 

 SupervisorJob  它类似于常规的 [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html)， 但他的取消只会向下传播 

```
val supervisor = SupervisorJob()with(CoroutineScope(coroutineContext + supervisor)) {

}
supervisor取消的话，会取消掉所有子协程
```

#### 监督作业

[SupervisorJob](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html) 可以被用于这些目的。它类似于常规的 [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html)，唯一的不同是：SupervisorJob 的取消只会向下传播。这是非常容易从示例中观察到的：

#### 监督作用域

对于*作用域*的并发，[supervisorScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html) 可以被用来替代 [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) 来实现相同的目的。它只会单向的传播并且当子作业自身执行失败的时候将它们全部取消。它也会在所有的子作业执行结束前等待， 就像 [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) 所做的那样。

#### 监督协程中的异常

常规的作业和监督作业之间的另一个重要区别是异常处理。 每一个子作业应该通过异常处理机制处理自身的异常。 这种差异来自于子作业的执行失败不会传播给它的父作业的事实。

## 协程的线程安全

1. 使用线程安全的数据结构

   

   ```kotlin
   var counter = AtomicInteger()
    withContext(Dispatchers.Default) {
           counter.incrementAndGet()
       }
   ```

   

2. 以细粒度限制线程

3. 以粗粒度限制线程

   2、3都是保证将对共享变量的操作限制在同一个线程中，从而保证线程安全。

4. 互斥

   类似于线程的锁，协程的 [Mutex](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/index.html) 的lock和unlock方法可以保证同一时间只有一个协程访问指定代码。Mutex不会阻塞线程。

5. Actors

    一个 [actor](https://en.wikipedia.org/wiki/Actor_model) 是由协程、 被限制并封装到该协程中的状态以及一个与其它协程通信的 *通道* 组合而成的一个实体。 

   ```kotlin
   // 这个函数启动一个新的计数器 actor
   fun CoroutineScope.counterActor() = actor<CounterMsg> {
       var counter = 0 // actor 状态
       for (msg in channel) { // 即将到来消息的迭代器
           when (msg) {
               is IncCounter -> counter++
               is GetCounter -> msg.response.complete(counter)
           }
       }
   }
   //1.要递增状态时
   counter.send(IncCounter)
   //2.要获取当前状态时
   // 发送一条消息以用来从一个 actor 中获取计数值
   val response = CompletableDeferred<Int>()
   counter.send(GetCounter(response))
   println("Counter = ${response.await()}")
   ```

     [CompletableDeferred](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/index.html) 通信原语表示未来可知（可传达）的单个值 。

   在使用时，由于actor 是一个协程，[` SendChannel .send()`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html ) 方法会在通道缓存满的时候挂起调用方，从而最终保证了`counter++`方法是依次执行的，不会产生并发问题。

   actor 在高负载下比锁更有效，因为在这种情况下它总是有工作要做，而且根本不需要切换到不同的上下文。 

>  注意，[actor](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html) 协程构建器是一个双重的 [produce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html) 协程构建器。一个 actor 与它接收消息的通道相关联，而一个 producer 与它发送元素的通道相关联。 



# Select表达式

  [select](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html) 表达式允许我们使用其 [onReceive](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/on-receive.html) 子句 *同时* 从两个生产者接收数据：

```kotlin
suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> 意味着该 select 表达式不返回任何结果
        fizz.onReceive { value ->  // 这是第一个 select 子句
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // 这是第二个 select 子句
            println("buzz -> '$value'")
        }
    }
}
```

 [onReceiveOrNull](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/on-receive-or-null.html)  可以允许为空，这样可以在关闭通道时执行特定操作 

 [onSend](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/on-send.html) 子句 发送消息

```kotlin
fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // 生产从 1 到 10 的 10 个数值
        delay(100) // 延迟 100 毫秒
        select<Unit> {
            onSend(num) {} // 发送到主通道
            side.onSend(num) {} // 或者发送到 side 通道
        }
    }
}
```

 Select延迟值可以使用 [onAwait](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/on-await.html) 子句查询 





<iframe width='853' height='480' src='https://embed.coggle.it/diagram/Xb_CZoumpCamgUAj/1235701ea5f157867e045f0f5ac28f886effdb5fa1e27b72afa5266e6e3e8891' frameborder='0' allowfullscreen></iframe>






# 参考资料

[Kotlin的独门秘籍Reified实化类型参数(下篇)](https://blog.csdn.net/u013064109/article/details/83507076)

[Kotlin 协程 中文官网--异步流](https://www.kotlincn.net/docs/reference/coroutines/flow.html)
