---
title: Android事件分发
tags: android
abbrlink: c0fefed0
date: 2018-04-24 20:25:33
---

Android事件分发，指手指点击屏幕后，从Activity、ViewGroup到View的一系列过程。

# 简介

Android系统的窗口机制如下图：

Activity内有一个Window对象，其实现类是PhoneWindow；

DecorView为顶层View，DecorView是一个FrameLayout，其中有TitleView和ContentView；



![Android系统窗口管理机制](https://github.com/jixiaoyong/jixiaoyong.github.io/blob/master/images/blog/2018-04/AndroidDispatchTouchEvent.png?raw=true)



TitleView为标题栏，ContentView就是平时在Activity的onCreate()方法中设置的视图，TitleView可以用`this.requestWindowFeature(Window.FEATURE_NO_TITLE);`隐藏掉，但是必须注意要在setContentView()之前，原因如下所示：

```java
public void setContentView(View view) {
    getWindow().setContentView(view);
    initWindowDecorActionBar();
}
```
# 点击事件 Activity --> ViewGroup

点击事件发生后，首先被调用的是`Activity.dispatchTouchEvent()`

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
可以看到，其内部先调用了`getWindow().superDispatchTouchEvent(ev)`这个方法，getWindow()返回的mWindow是PhoneWindow的对象。

```java
mWindow = new PhoneWindow(this, window, activityConfigCallback);
```

再看看PhoneWindow.superDispatchTouchEvent()方法，显然又调用了DecorView的superDispatchTouchEvent()方法,在该方法中，调用了FrameLayout.dispatchKeyEvent(event)，此时**点击事件从Activity转到了ViewGroup中**。

```java
//PhoneWindow
@Override
public boolean superDispatchKeyEvent(KeyEvent event) {
    return mDecor.superDispatchKeyEvent(event);
  }

//DecorView extends FrameLayout
public boolean superDispatchKeyEvent(KeyEvent event) {
    // Give priority to closing action modes if applicable.
    if (event.getKeyCode() == KeyEvent.KEYCODE_BACK) {
        final int action = event.getAction();
        // Back cancels action modes first.
        if (mPrimaryActionMode != null) {
            if (action == KeyEvent.ACTION_UP) {
                mPrimaryActionMode.finish();
            }
            return true;
        }
    }
    return super.dispatchKeyEvent(event);
}
```

# 点击事件 ViewGroup --> View

ViewGroup与事件分发的方法有三个：

* `dispatchTouchEvent()`  分发事件，每次都会被调用
* `onInterceptTouchEvent()`  拦截事件，如果当前ViewGroup已经决定拦截事件，那么不会再调用
* `onTouchEvent()`  处理点击事件,如果设置了`mOnTouchListener`的话，则不会回调本方法

这三个主要方法关系如下（伪代码，来自《Android开发艺术探索》）：

```kotlin
//每次点击事件回调该方法
override fun dispatchTouchEvent(event: MotionEvent): Boolean {
    var result = false
    if (onInterceptTouchEvent(event)) {//viewGroup会回调该方法，确认是否拦截点击事件
        result = onTouchEvent(event)//对点击事件进行处理
    } else {
        result = child.dispatchTouchEvent(event)
        }
    }
    return result
}
```

当ViewGroup.dispatchTouchEvent()被调用后，会通过一系列条件判断是由ViewGroup拦截该事件，还是由子View消耗该事件。

主要流程分为两部分

**1.检查是否需要拦截**

* 每次ACTION_DOWN事件都需要调用`onInterceptTouchEvent()`方法判断是否需要拦截
* 其他MotionEvent事件，如果有能处理点击事件的子View（`mFirstTouchTarget != null`）且`disallowIntercept`为false也需要调用`onInterceptTouchEvent()`方法判断是否需要拦截，否则不需要拦截
* 其余情况都需要拦截（没有可以处理点击事件的子View，并且不是ACTION_DOWN事件）

**2.遍历ViewGroup的所有子View，寻找一个可以处理点击事件的子View**

* `dispatchTransformedTouchEvent()`   调用了子View的`dispatchTouchEvent()`
* `addTouchTarget()`   对`mFirstTouchTarget `进行更新

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    
        // 1. Check for interception.判断是否需要拦截
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {//mFirstTouchTarget表示能处理点击事件的子View
            //FLAG_DISALLOW_INTERCEPT每次ACTION_DOWN都会被重置
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);//调用拦截方法
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }
    
    	//2.遍历子View，寻找可以处理点击事件的子View
    	if (!canceled && !intercepted) {
       		 for (int i = childrenCount - 1; i >= 0; i--) {
                 if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                     // Child wants to receive touch within its bounds.
                     ...
                     newTouchTarget = addTouchTarget(child, idBitsToAssign);
                     alreadyDispatchedToNewTouchTarget = true;
                     break;
                  }
        	 }
        }
}
```

dispatchTransformedTouchEvent()方法如下，由于`child != null`其内部调用`child.dispatchTouchEvent(event)`方法，如此循环直到子View是一个View（单就ViewGroup和View而论）即**将点击事件从ViewGroup分发到了View**。

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
    ...
    if (child == null) {
        handled = super.dispatchTouchEvent(event);
    } else {
        handled = child.dispatchTouchEvent(event);
    }
}
```

如果有子View可以处理点击事件，在`addTouchTarget()`方法内部对`mFirstTouchTarget`进行更新

```java
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```
# 点击事件 View内部

View的点击事件分发主要涉及到两个方法：

* `dispatchTouchEvent()`
* `onTouchEvent()`

其点击事件分发用伪代码表示如下:

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    if(mListenerInfo.mOnTouchListener.onTouch(this, event)){
        return true;
    }else{
        return onTouchEvent(event);
    }
}
```

可见View的dispatchTouchEvent()方法中，如果View注册了OnTouchListener则会先执行`mOnTouchListener.onTouch()`方法,如果该方法返回false才会执行`onTouchEvent()`。

在看onTouchEvent()方法：

* 如果View处于**不可用状态**下，也会消耗点击事件，只不过没有反应
* 如果注册了OnClickListener会在ACTION_UP的时候调用`mOnClickListener.onClick(this)`

```java
public boolean onTouchEvent(MotionEvent event) {
    ...
    if(CLICKABLE&&LONG_CLICKABLE){//LONG_CLICKABLE默认为false，CLICKABLE、LONG_CLICKABLE会在设置点击事件时被设置为true
        switch (action) {
            case MotionEvent.ACTION_UP:{
                ...
                performClick();//如果注册了OnClickListener则会调用其onClick()方法
            }
        }
    }
}

public boolean performClick() {
    ...
    mListenerInfo.mOnClickListener.onClick(this);
    ...
}
```

# 总结

整个Android的时间分发始于Activity,经过PhoneWindow、DecorView到达ViewGroup，再逐层分发到View中。

如果底层没有处理点击事件，则又一层层向上返回，直到最顶层消耗掉点击事件。

# 参考资料

《Android开发艺术探索》