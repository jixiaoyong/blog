---
title: python自动创建发布hexo文章并同步github
date: 2018-02-01 00:09:12
tags: python
---

# 简介

> 环境 linux(deepin)
>
> python 2.7

这是一个Python脚本，用于实现hexo文章创建、生成网页并预览、发布到对应xxx.github.io博客的全过程。

# 使用方法

## 使用时需要根据自己的项目更新main.py的一下变量：

* hexo_url = 'your_path/hexo/blog'

  【必需】本地hexo博客路径

* hexo_public_dir = 'your_path/hexo/blog/public'

  【必需】本地hexo博客输出路径

* hexo_post_dir = 'your_path/hexo/blog/source/_posts'

  【可选】本地hexo博客文章源文件路径

* git_dir = 'your_path/xxx.github.io'

  【必需】博客要同步的git工程路径

* git_backup_dir = 'your_path/xxx.github.io/blog/backup/sources/_posts'

  【可选】本路径用于备份post源文件到github

* hexo.py 中的`post()`方法中`webbrowser.open('http://jixiaoyong.github.io/blog/')`中的博客地址，发布完后默认打开该网页。（后期也可以改为`post()`参数传入，这样只需要更改`main.py`就行）

## 运行`main.py`文件

* 在Linux命令行输入如下命令，并回车，根据提示操作即可。

```
python main.py
```

​    Windows下可以运行`start.cmd`脚本（待实现）

```
//start.cmd脚本内容
python main.py
cmd
```

* 操作过程提示及说明如下：（渣英语请忽略...）

  * input yout file name 输入要发布的文章名称xxx（当前版本暂不支持中文）

    输入回车会自动创建xxx.md文件并打开（需要系统支持该格式）

  * are you finish your post 输入y或n，选择是否用hexo编译文章

    y:编译文章  n:不编译文章，退出命令行

  * post or not  输入y或n，选择是否发布文章到网站,可以在打开的页面预览后做决定

    y:发布文章  n:不发布文章，退出命令行

  * update post  《xxx》 提示开始发布文章，自动打开网页，并保存源文件



# 源代码

源代码已经上传[github](https://github.com/jixiaoyong/AndroidNote/tree/master/code/2018-1-31/python%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B2%E6%96%87%E7%AB%A0)



# 后期计划

* 增加文件名中文支持
* 增加图片自动上传、替换为github链接