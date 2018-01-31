---
title: Android通过Hook启动未注册Activity
date: 2018-01-16 00:10:41
tags: Android hook
---

# 简介

hook是钩子的意思，hook的过程是通过反射、代理等改变系统原有的行为以达到自己的目的。

本文主要是通过hook android 中的ActivityManagerService和Handler.CallBack，欺骗系统调起activity的过程，在调用startActivity时将targetIntent通过proxy伪装为proxyIntent，等到通过系统验证，正式启动activity时，再讲proxyIntent恢复为targetIntent，从而实现调用未在AndroidManifest.xml中注册的activity。

> 需要注意，本方法只在Api<26下有效。具体原因见后面。

# 具体实现

## 1.新建Activity等

`IndexActivity.java`用于启动`targetIntent`

```java
 ((Button)findViewById(R.id.btn1)).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //启动未在AndroidManifest.xml注册的activity
                mContext.startActivity(new Intent(mContext,TargetActivity.class));
            }
        });
```

`TargetActivity.java` 和`ProxyActivity.java` 分别设置对应页面布局`setContentView(R.layout.activity_xxx);`

`HookApplication.java` 用于调用hook方法

```java
public class HookApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        Utils.hookAms(this);
        Utils.hookHandle();
    }
}
```

在`AndroidManifest.xml`中注册`IndexActivity`和`ProxyActivity`，Application使用`HookApplication`。

## 2.Utils.java实现hook具体逻辑

`Utils.hookAms()` 实现拦截targetIntent并发起proxyIntent，欺骗系统对activity是否已注册的验证，其中proxyIntent通过`proxyIntent.putExtra(TARGET_KEY, targetIntent);` 方法携带targetIntent。

```java
//hookAms()核心代码
Class hookActivityManagerNative = Class.forName("android.app.ActivityManagerNative");
            //在api>26时无此变量：gDefault，该方法失效
            Field gDefault = hookActivityManagerNative.getDeclaredField("gDefault");
            gDefault.setAccessible(true);
            Object object = gDefault.get(null);

            Class hookSingleton = Class.forName("android.util.Singleton");
            Field mInstance = hookSingleton.getDeclaredField("mInstance");
            mInstance.setAccessible(true);

            Object oldAms = mInstance.get(object);

            Class hookIActivityManagerService = Class.forName("android.app.IActivityManager");
            Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                    new Class<?>[]{hookIActivityManagerService},
                    new MAmsInvocationHandler(context,oldAms));

			//将原有的ActivityManagerService替换为我们自定义的
            mInstance.set(object,proxy);
```

在`MAmsInvocationHandler` 里面实现targetIntent和proxy的转换

```java
//MAmsInvocationHandler核心代码
public class MAmsInvocationHandler implements InvocationHandler{
  public static final String TARGET_KEY = "targetIntent";
  ...
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("startActivity".equals(method.getName())) {
            int index = 0;
            Intent targetIntent = null;
            for (int i = 0; i < args.length; i++) {
                if (args[i] instanceof Intent) {
                    index = i;
                    targetIntent = (Intent) args[i];
                    break;
                }
            }
            if (targetIntent != null) {
                Intent proxyIntent = new Intent(mContext, ProxyActivity.class);
                proxyIntent.putExtra(TARGET_KEY, targetIntent);
                args[index] = proxyIntent;
            }
        }
        return method.invoke(mOldAms,args);
    }
}
```

至此，已经对activity.startActivity做了拦截，所有的targetIntent都会被拦截，存储在proxyIntent中，以通过系统的检查。

接下来，通过系统检查后，`hookHandle()`通过重写Handler.CallBack，对启动proxyIntent事件做拦截，使之启动targetIntent对应的Activity。

```java
//hookHandle()核心代码
Class activityThreadCls = Class.forName("android.app.ActivityThread");
Method currentActivityThread = activityThreadCls.getDeclaredMethod("currentActivityThread");
currentActivityThread.setAccessible(true);

Object activityThread = currentActivityThread.invoke(null);

Field mH = activityThreadCls.getDeclaredField("mH");
mH.setAccessible(true);
Handler handler = (Handler) mH.get(activityThread);
Field callBack = Handler.class.getDeclaredField("mCallback");
callBack.setAccessible(true);
callBack.set(handler, new ActivityThreadHandlerCallBack(handler));
```

其中`ActivityThreadHandlerCallBack` 将返回我们自定义的CallBack以替换系统的，实现启动targetIntent而非proxyIntent。

```java
//ActivityThreadHandlerCallBack核心代码
public class ActivityThreadHandlerCallBack implements Handler.Callback{
   @Override
    public boolean handleMessage(Message msg) {
        if (msg.what == 100) {
            handleLaunchActivity(msg);
        }
        mHandler.handleMessage(msg);
        return true;
    }
  
  	//主要代码，在这里将proxyIntent转化为targetIntent
     private void handleLaunchActivity(Message msg) {
        Object object = msg.obj;
        try {
            Field intent = object.getClass().getDeclaredField("intent");
            intent.setAccessible(true);
            Intent proxyIntent = (Intent) intent.get(object);
            Intent targetIntent = proxyIntent.getParcelableExtra(MAmsInvocationHandler.TARGET_KEY);
            if (targetIntent != null) {
                proxyIntent.setComponent(targetIntent.getComponent());
            }
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

到这里，就实现了启动通过已经注册了的ProxyActivity启动未注册TargetActivity的全过程。

主要思想是找到系统实现该过程的逻辑，在对应地方通过反射获取到对应变量，插入自己的逻辑，从而达到目的。



# 附录

上面涉及到的代码路径：

[github源代码路径](https://github.com/jixiaoyong/AndroidNote/tree/master/code/AndroidHook/20180116)

参考了几篇文章，其中较为完整的一篇如下：

[Android插件化系列第（一）篇---Hook技术之Activity的启动过程拦截](https://www.jianshu.com/p/69bfbda302df)