---
title: Java笔记之序列化与反序列化：Serializable、Externalizable和Parcelable
tag: java
date: 2019-12-24 09:38:59

---

<img src="https://images.pexels.com/photos/2881229/pexels-photo-2881229.jpeg?cs=srgb&dl=white-and-blue-cables-2881229.jpg&fm=jpg" class="full-image" />
Photo by **[Pixabay ](https://www.pexels.com/@pixabay?utm_content=attributionCopyText&utm_medium=referral&utm_source=pexels)**from **[Pexels](https://www.pexels.com/photo/close-up-of-telephone-booth-257736/?utm_content=attributionCopyText&utm_medium=referral&utm_source=pexels)**



**序列化**：指将`Java对象`转化为`字节流`以便在网络、文件中保存、传输。

**反序列化**：指的是从字节流中恢复Java对象。

本文主要讨论Android和Java中实现序列化的4种方式，并探讨一下其实现原理。



Android & Java中实现序列化的方式有：

* `android.os.Parcelable` Android平台特有，需要自己实现具体细节，性能消耗小，只能在内存中存在
* `java.io.Serializable` 实现简单，只需要实现`Serializable`接口即可，可以输出到文件、网络等
* `java.io.Externalizable` 需要自己实现具体细节
* [`Twitter Serial`](https://github.com/twitter/Serial/blob/master/README-CHINESE.rst/) Twitter出品的高性能序列化方案，它力求帮助开发者实现高性能和高可控的序列化过程。（本文不详细介绍，可以参考[这篇文章](https://www.jianshu.com/p/1b42608478c0)）

# Serializable

`Serializable`接口没有任何方法，只是一个标记——表示这个类可以用来`序列化/反序列化`（由`ObjectOutputStream/ObjectInputStream`实现具体细节）。

一个类没有实现`Serializable接口`，或者包含没有`实现Serializable接口的变量`，则会序列化失败`NotSerializableException`。

## serialVersionUID

使用`serialVersionUID`标记当前`Serializable`的版本。

如果没有指定，系统会自动用对象的`hashCode()`指定`serialVersionUID`，该值会在类发生改变时变化，从而导致反序列化失败。

而如果`serialVersionUID`一致，即使类结构有变化，也会反序列化（给新增的变量默认值），所以最好赋予一个默认的值。

```java
//可以手动指定，也可以随机数，只要保持一致即可，如果不一致则会使反序列化失败
ANY-ACCESS-MODIFIER static final long serialVersionUID = 1L;
```

## readResolve()

如果`class`实现了`readResolve()`方法，会在反序列化时用到并返回这里提供的对象（反序列化得到的对象会被丢弃）。

```java
// 1. 反序列化
SerializableClass serializableClass = (SerializableClass) objectInputStream.readObject();
// 2.readObject()内部调用了readObject0(false):
private Object readObject0(boolean unshared) throws IOException {
        // ...
       
        try {
            switch (tc) {
                // 这里匹配了TC_NULL,TC_REFERENCE,TC_CLASS,TC_CLASSDESC,
                // TC_PROXYCLASSDESC,TC_STRING,TC_LONGSTRING,TC_ARRAY,TC_ENUM
                // TC_EXCEPTION,TC_BLOCKDATA,TC_BLOCKDATALONG,TC_ENDBLOCKDATA等等类型
               
                case TC_OBJECT://如果是OBECJT类型，就调用下面的方法👇
                    return checkResolve(readOrdinaryObject(unshared));
                // ... 
                default:
                    throw new StreamCorruptedException(
                        String.format("invalid type code: %02X", tc));
            }
        } finally {
            depth--;
            bin.setBlockDataMode(oldMode);
        }
    }
// 3. 在这里会检测是否存在readResolve()方法，有的话就返回从readResolve()获取的对象
private Object readOrdinaryObject(boolean unshared)
        throws IOException
    {
        // ...
        Object obj;
        // ...
        // 看这里，如果hasReadResolveMethod()为真则执行invokeReadResolve()并返回其结果
        if (obj != null &&
            handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod())
        {
            Object rep = desc.invokeReadResolve(obj);
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                // Filter the replacement object
                if (rep != null) {
                    if (rep.getClass().isArray()) {
                        filterCheck(rep.getClass(), Array.getLength(rep));
                    } else {
                        filterCheck(rep.getClass(), -1);
                    }
                }
                handles.setObject(passHandle, obj = rep);
            }
        }

        return obj;
    }
// Invokes the readResolve method of the represented serializable class and returns the result.
    Object invokeReadResolve(Object obj) throws IOException, UnsupportedOperationException{}
```

通过这个特性我们可以确保在反序列化的时候也能实现**单例**：

```
private Object readResolve() throws ObjectStreamException {
    return this;//返回单例本身，而非新建的对象
}
```

但是根据下面的说法，要实现可以序列化的单例最简单安全的，还是使用枚举：

> **事实上，如果依赖readResolve进行实例控制，带有对象引用类型的所有实例域则都必须声明为transient的**。否则，利用`readResolve()`方法实现的单例也会遭受到攻击。
>
> 实现可序列化最简单安全的方式是采用枚举的形式，应该尽可能采用这种方式。如果采用`readResolve`实现的话，要确保该类的所有实例域都为`基本类型`，或者是`transient`的。
>
> [77.单例模式，枚举类型优先于readResolve](https://cl0610.github.io/effective-java-learning/第十章 序列化/77.单例模式，枚举类型优先于readResolve.html)

## 自定义序列化过程

如果想要自己处理序列化的过程，可以实现下面的方法：

```java
private void writeObject(java.io.ObjectOutputStream out) throws IOException 
private void readObject(java.io.ObjectInputStream in) throws IOException, ClassNotFoundException;
private void readObjectNoData() throws ObjectStreamException;
```

其中，可以使用下面的方法实现`读/写`**该类自身的属性**（`All non-static and non-transient fields of the current class, include private`），然后在调用诸如`out.writeObject(string);`等方法**保存其他变量**。

> The method does not need to concern itself with the state belonging to its superclasses or subclasses.

```java
in.defaultReadObject();
out.defaultWriteObject();
```

`readObjectNoData`方法主要用在序列化流和我们要反序列化的类不一致时初始化一些必要的状态。

> 这种情况可能出现在接收方使用了一个与发送方不同版本的类。接收方的版本多扩展了一些字段，而发送方的版本没有这些字段。还有一种可能就是序列化流被篡改了。这时，无论是恶意的流还是不完整的流，都可以用 `readObjectNoData` 方法来将序列化得到的对象初始化到正确的状态。
> 作者：福尔马林
> 链接：https://juejin.im/post/5d7206c5f265da03ab427181

此外，还可以使用`ObjectOutputStream`的`putFields()`和`ObjectInputStream`的`readFields()`写入/读取变量。使用这种方法可以*加密/解密*部分变量，或者在序列化的时候*只处理部分变量*。

具体使用方法见如下：

> **注意**：`putFields()/readFields()`方法分别不能与`defaultWriteObject/defaultReadObject`一起使用；
>
> `putFields.put()`之后必须调用`out.writeFields()`方法
>
> 并且，没有在`putFields()`中加入的数据，在`readObject`中只能获取到该类型的默认值

```java
//这段示例代码来自 https://www.ibm.com/developerworks/cn/java/j-lo-serial/index.html
    private void writeObject(ObjectOutputStream out) {
       try {
           PutField putFields = out.putFields();
           System.out.println("原密码：" + password);
           password = "encryption";//模拟加密
           putFields.put("password", password);
           System.out.println("加密后的密码：" + password);
           out.writeFields();// putFields.put()之后必须调用本方法
       } catch (IOException e) {
           e.printStackTrace();
       }
   }
 
   private void readObject(ObjectInputStream in) {
       try {
           GetField readFields = in.readFields();
           Object object = readFields.get("password", "");
           System.out.println("要解密的字符串：" + object.toString());
           password = "pass";//模拟解密,需要获得本地的密钥
       } catch (IOException e) {
           e.printStackTrace();
       } catch (ClassNotFoundException e) {
           e.printStackTrace();
       }
 
   }
// 执行反序列化结果：
// 原密码：pass
// 加密后的密码：encryption
// 要解密的字符串：encryption
// 最后反序列化后的password为pass
```



## 父类未继承Serializable的类的序列化

如果一个类实现了序列化，但他的父类没有实现序列化，那么父类**必须要有一个公开的无参构造函数**，否则反序列化时会出错。

此时反序列化时，父类的变量值（`public, protected, and (if accessible) package fields`）都会是**默认的值**或者是在**父类无参构造函数中初始化的值**（即使这些值在子类对象中已经被修改了）。

要想使得这些值也可以支持序列化，可以通过`writeObject/readObject`自己处理这些值的序列化。

反之，如果一个类实现了`Serializable`接口，那么他的子类也自动支持序列化与反序列化。

## 实现

下面是使用`Serializable`实现序列化与反序列化的简单示例：

```java
/**
 * author: jixiaoyong
 * email: jixiaoyong1995@gmail.com
 * website: https://jixiaoyong.github.io
 * date: 12/24/19
 * description: 演示序列化功能
 */
class SerializableClass implements Serializable {
    private int anInt = 10;
    public long aLong = 100L;
    public transient String aTransient = "transient filed cannot be serialized";
    public static String A_STATIC_FILED = "static filed belong to class not object, cannot be serialized";
  
  public static void main(String[] args) {
        SerializableClass clazz = new SerializableClass();

        File file = new File("ObjectOutputFile");
        try (ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(file));
             ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(file))) {

            //write object to byte sequences
            objectOutputStream.writeObject(clazz);

            //chang the object filed
            clazz.aLong = 666L;
            // A_STATIC_FILED belong to the class, so you can see it has the value read form
            // the JVM rather the object you serialized before when you deserializes it.
            SerializableClass.A_STATIC_FILED = "Change the static filed!";

            SerializableClass serializableClass = (SerializableClass) objectInputStream.readObject();
            System.out.println(serializableClass);

        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

    }

    @Override
    public String toString() {
        return "SerializableClass{" +
                "anInt=" + anInt +
                ", aLong=" + aLong +
                ", aTransient='" + aTransient + '\'' +
                ", A_STATIC_FILED='" + A_STATIC_FILED + '\'' +
                '}';
    }
}
// output: 
// SerializableClass{anInt=10, aLong=100, aTransient='null', A_STATIC_FILED='Change the static filed!'}
```

## 多次序列化同一个对象

返序列化读取的过程在`readResolve()`方法一节已经涉及到了，我们在看一下保存的部分，这里会有一个有意思的现象：

> Java 序列化机制为了节省磁盘空间，具有特定的存储规则，当写入文件的为同一对象时，并不会再将对象的内容进行存储，而只是再次存储一份引用。
>
> https://www.ibm.com/developerworks/cn/java/j-lo-serial/index.html

这会导致一个问题：当使用同一个`ObjectOutputStream对象`序列化`同一个序列化对象`时，即使在第一次序列化并保存后修改了这个对象的部分属性，当再次序列化时保存的**只是前一个对象的引用**——也就是说将完全相同一个对象保存了两次，**第二次做的修改在序列化的时候并没有保存**。

我们写个简单的DEMO验证一下：

```java
private static void readAndwriteObject2(SerializableClass clazz) {
        File file = new File("ObjectOutputFile" + System.currentTimeMillis());
        try (
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(file));
                ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(file))
        ) {

            // 第一次序列化
            objectOutputStream.writeObject(clazz);
            objectOutputStream.flush();

            clazz.aLong = 9344L;//在这里修改了部分属性

            // 第二次序列化
            objectOutputStream.writeObject(clazz);
            objectOutputStream.flush();

           // 反序列化，读取之前序列化的两个对象
            SerializableClass serializableClass = (SerializableClass) objectInputStream.readObject();
            System.out.println(serializableClass);
            SerializableClass serializableClass1 = (SerializableClass) objectInputStream.readObject();
            System.out.println(serializableClass1);

            System.out.println("serializableClass == serializableClass1: " + (serializableClass == serializableClass1));

        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
// output 
// SerializableClass{anInt=10, aLong=100, aaLong=100, aTransient='null', A_STATIC_FILED='static filed belong to class not object, cannot be serialized'}
// SerializableClass{anInt=10, aLong=100, aaLong=100, aTransient='null', A_STATIC_FILED='static filed belong to class not object, cannot be serialized'}
// serializableClass == serializableClass1: true //可以看到两次获取的是完全相同的对象
```

这是为什么呢，我们可以在源码中看到原因：

```java
// 序列化时，我们会调用ObjectOutputStream的writeObject方法
public final void writeObject(Object obj) throws IOException {
        if (enableOverride) {
            writeObjectOverride(obj);
            return;
        }
        try {
            writeObject0(obj, false);//注意这里，第二个参数unshared是false
        } catch (IOException ex) {
            if (depth == 0) {
                writeFatalException(ex);
            }
            throw ex;
        }
    }

/** obj -> wire handle map */
private final HandleTable handles;

private void writeObject0(Object obj, boolean unshared)throws IOException{
        // ... 
            if ((obj = subs.lookup(obj)) == null) {
                writeNull();
                return;
            } else if (!unshared && (h = handles.lookup(obj)) != -1) {
              // 可以看到这里，如果unshared为false的话，
              // 就会去找这个对象是否已经被序列化过了，是的话就直接写入引用,
              // 而不是再次序列化
                writeHandle(h);
                return;
            } else if (obj instanceof Class) {
                writeClass((Class) obj, unshared);
                return;
            } else if (obj instanceof ObjectStreamClass) {
                writeClassDesc((ObjectStreamClass) obj, unshared);
                return;
            }
        }
}

    /**
     * Writes given object handle to stream.
     */
    private void writeHandle(int handle) throws IOException {
        bout.writeByte(TC_REFERENCE);
        bout.writeInt(baseWireHandle + handle);
    }
```

为了避免这种情况，在保存同一个对象时要注意使用不同的`ObjectOutputStream`对象，或者可以使用`writeUnshared`方法。

```java
// Writes an "unshared" object to the ObjectOutputStream.
public void writeUnshared(Object obj) throws IOException {
    try {
        writeObject0(obj, true);
    } catch (IOException ex) {
        if (depth == 0) {
            writeFatalException(ex);
        }
        throw ex;
    }
}
```

## 优缺点

* 简单，只需要实现接口
* 序列化的字节流可以在文件、网络中传递，可以持久化保存
* 性能差，序列化过程大量使用反射和临时变量

# Externalizable

`Externalizable`继承自`Serializable`。

用户需要通过`writeExternal(ObjectOutput out)`和`readExternal(ObjectInput in)`实现序列化与反序列化的细节，并且需要一个明确实现的**`public no-arg constructor`**

## 实现

```java
class NewClass implements Externalizable {
    public int anInt = 0;
    public String string = "string";
    public static Long aLong = 10L;
    public transient float aFloat = 10F;

    public NewClass(){}

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(anInt);
        out.writeObject(string);
        out.writeLong(aLong);
        out.writeFloat(aFloat);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        anInt = in.readInt();
        string = (String) in.readObject();
        aLong = in.readLong();
        aFloat = in.readFloat();
    }

    @Override
    public String toString() {
        return "NewClass{" +
                "anInt=" + anInt +
                ", string='" + string + '\'' +
                ", aFloat=" + aFloat +
                ", aLong=" + aLong +
                '}';
    }
}
```

## 原理

看源码可以知道，如果检测到当前对象是`Externalizable`时，就会去调用该对象的`writeExternal方法`。

```java
public interface Externalizable extends java.io.Serializable 
    
// writeObject0 方法中：
            if (obj instanceof String) {
                writeString((String) obj, unshared);
            } else if (cl.isArray()) {
                writeArray(obj, desc, unshared);
            } else if (obj instanceof Enum) {
                writeEnum((Enum<?>) obj, desc, unshared);
            } else if (obj instanceof Serializable) {
                writeOrdinaryObject(obj, desc, unshared);//如果是Serializable就执行这个
            } 
// writeOrdinaryObject方法中：
            if (desc.isExternalizable() && !desc.isProxy()) {
                writeExternalData((Externalizable) obj);//如果是Externalizable就执行这个
            } else {
                writeSerialData(obj, desc);
            }
// writeExternalData方法中
// Writes externalizable data of given object by invoking its writeExternal() method.
            if (protocol == PROTOCOL_VERSION_1) {
                obj.writeExternal(this);
            } else {
                bout.setBlockDataMode(true);
                obj.writeExternal(this);
                bout.setBlockDataMode(false);
                bout.writeByte(TC_ENDBLOCKDATA);
            }
```

## 优缺点

* 比`Serializable`麻烦，序列化与反序列化都需要用户自己实现
* 灵活，可以自定义要参与到序列化与反序列化的变量

# Parcelable

`Parcelable`是Android为了解决`Serializable`性能问题而推出的,主要用在Android的`Intent`或`线程间通信`中。

`Parcelable`通过`Parcel`传输到`IBinder`中，从而实现跨进程传输。

> 对于`kotlin`语言来说，Android Studio自动生成的`Parcelable`代码不会处理val变量（因为这些变量不会变化）

## 实现

下面是一个`Parcelable`的实现：

```kotlin
class AParcelable() : Parcelable {

    var i = 10

    // 从Parcel中恢复数据，必须按照写入的顺序读取
    constructor(parcel: Parcel) : this() {
        i = parcel.readInt()
    }

    // 将变量写入到Parcel中，必须与读取的顺序对应
    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeInt(i)
    }

    // 文件描述，一般默认为0
    // 如果这个对象的writeToParcel方法的输出中有特殊的对象则传递对应的描述代码
    // 如：如果包含一个文件描述符FileDescriptor，就要返回CONTENTS_FILE_DESCRIPTOR
    //  https://developer.android.google.cn/reference/android/os/Parcelable.html#CONTENTS_FILE_DESCRIPTOR
    override fun describeContents(): Int {
        return 0
    }

    // 必须有这个变量，用来从Parcel中创建Parcelable类
    // 在JAVA中是public static final Creator<Book> CREATOR = new Creator<Book>() {...}
    companion object CREATOR : Parcelable.Creator<AParcelable> {
        override fun createFromParcel(parcel: Parcel): AParcelable {
            return AParcelable(parcel)
        }

        // Create a new array of the Parcelable class.
        // Returns an array of the Parcelable class, with every entry
        // initialized to null.
        override fun newArray(size: Int): Array<AParcelable?> {
            return arrayOfNulls(size)
        }
    }

}
```

## 原理

原理参考这篇文章[Parcelable源码分析](https://www.kancloud.cn/xcy396/android_tech/1296524)

## 优缺点

* 性能好，`Parcelable`接口比`Serializable`接口效率更高，性能方面高出10多倍 [^Parcelable源码分析]:
* 较复杂，需要自己实现对象的序列化内容

# 总结

一般需要持久化保存数据或在网络间传输时推荐使用`Serializable`或者`Externalizable`。

在Android中`Activity`之间等传递对象，以及跨进程传递对象等时使用`Parcelable`以节省性能。

# 参考资料

[Android之序列化详解](http://darryrzhong.xyz/2019/09/15/Android之序列化详解/)

[Java 序列化的高级认识](https://www.ibm.com/developerworks/cn/java/j-lo-serial/index.html)

https://juejin.im/post/5d7206c5f265da03ab427181#heading-0

https://www.javacodegeeks.com/2019/08/serialization-everything-java-serialization-explained.htmlhttps://juejin.im/post/5ce3cdc8e51d45777b1a3cdf#heading-0

https://blog.csdn.net/qq_16628781/article/details/70049623

[77.单例模式，枚举类型优先于readResolve](https://cl0610.github.io/effective-java-learning/第十章 序列化/77.单例模式，枚举类型优先于readResolve.html)

Pareclable实现原理：[Parcelable最强解析](https://juejin.im/post/5a3b24ab6fb9a04515440bd7)

Parcelable使用：[详细介绍Android中Parcelable的原理和使用方法](https://www.jianshu.com/p/32a2ec8f35ae)

[Parcelable源码分析](https://www.kancloud.cn/xcy396/android_tech/1296524)

[^Parcelable源码分析]: https://www.kancloud.cn/xcy396/android_tech/1296524
