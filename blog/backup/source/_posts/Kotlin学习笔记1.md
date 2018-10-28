---
title: Kotlin学习笔记1
tags: kotlin
abbrlink: 135b2780
date: 2018-02-09 16:18:01
---

这是我在学习Bennyhuo（ [github](https://github.com/enbandari/Kotlin-Tutorials) ）的kotlin入门视频时的一些笔记，比较偏基础，用于查缺补漏。

- xx.map() & xx.flatMap()

  xx.flatMap()用于返回**可迭代**的数组，而xx.map()则是任何**可迭代**数据都有的用来遍历的方法。

```kotlin
    var arr = arrayListOf<String>("a c v de  fb s e  gf d")
    arr.flatMap {
        it.split(" ")
    }.map{ 
        print("${it.toUpperCase()}") 
    }
```

- enum class 枚举类型

  分为有参和无参，枚举变量以`,`分隔，如果enum还有方法或者伴生对象，则最后一个变量后为`;`，否则可为`,`、`;`或者没有。

```kotlin
enum class City{
    UK,USA,EU;
    
    //以下为非必须代码，仅表示可以有这些功能
    fun say(){...}
    
    companion object{
        fun fun1(s:String):City{
            return valueOf(s.toUpperCase())
        }
    }
}
```

​        有参的情况如下

```kotlin
enum class Country(val aName:String){
    CHINA("中国"),
    JAPAN("日本"),
    USA("美国"),
    UK("英国");
    }
```



​        使用：通过enum的valueOf()方法获取枚举对象实例

```
var s = "uk"
var city = City.valueOf(s.toUpperCase())
//或者通过伴生对象：
var city = City.fun1(s)
```

- companion object伴生对象

  在类的定义，可以直接用`类名.方法名()`调用，相当于java中的静态方法

  一个类中只能有一个伴生对象

```kotlin
class xxx{
    ...
    companion object{
        fun parse(x: String): Country {
            return valueOf(x.toUpperCase())
        }
        ...
    }
}
```

- object修饰的类

  等同于只有一个实例的类，相当于java中的静态类，所有方法可以直接用类名调用

```kotlin
object ClassName{
    fun(){...}
}
```

- fun ClassName.funName()为类添加新的方法

  对于不能直接修改的类，有需要对其增加一个方法，可以自定义一个`ClassName.funName()`的方法来达到这个目的。

```kotlin
private fun Country.sayNum() {

    //this引用的是country对象
    var num = when (this) {

        CHINA -> 1
        JAPAN -> 2
        USA -> 3
        UK -> 4
    }
}
```

​        在使用时可以通过`Country`的对象调用`syaNum()`方法

- data class 数据类

  可以有方法，方便复制。

  必须至少有一个参数，并且参数都需要用var/val修饰

```kotlin
data class dataClass(var name: String, val age :Int)
```

- 文件读取
  - resource目录下的文件读取

```kotlin
    var input = File(ClassLoader.getSystemResource("input").path).readText()
```

- 与RxJava结合

  统计文本中字母个数，基于RxJava 1.2.1

```kotlin
    Observable.from(input.toCharArray().asIterable())
            .filter { !it.isWhitespace() }
            .groupBy { it }
            .map{
                o ->o.count().subscribe{
                    print("${o.key}-> $it  ,")
                }
            }
            .subscribe()
```

