---
title: Android中的SpareArray和ArrayMap实现分析
tag: android
date: 2019-12-22 09:35:37
---

日常开发中，常用的存储键值对的数据结构是`HashMap`，根据[Java笔记之HashMap保存数据](https://xiaoyong.ml/blog/posts/ff927bd4/)和[Java笔记之计算Java对象的大小及其应用](https://xiaoyong.ml/blog/posts/b0793c74/)可以知道，`HashMap`存储**键值对**会占用比较多的内存控件，而对于内存限制较大的Android平台来说，为了避免这种浪费，官方推荐我们使用`SpareArray`和`ArrayMap`，本文对这两个类的实现进行分析比较。

**`SpareArray`以及他的衍生类**都是以**基本类型**为`key`，因为避免了*自动装箱*，并且**用数组直接保存key、value**（而非像`HashMap`那样将其封装为`Node`对象后再保存），因而节省了内存。

**`ArrayMap`**则支持**所有类型的key**，他是**将`key`和`value`全部保存在一个数组中**（`n`位为`key`，`n+1`位为`value`），避免了将其封装为`Node`对象带来的内存消耗。

**当要保存的数据量比较小（小于几千个）的时候，如果KEY是基本类型，推荐使用`SparseArray`及其衍生类以节省内存，如果KEY是其他类型则使用`ArrayMap`;否则使用`HashMap`更加高效**。

# SpareArray

`SpareArray`以及他的衍生类主要用于**以`基本类型`为`key`保存非大量数据的场景**。

相比`HashMap`而言，他的优点主要在于**没有对保存的数据二次封装**，没有对基本类型的数据**自动装箱**，存储单个数据的成本小，也没有`hash`计算。

但他在添加数据时需要扩展数组(涉及到新建、复制数组，`gc()`等)，~~在删除数据时需要缩减数组~~ (查看`gc()`等源码发现他的数组只会增加，不会缩减)，以及通过二分法查找索引都会消耗性能。

> 为了避免每次删除时都需要缩减数组，`SpareArray`在删除数组时只会将其赋值为`DELETED`，在下次调用其`private void gc()`方法时丢弃掉这些数据

先看一下`SpareArray`的结构：

```java
public class SparseArray<E> implements Cloneable {
    private boolean mGarbage = false; //是否调用gc()方法

    private int[] mKeys;//所有的key
    private Object[] mValues;//所有的value
    private int mSize;//所保存的数据个数
}
```

## void put(int key, E value)

添加方法先用**二分法**查找`key`对应的位置：

* 如果有，则直接覆盖
* 如果没有，则**取反**得到应该`插入的位置`，并分别插入`key`和`value`

```java
public void put(int key, E value) {
  // 先用二分法查找key对应的索引,找到的话返回对应索引，
  // 否则返回key应该插入的位置的取反值
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
      // 如果已存在值则直接覆盖
        mValues[i] = value;
    } else {
      // 对二分法查找到的值再取反，得到key应该插入的位置
        i = ~i;

        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }

        if (mGarbage && mSize >= mKeys.length) {
            gc();

            // Search again because indices may have changed.
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }

      // public static <T> T[] insert(T[] array, int currentSize, int index, T element)
      // Inserts an element into the array at the specified index,
      // growing the array if there is no more room.
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```

## E get(int key) 

获取数据，先用**二分法**查找，如果找到就返回`对应的值`，否则返回`null`。

```java
public E get(int key) {
        return get(key, null);
}

public E get(int key, E valueIfKeyNotFound) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
}
```

## void remove(int key)

删除`key`以及`对应的数据`。

同样先用**二分法**查找`对应位置`，有的话则标记为`DELETED`，**等待下次`gc()`时丢弃**。

```java
public void remove(int key) {
        delete(key);
}

public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}
```

## gc()

在上文中我们看到，删除数据时，`mGarbage`被标记为`true`，这样当下一次进行`put/valueAt/append/size`等涉及到数组大小查询、改动等时，就出触发`gc()`以便整理数组结构。

```java
private void gc() {
    // Log.e("SparseArray", "gc start with " + mSize);

    int n = mSize;
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;

    for (int i = 0; i < n; i++) {
        Object val = values[i];

      // 这里的操作只是将没有被删除的数据移动到了数组的前面
      // 而保证了数组后面都是DELETED或null，方便后续操作
        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                values[i] = null;
            }

            o++;
        }
    }

    mGarbage = false;
    mSize = o;
}
```

## HashMap与SpareArray及其衍生类对应关系

参考[下图](https://android.jlelse.eu/app-optimization-with-arraymap-sparsearray-in-android-c0b7de22541a)

![](https://jixiaoyong.github.io/images/20191222131219.png)

# ArrayMap

`ArrayMap`实现了`Map<K, V>`接口，他的API和`HashMap`相差无几，但是由于**没有对数据再包装**，**动态调整数组的大小**，一定范围内他比`HashMap`内存效率高。

但是如果保存大量数据（超过千位）时，由于他需要**二分法查找**的影响会比`HashMap`慢很多。

> **`ArrayMap`特殊之处在于将`key`，`value`保存到了同一个数组mArray中（n位保存key，n+1位保存value**）。

先看一下`ArrayMap`的结构：

```java
static Object[] mBaseCache;
static int mBaseCacheSize;
static Object[] mTwiceBaseCache;
static int mTwiceBaseCacheSize;

final boolean mIdentityHashCode;//是否强制使用System.identityHashCode(key)获取key的HashCode
//System.identityHashCode(key)方法无论类是否重写了hashCode()方法，
//都会调用Object.identityHashCode(key)来获取对象的hashCode
int[] mHashes;//存储所有key的hash值
Object[] mArray;//存储key和value，大小是mHashes的两倍
//n位保存key，n+1位保存value
int mSize;
MapCollections<K, V> mCollections;
```

在使用时：

1. 计算`key`的`hash值`，

   ```java
   hash = mIdentityHashCode ? System.identityHashCode(key) : key.hashCode();
   ```

2. 然后使用`indexOf()`在`mHashes`中进行二分法查找对应的`index`

   ```java
   index = indexOf(key, hash);
   ```

   `indexOf()`方法会先用**二分法**查找`hash`对应的`index`,如果`index<0`则返回`index`；否则在对比`mArray`中对应位置`mArray[index<<1]`的`key`与要`查询的key`：

   * 两者一致：返回`index`
   * 两者不一致：从`index`开始，**先向后，再向前**查询是否有相同的`key`,如果有返回`对应index`
   * 以上都没有找到：**对`mHashes`中`最后一个与key的hash一致的后一位index`取反**，并返回

   ```java
   int indexOf(Object key, int hash) {
       final int N = mSize;
   
       // Important fast case: if nothing is in here, nothing to look for.
       if (N == 0) {
           return ~0;
       }
   
       int index = binarySearchHashes(mHashes, N, hash);
   
       // If the hash code wasn't found, then we have no entry for this key.
       if (index < 0) {
           return index;
       }
   
       // If the key at the returned index matches, that's what we want.
       if (key.equals(mArray[index<<1])) {
           return index;
       }
   
       // Search for a matching key after the index.
       int end;
       for (end = index + 1; end < N && mHashes[end] == hash; end++) {
           if (key.equals(mArray[end << 1])) return end;
       }
   
       // Search for a matching key before the index.
       for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
           if (key.equals(mArray[i << 1])) return i;
       }
   
       // Key not found -- return negative value indicating where a
       // new entry for this key should go.  We use the end of the
       // hash chain to reduce the number of array entries that will
       // need to be copied when inserting.
       return ~end;
   }
   ```



## V put(K key, V value)

当添加`item`时，按照前述规则，先在`mArray`中查找`key`对应的`索引index`：

* `index >= 0` ：已经有键为`key`的数据，直接覆盖旧值并返回

* `index < 0 `：没有键为`key`的数据，对数组进行扩容，并保存对应数据

  ```java
  index = ~index;//上文indexOf()中计算得出的key应该添加的位置
  mHashes[index] = hash;
  mArray[index<<1] = key;
  mArray[(index<<1)+1] = value;
  ```

## V get(Object key)

`get()`方法就比较简单了，先查找`key`的索引，然后取出`对应的数据value`并返回即可：

```java
    public V get(Object key) {
        final int index = indexOfKey(key);
        return index >= 0 ? (V)mArray[(index<<1)+1] : null;
    }
```

## V remove(Object key)

`remove()`方法也会先使用`indexOfKey()`计算`key的index`，然后删除`对应位置的数据`。

此外，如果`mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3`的话，还会缩减`数组的大小`为`osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2)`：

```java
public V removeAt(int index) {
  //...其他代码
  if(mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3){
    //...其他代码
    final int n = osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2);
    allocArrays(n);
  }
  //...其他代码
}

 private void allocArrays(final int size) {
   //...其他代码
   mHashes = new int[size];
   mArray = new Object[size<<1];
 }
```



# 参考资料

[GrowingArrayUtils.java 源码](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/util/GrowingArrayUtils.java)

[SparseArray.java 源码](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/util/SparseArray.java)

[ArrayMap.java 源码](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/util/ArrayMap.java)

[App optimization with ArrayMap & SparseArray in Android](https://android.jlelse.eu/app-optimization-with-arraymap-sparsearray-in-android-c0b7de22541a)

[Java笔记之HashMap保存数据](https://xiaoyong.ml/blog/posts/ff927bd4/)

[Java笔记之计算Java对象的大小及其应用](https://xiaoyong.ml/blog/posts/b0793c74/)

[SparseArray 的使用及实现原理](https://extremej.itscoder.com/sparsearray_source_analyse/)