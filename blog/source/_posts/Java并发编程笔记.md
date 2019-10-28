---
title: Java并发编程笔记
date: 2019-10-28 18:26:46
tag: java

---

# Thread.join()

`cThread.join()`方法使当前线程阻塞，直到子线程`cThread`执行完毕后，当前线程才会恢复运行。

实现原理：

1. ` join()`方法调用了`join(0)`

   ```java
   public final void join() throws InterruptedException {
       join(0);
   }
   ```

2. ` join(long millis)`是一个同步方法，最早通过调用`wait()`方法挂起当前线程，直到其他线程调用子线程`cThread`的`notify()`或者`notifyAll()`方法

   ```java
   public final synchronized void join(long millis)
   throws InterruptedException {
       long base = System.currentTimeMillis();
       long now = 0;
   
       if (millis < 0) {
           throw new IllegalArgumentException("timeout value is negative");
       }
   
       if (millis == 0) {
           while (isAlive()) {
               wait(0);
           }
       } else {
           while (isAlive()) {
               long delay = millis - now;
               if (delay <= 0) {
                   break;
               }
               wait(delay);
               now = System.currentTimeMillis() - base;
           }
       }
   }
   ```

3. 子线程`run()`执行完毕后，系统在关闭该子线程前，会调用其`exit()`方法，继而在`ThreadGroup.threadTerminated(Thread t)`中唤醒被阻塞的调用线程。

   ```java
   /**
    * This method is called by the system to give a Thread
    * a chance to clean up before it actually exits.
    */
   private void exit() {
       if (group != null) {
           group.threadTerminated(this);//提心ThreadGroup当前线程已经被终止
           group = null;
       }
       
       //其他代码 ……
   }
   
   // ThreadGroup
   void threadTerminated(Thread t) {
       synchronized (this) {
           remove(t);
   
           if (nthreads == 0) {//线程组线程数为0时
               notifyAll();//唤醒所有等待中的线程
           }
           if (daemon && (nthreads == 0) &&
               (nUnstartedThreads == 0) && (ngroups == 0))
           {
               destroy();
           }
       }
   }
   ```

   

# notify()和notifyAll()的区别

先了解两个概念：

**锁池**：某个`Thread`调用某个对象的同步方法（`synchronized`），但还没获取到该对象的锁时，会进入锁池和其他类似的线程一起竞争该对象的锁。

**等待池**：当某个`Thread`调用某个对象的`wait()`方法释放掉改对象的锁进入阻塞后（waiting on this object's monitor），会进入等待池。等待池中的线程不会竞争该对象的锁。



`notify()`方法会从等待池中唤醒一个指定线程，该线程可以再次回到锁池竞争该对象的锁，**但可能会导致死锁**（如果唯一唤醒的线程阻塞了并依赖其他线程唤醒，但其他线程都在等待池无法竞争锁，导致死锁）。

`notifyAll()`方法则会唤醒所有在等待池中的线程，之后他们都可以回到锁池竞争该对象的锁。

# 参考资料

[Java Thread的join() 之刨根问底](https://juejin.im/post/5b3054c66fb9a00e4d53ef75#heading-2 )

[java中的notify和notifyAll有什么区别？ - 大王叫我来巡山的回答 - 知乎]( https://www.zhihu.com/question/37601861/answer/145545371 )
