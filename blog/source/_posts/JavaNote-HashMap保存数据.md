---
title: Java笔记之HashMap保存数据
tags: java
date: 2019-12-15 16:41:35
---

<img src="https://images.pexels.com/photos/159644/art-supplies-brushes-rulers-scissors-159644.jpeg?cs=srgb&dl=art-supplies-arts-and-crafts-ballpens-159644.jpg&fm=jpg" class="full-image" />

Photo by **[Pixabay ](https://www.pexels.com/@pixabay?utm_content=attributionCopyText&utm_medium=referral&utm_source=pexels)**from **[Pexels](https://www.pexels.com/photo/pencils-in-stainless-steel-bucket-159644/?utm_content=attributionCopyText&utm_medium=referral&utm_source=pexels)**



`HashMap`使用由`Node<K,V>`（继承自`Map.Entry<K,V>`）组成的**数组**`table`保存数据。

在`table`中保存数据时根据`key`的`hashCode`计算到一个**随机保存位置（但都在`table`数组的大小范围内）**，当存储的**数据总量**超过加载系数`loadFactor`规定的**阈值**时则对`table`进行**扩容**。

# HashMap有以下全局变量

```java
transient Node<K,V>[] table;//实际保存键值对的数组
transient Set<Map.Entry<K,V>> entrySet;//Holds cached entrySet().用来遍历HashMap
transient int size;//本HashMap实际保存的键值对个数
transient int modCount;//HashMap修改的次数，每次修改HashMap都会叠加，
//用来在遍历的过程中检查HashMap是否被改动过来，如果有则抛出异常ConcurrentModificationException
int threshold;//是否扩容的阈值
final float loadFactor;//加载系数,默认0.75f
```

> `loadFactor`：默认的负载因子**0.75**是对空间和时间效率的一个平衡选择，建议大家不要修改，除非在时间和空间比较特殊的情况下：
>
> 如果内存空间很多而又对时间效率要求很高，可以降低负载因子Load factor的值；
>
> 相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子`loadFactor`的值，这个值可以大于1。
>
> https://tech.meituan.com/2016/06/24/java-hashmap.html

每个`Node`包含了以下信息：

```java
Node<K,V> implements Map.Entry<K,V>{
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
  }
```

在执行`hashMap.put("k", "v");`时，会先计算`key`的`hash`值

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  //这里原始hash值(32位)的高位和地位进行按位异或（不同为，相同为0），
  //增加了随机性，避免因为hashCode计算得到的hash值（低位相同概率高）
  //计算索引时（见下文↓）一直取低位值而可能导致的索引一直的重复问题。
    }
```



# `V put(K key, V value)`

使用`HashMap`保存数据时：

1. 使用`hash(Object key)`计算`key`的`hash`值

2. 通过`hash`值计算`value`应该保存的位置`i`

   ```java
   i = (table.length - 1) & hash
   //由于table.length限定为2的n次方，所以上面的等式相当于给table.length取余数
   //即i永远<=table.length
   ```

   在此时会判断是否需要扩容(**只有`table`为空，或者当前存储的数据总数`size`大于阈值`threshold`时才会扩容**`resize()`)

   ```java
   threshold = capacity * loadFactor
   阈值 = 容量 * 负载系数（默认为0.75）
   ```

3. 接下来会插入数据

   * 指定位置为空（没有*hash冲突*），或已有`key`相同的值：则直接插入`value`
   * 已经存在值并且数量大于8：则将链表转化为红黑树（JDK1.8），否则以链表形式保存数据
   * 在移除数据时，如果红黑树数量小于6：则将红黑树转化为链表

> 在JDK1.7中，数据以数组或链表形式保存，JDK1.8中则新增了红黑树。
>
> 发生hash冲突时，JDK1.7采用采用头插法，可能会产生逆序和环形链表；JDK1.8采用尾插法，直接插入链表或红黑树尾部。
>
> 具体JDK1.7与1.8对比查看[这里](https://blog.csdn.net/qq_36520235/article/details/82417949)

# `V get(Object key)`

使用`HashMap`获取数据时:

1. 计算key的`hash值`

2. 查找对应位置的`node`

   * `null`：返回`null`

   * `node`不为空且`key`一致：返回该`node`

   * `node`不为空且`key`不一致：

     如果是*链表*：遍历链表查找是否存在与`key`一致的`node`

     如果是*树*：遍历树查找是否存在与`key`一致的`node`

# `V remove(Object key)`

使用`HashMap`移除数据时:

其大体过程与`get(Object key)`类似，遍历找到对应的`node`并删除。

# 计算索引

一个`key`对应的索引`index`是由这个`key`的`hash()`值对`HashMap`的数组长度`length`的余数：

```java
index = hash % length;
```

又有**在`Length`为2^n^**时：

hash % 2^n^ =  hash & ( 2^n^ - 1)

> hash % 2^n^ = hash - (hash / 2^n^) * 2^n^ 
= hash - (hash>>n) * 2^n^ 
= hash & ( 2^n^ - 1)

而`HashMap`的长度`Length`又只能是2^n^，所以：

```java
index = (length - 1) & hash;
```

# 保存值

* 当`table`为空或者长度超过加载因子`DEFAULT_LOAD_FACTOR`规定的容量(默认容量为16，加载因子为0.75)时会自动扩容。

* 当`table[index]`为空时，直接新建`Node`并保存到`table[index]`中。

* 当`table[index]`不为空时：
  * 如果是同一个`key`则覆盖旧的值
  * 如果是不同的`key`则先尝试以链表保存数据
  * 如果是不同的`key`，并且链表长度超过`MIN_TREEIFY_CAPACITY`规定的长度（默认64），则将链表转化为红黑树(JDK1.8新增)

# 序列化

在[第一节](#HashMap有以下全局变量)我们可以看到，`HashMap`的很多变量都被标记为`transient`，这表示在`Serializable`序列化时不主动去序列化这些值，那这样岂不是没法反序列化这些数据了？

其实在后面我们可以看到，**`HashMap`在`writeObject()`方法中主动保存了部分数据**（原因是默认的`Serializable`由于不同JVM实现对同一对象如`String`的`HashCode`不一定一致，会导致严重的问题——`HashMap`基于`hash`值保存数据）：

```java
private void writeObject(java.io.ObjectOutputStream s)
    throws IOException {
    int buckets = capacity();//容量
    // Write out the threshold, loadfactor, and any hidden stuff
    s.defaultWriteObject();
    s.writeInt(buckets);
    s.writeInt(size);//保存size
    internalWriteEntries(s);//保存table数据
}

/**
* table不为空则返回table长度
* 否则threshold不为空则返回threshold
* 否则返回默认的DEFAULT_INITIAL_CAPACITY
*/
final int capacity() {
        return (table != null) ? table.length :
            (threshold > 0) ? threshold :
            DEFAULT_INITIAL_CAPACITY;
}
```

并在`readObject()`恢复了这些值。

# 位运算

| 位运算     | 符号   | 计算             |
| ---------- | ------ | ---------------- |
| 按位与     | &      | 相同为1，不同为0 |
| 按位或     | &#124; | 有1则1           |
| 按位异或   | ^      | 相同为0，不同位1 |
| 按位取反   | ~      |                  |
| 左移       | <<     | 相当于乘以2^n^   |
| 右移       | `>>`   | 相当于除以2^n^   |
| 无符号右移 | `>>>`  |                  |



# 参考资料

[HashCode计算扰动分析-关于hashMap的一些按位与计算的问题？ - 胖君的回答 - 知乎 ](https://www.zhihu.com/question/28562088/answer/111668116)

[一文读懂Java之HashMap索引位置计算](http://baijiahao.baidu.com/s?id=1646023968436883100&wfr=spider&for=pc)

[hashMap在jdk1.7与jdk1.8中的原理及不同](https://blog.csdn.net/changhangshi/article/details/82114727)

[真实面试题之：Hashmap的结构，1.7和1.8有哪些区别](https://blog.csdn.net/qq_36520235/article/details/82417949)

[Java 8系列之重新认识HashMap](https://tech.meituan.com/2016/06/24/java-hashmap.html)

[java.util.HashMap源码要点浅析](http://www.blogjava.net/killme2008/archive/2009/04/15/265721.html)

[Why HashMap.Entry is transient?](https://coderanch.com/t/469720/java/HashMap-Entry-transient)

[Java中HashMap关键字transient的疑惑](https://segmentfault.com/q/1010000000630486)
