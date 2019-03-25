---
title: Android多渠道打包知识
tags: android
abbrlink: a3a3dc4c
date: 2018-03-25 22:17:57
---

国内Android应用常常要分发到多个应用商店，使用Android Studio正确配置build.gradle与AndroidManifest.xml文件可以**一步打包多个渠道**。

本文实现的多渠道打包可实现不同渠道：

* 有不同的项目id（applicationId）
* 不同App名称（android:label）
* 不同App图标（android:icon）
* 等等



# 1.友盟配置

*具体配置请参考UMeng官方文档。

作为第三方统计平台，国内很多软件都使用的是Umeng的产品，故而大多数软件多渠道打包配置如下：

* 添加依赖

```java
../app/build.gradle
dependencies {
//友盟sdk
compile 'com.umeng.sdk:common:latest.integration'
compile 'com.umeng.sdk:analytics:latest.integration'
...}
```

* 修改AndroidManifest.xml

```xml
<application>
	...
	<!--友盟初始化appkey和channel-->
    <meta-data android:value="${APP_KEY}" android:name="UMENG_APPKEY"/>
    <meta-data android:name="UMENG_CHANNEL" android:value="${UMENG_CHANNEL_VALUE}" />
</application>
```

* 修改build.gradle

```java
android {

 productFlavors {
        beta {}
        baidu {}
        zhushou91  {} //不能以数字开头
        anzhi {}
    }

    productFlavors.all {

        flavor -> flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name
                                                 ,APP_KEY:umenInfo['APP_KEY']]
            //这里有一个知识点，用build.gradle读取properties文件信息，用于将部分信息统一放置在本地配置文件中，避免泄漏，若无此类要求可直接使用 APP_KEY:'da15d26d1a'等
    }

    //解决flavor Dimensions问题  http://blog.csdn.net/syif88/article/details/75009663
    flavorDimensions "versionCode"
}
//其他umeng要求的配置
```

这样编译完之后，通过通过build>Generate Signed APK...便可以打包不同渠道的apk，在友盟统计平台上统计各个渠道的App信息了。

# 2.Android Studio实现多渠道打包

方法1要求依赖umeng模块，使用场景难免有些受限，其实我们也可以自己实现多渠道打包，方法1使用的应该也是此原理。

* AndroidManifest.xml

在需要根据渠道不同而变化的地方使用`${KEY}`形式替换掉原先的值。

例如：

```xml
<application
android:label="${APP_NAME}"
...>
<meta-data android:name="APP_TEXT" android:value="${APP_TEXT}"/>//可以在java文件中获取到
```

* app/build.gradle

```java
    productFlavors {
        beta {applicationId = "cf.android666.mykotlin.beta"//每个渠道有不同的包名
            manifestPlaceholders = [APP_NAME : name ,APP_TEXT:'beta']
            }
        baidu {applicationId = "cf.android666.mykotlin.baidu"
            manifestPlaceholders = [APP_NAME:'A APP',APP_TEXT:'baidu']
          }

    }
```

* 在java中获取`meta-data`（非必须）

> Android获取Manifest中<meta-data>元素的值 - CSDN博客  https://blog.csdn.net/zhang31jian/article/details/29868235

```kotlin
//application中的meta-data
var appInfo = context.packageManager.getApplicationInfo(context.packageName,
        PackageManager.GET_META_DATA)
//service、receiver中的meta-data
var appInfo = context.packageManager.getServiceInfo(ComponentName(context,MService::class.java),
                PackageManager.GET_META_DATA)
var appName = appInfo.metaData.getString("APP_NAME")
```

# 3.生成多个渠道文件夹

还有一种方法，通过在项目中生成多个渠道的文件夹，在里面替换对应的资源文件，从而实现多渠道打包不同项目名，不同icon等等

* 在../app/src/目录下新建对应渠道文件夹，和main同级

* 在该渠道目录下新建对应的资源目录，在打包时自动替换对应资源

  ​

  目录树如下

```xml
src

--baidu

----res/drawable

--beta

--main

----res/drawable

```

# 4.More

此外还有美团的多渠道打包技术等

具体可参考文章：[美团Android自动化之旅—生成渠道包](https://tech.meituan.com/mt-apk-packaging.html)


<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android多渠道打包知识.md</div>"></iframe>
