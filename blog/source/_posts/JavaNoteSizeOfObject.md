---
title: Java笔记之计算Java对象的大小及其应用
tags: java
date: 2019-12-21 09:44:32

---

# 原理

**注意 除非特殊说明，以下所说的计算Java对象大小，不涉及该对象所持有的对象本身的大小，只计算该Java对象本身的大小（其中引用类型对象大小只计算为4 bytes），如果要遍历计算Java对象大小（包含其持有对象的大小）可以参考[这篇文章 Sizeof for Java](https://www.javaworld.com/article/2077408/sizeof-for-java.html)**

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

可以参照[这张图](https://www.jianshu.com/p/9d729c9c94c4)

![图片来自https://www.jianshu.com/p/9d729c9c94c4](https://jixiaoyong.github.io/images/20191221191518.webp)

其中，数据区占用的大小如下：

（图片来自于[android-memories](https://speakerdeck.com/romainguy/android-memories?slide=29)）

![Size of data from speakerdeck.com](https://jixiaoyong.github.io/images/20191221104050.png)



#示例

根据[Romain Guy](https://speakerdeck.com/romainguy)在[SpeakerDeck](https://speakerdeck.com/romainguy/android-memories?slide=34)中的说法：

> 一个空的class占用了4+8=12个byte的内存，再加上8比特对齐，实际占用大小为16比特。

```java
class Empty{

}
```

> 占用大小：
>
> | Allocation                | Size in bytes |
> | ------------------------- | ------------- |
> | dlmalloc 引用             | 4             |
> | Object overhead（对象头） | 8             |
>
> Total  = 4 + 8 =12 bytes
>
> 经过*8-byte aligned*后： total = 16 bytes
>
> https://speakerdeck.com/romainguy/android-memories?slide=34

此外还有**包含了数据的对象**大小计算方式如下：

![图片来自https://speakerdeck.com/romainguy/android-memories?slide=42](https://jixiaoyong.github.io/images/20191221171816.png)

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

# 计算对象大小的工具

具体的如何计算Java中Object大小，可以参考[stackoverflow的这个回答](https://stackoverflow.com/a/52682/8389461)（[这里](https://github.com/cirosantilli/java-cheat/tree/a907ad2243dce2109c54d27323f9387065b5ca5c/instrument)有一份Github上面的实现源码）

可以参考文章：

[聊聊JVM（三）两种计算Java对象大小的方法](https://blog.csdn.net/ITer_ZC/article/details/41822719)

[准确计算Java中对象的大小](https://www.iteye.com/blog/brandnewuser-2113828)

[一个Java对象到底占用多大内存？](https://www.cnblogs.com/zhanjindong/p/3757767.html)

这里提供一个实例（[参考自这里](https://github.com/cirosantilli/java-cheat/tree/a907ad2243dce2109c54d27323f9387065b5ca5c/instrument)）：

`Sizeof.java`

```java
import java.lang.instrument.Instrumentation;

final public class Sizeof {
    private static Instrumentation instrumentation;

    public static void premain(String args, Instrumentation inst) {
        instrumentation = inst;
    }

    public static long sizeof(Object o) {
        return instrumentation.getObjectSize(o);
    }
}
```

`Makefile`

```makefile
//Makefile文件
.POSIX:
.PHONY: all clean

all:
	javac *.java
	jar -cfm Sizeof.jar META-INF/MANIFEST.MF Sizeof.class
	java -ea -javaagent:Sizeof.jar Main

clean:
	rm -f *.class *.jar
```

在使用时先新建一个Java类，在其中调用`sizeof()`方法：

```java
public class Main {
  public static void main(String[] args) {
    System.out.println(Sizeof.sizeof(new int[0]));
  }
}
```

可以用如下命令：

```shell
	javac *.java //编译当前目录下的java文件
	jar -cfm Sizeof.jar META-INF/MANIFEST.MF Sizeof.class //将Sizeof.class打包为Sizeof.jar
	java -ea -javaagent:Sizeof.jar Main //输出sizeOf计算结果
```

# 实际应用

## `String`最长为65534

`String s = “”;`中，在编译期最多可以有65534个字符

> ~~原因是，Java中的UTF-8编码的Unicode字符串在常量池中以`CONSTANT_Utf8`类型表示，常量池中的所有字面量几乎都是通过`CONSTANT_Utf8_info`描述的。~~
>
>~~这里面的`u2 length`表明了该类型存储数据的长度，而`u2`是无符号的16位整数，因此理论上允许的的最大长度是`2^16=65536`。而 Java class 文件是使用一种变体`UTF-8`格式来存放字符的，`null` 值使用两个字节来表示，因此只剩下` 65536－ 2 ＝ 65534`个字节。~~
>
>```c
>CONSTANT_Utf8_info {
>u1 tag;
>u2 length;
>u1 bytes[length];
>}
>```
>
>**所以，在Java中，所有需要保存在常量池中的数据，长度最大不能超过65535，这当然也包括字符串的定义**
>
>上面提到的这种String长度的限制是*编译期的限制*，也就是使用`String s= “”;`这种字面值方式定义的时候才会有的限制。
>
>String在*运行期*有没有限制呢，答案是有的，就是我们前文提到的那个`Integer.MAX_VALUE `，这个值约等于4G，在运行期，如果String的长度超过这个范围，就可能会抛出异常。(在jdk 1.9之前）
>
>https://blog.csdn.net/u013380694/article/details/102739636

**一个String对象,占用大小（JDK1.8）为24 bytes**（不计算持有的char数组占用的大小）：

```java
/** The value is used for character storage. */
    private final char value[];  //一个数组对象的引用，占用4 bytes
/** Cache the hash code for the string */
    private int hash; // Default to 0   //一个int类型，占用4 bytes
```

再加上在64位JVM中，一个对象具有12 bytes的`对象头+引用`，要求对齐到8的倍数(来源[2.1. Objects, References and Wrapper Classes](https://www.baeldung.com/java-size-of-object#1-objects-references-and-wrapper-classes))，所以一个String对象的大小是：

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

简单计算一下这个`EnumClazz`的大小（不含引用对象的大小）：

```java
enumClassSize = 8 + 4 + 4*4 + 4 = 32 bytes
       对象头 +  引用 + 枚举类值的引用类型 * 4个 + 4 数组引用类型
```

然后，我们再看一下每个枚举类的值（以`EnumClazz.Day`为例）的大小：

`enum`类的每个值实际上都继承自`java.lang.Enum`类：

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

也就是说，本例中**每一个枚举类值占用24 bytes**，由此可以计算出`EnumClazz`实际占用的大小应该是：

```java
realSize = enumClassSize + daySize * 4  = 128 bytes
```

### Android中是否应该使用枚举

关于Android中使用枚举和常量所占用的大小对比*RomainGuy*有[下图](https://speakerdeck.com/romainguy/android-memories?slide=67)的对比。

![https://speakerdeck.com/romainguy/android-memories?slide=67](https://jixiaoyong.github.io/images/20191221172243.png)

关于是否应该在Android中使用枚举类，可以参考下文：

https://www.liaohuqiu.net/cn/posts/android-enum-memory-usage/

https://stackoverflow.com/a/29972028/8389461

总结起来其结论就是：

**当需要用到枚举类的特性时，比如非连续判断，方法重载等时就使用枚举，否则就使用占用内存更小的常量类。**

## SparseArray&ArrayMap VS HashMap

`HashMap`的数据是经过包装后保存在`HashMap.Node<K,V>`数组中。

下面是`HashMap`的结构：

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
  
  transient Node<K,V>[] table; //4+ bytes，保存HashMap的键值对等信息
  transient Set<Map.Entry<K,V>> entrySet; //4+ bytes
  transient int size; //4 bytes
  transient int modCount; //4 bytes
  int threshold; //4 bytes
  final float loadFactor; //4 bytes
  
  //继承自AbstractMap
  transient Set<K>        keySet; //4+ bytes
  transient Collection<V> values; //4+ bytes
}
```

再看看Android提供的`android.util.SparseArray`类(具体分析可参考：[SparseArray 的使用及实现原理](https://juejin.im/entry/57c3e8c48ac24700634bd3cf))

```java
public class SparseArray<E> implements Cloneable {
    private boolean mGarbage = false; //4 bytes

    private int[] mKeys; //4+ bytes
    private Object[] mValues; //4+ bytes
    private int mSize; //4 bytes
}
```

再结合[官方的描述](https://developer.android.google.cn/reference/android/util/SparseArray)，`SparseArray`类很明显要比`HashMap`占用更少的内存：

* 将`KEY`和`VALUE`直接保存在数组中，避免了将其包装为一个`Node`对象的开销
* 由于`SparseArray`类的key是`int`类型而非被自动装箱后的`Integer`对象，所以当同样使用`int`类型的`key`保存数据时，`SparseArray`类的`key`要占用更少的内存。

> `SparseArray` is intended to be more memory-efficient than a [`HashMap`](https://developer.android.google.cn/reference/java/util/HashMap), because it **avoids auto-boxing keys** and its data structure **doesn't rely on an extra entry** object for each mapping.
>
> https://developer.android.google.cn/reference/android/util/SparseArray

但是，`SparseArray`有以下局限性：

* 在每次`put/get/remove`的时候都需要使用二分法(`ContainerHelpers.binarySearch(mKeys, mSize, key)`)查找是否已经存在`KEY`对应的值（有的话查找其位置）

* 在添加和删除item的时候都需要在数组中增删条目（耗时，尽管为了优化性能，`SparseArray`在删除时只是将对于的值标记为`DELETED`，在下次更新该`KEY`对于的值时直接覆盖，或者在`GC`时删除）。

  `  private static final Object DELETED = new Object();`

  HashMap的删除涉及到数组、链表和红黑树（JDK1.8）

* **在容纳数百个项目时性能会比HashMap小大约50%**。

> 每当需要**增长数组**或**获取数组大小**或**获取条目值**时，都必须执行垃圾回收GC。

此外，还有以下可以替换HashMap的(数据来自[这里](https://stackoverflow.com/a/31413003/8389461))：

```java
SparseArray          <Integer, Object>
SparseBooleanArray   <Integer, Boolean>
SparseIntArray       <Integer, Integer>
SparseLongArray      <Integer, Long>
LongSparseArray      <Long, Object>
LongSparseLongArray  <Long, Long>   //this is not a public class                                 
                                    //but can be copied from  Android source code 
```

此外，还有`android.util.ArrayMap`其特性与`SparseArray`类似（**两者占用内存小，但是慢并且最好不要用来存储大容量的数据**），只不过它支持key值为其他类型，占用内存大小在`SparseArray`和`HashMap`之间(参考[这里](https://blog.csdn.net/u010687392/article/details/47809295))，此外`ArrayMap`的API和`HashMap`类似。



>根据[Romain Guy](https://speakerdeck.com/romainguy)的[计算](https://speakerdeck.com/romainguy/android-memories?slide=94)：
>
>保存1000个int对象的`SparseArray`                       占用大小为：8072 bytes
>
>保存1000个对象的`HashMap<Integer,Integer>`    占用大小为：64136 bytes
>
>几乎相差8倍！



综上，**当要保存的数据量比较小（小于几千个）的时候，如果KEY是基本类型，推荐使用`SparseArray`及其衍生类以节省内存，如果KEY是其他类型则使用`ArrayMap`;否则使用`HashMap`更加高效**。



# 参考资料

除文章中罗列的链接外：

https://blog.csdn.net/u013380694/article/details/102739636

[Sizeof for Java -- javaworld.com](https://www.javaworld.com/article/2077408/sizeof-for-java.html)

[RomainGuy-Android Memories](https://speakerdeck.com/romainguy/android-memories)（推荐）

[SparseArray vs HashMap](https://stackoverflow.com/a/31413003/8389461)

[Android内存优化（使用SparseArray和ArrayMap代替HashMap）](https://blog.csdn.net/u010687392/article/details/47809295)

[SparseArray 的使用及实现原理](https://juejin.im/entry/57c3e8c48ac24700634bd3cf)

[SparseArray -- developer.android.google.cn](https://developer.android.google.cn/reference/android/util/SparseArray)
