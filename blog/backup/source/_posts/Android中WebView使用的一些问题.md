---
title: Android中WebView使用的一些问题
date: 2018-02-08 00:11:13
tags: android
---



# 问题描述：WebView在fragment中不显示

解决代码如下：

```kotlin
//kotlin 代码
webView.webViewClient = object : WebViewClient() {
            override fun shouldOverrideUrlLoading(view: WebView?, url: String?): Boolean {
                view!!.loadUrl(url)
                return true
            }
        }
```

此代码同样强制在webview中打开对应的网页

