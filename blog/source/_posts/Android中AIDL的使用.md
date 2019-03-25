---
title: Android中AIDL的使用
tags: aidl
abbrlink: f931e8ae
date: 2018-08-07 21:15:39
---

AIDL（Android Interface Definition Language ，Android接口定义语言）用于Android IPC，适用于**大量并发**请求。

主要分为两部分：

1. 服务端 创建Service监听Client的请求，通过创建AIDL将接口暴露给客户端
2. 客户端 绑定到服务端获取BInder对象，将其转化为对应AIDL，并调用接口对应方法。

两者的连线就是AIDL，因此两个APP的AIDL必须一致，可以将AIDL文件放到一个Android Library中，或者打成aar文件供二者依赖。

也可以将AIDL涉及到的AIDL文件、java都放到AIDL文件夹下，然后在build.gradle的`android{...}`中添加

```
    sourceSets{
         main{
             java.srcDirs = ['src/main/java','src/main/adil']
         }
     }
```

 即添加一个java路径

# AIDL 文件特点

## 支持的数据格式

基本数据类型、List（ArrayList）、Map（HashMap）以及实现了Parcelable接口的对象、AIDL接口。

## 注意事项

* 自定义的Parcelable对象、AIDL对象必须显示import。
* AIDL中用到的Parcelable对象必须新建一个同名AIDL接口，声明其为Parcelable类型。

```java
// People.aidl
package cf.android666.androidlib;
import cf.android666.androidlib.People;
// Declare any non-default types here with import statements

parcelable People;
```

* AIDL中除了基本数据类型，其他的参数必须标记方向（in,out,inout）。
* AIDL中只支持方法，不支持静态变量。

# AIDL用法

## AIDL

```java
// ManagerAidl.aidl
package cf.android666.androidlib;

import cf.android666.androidlib.People;
import cf.android666.androidlib.TaskCallBack;

interface ManagerAidl {
    //客户端提供的方法
    List<People> getPeopleList();
    void addPeople(in People people);

    //回调接口，用于服务端往客户端通信
    void registerCallBack(in TaskCallBack callback);
    void unregisterCallBack(in TaskCallBack callback);
}
```

```java
// TaskCallBack.aidl
package cf.android666.androidlib;
import cf.android666.androidlib.People;

// https://blog.csdn.net/woshiwoshiyu/article/details/54266101
//回调的具体方法，供服务端回调
interface TaskCallBack {
    void callBack(in int size);
    void onPeopleChange(in List<People> peoples);
}
```

```java
//People.java
package cf.android666.androidlib;
public class People implements Parcelable {
    ...
}
```

```java
//PeopleManager.java
package cf.android666.androidlib;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.IBinder;
import android.os.RemoteException;
import android.util.Log;

import java.util.ArrayList;
import java.util.List;

/**
 * 一个管理类，封装了客户端绑定服务端的一些方法
 * 属于客户端部分，不过放在AIDL中便于多个客户端开发
 * Created by jixiaoyong on 2018/8/6.
 * email:jixiaoyong1995@gmail.com
 */
public class PeopleManager {

    private static PeopleManager mPeopleManager;

    private  Context mContext;
    private  Listener mListener;
    private ManagerAidl managerAidl;

    private List<People> peopleList;

    //实现该回调方法，用于调用客户端的具体方法
    //注意这里是new TaskCallBack.Stub()，而非new TaskCallBack(),否则服务器无法接收到callback
    //TaskCallBack.Stub()是TaskCallBack的子类，当跨进程通信时传递的是proxy类
    private TaskCallBack callBack = new TaskCallBack.Stub() {
        @Override
        public void callBack(int size) throws RemoteException {
            mListener.onCallback(size);
        }

        @Override
        public void onPeopleChange(List<People> peoples) throws RemoteException {
            peopleList = peoples;
            mListener.onPeopleListChange(peoples);
        }
    };

    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //在连接上服务端后，客户端从IBinder对象中获取到AIDL接口对象，并执行其方法
            managerAidl =  ManagerAidl.Stub.asInterface(service);
            try {
                peopleList = managerAidl.getPeopleList();
                managerAidl.registerCallBack(callBack);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            mListener.onCreate(mPeopleManager);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            try {
                managerAidl.unregisterCallBack(callBack);
            } catch (RemoteException e) {
                e.printStackTrace();
            }

        }
    };

    private PeopleManager(Context context,Listener listener) {
        mContext = context;
        mListener = listener;
        peopleList = new ArrayList<>();

        Intent intent = new Intent();
        intent.setComponent(new ComponentName("cf.android666.demo",
                "cf.android666.demo.MService"));//Android5.0后必须显示的启动服务
        context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
    }

    public static void init(Context context,Listener listener) {
        mPeopleManager = new PeopleManager( context, listener);
    }

    public void addPeople(People people) {
        try {
            managerAidl.addPeople(people);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    public List<People> getPeopleList() {
        return peopleList;
    }

    //子类可以实现该Listener的方法，在服务端调用这些方法时执行对应操作
    public interface Listener {
        void onCreate(PeopleManager peopleManager);//服务连接成功
        void onCallback(int size);
        void onPeopleListChange(List<People> peoples);
    }
}
```

## 服务端

注意MService在AndroidManife.xml中配置:

`android:exported="true"android:enabled="true"android:process=":people"`

```java
//MService.java
package cf.android666.demo;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.os.RemoteCallbackList;
import android.os.RemoteException;
import android.support.annotation.Nullable;
import android.text.InputFilter;
import android.util.Log;

import java.util.ArrayList;
import java.util.List;

import cf.android666.androidlib.ManagerAidl;
import cf.android666.androidlib.People;
import cf.android666.androidlib.TaskCallBack;


/**
 * Created by jixiaoyong on 2018/8/6.
 * email:jixiaoyong1995@gmail.com
 */
public class MService extends Service implements ManagerAidl.Stub.DeathRecipient {

    private List<People> mPeopleList;
    private static RemoteCallbackList<TaskCallBack> callbackList = new RemoteCallbackList<>();;
    private TaskCallBack mCallBack;

	//使用AIDL接口生成mIBinder，在服务端实现接口各个方法，供客户端调用
    private IBinder mIBinder = new ManagerAidl.Stub() {

        @Override
        public List<People> getPeopleList() throws RemoteException {
            return mPeopleList;
        }

        @Override
        public void addPeople(People people) throws RemoteException {
            mPeopleList.add(people);
            onPeopleChange(mPeopleList);
        }

        @Override
        public void registerCallBack(TaskCallBack callback) throws RemoteException {
            mCallBack = callback;
            Log.d("TAG", "registerCallBack注册回调方法 callback == null" + callback);
            if (callback != null) {//注意这里一定要判断非空
                callbackList.register(callback);
            }
        }

        @Override
        public void unregisterCallBack(TaskCallBack callback) throws RemoteException {
            if (callback != null) {
                callbackList.unregister(callback);
            }
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.d("tag", "onBind  MService开始了" );

        if (mPeopleList == null) {
            mPeopleList = new ArrayList<>();
        }

        for (int i = 0; i < 20; i++) {
            mPeopleList.add(new People("people" + i, i));
        }

        return mIBinder;//返回开始用AIDL创建的IBinder
    }

	//实现DeathRecipient接口的方法，在客户端终止后自动调用该方法
    @Override
    public void binderDied() {
        callbackList.unregister(mCallBack);
    }

    //这里时在服务端调用回调方法的写法，是从callbackList依次取出来执行
    private void onPeopleChange(List<People> peoples) {
        if (callbackList == null) {
            return;
        }
        int len = callbackList.beginBroadcast();
        try {
            for (int i = 0; i < len; i++) {
                callbackList.getBroadcastItem(i).onPeopleChange(peoples);
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }finally {
            callbackList.finishBroadcast();
        }
    }
}
```

注意：

1. 这里用来注册监听的类是RemoteCallbackList

   我们知道跨进程的两个listener是两个不同的对象，那他是怎么保证跨进程注册、注销的是指定的listener呢？

   这是因为虽然两个listener对象不同，但是他们底层的Binder对象是同一个，在RemoteCallbackList中有一个以Binder对象为KEY的map来存放这些listener对象，当要注销时，只需要按当前待注销的listener的Binder对象找到已经注册了的listener并删除掉即可。

   ```java
   ArrayMap<IBinder, Callback> mCallbacks
           = new ArrayMap<IBinder, Callback>()
   ```

   此外，RemoteCallbackList可以在客户端死亡的时候自动注销掉对应的listener，这是因为他在注册的同时也对Binder的死亡就行了监听。

   ```java
   public boolean register(E callback, Object cookie) {
       synchronized (mCallbacks) {
           if (mKilled) {
               return false;
           }
           // Flag unusual case that could be caused by a leak. b/36778087
           logExcessiveCallbacks();
           IBinder binder = callback.asBinder();
           try {
               Callback cb = new Callback(callback, cookie);
               binder.linkToDeath(cb, 0);//监听binder的死亡事件
               mCallbacks.put(binder, cb);
               return true;
           } catch (RemoteException e) {
               return false;
           }
       }
   }
   ...
   //当binder死亡时，会主动移除其注册的listener
   public void binderDied() {
               synchronized (mCallbacks) {
                   mCallbacks.remove(mCallback.asBinder());
               }
               onCallbackDied(mCallback, mCookie);
           }
   ```

2. 方法运行的线程

   **如果客户端和服务端运行在同一进程**：客户端调用服务端和服务端回调客户端方法（RemoteCallbackList，下同）都会运行在同一线程，即客户端调用服务端时所在的线程，默认为主线程

   **如果客户端和服务端运行在不同进程**：客户端调用服务端方法，客户端会被挂起，直到服务端方法在Binder线程池中运行完毕,这种情况下服务端可以执行耗时操作而无需另建线程；服务端回调客户端方法运行在客户端主线程(与客户端调用服务端方法在同一线程)

   通过上述分析，可以注意到一个细节：**虽然在服务端中回调客户端的方法是在服务端的Binder线程，但是在客户端中被回调的方法却是和客户端中主动调用服务端方法的线程一致**。

## 客户端

```java
PeopleManager.init(this, this);
```

```java
//服务连接成功后，可以开始调用服务的一系列方法
@Override
public void onCreate(PeopleManager peopleManager) {
    mPeopleManager = peopleManager;
    peopleList = peopleManager.getPeopleList();

    for (int i = 0; i < peopleList.size(); i++) {
        Log.d("tag", "people list is " + peopleList.get(i));
    }
}

//其他回调方法，等服务端回调时会执行对应方法
@Override
public void onPeopleListChange(List<People> peoples) {
    Log.d("TAG", "demo2 people变化了" + peoples.size());
}
```

## 监听并处理Binder死亡事件

当服务端进程意外死亡时，我们可以选择重新连接服务，一般有两种方式：

1. binderDied 在客户端的Binder线程池中
2. onServiceDisconnected 在客户端UI线程

## AIDL的权限验证

可以在服务的onBind(Intent intent)或者onTransact()方法中做验证

做验证的手段有：1.permission验证；2.Uid，Pid等做验证

# 参考资料

《Android开发艺术探索》

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android中AIDL的使用.md</div>"></iframe>
