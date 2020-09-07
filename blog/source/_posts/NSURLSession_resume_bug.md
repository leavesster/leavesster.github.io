---
title: NSURLSession Resume Bug(BackgroundSession)
date: 2016-11-14
tags: iOS
---

复现系统环境：iOS 10~10.1 (必要条件)，后台断点下载。

这是在项目中的实际遭遇，iOS 9下断点下载功能正常运行，iOS 10下，无法继续断点下载。虽然这里就一句话，但是花了接近一个下午才排查出来。（侧面说明留一台旧系统的重要性，试了半天，换上iOS 9，才发现是iOS 10的bug。）

关键字搜索：[NSURLSession cancelByProducingResumeData](https://forums.developer.apple.com/thread/63585)

> The iOS 10 side of the fix is being tracked as 27744391.  At this point it looks like it won’t catch the 10.1 bus …
27744391 is reported as fixed in the current 10.2 beta seed (14C5062e).  If you’re having problems with:
explicit resumes failing with NSURLErrorCannotWriteToFile (-3003)
downloads failing with NSURLErrorUnsupportedURL (-1002) after a network connectivity failure
please try again on that seed and, if you still see problems, please file a bug and post the bug number here.
Share and Enjoy 

可以看到，报告在iOS 10.2 beta中修复，且该bug只在iOS 10存在，所以如果是iOS 9下也有问题，就可能不仅仅是这个bug的问题了。

---

### iOS 10.2 Beta前的修复办法

#### TD:TR
后面就简单写了点自己修复中的遭遇，TDLR,可以直接看下面给的超链接。
swift为原作者回答，OC版为我自行转换后的代码。

---

#### swift2.3与swift3.0代码
有人在上面的链接下贴了一个他的Fix方法：[resume-nsurlsession-on-ios10](http://stackoverflow.com/questions/39346231/resume-nsurlsession-on-ios10/39347461#39347461)。
作者还很贴心的贴上了swift2.3和swift3.0版本。
 
#### Objective-C代码
在项目里用的还是Objective-C，也不想因此加个桥接混编。于是下面就自己翻译了一个版本。也贴在了StackOverflow [resume-nsurlsession-on-ios10-the-objective-c-version](http://stackoverflow.com/a/40501353/4770006)。
如果有帮助，欢迎upvote。

---

#### 转换吐槽
这里因为想了解下3的写法，所以只看了swift3的代码。
swift的判断语法糖真多。
swift3的API真是大变样，swift2.3不怎么看还能认出不少来，到swift3同一个API，都大变样了。
据说swift4还要大变样，这么玩好么，python2和3还没完事呢，也亏你是Apple御用语言，才能这方便跃进。

---

#### demo大法好
自己转换了一遍swift代码，直接放在项目里面，结果跑不通，索性把原答案的代码做了一个空项目，自己转换的代码也做了一个空项目。

确定了几个点：

* URLSession必须为background才会出现断点下载问题。
* 在继续下载时报错

> 2016-xxx URLSessionbug[32176:6751444] *** -[NSKeyedUnarchiver initForReadingWithData:]: data is NULL
2016-xxx URLSessionbug[32176:6751444] *** -[NSKeyedUnarchiver initForReadingWithData:]: data is NULL
2016-xxx URLSessionbug[32176:6751444] Invalid resume data for background download. Background downloads must use http or https and must download to an accessible file

#### 查看代码后的异想天开
对比swift修复好了的代码，会发现`Invalid resume data`不再出现。

在swift的Fix代码中

```swift
let task = self.downloadTaskWithResumeData(cData)
```
追踪实现代码，可以发现第一次生成的task也是错误的，两个request属性都是nil，实际也是用setValue: forkKey:方法重新再次设置了属性的，但是他没有最后一句提示。
在此时打印两个属性，也是一样的NSURLRequest为nil。

这里我做了一件比较异想天开的事情，在downloadTask进行cancelByProducingResumeData时，直接调取task，存下两个reques属性，打算在重新生成时，用同样setValue: forKey:的方法再次设置属性。
结果是，同时跳出了三句错误代码。而不是和原答案中一样只跳两句。

setValue: forKey在我异想天开的代码和Fix方法中，都成功绕过了readOnly的限制，成功对这两个属性进行了赋值。但是Fixed方法通过修改ResumeData数据，延缓了方法报错。
就像Fix方法中说的，他全部的代码主要是在修复这个问题。
>This problem arose from currentRequest and originalRequest NSKeyArchived encoded with an unusual root of "NSKeyedArchiveRootObjectKey" instead of NSKeyedArchiveRootObjectKey constant which is "root" literally and some other misbehaves in encoding process of NSURL(Mutable)Request

---
iOS 10中由于对Request属性encode时，选择了错误的key导致了这个问题。代码里面也是因此无法恢复出Request属性，不过最后correctData并没有完全是request属性成功修复，所以要后续使用kvc赋值。

```swift
// a compensation for inability to set task requests in CFNetwork.
// While you still get -[NSKeyedUnarchiver initForReadingWithData:]: data is NULL error,
// this section will set them to real objects
```

---
后记，自己的OC代码其实是因为某个地方不小心把两个代码写成了死循环，导致无法成功修复bug。也是最后在OC版本的空项目里面发现的，发现过程：看着几个名字太类似了，于是加长方法名，于是发现问题。

用空项目解决了不少问题，越来越发现隔离项目的必要性了。但是又懒的新建项目，所以还是需要常备不少空项目demo才是。

