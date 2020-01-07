---
title: Java笔记之YYYY格式化日期
date: 2020-1-7 11:38:54

---

最近看到一个[帖子](https://v2ex.com/t/633650?p=1)，表示有人以`"YYYY-MM-dd"`格式化日期时，在`2019-12-30`时出现`2020-12-30`的BUG。

本文来简单分析一下为什么会出现这个情况。

根据[JDK文档关于日期的定义](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#year)，`y`表示的是我们日常使用的年份，而`Y`表示的是`Week year`。

先了解几个知识点：



# Week year

`Week year`表示的是**这个周所属的年份**。

> A *week year* is in sync with a `WEEK_OF_YEAR` cycle. All weeks between the first and last weeks (inclusive) have the same *week year* value. Therefore, the first and last days of a week year may have different calendar year values.
>
> 来源：https://docs.oracle.com/javase/7/docs/api/java/util/GregorianCalendar.html#week_year

# WEAK_OF_YEAR

指的是这一年所有的周，从第01周开始到该年最后一周。

要注意这个周不一定是自然周，所包含的日期也不一定全部都是当年的日期。

> Values calculated for the [`WEEK_OF_YEAR`](https://docs.oracle.com/javase/7/docs/api/java/util/Calendar.html#WEEK_OF_YEAR) field range from 1 to 53. The first week of a calendar year is the earliest seven day period starting on [`getFirstDayOfWeek()`](https://docs.oracle.com/javase/7/docs/api/java/util/Calendar.html#getFirstDayOfWeek()) that contains at least [`getMinimalDaysInFirstWeek()`](https://docs.oracle.com/javase/7/docs/api/java/util/Calendar.html#getMinimalDaysInFirstWeek()) days from that year.

# 第01周

根据这份[JDK文档](https://docs.oracle.com/javase/7/docs/api/java/util/GregorianCalendar.html#week_year)，当 `getFirstDayOfWeek()` is `MONDAY`（2） and `getMinimalDaysInFirstWeek()` is 4时，JAVA判断周日期的标准与[ISO_8601](https://en.wikipedia.org/wiki/ISO_8601)兼容：

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



按照JAVA文档中的定义，每年最开始的几天和最后的几天的`Week year`不一定是当年的年份值，而是受到每年的*第01周/最后一周*的影响。

JAVA中判断周主要受到`Calendar`对象的`getFirstDayOfWeek()`和`getMinimalDaysInFirstWeek()`这两个本地值的影响。

其中：

* `getFirstDayOfWeek() `指定一周的第一天，比如, 美国一周从`SUNDAY` 开始,法国则是`MONDAY` 。
* `getMinimalDaysInFirstWeek()` 一年第一周所需最小的天数。比如1表示只要包含第一天就算该年的第一周，而7表示只有完整的一周都在该年才算该年的第一周。

**注意**：真正影响我们格式化日期结果的是`SimpleDateFormat`中的`calendar`对象对应的值。

而通过打印这个`simpleDateFormat.calendar`，我们看到：

```java
//JDK1.7
minimalDaysInFirstWeek:1
firstDayOfWeek:1 //SUNDAY
```

所以可以得出结论，JAVA**默认只要次年的1月1日在在这个跨年周，那么本周所有日期的`Week year`都是次年的**（`JDK1.7`）。

# 问题分析

有了以上知识，我们再看看`2019-12-30`以`YYYY`格式化为什么会出现问题：

先看一下这些日期对应的星期：

| 周日 | 周一 | 周二 | 周三  | 周四 | 周五 | 周六 |
| ---- | ---- | ---- | ----- | ---- | ---- | ---- |
| 29   | 30   | 31   | **1** | 2    | 3    | 4    |

首先根据JDK默认的`第01周`的定义，`2020-01-01`所在的周为`2020的第一周`，所以`2019-12-29到2020-01-04`都属于是`2020年的第01周`。

再根据`YYYY`表示的是`Week year`的结论，可以知道，当使用`YYYY`格式化时，`2019-12-29到2020-01-04`都会得到`2020`。

```kotlin
val calendar = Calendar.getInstance()
val simpleDateFormat = SimpleDateFormat("YYYY yyyy MM dd")
calendar.set(2019, 12-1, 29)
println(simpleDateFormat.format(calendar.time))

//output
DATE:2019 12 29
minimalDaysInFirstWeek:1
firstDayOfWeek:1
YYYY yyyy MM dd
2020 2019 12 29
```

而如果我们把第一周最小天数`minimalDaysInFirstWeek`改为`5`天，那么很明显这一周属于`2020`年的天数（从周日到周一，只有1号到4号4天）不够5天，所以这一周被划归为`2019`年的第`53周`，`2019-12-29到2020-01-04`的`week year`都是属于`2019`。

```kotlin
val calendar = Calendar.getInstance()
val simpleDateFormat = SimpleDateFormat("YYYY yyyy MM dd")
calendar.set(2019, 12-1, 29)
simpleDateFormat.calendar.minimalDaysInFirstWeek = 5
println(simpleDateFormat.format(calendar.time))

//output
DATE:2019 12 29
minimalDaysInFirstWeek:5
firstDayOfWeek:1
YYYY yyyy MM dd
2019 2019 12 29
```

再比如下面这个[示例](https://blog.csdn.net/bewilderment/article/details/48391717)中的`2010-12-26`。

![](https://img-blog.csdn.net/20150912103324519?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

按照`JDK1.7`默认算法，一周从周日（`2010-12-26`）开始，并且当年的1月1日（`2011-01-01`）所在周为该年第一周，所以`2010-12-26到2011-01-01`都被划到了`2011`年的第一周。

但如果按照[ISO_8601](https://en.wikipedia.org/wiki/ISO_8601)的标准，一周从周一开始，并且起始年份包含的天数至少要有`4`天：

则很明显`2010-12-26`属于`2010`年的`51周`，而`2010-12-27到2011-01-02`都属于`2010`年的第`52周`（属于2020年的只有2天，不满足第一周的条件）。

| 周一 | 周二 | 周三 | 周四 | 周五 | 周六 | 周日 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 20   | 21   | 22   | 23   | 24   | 25   | 26   |
| 27   | 28   | 29   | 30   | 31   | 1    | 2    |



# 总结

结合以上结论，我们可以看到，在JAVA中（`JDK1.7`）：

* `“YYYY” `表示`Week year`

* 每年最开始的几天和最后的几天的`Week year`不一定是当年的值，而是受到当年的第一周/最后一周的影响。

* JAVA周的判断与`simpleDateFormat.calendar.minimalDaysInFirstWeek`和`simpleDateFormat.calendar.firstDayOfWeek`有关。

  而这两个值都属于本地化值，**在国内可以简单理解为一年1月1日所在的周就是当年的第一周。**

* 我们可以通过修改`minimalDaysInFirstWeek`和`firstDayOfWeek`来更改`YYYY`格式化的值。

# 附录

JDK中日期格式化的参数及含义（来自https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#month）：

| Letter | Date or Time Component                           | Presentation                                                 | Examples                                    |
| ------ | ------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------- |
| `G`    | Era designator                                   | [Text](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#text) | `AD`                                        |
| `y`    | Year                                             | [Year](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#year) | `1996`; `96`                                |
| `Y`    | Week year                                        | [Year](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#year) | `2009`; `09`                                |
| `M`    | Month in year                                    | [Month](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#month) | `July`; `Jul`; `07`                         |
| `w`    | Week in year                                     | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `27`                                        |
| `W`    | Week in month                                    | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `2`                                         |
| `D`    | Day in year                                      | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `189`                                       |
| `d`    | Day in month                                     | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `10`                                        |
| `F`    | Day of week in month                             | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `2`                                         |
| `E`    | Day name in week                                 | [Text](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#text) | `Tuesday`; `Tue`                            |
| `u`    | Day number of week (1 = Monday, ..., 7 = Sunday) | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `1`                                         |
| `a`    | Am/pm marker                                     | [Text](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#text) | `PM`                                        |
| `H`    | Hour in day (0-23)                               | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `0`                                         |
| `k`    | Hour in day (1-24)                               | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `24`                                        |
| `K`    | Hour in am/pm (0-11)                             | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `0`                                         |
| `h`    | Hour in am/pm (1-12)                             | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `12`                                        |
| `m`    | Minute in hour                                   | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `30`                                        |
| `s`    | Second in minute                                 | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `55`                                        |
| `S`    | Millisecond                                      | [Number](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#number) | `978`                                       |
| `z`    | Time zone                                        | [General time zone](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#timezone) | `Pacific Standard Time`; `PST`; `GMT-08:00` |
| `Z`    | Time zone                                        | [RFC 822 time zone](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#rfc822timezone) | `-0800`                                     |
| `X`    | Time zone                                        | [ISO 8601 time zone](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html#iso8601timezone) | `-08`; `-0800`; `-08:00`                    |

# 参考资料

感谢这篇文章，让我推翻了上一次的结论，发现了真正的原因：[JAVA中的SimpleDateFormat yyyy和YYYY的区别](https://blog.csdn.net/bewilderment/article/details/48391717)

[GregorianCalenda jdk1.7](https://docs.oracle.com/javase/7/docs/api/java/util/GregorianCalendar.html#week_year)

在线显示本周是一年第几周的网站：[What's the Current Week Number?](https://www.epochconverter.com/weeknumbers)
