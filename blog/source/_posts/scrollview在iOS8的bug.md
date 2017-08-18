---
title: scrollview在iOS8的bug
categories: iOS
tags: 基础知识
date: 2017-08-18 13:55:17

---

# 问题
iOS9之前版本的scrollview在开启弹簧效果的时候，滑动到最底部或者顶部触发弹簧动画，此时退出控制器，会触发crash。

# 原因
> The UIScrollView on stack frame #1 probably wants to inform its delegate about the animation ending, but the delegate is gone at that point. Setting NSZombieEnabled would probably confirm this.
Delegates are not retained, so this is a common error in Cocoa and Cocoa Touch. Look for delegates on UIScrollView or UITableView in your code and try to find out which one might be released before its time.

> 在iOS8.x的系统上delegate并不是weak属性, 而是__unsafe_unretained.

# 解决方案

既然是因为scrollview被释放而代理还在，引发的bug。那么hook dealloc释放delegate即可。
首先为所有的object添加一个释放时触发的block

<!-- more -->

```objectivec
.h
/**
 @brief 添加一个block,当该对象释放时被调用
 **/
- (void)guard_addDeallocBlock:(void(^)(void))block;

.m
@interface Parasite : NSObject
@property (nonatomic, copy) void(^deallocBlock)(void);
@end
@implementation Parasite
- (void)dealloc
{
    !_deallocBlock ?: _deallocBlock();
}
@end

@implementation NSObject (Guard)

- (void)guard_addDeallocBlock:(void(^)(void))block {
    @synchronized (self) {
        static NSString *kAssociatedKey = nil;
        NSMutableArray *parasiteList = objc_getAssociatedObject(self, &kAssociatedKey);
        if (!parasiteList)
        {
            parasiteList = [NSMutableArray new];
            objc_setAssociatedObject(self, &kAssociatedKey, parasiteList, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        }
        Parasite *parasite = [Parasite new];
        parasite.deallocBlock = block;
        [parasiteList addObject: parasite];
    }
}

@end
```

然后将在scrollview分类中hook delegate在dealloc时释放

```
@implementation UIScrollView (safeDelegate)
+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        SEL oldSEL = @selector(setDelegate:);
        SEL newSEL = @selector(myselfSetDelegate:);
        Method oldMethod = class_getInstanceMethod(class, oldSEL);
        Method newMethod = class_getInstanceMethod(class, newSEL);
        BOOL success = class_addMethod(class, oldSEL, method_getImplementation(newMethod), method_getTypeEncoding(newMethod));
        if (success)
        {
            class_replaceMethod(class, newSEL, method_getImplementation(oldMethod), method_getTypeEncoding(oldMethod));
        }
        else
        {
            method_exchangeImplementations(oldMethod, newMethod);
        }
    });
}

- (void)myselfSetDelegate:(id )delegate
{
    if (delegate) {
        __weak typeof(UIScrollView *)weak_self = self;
        [delegate guard_addDeallocBlock:^{
            weak_self.delegate = nil;
            if ([weak_self isKindOfClass:[UITableView class]])
            {
                ((UITableView *)weak_self).editing = NO;
                ((UITableView *)weak_self).dataSource = nil;
                ((UITableView *)weak_self).delegate = nil;

            }
            else if ([weak_self isKindOfClass:[UICollectionView class]])
            {
                ((UICollectionView *)weak_self).dataSource = nil;
                ((UICollectionView *)weak_self).delegate = nil;
            }
        }];
    }

    [self myselfSetDelegate:delegate];
}
```


参考文档：
- [UIScrollView EXC_BAD_ACCESS crash in iOS SDK](https://stackoverflow.com/questions/3686803/uiscrollview-exc-bad-access-crash-in-ios-sdk)
- [为对象添加一个释放时触发的block](http://www.tanhao.me/pieces/160626.html/)
