---
title: Android中AIDL详解
date: 2019年03月20日19:37:40
tag: 
---

# 前言

`AIDL`是Android中用于IPC的语言，具体使用可以参见[这篇文章](https://jixiaoyong.github.io/blog/posts/f931e8ae/)，这篇文章主要想总结一下`AIDL`具体为我们做了什么工作，主要参考书目《Android开发艺术探索》。

在Android中，除了`Socket`、`Intent`中使用`Bundle`、本地文件共享，`ContentProvider`等等之外，还有一个独有的IPC方式即`Binder`。在日常编程中使用`Binder`的主要有`AIDL`和`Messenger`两种方式，而`Messenger`也是用`AIDL`来实现的。

# 准备

1. 新建一个AIDL文件

```java
// IBookManager.aidl
package cf.android666.myapplication;

interface IBookManager {

    void getSth();
}
```

2. 用AndroidStudio自动生成一个Binder类

使用`Build`->`Make Project`，会在`app/build/generated/aidl_source_output_dir/debug/compileDebugAidl/out`目录下生成`IBookManager.java`。

# 分析

AIDL从客户端(Client)发起请求至服务端(Server)相应的工作流程概览，图片来源(https://blog.csdn.net/qian520ao/article/details/78074983)

![AIDL从客户端(Client)发起请求至服务端(Server)的流程](https://jixiaoyong.github.io/images/20190320205619.png)

下面我们对`IBookManager.java`这个文件简单分析一下

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: app/src/main/aidl/cf/android666/myapplication/IBookManager.aidl
 */
package cf.android666.myapplication;

public interface IBookManager extends
  android.os.IInterface//IInterface接口，所有可以在Binder中传输的接口都要继承自该接口
  {
    /**
     * Local-side IPC implementation stub class.
     * 持有Binder对象
     * 获取客户端传过来的数据，根据方法 ID 执行相应操作。
     * 将传过来的数据取出来，调用本地写好的对应方法。
     * 将需要回传的数据写入 reply 流，传回客户端。
     */
    public static abstract class Stub extends android.os.Binder implements cf.android666.myapplication.IBookManager {
        private static final java.lang.String DESCRIPTOR = "cf.android666.myapplication.IBookManager";//是Binder的唯一标识，一般为当前Binder的类目

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);//将Binder和指定的接口绑定，这样当queryLocalInterface时会返回与DESCRIPTOR一致的IInterface
        }

        /**
         * Cast an IBinder object into an cf.android666.myapplication.IBookManager interface,
         * generating a proxy if needed.
         * 将服务端的Binder转化为客户端需要的IInterface
         * 如果是相同的进程，则直接返回服务端的Stub对象本身（没有跨进程）；
         * 如果是不同的进程，则返回的是Stub.Proxy代理类对象
         */
        public static cf.android666.myapplication.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof cf.android666.myapplication.IBookManager))) {
                return ((cf.android666.myapplication.IBookManager) iin);
            }
            return new cf.android666.myapplication.IBookManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }
      
        /**
        * 客户端远程请求经过系统封装后调用该方法，
        * 生成 _data 和 _reply 数据流，并向 _data 中存入客户端的数据。
        * 通过 transact() 方法将它们传递给服务端，并请求服务端调用指定方法。
        * 接收 _reply 数据流，并从中取出服务端传回来的数据
        */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_getSth: {
                    data.enforceInterface(descriptor);//从data中可以读取参数
                    this.getSth();//注意，这里调用的是IBookManager的getSth()，也就是需要我们在使用该Binder时实现的方法
                    reply.writeNoException();//可以往reply中写入结果
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }
      
        /**
        * Proxy类持有IBinder的引用
        * 
        */
        private static class Proxy implements cf.android666.myapplication.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public void getSth() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getSth, _data, _reply, 0);//这里实际上是调用了远程的IBinder的transact()方法
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_getSth = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);//这个是我们在AIDL中定义的getSth()方法的标志，用于在onTransact中区分调用的是哪个方法
    }

    public void getSth() throws android.os.RemoteException;//这个是我们在AIDL中定义的方法,需要在服务端实现，并且会在客户端被调用
}
```

# 参考资料

[Android 深入浅出AIDL（二）](https://blog.csdn.net/qian520ao/article/details/78074983)

[Android Binder之应用层总结与分析](https://blog.csdn.net/qian520ao/article/details/78089877)