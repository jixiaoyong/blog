---
title: java创建线程安全的单例Singleton
date: 2018-02-14 18:34:16
tags: java
---

# 简介

在编码中常常会用到单例，来确保类有一个唯一的对象，一般情况下将构造方法私有化既可以实现，但当考虑到多线程时事情会变得有些复杂，本文讨论的正是几种多线程的情况下实现单例的方法。

# 1.普通单例

私有化构造方法，对外提供一个公有、静态的方法，在其内部判断类对象是否已经存在，否的话生成类对象再返回。

```java
class ASingleton{
	
	private static ASingleton as;
	
	private void ASinleton() {
		System.out.print("ASingleton init!\n");
	}
	
	public static ASingleton getInstance() {
		if(as == null) {              //tag1
			as = new ASingleton();    //tag2
		}
		return as;
	}
}
```

但是，在考虑多线程时，由于java代码是一行行进行的，假设有两个线程t1、t2，当as为null的时候t1执行到tag1位置，判断为true，于是准备执行tag2，就在此时，cup调度t2开始执行tag1，此时t1尚未执行tag2，所以在t2中tag1判断为true,t2也开始执行tag2生成一个新对象,这样当t1再次执行tag2时就会再生成一个新对象，这样就同时存在多个类的对象。

# 2.同步锁

对上面的代码稍作优化,可以看到使用了synchronized，对判断是否需要初始化进行了同步锁，这样当线程t1访问时，语句被锁定，t2运行到这里时，只能等t1运行完这段语句并释放之后，才能继续访问，此时as已经被赋予了对象，所以不会再继续新建，这样就保证了单例。

```java
class ASingleton {

	private static ASingleton as;

	private void ASinleton() {
		System.out.print("ASingleton init!\n");
	}

	public static ASingleton getInstance() {
		synchronized (ASingleton.class) {
			if (as == null) {
				as = new ASingleton();
			}
		}
		return as;
	}
}
```

但是这样也存在一个问题，每个线程每次获取单例都要进入同步锁，这样累计下来必然影响效率。

# 3.双重检查锁定

那么在判断as为null后，对as的初始化进行同步锁呢？

```java
public static ASingleton getInstance() {

		if (as == null) {
			synchronized (ASingleton.class) {
				if (as == null) {           //tag1
					as = new ASingleton();  //tag2
				}
			}
		}

		return as;
	}
```

这样子，当判断as为null时，才会进行初始化，同时由于初始化过程加锁，所以t1和t2无法同时访问初始化语句tag2，也保证了只能创建一个单例。

看起来很完美，但是由于java语言的特性，在该段代码编译为汇编语言时，上述方法会被编译为类似下面的过程：

```c
1.判断as是否为null
2.令as = ASingleton() //注意此时只是为as分配了内存，并未执行ASingleton的构造方法
3.开始执行ASingleton构造方法，as有了初始化的值
4.返回as
```

那么，当t1执行到语句2，而t2开始执行语句1时，此时由于as已经分配了内存不为null，所以t2直接执行语句4，此时t2获取到的是一个没有执行构造方法的ASingleton对象，显然这样十分危险。在线程复杂的情况下很容易出现问题。

下面提供了两个结局思路，为简便起见，将其简单分为“饿汉模式”和“懒汉模式”（其实上述方法也可分为这两个模式，but，who cares...）。

# 4.饿汉模式实现单例

饿汉模式，即在声明的时候就将对象初始化。

这样实现单例的原理是类的静态变量全局唯一。

```java
class ASingleton {

	private static ASingleton as = new ASingleton();

	private void ASinleton() {
		System.out.print("ASingleton init!\n");
	}

	public static ASingleton getInstance() {
		return as;
	}
}
```

但是这样仍然有个问题，可能在用到ASingleton类的时候，并不需要立即获取到其单例，在这种情况下，饿汉模式仍然有浪费资源的嫌疑。

# 5.懒汉模式实现单例

懒汉模式，只有要用到该实例时，才获取该单例。

这次我们用到的时静态内部类，静态内部类与类的静态变量不同，只有明确调用静态内部类的时候才会初始化静态内部类。

```java

class ASingletonFactory{
	
	static class ASingleton {

		public static ASingleton as = new ASingleton();

		private void ASinleton() {
			System.out.print("ASingleton init!\n");
		}	
	}
	
	
	public static ASingleton getInstance() {
		return ASingleton.as;
	}	
}
```

嗯，至此已经完成了java实现单例的绝大部分方法，但其实还有一张更加简洁的方法，那就是用enmu实现。

# 6.enmu实现单例

由于枚举类型的对象是唯一的，所以是实现单例的较优选择。

```java
enum SingletonEnum{
	INSTANCE;
	
	public void dosth(){
		
	}
}
```

但是，android开发者要注意，枚举占用的内存是普通单例的两倍多，所以，并不推荐在android中使用。

关于枚举的更详细资料，参阅(深入理解Java枚举类型(enum))[http://blog.csdn.net/javazejian/article/details/71333103#t7]