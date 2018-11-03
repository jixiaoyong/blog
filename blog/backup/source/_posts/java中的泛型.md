---
title: java中的泛型
tags: java
abbrlink: b2cdb69e
date: 2018-11-03 11:01:34
---

Java中的泛型实现了**参数类型化**的概念。

主要有以下形式：

```java
class OneClazz<T>{
    T t;
    <Y> void fun(){}
}
```

本文主要记录Java泛型一些比较特殊的知识点。

# 泛型特性

泛型在Java SE5被引入，可以在类和方法中，将类型作为类型参数传入。

泛型类型参数会在实际运行时被**擦除**到他的第一个边界。如`<T>`会被擦除为`Objet`，而`<T extends ClazzA>`则会被擦除为`ClazzA`。



# 需要注意的地方

## 不能有泛型数组

这是因为Java中Object[]默认为所有数组的父类，如下代码虽然在编译期不会报错，但是在运行时会被检查出objArr指向的数组实际类型（String）和要赋予的类型（Integer）不一致而报错。

也就是说，数组只能存放**定义的实际类型**以及他们的**子类型**。

```java
Object[] objArr = new String[10];
objArr[0] = 1;
```

但是，如果支持泛型数组：由于泛型类型参数会在运行时被擦除，导致即使到了运行时也无法发现这个错误，从而会导致错误。

如下，加入支持泛型参数，则objArr1中实际保存的类型（Map<String,Integer>），在编译的时候由于objArr1和objArr2都是Object类型的数组，编译通过；在运行的时候，由于Map中的泛型参数类型已经被擦除，也无法区分objArr1和objArr2中实际指向的两个Map<K,V>数组，也是合法的，这样原本定义的是Map<String,Integer>数组，却可以保存任何类似的Map，而这本来是不允许的。

```java
Object[] objArr1 = new Map<String,Integer>[10];
Object[] objArr2 = new Map<Double,Integer>[10];
objArr1[0] = objArr2[0];
```



> Collections 类通过一种别扭的方法绕过了这个问题，在 Collections 类编译时会产生类型未检查转换的警告。
>
> `ArrayList`具体实现的构造函数如下：
>
> ```java
> class ArrayList<V>{
>     private V[] backingArray;
>     public ArrayList(){
>         backingArray = (V[])new Object()[DEFAULT_SIZE];
>     }
> }
> ```
>
> 为何这些代码在访问 `backingArray`时没有产生 `ArrayStoreException`呢？无论如何，都不能将 `Object`数组赋给 `String`数组。因为泛型是通过擦除实现的，`backingArray`的类型实际上就是 `Object[]`，因为 `Object`代替了 `V`。
>
> **这意味着：实际上这个类期望 `backingArray`是一个 `Object`数组，但是编译器要进行额外的类型检查，以确保它包含 `V`类型的对象。**
>
> 来源：https://www.ibm.com/developerworks/cn/java/j-jtp01255.html

# 泛型容器

泛型数组只能保存指定泛型类型的数据，而不能保存其子类。

```java
class Fruit{}
class Apple extends Fruit{}
//编译时报错，类型不兼容
List<Fruit> fruits = new ArrayList<Apple>();
```

可以用通配符*“在两个类型之间建立某种类型的**向上转型**关系”*。

List<? extends Fruit>可以合法的指向一个List< Apple>，这个过程会完成自动**向上转型**，成为可以持有**某个诸如Fruit或者Apple的类**的List，但是编译器不知道这个**类**具体是什么，所以拒绝向其中传递任何类型对象，即使Object也不行。

```java
List<? extends Fruit> fruits = new ArrayList<Apple>();
fruits2.add(new Apple());//编译时报错，类型转化错误
fruits2.add(new Fruit());//编译时报错
```

# 参考资料

[java为什么不支持泛型数组？ - ylxfc的回答 - 知乎](https://www.zhihu.com/question/20928981/answer/39234969)

[Oracle Java 泛型原理](https://www.oracle.com/technetwork/cn/articles/java/juneau-generics-2255374-zhs.html)

[Java 理论和实践-了解泛型-识别和避免学习使用泛型过程中的陷阱](https://www.ibm.com/developerworks/cn/java/j-jtp01255.html)

《Java编程思想 第4版》