---
title: '[update]Python爬取gityuan所有文章列表'
date: 2018-01-28 22:03:50
tags: python
---

# 简介

更新内容：

* 爬取gityuan.com网站所有文章列表并输出json
* 汇总信息输出config.json为后面的客户端做准备

更新文件：

* spider_main.py
* html_output.py
* **gityuan_urls.py**
* **html_downloader.py**

# 代码

* **spider_main.py**

作为入口类，主要增加了初始化所有URL，以及便利这些URL的功能。

```python
sp = SpiderMain()

gityuan = gityuan_urls.GitYuanUrls()

gityuan_urls = gityuan.get_urls(17)


file_name = 'gityuan_page_'

sp.craw(gityuan_urls,file_name)
```

其`craw()`方法修改如下：

```python
from time import sleep

def craw(self, root_urls, file_name):
        
        #将所有有效链接全部加入
        self.urls.add_new_urls(root_urls)

        i = 0

        #循环遍历这些链接
        while self.urls.has_next():

            i = i + 1

            new_url = self.urls.get_new_url()

            html_cont = self.downloader.download(new_url)

            new_url, new_data = self.parser.parse(new_url, html_cont)

            self.output.collect_data(new_data)

            new_file_name = file_name + ('%d.json'%i)

            self.output.output_html(new_file_name)
            
            #等待3s，防止太频繁访问被识别
            sleep(3)

        #结束遍历，输出汇总信息
        self.output.end(file_name,root_urls[0],i)
```

* **html_output.py**

主要改动如下：

1. `output_html(self,file_name)`方法增加一个`file_name`的参数，并在内部调用`self.mkdir()`方法生成output目录，方便同时输出多个文档
2. `mkdir()`方法，创建文件
3. `end(self,file_name_start, url, num)`方法，输出汇总文档，代码如下

```python

    def end(self,file_name_start, url, num):

        self.mkdir()

        file_name = self.output_dir + 'config.json'

        file_out = open(file_name,'w')
        
        current_time = time.time()

        config_str = ('{"url":"%s","total":"%d","update_time":"%d","file_name":"%s"}' % (url,num,current_time,file_name_start))

        file_out.write(config_str)
```

* **gityuan_urls.py**

主要代码如下，通过循环遍历获取所有文章列表信息

```python
class GitYuanUrls(object):
	"""docstring for GitYuanUrls"""
	def get_urls(self, num):

		urls = []

		urls.append('http://www.gityuan.com')

		for x in xrange(1,num):
			url_ = ('http://www.gityuan.com/page%d/'%x)
			urls.append(url_)

		return urls
```

* **html_downloader.py**

就在本文编辑的过程中，爬虫被识别，并且限制访问文件数量，所以对下载功能做了简单的伪装、增加超时处理。

```python
def download2(self, url):
        #升级版

        if url is None:
            return None

        #伪装为浏览器
        req_header = {'User-Agent':'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.87 Safari/537.36'}

        try:
            request = urllib2.Request(url,None,req_header)

            response = urllib2.urlopen(request,None,300)

            return response.read()
        except socket.timeout as e:
            #超时处理
            print(type(e))
        
        return None

```



# 结语

当前爬虫主体功能以及实现，可以爬取gityuan.com所有有效文章列表，可以满足客户端需求。但仍然存在以下问题：

1. 没有伪装，爬虫**很容易**被识别并被拒绝服务（~~就在刚刚写下这句话的时候，就发生了被限制访问，真*乌鸦嘴~~）。
2. 由于原网站特性，其置顶文章每页都有，会导致部分数据重复。
3. 未爬取具体文章内容。



> **说明**
>
> 本文只为学术研究，其中涉及到的第三方网站及其所有资源均属原主所有。向gityuan大神致敬，欢迎访问其[blog](http://gityuan.com/)。



# 源码

[github链接](https://github.com/jixiaoyong/AndroidNote/tree/master/code/2018-1-26/gityuan_spider)

`tag`为`gityuan_spider1.5`