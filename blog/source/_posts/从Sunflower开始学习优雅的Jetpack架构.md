---
title: 从Sunflower开始学习优雅的Jetpack架构
date: 2019-01-24 19:48:31
tag: android
---

> Google大法NB！！！

# 前言

[Jetpack](https://developer.android.google.cn/jetpack/)是Google推出的一系列Android软件集合，*"使您可以更轻松地开发出色的 Android 应用。这些组件可帮助您遵循最佳做法、让您摆脱编写样板代码的工作并简化复杂任务，以便您将精力集中放在所需的代码上"*。

[Sunflower](https://github.com/googlesamples/android-sunflower)则是Google用来演示如何使用Jetpack进行Android开发的Demo，有着非常优雅的架构与十分简洁的代码，可以帮助我们很好地学习Jetpack以及MVVM思想。

本文主要是结合Sunflower中的示例代码，分析Jetpack架构中各部分的作用，方便指导日后对Jetpack的使用。

<center>     <img style="border-radius: 0.3125em;     box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"      src="https://jixiaoyong.github.io/images/jetpack_donut.png">     <br>     <div style="color:orange; border-bottom: 1px solid #d9d9d9;     display: inline-block;     color: #999;     padding: 2px;">Jetpack示意图</div> </center>

# 对Sunflower的整体分析

下图是Sunflower架构的简单示意图：

![](https://jixiaoyong.github.io/images/20190124212220.png)

可以看到，APP的界面有`我的花园`、`植物目录`和`植物介绍`三部分，这三者的切换逻辑通过<font color=#0288d1 size=4>**`Navigation`**</font>实现。

每个界面的**`XML`**中的布局信息（包括`数据、事件（clickListener等），RecycleView的LayoutManager，Adapter等等`）通过<font color=#0288d1 size=4>**`DataBinding`**</font>与<font color=#0288d1 size=4>**`ViewModel`**</font>中的可观察数据<font color=#0288d1 size=4>**`LiveData`**</font>绑定在一起，只要数据库中的`数据`有更新，就会主动通知布局更新界面；同时`DataBinding`还通过与`Adapter`（这些继承自**`ListAdapter`**的Adapter实现了<font color=#0288d1 size=4>**`Paging`**</font>的作用）将`ItemView`的`ViewModel`与布局`XML`中绑定在一起，通过`BindingAdapter`对`XML`中的数据做预处理（加载imgUrl中的图片到ImageView等等）。

在<font color=black size=5>**`View`**</font>中指定这些`DataBinding`与<font color=black size=5>**`ViewModel`**</font>之间以及`ViewModel`与<font color=black size=5>**`Model`**</font>`数据库`之间的逻辑关系，这些数据与操作都受着<font color=#0288d1 size=4>**`Lifecycle`**</font>的影响。

`ViewModel`的数据来源——`Model`在这里的实现是一个`数据库`。每个`ViewModel`有一个`XXXViewModelFactory`类，用来使用数据类`XXXRepository`类的实例创建对应的`ViewModel`。`XXXViewModelFactory`向`Activity`等屏蔽了`ViewModel`的具体实现。

`XXXRepository`类的出现时为了将`ViewModel`与数据的具体实现解耦合，这样`ViewModel`只需要关心他要的操作而不必关系数据来源的具体实现。在本例中，`XXXRepository`类对应封装了这数据库`AppDatabase`中对两个表的操作。

数据库使用<font color=#0288d1 size=4>**`Room`**</font>实现，从底层开始依次分为`表Entity`，`数据访问对象DAO`和`数据库DataBase`三个层次。每个DAO对应一个包装类`XXXRepository`类供`ViewModel`使用。

```
@Entity    GardenPlanting    //表，定义了存储的数据项及其格式
@Dao    GardenPlantingDao    //数据访问对象，定义了例如插入数据、查询数据等操作
 GardenPlantingRepository    //对DAO的封装，将数据的的具体实现与ViewModel对数据的操作解耦
@Database     AppDatabase    //数据库，包括表和对表的操作
```

<font color=#0288d1 size=4>**`WorkManager`**</font>则管理着一个从Json读取数据并加载到数据库中的后台任务`SeedDatabaseWorker`。

# 具体实现分析

首先看一下**`View`**部分，Sunflower只有简单的3个页面，全都是用`Fragment`实现，由`Activity`通过`Navigation`控制切换：

* `GardenActivity` 主页面，唯一的一个Activity
* `GardenFragment` 我的花园 界面，会显示用户在植物目录中选择并种植的植物信息
* `PlantListFragment` 植物目录 界面，所有的植物信息列表
* `PlantDetailFragment` 植物介绍 界面，当在“我的花园”或“植物列表”选择了某个植物后，会进入该界面显示植物详细介绍

## Navigation控制界面切换

那么`Navigation`是怎么控制界面切换的呢？

首先，在`res/navigation/`目录下面新建一个`嵌套导航图(Nested navigation graphs)`,定义各个界面之前的切换关系：

```xml
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    app:startDestination="@+id/garden_fragment">
//startDestination定义了在这个导航图中首次启动展示的界面
    <fragment
        android:id="@+id/garden_fragment"
        android:name="com.google.samples.apps.sunflower.GardenFragment"
        android:label="@string/my_garden_title"
        tools:layout="@layout/fragment_garden">
//action定义了在各个界面的切换关系
        <action
            android:id="@+id/action_garden_fragment_to_plant_detail_fragment"
            app:destination="@id/plant_detail_fragment"
            app:enterAnim="@anim/slide_in_right"//enterAnim等指定执行action时的动画
          .../>
    </fragment>

    <fragment
        ...>
//argument定义了在切换界面时需要带的参数，需要androidx.navigation.safeargs的支持,具体见参考资料-Android Jetpack-Navigation 使用中参数的传递
        <argument
            android:name="plantId"
            app:argType="string" />
    </fragment>
</navigation>
```

然后在Activity对应的XML中插入该导航：

```xml
<LinearLayout ...>
	<fragment
        android:id="@+id/garden_nav_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:defaultNavHost="true"
        app:navGraph="@navigation/nav_garden" />

</LinearLayout>
```

之后就可以在Activity或者Fragment中获取该导航的实力，用来切换界面了：

```kotlin
//activity
val navController = Navigation.findNavController(this, R.id.garden_nav_fragment)
//fragment或其他地方
val direction = GardenFragmentDirections//嵌套导航图中Fragment自动生成的类
.ActionGardenFragmentToPlantDetailFragment(plantId)
it.findNavController().navigate(direction)
```











# 参考资料

[Android Jetpack官网](https://developer.android.google.cn/jetpack/)

[Android Jetpack-Navigation 使用中参数的传递](https://blog.csdn.net/weixin_42215792/article/details/80395379)