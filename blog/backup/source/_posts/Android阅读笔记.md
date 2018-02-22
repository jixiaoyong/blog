---
title: Android阅读笔记
date: 2018-02-22 19:29:48
tags: android
---



## layout_weight

layout_weight 重要性，默认的是0,0等级最高，要显示，数字越大重要性越低。

例：a，b的宽度为0，layout_weight分别为1、2，则a，b宽度分别为父容器的2/3、1/3。

## PendingIntent

PendingIntent是封装后的intent，有intent执行所需的context，所以即使要执行intent的activity已经消失或者还没生成，其他activity依然可以通过PendingIntent执行intent。

> PendingIntent is a description of an Intent and target action to perform with it. Instances of this class are created with `getActivity(Context, int, Intent, int)`, `getActivities(Context, int, Intent[], int)`, `getBroadcast(Context, int, Intent, int)`, and `getService(Context, int, Intent, int)`; the returned object can be handed to other applications so that they can perform the action you described on your behalf at a later time.

也就是把自己要执行的intent和执行所需的context封装后给别人，请别人在适当的时候执行。

## android模拟器访问电脑localhost

电脑`localhost`或者`127.0.0.1`访问本地网址。

模拟器访问`localhost`会默认访问手机的本地网址，要访问电脑的本地网址则需要访问`10.0.2.2:8080`，记得加上对应的端口。

## 获取屏幕画面

```java
View decor = MainActivity.this.getWindow().getDecorView();
decor.setDrawingCacheEnabled(true);
imageView.setImageBitmap(decor.getDrawingCache());
```

## 获取网络信息，请求网络

需要请求权限

```xml
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

java代码如下：

```java
    private void chackNetWork(Context context) {
        boolean isNetAvailable = false;
        ConnectivityManager manager = (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);
        if (manager.getActiveNetworkInfo() != null) {
            isNetAvailable = manager.getActiveNetworkInfo().isAvailable();
        }
        if (!isNetAvailable) {
            Toast.makeText(this, "open network", Toast.LENGTH_SHORT).show();
            Intent intent = new Intent();
            intent.setAction(Settings.ACTION_NETWORK_OPERATOR_SETTINGS);
            context.startActivity(intent);
        }
    }
```

