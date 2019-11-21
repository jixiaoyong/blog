---
title: 从Sunflower开始学习优雅的Jetpack架构
date: 2019-01-24 19:48:31
tag: android
---

> Google大法NB！！！(破音)

# 前言

[Jetpack](https://developer.android.google.cn/jetpack/)是Google推出的一系列Android软件集合，*"使您可以更轻松地开发出色的 Android 应用。这些组件可帮助您遵循最佳做法、让您摆脱编写样板代码的工作并简化复杂任务，以便您将精力集中放在所需的代码上"*。

[Sunflower](https://github.com/googlesamples/android-sunflower)则是Google用来演示如何使用Jetpack进行Android开发的Demo，有着非常优雅的架构与十分简洁的代码，可以帮助我们很好地学习Jetpack以及MVVM思想。

本文主要是结合Sunflower中的示例代码，分析Jetpack架构中各部分的作用，以及他们如何巧妙的搭配使用，方便指导日后对Jetpack的使用。

> 本文中的大部分代码、示意图除非特殊注明外，皆来自Google的[Sunflower工程](https://github.com/googlesamples/android-sunflower)或其他互联网资源，根据篇幅需要做了部分精简，所有权益归原作者所有。

下图是[Google Jetpack官网](https://developer.android.google.cn/jetpack/)对Jetpack的介绍图：

<center>     <img style="border-radius: 0.3125em;     box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"      src="https://jixiaoyong.github.io/images/jetpack_donut.png">     <br>     <div style="color:orange; border-bottom: 1px solid #d9d9d9;     display: inline-block;     color: #999;     padding: 2px;">Jetpack示意图 <font style="color: #BEBEBE">来自GoogleJetpack官网</font></div> </center>
# 对Sunflower的整体分析

下图是Sunflower架构的简单示意图：

![](https://jixiaoyong.github.io/images/20190124212220.png)

可以看到，APP的界面有`我的花园`、`植物目录`和`植物介绍`三部分，这三者的切换逻辑通过<font color=#0288d1 size=4>**`Navigation`**</font>实现。

每个界面的**`XML`**中的布局信息（包括`数据、事件（clickListener等），RecycleView的LayoutManager，Adapter等等`）通过<font color=#0288d1 size=4>**`DataBinding`**</font>与<font color=#0288d1 size=4>**`ViewModel`**</font>中的可观察数据<font color=#0288d1 size=4>**`LiveData`**</font>绑定在一起，只要数据库中的`数据`有更新，就会通过`LiveData`主动通知布局更新界面；同时`DataBinding`还通过与`Adapter`（这些继承自**`ListAdapter`**的Adapter实现了<font color=#0288d1 size=4>**`Paging`**</font>的作用）将`ItemView`的`ViewModel`与布局`XML`中绑定在一起，通过`BindingAdapter`对`XML`中的数据做预处理（加载imgUrl中的图片到ImageView等等）。

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

先看一下[Navigation](https://developer.android.google.cn/topic/libraries/architecture/navigation/)的定义：

> Navigation是APP设计中的关键部分，可以用来定义用户从不同的界面切换、进入和推出的交互逻辑。

和布局文件一样，我们可以在编译器的可视化界面中，直接预览、设计不同界面切换效果。他可以负责`Fragment`、`Activity`、`Navigation graphs` 与 `subgraphs` 以及`Custom destination types`，他们之间通过不同的`action`连接起来。

通过官方文档可知，`Navigation`可以和`AppBar`，`ToolBar`等组合起来控制Fragment显示，此外可以通过`ViewModel`在绑定到同一个`Activity`的`Fragment`之间共享数据，或者也可以通过[`Bundle`或`Safe Args`](https://developer.android.google.cn/topic/libraries/architecture/navigation/navigation-pass-data)在两个`Fragment`之间传递数据。

那么，在`Sunflower`中`Navigation`是怎么控制界面切换的呢？

首先，在`res/navigation/`目录下面新建一个`嵌套导航图(Nested navigation graphs)`,定义各个界面之前的切换关系：

```xml
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    app:startDestination="@+id/garden_fragment">
//app:startDestination定义了在这个导航图中首次启动展示的界面
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
            app:argType="string" />//参数类型小写
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

## DataBinding绑定布局和数据

Navigation解决了不同的布局间交互的逻辑，DataBinding则充当布局View和数据（ViewModel、LiveData）之间的桥梁，将二者联系起来。

从[官网](https://developer.android.google.cn/topic/libraries/data-binding/)的表述中我们知道，DataBinding使用在XML中声明的方式（而非编程的方式），将布局中的组件捆绑到APP中使用到的数据上，这样当数据更新时，布局也会随之自动更新。

DataBinding在XML中的形式如下：

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto">
    <data>
        <variable
            name="viewmodel"
            type="com.myapp.data.ViewModel" />
    </data>
    <ConstraintLayout... /> <!-- UI layout's root element -->
</layout>
```

需要注意的是原先的页面布局信息`<ConstraintLayout... />`包裹在`<layout... />`中，同时多了一个数据域`<data... />`，我们可以在其中定义一些变量`<variable... />`，并在布局中使用：

```xml
<TextView
    android:text="@{viewmodel.userName}" />
```

除了常见的`android:text`，`android:onClick`等通用的属性可以直接绑定外，我们还可以通过自定义[**Binding adapters**](https://developer.android.google.cn/topic/libraries/data-binding/binding-adapters.html)支持更多形式的属性绑定：

```kotlin
@BindingAdapter("app:goneUnless")
fun goneUnless(view: View, visible: Boolean) {
    view.visibility = if (visible) View.VISIBLE else View.GONE
}
```

上面的代码就支持了`app:goneUnless`的解析，我们只要在XML中为组件加上这个属性就可以实现相应的效果：

```xml
<TextView
          android:text="@{viewmodel.userName}"
          app:goneUnless="@{viewmodel.isGone}"/>
```

最后，我们需要在对应的Activity或Fragment中，用如下代码将布局与页面绑定到一起：

```kotlin
//setContentView(R.layout.activity_main)
val binding: ActivityMainBinding = DataBindingUtil.setContentView(
            this, R.layout.activity_main)
binding.viewmodel = ...
```

这里的`ActivityMainBinding`类是`DataBinding`根据XML文件的名字自动替我们生成的，规律是`XML文件名+Binding`的驼峰命名。

在Sunflower中有类似的应用有很多处：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--
  ~ Copyright 2018 Google LLC ...
  -->
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
                name="hasPlantings"
                type="boolean" />
    </data>

    <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent">

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/garden_list"
                app:isGone="@{!hasPlantings}"
                app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"
                tools:listitem="@layout/list_item_garden_planting"/>

    </FrameLayout>

</layout>
```

## ViewModel管理数据与页面的交互

`DataBinding`通过标记的形式将数据和组件绑定，在这个过程中他使用的数据则是来自于`ViewModel`的。在页面`Activity`(或`Fragment`)中，我们可以处理这两者之间的关系。

`ViewModel`是设计用来以一种可以感知生命周期（`lifecycle`）的方式存储和管理与UI相关的数据，它可以允许数据在诸如屏幕旋转的变化中存活下来，也就是说`VideModule`的数据生命周期可能要比他附着的`Activity`或`Fragment`的生命周期长。

同时，`UI controller`可以在`Activity`等不再需要数据时，自动调用`ViewModel`的`onCleared()`方法清除这些数据以避免内存泄漏。

下图是`ViewModel`和`Activity`的生命周期对比：

<center>     <img style="border-radius: 0.3125em;     box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"      src="https://jixiaoyong.github.io/images/20190202161443.png">     <br>     <div style="color:orange; border-bottom: 1px solid #d9d9d9;     display: inline-block;     color: #999;     padding: 2px;">ViewModel和Activity的生命周期对比图：左图Activity先经历了一次旋转，然后finish，右边是与此相关的ViewModel的生命周期<br/><font style="color: #BEBEBE">来自GoogleJetpack官网</font></div> </center>
此外，由于默认的获取ViewModel的方法只能调取无参构造函数，当需要向ViewModel传递参数时，就需要用到Factory工厂模式来创建ViewModel：

```kotlin
val viewModel = ViewModelProviders.of(this, factory).get(GardenPlantingListViewModel::class.java)

class PlantDetailViewModelFactory(args:Any) : ViewModelProvider.NewInstanceFactory() {...}
```

还可以将`ViewModel`于`LiveData`结合，这样在`Activity`等地方对`LiveData`进行订阅后，当`LiveData`的值发生变化时`Activity`等可以及时得到通知，而做出相应变化。此外`ViewModel`与`lifecycle`的结合可以保证在`Activity`等生命周期结束后数据得到及时的清理。

## Room保存数据

> Room持久性库在SQLite上提供了一个抽象层，以便在充分利用SQLite强大功能的同时，能够流畅的访问数据库。——Android Developers

`Room`需要3个元素：

* `Database` 数据库，可以提供对表格的操作方法`@DAO`。是一个继承自`RoomDatabase`的抽象类。
* `Entity` 表格，规定了每个表格可以保存的数据格式。是一个普通类。
* `Dao` 数据访问结构（`Data Access Object`），定义了对表格`@Entity`中的数据的操作。是一个接口或者抽象类。

此外，还可以对`@DAO`进行进一步的封装得到一个`XXXRepository`类，`ViewModel`通过这个`XXXRepository`类来操作数据，从而将其与数据的具体实现解耦。

## WorkManager管理任务

`WorkManager`用来管理即时或定时任务，官方定义是在指定约束条件成熟时可靠的在后台执行对应的任务。

具体使用可以参考这个[GIST](https://gist.github.com/jixiaoyong/041d8b0775e392302b4cd57a98b4f6fa)。

和他相关的有下面几个关键类：

* `Worker` 定义要执行的任务内容
* `WorkRequest` 代表一项单独的任务，明确具体要执行的任务内容（Worker）、任务的类型（WorkRequest.Builder的子类，决定任务一次性还是重复的）以及任务执行的条件（Constraints，如联网、电池电量等等）
* WorkManager 执行管理WorkRequest，安排执行Worker中的工作内容。

# 参考资料

[Android Jetpack官网](https://developer.android.google.cn/jetpack/)

[Android Jetpack-Navigation 使用中参数的传递](https://blog.csdn.net/weixin_42215792/article/details/80395379)
