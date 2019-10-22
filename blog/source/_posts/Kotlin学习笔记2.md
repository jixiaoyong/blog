---
title: Kotlin学习笔记2
tags: kotlin
abbrlink: 8a52763a
date: 2018-02-11 11:38:43
---

- 尾递归优化

  把递归通过编译器转化为迭代，从而避免Stack Overflow

  “以时间换取空间”

  普通递归：

  调用函数之后，还需要使用其返回值供自己使用，即自身返回值依赖于下一级函数，一般是调用自身的代码后面，还有其他的代码要执行。

```kotlin
fun fun1(n: Int): BigInteger {

    if (n == 0) return BigInteger.valueOf(1L)

    return n.toBigInteger().times(fun1(n - 1))

}
```

​	尾递归：

​	调用自身之后，无需再返回当前函数,将处理结果以其他形式返回。

​	普通递归和尾递归都存在栈溢出风险（未优化前，例子中的函数计算10000到100000的阶乘时会溢出），kotlin提供了一种尾递归优化的方法——`tailrec`，使得编译器在编译时将递归转化为迭代，从而避免栈溢出。

```kotlin
data class Result(var value: BigInteger = BigInteger.valueOf(1L))

//尾递归，tailrec为kotlin中优化关键字
tailrec fun fun2(n: Int, m: Result) {
    if (n == 0) {
        m.value = m.value.times(BigInteger.valueOf(1L))
        return
    } else {
        m.value = m.value.times(n.toBigInteger())
        fun2(n-1,m)
    }
}
```

​	本例中传入`fun2()`的`Result`实例保存了计算结果

- sealed class 密封类

  密封类的所有子类必须在一个文件(xx.kt)中，他的子类是有限的，所以当`when()`的时候不需要`else`。

  某种意义上他们像是一种`enum class`，只不过他的子类可以有多个实例。

  > Sealed classes are used for representing restricted class hierarchies, when a value can have one of the types from a limited set, but cannot have any other type. They are, in a sense, an extension of enum classes: the set of values for an enum type is also restricted, but each enum constant exists only as a single instance, whereas a subclass of a sealed class can have multiple instances which can contain state.

```kotlin
//sealed class
sealed class Player{

    class Play(var arg:String) : Player()

    object Stop : Player()
}

class p2():Player()
```

* kotlin抛出异常

```kotlin
@Throws(RemoteException::class)
fun getBookList():List<Book>
```

* kotlin中的泛型

  `out` 协变，使用子类泛型的对象可以赋值给使用父类泛型的对象，相当于`extend`，用于方法的返回值（生产者）时使用

  `in` 逆变，使用父类泛型的对象可以赋值给使用子类泛型的对象，相当于`super`，用于方法的参数（消费者）时使用

  不变，当泛型即当消费者，又当生产者时，不用`in`或者`out`

```kotlin
fun main(args: Array<String>) {
    val from = arrayOf(1,2,4)
    val to = arrayOf(Any())

    copyArray(from,to)
}

fun copyArray(from: Array<out Any>, to: Array<in Int>) {
    //这里的from被out修饰，只能作为生产者调用get之类的方法，不能作为消费者调用set之类的方法
    //...
}
```

* 星号投射

> 你对类型参数一无所知，但仍然希望以安全的方式使用它。

**安全的使用**，则表示该类`Group<T>`满足

> 1.子类至少接收和父类一样范围的参数 >=  ---> 父类入参为Noting 不能安全写入
>
>  2.子类最多返回和父类一样范围的参数 <=  ---> 父类出参为Any? 可以安全读取

则有以下三种实现方式

![in-out-star-projection-approaches](https://jixiaoyong.github.io/images/20191022194750.png)

其中：

`Group<in Noting>` 的`fetch()`方法一直返回`Any?`

`Group<out Any?>` 的`T`需要与实际的`Group`的`T`保持一致，否则会报错

`Group<*>` 既能`insert`正确返回对应的类型，也不用实时修改

```kotlin
object TClass {

    fun readIn(group: Group<in Nothing>) {
        val d = group.fetch()
    }

    fun readOut(group: Group<out Animal>) {
        val d = group.fetch()
    }

    fun read(group: Group<*>) {
        val d = group.fetch()
    }

}
interface Group<T : Dog> {
    fun insert(member: T): Unit
    fun fetch(): T
}
```



* 参考资料

[Star-Projections and How They Work](https://typealias.com/guides/star-projections-and-how-they-work/)

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Kotlin学习笔记2.md</div>"></iframe>