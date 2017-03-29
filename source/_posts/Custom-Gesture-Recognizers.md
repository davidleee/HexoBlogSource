---
title: Gesture Recognizers for iOS - The Customization
date: 2017-03-29 15:49:15
tags:
- iOS
- Gesture
---

总觉得看英文文档并做笔记，写着写着就会变成原文翻译...不过这也算是对文档中内容的一种学习和吸收吧。
上一篇文章主要讲的是系统自带的手势识别器的一些使用方法，这篇文章将会把注意力放在自定义的手势处理上。

<!-- more -->

## 面向对象
当一次*触摸*发生时，会根据触摸手势的不同而产生相应个数的 UITouch 对象，而一个*事件*是一系列多指触摸事件的汇总。在 iOS 中，触摸用 [UITouch](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UITouch_Class/index.html#//apple_ref/occ/cl/UITouch) 对象来表示，触摸事件用`UIEventTypeTouches`类型的 [UIEvent](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIEvent_Class/index.html#//apple_ref/doc/uid/TP40006780-CH3-SW12) 对象来表示。

### UITouch
每一个触摸对象只对应一只手指的触摸事件，从手指放到屏幕上到手指抬起，持续的捕获这个对象的相关属性，包括：触摸状态、触摸位置、上一个触摸位置和时间戳。

> 手指触摸的精度可远比不上鼠标。当触摸事件发生时，触摸的区域其实是个椭圆形，而且触摸的位置比用户认为的会略低一些。这些区别的产生会因为手指的大小和方向不同而不同，另外还可能受到按压力度、使用不同手指和一些其他因素的影响。
> 不过这些问题已经被底层的多点触控系统处理好了，在上层看来，用户就是触摸到了一个点而已。

### UIEvent
一个触摸事件包含了所有与这个事件相关的触摸对象（UITouch），所以它很好的反映了这一次触摸的*综合*情况。所以我们通常都是捕获事件，而不是直接捕获触摸对象。

* `touchesBegan:withEvent:` 一只或多只手指触摸到了屏幕。
* `touchesMoved:withEvent:` 一只或多只手指在屏幕上移动。
* `touchesEnded:withEvent:` 一只或多只手指从屏幕上抬起。
* `touchesCancelled:withEvent:` 手势队列被系统事件打断，比如来了个电话。

## 触摸事件的传递链
如果一个视图上添加了系统的手势识别器，那么当这个视图所在 window 收到了触摸消息时，它会优先传递给手势识别器，然后延迟一小段时间再传递给视图。当手势识别器的状态改变为 Recognized 的时候，接下来还没发送给视图的触摸事件将不会再发送给视图，而视图的事件状态也会被置为 Cancelled。

{% img center /uploads/Custom-Gesture-Recognizers/continuous_gesture_2x.png 手势状态 %}

关于手势识别器和视图的手势响应事件的一些冲突，可以通过 UIGestureRecognizer 的属性来进行一定的控制，具体就不在这里展开了。

## 自定义手势识别器
想要实现自定义的手势识别器，最好的方法就是继承 UIGestureRecognizer，在子类的头文件中引入这个头文件：

```objc
#import <UIKit/UIGestureRecognizerSubclass.h>
```

并且重写下面的方法：

```objc
- (void)reset; // 到达手势的终止状态时，就会调用这个方法
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event;
```

## 举例子
说了那么多，还是来写点代码吧，实现一个自定义的缩放手势`CMLPinchGestureRecognizer`。

对于缩放手势的特性，先做一些限制：

1. 只能两指缩放（`touches.count == 2`）
2. 缩放比例不累加（系统 Pinch 手势会累加）

清楚了这些之后就可以开始动手了！

### 手指数目的判断
上面提到的需要重写的方法中，传入的 `touches` 参数是动态改变的，所以 `touches.count` 并不能反映真实的在屏幕上的手指个数。
什么意思呢？看看下面两种情况：

1. 两只手指**同时**放到屏幕上，并执行捏合/张开的操作。
2. 先放一只手指，再通过反复滑动另一只手指来达到缩放的效果。

第一种情况，`touches.count` **应该**是等于2的。
而第二种情况，会先收到一次 `touchesBegan:withEvent:`，它的 `touches` 参数里只有一个 UITouch（对应第一次放下的手指），然后收到多次`touchesBegan:withEvent:`，`touches` 参数里也是只有一个 UITouch（对应后面放下的手指）。

> 第一种情况的结果为什么说是**应该**呢？因为如果第一种操作下的**同时**不是很准确的话，也会引起第二种操作的情况。

所以为了实现手指个数的判断，需要在子类里面保存一个 UITouch 数组。
我们要在`touchesBegan:withEvent:`记录下新的 UITouch，并在`touchesEnded:withEvent:`中把抬起的手指对应的 UITouch 从数组中移除掉。

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesBegan:touches withEvent:event];
    
    if ([self.beganTouches count] == 2) {
        return;
    }
    
    NSArray *allTouches = [touches allObjects];
    for (UITouch *touch in allTouches) {
        if (![self.beganTouches containsObject:touch]) {
            [self.beganTouches addObject:touch];
        }
    }
}
```

```objc
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesEnded:touches withEvent:event];
    
    NSArray *allTouches = [touches allObjects];
    for (UITouch *touch in allTouches) {
        [self.beganTouches removeObject:touch];
    }
    
    ...
}
```

### 计算缩放比例
我们已经知道一个 UITouch 里面会保存有当前位置和上一个位置，那么只要用两个触摸点的当前距离除以上一次的距离，就可以得出缩放比例了。

计算距离的方法：

```objc
- (CGFloat)distanceBetweenPoint1:(CGPoint)p1 point2:(CGPoint)p2 {
    return sqrt( (p1.x - p2.x) * (p1.x - p2.x) + (p1.y - p2.y) * (p1.y - p2.y) );
}
```

在`touchesMoved:withEvent:`里面做计算：

```objc
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesMoved:touches withEvent:event];
    
    ...
    
    NSArray *touchesArray = [self.beganTouches copy];
    UITouch *f1 = touchesArray[0];
    UITouch *f2 = touchesArray[1];
    
    CGFloat currentDistance = [self distanceBetweenPoint1:[f1 locationInView:self.view] point2:[f2 locationInView:self.view]];
    CGFloat previousDistance = [self distanceBetweenPoint1:[f1 previousLocationInView:self.view] point2:[f2 previousLocationInView:self.view]];
    
    self.scale = currentDistance / previousDistance;
}
```

## 总结
到这里，已经可以基本实现一个缩放手势的识别并给出缩放比例了。当然上面给出的代码片段还不完整，还有很多地方需要完善。
例如识别器状态切换、延迟识别（主要为了防止与别的手势冲突）、特殊情况处理等等等等。
另外，在多个手势识别器协同工作的时候，也会有这样那样的一些坑，有时间的话再另开一篇番外篇来描述吧。

参考资料：

> 以前参考的是苹果官方的 **Event Handling Guide for iOS**，不过这篇文章已经从官网下架了，想了解更多信息的同学们可以看另外一篇 [Implementing a Custom Gesture Recognizer](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/ImplementingaCustomGestureRecognizer.html)

