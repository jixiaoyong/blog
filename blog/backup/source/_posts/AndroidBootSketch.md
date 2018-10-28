---
title: Android系统架构简介
tags: android
abbrlink: e3bb9a54
date: 2018-02-23 22:35:14
---



> 说明：本文基于[Android系统开篇 - Gityuan博客 | 袁辉辉博客 ](http://gityuan.com/android/) 的学习笔记整理

Android系统大体分为4个模块，从底层开始依次是1.linux内核、2.系统库+Android运行时、3.框架层、4.应用层。

![](https://raw.githubusercontent.com/jixiaoyong/jixiaoyong.github.io/master/images/blog/2018-02/AndroidSystemArchitecture.png)

下图描述了Android系统从开机到Apk运行的整个流程。

![系统启动框架图，来自gityuan.com](https://raw.githubusercontent.com/jixiaoyong/jixiaoyong.github.io/master/images/blog/2018-02/androidBoot.jpg)

流程如下：`Loader` -> `Kernel` -> `Native`-> `Framework` -> `App`

**Loader层**

1. Boot ROM ：开机时，引导芯片从ROM读取读取初始化代码，加载引导程序到RAM中。
2. Boot Loader：是启动Android系统之前的引导程序，检查RAM、初始化硬件参数等。

**Kernel层**（即Android内核层，进入Android系统）

1. swapper进程（pid=0）：Boot Loader启动swapper（idle）进程，是由内核创建的第一个进程，用来初始化进程管理、内存管理、驱动等等。
2. kthreadd进程（pid=2）：是Linux系统的内核进程，**是所有内核进程的鼻祖**。

------

**Syscall**，在Native和Kernel之间的系统调用层。

------

**Native层**

1. init进程（pid=1）：由swapper进程创建，**是所有用户进程鼻祖**
2. init进程孵化出用户守护进程、启动ServiceManager管理系统服务，启动开机动画Bootnaim。

------

**JNI**，Java层和Native（C/C++）层之间。

------

**Framework层**

1. Zygote进程：由init进程fork生成，是**Android系统第一个java进程，是所有java进程的父进程**
2. SystemServer进程：由Zygote进程fork而来，**是Zygote孵化的第一个进程**，负责启动和管理整个**java framework**，如ActivityManager、PowerManager...
3. MediaServer进程：由init进程fork而来，负责启动和管理整个**C++ framework**

**APP层**

1. Launcher：**Zygote进程孵化的第一个App进程**，桌面App。
2. 其他由Zygote进程孵化的系统进程（Browser、Phone...）和非系统app进程。



扼要内容如图：

![系统启动示意图](https://raw.githubusercontent.com/jixiaoyong/jixiaoyong.github.io/master/images/blog/2018-02/AndroidBootImg.png)



Android常用的通信方式

1. Binder
2. Socket
3. Handler

**Binder/Socket用于进程间（都具有独立的地址空间）通信，而Handler消息机制用于同进程的线程间（共享内存空间）通信**

在Android系统中：

- Zygote进程  -->  Socket机制
- SystemServer、MediaServer、App之间  -->  Binder IPC
- 同一进程不同线程间 -->  Handler