---
title: Lambda表达式在kotlin中的应用
tags: lambda
abbrlink: '53875104'
date: 2018-02-09 11:09:06
---

# 一个例子

**普通写法**：

1. 定义一个接口OnClickListener

```kotlin
interface ClickListener{
    fun onClick(view: View)
}
```

1. 定义方法SetOnClickListener

```kotlin
fun setOnCLickListener(listener: ClickListener){
  this.listener = listener;
}
```

定义的方法和Java中写法类似，在使用该方法时也类似：

```kotlin
var testInterface = TestInterface()

testInterface.setOnCLickListener(object : TestInterface.ClickListener{
        override fun onClick(view: View) {
            TODO("not implemented") //To change body of created functions use File | Settings | File Templates.
        }
    })
```

**lambda写法**：

定义只需要一步：

```kotlin
//在初始化的时初始化listener
class AClass(var listener : (uri:String) -> Unit){...}

//或者直接定义这个变量
listener:((uri : String)-> Unit)? = null

//在需要用到方法时，listener的方法，比如onClickListener(){}
listener.invoke(agrs)
```

使用起来也更加简洁：

```kotlin
var t = TestInterface{ uri: String -> print(uri) }//获取对象的同时初始化listener
```

> 方法最后一个参数是lambda表达式时，lambda表达式的方法`{}`可以放到`()`的后面，如果只有这一个参数时，`()`也可以省略

当方法只有一个参数时，可以省略参数，还用`it`代替：

```kotlin
testInterface.setNewOnClickListener { print(it) }
```

甚至更加简洁，如果要执行的方法和listener定义的方法返回值类型相同，可以直接引用该方法：

```kotlin
testInterface.setNewOnClickListener(::print)
```



# lambda

lambda在Java8中引进，可以很好的替代匿名内部类，使代码更加简洁。

lambda表达式形式如下：

```kotlin
val sum = { x: Int, y: Int -> x + y }
```

> lambda 表达式总是被大括号括着， 完整语法形式的参数声明放在大括号内，并有可选的类型标注， 函数体跟在一个 `->` 符号之后。如果推断出的该 lambda 的返回类型不是 `Unit`，那么该 lambda 主体中的最后一个（或可能是单个）表达式会视为返回值。
>
> kotlincn.net [高阶函数和lambda表达式](http://www.kotlincn.net/docs/reference/lambdas.html#高阶函数和-lambda-表达式)

使用lambda的形式如下`() -> {}`,`()`内是参数，`{}`是函数具体的行为。

```java
//Java 8方式：
new Thread( () -> System.out.println("In Java8, Lambda expression rocks !!") ).start();
```

这个例子来自importNew.com,[Java8 lambda表达式10个示例](http://www.importnew.com/16436.html)

# 小知识点

* xx.map()

凡是**可迭代**的数据都可以使用`map()`函数

```kotlin
var args: Array<String> = arrayOf()
args.map {
    print(it)
}
```

还可以更简洁：

```kotlin
args.map(::print)
//::print表示引用该方法
```

* xx.flatMap()

返回**可迭代**的数组，可以和`xx.map()`一起使用

```kotlin
args.flatMap {
    it.split(" ") //把字符串按照" "切割
}.map{ 
    print("${it.toUpperCase()}") 
}
```

