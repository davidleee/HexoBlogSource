---
title: Gesture Recognizers for iOS - The Basic
date: 2017-03-28 08:51:28
tags:
- iOS
- Gesture
---

> Your app should respond to gestures only in ways that users expect.

既然苹果已经将培养用户习惯的事情给做了，那么除非必要，开发的过程中还是尽量使用苹果自带的手势识别比较好。用文档的话来说，好处有这些：

* 简化代码数量
* 确保应用行为与用户期望的行为一致

<!-- more -->

那就先来看点基础的。

## 基础知识
### Gesture Recognizers与View的关系

一个手势识别器只能对应一个 view，但是反过来，一个 view可以对应多个手势识别器，因此一个 view 可以根据不同的手势作出不同的响应。
当 recognizer 连接到 view 上去之后，recognizer 会比 view 本身更早的接收到用户触摸事件，所以它也会代替 view 去对这些事件进行响应。

### 分离与连续

系统给出的默认手势处理可以分为两类：**分离式**和**连续式**。

分离式手势，比如说点击，每次用户操作一次，只会发来一个消息；而连续式手势，比如缩放，在用户操作的过程中，会不断地发消息过来，直到这个手势结束。
借用文档里的一张图，这两个过程长这个样子：
{% img center /uploads/Basic-Gesture-Recognizers/discrete_vs_continuous_2x.png Discrete vs Continuous %}

### 事件响应

将手势识别器与视图关联起来有两种途径：

1. 通过 IB 连线
2. 代码添加

事件的接收和处理都比较简单直接，可以参考官方的实例项目：[Simple Gesture Recognizers](https://developer.apple.com/library/ios/samplecode/SimpleGestureRecognizers/Introduction/Intro.html#//apple_ref/doc/uid/DTS40009460)

## 复杂一些的情况

### 有限状态机
Gesture Recognizers 其实是在一个有限状态机的基础上工作的，它们都是通过一个预先设定好的方式从一个状态变化到另一种状态。在开始状态和结束状态之间，手势识别器会分析接收到的多点触控消息队列，并根据分析的结果发出不同的消息。
还是盗文档的图来说明：
{% img center /uploads/Basic-Gesture-Recognizers/gr_state_transitions_2x.png GestureRecognizer State Transitions %}

每次状态的转换手势识别器都会发出对应的通知消息。而当状态到达 Recognized 的时候，手势识别器会将状态重置为 Possible，这个切换并不会向外给出通知。

### 多手势共存
通过 view 的 `gestureRecognizers` 属性可以访问到当前绑定在这个 view 上的所有手势识别器。

默认情况下，同一个 view 上的手势识别器是没有顺序可言的。也就是说，默认并不能确认每次操作会先触发哪个手势后触发哪个手势。但是我们可以重写这些行为已达到这样的效果：

1. 某些手势比另一些手势更优先地开始分析处理一个事件。
2. 阻止某个手势的事件处理。
3. 允许两个手势同时处理。

#### 手势间的顺序
使用某个手势的 `requireGestureRecognizerToFail:` 方法可以让这个手势延迟触发，知道作为参数的手势识别器状态改变为 fail。

e.g.
swipe 和 pan 一起添加到 view 上的时候，pan 总是在 swipe 之前被识别到，因为 pan 是一个持续性的手势，所以用户操作一开始就会被识别为 pan；而 swipe 则会在用户手指离开后才进行判断。

所以如果想要在 pan 之前优先识别为 swipe 的话，可以这样做

```objc
[self.panRecognizer requireGestureRecognizerToFail:self.swipeRecognizer];
```

这样就是告诉 pan 手势：你丫给我等到 swipe 手势 fail 了之后再说话！

所以在 swipe 手势 fail 之前，pan 手势识别器会一直处于 Possible 的状态，直到 swipe 手势 fail；反之，如果swipe手势识别器状态变成了 Recognized 或 Began 的话，pan 手势识别器就会自动转变为 Failed 状态。

> 如果一个地方同时识别单击和双击，会出现两种不那么友好的情况，最好可以在交互上规避：
>
> 1. 不使用 `requireGestureRecognizerToFail:` 方法：在收到双击通知之前，会收到一次单击的通知，需要在单击方法的处理上区分这些情况。
> 2. 使用 `requireGestureRecognizerToFail:` 方法：单击方法会有一些延迟，因为它需要等到双击手势识别器确认为 Failed 状态之后才可以执行。

#### 阻止手势识别器处理事件
这种高级玩法需要实现 `UIGestureRecognizerDelegate`，使用里面的两个 optional 代理方法 `gestureRecognizerShouldBegin:` 和 `gestureRecognizer:shouldReceiveTouch:`。

在用户进行操作时，如果可以立马判断出是不是需要响应这个事件的话，那就使用 `gestureRecognizer:shouldReceiveTouch: ` 方法，这个方法会在每次新用户事件产生的时候被调到。

当需要等待一段时间之后才能判断是不是要响应这个事件的话，那就应该使用 `gestureRecognizerShouldBegin:` 方法。这个方法会在手势识别器尝试从Possible状态改变为其他状态的时候被调用，如果返回 `NO` 的话，这个手势识别器会立马被置为 Failed 状态，然后让下一个识别器继续处理这个用户事件。

UIView 里面也有一个同样的 `gestureRecognizerShouldBegin:` 方法，方法签名和实现都跟上面提到的这个代理方法一致，在不想实现 delegate 的时候很方便实用。

#### 允许多手势同时识别
默认情况下，多个手势是不能同时被识别的到的，但是可以通过实现代理方法 `gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer: ` 来达到多手势同时识别的目的。

> 想要两个手势同时被识别的话，你只需要针对一个手势识别器实现上面这个方法，并返回 YES。
> 所以，返回一个 NO 并不能确保这两个手势的识别互斥，因为有可能在另一个手势的对应代理方法中返回的是 YES。

#### 多手势识别中的单向关系
这标题比较拗口，举个例子：
触发旋转手势的时候禁用缩放，但是触发缩放的时候可以旋转。

这时候就要重写 `canPreventGestureRecognizer: ` 或 `canBePreventedByGestureRecognizer:` 方法了。

像上面的例子那种需求，首先要这样写：

```objc
[rotationGestureRecognizer canPreventGestureRecognizer:pinchGestureRecognizer];
```

然后重写旋转手势识别器里面的这个方法，并返回一个 NO。这其实算是自定义手势识别器了，毕竟重写方法最好的做法还是先继承。

## 总结
这里其实只涵盖了非常有限的一些基础知识，后面如果有机会就把这个写成一个系列文章。

参考资料：

> 以前参考的是苹果官方的 **Event Handling Guide for iOS**，不过这篇文章已经从官网下架了，想了解更多信息的同学们可以看另外一篇 [Gesture Recognizer Basics](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/)

