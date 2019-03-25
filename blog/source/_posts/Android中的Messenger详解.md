---
title: Android中的Messenger源码详解
date: 2019-3-21 10:26:32
---


# 前言

Messenger是Android中用于IPC的方式之一，使用Handler发送有序消息队列，底层是通过AIDL调用Binder实现。

Messenger只用于服务端和客户端串行的传递消息，如果大量并发或者跨进程调用服务端的方法，就需要考虑AIDL而非Messenger。

Messenger的使用可以参考[这篇文章](https://jixiaoyong.github.io/blog/posts/ad4c562c/),本文主要探索一下Messenger源码实现。

主要使用到的文件：

[IMessenger.aidl](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/os/IMessenger.aidl)

[Messenger.java](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/os/Messenger.java)

[Handler.java](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/os/Handler.java)

# 解析

一个典型的Messenger服务如下所示：

```kotlin
class MessengerService : Service() {

    private val messenger = Messenger(MessengerHandler())

    override fun onBind(intent: Intent?): IBinder? {
        return messenger.binder
    }

    //可以从客户端的得到的Messenger中取出该Handler，并实现客户端->服务端通信
    class MessengerHandler : Handler() {
        override fun handleMessage(msg: Message?) {
            super.handleMessage(msg)
            //客户端的Messenger，用于服务端->客户端通信,可选
            val client = msg?.replyTo
            client?.send(Message.obtain(null, 2, 1, 2))
        }
    }
}
```

我们可以看到使用Handler创建一个Messenger，进入到源码看一下：

```java
private final IMessenger mTarget;
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}
```

我们看到，在这里创建了一个新的与给定的Handler绑定在一起的Messenger，再看看`getIMessenger()`方法：

```java
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}
private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);//使用Handler发送消息
    }
}
```

这里我们可以看到`getIMessenger()`方法会创建一个MessengerImpl对象，而这个对象

实现了`send()`方法，也证实了我们之前的一个观点——Messenger底层是使用Handler发送消息。

同时，看到MessengerImpl继承的IMessenger.Stub类我们可以联想到这里应该有一个AIDL实现：

```java
// /frameworks/base/core/java/android/os/IMessenger.aidl
package android.os;

import android.os.Message;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
```

> `oneway` 关键字用于修改远程调用的行为。使用该关键字时，远程调用不会阻塞；它只是发送事务数据并立即返回。接口的实现最终接收此调用时，是以正常远程调用形式将其作为来自 `Binder` 线程池的常规调用进行接收。 如果 `oneway` 用于本地调用，则不会有任何影响，调用仍是同步调用
>
> <https://developer.android.google.cn/guide/components/aidl?hl=zh-cn>

这也就解释了在服务的`onBind(intent: Intent?)`方法中，我们可以直接使用`messenger.binder`获取到Binder对象的原因。

再看看Messenger客户端的实现：

```kotlin
private lateinit var messenger: Messenger//服务端的Messenger
private val replyMessenger: Messenger = Messenger(ReplyHandler())//客户端的Messenger，用于服务端->客户端通信，可选
private val mServiceConnection = object : ServiceConnection {
    override fun onServiceDisconnected(name: ComponentName?) {

    }

    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        messenger = Messenger(service)//注意这里的构造方法传入的是IBinder对象
        val message = Message.obtain(null, 1)
        message.replyTo = replyMessenger
        val data = Bundle()
        data.putString("msg", "Hello World")
        try {
            messenger.send(message)//使用服务端的Messenger向服务端发送消息
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

}
```

可以注意到在客户端通过`Messenger(IBinder target)`取得服务端的Messenger，而这里的IBinder对象则是通过服务端的Messenger的`getBinder()`获取的：

```java
public Messenger(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```

`Stub.asInterface()`方法我们在[之前的文章](http://jixiaoyong.github.io/blog/posts/88d0bcd1/)中介绍过，他会根据客户端和服务端是否在同一进程而决定返回Stub实例还是Proxy类实例以实现跨进程通信。

而通过比较 `Messenger(IBinder target)`和`Messenger(Handler target)`两个构造方法我们也可以知道，两个方法都只是用来初始化了`IMessenger mTarget`对象，这也就解释了在服务端和客户端可以通过两个不同的构造方法获取到有同样功能的Messenger。

# 参考资料

《Android开发艺术探索》

[Android 接口定义语言 (AIDL)](https://developer.android.google.cn/guide/components/aidl?hl=zh-cn)

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android中的Messenger详解.md</div>"></iframe>
