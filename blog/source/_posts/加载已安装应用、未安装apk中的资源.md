---
title: 加载已安装应用、未安装apk中的资源
tags: android
abbrlink: 8d60b485
date: 2018-04-15 21:40:32
---

 加载已安装应用、未安装apk中的资源，其思路主要是获取到对应的ClassLoader/Context，通过ClassLoader加载R.java等类，再通过反射获取对应的资源id及资源。

# 加载已安装应用资源

## sharedUserId

在当前应用中加载已安装的其他应用资源，需要二者有相同的`sharedUserId`，这样Android系统为二者分配同一个Linux用户ID，两个App可以相互访问代码、资源等。

> 通过Shared User id,拥有同一个User id的多个APK可以配置成运行在同一个进程中.所以默认就是可以互相访问任意数据. 也可以配置成运行成不同的进程, 同时可以访问其他APK的数据目录下的数据库和文件.就像访问本程序的数据一样。
>
> [Android逆向之旅---Android中的sharedUserId属性详解 - CSDN博客]( https://blog.csdn.net/jiangwei0910410003/article/details/51316688)

具体设置方法如下

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="cf.android666.dynamicloadapk"
    android:sharedUserId="cf.android666.dynamic">
</manifest>
```

## 筛选所有已安装应用信息

```kotlin
private var packageBeanList: ArrayList<PackageInfoBean> = arrayListOf()
private var packageInfoList: ArrayList<PackageInfo> = arrayListOf()

var packageInfoList = packageManager.getInstalledPackages(PackageManager.GET_UNINSTALLED_PACKAGES) as ArrayList<PackageInfo>

if (packageInfoList.isNotEmpty()) {
    for (x in packageInfoList) {
        if (x.sharedUserId != null
            && x.sharedUserId.equals(sharedUid)
            && !x.packageName.equals(packageName)) {
            //sharedUserId与当前App相同，且packageName和当前App不同的App信息，即插件App
            packageBeanList.add(PackageInfoBean(packageManager
                                                .getApplicationLabel(x.applicationInfo).toString(), x.packageName))
                }
            }
        }
```

## 生成插件App的Context

```kotlin
activity.createPackageContext("cf.android666.pluginapp",
        Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY)
```

## 通过Context反射获取插件App中的资源

```kotlin
//获取ClassLoader
var pClassLoader = PathClassLoader(pluginContext.packageResourcePath
                , ClassLoader.getSystemClassLoader())
//反射获取该类及其资源
var clazz = pluginContext.classLoader
        .loadClass(pluginContext.packageName + ".R\$mipmap")
var abc = clazz.getField(s)
var id = abc.getInt(R.mipmap::class.java)
//调用插件App的Context获取其资源
var bg = pluginContext.resources.getDrawable(id)
```

# 加载未安装Apk内资源

## 获取apk信息

```kotlin
val sdPath = Environment.getExternalStorageDirectory().absolutePath
val apkPath = "$sdPath/plugin/plugin.apk"
var info = packageManager.getPackageArchiveInfo(apkPath, PackageManager.GET_ACTIVITIES)//获取未安装apk的packageInfo
```

## 获取ClassLoader

```kotlin
var file = getDir("dex", Context.MODE_PRIVATE)
var dexClassLoader = DexClassLoader(apkPath, file.absolutePath, null, ClassLoader.getSystemClassLoader())
```

> getDir()调用了Context的getDir()
>
> Retrieve, creating if needed, a new directory in which the application can place its own custom data files.  You can use the returned File object to create and access files in this directory.  Note that files created through a File object will only be accessible by your own application; you can only set the mode of the entire directory, not of individual files.

## 通过反射加载类，获取资源

```kotlin
var drawableClazz = dexClassLoader.loadClass("cf.android666.pluginapp.R\$drawable")
var onePng = drawableClazz.getDeclaredField("abc")
var onId = onePng.getInt(R.id::class.java)//反射获取资源id

var resources = getUninstallApkResource()//resource也是通过反射获取到
var drawable = resources.getDrawable(onId)
```

`AssetManager.addAssetPath()`方法是用来将apk等中的资源添加到`AssetManager`中，再通过其获取到`Resources对象`，这样就获取到未安装apk中的资源了。

```kotlin
fun getUninstallApkResource(): Resources {
    var assetManager = AssetManager::class.java.newInstance()
    var addAssetPath = assetManager.javaClass.getMethod("addAssetPath",String::class.java)
    addAssetPath.invoke(assetManager, apkPath)//设置了apkPath
    return Resources(assetManager, resources.displayMetrics, resources.configuration)
}
```

# 参考资源

[Android之Android apk动态加载机制的研究（二）：资源加载和activity生命周期管理 - lee0oo0 - 博客园  ](https://www.cnblogs.com/lee0oo0/p/3665066.html)

[Android逆向之旅---Android中的sharedUserId属性详解 - CSDN博客]( https://blog.csdn.net/jiangwei0910410003/article/details/51316688)


