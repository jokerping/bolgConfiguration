---
title: iOS8中那些我可能用到的API
categories: iOS
tags: 基础知识
date: 2017-02-16 13:42:32

---

公司新的项目是以iOS8为最低支持版本的，所以写一份备忘录记录一下那些iOS8中更好用且常用的API

    let yosemite = NSOperatingSystemVersion(majorVersion: 10, minorVersion: 10, patchVersion: 0)
NSProcessInfo().isOperatingSystemAtLeastVersion(yosemite) // false
新的判断版本的方法

    localizedCaseInsensitiveContainsString
不区分大小写匹配字符串是否包含相比以前rangOfstring要简单

    wkwebview
iOS8可以考虑开始使用wkwebView来代替UIWebview

    NSQualityOfService
    
> 线程这个概念已经在苹果的框架中被系统性的忽略。这对于开发者而言是件好事。
沿着这个趋势，NSOperation中新的qualityOfService的属性取代了原来的threadPriority。通过它可以推迟那些不重要的任务，从而让用户体验更加流畅。
NSQualityOfService枚举定义了以下值：
UserInteractive：和图形处理相关的任务，比如滚动和动画。
UserInitiated：用户请求的任务，但是不需要精确到毫秒级。例如，如果用户请求打开电子邮件App来查看邮件。
Utility：周期性的用户请求任务。比如，电子邮件App可能被设置成每五分钟自动检查新邮件。但是在系统资源极度匮乏的时候，将这个周期性的任务推迟几分钟也没有大碍。
Background：后台任务，用户可能并不会察觉对这些任务。比如，电子邮件App对邮件进行引索以方便搜索。
Quality of Service将在iOS 8和OS X Yosemite中广泛的应用，所以留意所有能利用它们的机会。

	 LocalAuthentication
TouchID的类，指纹解锁还是相当方便的 配合keychian 一个指纹处处登陆。

[iOS7到iOS8变化的API](https://developer.apple.com/library/content/releasenotes/General/iOS80APIDiffs/index.html#//apple_ref/doc/uid/TP40014455)

