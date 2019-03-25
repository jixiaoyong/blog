---
title: Dagger 2 从0到1之旅
date: 2019-01-26 15:16:17
tag: dagger2
---

#  前言

`Dagger 2`是Google维护的一款可用于`Java`和`Android`的依赖注入框架。

本文主要是简单梳理`Dagger 2`中各个注解的作用，以及其简单用法，不涉及具体项目应用。

先解释几个概念：

* **`依赖注入`**：是一个对象（或静态方法）给另一个对象提供依赖的技术。

* **`依赖`**是可以使用的对象（`Service`），而把依赖提供给使用该依赖的对象（`Client`）的过程叫做`注入`。

例如，下面这段代码中`Service`就是`Client`的依赖。：

```kotlin
class Service

class Client(){
    init {
        val service = Service()
    }
}
```

但是如果每个依赖都这样写的话，如果`Service`类的构造方法有变更，就需要同时也更改`Client`对应的方法，这样深耦合的代码显然不是我们需要的。

`Dagger 2 `就是为了帮助我们解决这个问题，在使用它之后，`Client`类的代码只需要这样写成类似下面这样（示例代码）：

```kotlin
class Client{

    lateinit var service: Service

    init {
        //TODO 某个将Service依赖注入的方法 magicFun()
        val newService = service//使用Service的实例service
    }
}
```

可以看到，这时`Service`的实例化过程被移到了`Client`的外部某处，这样如果`Service`构造方法有更新时，我们只需要统一去修改`magicFun()`中对应的代码即可。

那么这一切`Dagger 2`到底是如何实现的呢？

# Dagger 2 具体实现

## @Inject

首先需要请出第一个主角——**`@Inject`**。

在`Dagger 2`中，`@Inject`主要做两件事❶标记依赖类的构造方法；❷标记需要框架自动实例化的对象：

```kotlin
class Service @Inject constructor()//❶标记依赖类的构造方法

class Client{

    @Inject
    lateinit var service: Service//❷标记需要框架自动实例化的对象
	...
}
```

这样`Dagger 2 `就知道了有个对象需要它来帮助我们注入，同时也知道了有一个构造方法来实例化`Service`对象。但这时如何将二者联系起来呢？

## @Component

这就要提到第二个主角——**`@Component`**。

`@Component`标记的类是将一个类和他的依赖联系在一起的桥梁，通常是一个**抽象类或者接口**：

```kotlin
@Component
interface ClientComponent{
    fun inject(client: Client)
}
```

至此，`Client`和`Service`通过`ClientComponent`联系在一起，在使用时只需要将`Client`的引用传入即可：

```kotlin
    init {
        //方式❶ DaggerClientComponent.create().inject(this)
        //方式❷ DaggerClientComponent.builder().build().inject(this)
        val newService = service
    }
```

以上完整的代码可以参考这里,[若无法显示可点击这里查看](https://gist.github.com/jixiaoyong/8abd44901a2816e2b6722566bb8f08d8#file-dagger2_basic_guide_line_part1-kt)：

<script src="https://gist.github.com/jixiaoyong/8abd44901a2816e2b6722566bb8f08d8.js"></script>

到目前为止，对于我们自己定义的类，我们只需要使用`@Inject`标记其构造方法，然后再在使用该类的时候使用`@Inject`标记该对象，在需要使用该对象的地方通过`@Component`类传入使用该依赖的类的引用即可。

<center> <img style="border-radius: 0.3125em; box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://jixiaoyong.github.io/images/20190126184230.png"> <br> <div style="color:orange; border-bottom: 1px solid #d9d9d9; display: inline-block; color: #999; padding: 2px;">@Component自动实现依赖注入的示意图</div> </center>

但是很显然实际开发中，不是所有的`Service`类都可以被我们随意修改，如果`Service`类是第三方提供的类，显然我们是无法用`@Inject`修饰其构造函数的。

## @Module和@Provides

为了解决第三方依赖的问题，我们要引入另外两个主角——**`@Module`**和**`@Provides`**。

`@Provides`用来提供一个方法，我们可以在其内部实例化并返回`Service`类，这样子当用到`Service`的时候，`@Component`类只需要找到`@Provides`提供的这个方法，并获取到他实例化好的`Service`对象注入到`Client`中就可以了。

`@Module`则是提供一个**类**（注意是类，而非接口），像一个袋子一样把`@Provides`提供的方法“装”到一起，打包提供给`@Component`类。

```kotlin
@Module
class ClientModule{

    @Provides
    fun getService() = Service()
}

@Component(modules = [ClientModule::class])//Component可以有多个Module类
interface ClientComponent{
    fun inject(client: Client)
}
```

上述代码中的`@Component(modules = [ClientModule::class])`将装有可以产生依赖的`@Provides`方法的“大袋子”`@Module`和“桥梁”`@Component`关联到了一起。

`@Component`在产生依赖的时候会先到`@Module`类中的`@Provides`方法中查找；如果找不到才会再到`@Inject`中查找。（也就是说，此时`Service`类的`@Inject`构造方法其实是失效了的，完全可以没有`@Inject`注解——第三方类即是如此）。

上述完整代码如下,[若无法显示可点击这里查看](https://gist.github.com/jixiaoyong/35714a6287617260d8578c1b716427cb)：

<script src="https://gist.github.com/jixiaoyong/35714a6287617260d8578c1b716427cb.js"></script>

解决了第三方依赖引用的问题，还有一个非常重要的问题——我们使用的绝大多数类肯定不止一个构造方法，那么假设依赖类`Service`现在有两个构造方法，我们需要分别这两个构造方法，这种情况又该怎么处理呢？

```kotlin
class Service @Inject constructor(var string: String = "default")
```

很明显，这时候`@Inject`注解已经没用了，一个类只能有一个构造方法被`@Inject`修饰，否则会报错：`错误: Types may only contain one @Inject constructor`。

去掉`@Inject`后`Service`类变成如下：

```kotlin
class Service(var string: String = "default")
```

尝试在`@Module`中添加另外一个`@Provides`方法使用另外一个带参构造函数：

```kotlin
@Module
class ClientModule{

    @Provides
    fun getService() = Service()

    @Provides 
    fun getServiceWithArgs() = Service("Args")
}
```

运行时发现会出错，因为有两个方法都可以提供`Service`，`@Component`产生了迷失，不知道用哪一个好，导致错误。

## @Named和@Qualifier

为了解决多个构造函数导致的问题，这时就需要第五个主角**`@Named`**以及幕后英雄**`@Qualifier`**

首先，上述问题的解决方案是在另外一个方法上加一个注解`@Named`，表示他是一个特殊的方法：

```kotlin
    @Provides @Named("Args")
    fun getServiceWithArgs() = Service("Args")
```

当在`Client`中想使用这个方法的依赖时：

```kotlin
    //@field:是kotlin中注解字段特别需要的，在Java中可以直接写成@Named("Args")
    @Inject
    @field:Named("Args")
    lateinit var service: Service
```

查看`@Named`源码：

```java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {

    /** The name. */
    String value() default "";
}
```

发现`@Qualifier`才是他实现标识限定符注解（Identifies qualifier annotations）的力量之源。查看`@Qualifier`注解可以知道，我们也可以自定义基于`@Qualifier`的注解来实现和`@Named`完全一致的功能。

```kotlin
@Qualifier
@MustBeDocumented
@kotlin.annotation.Retention(AnnotationRetention.RUNTIME)
annotation class YourQualifierName(//YourQualifierName可以是任意你喜欢的名字
    /** The name.  */
    val value: String = ""
)
```

之后我们就可以使用`@YourQualifierName`替代`@Named`实现标识不同注解的作用，从而支持有多个构造函数的`Service`类的初始化。

上述完整代码如下,[若无法显示可点击这里查看](https://gist.github.com/jixiaoyong/fec4426183328346b955f62eb6e9c91a)：：

<script src="https://gist.github.com/jixiaoyong/fec4426183328346b955f62eb6e9c91a.js"></script>

`@Component`可以有多个`@Module`，他们之间的关系可以用下图表示：

<center> <img style="border-radius: 0.3125em; box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://jixiaoyong.github.io/images/20190126201436.png"> <br> <div style="color:orange; border-bottom: 1px solid #d9d9d9; display: inline-block; color: #999; padding: 2px;">@Component 会先到他拥有的多个@Module中去查找Service类</div> </center>

## @Singleton和@Scope

在实际开发中，我们需要有的类只能有一个实例，从而在不同的地方共享一些数据——即单例，这种情况就需要另外一个角色`@Singleton`和他的幕后英雄`@Scope`。

@Singleton是用来标记类在其范围内只能被实例化一次。

```kotlin
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

通过查看其源码可以知道其背后是`@Scope`在起作用，`@Scope`的作用是限定其修饰的类的范围，适用于有可注入的构造函数并且包含控制类型实例如何重用的类。没有`@Scope`修饰的实例在构造完毕后就会失去控制，不再关心后续的发展（*then forgets it*），而`@Scope`修饰的类会在实例构造完毕后，继续保留一遍下一次可能的复用，当有多个线程可以访问该实例时，他的实现应该是线程安全的（*it‘s implementation should be thread safe*）。

此外`@Component`应该和他所包含的`@Module`的`@Provides`的`@Scope`范围一致：

```kotlin
@Singleton
@Component(modules = [ClientModule::class])
interface ClientComponent{
    fun inject(client: Client)
}
@Module
class ClientModule{

    @Provides
    fun getService() = Service()

    @Singleton
    @Provides @Choose("Args")
    fun getServiceWithArgs() = Service("Args")
}
```



此外两个关系为**`dependencies`**的`@Component`可以分别拥有相同名称的`@Inject`、`@Module`、`@Provides`而不会被*merge*，两者可以相互访问。

而**`subcomponents`**则不能和`@Component`有以上相同的项。

> `Subcomponent`从它的父类访问所有依赖
>
> `@Component`只能访问在基类`@Component`接口暴露的公共性的依赖
>
> ——[Subcomponents和Component Dependencies——Sinyuk Blog](https://sinyuk.me/2016/04/11/Subcomponents%E5%92%8CComponent-Dependencies/)

他们之间的关系可以表示为下图：

<center> <img style="border-radius: 0.3125em; box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://jixiaoyong.github.io/images/20190127204658.png"> <br> <div style="color:orange; border-bottom: 1px solid #d9d9d9; display: inline-block; color: #999; padding: 2px;">@Subcomponent,@Component之间的关系</div> </center>



# 参考资料

[android-cn：依赖注入—— Github](https://github.com/android-cn/blog/tree/master/java/dependency-injection)

[Dependency Injection ——wikipedia](https://en.wikipedia.org/wiki/Dependency_injection)

[Elye的Dagger 2 系列](https://medium.com/@elye.project)

[Dagger 2 官方手册](https://google.github.io/dagger/users-guide)

[Android - Dagger2使用详解——简书](https://www.jianshu.com/p/2cd491f0da01)

[Subcomponents和Component Dependencies——Sinyuk Blog](https://sinyuk.me/2016/04/11/Subcomponents%E5%92%8CComponent-Dependencies/)



<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Dagger2从0到1之旅.md</div>"></iframe>