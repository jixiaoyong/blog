---
title: Linux下配置Git，使用AndroidStudio同步工程到Github
abbrlink: 25db0a11
date: 2016-04-29 06:06:06
---

这篇文章介绍了如何在 linux 环境下安装和配置 git 与 github ，并且使用 Android Studio 将本地的项目同步到 github 上面。

# 安装 git

```
sudo apt-gat install git
```

# 配置 git 和 github

- 创建 Github 账号

- 生成 ssh key

  ```
  ssh-keygen -t rsa -C "your_email@youremail.com
  ```

- 在 github 上面添加 ssh key

进入 Account Settings –> SSH Keys –> Add SSH Key 添加 SSH Keys ：
名字起一个容易识别的名字，key 是生成的 `/home/username/.ssh/id_rsa.pub.` 中的内容，直接粘贴到指定位置就行

- 测试 ssh key 是否成功

  ```
  ssh -T git@github.com
  ```

提示如`You’ve successfully authenticated, but GitHub does not provide shell access`则说明成功连接 github

- 配置 Github

  ```
  git config --global user.name "your name" //配置用户名
  git config --global user.email "your email" //配置email
  ```

# 用 Android Studio 同步工程到 Github

- 启动android studio

  进入`android studio/bin`，终端输入`./studio.sh`

- 选择 `VCS ---> Import into Version Control --> Share Project on Github`

第一次进入会要求输入 github 的账号和密码 按照要求输入即可
此后还会要求你输入一个本地密码，当下次同步的时候需要输入
之后就进入到选择同步的仓库，新建一个仓库，开始同步就可以了

**到这里就顺利的在 Android Studio 上面将工程同步到 Github 上面了**

------

**以下为原文提到的其他方法，摘录如下，以备后用：**

# 利用Git从本地上传到GitHub

第一步： 进入要所要上传文件的目录

输入命令 `git init`

第二步： 创建一个本地仓库 origin

使用命令

```
git remote add origin git@github.com:yourName/yourRepo.git
```

`youname`是你的GitHub的用户名，`yourRepo`是你要上传到GitHub的仓库

第三步： 比如你要添加一个文件xxx到本地仓库，使用命令 `git add xxx`，可以使用 `git add .` 自动判断添加哪些文件

然后把这个添加提交到本地的仓库，使用命令 `git commit -m`说明这次的提交

最后把本地仓库origin提交到远程的GitHub仓库，使用命令 `git push origin master`

# 从GitHub克隆项目到本地

第一步： 到GitHub的某个仓库，然后复制右边的有个`HTTPS clone url`

第二步： 回到要存放的目录下，使用命令 `git clone https://github.com/chenguolin/scrapy.git`，这里的url只是一个例子

第三步： 如果本地的版本不是最新的，可以使用命令 `git fetch origin`，origin是本地仓库

第四步： 把更新的内容合并到本地分支，可以使用命令 `git merge origin/master`

如果你不想手动去合并，那么你可以使用：
`git pull <本地仓库> master` // 这个命令可以拉去最新版本并自动合并

# GitHub的分支管理

- 创建

1 创建一个本地分支： `git branch <新分支名字>`

2 将本地分支同步到GitHub上面： `git push <本地仓库名> <新分支名>`

3 切换到新建立的分支： `git checkout <新分支名>`

4 为你的分支加入一个新的远程端： `git remote add <远程端名字> <地址>`

5 查看当前仓库有几个分支: `git branch`

- 删除

1 从本地删除一个分支： `git branch -d <分支名称>`

2 同步到GitHub上面删除这个分支： `git push <本地仓库名> :<GitHub端分支>`

# 说明

这篇文章是我今天在 linux 下安装 git ，上传工程到 github 上面时的步骤的总结，大部分内容都参考/摘录自下面这篇文章，感谢原作者的分享，原文信息及链接如下：

> Linux下Git和GitHub使用方法总结
> [日期：2014-03-07] 来源：Linux社区 作者：chenguolin
> <http://www.linuxidc.com/Linux/2014-03/97821.htm>