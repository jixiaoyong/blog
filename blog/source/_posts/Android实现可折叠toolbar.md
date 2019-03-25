---
title: Android实现可折叠toolbar
tags: android
abbrlink: b4832cd1
date: 2018-02-22 15:01:10
---



使用到的类有：

- android.support.design.widget.CoordinatorLayout
- android.support.design.widget.AppBarLayout
- android.support.design.widget.CollapsingToolbarLayout
- android.support.v7.widget.Toolbar

# 效果预览

如图：

![](http://jixiaoyong.github.io/blog/images/default/2018-02-22/coordinatorlayout.gif)

# 简要说明

CoordinatorLayout类，协调者布局，通过Behavior将一个子view（`child`）的行为和另一个子view（`dependency`）的活动联结起来，从而实现子view之间的联动。

AppBarLayout类，是一个实现了材料设计的默认垂直布局的ViewGroup，当其是CoordinatorLayout类的直接子view时,另外一个CoordinatorLayout的子view指定了behavior为AppBarLayout.ScrollingViewBehavior的实例（`app:layout_behavior="@string/appbar_scrolling_view_behavior"`）,且该子view需要是NestedScrollingChild的实现类。

CollapsingToolbarLayout类，提供一个可以折叠的toolbar布局，可以在这个布局里面，设置toolbar以及和toolbar一起联动的子view，本案例中是一张图片。

Toolbar类，实现toolbar的效果。

# 具体实现

源码：[github](https://github.com/jixiaoyong/AndroidNote/tree/master/code/2018-02-22)

```xml
<android.support.design.widget.CoordinatorLayout>

    <android.support.design.widget.AppBarLayout>

         <android.support.design.widget.CollapsingToolbarLayout
              app:layout_scrollFlags="scroll|exitUntilCollapsed">

            <ImageView 
                  app:layout_collapseMode="parallax" />

            <android.support.v7.widget.Toolbar
                  app:layout_collapseMode="pin" />

        </android.support.design.widget.CollapsingToolbarLayout>

    </android.support.design.widget.AppBarLayout>

    <android.support.v4.view.ViewPager
            app:layout_behavior="@string/appbar_scrolling_view_behavior" />

</android.support.design.widget.CoordinatorLayout>
```

1. CoordinatorLayout在最外层，注意其直接子view必须就是要实现联动的view，否则联动失效。
2. CollapsingToolbarLayout必须设置layout_scrollFlags，其余属性可选。

> layout_scrollFlags说明如下：
>
> **scroll**：所有想滚动出屏幕的view都需要设置这个flag， 没有设置这个flag的view将被固定在屏幕顶部。
>
> **enterAlways**：这个flag让任意向下的滚动都会导致该view变为可见，启用快速“返回模式”。
>
> **enterAlwaysCollapsed**：假设你定义了一个最小高度（minHeight）同时enterAlways也定义了，那么view将在到达这个最小高度的时候开始显示，并且从这个时候开始慢慢展开，当滚动到顶部的时候展开完。
>
> **exitUntilCollapsed**：当你定义了一个minHeight，此布局将在滚动到达这个最小高度的时候折叠。
>
> **snap**：当一个滚动事件结束，如果视图是部分可见的，那么它将被滚动到收缩或展开。例如，如果视图只有底部25%显示，它将折叠。相反，如果它的底部75%可见，那么它将完全展开。
>
> 作者：尹star
>
> 链接：https://www.jianshu.com/p/5287d090e777

1. CollapsingToolbarLayout的子view需要指定layout_collapseMode，还有一点需注意：**和toolbar联动的子view高度需大于toolbar高度，否则无效果。**
2. ViewPager就是本案例中触发子view联动效果的`dependency`，需要指定其behavior：

```xml
app:layout_behavior="@string/appbar_scrolling_view_behavior"
```

其实际对应于android.support.design.widget.AppBarLayout$ScrollingViewBehavior ，这个是系统实现的一个behavior，用于和嵌套滑动事件绑定，**指定该behavior的子view需要是NestedScrollingChild的实现类**（系统提供了4个实现类：NavigationMenuView、NestedScrollView、RecyclerView、SwipleRefreshLayout），所以viewPager中页面有上述4个类或其子类时，才能实现绑定效果。

# 延伸

**自定义Behavior**

自定义Behavior有两个目的：

1. 将两个或多个子view绑定；
2. 将一个子view与另一个子view的滑动事件绑定在一起

两者的差异在于在实现`CoordinatorLayout.Behavior<T>` 类时候具体重写的方法不一样。

**目的1**：需要重写的方法有：

```java
@Override
public boolean layoutDependsOn(CoordinatorLayout parent, T child, View dependency) {
    //如果dependency是要依赖的子view（此处是TempView类）的实例，说明它就是我们所需要的Dependency
    return dependency instanceof TempView;
}

//每次dependency位置发生变化，都会执行onDependentViewChanged方法
@Override
public boolean onDependentViewChanged(CoordinatorLayout parent, T child, View dependency) {

    //根据dependency的位置，设置child的位置,对child进行想要实现的变化

    return true;//返回true表示改变了child 的尺寸和位置参数，否则返回false
}
```

**目的2**：需要重写的方法有：

```java
//判断是否要开始根据dependency子view的行为改变child的状态
@Override
public boolean onStartNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull ImageView child,@NonNull View directTargetChild, @NonNull View target, int axes, int type) {
    return child instanceof ImageView && axes == View.SCROLL_AXIS_VERTICAL;//子view是ImageView，并且滑动的方向是垂直的
}

//当dependency子view滑动时，对child进行相应处理
@Override
public void onNestedPreScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull ImageView child, @NonNull View target, int dx, int dy, @NonNull int[] consumed, int type) {
    super.onNestedPreScroll(coordinatorLayout, child, target, dx, dy, consumed, type);

}
```



> # 自定义 Behavior 的总结
>
> 1. 确定 CoordinatorLayout 中 View 与 View 之间的依赖关系，通过 layoutDependsOn() 方法，返回值为 true 则依赖，否则不依赖。
> 2. 当一个被依赖项 dependency 尺寸或者位置发生变化时，依赖方会通过 Byhavior 获取到，然后在 onDependentViewChanged 中处理。如果在这个方法中 child 尺寸或者位置发生了变化，则需要 return true。
> 3. 当 Behavior 中的 View 准备响应嵌套滑动时，它不需要通过 layoutDependsOn() 来进行依赖绑定。只需要在 onStartNestedScroll() 方法中通过返回值告知 ViewParent，它是否对嵌套滑动感兴趣。返回值为 true 时，后续的滑动事件才能被响应。
> 4. 嵌套滑动包括滑动(scroll) 和 快速滑动(fling) 两种情况。开发者根据实际情况运用就好了。
> 5. Behavior 通过 3 种方式绑定：1. xml 布局文件。2. 代码设置 layoutparam。3. 自定义 View 的注解。
>
> 来源 ：[针对 CoordinatorLayout 及 Behavior 的一次细节较真 - frank 的专栏 - CSDN博客]( http://blog.csdn.net/briblue/article/details/73076458)

# 参考文献

- [针对 CoordinatorLayout 及 Behavior 的一次细节较真 - frank 的专栏 - CSDN博客](http://blog.csdn.net/briblue/article/details/73076458)  
- [CoordinatorLayout 自定义Behavior并不难，由简到难手把手带你撸三款！ - 泡在网上的日子](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0824/6565.html)
- [一步一步深入理解CoordinatorLayout - 简书](https://www.jianshu.com/p/8c92d0a1e591)
- [使用CoordinatorLayout打造一个炫酷的详情页 - 简书](https://www.jianshu.com/p/5287d090e777)

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android实现可折叠toolbar.md</div>"></iframe>