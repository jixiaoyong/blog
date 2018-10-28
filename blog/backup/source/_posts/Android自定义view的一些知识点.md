---
title: Android自定义view的一些知识点
tags: android
abbrlink: e41787ec
date: 2018-02-19 21:59:28
---

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

![](http://jixiaoyong.github.io/blog/images/default/2018-02-19/PorterDuffMode.png)

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

