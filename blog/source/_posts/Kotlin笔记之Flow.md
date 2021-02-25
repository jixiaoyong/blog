---
title: Kotlin笔记之Flow
tags: kotlin
date: 2021-2-25 12:20:12
---

# 前言

`Flow`是Kotlin协程库中的库，用于**异步返回多个值**，官方介绍是参考`RxJava`等响应式流实现的，但是“*拥有尽可能简单的设计， 对 Kotlin 以及挂起友好且遵从结构化并发*”。本文主要参考[Flow中文文档](https://www.kotlincn.net/docs/reference/coroutines/flow.html)，梳理了学习过程中的要点和理解，以便日后查验。

# 正文

对于异步返回多个值的需求，集合（如`List`等）只能一次性返回多个值，而序列（ [`Sequence`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/index.html) ）只支持阻塞代码，`Flow`则支持挂起函数异步返回多个值。

## 创建Flow

1. `flow{...}`

   ```kotlin
   fun simple(): Flow<Int> = flow<Int> {
       for (i in 1..3) {
           delay(100) // 假装我们异步等待了 100 毫秒，也可以用Thread.sleep()但是会阻塞当前线程
           emit(i) // 发射下一个值
       }
   }
   ```

2. `.asFlow()`

   ```kotlin
   fun simple(): Flow<Int> = (1..10).asFlow()
   ```

3. `flowOf{}`

   ```kotlin
   fun simple(): Flow<Int> = flowOf(1, 2, 3, 4, 5)
   ```

因为**流只会在被收集的时候才会被启动**（指执行类似`flow{...}`中的内容），所以上述`simple()`在被调用时会尽快返回且不等待，所以无需`suspend`修饰。

## 流的收集/末端流操作符

* `collect{...}` 收集`emit`发送的值

  配合`onEach{}`可以将`collect`中执行的代码放到`onEach`中。

* `collectLatest{...}` 收集`emit`发送的值，但每次新的`emit`到来时，取消之前的收集器，创建新的收集器（用新的值执行`{...}`中的代码）

* `launchIn` 指定在单独的协程中启动流的收集，这样就可以立即继续进一步执行代码，不会挂起后面的协程代码。

* `single()` 只接受flow发送的一个值，0个或多个都会报错

* `first{...}` 查找符合条件的第一个值

* `reduce()` 求和

* `fold(initial,{...})` 在初始值`initial`的基础上求和

* `toList`、`toSet`

## 过渡流操作符

过渡操作符应用于上游流，并返回下游流。就像流一样。这类操作符本身不是挂起函数。它运行的速度很快，返回新的转换流的定义。

* `map{}`
* `filter{}`
* `take(n)` 限长操作符，只取前n个发射的值

## 流上下文

流默认运行在收集器提供的上下文中，但是可以通过`flowOn `更改：

```kotlin
fun simple(): Flow<Int> = flow {
    ...
}.flowOn(Dispatchers.Default) // 在流构建器中改变消耗 CPU 代码上下文的正确方式
```

## 展平流

将嵌套有`Flow`的`Flow`（如`Flow<Flow<String>>`）**展平**为单个流（如`Flow<String>`）。

* `flatMapConcat` 将收集到的流交给`{...}`处理后，等待内部流处理完毕后，再去请求下一个流

* `flatMapMerge` 先顺序收集所有流，再同时收集结果流

* `flatMapLatest{...}` 类似于`collectLatest{...}`，在新柳发出的时候，立即取消`{...}`中所有的代码

  

* `flattenConcat` 依次展平流

* `flattenMerge{...}` 并发拼接，先执行`{...}`中的方法，再执行`collect`等方法，顺序会乱。

## 异常处理

* `try/catch`

  ```kotlin
  fun simple(): Flow<Int> = flow {
      for (i in 1..3) {
          println("Emitting $i")
          emit(i) // 发射下一个值
      }
  }
  
  fun main() = runBlocking<Unit> {
      try {
          simple().collect { value ->         
              println(value)
              check(value <= 1) { "Collected $value" }
          }
      } catch (e: Throwable) {
          println("Caught $e")
      } 
  }    
  ```

* `catch()`

  **透明捕获**：只捕获上游异常，其之后的异常不会被处理。

  ```kotlin
  simple()
      .catch { e -> emit("Caught $e") } // 发射一个异常
      .collect { 
          value -> println(value) //此处如有异常，不会被catch捕获
               }
  ```

  **声明式捕获**：将`collect`的代码移动到`onEach`中，将其放到`catch`之前，从而使其被`catch`捕获。

  ```kotlin
  simple()
      .onEach { value ->
          check(value <= 1) { "Collected $value" } //此处异常会被catch捕获             
          println(value) 
      }
      .catch { e -> println("Caught $e") }
      .collect()
  ```

## 流取消

* `flow { ... }` 创建的流的繁忙循环默认可以取消
* 其他流如果需要取消，可以添加 `.onEach { currentCoroutineContext().ensureActive() }` 或者`.cancellable()`

## 流完成

* 命令式

  ```kotlin
   try {
          simple().collect { value -> println(value) }
      } finally {
          println("Done") //监听流完成
      }
  ```

* 声明式

  ```kotlin
  simple()
      .onCompletion { println("Done") } //监听流完成，在collect执行结束后才执行
      .collect { value -> println(value) }
  ```

  `onCompletion`的可空参数 `Throwable` 可以用于确定流收集是正常完成（为`null`）还是有异常发生。他不会处理异常。

## 其余操作

* `buffer()` 缓冲发射项，收集完成后再传给下一步

* `conflate()` 合并发射项，会丢弃来不及处理的中间值，只获取并处理最新的值

* `zip()` 合并两个流的值,两个流中的值一一对应

  例如`(1,2,3) 3s发射一次,(a,b,c) 4s发射一次`直接拼接，合并之后为 `(1a,2b,3c)`

* `combine()` 结合两个流的值，任意一个流中的值发生变化都会触发执行计算

  例如`(1,2,3) 3s发射一次,(a,b,c) 4s发射一次`直接拼接，合并之后为 `(1a,2a,2b,3b,3c)`



# 参考文献

[Kotlin Flow 中文文档](https://www.kotlincn.net/docs/reference/coroutines/flow.html)