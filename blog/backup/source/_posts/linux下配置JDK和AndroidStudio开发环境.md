---
title: linux下配置JDK和AndroidStudio开发环境
abbrlink: ca532ade
date: 2016-04-28 06:06:06
---

# 下载 JDK 并解压

- 到官网下载 jdk
- 下载到的 JDK 文件解压

# 设置环境变量

管理员权限进入 etc/environment 写入以下代码

```
JAVA_HOME="JDK主目录的绝对路径"
```

# 配置 alternatives

打开终端执行以下命令：

```
sudo update-alternatives --install  /usr/bin/java java  JDK主目录的绝对路径/bin/java 300

sudo update-alternatives --install  /usr/bin/javac javac  JDK主目录的绝对路径/bin/javac 300
```

到这里 JDK 的环境就配置好了

# 运行 Android Studio

进入 android studio/bin 目录下，打开终端，

输入 `./studio.sh`

到这里，就可以正常运行 android studio 了