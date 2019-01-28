---
title: Dagger 2 ❤️ Android
date: 2019-01-27 21:23:50
tag: dagger2
---

# 前言

[上篇文章](http://jixiaoyong.github.io/blog/posts/2822c354/)介绍了`Dagger 2` 的基本使用，本文跟随[官方文档](https://google.github.io/dagger/android)实践一下`Dagger 2 `在`Android`中的使用，可以看做是官方文档的不完全翻译。

本文有关`Dagger 2`的使用分为`Activity`和`Fragment`两部分，二者的使用几乎没有差别，最后介绍一下在Google官方Demo中学到的一个小技巧，可以将几乎所有的和`Dagger 2`的逻辑放到一份代码里面，对`Android`工程的影响极小。

首先要添加相关依赖（Kotlin环境）：

```gradle
apply plugin: 'kotlin-kapt'//引用该插件
implementation "com.google.dagger:dagger:$rootProject.dagger2Version"
implementation "com.google.dagger:dagger-android-support:$rootProject.dagger2Version"//Android特需
kapt "com.google.dagger:dagger-compiler:$rootProject.dagger2Version"//注意如果是kotlin语言，这里需要时#kapt#
```

# 在Activity中的使用

## Application范围内的@Component

首先创建整个应用程序使用的`@Component`，并将`AndroidInjectionModule`加入其中，实现`Inject`注入入口：

```kotlin
@Component(modules = [AndroidInjectionModule::class, MainActivityModule::class])
interface AppComponent {

    fun inject(application: MainApplication)
}
```

`AppComponent`的范围是整个应用程序都有效。

## 创建单个Activity的@Subcomponent

创建某个`Activity`专属的`@Subcomponent`，用于提供`AndroidInjector.Builder`。

```kotlin
@Subcomponent
interface MainActivitySubComponent : AndroidInjector<MainActivity> {

    @Subcomponent.Builder
    abstract class Builder : AndroidInjector.Builder<MainActivity>()

}
```

## 创建单个Activity的@Module

创建属于整个`Activity`的`@Module`，注意这里要指明`@subcomponents`为刚刚创建的`MainActivitySubComponent`。

```kotlin
@Module(includes = [MainActivityModule.InnerModule::class], subcomponents = [MainActivitySubComponent::class])
class MainActivityModule {

    @Provides
    fun bindWaitForInjectClass() = WaitForInjectClass()

    @Module
    abstract class InnerModule {
        @Binds
        @IntoMap
        @ClassKey(MainActivity::class)
        abstract fun bindInjectorFactory(builder: MainActivitySubComponent.Builder): AndroidInjector.Factory<*>//注意这里是 Factory<*>
    }
}

class WaitForInjectClass //一个供依赖注入的类
```

可以看到在`MainActivityModule`中提供了一个方法利用刚刚`MainActivitySubComponent`中提供的`MainActivitySubComponent.Builder`实例生成了一个`AndroidInjector.Factory`，而这个`Factory`就是我们后面要将`MainActivityModule`中的依赖实例通过`AppComponent`传递给`MainActivity`实例的关键。

> 此外还可以看到提供该Factory的方法是放到了另外一个抽象类里面然后再导入MainActivityModule中的，这是因为该方法的注解@Binds要求方法是抽象的，而MainActivityModule要是需要给Activity提供依赖实例所必须的@Provides又要求类不能是抽象的，否则就要求该方法是静态的。权衡之下我觉得这种方式是比较能接受的，当然也不排除有其他更优雅的解决方案，欢迎提[Issue](https://github.com/jixiaoyong/jixiaoyong.github.io/issues)告知。

然后，将`MainActivityModule`加入到应用程序的`@Component`——`AppComponent`中。

## 使Application继承自 [`HasActivityInjector`](https://google.github.io/dagger/api/latest/dagger/android/HasActivityInjector.html)

使当前`MainApplication`继承自`HasActivityInjector`，该接口只有一个方法：

```kotlin
/** Returns an {@link AndroidInjector} of {@link Activity}s. */
AndroidInjector<Activity> activityInjector();
```

这个类是用来为相应的`Activity`提供一个`AndroidInjector`。由于我们已经在`AppComponent`中包括了`AndroidInjectionModule`，所以`Dagger 2`已经可以自动为我们注入`DispatchingAndroidInjector`依赖，所以接下来的代码如下：

```kotlin
class MainApplication : HasActivityInjector, Application() {

    override fun onCreate() {
        super.onCreate()
        DaggerAppComponent.create().inject(this)
    }

    @Inject
    lateinit var dispatchActivityInjector: DispatchingAndroidInjector<Activity>

    override fun activityInjector() = dispatchActivityInjector//返回Dagger 2为我们注入的dispatchActivityInjector对象
}
```

在`onCreate()`方法中传入当前`Application`的依赖。

## 在Activity中使用自动注入依赖

做完了以上所有内容，我们只需要在`Activity`中添加如下代码就可以实现自动注入：

```kotlin
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var waitForInjectClass: WaitForInjectClass

    override fun onCreate(savedInstanceState: Bundle?) {
        AndroidInjection.inject(this)//注意这里，在super()之前调用
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_dagger2_x_android_main)

        Log.d("TAG", "The Class is $waitForInjectClass")
    }
}
```

以上所有代码如下，或者也可以[在这里找到](https://gist.github.com/jixiaoyong/db7e5f18b7106ce9cb36b61bf7134340)：

<script src="https://gist.github.com/jixiaoyong/db7e5f18b7106ce9cb36b61bf7134340.js"></script>

## 这一切是怎么实现的呢？

在`Android`程序运行时，`AndroidInjection.inject()`从`Application`中的`activityInjector()`方法获取到 `DispatchingAndroidInjector<Activity>` ，然后将`Activity`传入`inject(Activity)`。

`DispatchingAndroidInjector` 通过`AppComponent`找到我们在`MainActivityModule`提供的对应的`AndroidInjector.Factory`，然后创建了 `AndroidInjector` ——这就是我们当前`Activity`对应的`MainActivitySubComponent`。

接下来便按照之前的逻辑，从`MainActivitySubComponent`中查找提供`waitForInjectClass`的实例方法完成注入。

# 在Fragment中的使用

`Dagger 2`在`Fragment`的使用和在`Activity`中的使用十分相似。

通过之前的代码我们可以知道，其基本的原理依旧是利用`@Component`和`@subcomponent`，`@Module`之间的关联关系将`Application`和`Activity`等的依赖注入通过`AndroidInjector`关联起来的：

`MainActivitySubComponent`通过将`MainActivityModule`加入到`AppComponent`之中，然后当`MainActivity`之中需要使用到`MainActivitySubComponent`时，又通过`AndroidInjector`从`AppComponent`中拿到`MainActivityModule`中的`AndroidInjector.Factory`，通过该`Factory`和`MainActivitySubComponent`中的`Builder`产生关联，从而获取到了`MainActivitySubComponent`的实例供`Activity`使用。

在`Fragment`中我们也可以这样处理，只不过由于`Fragment`的特性，他的`@Module`不仅可以交给`Application`的`@Component`，也可以交给其他`Fragment`或者`Activity`的`@Component`，让其实现`HasFragmentInjector`即可，这取决于我们想要给`Fragment`绑定的依赖。

具体的实现一般分为下面几步：

* 创建`Application`的`@Component`并添加`AndroidInjectionModule`

* 创建实现了`AndroidInjector<MainFragment>`的`MainFragmentSubComponent`，其内部有方法提供`AndroidInjector.Builder<MainFragment>`

* 创建包含了提供`AndroidInjector.Factory<*>`的抽象方法的`MainFragmentModule`，指定其`subcomponents`为`MainFragmentSubComponent`；

* 将`MainFragmentSubComponent`加入到想要加入的类的`@Component`中，比如`AppComponent`类

* 在`Application`（如果上一步是`Activity`，则本步也是`Activity`等）中参照在`Activity`实现的步骤实现`HasFragmentInjector`

  上述完整的代码如下，或者也可以[在这里找到](https://gist.github.com/jixiaoyong/a24e76ca29f4c8062bf5c6a98529d252)：

  <script src="https://gist.github.com/jixiaoyong/a24e76ca29f4c8062bf5c6a98529d252.js"></script>

关于`Fragment`加入到`Activity`的Demo在官方文档有，这里就不再赘述了，其实只要掌握原理，其他用法的完全可以触类旁通。

# 一个小技巧

通过观察上面的两份代码，我们发现虽然这`Dagger 2`已经替我们做了好多事情，我们只需要在需要使用依赖注入的类中使用诸如`AndroidInjection.inject(this)`这样的代码就可以了，但是如果`Activity`、`Fragment`类过多的时候，这样的重复性工作仍然是个不小的工作量，万一有某处遗忘了便会导致出错。

这时就可以用到我在[Google官方示例代码](https://github.com/googlesamples/android-architecture-components/tree/master/GithubBrowserSample)中学到的一个小技巧了(针对本文中的例子做了一些修改)，或者你也可以[到这里查看源码](https://gist.github.com/jixiaoyong/9260c3ae2a70555e14f40c4b95364715)：

<script src="https://gist.github.com/jixiaoyong/9260c3ae2a70555e14f40c4b95364715.js"></script>



# 参考资料

[Dagger 2 官方文档 Android篇](https://google.github.io/dagger/android)

[Google官方示例代码——GithubBrowserSample](https://github.com/googlesamples/android-architecture-components/tree/master/GithubBrowserSample)