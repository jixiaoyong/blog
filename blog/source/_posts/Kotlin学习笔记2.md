---
title: Kotlin学习笔记2
tags: kotlin
abbrlink: 8a52763a
date: 2018-02-11 11:38:43
---

# 尾递归优化

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

# sealed class 密封类

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

# kotlin抛出异常

```kotlin
@Throws(RemoteException::class)
fun getBookList():List<Book>
```

# kotlin中的泛型

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

# 星号投射

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



# 委托

委托是将重复出现的代码放到一个地方。

![委托示意图](https://jixiaoyong.github.io/images/20191023194526.png)

* 类委托 ：

```kotlin
interface Interface {	fun a()	}
class A : Interface
class B(a: Interface) : Interface by a
```

这样B便可以将`Interface`中方法的实现委托给类`A`的对象`a`

* 委托属性：

将同一类型的属性的`get`、`set`方法放到一个地方实现，可以在加入`其它操作`

```kotlin
class Delegate {
    var name: String = ""

    operator fun getValue(clazz: Any?, property: KProperty<*>): String {
        println("get()")//其它操作
        return name
    }

    operator fun setValue(clazz: Any?, property: KProperty<*>, t: String) {
        println(" set()")
        name = t
    }
}

class ClassA {
    var name: String by Delegate()
    var age: String by Delegate()
}
```

* 委托类的初始化函数：

```kotlin
fun <T> delegate(initializer: () -> T) = Delegate(initializer)

class MyClass1 {
    var name: String by delegate {
        println("MyClass1.name init")
        "MyClass1"
    }
}

class Delegate<T>(initializer: () -> T) {
    operator fun getValue(myClass1: T, property: KProperty<*>): String {
        println("$className get()")
        return name
    }

    operator fun setValue(myClass1: T, property: KProperty<*>, t: String) {
        println("$className set()")
        name = t
    }

    var name: String = ""
    var className = initializer()
}
```

* Map委托：

将类的`属性`名称和`map`中的`key`一一对应，从而将对于`value`赋值给`属性`

```kotlin
class ClassB(map: Map<String, Any>) {
    val name: String by map
    val age: Int by map
}
fun main() {
    val map = mapOf("name" to "shany",
            "age" to 18)
    val b = ClassB(map)
    print(b.name)//shayn
    print(b.age)//18
}
```

* veroable

可以拦截赋值操作

```kotlin
class ClassB() {
   var name:String by Delegates.vetoable("ThisIsInitialValue"){
       property, oldValue, newValue ->
       return@vetoable false //返回true允许更改值，false不允许更改
   }
}
```



# 中缀函数

需要满足三个条件：

1. 成员函数或拓展函数
2. 只有一个参数
3. infix声明

```kotlin
infix fun String.div(string: String):String{
    return this.replace(string,"")
}

使用：
val s = "bababbaab" div "a"
```

# inline内联函数

`inline`修饰的函数在被调用时将字节码动态插入到被到调用的地方。

`inline`修饰的函数的`lambda参数`如果运行在该函数内部的*`子函数/其他环境`*，则不允许这个lambda函数**非局部返回**（因为没有办法从该 `子函数/其他环境` 中直接退出lambda所在的外层函数），对于这种lambda函数需要添加**`crossinline`**修饰。

```kotlin
//**非局部返回**指从lambda2中执行return语句，推出的是整个func()
inline fun func1(crossinline lambda1:()->Unit,  lambda2:()->Unit){
    val f = Runnable {
        lambda1()//不可以调用非局部返回，所以用crossinline修饰
    }
    
    lambda2()//可以调用非局部返回
}

```

Kotlin也存在Java泛型所具有的**类型擦除**问题，为了优化该问题，inline函数可以结合`reified`实现**实体化类型参数**

```kotlin
inline fun <reified T> isInstanceOf(value: Any) = value is T //在这里仍然可以知道T是什么类型的，所以可以执行value is T 
print(isInstanceOf<String>(""))//true
```

**原理:**内联函数会直接被插入到被调用的地方，而`reified`修饰的类型参数会保证将用户调用时写的类型`String`同时也写入到被调用的地方，如此便没有发生类型擦除。



# 参考资料

[Star-Projections and How They Work](https://typealias.com/guides/star-projections-and-how-they-work/)

[Kotlin的独门秘籍Reified实化类型参数(下篇)](https://blog.csdn.net/u013064109/article/details/83507076)


