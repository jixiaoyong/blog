---
title: java注解
date: 2018-01-21 20:53:07
tags: 注解
---

注解，*是描述Java代码的代码，它能够被编译器解析，向编译器、虚拟机说明一些事情，就像java中给程序员看的注释一样*。

Android应用开发这方面比较火的是[Butter Knife](http://jakewharton.github.io/butterknife/) ,本文讲述如何自定义注解替换findViewById()。

实现**注解（annotation）**的思路：通过**反射**获取到类中使用注解的变量，方法，再调用不同的方法对这些变量，方法进行处理以达到目的。

主要涉及三方面：

* 定义一个注解类
* 定义一个注解帮助类
* 使用注解

# java元注解

java语言有四个预留的注解，用来生成其他自定义的注解：

* @Target

说明注解所能修饰的范围。其值一般为ElementType.xxx，主要有：

1. CONSTRUCTOR 描述构造器
2. FIELD 描述域
3. LOCAL_VARIABLE 描述局部变量
4. METHOD 描述方法
5. PACKAGE 描述包
6. PARAMETER 描述参数
7. TYPE 描述类，接口，enum声明

* @Retention

说明注解存活的生命周期,其值一般为RetentionPolicy.xxx，主要有

1. SOURCE 仅源文件有效，被编译器丢弃
2. CLASS 在class文件中有效，可能被虚拟机忽略
3. RUNTIME 在运行时有效，在class被装载时被获取

* @Documented

> 用于描述其它类型的annotation应该被作为被标注的程序成员的公共API

表示是否将注解信息添加在java文档中。有该注解则会被Javadoc工具文档化

是一个标记注解，没有值

* @Inherited

表示该标记会被标记的class的子类继承，在查找该注解时，如果当前类没有，会自动向上到其父类中查找，直到*该注解类型被找到或是查找完了Object类还未找到*

是一个标记注解，没有值

**注解不能继承其他注解或接口**

# 内建注解

java中常见的内建注解：

* `@Override` 重写父类方法
* `@Deprecated` 不赞成使用的api
* `@SuppressWarnings() ` 忽略指定警告

参数如下：

| 参数          | 含义                                |
| ----------- | --------------------------------- |
| deprecation | 使用了过时的类或方法时的警告                    |
| unchecked   | 执行了未检查的转换时的警告                     |
| fallthrough | 当Switch程序块进入进入下一个case而没有Break时的警告 |
| path        | 在类路径、源文件路径等有不存在路径时的警告             |
| serial      | 当可序列化的类缺少serialVersionUID定义时的警告   |
| finally     | 任意finally子句不能正常完成时的警告             |
| all         | 以上所有情况的警告                         |

# 自定义注解

## 注解类

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface BindView {
    //注解参数只可以为public或者默认
    //如果注解中的值不是value，那么在进行注解的时候，需要给出对应的值的名字
    //如@ViewInject(id = R.id.buy)
    int value();
    //注解元素必须有明确的值，要不在定义注解时指定默认值，要不在使用注解时指定
    public int age() default 18;//指定默认值
}
```

> 注解参数支持数据类型如下：
>
> 1.所有基本数据类型（int,float,boolean,byte,double,char,long,short)
> 2.String类型
> 3.Class类型
> 4.enum类型
> 5.Annotation类型
> 6.以上所有类型的数组

## 注解帮助类

主要提供使用注解的方法，代码中的注解替换为真正要实现的逻辑，为注解和使用注解的类搭建一个桥梁。

```java
//核心方法如下
public static void bindViews(Activity activity) {
  		//获取到使用注解的类
        Class<? extends Activity> clazz = activity.getClass();
  		//获取该类中的所有域变量
        Field[] fields = clazz.getDeclaredFields();
  		//通过遍历，将使用到注解的变量初始化
        for (Field field : fields) {
          	//获取注解对象
            BindView bindView = field.getAnnotation(BindView.class);
            if (bindView != null) {
              	//获取注解的值
                int viewId = bindView.value();
                if (viewId != -1) {
                    try {
                      	//注解要实现的逻辑，此处为替代clazz中的findViewById()方法，注意getMethod()是获取该类及其实现的接口中所有的public方法
                        Method findViewById = clazz.getMethod("findViewById", int.class);
                        findViewById.setAccessible(true);
                        Object o = findViewById.invoke(activity, viewId);
                      	//修改要注解的类，到此注解目的达到
                        field.setAccessible(true);
                        field.set(activity,o);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
```

## 使用注解

在类中通过`@xxx()` 使用注解，并通过帮助类真正实现注解逻辑

```java
//标记注解
@BindView(R.id.text)
private TextView textView;

//调用帮助类方法
AnnotationUtils.bindViews(ASampleActivity.this);

//使用初始化之后的变量
textView.setText("hello annotation");

```



# 参考文章

* [Java核心技术点之注解 - ImportNew](http://www.importnew.com/23816.html)
* [java注解--gityuan](http://gityuan.com/2016/01/23/java-annotation/)