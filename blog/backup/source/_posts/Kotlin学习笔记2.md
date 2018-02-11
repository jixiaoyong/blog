---
title: Kotlin学习笔记2
date: 2018-02-11 11:38:43
tags: kotlin
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

