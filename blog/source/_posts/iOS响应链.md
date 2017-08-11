---
title: iOS事件响应链
categories: iOS
tags: 基础知识
date: 2016-08-24 09:55:36

---


# 系统如何获取到了用户进行了“单击”这一行为？
操作系统把包含这些点击事件的信息包装成UITouch和UIEvent形式的实例，然后找到当前运行的程序，逐级寻找能够响应这个事件的对象，直到没有响应者响应。这一寻找的过程，被称作事件的响应链，如下图所示，不用的响应者以链式的方式寻找

<img src="http://7xvz3k.com1.z0.glb.clouddn.com/blog_%E5%93%8D%E5%BA%94%E9%93%BE.jpg" width = "40%" />
<!--more--> 
# 响应者
在iOS中，能够响应事件的对象都是UIResponder的子类对象。UIResponder提供了四个用户点击的回调方法，分别对应用户点击开始、移动、点击结束以及取消点击，其中只有在程序强制退出或者来电时，取消点击事件才会调用。
```objectivec
// Generally, all responders which do custom touch handling should override all four of these methods.
// Your responder will receive either touchesEnded:withEvent: or touchesCancelled:withEvent: for each
// touch it is handling (those touches it received in touchesBegan:withEvent:).
// *** You must handle cancelled touches to ensure correct behavior in your application.  Failure to
// do so is very likely to lead to incorrect behavior or crashes.
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesCancelled:(nullable NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesEstimatedPropertiesUpdated:(NSSet * _Nonnull)touches NS_AVAILABLE_IOS(9_1);//9.1之后加入的3D触摸事件。

```

在自定义UIView为基类的控件时，我们可以重写这几个方法来进行点击回调。在回调中，我们可以看到方法接收两个参数，一个UITouch对象的集合，还有一个UIEvent对象。这两个参数分别代表的是点击对象和事件对象。

- 事件对象
iOS使用UIEvent表示用户交互的事件对象，在UIEvent.h文件中，我们可以看到有一个UIEventType类型的属性，这个属性表示了当前的响应事件类型。分别有多点触控、摇一摇以及远程操作（在iOS9之后新增了3DTouch事件类型）。在一个用户点击事件处理过程中，UIEvent对象是唯一的
- 点击对象
UITouch表示单个点击，其类文件中存在枚举类型UITouchPhase的属性，用来表示当前点击的状态。这些状态包括点击开始、移动、停止不动、结束和取消五个状态。每次点击发生的时候，点击对象都放在一个集合中传入UIResponder的回调方法中，我们通过集合中对象获取用户点击的位置。其中通过`- (CGPoint)locationInView:(nullable UIView *)view`获取当前点击坐标点，`- (CGPoint)previousLocationInView:(nullable UIView *)view`获取上个点击位置的坐标点。
为了确认UIView确实是通过UIResponder的点击方法响应点击事件的，我创建了UIView的类别，并重写+ (void)load方法，使用method_swizzling的方式交换点击事件的实现
```objectivec
+ (void)load
    Method origin = class_getInstanceMethod([UIView class], @selector(touchesBegan:withEvent:));
    Method custom = class_getInstanceMethod([UIView class], @selector(lxd_touchesBegan:withEvent:));
    method_exchangeImplementations(origin, custom);

    origin = class_getInstanceMethod([UIView class], @selector(touchesMoved:withEvent:));
    custom = class_getInstanceMethod([UIView class], @selector(lxd_touchesMoved:withEvent:));
    method_exchangeImplementations(origin, custom);

    origin = class_getInstanceMethod([UIView class], @selector(touchesEnded:withEvent:));
    custom = class_getInstanceMethod([UIView class], @selector(lxd_touchesEnded:withEvent:));
    method_exchangeImplementations(origin, custom);
}

- (void)lxd_touchesBegan: (NSSet<UITouch *> *)touches withEvent: (UIEvent *)event
{
    NSLog(@"%@ --- begin", self.class);
    [self lxd_touchesBegan: touches withEvent: event];
}

- (void)lxd_touchesMoved: (NSSet<UITouch *> *)touches withEvent: (UIEvent *)event
{
    NSLog(@"%@ --- move", self.class);
    [self lxd_touchesMoved: touches withEvent: event];
}

- (void)lxd_touchesEnded: (NSSet<UITouch *> *)touches withEvent: (UIEvent *)event
{
    NSLog(@"%@ --- end", self.class);
    [self lxd_touchesEnded: touches withEvent: event];
}
```
在新建的项目中，我分别创建了AView、BView、CView和DView四个UIView的子类，然后点击任意一个位置：

<img src="http://7xvz3k.com1.z0.glb.clouddn.com/blog_%E5%93%8D%E5%BA%94%E9%93%BE_2.jpeg" width = "40%" />

在我点击上图绿色视图的时候，控制台输出了下面的日志
```
CView --- begin
CView --- end
```
由此可见在我们点击UIView的时候，是通过touches相关的点击事件进行回调处理的。

除了touches回调的几个点击事件，手势UIGestureRecognizer对象也可以附加在view上，来实现其他丰富的手势事件。在view添加单击手势之后，原来的touchesEnded方法就无效了。最开始我一直认为view添加手势之后，原有的touches系列方法全部无效。但是在测试demo中，发现view添加手势之后，touchesBegan方法是有进行回调的，但是moved跟ended就没有进行回调。因此，在系统的touches事件处理中，在touchesBegan之后，应该是存在着一个调度后续事件（nextHandler）处理的方法，个人猜测事件调度的处理大致如下图示：

<img src="http://7xvz3k.com1.z0.glb.clouddn.com/blog_%E5%93%8D%E5%BA%94%E9%93%BE_3.jpeg" width = "40%" />

# 响应链传递
上面已经介绍了某个控件在接收到点击事件时的处理，那么系统是怎么通过用户点击的位置找到处理点击事件的view的呢？
在上文我们已经说过了系统通过不断查找下一个响应者来响应点击事件，而所有的可交互控件都是UIResponder直接或者间接的子类，那么我们是否可以在这个类的头文件中找到关键的属性呢？

正好存在着这么一个方法：`- (nullable UIResponder *)nextResponder`，通过方法名我们不难发现这是获取当前view的下一个响应者，那么我们重写touchesBegan方法，逐级获取下一响应者，直到没有下一个响应者位置。相关代码如下：
```objectivec
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UIResponder * next = [self nextResponder];
    NSMutableString * prefix = @"".mutableCopy;

    while (next != nil) {
        NSLog(@"%@%@", prefix, [next class]);
        [prefix appendString: @"--"];
        next = [next nextResponder];
    }    
}
```
控制台输出的所有下级事件响应者如下：
```
AView
--UIView
----ViewController
------UIWindow
--------UIApplication
----------AppDelegate
```

虽然结果非常有层次，但是从系统逐级查找响应者的角度上来说，这个输出的顺序是刚好相反的。为什么会出现这种问题呢？我们可以看到输出中存在一个ViewController类，说明UIViewController也是UIResponder的子类。但是我们可以发现，controller是一个view的管理者，即便它是响应链的成员之一，但是按照逻辑来说，控制器不应该是系统查找对象之一，通过nextResponder方法查找的这个思路是不正确的。
后来，发现在UIView的头文件中存在这么两个方法，分别返回UIView和BOOL类型的方法：
```objectivec
- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;   // recursively calls -pointInside:withEvent:. point is in the receiver's coordinate system
- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;   // default returns YES if point is in bounds
```
根据方法名，一个是根据点击坐标返回事件是否发生在本视图以内，另一个方法是返回响应点击事件的对象。通过这两个方法，我们可以猜到，系统在收到点击事件的时候通过不断遍历当前视图上的子视图的这些方法，获取下一个响应的视图。因此，继续通过method_swizzling方式修改这两个方法的实现，并且测试输出如下:

```
UIStatusBarWindow can answer 1
UIStatusBar can answer 0
UIStatusBarForegroundView can answer 0
UIStatusBarServiceItemView can answer 0
UIStatusBarDataNetworkItemView can answer 0
UIStatusBarBatteryItemView can answer 0
UIStatusBarTimeItemView can answer 0
hit view: UIStatusBar
hit view: UIStatusBarWindow
UIWindow can answer 1
UIView can answer 1
hit view: _UILayoutGuide
hit view: _UILayoutGuide
AView can answer 1
DView can answer 0
hit view: DView
BView can answer 0
hit view: BView
hit view: AView
hit view: UIView
hit view: UIWindow
......  //下面是touches方法的输出
```
最上面的UIStatusBar开头的类型大家可能没见过，但是不妨碍我们猜到这是状态栏相关的一些视图，具体可以查找苹果的文档中心（Xcode中快捷键shift+command+0打开）。从输出中不难看出系统先调用`pointInSide: WithEvent:`判断当前视图以及这些视图的子视图是否能接收这次点击事件，然后在调用`hitTest: withEvent:`依次获取处理这个事件的所有视图对象，在获取所有的可处理事件对象后，开始调用这些对象的touches回调方法

通过输出的方法调用，我们可以看到响应查找的顺序是： UIStatusBar相关的视图 -> UIWindow -> UIView -> AView -> DView -> BView（系统在事件链传递的过程中一定会遍历所有的子视图判断是否能够响应点击事件），以本文demo为例，我们可以得出事件响应链查找的图示如下：

<img src="http://7xvz3k.com1.z0.glb.clouddn.com/blog_%E5%93%8D%E5%BA%94%E9%93%BE_4.jpeg" width = "40%" />

那么在上面的查找响应者流程完成之后，系统会将本次事件中的点击转换成UITouch对象，然后将这些对象和UIEvent类型的事件对象传递给touchesBegan方法
不仅如此，从上面输出的nextResponder来看，所有的响应者都是在查找中返回可响应点击的视图。因此，我们可以推测出UIApplication对象维护着自己的一个响应者栈，当`pointInSide: withEvent:`返回yes的时候，响应者入栈。

<img src="http://7xvz3k.com1.z0.glb.clouddn.com/blog_%E5%93%8D%E5%BA%94%E9%93%BE_5.jpeg" width = "40%" />

栈顶的响应者作为最优先处理事件的对象，假设AView不处理事件，那么出栈，移交给UIView，以此下去，直到事件得到了处理或者到达AppDelegate后依旧未响应，事件被摒弃为止。通过这个机制我们也可以看到controller是响应者栈中的例外，即便没有`pointInSide: withEvent:`的方法返回可响应，controller依旧能够入栈成为UIView的下一个响应者。

# 响应链应用
既然已经知道了系统是怎么获取响应视图的流程了，那么我们可以通过重写查找事件处理者的方法来实现不规则形状点击。最常见的不规则视图就是圆形视图，在demo中我设置view的宽高为200，那么重写方法事件如下:
```objectivec
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event
{
    const CGFloat halfWidth = 100;
    CGFloat xOffset = point.x - 100;
    CGFloat yOffset = point.y - 100;
    CGFloat radius = sqrt(xOffset * xOffset + yOffset * yOffset);
    return radius <= halfWidth;
}
```

<img src="http://7xvz3k.com1.z0.glb.clouddn.com/blog_%E5%93%8D%E5%BA%94%E9%93%BE_6.gif" width = "40%" />

根据view找到对应的controller:

```objectivec
static inline UIViewController* UIViewParentController(UIView *view)

{

    UIResponder *responder=view;

    while ([responder isKindOfClass:[UIView class]])

    {

        responder=[responder nextResponder];

    }

   return (UIViewController *)responder;

}
```

