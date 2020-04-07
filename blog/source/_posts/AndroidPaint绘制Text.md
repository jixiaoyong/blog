---
title: Android paint绘制text
date: 2020-04-07 22:05:20
---



Android中绘制文字的方法如下：
```java
    /**
     * Draw the text, with origin at (x,y), using the specified paint. The origin is interpreted
     * based on the Align setting in the paint.
     *
     * @param text The text to be drawn
     * @param x The x-coordinate of the origin of the text being drawn
     * @param y The y-coordinate of the baseline of the text being drawn
     * @param paint The paint used for the text (e.g. color, size, style)
     */
    public void drawText(@NonNull String text, float x, float y, @NonNull Paint paint) {
        super.drawText(text, x, y, paint);
    }
```
其中`y`是**文字baseline的y坐标**。

下图表示`Paint.FontMetrics`中存储的文字的各种信息（来源：[简书](https://www.jianshu.com/p/c1575636741e)）：
![](https://jixiaoyong.github.io/images/20200407220410.png)

我们没法直接获取到`baseline的坐标`，所以只能从另外一个角度考虑：
因为在绘制文字时，文字的上下中心（即上图中的`center`）是确定的，我们只要计算出`center`到`baseline`之间的偏移量，就可以计算出`baseline的y坐标`。

又根据这个[文章](http://www.imooc.com/article/277490?block_id=tuijian_wz )：
```
基线到中线的距离 = (descent + ascent) / 2 - descent = (ascent - descent) / 2
```
> `(descent + ascent) / 2`是中线center的值，而根据上图可知`(descent + ascent) / 2 - descent`的值就是baseline到center的距离。

所以
```
baseline的y坐标 = 文字的上下高度中心 + baseline的竖坐标和文字上下实际中心的偏移量
                = center.y + 基线到中线的距离
                = center.y + (ascent - descent) / 2
```
> 这个`center.y`根据场景不同可以是一行的行中心（文字在一行居中显示），或者控件的上下中心（文字在控件上下居中）

得出结论：

由于android绘制文字时，**并不是从文字高度的中间开始绘制，而是从baseline开始绘制**。所以在绘制文字时，为了使文字高度居中（在所指定的空间内居中，比如某一行，就在该行限定的高度内居中显示；某一控件，则整个控件的上下中间显示），需要在计算出来的**文字上下中心的y坐标基础上加上baseline到文字中线的偏移量**。

除此之外，也可以类比得到：`baseline.y = center.y + (bottom.y - top.y) / 2 - bottom.y`



> 基线（baeseline），坡顶（ascenter）,坡底（descenter）
>
> 上坡度（ascent），下坡度（descent）
>
> 行间距（leading）：坡底到下一行坡顶的距离
>
> 字体的高度＝上坡度＋下坡度＋行间距
>
> https://blog.csdn.net/hanyongbai/article/details/84418369

参考文章：
https://www.jianshu.com/p/c1575636741e
https://blog.csdn.net/hanyongbai/article/details/84418369
http://www.imooc.com/article/277490?block_id=tuijian_wz
https://blog.csdn.net/xuxingxing002/article/details/50971606