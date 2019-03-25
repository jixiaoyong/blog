---
title: Java反射简单应用
tags: java
abbrlink: 3e9e8cb1
date: 2018-01-16 21:28:57
---

# 简介

反射，用来在运行时获取给定类的构造函数，变量，方法，并对其作以修改，而不必在编译时获取该类。

> Reflection allows programmatic access to information about the fields, methods and constructors of loaded classes, and the use of reflected fields, methods, and constructors to operate on their underlying counterparts, within security restrictions.
>
> --https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/package-summary.html

# 简单使用

定义一个待反射的类ATestClass.java

```java
package cf.android666.reflect;

public class ATestClass {

	public String name;
	private int age;
	
	public ATestClass() {
		// TODO Auto-generated constructor stub
	}

	private void init(String name, int age) {
		this.name = name;
		this.age = age;
	}

	public String getAge() {
		// TODO Auto-generated method stub
		return " age: " + age;
	}
	
}
```

在TestReflect.java中反射

```java
//核心代码
public static void main(String[] args){
	//注意这里需要是完整的类名，包括包名
	Class<?> clazz = Class.forName("cf.android666.reflect.ATestClass");
	ATestClass aTestClsObj=(ATestClass) clazz.newInstance();
  
	//反射获取变量
	Field mName = clazz.getDeclaredField("name");
	mName.setAccessible(true);
			
	mName.set(aTestClsObj, "aReflect");
	System.out.println(aTestClsObj.name);
			
	//反射获取方法
	Method mInit= clazz.getDeclaredMethod("init", String.class,int.class);
	mInit.setAccessible(true);//解除私有限定，让我们在用反射时访问私有变量
	mInit.invoke(aTestClsObj, "aInitName",66);
	System.out.println(aTestClsObj.name + aTestClsObj.getAge());

}
```

# 小结

反射的用法较为简单

* 通过`Class.froName()` 获取Class对象`clazz` ，获取要反射的Class对象`aTestClsObj` 
* 通过`clazz` 获取要反射Class的变量、方法
* 通过`aTestClsObj` 操作这些变量，方法

其中需要注意的有

* `f.setAccessible(true);` 方法可以解除`private` 限制，进而可以操作类的私有变量，方法
* `clazz.getXXX()` 方法获取**全部公有**变量、方法 ，**包括父类或接口**的xx，`clazz.getDeclaredXXX()` 方法获取**全部** 变量、方法，包括私有的，实现接口的方法，**但是不包括父类的**

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Java反射简单应用.md</div>"></iframe>