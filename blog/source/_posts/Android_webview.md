---
title: Android WebView 资源拦截
date: 2020-07-08
tags: Android
---

## 参考资料

[WebViewClient#shouldInterceptRequest](https://developer.android.com/reference/android/webkit/WebViewClient#shouldInterceptRequest(android.webkit.WebView,%20android.webkit.WebResourceRequest))
[Handle Range Requests for proper media file handling](https://github.com/ionic-team/cordova-plugin-ionic-webview/pull/298/files)

## 拦截原理

Android 在这方面比 iOS 开放一些，Android 原生就可以对 webview 的请求进行拦截，完全不需要网页端自定义 scheme，直接可以拦截 https 和 http 请求，并且对于 POST 请求，还有比较优雅的回落策略。（Android👍，Apple💊）

我们只需要对实现 WebViewClient 协议，然后通过将其设置为 WebView 的 webViewClient 属性即可接管网络请求。

## 处理过程

1. 实现 WebViewClient 协议，主要在 shouldInterceptRequest 方法中。

>有两个同名方法，其中一个只提供了 string，另一个则是完整的 WebResourceRequest 类。一般都使用后者，后者需要 Android API 21（Android 5.0)，不建议对在此之前的版本进行拦截，因为你无法知道是什么 HTTP 请求方法。

```kotlin
override fun shouldInterceptRequest(
        view: WebView?,
        request: WebResourceRequest?
    ): WebResourceResponse? {
        ...
    }
```

2. 赋值 webview 的 webViewClient 属性

```kotlin
# ...初始化 webViewClient
webView.webViewClinet = webViewClient
```

### 细节

原理而言，就这么一点内容，但是实际上有一些注意事项：

1. 无法获取 post 请求的 body 部分，所以最好不要拦截 post 请求，直接在`shouldInterceptRequest`返回 null 即可。我这里主要为了做资源拦截，所以不存在需要拦截 POST 请求的情况。
1. 需要自己伪造跨域请求头。一般而言，Android 的 webview 是本地网络，而请求的资源都是网络请求，所以需要自己构造跨域头。
    * 也可以选择使用 webview 的 WebSetting 方法 [setAllowUniversalAccessFromFileURLs](https://developer.android.com/reference/android/webkit/WebSettings#setAllowUniversalAccessFromFileURLs(boolean))来绕过该问题。（Android API 30 该方法已弃用）
    * iOS 伪造跨域头失败，只能使用私有方法，设置`allowUniversalAccessFromFileURLs`(让我们再说一遍 apple💊)
1. 带 Range 的请求，也需要手动构造响应头（这一点不如 Apple，Apple 什么都给你做了，但是 Apple 也束缚了手脚，不能多做一些。所以 Android 的这个处理非常 Android）

### 伪造跨域头

>在 native 端的网络请求，不存在跨域问题。
>跨域请求是浏览器(包含移动端浏览器)特有的内容。具体可以查看[CORS](http://mdn.io/cors)

```kotlin
@RequiresApi(Build.VERSION_CODES.LOLLIPOP)
private fun normalResponse(request: WebResourceRequest): WebResourceResponse? {
    var mime = MimeTypeMap.getSingleton().getMimeTypeFromExtension(MimeTypeMap.getFileExtensionFromUrl(request.url.toString()))
    var file = File("${context.cacheDir.absolutePath}/${cachedPath(request.url.toString())}")
    if (!file.exists()) {
        Log.i(tag,"file ${file.absolutePath} not exist")
        return null
    }
    val tempResponseHeaders: MutableMap<String, String> = HashMap();

    // CORS
    tempResponseHeaders["Access-Control-Allow-Origin"] = "*"
    tempResponseHeaders["Access-Control-Allow-Methods"] = "POST, GET, OPTIONS"
    tempResponseHeaders["Access-Control-Allow-Headers"] = "Content-Type"

    var inputStream = FileInputStream(file)
    return WebResourceResponse(mime, "UTF-8", 200, "ok", tempResponseHeaders, inputStream)
}
```

### 伪造 Range 响应头

>Range 响应头是网络请求内容，主要是用于处理音视频等大文件时，不需要一次性下载全部内容，增加出来的内容。在对在线音视频进行抓包时，常常可以看到该响应头，此时 http code 为 `206 Partial Content`。具体可以查看 [Range](mdn.io/Range)

```kotlin
// https://github.com/ionic-team/cordova-plugin-ionic-webview/pull/298/files
@RequiresApi(Build.VERSION_CODES.LOLLIPOP)
private fun mediaResponse(request: WebResourceRequest): WebResourceResponse? {
    var mime = MimeTypeMap.getSingleton().getMimeTypeFromExtension(MimeTypeMap.getFileExtensionFromUrl(request.url.toString()))
    var inputStream = if (CommonVariables.useLocalMedia) {
        context.assets.open("media.mp4")
    } else {
        var media = File("${context.cacheDir.absolutePath}/${cachedPath(request.url.toString())}")
        if (!media.exists()) {
            Log.i(tag,"file ${media.absolutePath} not exist")
            return null
        }
        FileInputStream(media)
    }

    val tempResponseHeaders: MutableMap<String, String> = HashMap();
    try {
        val totalRange: Int = inputStream.available()
        val rangeString = request.requestHeaders["Range"]
        val parts =
            rangeString!!.split("=".toRegex()).toTypedArray()
        val streamParts =
            parts[1].split("-".toRegex()).toTypedArray()
        val fromRange = streamParts[0]
        var range = totalRange - 1
        if (streamParts.size > 1 && streamParts[1] != "") {
            range = streamParts[1].toInt()
        }
        tempResponseHeaders["Accept-Ranges"] = "bytes"
        tempResponseHeaders["Content-Range"] = "bytes $fromRange-$range/$totalRange"
    } catch (e: IOException) {
        e.printStackTrace()
        return null
    }

    Log.i(tag, "request hit $tempResponseHeaders")
    return WebResourceResponse(mime, "UTF-8", 206, "ok", tempResponseHeaders, inputStream)
}

```

## Demo 源码

[demo 源码](https://github.com/leavesster/Android_webView_intercept)