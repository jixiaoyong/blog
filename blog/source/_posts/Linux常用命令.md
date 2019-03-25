---
title: Linux常用命令
date: 2019-01-14 20:16:53
tag: linux
---

# 复制，删除，移动

`cp`拷贝，`rm`删除，`mv`移动。

`-r`表示递归 `-f`强制，无提示

```bash
cp [-r] fromFilePath toFilePath
rm [-r] fromFilePath toFilePath
mv [-r] fromFilePath toFilePath
```

# 切换目录

```bash
cd - 返回上次所在目录
cd ~ 切换到当前用户home路径下
cd . 当前路径
cd .. 上层路径
cd ../linux 切换到同一级的linux目录
```

# 新建文件、文件夹

```bash
mkdir dirName 创建文件夹
touch fileName 创建文件
```

Linux文件和目录名字除了“/”都合法，但是尽量不要用正则表达式之类的符号，因为有可能会在进行正则匹配时造成误删等问题

假设当前目录有文件`f1,f2,f3`和`f[123]`
执行：`rm f[123]`本来是希望删除`f[123]`,但是由于正则匹配，会先删除`f1,f2,f3`这三个文件。

# 查看文件信息

```bash
file fileName 查看文件格式信息
cat fileName 以文本格式查看文件全部内容
less fileName 以分页形式查看文件内容，Q键退出
```

# 常用目录

```bash
/home 当前用户主目录，root用户为/root
/bin、/usr/bin 常用的可执行文件，root用户为/sbin
/media、/mnt 用户硬件挂载点
/etc 系统的配置文件，所有用户可见，root用户可以更改
/boot 系统内核，开机必备文件
/dev 系统的所有设备文件，如硬盘、光驱等
/var和/srv 系统运行时的用户数据
/proc 内存中的状态信息
/lib、/usr/lib、/usr/local/lib 库文件
/temp 临时文件，所有用户可见
/usr 程序相关文件unix system resource
```

# 文件相关

## ls

**`ls`**  展示当前目录下文件信息：

```
ls [-alhd] l 
```

`l`展示目录下的文件列表，`a `展示所有文件（包括隐藏文件），` h` 展示带单位的文件大小， `d`展示当前目录本身信息

## chmod

**`chmod`**  更改权限

```shell
chmod [-R] mode fileName
```

`mode`组成如下：`[范围] [操作] [权限]`

`范围`：`u`用户、`g`群组、`o`其他、`a`以上所有（ugo）

`操作`：`+` 增加、` - ` 减去、`=` 等于

`权限`：`r` 读权限`4`、` w` 写权限`2`、 `x` 执行权限`1` 、无权限  `0`

**权限验证** ： root用户可以访问任何用户文件，不受权限限制；普通用户需要验证权限

*要读取文件夹中的内容，也需要执行权限`x`*

## 文件权限与umask

Linux创建新项目时默认的权限分别是：

```
文件夹 777
文件 666
```

但是，经过umask（此处为0022）遮盖后，变成了755 ，644，这才是真正创建后的结果

可以通过`umask查看`umask的值，一般只去其**后3位**，遮盖的原则是从原先的权限中**减去**umask中的权限：

```
原始权限  ： r w x      7
umask	： -  w x      3
结   果  ： r - -      4
```

## 查看、管理当前用户信息

`users` 和`whoami`输出当前用户名

增、删、改用户：

```
useradd / userdel / usermod username
group群组管理也类似
groupadd / groupmod ...
```

其中`userdel -r username`在删除用户时，也会删除用户对应的主目录`home`

`groups` 查看用户所在群组，其中第一个是主要群组，其余是次要群组。

*主要群组* 在用户创建新的文件时，文件群组权限一项默认为该群组

`who` 、`w`可以查看用户相关信息

`id` 查看某人或者自己相关的`UID、GID`

```
finger [-s] username
查看用户相关信息
-s 仅显示用户账号、全名、登录时间
```

`GID` 系统 <500 ，用户 >500

## 改密码

```
passwd username
```

## 文件打包、压缩和解压缩

`.gz` 压缩后格式，`.tar `打包后格式，`tar.gz`先打包后压缩的格式（常用）

### gzip

gzip压缩会删除源文件,

```
gzip [-cdtv#] filename
```

`#` 压缩等级

`v` 显示压缩前后压缩比

`t` 校验是否是gzip压缩的文件

`c` 压缩文件并输出到屏幕

`d` 解压文件

使用:

```
gizp file 将file压缩成file.gz，会删除file
gzip -c file > file.gz 压缩文件file并输出到file.gz
```

### tar

打包，在压缩文件夹时，一般为了效率都会先打包，在压缩，由此形成的格式一般是类似`*.tar.gz*`的后缀。

```
打包
tar [-jcv] -f outFileName.tar inDirPath
解包
tar [-jxv] -f inFileName.tar -C outputPath
```

`c` 建立打包文档

`x` 解包` -C` 输出目录

`t` 查看打包文件的内容

`j` / `z` 使用`bz2` / `gzip` 压缩、解压

`v` 输出信息

`f` 后面紧跟要操作的文件

# bash shell

bash是用户和内容交互的桥梁 `用户 ↔ bash ↔ Unix内核`



`env` 查看环境变量

`type` 查看类型

`which` 查看指令的位置

`clear` 、 `cls` 清屏

**bash shell 设置**

## 自定义变量

`key=value` 增加一个值为`value`的变量`key`

其中，如果value有空格的话需要用引号包住：

```
双引号 可以用$KEY 引用其他KEY的值
单引号 内容是纯文本
```

`echo $KEY​ `可以输出`KEY`的值

`set` 查看所有变量

```
set | grep HIST 查看shell命令历史
set | grep PSI 提示符前面的内容，username-MBP:dirpath username$ 
```

## 别名配置

`alias` 查看所有别名

`alias newCmd=oldCmd`使用`newCmd`表示`oldCmd`

`unalias newCmd` 删除别名

如:`alias cls=clear`,执行`cls`就等于执行`clear`

## 环境变量

`export KEY=VALUE` 将值为`VALUE`的`KEY`添加到环境变量（本次shell有效）

此外还可以写到一些文件中，在开机、登录、注销登录时调用执行——自动执行脚本**`shell startup scripts`**

## shell startup scripts

开机时执行：

* `/etc/profile`
* `/ect/profile.d/*.sh`

* `~/.bash_profile , ~/.bash_login , ~/.profile`这三个只要其中一个成功执行了，后面的就不会执行，`~/.bash_profile`会执行`~/.bashrc`

* `/etc/.bashrc`

未登录时会执行：

* `~/.bashrc`
* `/etc/bashrc`
* `/etc/profile.d/*.sh`

注销时执行`~/bash_logout`

**在修改了以上文件后，可以使用`source path_to_file`或者重新登录使其立即生效**

# 标准输入输出等

| 代码编号 | 名称     | 代码   | 作用对象 |
| -------- | -------- | ------ | -------- |
| 0        | 标准输入 | stdin  | 键盘等   |
| 1        | 标准输出 | stdout | 屏幕等   |
| 2        | 标准错误 | stderr | 屏幕等   |

定向

* `<`和`<<` 输入和 追加输入
* `>` 和`>>` 输出 和追加输出

使用：`ls -al | >> result.txt`将`ls`的内容追加输出到`result.txt`文件中。

`|`叫做管道，可以将前者的**标准输出**当做后者的输入。

 `cmd0 && cmd1` 前者执行成功才会执行后者；

`cmd0 || cmd1` 前者执行失败才会执行后者。

# grep

查询内容

```shell
grep [-cinv] 'key' filename
```

`c` 计算次数

`i` 忽略大小写

`n` 行号

`v` 显示没有该字符的行号

`'key' `可以是正则表达式

`--color=auto` 对查找到的文本显示颜色

# sort

排序，默认以第一列排序

```shell
sort [-fbknrtu] filename
```

`f` 忽略大小写

`b`忽略最前面的空格（要是排序不生效时可以试一下，推荐）

`k` 以第几列为标准排序，默认第一列

`n` 以数组排序

`r` 逆序

`t` 待排序的文件的分隔符，默认是tab

`M` 以英文月份排序

# wc

统计字符数

```shell
wc [lwm] filename
```

`l` 行

`w` 词

`m` 字符





<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Linux常用命令.md</div>"></iframe>