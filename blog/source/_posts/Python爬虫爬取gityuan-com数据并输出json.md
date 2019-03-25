---
title: Python爬虫爬取gityuan.com数据并输出json
tags: python
abbrlink: 9c0ccf0d
date: 2018-01-27 22:25:34
---

# 简介

> 本文基于Python2.7

这篇文章基于我在[慕课网](www.imooc.com)上面学习Python简单爬虫写的内容，教程内容是爬取1000条百度百科的数据，但是教程中爬虫截止2018-01-27已经失效，刚好看到大神gityuan.com的内容，于是用Python实现爬取其网页内容并生成json数据。

本文即上述过程整理。

本文涉及源代码已上传github（[点这里查看](https://github.com/jixiaoyong/AndroidNote/tree/master/code/2018-1-26/gityuan_spider)）。

# 框架

爬虫主要活动是：

1. 爬取目标网页内容
2. 对获取到的内容进行分析，获取有用数据
3. 将处理好的数据按格式输出

此外还需要有一个专门管理爬虫活动的主类，故而文件结构如下：

1. spider_main.py             入口类
2. url_manager.py             管理要下载的链接
3. html_downloader.py    下载网页内容
4. html_paeser.py              对获取到的数据进行解析、加工
5. html_out.py                     输出格式化的数据

目前只实现了爬取gityuan.com第一页内容并输出json，所以暂时不需要实现url_manager.py

# 关键代码

**spider_main.py**

```python
#导入用到的各个类
import html_downloader
...

#定义入口类
class SpiderMain(object):
  def __init__(self):
    
    #初始化各个变量downloader、parser、output...
    self.downloader = html_downloader.HtmlDownloader()
    
    #略
   def craw(self,root_url):
    
    html_cont = self.downloader.download(root_url)
    
    new_data = self.parser.parse(html_cont)
    
    self.output.collect_data(new_data)
    
    self.output.output_json()
    
root_url = 'http://www.gityuan.com/'
sp = SpiderMain()
sp.craw(root_url)
```

* 在`__init__()` 方法初始化各个变量；
* 在`craw()`中分别实现下载、解析网页内容、输出加工数据

**html_download.py**

```python
import urllib2

class HtmlDownLoader(object):
  
  def download(self,url):
    if url is None:
      return None
    
    respone = urllib2.urlopen(url,timeout=300)
    
    if respone.getcode() != 200:
      return None
    
    return respone.read()
    
```

下载并返回网页内容，比较简单

**html_parser.py**

```python
import urlparse
from bs4 import BeautifulSoup #第三方包，需要单独下载
import re

class HtmlParser(object):
  
  def parse(self,html_cont):
    if html_cont is None:
      return
    
    #用BeautifulSoup解析文档内容
    soup = BeautifulSoup(html_cont,'html.parser')
    
    res_data = [] #数组
    
    #获取所有的文章节点nodes
    post_div_nodes = soup.find_all('div',class_='post-preview')
    
    #遍历nodes，读取每一项内容并保存
    for post_div_node in post_div_nodes:
      post_div_soup = BeautifulSoup(str(post_div_node))
      
      post_info = {} #字典dict
      
      #判断URL是否是完整
      url_ = post_div_soup.a['href']
      if 'http://' not in url_:
        url_ = "http://gityuan.com" + url_
       
      #保存数据
      post_info['url'] = url_
      post_info['title'] = post_div_soup.find('h2').get_text()
      post_info['summary'] = post_div_soup.find('div',class_='post-content-preview').get_text()
       
      res_data.append(post_info)
      
     return res_data   
    
```

这是爬虫功能的重点之一：对网页数据进行解析，由此数据才变为可用数据

主要是通过第三方插件`BeautifulSoup`解析数据，并保存到数组`res_data`中，具体见代码中实现

**html_output.py**

```python
import sys
#下面两行代码解决编码问题，强制使用utf-8，而非默认的unicode编码
reload(sys)
sys.setdefaultencoding('utf-8')

class JsonOutput(object):
  def __init__(self):
    self.datas = []
  
  def collect_data(self,new_data):
    if new_data is not None:
      self.datas.append(new_data)
    
  def output_json(self):
    #打开文件，并以json格式输出
    fout = open('output.json','w')
    
    fout.write('{')
    fout.write(r'"data":[')
    
    for data in self.datas:
      for post_info in data:
        
        fout.write('{')
        fout.write('"url":"%",' % post_info['url'])
        fout.write('"title":"%",' % post_info['title'])
        fout.write('"summary":"%",' % post_info['summary'])
        fout.write('},')
    #为了符合json规范，最后一个输入空数据，无末尾逗号
    fout.write(r'{}')
    
    fout.write(']}')
```

本类也很重要，主要是数据存取，以及将解析好的数据格式化输出

# 说明

本文中代码经二次处理，不一定与源代码一致，但思路如此，以供参考。



<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Python爬虫爬取gityuan-com数据并输出json.md</div>"></iframe>