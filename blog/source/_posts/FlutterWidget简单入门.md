---
title: Flutter Widget简单入门
tags: flutter
abbrlink: e35c106d
date: 2018-10-21 22:26:41
---
Flutter是Google提出的跨平台开发框架，使用Dart语言，支持Android，IOS系统。Flutter一个重要的概念即是——*“万物皆控件（Widget）”*，像`Padding`,`Center`等都是Widget。

Widget和Android中的View很相似但又有不同，Widget一旦生成便“一成不变”，直到下一次因为Widget更改或者state更新而被重新创建（Flutter’s framework creates a new tree of widget instances.），而View则只会被`drawn`一次，直到`invalidate`方法被调用。

本文主要记录一下Flutter中两个重要的控件：StatelessWidget和StatefulWidget，以及Flutter开发的一些基础知识。

# Flutter基础知识

Flutter以Dart开发，其工程基本的结构如下：

* android
* ios
* lib
  * main.dart
* pubspec.yaml //Flutter工程的配置信息

Flutter项目启动后会首先加载`/lib/main.dart`中的`main()`方法。
一个标准的material app的main.dart内容如下：

```dart
import 'package:flutter/material.dart';
import './product_manager.dart';

main() => runApp(MyApp());//在main()方法中调用了material的runApp()方法，里面传入了要展示的Widget——APP的界面，相当于Android的setContentView()

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(
        primarySwatch: Colors.deepOrange
      ),
      home: Scaffold(//脚手架，一个预制的APP界面结构，也可以使用自定义Widget
        appBar: AppBar(
          title: Text("EasyList"),
        ),
        body: ProductManager("Test"),//这里是自定义的控件，布局信息主要在这里展示
      ),
    );
  }
}

```

# StatelessWidget & StatefulWidget

StatelessWidget和StateFulWidget区别在于：前者一旦创建，状态便不会再更改，而后者则可以动态改变State从而使flutter改变其状态。但是两者都会在每一帧被rebuild。

## StatelessWidget

> A `StatelessWidget` is just what it sounds like—a widget with no state information.

StatelessWidget一旦创建便不会更改，其状态只和构造函数中的参数有关。下面是一个StatelessWidget示例，一般只需要重写其build()方法，返回要展示的控件即可：

```dart
class MyWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return CustomerWidget();//在这里构建一个页面并返回
  }
}
```

## StatefulWidget

> `StatefulWidget` has a `State` object that stores state data across frames and restores it.

StatefulWidget可以通过动态更改其包含的State，从而使flutter在下一次更新界面时依据state更新StateWidget，*本质上还是更新了一个可以在多帧之间存活的State，在下一帧更新控件*。

下面是一个StatefulWidget的示例:

```dart
class ProductManager extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return ProductManagerState();
  }
}

class ProductManagerState extends State<ProductManager> {
  @override
  Widget build(BuildContext context) {
    return CustomerWidget();
  }
}
```

可以注意到StatefulWidget重写了`createState()`，而该方法返回了自定义的`ProductManagerState`类对象，在该类中`build()`方法实现和StatelessWidget中的方法类似，返回要展示的页面控件。



两者的不同之处在于，StatefulWidget中可以调用`setState()`，更改其相应的`state`，以便告诉flutter在下一次rebuild的时候更新UI。

StatelessWidget要想实现动态更新其内容，可以在其外部包裹一层StatefulWidget，通过StatefulWidget更改状态state，将更改后的state传给StatelessWidget，从而间接更新了StatelessWidget的状态。

可以通过对该方法就行包装，使得在StatelessWidget控件中调用StatefulWidget控件的`setState()`方法，达到刷新页面的效果：

```dart
// StatefulWidget
  void aFun(){
    setState(() {
      // update UI
    });
  }
AStatelessWidget(aFun);// 将该方法传入StatelessWidget中
// StatelessWidget
final Function aFun
AStatelessWidget(this.aFun);// 接收传入的方法
aFun();// 执行该方法，从而实现调用StatelessWidget中的方法也可以刷新UI
```



# 与Android的对比

## Intent

Android的Intent有两个主要作用：

* Activity间跳转
* 组件间传递数据

Flutter对此相应：

* 使用Navigator和`Route`s实现在同一个“Activity”中不同的界面间（ “screen” or “page”）跳转（push，pop），Navigator类似于Android中的Activity栈。
* 通过Android原生Intent组件获取到其他App传来的数据，然后中通过下面的方法实现Android和Flutter交互：

示例代码：

```dart
 //Android
 MethodChannel(getFlutterView(), "app.channel.shared.data")
      .setMethodCallHandler(MethodChannel.MethodCallHandler() {
        @Override
        public void onMethodCall(MethodCall methodCall, MethodChannel.Result result) {
          if (methodCall.method.contentEquals("getSharedText")) {
            result.success(sharedText);
            sharedText = null;
          }
        }
      });
 //Flutter
 class _SampleAppPageState extends State<SampleAppPage> {
  static const platform = const MethodChannel('app.channel.shared.data');
  String dataShared = "No data";

  @override
  void initState() {
    super.initState();
    getSharedText();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(body: Center(child: Text(dataShared)));
  }

  getSharedText() async {
    var sharedData = await platform.invokeMethod("getSharedText");
    if (sharedData != null) {
      setState(() {
        dataShared = sharedData;
      });
    }
  }
}
```



## 线程

Flutter是单线程的，他的线程和Android的UI线程绑定，在进行网络请求，IO操作等时，可以使用`sync/await` 在执行完耗时操作后，再去更新state刷新UI。

> Since Flutter is single threaded and runs an event loop (like Node.js), you don’t have to worry about thread management or spawning background threads. If you’re doing I/O-bound work, such as disk access or a network call, then you can safely use async/await and you’re all set. 

```dart
loadData() async {
  String dataURL = "https://jsonplaceholder.typicode.com/posts";
  http.Response response = await http.get(dataURL);
  setState(() {
    widgets = json.decode(response.body);
  });
}
```

而如果有特别频繁的cpu计算以至于能导致UI挂起，可以考虑使用`Isolate`s利用CPU多核心处理任务，但是这样就不能和主线程共享数据，通过`ReceivePort`，`SendPort`等传递数据。

> Isolates(隔离) are separate execution threads that do not share any memory with the main execution memory heap. This means you can’t access variables from the main thread, or update your UI by calling `setState()`. Unlike Android threads, Isolates are true to their name, and cannot share memory (in the form of static fields, for example).

## 本地资源

截止Flutter beta 2 仍然不能直接访问Android assets或者其他本地资源，但是Android可以访问flutter的assets资源：

```dart
val flutterAssetStream = assetManager.open("flutter_assets/assets/my_flutter_asset.png")
```

通过Channel，flutter可以间接访问Android资源，反之亦然。

> 主要是通过Channel完成，可以称之为隧道。主要是MethodChannel和MessageChannel两种，第一种是调用方法，第二种是传递信息。首先通信的双方是Flutter和本地操作系统或者应用，而且方法的调用和消息的方法可以从任何一方发起，类似RPC（远程过程调用）。
>
> 作者：黄马
>
> 链接：掘金  https://juejin.im/post/5b35a75e51882574ea3a25e3
>
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

## 生命周期

Flutter生命周期没有Android中那么“重要”，可以重写 `didChangeAppLifecycleState()` 监听。

* `inactive` — 应用处于非活动状态，不接受输入。iOS
* `paused` — 应用在后台运行，不可见，不接受输入。类似Android的`onPause()`
* `resumed` — 应用可见，并接受输入。类似Android的`onPostResume()`
* `suspending` — 应用请求暂停。类似Android的`onStop()`

## 布局

Flutter有布局Widget如：

* Column 列
* Row 行
* Stack 左上角堆积，类似FrameLayout

## 点击事件

FLutter中的“`onClick()`”: `onPressed`,`onTap`等等。

添加点击事件,在Widget外面添加一个`GestureDetector`Widget：

```dart
GestureDetector(
  child: Padding(
      padding: EdgeInsets.all(10.0),
      child: Text("Row $i")),
  onTap: () {
    print('row tapped');
  },
);
```



# 参考链接

[Flutter Tutorial for Beginners - Build iOS and Android Apps with Google's Flutter & Dart](https://www.youtube.com/watch?v=GLSG_Wh_YWc)

[Flutter for android](https://flutter.io/flutter-for-android/)

[Flutter 访问本地资源](https://juejin.im/post/5b35a75e51882574ea3a25e3)



<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/FlutterWidget简单入门.md</div>"></iframe>