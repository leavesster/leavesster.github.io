---
title: WKWebview的一些注意点
date: 2018-01-04
tags: iOS
---

## 注意点

### 循环引用陷阱

由于`WKWebview.configuration.userContentController`会强引用`ScriptMessageHandler`，如果不小心就会有`retainCycle`.

相关资料：
    1. [wkwebview-causes-my-view-controller-to-leak](https://stackoverflow.com/questions/26383031/wkwebview-causes-my-view-controller-to-leak)
    1. [memory-leak-when-using-wkscriptmessagehandler](https://stackoverflow.com/questions/31094110/memory-leak-when-using-wkscriptmessagehandler)
### `loadHTMLString: baseURL:`方法中`baseURL`有权限限制。

WKWebview使用`loadHTMLString: baseURL:`读取文件时，`baseURL`有权限限制。

1. 在application文件夹：不能加载baseURL上一级目录的文件。（存疑，忘了具体情况）
1. 在temp文件夹：baseURL为temp的子目录，则可以读取temp文件资源（在iOS 8中，BaseURL在temp子文件夹中，仍然不能读取temp文件夹内容，项目中放弃iOS 8支持）
    
[测试demo](https://github.com/shazron/WKWebViewFIleUrlTest)试验读取上一级目录，需要修改源码。
> 加载本地server的方法不可持续。当长时间运行（系统锁屏较长时间）后，该server存在被系统主动关闭的可能性。

### `loadHTMLString: baseURL:`锚点问题

仅iOS 11，第一次使用`loadHTMLString: baseURL:`方法加载的网页。锚点(#tag) 一开始会失效，第二次打开页面，就有效了。

相关连接：[wkwebview-anchor-tags-within-same-page-not-woking](https://stackoverflow.com/questions/46554163/wkwebview-anchor-tags-within-same-page-not-woking)

* 解决办法：

1. 生成WKWebview时，手动执行一次loadRequest，加载一个空白页面。
2. 此时有一个问题，loadRequest时，有时候会多次回调`WKNavigationDelegate`需要手动阻止第二次加载空白页，否则会有一些问题。因为这个loadRequest也会正常回调success之类的回调。

### 禁用 MenuItem

* iOS 11: 
    只需要简单的在WKWebview内部重写`- (BOOL)canPerformAction:(SEL)action withSender:(nullable id)sender`返回NO即可。
* iOS 9-10:
    还需要对持有Webview的ViewController进行重写。见https://github.com/dwieringa/WKWebViewEditMenuHidingTest/pull/1 。而且为了不影响其他Controller，该行为只有在当前ViewController可见（push和present），且不需要弹出键盘，才重写返回上述值，否则会造成键盘无法弹出。
### 在JS中无法加载跨域的CSS。

不要在JS中用CSSRule去读取跨域的CSS

### JS回调时机不同

与UIWebview相比，js回调后的时机不同，JS执行完后，DOM的渲染不一定完成，导致和UIWebview可能有略微情况不同

### js回调的result类型为id

返回值根据js代码返回值不同，会有不同类型。目前看到有，Number，String，应该是 JSON 中可以表达的基础基础类型。

### 修改WKWebview的URL，仅在URL上添加`#fragment`部分，也会增加backlist。

### 如果仍然适配iOS10及其以下的系统，WKWebview无法直接嵌入到xib中。
    
放一个空的`View`，在代码中在进行替换。如果原来是`UIWebview`，可以考虑使用文本编辑器，将XML中的把`UIWebview`改成`UIView`。

## 相关资料

以下链接为替换`WKWebview`控件时，主要的参考内容。`WKWebview`大多数注意点，在这些文章中已经被提及。
本文是在替换`WKWebview`过程中，对一些特殊需求和遭遇的补充，所以内容并不重叠。尤其是前两篇，推荐阅读。

[WKWebView 那些坑](https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA)  
[js(javascript)与ios(Objective-C)相互通信交互](http://www.skyfox.org/javascript-ios-navive-message.html)  
[IMYWebLoader](https://github.com/li6185377/IMYWebLoader)  
[WKWebView的使用和各种坑的解决方法（OC＋Swift）](http://www.jianshu.com/p/403853b63537)
