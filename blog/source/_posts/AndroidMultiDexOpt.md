---
title: Android 5.x以下加载MultiDex白屏的处理优化
date: 2019-08-10 11:39:00
tag: android
---

当APP的minSdkVersion低于Android 5时，在方法数大于65536时，需要将APP打包为多个DEX文件，此时需要添加MultiDex依赖。

官方方法如下：

1.`build.gradle`

```groovy
android {
    defaultConfig {
        ...
        minSdkVersion 15 
        targetSdkVersion 28
        multiDexEnabled true
    }
    ...
}

dependencies {
  compile 'com.android.support:multidex:1.0.3'
}
```

2.`MyApplication`

```java
方式❶：
public class MyApplication extends MultiDexApplication { ... }

方式❷：
public class MyApplication extends SomeOtherApplication {
  @Override
  protected void attachBaseContext(Context base) {
     super.attachBaseContext(base);
     MultiDex.install(this);
  }
}
```

此外，为了避免一些启动期间需要的任何类未在主 DEX 文件中提供而导致`java.lang.NoClassDefFoundError`，还需要告诉AS将这些类添加到主DEX文件中：

3.`build.gradle`

```groovy
android {
    buildTypes {
        release {
            ❶ multiDexKeepFile file('multidex-config.txt')
            ❷ multiDexKeepProguard('multidex-config.pro')
            ...
        }
    }
}
//multidex-config.txt
com/example/MyClass.class
com/example/MyOtherClass.class

//multidex-config.pro
-keep class com.example.MyClass
-keep class com.example.MyClassToo
-keep class com.example.** { *; } // All classes in the com.example package
```

但是在实际运行中，Android 4.x的系统会在APP安装后第一次启动时，在`MultiDex.install(this)`方法中进行DEX文件合并优化等耗时操作（主线程），往往会持续数十秒以上，从而导致APP第一次启动时长时间白屏，十分影响体验。

查阅相应的资料后大体有以下几种方案

1. 设置主Activity的背景为透明色

   这样当用户点击APP图标启动APP时，在主Activity启动之前看到的一直是桌面的样子而非白屏，但这只是一种障眼法，用户可能会以为系统卡顿，体验并不好。

2. 在Application中检测到是第一次启动的话，新开一个进程并在其中进行`MultiDex.install(this)`

   这种方法在主进程启动时，检测到尚未进行MultidexOpt，则阻塞当前进程，新开一个进程，在其中加载一个Activity，并在后台进程运行`MultiDex.install(this)`，当MultidexOpt完成后再关闭当前进程，返回主进程继续正常开启APP。

   由于主进程被阻塞的同时成为了后台进程，所以也不会触发ANR，此外子进程中的过渡Activity也只用到了基本的类，所以基本不用担心会触发`java.lang.NoClassDefFoundError`，而且过渡Activity可以展示进度、提示等用户友好的页面，相对来说体验也好了很多。

   但是这种方法从子进程返回主进程涉及到进程间通信，以及主进程的主Activity启动时生命周期会出现异常(`onCreate()` -> `onStart()` -> `onResume()` -> `onPause()`->`onResume()`)，仍然不是很好的解决方法。

结合上述的分析后，可以看到这种问题的优化思路主要在于如何在避免`java.lang.NoClassDefFoundError`的同时，在后台可靠的通过`MultiDex.install(this)`执行MultidexOpt操作。

通过以上方案1和2的结合，可以有一个比较完美的解决方案：

1. 方案2中在过渡Activity的后台线程进行MultidexOpt操作思路是正确的，但是不需要再单独开一个进程，我们完全可以将其当做主进程的第一个Activity，等待MultidexOpt操作完成后再跳转到主Activity并finish掉本Activity，这样主Activity的生命周期也不会受影响。
2. 这种情况下在部分低端机上，过渡Activity到主Activity跳转时会出现短暂黑屏，我们可以在过渡页面将Activity切换动画设置为渐变效果，并将主Activity背景设置为透明，待主Activity完全加载好后再将背景切换为普通模式。

综上处理，我们的Application无需改动，甚至主Activity也可以不做改动，只需要添加一个过渡页面为启动Activity，在其中后台进行MultidexOpt，等DEX文件处理完毕后再加载主Activity。对项目改动少并且逻辑较为简单。



# 参考文档

[配置方法数超过 64K 的应用](https://developer.android.google.cn/studio/build/multidex.html?hl=zh-CN)

[Android MultiDex初次启动APP优化方案优雅的实现](https://www.jianshu.com/p/c2d7b76ff063)

[MultiDex深入学习](https://www.zybuluo.com/946898963/note/1219741)


<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/edit/hexo_blog/blog/source/_posts/AndroidMultiDexOpt.md</div>"></iframe>


