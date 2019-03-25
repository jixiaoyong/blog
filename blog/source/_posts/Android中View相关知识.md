---
title: Android中View相关知识
date: 2019-03-24 12:43:21
tag: 
---



# View的坐标

Android中的坐标，以屏幕左上角顶点为原点(0,0)，以横轴为x轴，竖轴为y轴，数值依次递增。

View的坐标信息有以下几种，其坐标都是以父View的左上角顶点为原点：

* x，y 是View的左上角坐标。

* translationX，translationY是View左上角顶点与父容器左上角顶点的偏移量，默认为0。

* left，top 是分别是View左上角顶点的x轴，y轴坐标。

* right，bottom分别是View右下角顶点的x轴，y轴坐标。

> 注意

x = translationX + left；

y = translationY + top；

改变translationX/Y的值便可以更改View的位置。当View平移的时候，代表原始位置信息的left，right，top，bottom的值并不会变化。

在OnTouch事件中，我们可以从event得到两种值：

event.rawX,event.rawY 代表 相对于手机屏幕原点的坐标

event.X,event.Y 代表 相对于当前View左上角的坐标

TouchSlop则代表认为滑动开始的最小距离

```kotlin
ViewConfiguration.get(this).scaledTouchSlop
```

# 滑动

mScroller.startScroll()方法可以实现平滑的滑动

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

Scroller不能使View滑动，而只能配合View的computeScroll()方法实现是View的内容滑动的效果。

* mScroller.startScroll()记录下要滑动的数据，而invalidate()通知View重绘；

* 每次重绘都会调用computeScroll()方法，利用mScroller计算出接下来要scrollTo()的具体值并执行，再次postInvalidate()通知View重绘；

* 如此反复直到绘制滑动完毕。


<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="39" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android中View相关知识.md</div>"></iframe>
