---
title: EventBus3简介
tags: android
abbrlink: c1d65822
date: 2018-03-06 22:46:31
---



# 简介

> EventBus is a publish/subscribe event bus for Android and Java.
>
> 若无特殊说明，英文注释引自官方源码或文档，后同

EventBus在android中可以用于组件间，组件和后台线程间的通信，他基于订阅、发布的机制，将事件的收/发解耦，开销小。

其流程如下（示意图来自官方github库）：

![EventBus-Publish-Subscribe](https://github.com/greenrobot/EventBus/raw/master/EventBus-Publish-Subscribe.png)



# 要点

## EventBus的三个要素

* Event

  事件，要传递的事件（一个类），EventBus传递该事件的对象，要传递的信息包含在该对象的属性中

  ```kotlin
  class Event{ }
  ```

  

* Subscriber

  订阅者，当事件发生时要对事件执行的操作。

  ````kotlin
  @Subscribe(threadMode = ThreadMode.MAIN)  fun fun1() {}
  ````

* Publisher

  发布者，在合适的时候发布事件

  ```kotlin
  EventBus.getDefault().post(Event())
  ```

  此外还需要注册、注销EventBus以便订阅者能够正常接收消息

  

## Subscriber的5个线程模式（ThreadMode）

* POSTING

  默认的，订阅者和发布者在同一个线程（Subscriber will be called directly in the same thread, which is posting the event）。因为有可能是在主线程，所以不要耗时操作，以免ANR。

* MAIN

  Android中，订阅者会在UI线程被唤起，同样不能耗时操作

  > If the posting thread is the main thread, subscriber methods will be called directly, blocking the posting thread. Otherwise the event is queued for delivery (non-blocking). 

* MAIN_ORDERED

  Android，订阅者会在UI线程被唤起，不能耗时操作。

  > Different from MAIN, the event will always be queued for delivery. This ensures that the post call is non-blocking.

* BACKGROUND

  Android中，如果post在主线程，那么订阅者在新开的后台线程，否则在post的线程被唤起。

  在如果不是在Android中，则都在后台线程唤起。

* ASYNC

  订阅者在新的子线程中运行，独立于UI线程和post线程

# 使用举例

## 普通事件

1. 定义事件

   ```kotlin
   class Event(s: String){
       var string : String? = null
       init {
           string = s
       }
   }
   ```

2. 注册

   ```kotlin
   EventBus.getDefault().register(context)
   ```

3. 设置订阅者

   ```kotlin
   @Subscribe(threadMode = ThreadMode.MAIN)
     fun fun1(event: Event) {
         Log.d("tag",event.string)
   }
   ```

4. 发布

   ```kotlin
   EventBus.getDefault().post(Event("hello"))
   ```

5. 注销

   ```
   EventBus.getDefault().unregister(context)
   ```



## 粘性事件

粘性事件即事件发生（`postSticky()`）之后，再订阅，也可对事件进行处理

发布（`postSticky()`）：

```kotlin
EventBus.getDefault().postSticky(Event("粘性事件"))
```

订阅者（`sticky = true`）：

```kotlin
@Subscribe(threadMode = ThreadMode.MAIN,sticky = true)//sticky默认为false，可以不写
fun fun1(event: Event) {
    Log.d("tag",event.string)
}
```

如上修改`发布`和`订阅者` 之后，当`postSticky()`之后，只有下次`注册事件`时，`订阅者`才会对事件进行反应。

如下代码就只会在点击事件之后，`订阅者`才会对`postSticky()`事件做反应

```kotlin
fun onClick(view: View){
    EventBus.getDefault().register(context)
}
```



# 参考资料

[greenrobot/EventBus: Event bus for Android and Java that simplifies communication between Activities, Fragments, Threads, Services, etc. Less code, better quality.](https://github.com/greenrobot/EventBus)

[Android事件总线（一）EventBus3.0用法全解析 - CSDN博客](http://blog.csdn.net/itachi85/article/details/52205464)



