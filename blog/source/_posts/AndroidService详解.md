---
title: AndroidService详解
tags: Android
abbrlink: ad4c562c
date: 2018-01-14 01:10:01
---



## 启动一个Service

- MyServices.java

  必须继承自Service，或者如IntentService本身就是等其子类

  ```java
  public class MyServices extends Service {
      
      @Nullable
      @Override
      public IBinder onBind(Intent intent) {
          Log.d("TAG","onBind");
          return null;
      }

      @Override
      public void onCreate() {
          super.onCreate();
          Log.d("TAG","onCreate");
      }

      @Override
      public void onDestroy() {
          super.onDestroy();
          Log.d("TAG", "onDestroy: ");
      }
  }
  ```

  ​

- AndroidManifest.xml

  注册MyServices

  ```xml
  <application>
  		<service android:name=".MyServices"
              android:exported="true">
              <intent-filter>
                  <action android:name="cf.android666.myservices" />
              </intent-filter>
          </service>
  </application>
  ```

  ​

- MainActivity.java

  在java中调用Service，需要`ServiceConnection`类

  ```java
  ServiceConnection mConnection = new ServiceConnection() {
              @Override
              public void onServiceConnected(ComponentName name, IBinder service) {
                  Log.d(TAG, "onServiceConnected: 服务绑定");
              }

              @Override
              public void onServiceDisconnected(ComponentName name) {
                  Log.d(TAG, "onServiceDisconnected: 服务解绑");
              }
          };
  Intent intent = new Intent(context, MyServices.class);
  bindService(intent, mConnection, Service.BIND_AUTO_CREATE);//绑定Service
  //startService(intent); 启动service
  unbindService(mConnection);//解绑Service
  ```

  `bindService()`和`startService()`的区别在于：

  ** `bindService()`将service和当前的activity绑定在一起，activity销毁时，service也会被销毁；

  ** `startService()`则只是“启动”service，在此后service的活动和activity无关，并一直存活。



# Service具体分析

Service在AndroidManifest.xml中的属性：

```java
android:name=".MyService"//必须被指定
android:exported=true/false //是否能被其他应用隐式调用
//有intent-filter则默认为true，否则默认false；若手动指定为false则即使有intent-filter也无法隐式调用
android:process="remote"/":remote"//前者在共有的进程中进行，后者在名字为{packageName}:remote 的私有进程中进行，其他进行不可访问；如果不设置该属性，则service在应用自己的进程里面运行
```

Service默认运行在创建他的线程中，要是进行耗时操作，最好在service中单独创建一个线程，这样子可以在子线程工作，在主线程中更新工作进度。

Service中的方法：

```java
//在初次创建服务时调用，并且直至服务死亡，只会被调用一次
void onCreate()

//在绑定服务是才会被调用，必须实现该方法
IBinder onBind(Intent intent)
  
//每一次通过startService()方法启动Service的时候都会被调用
int onStartCommand(Intent intent, int flags, int startId)
  //1.intent 启动时，启动组件传递过来的Intent
  //2.flags 表示启动请求时是否有额外数据，可以是：
  //  0：无
  //  START_FLAG_REDELIVERY：表示该方法返回值为START_REDELIVER_INTENT，在上个服务被杀死之前调用stopSelf()停止服务
  //  START_FLAG_RETRY：在onStartCommand()被调用后一直无返回值时，会尝试重新调用onStartCommand()
  //3.当前服务id
```

其中`onStartCommand()`方法的返回值意义如下：

`START_STICKY `:service在内存不足被杀死后，内存空闲时系统会重新创建service，一旦成功创建会回调`onStartCommand()`方法，此时intent是null，除非是挂起的intent如pendingintent，无限期运行

`START_NOT_STICKY`：service因内存不足被杀死，内存再次空闲系统也不会再重新创建服务，最安全

`START_REDELIVER_INTENT`：service因内存不足被杀死，会重建服务并传递给最后一个intent（最后一次调用`startService()` 时的intent），用于连续作业，如下载等



## Service绑定服务的三种方式

### 1.拓展Binder类

**要求客户端和服务在同一应用的同一进程内**。客户端通过其访问service中的公共方法。

步骤如下：

1. 创建BindService服务端，在类中创建一个实现了IBinder接口的实力对象并提供公共方法给客户端使用
2. 在onBind()回调方法返回此Binder实例
3. 在客户端的onServiceConnected()方法接收Binder，使用提供的方法绑定服务

```java
//service服务端
public class LocalService extends Service{
  LocalService mService;
  private LocalBinder binder = new LocalBinder();
  ...
  public IBinder onBind(Intent intent){
    return binder;
  }
  
  public void doSomeThing(){
    //服务中公共方法，可以被客户端通过IBInder获取实例调用
  }
  
  public class LocalBinder extends Binder{
    LocalService getService(){
      return LocalService.this;
    }
  }
}
//客户端
public class BindActivity extends Activity{
  protected void onCreate(...){
    ServiceConnection conn = new ServiceConnection(){
      //绑定服务时被调用，实现客户端和服务端交互（IBinder）
      public void onServiceConnected(ComponentName name, IBinder service){
        LocalService.LocalBinder binder = (LocalService.LocalBinder)service;//获取服务端IBinder
        mService = binder.getService();//获取服务实例，以调用服务的公共方法
      }
      //取消绑定时回调，多数时候是service被意外销毁，如内存不足
      //当客户端取消绑定时，系统“绝对不会”调用该方法。
      public void onServiceDisconnected(ComponentName name){
        mService = null;
      }
    };
    //创建绑定对象
    Intent intent = new Intent(this,LocalService.class);
    //绑定服务
    //参数3 flags则是指定绑定时是否自动创建Service。0代表不自动创建、BIND_AUTO_CREATE则代表自动创建
    bindService(intent,conn,Service.BIND_AUTO_CREATE);
    //调用服务中的方法，最好先判断是否为null
    mService.doSomeThing();
    //解除绑定
    unbindService(conn);
  }
}
```



### 2.Message

**service与不同进程通信（IPC）** 。

步骤如下：

1.  Service实现一个Handler，接收客户端每个调用的回调
2. 用Handler创建Messenger对象
3. 用Messenger创建IBinder对象，并通过onBind()返回客户端
4. 客户端使用IBinder实例化Messenger，用其将Message对象发送给Service
5. Service在Handler接收并处理Message

```java
//Service
public class MessageService extends Service{
  public final static int MSG_WHAT = 1;
  //创建Handler接收、处理客户端msg
  class IncomingHanler extends Handler{
    public void handleMessage(Message msg){
      //do sth with msg...
    }
  }
  Messenger messenger = new Messenger(new IncomingHanlder());
  public IBinder onBind(Intent intent){
    return messenger.getBinder();
  }
}

//客户端
//onCreate()方法中：
mConnection = new ServiceConnection(){
  public void onServiceConnected(ComponentName className, IBinder service){
    Messenger mService = new Messenger(service);
  }
};
//给服务发消息
Message msg = Message.obtain(null,MessengerService.MSG_WHAT,0,0);
mService.send(msg);
```

注意service要在不同的进程中：

```xml
AndroidMinafast.xml
<service android:name=".messenger.MessengerService"
         android:process=":remote"
        />
```

**服务与客户端双向通信**

服务端，修改IncomingHandler，回复客户端消息

```java
class IncomingHandler extends Handler{
  public void handleMessage(Message msg){
    //回复消息
    Messenger client = msg.replyTo;
    Message replyMsg = Message.obtain(null,MessengerService.MSG_WHAT);
    Bundle bundle = new Bundle();
    bundle.putString("key","value");
    replyMsg.setData(bundle);
    try{
      client.send(replyMsg);
    }catch(){}
  }
}
```

客户端，增加Messenger和Handler处理服务端回复

```java
private static class RecyclerReplyMsgHandler extends Hanlder{
  public void handleMessage(Message msg){
    //接收服务端返回的msg
  //do sth ...
  }
}
private Messenger mRecevierReplyMsg = new Messenger(new RecyclerReplyMsgHandler());
```

此外，在发送消息是需要将接收服务端回复的Messenger通过Message的replyTo传递给服务端

```java
//create msg...
msg.replyTo = mRecevierReplyMsg;
//send msg...
```

### 3.AIDL

一般不会使用



# 绑定服务时的注意事项

- 多个客户端可连接一个服务端，只有第一个客户端绑定时才会调用服务`onBind()`方法来检索IBinder，此后无需调用就可将同一个IBinder传递给其他客户端
- `bindService()` 绑定服务是异步进行的
- 一般在activity可见生命周期内绑定-取消服务，不要在`onResume()`、`onPause()`期间执行绑定/解绑



# Service绑定和启动转换

| 顺序            | 结果                           |
| ------------- | ---------------------------- |
| 先绑定后启动service | 启动service                    |
| 先启动后绑定service | 会绑定宿主，但是宿主死后仍按照启动service方式存活 |



# 前台服务和通知

> - **startForeground(int id, Notification notification)** 
>   该方法的作用是把当前服务设置为前台服务，其中id参数代表唯一标识通知的整型数，需要注意的是提供给 startForeground() 的整型 ID 不得为 0，而notification是一个状态栏的通知。
> - **stopForeground(boolean removeNotification)** 
>   该方法是用来从前台删除服务，此方法传入一个布尔值，指示是否也删除状态栏通知，true为删除。 注意该方法并不会停止服务。 但是，如果在服务正在前台运行时将其停止，则通知也会被删除。



文章参考：

[关于Android Service真正的完全详解，你需要知道的一切 - CSDN博客](http://blog.csdn.net/javazejian/article/details/52709857#t3)