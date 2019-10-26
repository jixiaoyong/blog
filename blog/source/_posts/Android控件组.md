---
title: Android控件组
abbrlink: 3b69424a
date: 2017-09-20 23:25:05
---



这几天的工作中用到了控件组来实现复杂布局，效果不错，记录下来备用。

# 1. 定义控件组布局xxx_layout.xml



在这里定义要使用的控件组布局，这里的布局决定了布局显示的样子。

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="10dp">

    <ImageView
        ... />

    <EditText
        .../>

    <ImageView
        ... />
      
</LinearLayout>
```



# 2.新建自定义属性文件attr.xml（可选）

* 在```values/attr.xml```下新建对应文件，并添加对于自定义属性，以便可以在activity布局文件里面使用到该件组时自定义控一些属性。

  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <resources>

      <!-- MSearchBar表明是给该控件使用的自定义属性 -->
      <declare-styleable name="MSearchBar">

          <!-- 以下为示例，可以根据需求增减 -->
          <attr name="textColor" format="color" />
          <attr name="textSize" format="dimension" />

      </declare-styleable>

  </resources>
  ```

* 在该自定义控件的类xxx.java中，通过如下语句获取从用户使用时赋给这些属性的值：

  ```java
  TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.MSearchBar);
  float textSize = array.getDimension(R.styleable.MSearchBar_textSize, 13);
  ```

  ​

* 用户动态给这些属性赋值：

  ```xml
   <cf.android666.jixiaoyong.weightgroup.weight.MSearchBar
          ...
          app:textColor="@color/colorPrimaryDark">

  </cf.android666.jixiaoyong.weightgroup.weight.MSearchBar>
  ```

  注意，这里写的是```app:```而非```android:```。

  ​

# 3. 新建组合控件的类XXX.java

* 新建XXX.java，继承自布局文件的父布局LinearLayout

* 更改参数少的构造方法的```super(a1,a2,a3)```为```this(a1,a2,a3)```，其中```this()```中的参数个数为参数最多的构造方法的参数数。

  **注意** ：一定要做这一步，否则在使用该自定义控件组时，新建该类的对象会提示出错

* 在最终会被调用的构造方法里面将xml里面定义的布局加载进来：

  ```java
  //注意三个参数：布局文件：R.layout.weight_group_layout, root：this,是否依附到root：true
  //必须有前两个参数，否则控件的宽高等会有异常
  View view = LayoutInflater.from(context).inflate(R.layout.weight_group_layout, this,true);
  ```

* 使用自定义属性：

  ```java
  public MSearchBar(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
          super(context, attrs, defStyleAttr);
    
    		//do sth
    
          TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.MSearchBar);
          float textSize = array.getDimension(R.styleable.MSearchBar_textSize, 13);
          editText.setTextSize(textSize);
    
      }
  ```



* 给控件组内部控件添加点击事件监听：

  * xxx.java 要实现点击事件监听接口：

    ```java
    public class MSearchBar extends LinearLayout implements View.OnClickListener{}
    ```

  * 自定义接口，供使用xxx.java类时实现对监听事件的处理：

    ```java
        public void setImgLeftOnClickListener(OnImgClickListener listener){
            listenerL = listener;
        }

        public interface OnImgClickListener{
            public void onClick();
        }

        private OnImgClickListener listenerL;
    ```

  * 对要监听点击事件的控件设置监听，并调用```listener.onClick()```方法：

    ```java
    	//在构造方法等地方设置监听事件
    	imageViewLeft.setOnClickListener(this);

    	//在xxx.java中重写onClick()方法
        @Override
        public void onClick(View view) {
            switch (view.getId()) {
                //ImageLeft
                case R.id.imageView:
                    if (listenerL != null) {
                        listenerL.onClick();
                    }
                    break;
                //imageRight
                case R.id.imageView2:
                    if (listenerR != null) {
                        listenerR.onClick();
                    }
                    break;
            }
        }
    ```



# 4. 使用自定义控件组xxx.java

* 在布局文件main_activity.xml中添加该控件

  ```xml
   <cf.android666.jixiaoyong.weightgroup.weight.MSearchBar
          android:id="@+id/search_bar"
          android:layout_width="match_parent"
          android:layout_height="wrap_content"
          app:textColor="@color/colorPrimaryDark">

   </cf.android666.jixiaoyong.weightgroup.weight.MSearchBar>
  ```

  ​

* 在java中使用该控件，设置监听事件

```java
    MSearchBar searchBar = (MSearchBar) findViewById(R.id.search_bar);
    searchBar.setImgLeftOnClickListener(new MSearchBar.OnImgClickListener() {
            @SuppressLint("WrongConstant")
            @Override
            public void onClick() {
                Toast.makeText(MainActivity.this, "Click on Left", Toast.LENGTH_SHORT).show();
            }
        });

```



# 5.效果预览


![效果预览](http://upload-images.jianshu.io/upload_images/120748-48d0cbbf03ded7f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 6.源码

demo的github链接:[github](https://github.com/jixiaoyong/my_application_on_deepin/tree/master/weightgroup)


