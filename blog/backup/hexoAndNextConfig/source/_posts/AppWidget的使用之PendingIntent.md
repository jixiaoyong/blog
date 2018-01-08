---
title: AppWidget的使用之PendingIntent
date: 2016-04-25 06:06:06
---

这几天学习 AppWidget ，很简单的组件却花费了不少功夫，今天对 PendingIntent 的用法做了一些简单的整理。

**PendingIntent**

> PandingIntent 就像是一个设计好的处理预案，当达到某个特定条件时，便会调用该 Intent 所指定动作（打开服务，Activity或者发送广播）。
>
> 这里使用该方法在 AppWidget 里面为按钮添加监听事件，当按钮被点击的时候触发相应的动作

AppWidget 和应用程序不再同一个进程当中，而是在 HomeScreen 上面执行,所以不能直接为 AppWidget 中的 Button 添加监听事件，需要用 `remoteViews.setPendingIntent(R.id.widget_button,pendingIntent);`意思是当按下按钮的时候 pendingIntent 中的 Intent 就会执行

PendingIntent 当某个事件出现之后才会执行

RemoteViews对象 代表了一系列的 View 对象，和主程序不在同一个进程为 AppWidget 控件绑定处理器

**流程概述：**

- 添加 appwidget_provider_info.xml 在 res/xml 下新建 appwidget_provider_info.xml

  - 描述 AppWidget 的基本信息如最小高度、宽度等，还有就是该挂件的布局文件

- 在 res/layout 下面为该挂件设置具体的布局样式

  - 向 AppWidget 的布局文件中添加一个 Button
  - 向 AppWidget 的布局文件中添加一个 TextView

- 新建 MyAppWidget.java 继承自 AppWidgetProvider

  在该类的 onUpdate() 方法中为 Button 设置、添加监听事件

  - 建立一个 Intent 对象
  - 用该 Intent 对象创建一个 PendingIntent 对象
  - 创建一个 RemoteViews 对象
  - 用该 RemoveViews 对象为 按钮绑定事件处理器
  - 更新按钮

- 注册事件

- 备注：要是为 AppWidget 中的 Button 设置的事件是打开一个 TargetActivity ，还需要添加一个 TargetActivity 类和对应的布局文件

**以下是代码**

- appwidget_provider_info.xml

这个布局文件是 AppWidget 的信息

描述了 AppWidget 的最小高，最小宽以及它的布局文件

```
<appwidget-provider
    android:minHeight="200dp"
    android:minWidth="300dp"
    android:initialLayout="@layout/app_widget"
    xmlns:android="http://schemas.android.com/apk/res/android" >
</appwidget-provider>

```

- app_widget.xml

这个布局文件是 Widget 在桌面上显示的样式

定义了 AppWidget 中各个组件及其样式

其中 Button 用来响应点击事件，加入 TargetActivity

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="200dp"
    android:layout_height="200dp"
    android:orientation="vertical">

<TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="hello,world!"/>

<Button
        android:id="@+id/app_widget_btn"
        android:layout_width="200dp"
        android:layout_height="150dp"
        android:background="#ff00ff"
        android:text="this is my app widget button"/>
</LinearLayout>

```

- target_activity.xml

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="50sp"
        android:background="#00ff00"
        android:text="\n hello,welcome to target activity!"/>

</LinearLayout>

```

- MyAppWidget.java

主要是修改了 update() 方法：

定义了一个预先设定的动作—- Intent 对象；

利用该 Intent 读写，创建一个 PendingIntent 对象；

创建一个 RemoteView 对象，并为按钮绑定监听事件

刷新 AppWidget。

```
public class MyAppWidget extends AppWidgetProvider {

    @Override
    public void onReceive(Context context, Intent intent) {
        // TODO Auto-generated method stub
        super.onReceive(context, intent);
    }

    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager,
                         int[] appWidgetIds) {
        // TODO Auto-generated method stub
        super.onUpdate(context, appWidgetManager, appWidgetIds);

        //appWidgetIds 每一次向屏幕添加 AppWidget 的时候都会增加一个唯一的 appWidget 的 Id
        for(int i = 0; i < appWidgetIds.length;i++){
          //创建一个 Intent 对象
            Intent intent = new Intent(context,TargetActivity.class);
            //创建一个 PendingIntent 对象
            PendingIntent pendingIntent = PendingIntent.getActivity(context,0,intent,0);
            // remoteViews 代表 AppWidget 上所有的控件
            RemoteViews remoteViews = new RemoteViews(context.getPackageName(), R.layout.app_widget);
            //为按钮绑定事件处理器
            /*
            * 参1，指定被绑定处理器的控件id
            * 参2，指定事件发生时会被执行的 PendingIntent
             */
            remoteViews.setOnClickPendingIntent(R.id.app_widget_btn,pendingIntent);
            //更新 AppWidget ，参1是用于指定被更新 appWidget 的ID
            appWidgetManager.updateAppWidget(appWidgetIds[i],remoteViews);
        }
    }

    @Override
    public void onDeleted(Context context, int[] appWidgetIds) {
        // TODO Auto-generated method stub
        super.onDeleted(context, appWidgetIds);
    }

    @Override
    public void onEnabled(Context context) {
        // TODO Auto-generated method stub
        super.onEnabled(context);
    }

    @Override
    public void onDisabled(Context context) {
        // TODO Auto-generated method stub
        super.onDisabled(context);
    }
}

```

- TargetActivity.java

```
public class TargetActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.target_activity);
    }
}

```

- AndroidManifest.xml

在 AndroidManifest.xml 中注册 TargetActivity 和 MyAppWidget

```
<application>
...
    <activity android:name=".TargetActivity">
    </activity>

    <!-- 注意这里注册了一个 MyAppWidget 接收数据-->
    <receiver android:name=".MyAppWidget">
        <intent-filter>
            <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
        </intent-filter>
        <meta-data
            android:name="android.appwidget.provider"
            android:resource="@xml/appwidget_provider_info"/>
    </receiver>
</application>
```