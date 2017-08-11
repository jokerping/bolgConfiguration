---
title: iOS异常处理及signal信号
categories: iOS
tags: 基础知识
date: 2017-08-10 22:53:19

---

# 系统crash
不讲2333 什么时候想起来再填充

# singal信号
> 软中断信号（signal，又简称为信号）用来通知进程发生了异步事件。进程之间可以互相通过系统调用kill发送软中断信号。内核也可以因为内部事件而给进程发送信号，通知进程发生了某个事件。注意，信号只是用来通知某进程发生了什么事件，并不给该进程传递任何数据。

下面是一些信号说明，添加的备注是我的理解，未添加的表示没遇到过

## 1) SIGHUP
> 本信号在用户终端连接(正常或非正常)结束时发出, 通常是在终端的控制进程结束时, 通知同一session内的各个作业, 这时它们与控制终端不再关联。
登录Linux时，系统会分配给登录用户一个终端(Session)。在这个终端运行的所有程序，包括前台进程组和后台进程组，一般都属于这个 Session。当用户退出Linux登录时，前台进程组和后台有对终端输出的进程将会收到SIGHUP信号。这个信号的默认操作为终止进程，因此前台进 程组和后台有终端输出的进程就会中止。不过可以捕获这个信号，比如wget能捕获SIGHUP信号，并忽略它，这样就算退出了Linux登录， wget也 能继续下载。
此外，对于与终端脱离关系的守护进程，这个信号用于通知它重新读取配置文件。

## 2) SIGINT
> 程序终止(interrupt)信号, 在用户键入INTR字符(通常是Ctrl-C)时发出，用于通知前台进程组终止进程。

貌似手机上发不出这个信号。

<!-- more -->

## 3) SIGQUIT

> 和SIGINT类似, 但由QUIT字符(通常是Ctrl-)来控制. 进程在因收到SIGQUIT退出时会产生core文件, 在这个意义上类似于一个程序错误信号。

貌似也收不到

## 4) SIGILL
> 执行了非法指令. 通常是因为可执行文件本身出现错误, 或者试图执行数据段. 堆栈溢出时也有可能产生这个信号。

不管在任何情况下得杀死进程的信号,iOS 禁止app内kill进程，不用操心

## 5) SIGTRAP
> 由断点指令或其它trap指令产生. 由debugger使用。

release环境一定不可能是断点，所以就是trap指令了，比如调用下面方法就会发出这个信号

```objectivec
NSException
+ (void)raise:(NSExceptionName)name format:(NSString *)format, ... NS_FORMAT_FUNCTION(2,3);
+ (void)raise:(NSExceptionName)name format:(NSString *)format arguments:(va_list)argList NS_FORMAT_FUNCTION(2,0);
```

## 6) SIGABRT
> 调用abort函数生成的信号。

iOS中一般是手动调用或者数组越界。FMDB中有这个函数

## 7) SIGBUS
> 非法地址, 包括内存地址对齐(alignment)出错。比如访问一个四个字长的整数, 但其地址不是4的倍数。它与SIGSEGV的区别在于后者是由于对合法存储地址的非法访问触发的(如访问不属于自己存储空间或只读存储空间)。

比如多线程读写一个对象，代理用assign
解决此类问题需要有一定的汇编知识,[iOS汇编教程：理解ARM](http://www.jianshu.com/p/544464a5e630)介绍了常用的汇编代码。

另外[教你 Debug 的正确姿势——记一次 CoreMotion 的 Crash](https://my.oschina.net/bugly/blog/919482)手Q一个小哥写的关于SIGBUS的文章非常棒。对于5，7，11这些都可以用这篇文章中提供的方法来找到

## 8) SIGFPE
> 在发生致命的算术运算错误时发出. 不仅包括浮点运算错误, 还包括溢出及除数为0等其它所有的算术的错误。

## 9) SIGKILL
> 用来立即结束程序的运行. 本信号不能被阻塞、处理和忽略。如果管理员发现某个进程终止不了，可尝试发送这个信号。

用户或者系统删除后台进程会发送此信号，需要注意系统杀死后台的进程的情况并非只有内存使用过多，CPU占用大也会被杀死

## 10) SIGUSR1
> 没在POSIX.1中列出，而在SUSv2列出 同7 SIGBUS

## 11) SIGSEGV
> 试图访问未分配给自己的内存, 或试图往没有写权限的内存地址写数据.

比如空指针，未初始化指针，栈溢出等；

## 12) SIGUSR2
>没在POSIX.1中列出，而在SUSv2列出 无效的系统调用 (SVID) 同SIGSYS

## 13) SIGPIPE
> 管道破裂。这个信号通常在进程间通信产生，比如采用FIFO(管道)通信的两个进程，读管道没打开或者意外终止就往管道写，写进程会收到SIGPIPE信号。此外用Socket通信的两个进程，写进程在写Socket的时候，读进程已经终止。

## 14) SIGALRM
> 时钟定时信号, 计算的是实际的时间或时钟时间. alarm函数使用该信号.

剩下的基本都不会遇到

# Exception Code

有一些值代表固定的意思比如下面这些

值 | 解释
----|------
0x8badf00d | 在启动、终⽌止应⽤用或响应系统事件花费过⻓长时间,意为“ate bad food”。	<br> <br>对于此类Crash，我们应该去审视自己App初始化时做的事情是否正确，是否在主线程请求了网络，或者其他耗时的事情卡住了正常初始化流程。通常系统允许一个App从启动到可以相应用户事件的时间最多为5S，如果超过了5S，App就会被系统终止掉。在Launch，resume，suspend，quit时都会有相应的时间要求。在Highlight Thread里面我们可以看到被终止时调用到的位置，xxxAppDelegate加上行<br>
0xdeadfa11 | ⽤用户强制退出,意为“dead fall”。(系统⽆无响应时,⽤用户按电源开关和HOME)<br><br>这个强制退出跟我们平时所说的kill掉后台任务操作还不太一样，通常在程序bug造成系统无法响应时可以采用长按电源键，当屏幕出现关机确认画面时按下Home键即可关闭当前程序。<br>
0xbaaaaaad | ⽤用户按住Home键和⾳音量键,获取当前内存状态,不代表崩溃<br>
0xbad22222 | VoIP应⽤用因为恢复得太频繁导致crash<br>
0xc00010ff | 因为太烫了被干掉,意为“cool off”<br>
0xdead10cc | 因为在后台时仍然占据系统资源(⽐比如通讯录)被干掉,意为“dead lock”<br>


参考文档：
- [iOS异常捕获](http://www.iosxxx.com/blog/2015-08-29-iosyi-chang-bu-huo.html)
- [iOS Crash文件的解析](http://www.cocoachina.com/ios/20150122/10991.html)
- [iOS中常见的crash信号处理](http://www.jianshu.com/p/5c0e1768ba54)
- [hiOS疑难问题排查之深入探究dispatch_group crash](http://satanwoo.github.io/2017/01/07/DispatchGroupCrash/)
- [浅谈一种解决多线程野指针的新思路](https://satanwoo.github.io/2016/10/23/multithread-dangling-pointer/)
- [手把手教你如何分析 iOS 系统栈 crash](https://dev.qq.com/topic/59150e72142eee2b6b97359a)
- [Unix Signals](https://people.cs.pitt.edu/~alanjawi/cs449/code/shell/UnixSignals.htm)
