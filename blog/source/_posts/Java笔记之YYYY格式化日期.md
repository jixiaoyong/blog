---
title: Java笔记之YYYY格式化日期
date: 2020-1-7 11:38:54
---
最近看到一个[帖子](https://v2ex.com/t/633650?p=1)，表示有人以`"YYYY-MM-dd"`格式化日期时，在`2019-12-30`时出现`2020-12-30`的BUG。

本文来简单分析一下为什么会出现这个情况

根据[JDK文档关于日期的定义](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#year) ，`y`表示的是我们日常使用的年份，而`Y`表示的是`Week year`。

先了解2个知识点：

# Week year

`Week year`表示的是**这个周所属的年份**。每年最开始的几天和最后的几天的`Week year`不一定是当年的值，而是受到每年的第一周的影响。

> A *week year* is in sync with a `WEEK_OF_YEAR` cycle. All weeks between the first and last weeks (inclusive) have the same *week year* value. Therefore, the first and last days of a week year may have different calendar year values.
>
> 来源：https://docs.oracle.com/javase/7/docs/api/java/util/GregorianCalendar.html#week_year

# 第01周

又有根据这份[JDK文档](https://docs.oracle.com/javase/7/docs/api/java/util/GregorianCalendar.html#week_year)，JAVA判断周日期的标准与[ISO_8601](https://en.wikipedia.org/wiki/ISO_8601)兼容：

> 第01周有几个相互等效且兼容的描述：
>
> 一年中第一个星期四的星期（正式的ISO定义），
>
> 1月4日这一周，
>
> 起始年份中大部分（四天或以上）的第一周，以及
>
> 从12月29日至1月4日的星期一开始的一周。
>
> 来源：https://en.wikipedia.org/wiki/ISO_8601

# 问题分析

有了以上知识，我们再看看`2019-12-30`以`YYYY`格式化为什么会出现问题：

先看一下这些日期对应的星期：

| 周一 | 周二 | 周三 | 周四 | 周五 | 周六 | 周日 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 30   | 31   | 1    | 2    | 3    | 4    | 5    |

首先根据`第01周`的定义，`2020-01-04`所在的周为`2020的第一周`，所以`2019-12-30到2020-01-05`都属于是`2020年的第01周`。

再根据`YYYY`表示的是`Week year`的结论，可以知道，当使用`YYYY`格式化时，`2019-12-30到2020-01-05`都会得到`2020`。


