---
title: Android常用知识	
date: 2017-9-20 23:25:05
---

# 自定义view


* 画矩形

  ```java
  canvas.drawRect(left, top, right, bottom, paint);
  ```

  其中：`left/top ` 是该view的左上顶点到父容器左边和顶端的距离`right/bottom` 是该view的右下顶点到父容器左边和顶端的距离

* 写字

  ```java
  Rects bounds = new Rects();
  mPaint.setColor(mColorText);
  mPaint.setTextSize(mSizeText);
  mPaint.setTextAlign(Paint.Align.CENTER);
  mPaint.getTextBounds(text, 0, text.length(), bounds);
  float x =  width / 2;
  float y = -bounds.centerY() + childHeight / 2 + i * childHeight;
  canvas.drawText(text, x, y, mPaint); 
  ```

  ​

  当text为居中对齐时，`centerX`和`centerY`是该字的正中心，以text的左下角为原点，故而`centerY`为`-sizeOfText/2`。


* 自定义属性

  **attrs.xml**

  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <resources>
      <declare-styleable name="LetterIndex">
          <attr name="color_text" format="color" />
          <attr name="color_index_bg" format="color" />
      </declare-styleable>
  </resources>
  ```

​	**MyView.java**

```java
AttributeSet attrs = ***;
TypedArray arr = context.obtainStyledAttributes(attrs, R.styleable.LetterIndex);
mColorText = arr.getColor(R.styleable.LetterIndex_color_text, 0x52000000);
```



注意默认值可以为```int```数字，但是需为```0x```开头八位数字，前两位表示透明度
​	**main_act.xml**
```xml
<com.text.MyView
***
app:color_text = "#003903"
/>
```
* 对外预留onClick监听接口
  **MyView.java**
 ```java
//定义
    private onIndexClickListener mClickListener;
    public interface onIndexClickListener{
        void onIndexClick(int chooseId);
    }
    public void setOnIndexClickListener(onIndexClickListener listener){
        this.mOnIndexClick = listener;
    }
//使用，在需要监听的动作发生时调用该方法
    mClickListener.onIndexClick(mChooseId);
 ```
​	**MainAct.java**
```java
 mView.setOnIndexClickListener(new LetterIndex.onIndexClickListener() {
            @Override
            public void onIndexClick(int chooseId) {
                Log.d("TAG", "onIndexClick: chooseid is" + chooseId);
            }
        });
```
* RecyclerView滑动事件
  `manager.scrollToPositionWithOffset(n, 0);`
  `n`为要滑到顶端的`position`
