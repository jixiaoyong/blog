---
title: OKHttpUtils分析
date: 2019年03月17日13:32:21
tag: android
---

# 前言

本文是对张鸿洋的OKHttp辅助类[**okhttputils**](https://github.com/hongyangAndroid/okhttputils)简要分析，以便学习如何封装常见工具的思想。

主要涉及类：

* OkHttpUtils
* OkHttpRequestBuilder
* OkHttpRequest
* RequestCall
* Callback

# 基础

[OkHttp](https://github.com/square/okhttp)是可以用于Android和Java的Http工具，经典的使用分为3步：

```java
//1. 创建一个OkHttpClient客户端，在这里配置网络超时等全局配置
OkHttpClient okHttpClient = new OkHttpClient();

//2. 创建一个网络请求，每个Http访问对应一个Request，详细配置了访问的URL，类型，参数等信息
Request request = new Request
        .Builder()
        .url("https://www.baidu.com")
        .build();

//3. 使用OkHttpClient客户端执行该网络请求，分为阻塞和异步两种方式，异步会有对应回调
okHttpClient.newCall(request)
        .enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
```

虽然整体的逻辑已经很简单了，但是在实际使用的时候，不可能对每个网络请求都写一次上述代码，所以就需要对齐进行必要的封装以简化网络请求流程。

okhttputils就做到了这一点，并且将上述第二步常见网络请求的过程也加入链式调用中，使用起来更加连贯：

```java
//1. 全局配置唯一的OkHttpClient
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .connectTimeout(10000L, TimeUnit.MILLISECONDS)
        .readTimeout(10000L, TimeUnit.MILLISECONDS)
        .build();
OkHttpUtils.initClient(okHttpClient);

//2.在需要网络请求的时候，执行对应代码
OkHttpUtils.get()
                .url("http://www.baidu.com")
                .build()
                .execute(new com.zhy.http.okhttp.callback.Callback() {
                  //回调方法
                });
```

其中`9~10`行相当于OKHttp步骤2创建网络请求，`11~14`则就是步骤3执行网络请求的过程。

每次使用网络请求时只需要选择`get`、`post`等方法获取并配置相应`builder`，然后选择`execute`执行即可。

# 实现分析

那么okhttputils是如何实现这一点的呢？

首先看看`OkHttpUtils`的结构：

![](https://jixiaoyong.github.io/images/20190317141417.png)

可以看到大体上可以将其分为3个部分：

1. OkHttpClient相关
2. 网络请求相关信息
3. 与具体执行网络请求有关的方法

## OkHttpClient相关

我们先来看第一部分，OkHttpUtils本质上只是对OkHttpClient的方法进行了一次封装，所以其肯定要持有OkHttpClient对象，一般来说一个APP只需要一个OkHttpClient对象即可，所以可以看到OkHttpUtils做了双重锁定的单例处理：

```java
public static OkHttpUtils initClient(OkHttpClient okHttpClient)
{
    if (mInstance == null)
    {
        synchronized (OkHttpUtils.class)
        {
            if (mInstance == null)
            {
                mInstance = new OkHttpUtils(okHttpClient);
            }
        }
    }
    return mInstance;
}
```

这样我们在第一次使用`OkHttpUtils`的时候初始化的`OkHttpClient`便会被保存到这里，之后的使用中就不需要再去反复创建了。

此外在`OkHttpUtils`的结构中可以注意到有一个`mPlatform`的变量，他会根据Android还是其他平台的不同被初始化为Android主线程或者普通线程池，这个我们在后面回调网络请求状态的时候会用到。

```java
private Platform mPlatform = findPlatform();

private static Platform findPlatform()
    {
        try
        {
            Class.forName("android.os.Build");
            if (Build.VERSION.SDK_INT != 0)
            {
                return new Android();
            }
        } catch (ClassNotFoundException ignored)
        {
        }
        return new Platform();
   }
```

## 网络请求相关信息

有了`OkHttpClient`对象之后，下一步便是创建一个适当的网络请求。

在`OkHttpUtils`中使用的是`OkHttpRequestBuilder <T extends OkHttpRequestBuilder>`的子类来收集、配置相关的一些属性。

在该类中，定义了一系列网络请求基本的参数：

```java
protected String url;
protected Object tag;
protected Map<String, String> headers;
protected Map<String, String> params;
protected int id;
```

此外还有一个抽象方法，用来创建执行网络请求的`RequestCall`。

```java
public abstract RequestCall build();
```

这个方法的实现一般是调用`OkHttpRequest`子类的`build`方法，可以看到`OkHttpRequestBuilder`也只是将网络请求的相关参数传递到`OkHttpRequest`中。

```java
//com.zhy.http.okhttp.builder.GetBuilder
@Override
public RequestCall build()
{
    if (params != null)
    {
        url = appendParams(url, params);
    }

    return new GetRequest(url, tag, params, headers,id).build();
}
```

在`OkHttpRequest`中，利用上述的参数可以并通过`generateRequest(Callback callback)`方法创建`Request`。

```java
//com.zhy.http.okhttp.request.OkHttpRequest
public Request generateRequest(Callback callback)
{
    RequestBody requestBody = buildRequestBody();
    RequestBody wrappedRequestBody = wrapRequestBody(requestBody, callback);//用于更新下载进度等，为okhttp3.Callback增加更多功能
    Request request = buildRequest(wrappedRequestBody);
    return request;
}
```

> `Callback`是在`okhttp3.Callback`的基础上增加了before，progress和对请求结果的处理等的回调。

`OkHttpRequest`类的`build`方法则只是将其自身传递给`okhttp3.Call`的封装类`RequestCall`：

```java
//com.zhy.http.okhttp.request.OkHttpRequest
public RequestCall build()
{
    return new RequestCall(this);
}
```

## 执行网络请求

当执行`RequestCall`的`execute`方法时：

```java
//com.zhy.http.okhttp.request.RequestCall
public void execute(Callback callback)
{
    buildCall(callback);//创建okhttp3.Call对象，其所用的Request对象来自于okHttpRequest.generateRequest(callback)

    if (callback != null)
    {
        callback.onBefore(request, getOkHttpRequest().getId());
    }

    OkHttpUtils.getInstance().execute(this, callback);
}
```

可以看其最后只是将`RequestCall`和`callback`传递给了`OkHttpUtils`类的`execute`方法，在这里执行了真正的网络请求：

```java
//com.zhy.http.okhttp.OkHttpUtils
public void execute(final RequestCall requestCall, Callback callback)
{
    if (callback == null)
        callback = Callback.CALLBACK_DEFAULT;
    final Callback finalCallback = callback;
    final int id = requestCall.getOkHttpRequest().getId();

    requestCall.getCall().enqueue(new okhttp3.Callback()
    {
        @Override
        public void onFailure(Call call, final IOException e)
        {
            sendFailResultCallback(call, e, finalCallback, id);
        }

        @Override
        public void onResponse(final Call call, final Response response)
        {
            try
            {
                if (call.isCanceled())
                {
                    sendFailResultCallback(call, new IOException("Canceled!"), finalCallback, id);
                    return;
                }

                if (!finalCallback.validateReponse(response, id))
                {
                    sendFailResultCallback(call, new IOException("request failed , reponse's code is : " + response.code()), finalCallback, id);
                    return;
                }

                Object o = finalCallback.parseNetworkResponse(response, id);
                sendSuccessResultCallback(o, finalCallback, id);
            } catch (Exception e)
            {
                sendFailResultCallback(call, e, finalCallback, id);
            } finally
            {
                if (response.body() != null)
                    response.body().close();
            }

        }
    });
}
```

而网络请求的回调，则是在本文最开始的`mPlatform`提供的线程中进行。这样保证了在Android中，`onBefore`、`onAfter`、`inProgress`等回调能够在UI线程进行。

```java
public void sendSuccessResultCallback(final Object object, final Callback callback, final int id)
{
    if (callback == null) return;
    mPlatform.execute(new Runnable()
    {
        @Override
        public void run()
        {
            callback.onResponse(object, id);
            callback.onAfter(id);
        }
    });
}
```