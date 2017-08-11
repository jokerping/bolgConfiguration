---
title: 点击statusBar滚回最顶部
categories: iOS
tags: 奇巧淫技
date: 2016-07-06 18:57:50

---

  iOS系统的statusBar自带滚回最顶部。就算在statusBar位置添加个view也是可以的。
但是，这些只在当前controller只有一个scrollview的时候才起作用。如果有多个scrollview，系统当时就懵逼了。
解决办法：
      1、每个页面中不需要滚动的scrollview 设置 scrollsToTop=No;只留下需要滚动回顶部的，但是，懒癌晚期的我怎么会这样做。
      2、[statck overflow](http://stackoverflow.com/questions/3753097/how-to-detect-touches-in-status-bar)上有个答案提供了一个方法
      
>在AppDelegate.m 中

```objectivec
static NSString * const kStatusBarTappedNotification = @"statusBarTappedNotification";

- (void) touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event {
    [super touchesBegan:touches withEvent:event];
    CGPoint location = [[[event allTouches] anyObject] locationInView:[self window]];
    CGRect statusBarFrame = [UIApplication sharedApplication].statusBarFrame;
    if (CGRectContainsPoint(statusBarFrame, location)) {
        [self statusBarTouchedAction];
    }
}

- (void)statusBarTouchedAction {
    [[NSNotificationCenter defaultCenter] postNotificationName:kStatusBarTappedNotification
                                                        object:nil];
}
```

<!--more--> 

>然后在viewWillAppear中

```objectivec
[[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(clickStatusBar)             
                                             name:kStatusBarTappedNotification
                                           object:nil];                                           
```

>viewDidDisappear中

```objectivec
[[NSNotificationCenter defaultCenter] removeObserver:self name:kStatusBarTappedNotification object:nil];
```


>通知方法clickStatusBar

```objectivec
- (void)clickStatusBar
{
    if ([self isKindOfClass:[UITableViewController class]])
    {
        UITableViewController* tvc=(UITableViewController*)self;
        [tvc.tableView scrollRectToVisible:CGRectMake(0, 0, 1, 1) animated:YES];
        
    }
    for ( id obj in self.view.subviews)
    {
        if ([obj isKindOfClass:[UITableView class]])
        {
             [obj scrollRectToVisible:CGRectMake(0, 0, 1, 1) animated:YES];

        }
    }
}
```

  前面说过，我是懒癌晚期啊。每个页面加能累死我。 所以，新建一个UIViewController的类别然后重写+load方法，最后利用Method swizzing Hook掉viewWillAppear和viewDidDisappear方法即可。
  关于如何使用Method swizzing 可以参考[这篇文章](https://blog.leichunfeng.com/blog/2015/06/14/objective-c-method-swizzling-best-practice/)
