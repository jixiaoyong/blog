---
title: Android笔记之Xfermode
tag: android
date: 2020-4-15 14:05:03

---

`Xfermode`是Android中用来指示`Paint`绘制的内容与View中已有内容的混合计算方式,也就是用来确定图形绘制到目标图形的时候，如何处理两个图形重合部分的颜色变化。共18个，分为Alpha合成和混合两种。

设要绘制的图形为`src`，已经绘制好的图形为`dst`。

> 需要注意的是，这些图片除了要绘制的图形有着色之外，其他部分要为透明，并且**包括透明区域在内的图片大小（宽高）要能完全覆盖另外一张图片的图形区域**，否则绘制出的图形可能与预设的效果不一致

按照官方的定义，不同`Xfermode`绘制结果如下：

![](https://camo.githubusercontent.com/9c73f1b36d2ee358cd51e1b080cbad4cba18f39d/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f312f31342f313630663261356630393964353963353f773d33313226683d33393126663d6a70656726733d313238363132)

# 注意事项

要实现如上效果，需要注意：

* `src`和`dst`符合要求（要有合适的透明区域）

  这是因为`xfermode`的效果，使用透明部分的像素与已有图形对应位置交叉作用，得出所需要的效果，如果透明区域过小，则无法作用到对应的图形。下面这个来自Hencoder.com的图可以很形象的解释：

  ![](https://jixiaoyong.github.io/images/20200416213802.jpg)

* 在新的图层绘制（在新的图层按照`xfermode`规则绘制，然后再将其绘制到原有图层）：

  ```kotlin
  //新建图层   
  val saveCount = canvas.saveLayer(0F,0F,width.toFloat(),height.toFloat(),null,Canvas.ALL_SAVE_FLAG)
  
  //dst  已经绘制的图形  ;  src 我们要绘制的图形
  canvas.drawBitmap(dst,0F, 0F, dstPaint)
  
  srcPaint.xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_OUT)
  canvas.drawBitmap(src,0F, 0F, srcPaint)
  srcPaint.xfermode = null
  
  //将新图层绘制到原有图层上
  canvas.restoreToCount(saveCount)
  ```

  

* 关闭硬件加速（可选）

  硬件加速的本质是把一部分CPU计算的工作量交给GPU完成，可以加速绘制速度。

  但是由于硬件加速不支持`canvas.drawXXX()`的部分方法，为了避免在某些机型上面无法使用这些方法，可以关闭硬件加速：

  ```java
  view.setLayerType(LAYER_TYPE_SOFTWARE, null);
  ```

  关于硬件加速更详细的说明可以参考这里：[HenCoder Android 自定义 View 1-8 硬件加速](https://hencoder.com/ui-1-8/)



# Xfermode分类

[HenCoder.com](https://hencoder.com/ui-1-2/)关于`PorterDuff.Mode.DST_IN`的动画解释：

![](https://jixiaoyong.github.io/images/20200416214402.gif)

可以看出，`Xfermode`的本质是处理`dst`与`src`**重合与未重合部分**的展示与否，以及颜色变化。

> 这里的“重合部分”与“未重合部分”，其实也包括了各个图形的透明部分，将`dst`与`src`的透明与不透明颜色相互作用，才会出现下述效果。



| 名称       | 含义                                            |
| ---------- | ----------------------------------------------- |
| `CLEAR`    | 清除所有内容                                    |
| `DST`      | 只绘制`DST`                                     |
| `DST_ATOP` | 先绘制`SRC`，再在顶部绘制`DST`与`SRC`重合的部分 |
| `DST_IN`   | 只绘制`DST`与`SRC`重合部分                      |
| `DST_OUT`  | 只绘制`DST`与`SRC`未重合部分                    |
| `DST_OVER` | 将`DST`绘制在`SRC`上面                          |
| `SRC`      | 只绘制`SRC`                                     |
| `SRC_ATOP` | 先绘制`DST`，再在顶部绘制`SRC`与`DST`重合的部分 |
| `SRC_IN`   | 只绘制`SRC`与`DST`重合部分                      |
| `SRC_OUT`  | 只绘制`SRC`与`DST`未重合部分                    |
| `SRC_OVER` | 将`SRC`绘制在`DST`上面                          |
|            |                                                 |
| `XOR`      |                                                 |
| `ADD`      |                                                 |
| `DARKEN`   |                                                 |
| `LIGHTEN`  |                                                 |
| `MULTIPLY` |                                                 |
| `OVERLAY`  |                                                 |
| `SCREEN`   |                                                 |

各个效果如下(源码及使用见[github](https://github.com/jixiaoyong/library/commit/71864b6546460acfaae6299890a0cf76da76b7d7))：

![](https://jixiaoyong.github.io/images/20200415211144.png)

# 参考文献

[HenCoder Android 开发进阶: 自定义 View 1-2 Paint 详解](https://hencoder.com/ui-1-2/)

[HenCoder Android 自定义 View 1-8 硬件加速](https://hencoder.com/ui-1-8/)

[PorterDuff.Mode](https://developer.android.google.cn/reference/android/graphics/PorterDuff.Mode.html)
