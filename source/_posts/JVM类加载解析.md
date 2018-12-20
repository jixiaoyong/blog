---
title: JVM类加载机制解析
tags: jvm
abbrlink: cf83ef31
date: 2018-02-26 19:28:56
---

# 简述

本文介绍了java虚拟机类加载机制。

# 类加载机制

JVM类加载一共7步，前五步是类加载机制，各个步骤按照顺序进行，但是并非固定的1,2,3步，在实际中有可能从其中间某一步开始。

类加载机制一般分为三部分：**加载Loading -> 连接Linking -> 初始化Initializing**

![JVM Class Loader](https://github.com/jixiaoyong/Notes-Files/blob/master/draw-io/png/JVMClassLoader.png?raw=true)



其中**加载、验证、准备和初始化**发生的顺序是确定的，但**解析**可以在初始化之后开始（java动态绑定）



> java绑定分为静态绑定和动态绑定：
>
> - **静态绑定**：即前期绑定。在程序执行前方法已经被绑定，此时由编译器或其它连接程序实现。针对java，简单的可以理解为程序编译期的绑定。java当中的方法只有final，static，private和构造方法是前期绑定的。
> - **动态绑定**：即晚期绑定，也叫运行时绑定。在运行时根据具体对象的类型进行绑定。在java中，几乎所有的方法都是后期绑定的。
>
> http://blog.csdn.net/ns_code/article/details/17881581

# 类加载机制具体过程

## **I.Loading **

**加载**，JVM将文件（class，jar，zip，网络等）中的二进制字节流保存到虚拟机方法区和堆中,并用该二进制表示形式创建类或者接口的过程。

> Loading is the process of finding the binary representation of a class or interface type with a particular name and *creating* a class or interface from that binary representation
>
> https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-5.html (*英文文档若无特殊说明都是引用官方文档，下同*)

1. 用**类全限定名**获取类的二进制字节流
2. 将字节流中**静态存储结构**转化为**方法区**的运行时数据结构
3. <u>*在**堆**中生成一个代表该类的java.lang.Class对象，作为方法区数据的访问入口。*</u>（**这句话存疑** ，有人说[在堆中](http://blog.csdn.net/ns_code/article/details/17881581) ,也有人说在[方法区](http://gityuan.com/2015/10/25/jvm-class-loading/) ,官方文档未相关描述）

类加载的地方是开发人员可控性最强的地方。除了可以使用系统的ClassLoader外还可以自定义ClassLoader（后文详述）。



-------------------------

## **II.Linking **

**连接**，是将类或者接口组合到java虚拟机运行状态的过程，这样他就可以被运行。

> Linking is the process of taking a class or interface and combining(组合) it into the run-time state of the Java Virtual Machine so that it can be executed(运行)

连接一般分为3部分：验证Verification 、准备Preparation 、解析Resolution 。

> Linking a class or interface involves(包括) verifying and preparing(验证和准备) that class or interface, its direct(直接) superclass, its direct superinterfaces, and its element type (if it is an array type), if necessary. Resolution of symbolic references in the class or interface is an optional part of linking.



### **Verification** 

**验证**，保证class文件中的字节流信息符合虚拟机的要求。

> Verification([§4.10](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.10)) ensures that the binary representation(二进制格式) of a class or interface is structurally correct(结构正确) ([§4.9](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.9)). Verification may cause additional classes and interfaces to be loaded ([§5.3](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-5.html#jvms-5.3)) but need not cause them to be verified or prepared.

验证内容包括：

1. 文件格式验证，验证字节流符合class文件格式规范；
2. 元数据验证
3. 字节码验证
4. 符号引用验证

### **Preparation **

**准备**，在方法区对类变量分配内存，**初始化为默认值**

比如：`static int i = 5；`在这一步只会进行到`i = 0` ，而`i = 5`要在初始化那一步才进行。

> *Preparation* involves creating the static fields for a class or interface and initializing such fields to their default values ([§2.3](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-2.html#jvms-2.3), [§2.4](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-2.html#jvms-2.4)). This does not require the execution of any Java Virtual Machine code; explicit initializers for static fields are executed as part of initialization ([§5.5](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-5.html#jvms-5.5)), not preparation.

准备工作可能在创建之后的任何时候发生，但是必须在初始化之前完成。

### **Resolution**

**解析**，是在运行时常量池中动态确定符号引用的具体值的过程。

每个栈帧frame都有一个`当前方法`到`运行时常量池` 的引用，用来支持方法代码(method code)的**动态链接（dynamic linking）**。

method code：要被执行的方法以及通过符号引用的变量。

> Each frame ([§2.6](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-2.html#jvms-2.6)) contains a reference to the run-time constant pool ([§2.5.5](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-2.html#jvms-2.5.5)) for the type of the current method to support *dynamic linking* of the method code.

动态链接将符号引用（symbolic references）转化为具体方法的调用（concrete method references），根据需要加载类来解析未定义的符号，将变量访问转化为运行时内存（runtime location）。

> This late binding of the methods and variables makes changes in other classes that a method uses less likely to break this code.
>
> *方法和变量的这种后期绑定,使得方法使用的其他类的更改不太可能破坏这个代码。*

解析分为：

1. 类，接口解析
2. 字段解析
3. 类方法解析
4. 接口方法解析



------------------

## **III.Initialization**

**初始化** ，*Initialization* of a class or interface consists of executing its class or interface initialization method（执行类，接口的构造方法`clinit()`）

类或接口在被初始化之前，必须先被连接linked（ verified, prepared, and optionally（可选） resolved.）。



`clinit()` ,有**类变量赋值，静态语句块**会由编译器合并为`clinit()`方法,分为两种：

1. 类 父类的`clinit()`方法会先于子类执行
2. 接口  接口`clinit()`方法无需调用父类接口的`clinit()`方法；接口的实现类也无需执行接口的`clinit()`方法



`clinit()`和`init()`不同如下：

> **init**是对象构造器方法，也就是说在程序执行 new 一个对象调用该对象类的 constructor 方法时才会执行init方法<u>（是在**new对象**的时候**初始化非静态变量**）</u>；
>
> 而clinit是类构造器方法，也就是在jvm进行类**加载—–验证—-解析—–初始化**，中的初始化阶段jvm会调用clinit方法<u>（是在**JVM初始化类**的时候**初始化静态变量**）</u>。
>
> http://blog.csdn.net/u013309870/article/details/72975536



`clinit()`先于`init()`执行。



# 参考文献

[Java Virtual Machine Specification Chapter 5. Loading, Linking, and Initializing](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-5.html)

[Chapter 2. The Structure of the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-2.html#jvms-2.5.2)

[【深入Java虚拟机】之四：类加载机制](http://blog.csdn.net/ns_code/article/details/17881581)

[JVM类加载机制详解（一）JVM类加载过程](http://blog.csdn.net/zhangliangzi/article/details/51319033)

[Jvm系列3—类的加载 - Gityuan博客 | 袁辉辉博客  ](http://gityuan.com/2015/10/25/jvm-class-loading/)

[类加载机制 - 深入理解 Java 虚拟机 - 极客学院Wiki ]( http://wiki.jikexueyuan.com/project/java-vm/class-loading-mechanism.html)

[深入理解Java类型信息(Class对象)与反射机制 - CSDN博客](  http://blog.csdn.net/javazejian/article/details/70768369)

[深入理解jvm--Java中init和clinit区别完全解析 - CSDN博客](http://blog.csdn.net/u013309870/article/details/72975536)