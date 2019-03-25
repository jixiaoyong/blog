---
title: Android中WebView使用的一些问题
tags: android
abbrlink: b35fe0f7
date: 2018-02-08 00:11:13
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

<script src="https://jixiaoyong.github.io/js/edit_on_github.js"></script>
<iframe id="iframeid" scrolling=false height="50" frameborder="no" border="0" marginwidth="0" marginheight="0" onload="Javascript:editOnGithub()" srcdoc="<div id=&quot;url&quot;>https://github.com/jixiaoyong/jixiaoyong.github.io/blob/hexo_blog/blog/source/_posts/Android中WebView使用的一些问题.md</div>"></iframe>
