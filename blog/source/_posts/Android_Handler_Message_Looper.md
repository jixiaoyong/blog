---
title: Android 异步消息机制 Handler、Message、Looper
tags: android
abbrlink: 5962504e
date: 2018-04-01 20:08:08
---



> 此文为鸿洋博客阅读笔记，配合原文食用口味更佳。
>
> [Android 异步消息处理机制 让你深入理解 Looper、Handler、Message三者关系 - CSDN博客](https://blog.csdn.net/lmj623565791/article/details/38377229)



# 导图

![20200429160005](https://jixiaoyong.github.io/images/20200429160005.png)


# 过程分析

## Looper

### **Looper.perpare()**

`Looper.perpare()`方法创建`Looper对象`（同时创建`MessageQueue对象`），并与当前线程关联保存在`sThreadLocal`中。

```java
public static final void prepare() {  
        if (sThreadLocal.get() != null) {  
            throw new RuntimeException("Only one Looper may be created per thread");  
        }  
        sThreadLocal.set(new Looper(true));  
}  

private Looper(boolean quitAllowed) {  
        mQueue = new MessageQueue(quitAllowed);  
        mRun = true;  
        mThread = Thread.currentThread();  
}  
```

### **Looper.loop()**

`Looper.loop()`方法获取保存的`Looper对象`并由此获取到`MessageQueue对象`。

通过`for循环`，不停的通过`mQueue`获取到`msg`，并调用`msg.target.dispatchMessage(msg)`执行msg对应的处理方法。

最后通过`msg.recycle()`回收使用完的msg。

```java
public static void loop() {  
    final Looper me = myLooper();  
    ...
    final MessageQueue queue = me.mQueue;  
    ...
    for (;;) {  
        Message msg = queue.next(); // might block  
        ...
        msg.target.dispatchMessage(msg); 
        ...
        msg.recycle();  
    }
  }
```

### **Looper.myLooper()**

`myLooper()`内部调用`sThreadLocal`获取已有的`Looper对象`

```
public static Looper myLooper() {
    return sThreadLocal.get();
}
```



**Android的Activity默认在UI线程调用了Looper的`prepare()`和`loop()`方法**



## Handler

### handler.sendMessage()

Handler构造方法会获取到`mLooper`和`mQueue`以及`mCallback`

```java
 mLooper = Looper.myLooper();  
 mQueue = mLooper.mQueue;  
 mCallback = callback;  // Handler()中此值为null
```

`sendMessage()`方法最终会调用`sendMessageAtTime()`方法,在其内部调用`enqueueMessage()`方法，将handler赋予msg.target，并将msg压入mQueue中

```java
//enqueueMessage方法
msg.target = this;
queue.enqueueMessage(msg, uptimeMillis);//将handler发送的msg压入到当前线程的Looper持有的MessageQueue中
```

### handler.dispatchMessage()

Handler的`dispatchMessage()`方法会在`Looper.loop()`中被调用

```java
public void dispatchMessage(Message msg) {  
        if (msg.callback != null) {  //msg自带的回调方法
            handleCallback(msg);  
        } else {  
            if (mCallback != null) {  //handler指定的回调方法
                if (mCallback.handleMessage(msg)) {  
                    return;  
                }  
            }  
            handleMessage(msg);  //handler的handleMessage()方法
        }  
    }  
```

其中执行顺序是：`msg.callback` > `mCallback` > `handleMessage()`

### handler.post()

`handler.post(new Runnable())`调用了`getPostMessage(r)`方法将r赋予msg.callback

```java
public final boolean post(Runnable r)  {  
      return  sendMessageDelayed(getPostMessage(r), 0);  
   }  
   
private static Message getPostMessage(Runnable r) {  
      Message m = Message.obtain();  
      m.callback = r;  
      return m;  
  }  
```

最后也是在`sendMessageDelayed方法`中调用`sendMessageAtTime()方法`将msg压入MessageQueue中

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)  {
    ...
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);  
}  
```

## msg的获取

* `Message.obtain();` 复用MessageMessage池中已有的对象，避免出现分配内存 **推荐** 
* `new Message();`

# 总结

**Looper**在**`perfare()`**方法中创建`Looper及MessageQueue对象`并保存在`sThreadLocal`中，

在**`loop()`**方法中通过`myLooper()`从`sThreadLocal`中取出`mLooper`，并由此获得`mQueue`，在for循环中通过`mQueue.next()`获取`msg`，用`msg.target.dispatchMessage()`方法回调`handler中的msg处理方法`。

**Handler**在**`构造函数`**中通过`Looper.myLooper()`获取到`当前线程的Looper和MessageQueue`；

**`sendMessage()`**方法最终通过`sendMessageAtTime()`调用`enqueueMessage()`方法`将msg压入到MessageQueue`中。

至此将*Looper和Handler通过MessageQueue联系在一起*，并共同参与处理Message。

此外**`handler.post(runnable)`**也是通过在**`post()`**内部调用`getPostMessage()`方法将`runnable赋予msg.callback`，并在`post()`中通过`sendMessageDelayed()`方法调用`sendMessageAtTime()方法`将`msg压入MessageQueue`中


