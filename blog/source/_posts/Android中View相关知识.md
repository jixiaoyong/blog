---
title: Android中View相关知识
date: 2019-03-24 12:43:21
tag: 
---

本文为笔记性质，尚未成文。

# View的坐标

Android中的坐标，以屏幕左上角顶点为原点(0,0)，以横轴为x轴，竖轴为y轴，数值依次递增。

View的坐标信息有以下几种，其坐标都是以父View的左上角顶点为原点：

* x，y 是View的左上角坐标。

* translationX，translationY是View左上角顶点与父容器左上角顶点的偏移量，默认为0。

* left，top 是分别是View左上角顶点的x轴，y轴坐标。

  right，bottom分别是View右下角顶点的x轴，y轴坐标。

> 注意

x = translationX + left；

y = translationY + top；

改变translationX/Y的值便可以更改**View的位置**。当View平移的时候，代表原始位置信息的left，right，top，bottom的值并不会变化。

在OnTouch事件中，我们可以从event得到两种值：

event.rawX,event.rawY 代表 相对于手机屏幕原点的坐标

event.X,event.Y 代表 相对于当前View左上角的坐标

TouchSlop则代表认为滑动开始的最小距离

```kotlin
ViewConfiguration.get(this).scaledTouchSlop
```

# 滑动

mScroller.startScroll()方法可以实现平滑的滑动

scrollX,scrollY表示的是*view的X，Y坐标减去view内容的X，Y坐标*。

所以scrollX>0，则表示view内容向左移动，scrollX<0表示view内容向右移动。类似于窗户(view)位置不变，景色(view内容)的scrollX>0即景色向右移动，则在窗户中看到的效果是景色向窗户左边移动。

```kotlin
private val mScroller = Scroller(context)

fun smoothScrollBy(destX: Int, destY: Int) {

    mScroller.startScroll(scrollX, scrollY, destX, destY, 1000)//destX, destY的值如果是正的话，会向左，上方移动
    invalidate()
}

override fun computeScroll() {
    if (mScroller.computeScrollOffset()) {
        scrollTo(mScroller.currX, mScroller.currY)
        postInvalidate()
    }
}
```

Scroller不能使View滑动，而只能配合View的computeScroll()方法实现是**View的内容滑动**的效果。

* mScroller.startScroll()记录下要滑动的数据，而invalidate()通知View重绘；
* 每次重绘都会调用computeScroll()方法，利用mScroller计算出接下来要scrollTo()的具体值并执行，再次postInvalidate()通知View重绘；
* 如此反复直到绘制滑动完毕。

上述无论是translationX还是scrollX等引起的view变化，都不能改变View的定位（left，right，top，bottom值），而如果**更改margin的值，则可以更改View的定位**。



# Window和WindowManager

WindowManager.LayoutParams.flags有三个常用选项：

* WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL // 只处理Window区域内的点击事件，之外的交给其他Window处理
* WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE // 不接受输入事件，不获取焦点，同时会开启FLAG_NOT_TOUCH_MODAL，最终事件会传递给下层具有焦点的Window
* WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED  // 让Window显示在锁屏界面上

WindowManager.LayoutParams.type代表Window的类型(三个)：

* 应用Window 对应一个Activity。`z-ordered`:1~99
* 子Window 不能单独存在，附属在特定的父Window中，如Dialog。`z-ordered`:1000~1999
* 系统Window 需要系统权限，如Toast，状态栏等。`z-ordered`:2000~2999

`z-ordered`值大的Window会覆盖掉低值的Window。

# TODO

recycleview滑动
ItemTouchHelper源码分析 https://www.jianshu.com/p/130fdd755471
嵌套滑动 https://blog.csdn.net/qq_15807167/article/details/51637678
https://www.cnblogs.com/dasusu/p/9159904.html
~~滑动展示删除按钮~~ https://www.jianshu.com/p/9bfed6e127cc >> 对应的demo：https://github.com/jixiaoyong/DiyWidget/blob/master/diy-widget/src/main/java/cf/android666/applibrary/SwipeRecyclerView.kt



# 参考资料

[View滑动效果常用属性详解：scroll、translation、LayoutParams](https://blog.csdn.net/Holmofy/article/details/53959511)

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android中View相关知识.md</div>"></iframe>
