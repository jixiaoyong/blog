---
title: Android笔记之贝塞尔曲线的应用
tag: android
date: 2020-4-10 09:49:03
---



[贝塞尔曲线](https://baike.baidu.com/item/贝塞尔曲线)是用节点和控制点绘制的高精度曲线，Android中常用的有二阶、三阶贝塞尔曲线。本文介绍使用贝塞尔曲线绘制折线图，并实现动画效果。

![](https://jixiaoyong.github.io/images/20200413215502.jpg)



本文代码链接：https://github.com/jixiaoyong/library/blob/master/library/src/main/java/cf/android666/applibrary/view/BezierViewAnim.kt

# 贝塞尔曲线介绍

下图是二阶贝塞尔曲线绘制方法介绍，只要各个点满足条件：AD/AB = BE/BC = DF/DE，那么当沿着当前线段移动D、E点时，F点的运动轨迹就是一个贝塞尔曲线：

![图片来自: https://www.cnblogs.com/wjtaigwh/p/6647114.html](https://jixiaoyong.github.io/images/20200413215622.png)

[^图片来源]: https://www.cnblogs.com/wjtaigwh/p/6647114.html

动图示意如下：

![](https://jixiaoyong.github.io/images/20200413222352.webp)

[^图片来源]: https://www.jianshu.com/p/0c9b4b681724

可以在下面的两个网站在线体验贝塞尔曲线：

[https://aaaaaaaty.github.io/bezierMaker.js/playground/playground.html](https://aaaaaaaty.github.io/bezierMaker.js/playground/playground.html)

[https://bezier.method.ac/](https://bezier.method.ac/)

# 计算控制点坐标

在绘制折线图时，我们获取的数据可以当做贝塞尔曲线的端点，Android为我们提供了绘制二阶和三阶贝塞尔曲线的方法：

```java
Path.quadTo(float x1, float y1, float x2, float y2)//二阶贝塞尔曲线：分别是控制点的x、y坐标和结束的的x、y坐标
Path.cubicTo(float x1, float y1, float x2, float y2, float x3, float y3)//三阶贝塞尔曲线：分别是控制点1、2的x、y坐标和结束的的x、y坐标
```

以`Path.cubicTo()`方法为例，在绘制三阶贝塞尔曲线时，起点和终点已知，剩下工作就是计算两个控制点的坐标。

## 方法1

按照贝塞尔曲线的定义，计算各个点对应控制点的坐标，具体的计算原理我们可以参考[这篇文章](https://wenku.baidu.com/view/c790f8d46bec0975f565e211.html)

假设起点、终点分别为`startPoint`，`endPoint`，起点前一个点为`beforePointF`，终点后一个点为`afterPoint`，那么终止点1、2（`controlPoint1`、`controlPoint2`）的坐标满足（其中a,b为任意正数，比如1/6）：

```kotlin
        val controlPoint1X = startPoint.x + (endPoint.x - beforePointF.x) * a
        val controlPoint1Y = startPoint.y + (endPoint.y - beforePointF.y) * a

        val controlPoint2X = endPoint.x - (afterPoint.x - startPoint.x) * b
        val controlPoint2Y = endPoint.y - (afterPoint.y - startPoint.y) * b
```

> 这里要处理特殊情况：第一个点P~0~的前一个仍然为P~0~，最后一个点P~n~的后一个点仍为P~n~

但这种情况绘制出来的贝塞尔曲线如下：

![](https://jixiaoyong.github.io/images/20200413220840.jpg)

可以看到除了P~0~和P~n~外，其他点的曲线坐标和对应的点坐标不一致。

## 方法2

为了解决方法1存在的问题，我们人为的在两个点之间加入两个控制点，这样在`startPoint`，`endPoint`之间的贝塞尔曲线首尾点的坐标必定落在起点和终点上（思路来自[这里](https://blog.csdn.net/laizuling/article/details/51162011)）。

所以，两个控制点的坐标为：

```kotlin
val controlPoint1X = (startPoint.x + endPoint.x) / 2
val controlPoint1Y = startPoint.y

val controlPoint2X = (startPoint.x + endPoint.x) / 2
val controlPoint2Y = endPoint.y
```

这样绘制出来的曲线比较符合我们的要求。

所以，最终贝塞尔曲线path计算方法如下：

```kotlin
var bezierPath = Path()
bezierPath.moveTo(pointList.first().x, -pointList.first().y)
pointList.forEachIndexed { index, startPoint ->
    when (index) {
        pointList.lastIndex -> {
            //在绘制P(n-1) ~ P(n)点的贝塞尔曲线时，已经绘制到了P(n)点，所以此处不用再绘制
        }
        else -> {
            val endPoint = pointList[index + 1]
            bezierPath.cubicTo(
                (startPoint.x + endPoint.x) / 2,
                -startPoint.y, //为了解决view坐标原点在左上角而做的特殊处理，下同
                (startPoint.x + endPoint.x) / 2,
                -endPoint.y,
                endPoint.x,
                -endPoint.y
            )
        }
    }
}
```

# 给Path添加渐变背景

我们可以使用`Paint.setShader(Shader shader)`方法，在绘制Path的时候绘制渐变背景。

渐变背景使用Shader实现。

为了确保绘制效果，我们需要在Path计算完成后，将其闭合，以确保绘制的背景在我们需要的范围内：

```kotlin
        val shadowPaint = Paint(Paint.ANTI_ALIAS_FLAG)
        shadowPaint.style = Paint.Style.FILL
        val shader =
            LinearGradient(0F, 0F, 0F, 500F, Color.GREEN, Color.TRANSPARENT, Shader.TileMode.CLAMP)

        shadowPaint.shader = shader

        val shadowPath = Path(path)
        shadowPath.lineTo(endPoint.x, 800F)
        shadowPath.lineTo(startPoint.x, 800F)
        shadowPath.lineTo(startPoint.x, startPoint.y)
        shadowPath.close()
        canvas.drawPath(shadowPath, shadowPaint)
```

# 给Path添加动画

为了让Path看起来是从起点慢慢绘制到终点去的，我们可以先计算path的总长度，然后结合`ValueAnimator`实时获得对应长度的path并绘制：

```kotlin
var mValueAnimator = ValueAnimator.ofFloat(0f, 1f)
mValueAnimator.duration = 10000
mValueAnimator.repeatCount = ValueAnimator.INFINITE
mValueAnimator.interpolator = AccelerateDecelerateInterpolator()
mValueAnimator.addUpdateListener { animation -> //获取从0-1的变化值
    progress = animation.animatedValue as Float
    //不断刷新绘图，实现路径绘制
    invalidate()
}
mValueAnimator.start()
```

然后在`onDraw()`方法中绘制对应的path：

```kotlin
var mPathMeasure: PathMeasure = PathMeasure(bezierPath, false)
val totalPathLength = mPathMeasure.length //获取path总长度

// 按照进度绘制贝塞尔曲线
val stopD = progress * totalPathLength
mPathMeasure.getSegment(0F, stopD, dstPath, true) //按照长度比例截取对应的path并赋值给dstPath

//bezier anim
canvas.drawPath(dstPath, bezierPaint) //绘制对应的path
```



![](https://jixiaoyong.github.io/images/20200413222302.jpg)



# 注意事项

* 使用canvas绘制坐标时，需要注意android的坐标原点位于屏幕左上角。所以在绘制曲线图时可以先将坐标原点向下平移一段距离，再绘制对应坐标（可以绘制实际的y坐标负值）

* 在拼接贝塞尔曲线的path时候注意，`path.moveTo()`方法会将path切断

  

# 参考资料

https://wenku.baidu.com/view/c790f8d46bec0975f565e211.html
https://blog.csdn.net/laizuling/article/details/51162011

