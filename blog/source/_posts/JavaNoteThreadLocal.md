---
title: Java笔记之ThreadLocal
tags: java
date: 2019-12-20 12:54:48

---

`ThreadLocal`是`Thread`中用来保存**线程私有变量**的数据结构。

一个`ThreadLocal`只能保存一个值，有`set/get/remove`方法。

在`Thread`有一个`threadLocals`（`ThreadLocal.ThreadLocalMap`）变量，该变量是一个定制的Hash Map，用来保存线程私有的数据（类型为`ThreadLocal<?> Key, Object Value`）。

# 特点

1. 一个`Thread`可以有多个`ThreadLocal`变量

2. **不同`Thread`可以通过一个`ThreadLocal`变量分别保存不同的变量而互不影响**。

3. 如果不同的`Thread`使用的`ThreadLocal`变量保存的是**同一个引用类型的对象**（假设为`obj`），无论这些`Thread`使用的是同一个`ThreadLocal`对象还是完全不同的`ThreadLocal`对象，只要`obj`指向的对象改变，其余线程中的`ThreadLocal`对象也会访问到`obj`的最新值。

# 使用与解析

当我们新建一个`ThreadLocal`并为之赋值时

```java
// 方式1.
ThreadLocal threadLocal = new ThreadLocal();
threadLocal.set("value");
//方式2.
ThreadLocal threadLocal2 = new ThreadLocal(){
            @Override
            protected Object initialValue() {
                return "initial value";
            }
        };
```

这个时候就会调用`set()`方法（方式1）或者`setInitialValue()`方法（方式2）。

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//注意这里获取到是线程本身的threadLocals对象
        if (map != null)
            map.set(this, value);//ThreadLocal对象只是在Thread所属的threadLocals中充当一个key，
            //所以即使在其他线程执行threadLocal.set(value);
            //也只是更新该线程本身的threadLocal对应的value，而不会影响其他线程分毫！！！（好精巧的设计）
        else
            createMap(t, value);
    }

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

可以看到这两个方法到最后都相当于调用了`Thread`对象的`threadLocals`的`set(ThreadLocal<?> key, Object value)`方法，这个方法最终以`ThreadLocal`对象为KEY，将数据保存到了`Thread`对象自己的`threadLocals`中。

```java
private Entry[] table;

private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    //其他逻辑...
    tab[i] = new Entry(key, value);
}
```

所以即使是**同一个`ThreadLocal`对象，在不同的线程中进行`set/get/remove`都只是更新了本线程中`ThreadLocal`对象对应的值**。

```java
        //MainThread
        final ThreadLocal threadLocal = new ThreadLocal();
        threadLocal.set("THREAD-Main-"+Thread.currentThread().getName());
        System.out.println("THREAD-Main-BEFORE："+threadLocal.get());//THREAD-Main-BEFORE：THREAD-Main-main

        new Thread(new Runnable() {
            public void run() {
                threadLocal.set("THREAD-1-"+Thread.currentThread().getName());
                System.out.println("THREAD-1："+threadLocal.get());//THREAD-1：THREAD-1-Thread-0
            }
        }).start();


        new Thread(new Runnable() {
            public void run() {
                System.out.println("THREAD-2："+threadLocal.get());//THREAD-2：null
                //本线程中threadLocal没有赋值，所以为null
            }
        }).start();

        //MainThread
        System.out.println("THREAD-Main-AFTER："+threadLocal.get());//THREAD-Main-AFTER：THREAD-Main-main 
        //其他线程对threadLocal对象的操作不会影响本线程
        //但是如果threadLocal保存的是一个引用类型的对象，并且这个对象在其他线程被更改，那么本线程获取到的也会是变更后的值
    }
```



# 内存泄漏

在`ThreadLocal.ThreadLocalMap`中，最终用来保存`ThreadLocal`以及对应值的是一个`Entry`数组：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);//对ThreadLocal对象的弱引用
                value = v;
            }
        }
```

从上面的代码可以看到，`Entry`对`ThreadLocal`是**弱引用**，按照“强软弱虚”引用的等级来划分，每次GC的时候，如果这个`ThreadLocal`对象没有被引用，就会被回收掉，这时如果该`Thread`还在运行，那么`threadLocals`中保存的`ThreadLocal<?> k`已经被回收了，但是`Object v`对象仍然保存在`threadLocals`中但是没有办法再访问到，造成内存泄漏。

解决方法参考：

> 使用`ThreadLocal`时会发生内存泄漏的前提条件：
>
> ①`ThreadLocal`引用被设置为`null`，且后面没有`set，get,remove`操作。
>
> ②线程一直运行，不停止。（线程池）
>
> ③触发了垃圾回收。（Minor GC或Full GC）
>
> 我们看到`ThreadLocal`出现内存泄漏条件还是很苛刻的，所以我们只要破坏其中一个条件就可以避免内存泄漏，单但为了更好的避免这种情况的发生我们使用`ThreadLocal`时遵守以下两个小原则:
>
> ①`ThreadLocal`申明为`private static final`。 `Private`与`final `尽可能不让他人修改变更引用， `Static `表示为类属性，只有在程序结束才会被回收。
>
> ②`ThreadLocal`使用后务必调用`remove`方法。 最简单有效的方法是使用后将其移除。
>
> 版权声明：本文为CSDN博主「pony-zi」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。 原文链接：https://blog.csdn.net/zzg1229059735/article/details/82715741

# 总结

现在我们知道了，所谓的*通过`ThreadLocal`实现线程本地变量与其他线程隔离*，是在创建`ThreadLocal`的时候，**保存的就是属于当前线程的独立的变量，并且之后的修改也不会（无法）修改到其他线程中对应的值**，但如果`ThreadLocal`本身保存的都是同一个对象，则这个对象在所有的线程中还是共享的。

# 参考资料

[Android线程管理之ThreadLocal理解及应用场景](https://www.cnblogs.com/whoislcj/p/5811989.html)

[ThreadLocal深入分析（Jdk 1.8）](https://blog.csdn.net/shenyo/article/details/80391818)
