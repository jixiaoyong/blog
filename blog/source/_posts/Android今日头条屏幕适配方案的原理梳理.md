---
title: Android今日头条屏幕适配方案的原理梳理
tags: android
date: 2020-10-13 11:13:18
---

# 前言

最近在项目里面遇到了屏幕适配的问题，UI要求APP在不同手机上展示效果和设计稿保持“像素级”同步，在对比了几种屏幕适配方案之后，选择了基于今日头条的[AndroidAutoSize](https://github.com/JessYanCoding/AndroidAutoSize)适配方案。

本文主要简单分析其适配原理，以及在实际使用中遇到的一个问题，需要更深入了解原理可以阅读文末参考文献。

# 正文

UI给的设计稿一般都是以像素px为单位，而在Android开发中官方推荐的使用的单位是dp。

> dp 是一个虚拟像素单位，1 dp 约等于中密度屏幕（160dpi；“基准”密度）上的 1 像素。对于其他每个密度，Android 会将此值转换为相应的实际像素数。
>
> —— Android Developer

根据Android官方的定义，dp在屏幕上实际对应的像素px计算方式如下：

```kotlin
px = dp * (dpi / 160)
```

其中 dpi表示：**屏幕每平方英寸有多少像素**，可以通过屏幕对角线的像素数px/屏幕尺寸inch计算。

而`DisplayMetrics.density` 字段表示根据当前像素密度指定将 `dp` 单位转换为像素时所必须使用的缩放系数，即上述方程等价于：

```kotlin
px = dp * (dpi / 160)
   = dp * getResources().getDisplayMetrics().density
```

这样，在dpi为160的屏幕上1dp占1px，在dpi为320的屏幕上占2px，那么就能保证同一dp的在不同dpi上占得像素是等比例变化的。



但是，在现实生活中面对千变万化的Android屏幕，根据Jessyan的文章可知由于每种屏幕宽/高对应的总dp数不一定都是相同的，所以即使使用了dp作为单位，还是会出现同一dp在有些屏幕上刚好占满全屏，在有的屏幕上会无法占满全屏或超出屏幕范围。

> **density** 在每个设备上都是固定的，**DPI / 160 = density**，**屏幕的总 px 宽度 / density = 屏幕的总 dp 宽度**
>
> - 设备 1，屏幕宽度为 **1080px**，**480DPI**，屏幕总 **dp** 宽度为 **1080 / (480 / 160) = 360dp**
> - 设备 2，屏幕宽度为 **1440px**，**560DPI**，屏幕总 **dp** 宽度为 **1440 / (560 / 160) = 411dp**
>
> ——Jessyan

那么该怎么适配呢，再看一眼上述的公式：

```kotlin
屏幕的总 px 宽度 / density = 屏幕的总 dp 宽度
```

以适配**屏幕宽度**为例，要使得dp在不同屏幕上对应的像素等比例变化，就要**保证屏幕的总dp宽度一致**，而屏幕的总 px 宽度是物理条件无法更改，那么就只能**更改density**。



以我们使用的设计稿宽度为375dp为例：

在分辨率为2160*1080 、尺寸为5.99英寸的屏幕上：

```kotlin
density = 1080px / 375dp = 2.88
```

 而在分辨率为2400*1176、尺寸为6.53英寸的屏幕上：

```kotlin
density = 1176px / 375dp = 3.136
```

这样就保证了，不管在什么样的屏幕上，375dp始终都能够占满屏幕宽度，保证了布局在不同大小的屏幕上，在屏幕宽度上的比例一致性，也就解决屏幕适配的问题。



# 获取状态栏高度的问题

上述的屏幕适配方案使用简单，且侵入小，在使用到项目中之后，除了部分字体等显示需要微调外，其余内容基本上都完美还原了设计稿的内容。

但是在后续使用到状态栏相关代码的时候发现**获取到的状态栏高度和实际高度不一致**，导致显示异常，而使用[Blankj](http://blankj.com)的工具类 `BarUtils.getStatusBarHeight()`却可以获取到正确的高度。

对比两种代码发现获取状态栏高度的代码逻辑几乎一样：

```java
public static int getStatusBarHeight(Resources resources) {
    int resourceId = resources.getIdentifier("status_bar_height", "dimen", "android");
    return resources.getDimensionPixelSize(resourceId);
}
```

不同的是，两种方法使用到的resources一个是APP的，一个是系统的

```kotlin
// 1. 我使用到的resources，从当前activity获取
resources.displayMetrics.density
// 2. Blankj使用的resources，从系统获取
Resources.getSystem().displayMetrics.density
```

通过分别打印这两种resources可以发现，二者的density值不一样（以2160*1080 、尺寸为5.99英寸的屏幕为例）：

```kotlin
context.resources.DisplayMetrics: DisplayMetrics{density=2.88, width=1080, height=2033, scaledDensity=2.88, xdpi=403.411, ydpi=403.411}

Resources.getSystem().DisplayMetrics: DisplayMetrics{density=2.7, width=1080, height=2033, scaledDensity=2.7, xdpi=403.411, ydpi=403.411}
```

这是由于使用了[AndroidAutoSize](https://github.com/JessYanCoding/AndroidAutoSize)适配方案后，APP内部的density已经被改成了2.88，而系统实际的density是2.7。

又知道android中将像素和dp等单位转化的方法如下：

```java
// android.util.TypedValue
public static float applyDimension(int unit, float value,
                                       DisplayMetrics metrics)
    {
        switch (unit) {
        case COMPLEX_UNIT_PX:
            return value;
        case COMPLEX_UNIT_DIP:
            return value * metrics.density;
        case COMPLEX_UNIT_SP:
            return value * metrics.scaledDensity;
        case COMPLEX_UNIT_PT:
            return value * metrics.xdpi * (1.0f/72);
        case COMPLEX_UNIT_IN:
            return value * metrics.xdpi;
        case COMPLEX_UNIT_MM:
            return value * metrics.xdpi * (1.0f/25.4f);
        }
        return 0;
    }
```

分析可知，通过getStatusBarHeight()获取到的状态栏是系统的状态栏69px（即25dp），但当使用APP内部的density=2.88计算时就会只有24dp，和实际的状态栏高度不一致，所以使用状态栏高度来控制布局的时候就会展示异常。





# 参考资料

[骚年你的屏幕适配方式该升级了!-今日头条适配方案——jessyan](http://jessyan.me/autosize-introduce/)

[一种极低成本的Android屏幕适配方式——字节跳动](https://mp.weixin.qq.com/s/d9QCoBP6kV9VSWvVldVVwA)

[支持不同的像素密度——Android Developers](https://developer.android.google.cn/training/multiscreen/screendensities#top_of_page)

[Android 目前稳定高效的UI适配方案——拉丁吴](https://mp.weixin.qq.com/s/X-aL2vb4uEhqnLzU5wjc4Q)

[AndroidAutoSize](https://github.com/JessYanCoding/AndroidAutoSize)

[请问两种获取屏幕密度的方式有什么区别，望解答多谢](https://github.com/gyf-dev/ImmersionBar/issues/298)
