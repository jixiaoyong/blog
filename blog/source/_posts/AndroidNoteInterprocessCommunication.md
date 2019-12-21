---
title: Android笔记之跨进程通信
tags: android
date: 2019-12-21 09:44:32
---

# 计算Java对象的大小

**注意 以下所说的计算Java对象大小，不涉及该对象所持有的对象本身的大小，只计算该Java对象本身的大小（其中引用类型对象大小只计算为4bytes），如果要遍历计算Java对象大小（包含其持有对象的大小）可以参考[这篇文章 Sizeof for Java](https://www.javaworld.com/article/2077408/sizeof-for-java.html)**

一个Java对象在内存中的大小包括以下(以64位JVM启用压缩为例，综合[这里](https://blog.csdn.net/ITer_ZC/article/details/41822719)和[这里](https://blog.csdn.net/u013380694/article/details/102739636)的信息整理)：

| 分类      | 大小（byte） | 备注                                        |
| --------- | ------------ | ------------------------------------------- |
| 对象头    | 8            | 保存对象的 class 信息、ID、在虚拟机中的状态 |
| Oop指针   | 4            |                                             |
| 数据区    |              | 对象实际包含的数据,引用类型大小为4 bytes    |
| 数组长度  | 4            | 只有数组对象才有                            |
| 8比特对齐 |              | 将对象总大小对齐到8字节所需的填充           |

> 此外，如果是（非静态）内部类的话，由于他默认持有外部类的引用，所以会比普通类的对象多4个byte。
>
> https://stackoverflow.com/a/12193259/8389461

可以参照这张图

![图片来自https://www.jianshu.com/p/9d729c9c94c4](https://upload-images.jianshu.io/upload_images/2509688-c69b5f36addd9a63.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

其中，数据区占用的大小如下：

<script async class="speakerdeck-embed" data-slide="29" data-id="6950a0f032390131a8f91e7b7986a513" data-ratio="1.77777777777778" src="https//speakerdeck.com/assets/embed.js"></script>

下图来自于[android-memories](https://speakerdeck.com/romainguy/android-memories?slide=29)

![Size of data from speakerdeck.com](https://jixiaoyong.github.io/images/20191221104050.png)

>```
>    // java.lang.Object shell size in bytes:
>    public static final int OBJECT_SHELL_SIZE   = 8;
>    public static final int OBJREF_SIZE         = 4;
>    public static final int LONG_FIELD_SIZE     = 8;
>    public static final int INT_FIELD_SIZE      = 4;
>    public static final int SHORT_FIELD_SIZE    = 2;
>    public static final int CHAR_FIELD_SIZE     = 2;
>    public static final int BYTE_FIELD_SIZE     = 1;
>    public static final int BOOLEAN_FIELD_SIZE  = 1;
>    public static final int DOUBLE_FIELD_SIZE   = 8;
>    public static final int FLOAT_FIELD_SIZE    = 4;
>```
>
>https://www.javaworld.com/article/2077408/sizeof-for-java.html

每个对象引用占用4个字节

下面是一个计算对象大小的示例：

根据[Romain Guy](https://speakerdeck.com/romainguy)在[SpeakerDeck](https://speakerdeck.com/romainguy/android-memories?slide=34)中的说法：

> 一个空的class占用了4+8=12个byte的内存，再加上8比特对齐，实际占用大小为16比特。
>
> ```java
> class Empty{
> 
> }
> ```
>
> 占用大小：
>
> | Allocation                | Size in bytes |
> | ------------------------- | ------------- |
> | dlmalloc 引用             | 4             |
> | Object overhead（对象头） | 8             |
>
> Total  = 4 + 8 =12 bytes
>
> 经过8-byte aligned后： total = 16 bytes
>
> https://speakerdeck.com/romainguy/android-memories?slide=34

此外还有包含了数据的对象计算方式如下：

<script async class="speakerdeck-embed" data-slide="42" data-id="6950a0f032390131a8f91e7b7986a513" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

对于数组的大小计算（参考[一个Java对象到底占用多大内存？](https://www.cnblogs.com/zhanjindong/p/3757767.html)和[romainguy/android-memories](https://speakerdeck.com/romainguy/android-memories?slide=54)，后者关于数组大小的计算中`width&padding = 8 `的意义存疑）:

按照开头的公式：`数组大小 = 8 对象头 + 4 Oop指针 + 4 数组大小标记length + 数组数据占用大小 + 8比特对齐 `

```java
// arr0大小 = 8 + 4 + 4 + 0 + 8比特对齐(0) = 16 bytes
int arr0 = new int[0];
// arr1大小 = 8 + 4 + 4 + 4*1 + 8比特对齐(4) = 16 + 4 = 20 + 8比特对齐(4) = 24 bytes
int arr1 = new int[1];
// arr1大小 = 8 + 4 + 4 + 4*10 + 8比特对齐(0) = 16 + 40 = 56 + 8比特对齐(0) = 56 bytes
int arra10 = new int[10];
```

# 计算对象大小

具体的如何计算Java中Object大小，可以参考[stackoverflow的这个回答](https://stackoverflow.com/a/52682/8389461)（[这里](https://github.com/cirosantilli/java-cheat/tree/a907ad2243dce2109c54d27323f9387065b5ca5c/instrument)有一份Github上面的实现源码）

可以参考文章：

[聊聊JVM（三）两种计算Java对象大小的方法](https://blog.csdn.net/ITer_ZC/article/details/41822719)

[准确计算Java中对象的大小](https://www.iteye.com/blog/brandnewuser-2113828)

[一个Java对象到底占用多大内存？](https://www.cnblogs.com/zhanjindong/p/3757767.html)

# 实际应用

##string最长为65534

String s = “”;中，在编译期最多可以有65534个字符

> 原因是，Java中的UTF-8编码的Unicode字符串在常量池中以CONSTANT_Utf8类型表示，常量池中的所有字面量几乎都是通过CONSTANT_Utf8_info描述的。
>
> 这里面的`u2 length`表明了该类型存储数据的长度，而`u2`是无符号的16位整数，因此理论上允许的的最大长度是2^16=65536。而 java class 文件是使用一种变体UTF-8格式来存放字符的，null 值使用两个字节来表示，因此只剩下 65536－ 2 ＝ 65534个字节。
>
> ```c
> CONSTANT_Utf8_info {
>     u1 tag;
>     u2 length;
>     u1 bytes[length];
> }
> ```
>
> **在Java中，所有需要保存在常量池中的数据，长度最大不能超过65535，这当然也包括字符串的定义**
>
> 上面提到的这种String长度的限制是编译期的限制，也就是使用String s= “”;这种字面值方式定义的时候才会有的限制。
>
> String在运行期有没有限制呢，答案是有的，就是我们前文提到的那个Integer.MAX_VALUE ，这个值约等于4G，在运行期，如果String的长度超过这个范围，就可能会抛出异常。(在jdk 1.9之前）
>
> https://blog.csdn.net/u013380694/article/details/102739636

一个String对象,占用大小（JDK1.8）：

```java
/** The value is used for character storage. */
    private final char value[];  //一个数组对象的引用，占用4 bytes
/** Cache the hash code for the string */
    private int hash; // Default to 0   //一个int类型，占用4 bytes
```

再加上在64位JVM中，一个对象具有12bytes的对象头+引用，要求对齐到8的倍数(来源[2.1. Objects, References and Wrapper Classes](https://www.baeldung.com/java-size-of-object#1-objects-references-and-wrapper-classes))，所以一个String对象的大小是：

```
size = ( 12 对象头 + 4 value + 4 hash ) + 4 8byte对齐 = 24 bytes
```

## 枚举类enum

### 枚举类大小的计算

枚举类中的每个枚举都是该枚举类的一个对象。

```java
enum EnumClazz{
    Day,Hour,Minute,Second
}
```

当我们用`javap`查看其编译后的字节码可以看到：

```java
//javac EnumClazz.java
//javap EnumClazz.class
final class EnumClazz extends java.lang.Enum<EnumClazz> {
  public static final EnumClazz Day;
  public static final EnumClazz Hour;
  public static final EnumClazz Minute;
  public static final EnumClazz Second;
  public static EnumClazz[] values();
  public static EnumClazz valueOf(java.lang.String);
  static {};
}
```

简单计算一下这个EnumClazz的大小（不含引用对象的大小）：

```java
enumClassSize = 8 + 4 + 4*4 + 4 = 32 bytes
       对象头 +  引用 + 枚举类值的引用类型 * 4个 + 4 数组引用类型
```

然后，我们再看一下每个枚举类的值（以EnumClazz.Day为例）的大小：

enum类的每个值实际上都继承自`java.lang.Enum`类：

```java
public abstract class Enum<E extends Enum<E>>
        implements Comparable<E>, Serializable {
        //枚举值名称
        private final String name;
        //枚举值次序，从0开始
        private final int ordinal;
    }
```

由此，我们可以计算`EnumClazz.Day`的大小：

```java
daySize = 8 + 4 + 4 + 4 + 8比特对齐(4) = 20 + 4 = 24 bytes
     对象头 + Oop引用 + name + ordinal + 8比特对齐
```

也就是说，本例中每一个枚举类值占用24 bytes，由此可以计算出EnumClazz实际占用的大小应该是：

```java
realSize = enumClassSize + daySize * 4  = 128 bytes
```



### Android中是否应该使用枚举

关于是否应该在Android中使用枚举类，可以参考下文：

https://www.liaohuqiu.net/cn/posts/android-enum-memory-usage/

https://stackoverflow.com/a/29972028/8389461