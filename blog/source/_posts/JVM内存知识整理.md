---
title: JVM内存分配
tags: jvm
abbrlink: f31c11c5
date: 2018-02-24 23:39:19
---



> 本笔记基于《深入理解Java虚拟机：JVM高级特性与最佳实践》及部分在线博客整理。

JVM：java virtual machine，一个java程序（进程）拥有一个jvm实例

# 内存

JVM区域总体分两类，heap区和非heap区:

**heap区**：Eden Space（伊甸园）、Survivor Space(幸存者区)、Tenured Gen（老年代-养老区）。
**非heap区**：Code Cache(代码缓存区)、Perm Gen（永久代）、Jvm Stack(Java虚拟机栈)、Local Method Statck(本地方 法栈)。

# 内存划分

## 1.head

堆，所有线程共享，存放所有对象实例、数组，GC主要场所，会OOM

分类

**1.新生代**

- eden  刚刚创建的对象优先


- s1  经历几次GC


- s2  经历几次GC

**2.老年代**

- 存活时间长的老年对象


- 大对象，如数组，大String...

## 2.stack

栈，线程私有，存放基本数据和对象的引用，LIFO，会OOM，StackOverflow

### java virtual machine stack

线程请求的栈深度大于JVM允许的深度会导致Stack Overflow
在编译期完成内存分配，如果虚拟机栈可以动态扩展，但是当拓展时无法申请到足够内存时会导致OutOfMemory

**stack  frame**

stack frame：栈帧，每执行一个方法就会产生一个栈帧并压入栈中

*  局部变量表
   * 基本数据类型
   * 对象引用
   * returnAddress 类型,指向了一条字节码指令的位置

* 操作数栈
* 动态链接
* 方法出口等

### native method stack

与java虚拟机栈作用类似，不过native method stack是为native方法服务。
jvm可以自由实现它，甚至在sun HotSpot VM中将他与虚拟机栈合并
会OOM，stackOverflow

## 3.method area

方法区，线程共享，存放类信息，常量，静态变量，即时编译器编译后的代码，会OOM

**运行时常量池**

类加载后，编译器生成的各种字面量和符号引用会放到方法区的运行时常量池中，会OOM
`String.intern()`，有该string对象则返回，无则创建并返回

> `String.intern()`方法的注意事项：
>
> JDK1.6及以下：将首次出现的对象实例**复制**到永久代，返回其引用
>
> JDK1.7及以上：只会**记录**下首次出现的实例的引用，返回其引用
>
> 所以：
>
> ```java
> String s2 = "java";
> System.out.println(s2.intern() == s2);
> ```
>
> 在JDK1.6及以下输出`false`，在JDK1.7及以上输出`true`
>
> 此外，由于`String`类是`final`的，每次`new String("str")`会产生两个对象：一个是字符串`str`本身，一个是值为`str`的字符串。



以`String s = "Hello";`为例，解释几个概念：

**字面量** 源码中表示具体的值，如`Hello`

**符号引用** 用来指代某种值得符号，如`s`

**直接引用** 可以定位到内存中的（类、对象、方法、变量）等的具体地址

## 4.program count

**程序计数器**，线程私有，占用内存小，当做当前线程执行字节码的行号指示器。

若执行java方法，计数器记录的是正在执行的虚拟机字节码指令的位置

若执行的是native方法，则计数器为空undefined。

此内存区域是唯一一个在java虚拟机规范字没有规定任何OOMError的区域

# 内存溢出

> 以Sun HotSpot VM 为例

## 1.java堆溢出

对象过多导致head内存溢出

1. 是内存泄漏memory leak，定位泄露对象
2. 是内存溢出memory overflow，检查虚拟机堆参数是否可以调大；去除非必须的生命周期长的对象

## 2.虚拟机栈和本地方法栈溢出

1. 单线程，Stack Overflow

   单线程下，栈帧过大或者虚拟机栈容量太小，当内存无法分配时都会导致Stack Overflow异常

2. 多线程，

   多线程时，每个线程栈分配的内存越大，越容易尝试内存溢出OOM

   原因：虚拟机最大内存一定的情况下，去掉共享的Head和MethodArea占的内存，剩下的内存/单个线程最大栈内存=最大线程数量，当单个线程最大栈内存增加时，可以产生的线程数就会越少

## 3.运行时常量池溢出

运行时常量池属于方法区，当常量过多时会导致OOM，可以用String.intern()方法尝试

## 4.方法区溢出

经常动态生成大量Class的应用，如Spring等框架，需要注意OOM

## 5.本地直接内存溢出

原生方法直接操作物理内存时导致物理内存不够，产生OOM

# GC垃圾回收

JVM中GC会根据不同情况采取以下一系列算法组合进行内存回收

## 回收算法

### 1.复制算法

**原理**：内存一分为二，只使用一半；GC时将存活对象复制到另一半内存，剩下的则清空

**优缺点**：1.无STW，但不适合对象过多的情况；2.内存利用效率低

### 2.标记清除法

**原理**：从GC Roots开始遍历，可达标记存活，不可达则未标记

java中，GC Roots可以是以下几种：

- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 方法区中的类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI引用的对象

**优缺点**：1.要StopTheWorld防止标记的时候新new的对象未被标记而出错；

2.清除对象后内存不连续，会有一定的浪费

### 3.标记压缩法

**原理**：类似【标记清除法】，但会对标记进行压缩，如a->b->c，会被压缩为a->c，具体试讲所有存活的对象都向一端移动，直接清理掉端边界外的内存

**优缺点**：1.也要StopTheWorld

### 4.引用计数算法

**原理**：引用+1，不引用-1，为0则删除，但是会有相互循环引用的问题，java未使用

**优缺点**：相互循环使用：
a = b
b = a
除此之外再没有用到a，b的地方，但是由于a，b的引用不为0所以无法被回收，导致内存浪费

## 回收过程

一个不可达对象在“死缓”到“执行死刑”前至少经历两个标记过程

1. 第一次标记，筛选：是否有必要执行finalize()方法，若是则放到F-Queue队列中【触发】该方法，但不保证执行完该方法。

   可以在finlize()方法中**自救一次**：在该方法中将*自身this*赋值给其他变量，这样在第二次标记时会被移出*即将回收*集合；但是由于finlize()方法只会被调用一次，所以只能自救一次。并*不推荐该方法*，该方法所有可以做的工作，可以用**try...finally**或者其他方法更好的实现

2. 第二次标记，若finalize()方法以及调用过，或者为重写该方法，则“没必要执行”，可以回收

# 对象

## 对象引用

### 强引用StrongReference

People p = new People();哪怕抛出OOM也不会被GC回收的对象

### 软引用SoftReference

SoftReference sf = new SoftReference(p);只要有足够内存就不会被GC回收，若内存不够则会被GC回收，常用作服务器缓存

### 弱引用WeakReference

在下次GC回收之前都存在，用作android等内存紧张的设备中的缓存

### 虚引用PhantomReference

无法影响其生存时间，也无法通过虚引用获取其实例，设置虚引用只是为了在对象被GC回收时获取系统通知

# 脑图

![](https://raw.githubusercontent.com/jixiaoyong/jixiaoyong.github.io/master/images/blog/2018-02/JVMMemory.png)