---
title: Java笔记之HashMap保存数据
tags: java
date: 2019-12-15 16:41:35
---

`HashMap`使用由`Map.Entry<K,V>`组成的**数组**`table`保存数据。

> 在JDK1.7中，数据以数组或链表形式保存，JDK1.8中则新增了红黑树。
>
> 发生hash冲突时，JDK1.7采用采用头插法，可能会产生逆序和环形链表；JDK1.8采用尾插法，直接插入链表或红黑树尾部。
>
> 具体JDK1.7与1.8对比查看[这里](https://blog.csdn.net/qq_36520235/article/details/82417949)

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

# 位运算

| 位运算     | 符号  | 计算             |
| ---------- | ----- | ---------------- |
| 按位与     | &     | 相同为1，不同为0 |
| 按位或     | \|    | 有1则1           |
| 按位异或   | ^     | 相同为0，不同位1 |
| 按位取反   | ~     |                  |
| 左移       | <<    | 相当于乘以2^n^   |
| 右移       | `>>`  | 相当于除以2^n^   |
| 无符号右移 | `>>>` |                  |



# 参考资料

[HashCode计算扰动分析-关于hashMap的一些按位与计算的问题？ - 胖君的回答 - 知乎 ](https://www.zhihu.com/question/28562088/answer/111668116)

[一文读懂Java之HashMap索引位置计算](http://baijiahao.baidu.com/s?id=1646023968436883100&wfr=spider&for=pc)

[hashMap在jdk1.7与jdk1.8中的原理及不同](https://blog.csdn.net/changhangshi/article/details/82114727)

[真实面试题之：Hashmap的结构，1.7和1.8有哪些区别](https://blog.csdn.net/qq_36520235/article/details/82417949)
