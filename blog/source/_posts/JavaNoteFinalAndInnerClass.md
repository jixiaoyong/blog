---
title: Java笔记之匿名内部类和final
tags: java
date: 2019-12-20 22:02:16

---

# 为什么`匿名内部类`使用`局部引用`要用`final`？

先说结论：

由于JAVA匿名内部类的实现并不是真正的闭包，而是在生成内部类的时候**将局部变量的引用拷贝了一份到内部类中**。如果不将这个外部类设置为`final`的话，外部类或者内部类修改这个局部变量后，另外一处使用的仍然是修改前的值，这样就会产生问题，而如果将其修改为`final`则保证了局部变量与内部类使用的值是一致的。

# JDK1.8 后局部变量不要求用`final`了？

不对，仍然是要求`final`的。

只不过编译器判断该局部变量不会再被修改时（*effectively final* 事实上的`final`），可以省略。

> Any local variable, formal parameter, or exception parameter used but not declared in an inner class **must either be declared final or be effectively final** (§4.12.4), or a compile-time error occurs where the use is attempted.
>
> Any local variable used but not declared in an inner class must be definitely assigned (§16 (Definite Assignment)) before the body of the inner class, or a compile-time error occurs.
>
> https://docs.oracle.com/javase/specs/jls/se13/html/jls-8.html#jls-8.1.3

# 为什么外部类的全局变量不需要`final`？

因为全局变量是通过传入内部类的**外部类引用`this$0`**来引用的(而非直接复制全局变量的值)，这样内部类和外部类持有的是同一个全部变量，自然不会存在两处更新不同步的问题。

# 源码解析

```java
public class Test {
    
    public static void main(String[] args) {
        OutClass outClass = new OutClass();
        outClass.outMethod();
    }
}

// 外部类
class OutClass {

    //全局变量
    public AnObj anObj = new AnObj(5);

    public void outMethod() {
        //局部变量
        final AnObj anObj1 = new AnObj(6);

        //匿名内部类
        Inner o = new Inner() {
            public void doSth() {
                //内部类访问外部类的全部变量和局部变量
                int value = anObj1.getI() + anObj.getI();
                System.out.println("inner:" + value);
            }
        };

        o.doSth();
    }
}

//下面这两个类无需关注
interface Inner {
    public void doSth();
}

class AnObj {

    private int i;

    public AnObj(int i) {
        this.i = i;
    }

    public int getI() {
        return i;
    }
}
```

编译后如下：

```java
//OutClass.class
class OutClass {
    public AnObj anObj = new AnObj(5);

    OutClass() {
    }

    public void outMethod() {
        //可以看到，在编译之后，将外部类的引用和局部变量作为内部类的参数传入到内部类中
        new 1(this, new AnObj(6)).doSth();
    }
}
```

再看看内部类`OutClass$1`

```java
//OutClass$1.class
class OutClass$1 implements Inner {
    final /* synthetic */ OutClass this$0;//这个是外部类的引用
    final /* synthetic */ AnObj val$anObj1;//这个引用指向局部变量引用指向的内存空间

    OutClass$1(OutClass this$0, AnObj anObj) {
        this.this$0 = this$0;
        this.val$anObj1 = anObj;
    }

    public void doSth() {
        //在这里可以看出，内部类通过局部变量的备份引用访问的this.val$anObj1.getI()，
        //所以如果内部类所处的方法，修改了这个局部变量（假设将这个引用指向了另外一个AnObj对象），
        //但这里的this.val$anObj1指向的仍然是旧的AnObj对象，从而出现了对“同一个局部变量”在内部类和内部类外部分别有不同值的问题，所以需要final来限制对局部变量的更改
        System.out.println("inner:" + (this.val$anObj1.getI() + 
                                       this.this$0.anObj.getI()));
        //但是再看外部类的全局变量，在内部类中仍然是通过外部类的引用this.this$0来引用anObj的，所以无论何时，内部类中的外部类全局变量都是最新的，所做的更改也会实时更新到外部类中，所以不需要final
    }
}
```

# 参考资料

https://www.runoob.com/w3cnote/inner-lambda-final.html

[Why are only final variables accessible in anonymous class?--stackoverflow](https://stackoverflow.com/questions/4732544/why-are-only-final-variables-accessible-in-anonymous-class/4732617#4732617)

[java为什么匿名内部类的参数引用时final？ - 胖君的回答 - 知乎](https://www.zhihu.com/question/21395848/answer/110829597)