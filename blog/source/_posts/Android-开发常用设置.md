---
title: Android开发常用设置
abbrlink: 1c56d6b9
date: 2017-08-27 23:25:05
---

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android-开发常用设置.md</div>"></iframe>

# Android Studio

* 国内较快的仓库：


    maven {url'http://maven.aliyun.com/nexus/content/groups/public/'}

* RecyclerView添加依赖
  注意RecyclerView的版本号要和当前工程中其他android.support包版本保持一致，否则虽然导入了对应的包，但是仍然无法正常使用。


    compile 'com.android.support:recyclerview-v7:26+'

* 设置：
  自动添加依赖：insert imports on paste: None
  自动删除无用依赖：Optimize imports on the fly

# Linux
* 设置ndk环境变量 /etc/profile


    #set ndk env
    NDKROOT=/home/jixiaoyong/AndroidDev/Sdk/ndk-bundle
    export PATH=$NDKROOT:$PATH
