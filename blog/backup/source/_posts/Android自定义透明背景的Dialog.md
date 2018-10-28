---
title: Android自定义透明背景的Dialog
tags: android
abbrlink: f20627c9
date: 2018-01-26 15:09:40
---

# 简介

通过自定义Dialog类，使用Style、AnimationDrawable等实现一个透明背景的、带进度更新的弹窗。

主要涉及Style自定义以及AnimationDrawable的使用。

# 代码

* **布局文件**

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:gravity="center"
    android:background="@drawable/dialog_bg">

    <ImageView
        android:id="@+id/image"
        android:layout_width="250dp"
        android:layout_height="250dp"
      />

</LinearLayout>
```

* **资源文件**

1）下载对应进度条的图片资源，放到drawable目录下

2）在drawable下新建dialog_progress.xml

```xml
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">

    <item
        android:drawable="@drawable/progress_1" //资源文件
        android:duration="300" /> //持续时间ms

    <item
        android:drawable="@drawable/progress_2"
        android:duration="300" />
  
  ...
  
</animation-list>
```

3）dialog圆角背景（非必须）

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android">

    <solid android:color="#6fb6d4" />
    <corners android:radius="500dp" />

</shape>
```

4）自定义dialog的style

`windowBackground`使背景透明

`backgroundDimEnabled`则可以去除半透明遮罩效果

```xml
    <style name="diyDialogStyle" parent="@android:style/Theme.Dialog" >
        
        <item name="android:windowBackground">@android:color/transparent</item><!--背景透明-->
        <item name="android:backgroundDimEnabled">false</item><!--半透明，模糊-->

    </style>
```

* **DIYDialog.java**

继承自`Dialog.java` ，并用构造函数调用`initView()`方法初始化dialog样式，有其他需求可以再自己实现。

```java
//初始化view、控件
View view = View.inflate(context, R.layout.layout_dialog, null);
ImageView imageView = view.findViewById(R.id.image);
imageView.setBackgroundResource(R.drawable.dialog_progress);
//填充布局
setContentView(view);
//实现动画
AnimationDrawable animationDrawable = (AnimationDrawable) imageView.getBackground();
animationDrawable.run();
```

# 源码

[github源码路径](https://github.com/jixiaoyong/AndroidNote/tree/master/code/2018-1-26/DIY_Dialog)