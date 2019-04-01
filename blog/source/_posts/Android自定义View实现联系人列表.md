---
title: Android自定义View实现联系人列表
abbrlink: a2c96aac
date: 2017-09-03 23:25:05
---



# 自定义的view

* LetterIndex.java extends View
* ContactsListView.java extends RecyclerView
#分析
* 联系人列表有两个要点
  * 字母导航栏
    通过自定义View画出26个字母，设置滑动监听事件，根据上下滑动的距离判断当前选中的字母，并相应更新界面。
  * 列表中的字母标题
    针对item中的联系人姓名首字母对应的tag作比较，若与前一个相同则不显示title，否则显示。
* 事件联动
  * 当滑动字母导航栏时，除了处理本身的变化外，还要留出接口，以便其他控件获取当前选中的字母。
  * 联系人列表滑动时，除了处理本身变化外，同样要留出接口以便获取当前置顶的item对应的字母
  * 字母导航栏要留出方法，以便其他控件指定选中的字母，并更新界面
# 具体代码
**ContactsListView.java**
重写该类主要是为了实现ItemDecoration根据不同的item变化，同时可以从xml布局文件中获取ItemDecoration的自定义属性。
主要代码：

```
    public ContactsListView(Context context) {
        this(context, null, 0);
    }

    public ContactsListView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public ContactsListView(Context context, @Nullable AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        mTypeArray = context.obtainStyledAttributes(attrs, R.styleable.MyRecyclerDecoration);
        mContext = context;
    }

```
故而在其内部自定义了一个继承自ItemDecoratio得静态内部类Decorationn类：

```
public Decoration(List<String> data){
//获取要显示的联系人数据对应的英文tag集合
//初始化各种自定义属性
//例如颜色：mColorLetterText = mTypeArray.getColor(R.styleable.MyRecyclerDecoration_color_letter_text, 0xff152648);

}

@Override
        public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state){
//画出各个导航title
}


@Override
        public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
//画出置顶的导航title
}

 @Override
        public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State
                state) {
//判断是否画出导航title
 super.getItemOffsets(outRect, view, parent, state);
            int position = ((RecyclerView.LayoutParams) (view.getLayoutParams())).getViewAdapterPosition();

            if (position != -1) {
                String text = mDatas.get(position).substring(0, 1).toUpperCase();
                if (position == 0) {
                    outRect.set(0, mTitleHeight, 0, 0);
                } else if (text != null && !text.equals(mDatas.get(position - 1).substring(0, 1).toUpperCase())) {
                    outRect.set(0, mTitleHeight, 0, 0);
                } else {
                    outRect.set(0, 0, 0, 0);
                }
            }
}

private void drawText(Canvas canvas, float left, float right, View child, String text) {
//画出文字
}
```

**LetterIndex.java**
该类用来画出字母导航栏，并且提供方法获取/设置当前选中的字母

```
 public interface onIndexClickListener {
        void onIndexClick(int chooseId);
        void onActionUp();
    }

    public void setOnIndexClickListener(onIndexClickListener listener) {
        this.mClickListener = listener;
    }

    public void setChooseId(int chooseId) {
        if (chooseId >= 0 && chooseId < mIndexTexts.length) {
            mChooseId = chooseId;
            invalidate();
        }
    }
```
然后重写`onTouchEvent(MotionEvent event)`方法，在ACTION_DOWN、ACTION_MOVE、ACTION_UP时调用对应的方法即可。

重写onDraw()方法，画出对应的界面

```
   @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        int height = getHeight() - getPaddingTop() - getPaddingBottom();
        float width = getWidth();
        //childHeight 是每一个字母所在单元的高度
        float childHeight = (float) height / mIndexTexts.length;

        //如果被点击了，就画出背景
        if (isClick) {
            mPaint.setColor(mColorIndexBg);
            canvas.drawRect(0, 0, width, height, mPaint);
        }

        Rect bounds = new Rect();
        mPaint.setTextSize(mSizeText);
        mPaint.setTextAlign(Paint.Align.CENTER);
        for (int i = 0; i < mIndexTexts.length; i++) {
            String text = mIndexTexts[i];
            mPaint.setColor(mColorText);
            //在被选中的字后面画一个圆，并改变字的颜色
            if (i == mChooseId) {
                mPaint.setColor(mColorChooseTextBg);
                canvas.drawCircle(width / 2, childHeight / 2 + i * childHeight,
                        mSizeText / 2 + 2, mPaint);
                mPaint.setColor(mColorChooseText);
            }
            mPaint.getTextBounds(text, 0, text.length(), bounds);
            //bounds里面保存着要画的字的一些属性，如x，y，centerX，centerY等，
            //要注意 canvas.drawText（text,x,y,mpaint）中y并不是text的最低端，而是baseline。
            float x = width / 2;
            float y = -bounds.centerY() + childHeight / 2 + i * childHeight;
            canvas.drawText(text, x, y, mPaint);
        }
    }

```
# 源码

源代码在我的Github，[点这里](https://github.com/jixiaoyong/my_application_on_deepin/tree/master/contactsdemo/src/main)可以找到。

# 预览如下

![预览.gif](http://upload-images.jianshu.io/upload_images/120748-183eea3cad2b42ac.gif?imageMogr2/auto-orient/strip)



<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android自定义View实现联系人列表.md</div>"></iframe>
