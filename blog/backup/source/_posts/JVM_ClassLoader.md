---
title: JVM类加载机制之ClassLoader
date: 2018-04-15 16:40:32
tags: JVM_ClassLoader
---

> 本文为[《一看你就懂，超详细java中的ClassLoader详解 - CSDN博客》]( https://blog.csdn.net/briblue/article/details/54973413)阅读笔记



# 简述

JVM有三种类加载器：

1. **BootStrap ClassLoader** 加载核心类库，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。
2. **Extention ClassLoader** 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。
3. **App ClassLoader** 加载当前应用的classpath的所有类。

# 加载过程

在加载类时，通过**“双亲委托”**机制，依次从**1** -> **3**向上查询，再从**3**->**1**依次返回结果：

1. 调用`findLoadedClass(className) `查询是否已经加载该类
2. 调用父加载器的`loadClass(className,false)`，若父加载器为空，则调用`BootStrap ClassLoader`
3. 如果还是没有加载到该类，调用`findClass(className)`

这样子保证了每个类都是先经过最顶端的类加载器`BootStrap ClassLoader`，如果没有加载到再依次经过`Extention ClassLoader`、`App ClassLoader` 加载，确保如String等关键类不会被自定义的ClassLoader加载而导致异常。

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检测是否已经加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //父加载器不为空则调用父加载器的loadClass
                        c = parent.loadClass(name, false);
                    } else {
                        //父加载器为空则调用Bootstrap Classloader
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    //父加载器没有找到，则调用findclass
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                //调用resolveClass()
                resolveClass(c);
            }
            return c;
        }
    }
```

`AppClassLoader`和`ExtClassLoader`都继承自`URLClassLoader`

`AppClassLoader`的父加载器是`ExtClassLoader`，`ExtClassLoader`的父加载器为`null`，故而会调用`BootStrap ClassLoader`

ClassLoader如果没有指定父加载器，则默认的父加载器为`AppClassLoader`，自定义ClassLoader也是如此。



# 自定义ClassLoader

自定义ClassLoader一般步骤：

1. 继承自`ClassLoader`
2. 重写`findClass()`
3. 在`findClass()`方法中调用并返回`defineClass()`

```kotlin
class MClasLoader:ClassLoader(){

    override fun findClass(name: String?): Class<*> {

        var sysDir = System.getProperty("user.dir")
        var classPath = "$sysDir/src/main/res/$name.class"

        var classFile = File(classPath)
        //【注意】这里一次只读取一个字节，否则会报错java.lang.ClassFormatError:
        // Extra bytes at the end of class file TestClass
        var bytes = ByteArray(1)
        var fileInputStream = FileInputStream(classFile)
        var len = -1
        var byteBuffer = ByteOutputStream()
        while (true) {

            len = fileInputStream.read(bytes)
            if (len > 0) {
                byteBuffer.write(bytes)
            } else {
                break
            }
        }
        var byteArr = byteBuffer.toByteArray()
        return defineClass(name,byteArr,0,byteArr.size)
    }
}

fun main(args: Array<String>) {

    var clazz = MClasLoader().loadClass("TestClass")
    var say = clazz.getDeclaredMethod("say",String::class.java)
    say.invoke(clazz.newInstance(),"hello world")

    print(MClasLoader().parent)
}
```

而`defineClass()`则将一个字节数组转化为一个类的实例（Converts an array of bytes into an instance of class with an optional ProtectionDomain）

# contextClassLoader

每个线程都有一个ClassLoader：`contextClassLoader`，通过将其设置为自定义的ClassLoader可以在加载类的时候做一些特殊的事情。

```kotlin
    Thread.currentThread().contextClassLoader = MClasLoader()
    var clazz = Class.forName("TestClass",
            true, Thread.currentThread().contextClassLoader)
    var say = clazz.getDeclaredMethod("say",String::class.java)
    say.invoke(clazz.newInstance(),"hello world")

    println(Thread.currentThread().contextClassLoader )
    println(Thread.currentThread().contextClassLoader.parent )
```

结果为：

```
hello world
MClasLoader@1d44bcfa
sun.misc.Launcher$AppClassLoader@18b4aac2
```