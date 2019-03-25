---
title: Python入门知识
tags: python
abbrlink: a8f97c56
date: 2018-01-20 23:49:17
---

基于Python3.x

Python文件默认格式`.py`

首行默认以下命令：

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
```



# 数据类型

* **数字**

  整数int  1，2，3

  长整数long  1112L

  浮点数float （小数）1.23，3.14

  复数complex  3.14j

* **字符串**

   'abc'，"abc"，'''abc‘’‘

  'x'和"x" 区别不大

  '''abc‘’‘ 文本可以跨行

  字符串前面加r或者R表示字符串内部不需要转义，否则要用`\` 转义

  支持`a[0]`取值

* **布尔值**

   `True` 和`False`

  布尔值可以用`and`、`or`和`not`运算

* **空值**

   `None`

* **变量** 

  命名规则：开头`aA_`，其后可以包含`aA_1`

* **常量**

   不能变的变量

# 集合

* **列表list**

   [1,2,3,3]

  插入list.insert(1,'vaule')

  删除list.pop() / list.pop(1)

* **元组tuple **

  (1,2,3,3)

  与列表类似，但是一旦初始化就不能再修改

  ------

* **字典dict**

   {'a':1,'b':'vaule'}

  键值对，读取快，相当于java的map

* **set**

   set([1,2,3])

  键的集合，不能有重复的，相当于java的set

# 逻辑语句

* if ... : ... elif ... : ... else : ...
* for x in xs : ...
* while x : ...

# 自定义函数

```python
def fun(n)
	return n

```

* **return**

可以没有return，默认返回None

可以return 多个值，实际上返回的是一个tuple

* **pass**

 不想执行任何语句，但是为了符合语法规范，可以用pass当做占位符

```python
def fun()
	pass

```

* **抛出异常** 

```python
raise TypeError('an error')
```

其中TypeError需要继承自`error`或者`Exception`

* **参数**

位置参数

```python
def fun(arg)
	pass
```

默认参数

```python
def fun(arg0,arg1 = 1)
	pass
```

**注意** 默认参数必须是参数中后面的几位；默认值必须不可变，如int，string等

可变参数

```python
def fun(arg,*args)
	pass
```

`*args` 表示参数个数可变，可以输入list/tuple等，或者依次输入多个参数，用逗号分隔

关键词参数

```python
def fun(arg,**keywords)
	if 'city' in kw:
		pass
```

`**keywords` 表示接受关键词作为参数传入，可以传入dict，或者依次输入多个关键词参数

命名关键词参数

```python
def fun0(args,*，name,age)
	pass
def fun1(arg.*args,name,age)#如果命名关键词前面有可变参数，则不用*分隔
	pass
```

限制输入的关键字，限制只有name和age作为关键词参数

# 使用其他文件的函数

```python
#使用时 sys.fun()
import sys
#使用时直接fun()
from xxFile import fun
form sys import *
form sys import fun
```

# 类

* **定义类**

```python
class AClass(object):
	'''doc for AClass
	you can use this by
	AClass.__doc__'''
  def __init__(self):
  	#默认的初始化方法
  	pass
  def aFun(self):
  	pass
#创建类对象
a = AClass()
#调用方法
a.aFun()
```

所有的类方法必须至少有一个参数，推荐命名为self，系统会自动传入类对象，无需手动传入。

* **继承**

```python
class Father(object):
  def __init__(self):
    print("father")
  def say(self):
    print("i am f")
    
class Child(Father):
  def __init__(self):
    #子类方法不会自己调用父类方法，需要手动调用
    super(Child,self).__init__()
    #调用父类方法2：
    #Father.__init__(self)
    print("child")
  def say(self):
    print('i am c')
  def go(self,where):
    print('go to %s'%where)

c = Child() #father child
c.say() #i am c
c.go('home') #go ro home
```

子类继承父类，则需要在子类定义时传入父类

子类如果有与父类同名方法，则优先调用子类方法，除非子类特别调用父类的方法



<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Python入门知识.md</div>"></iframe>