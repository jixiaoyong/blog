---
title: Android自定义view的一些知识点
tags: android
abbrlink: e41787ec
date: 2018-02-19 21:59:28
---

# View的绘制

View的绘制分为3部分：

1. measure

   测量，决定了View的测量宽、高。几乎所有情况下都等同于View的最终宽、高（如果View需要多次measure才能确定大小，或者重写了`layout()`方法，并修改了传入的值的话则不会相等）。

2. layout

   布局，决定View的四个顶点坐标和实际的宽、高。

3. draw

   绘制，决定了View的具体显示内容。

其中通过ViewRootImpl类的`performTraversals()`依次调用`performXXX()`方法。

# MeasureSpec

MeasureSpec是一个32位int值，高2位表示SpecMode，低30位表示SpecSize。

SpecMode有3种可能值：

* UNSPECIFIED 父容器没有限定View大小，可以是任意需要的大小
* EXACTLY 父类指定了View的具体大小，View的最终大小就是这个值(match_parent或者具体数值)
* AT_MOST View可以是这个值以内的任意大小(wrap_content)

我们指定的View的LayoutParams和父容器（DecorView则是窗口的尺寸，普通View是父容器的MeasureSpec）一起决定了View的MeasureSpec，进而决定了View的宽高。

SpecSize决定于父容器的尺寸、以及View的margin和padding。

# View绘制流程

final类型的`measure()`方法调用`onMeasure()`方法。

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec){
   if (forceLayout || needsLayout) {
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } 
   }
}
```

在`onMeasure()`调用了`setMeasuredDimension()`方法设置了View宽、高的测量值。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;//返回getSuggestedMinimumWidth/Height的大小
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;//返回测量大小
            break;
        }
        return result;
}
```

`getSuggestedMinimumXXX()`的值：

如果View没有背景，则返回的是View的`android:miniWidth`指定的值；

如果View有背景，则返回的是背景的`minimumWidth`的值和`android:miniWidth`指定的值中最大的一个值。

```java
protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }

protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());

    }
```

由此，我们知道，如果直接继承自View的控件必须重写`onMeasure()`方法，设置wrap_content时候控件的大小。这是因为：

wrap_content对应的specMode是AT_MOST模式，其宽高等于`specSize`。

根据ViewGroup的`getChildMeasureSpec()`方法，我们知道此时的`specSize`是父容器目前可以用的大小，即这种情况下wrap_content的效果和match_parent的效果是一样的。

要避免这种情况，就需要重写`onMeasure()`方法，在里面专门指定wrap_content时View对应的大小。

# 获取View的宽高

由于View的绘制和Activity的生命周期不同步，所以在`onCreate()/onStart()/onResume()`中都无法有效获取View的宽高。使用以下方式则可以正常获取View的宽高：

1. Activity/View#onWindowFocusChanged()

   当前的Window获取或失去焦点的时候调用，此时View已经初始化完毕，可以获取宽、高。

   Activity窗口焦点变化(onPause/onResume)时会被调用多次。

2. View#post(runnable)

   该runnable在view的消息队列尾部，被执行时View已经初始化好了，可以在这里获取宽高。

3. ViewTreeObserver

   注册onGlobalLayoutListener，当View树的状态变更，或者View树内部View可见性发生变化就会被回调。

   当View树的状态变更可能被调用多次。

4. View#measure()

   手动调用`measure()`方法获取宽高。

# draw过程

绘制过程分为以下几步：

1. 绘制背景 `background.draw(canvas);`
2. 绘制自身 `onDraw(canvas);`
3. 绘制children `dispatchDraw(canvas);`
4. 绘制装饰 `onDrawForeground(canvas);`

`setWillNotDraw()`表示当前的ViewGroup不需要绘制任何内容，系统会对此进行优化（默认启用）。如果ViewGroup需要绘制内容时，则需要手动关闭这个标志。

```java
public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```

# 绘制两个图形重叠部分

android自定义view时两个图形重叠部分的绘制方式,一定要调用`canvas.saveLayer()` ，否则不生效。

```java
        //这个步骤十分重要，将当前画布保存为新的一层
        int save = canvas.saveLayer(0,0,mWidth,mHeight,null,Canvas.ALL_SAVE_FLAG);

        Paint paint = new Paint();
        paint.setColor(mBackgroundColor);

        RectF backgroundRectF = new RectF(0, 0, mWidth, mHeight);
        canvas.drawRoundRect(backgroundRectF, mRadius, mRadius, paint);

        paint.setColor(mForwardColor);
        //设置二者重叠部分的绘制方式
        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
        RectF progressRectF = new RectF(0, 0, mWidth * mProgress, mHeight);
        canvas.drawRect(progressRectF,paint);

        //restore to canvas
        canvas.restoreToCount(save);
```

`paint.setXfermode()`可以设置的值参考下图：

![](http://jixiaoyong.github.io/images/20190408175100.png)

参考自[【原】使用Xfermode正确的绘制出遮罩效果 - sky0014 - 博客园  ](https://www.cnblogs.com/DarkMaster/p/4618872.html)



# 适配自定义view宽高，设置默认值

以其宽度为例，在`onMeasure(int widthMeasureSpec, int heightMeasureSpec)`方法中：

```java
int widthMode = MeasureSpec.getMode(widthMeasureSpec);

int widthSize = MeasureSpec.getSize(widthMeasureSpec);

if (widthMode == MeasureSpec.EXACTLY) {
    mWidth = widthSize;
} else {
    mWidth = 100;
    if (widthMode == MeasureSpec.AT_MOST) {
        mWidth = Math.min(mWidth, widthSize);
    }
}
```



# 参考资料

《Android开发艺术探索》

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android自定义view的一些知识点.md</div>"></iframe>
