---
title: 利用travis通过Hexo在Github上自动部署Markdown文档
tag: ci
date: 2018-12-20 22:35:34
---

# 前言

> 本文介绍了一个只需要更新Markdown文档到Github，即可实时更新博客内容的方法。
>
> 本文参考[这篇文章](https://juejin.im/post/5a1fa30c6fb9a045263b5d2a) 实现，并根据我的需求更改了部分内容，以实现**部署多个hexo工程到同一Github项目不同目录下**。

Github为我们提供了[Github Pages](https://pages.github.com/) 方便我们建立简单的网页来介绍项目，很多时候我们用他来搭建静态博客。

通过[Hexo](https://hexo.io/)可以将我们写的`Markdown文档`格式化为`静态网页`，再将其部署到Github上面对应的`user_name.github.io`上面，就可以拥有一个在线的静态博客。

但是受Hexo的限制，每次更新博客内容都需要在更新完Markdown文档后，都需要再次重新创建对应的静态网页、将更新提交到Github。这样的步骤繁琐且没有意义，而且更换电脑后这些环境都需要重新设置一次。

通过[travis](https://www.travis-ci.org)提供的免费CI技术，可以让云服务器代替我们实现Hexo创建以及同步Github等步骤，每次更新博客时**只需要将写好的Markdown文档推送到Github项目对应目录中，等待一会儿就可以看到更新后的博客了**。

具体搭建过程可以参考[这篇文章](https://juejin.im/post/5a1fa30c6fb9a045263b5d2a) 本文只讲述实现**部署多个hexo工程到同一Github项目不同目录下**需要注意的地方：。

> **懒——是第一生产力**



# 具体差异

## hexo分支的结构

因为有多个hexo项目，所以在github项目的hexo分支下，对不同的hexo项目分别新建文件夹存放。

```shell
-- your_name.github.io //github项目，切换到hexo分支
  --hexo_project1 //本地hexo项目1的所有文件
  --hexo_project2 //本地hexo项目2的所有文件
```

## .travis.yml

重点修改`script:`和`after_script:`两部分：

```yaml
script:
  # 1. 创建对应的静态博客内容
  - cd blog # 第一个本地hexo项目
  - hexo clean
  - hexo generate
  - cd ..
  - cd imissyou # 第二个本地hexo项目
  - hexo clean
  - hexo generate
  - cd ..

after_script:
  - git config user.name "jixiaoyong"
  - git config user.email "jixiaoyong1995@gmail.com"
  - cd ..
  - mkdir publish
  - cd publish
  # 2. 在这里再拉取master分支的文件，并删除旧的博客内容
  - git clone https://${GH_TOKEN}@github.com/jixiaoyong/jixiaoyong.github.io.git
  - rm -rf ./jixiaoyong.github.io/blog/*
  - rm -rf ./jixiaoyong.github.io/imissyou/*
   # 3. 将第1步生成的静态博客内容添加到master分支，并同步到github上面
  - cd ..
  - cp -rf jixiaoyong.github.io/blog/public/* publish/jixiaoyong.github.io/blog/
  - cp -rf jixiaoyong.github.io/imissyou/public/* publish/jixiaoyong.github.io/imissyou/
  - cd publish/jixiaoyong.github.io/
  - git add .
  - git commit -m "auto update by www.travis-ci.org"
  - git push
```

文档链接：[.travis.yml](https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/.travis.yml)

# 更新博客内容

当以上内容都配置完成后，只要新建一个符合hexo要求的文档，并提交到Github对应项目的hexo分支中`source`目录，Travis便会自动帮我们创建并更新静态网页。

# 参考文档

[Hexo遇上Travis-CI：可能是最通俗易懂的自动发布博客图文教程](https://juejin.im/post/5a1fa30c6fb9a045263b5d2a) （完全在该文档指导下完成，部分步骤有差异，感谢作者[MichaelX](https://juejin.im/user/56efe6461ea493005565dafd) ）
