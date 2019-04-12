---
title: NSTextField(1) —— macOS 输入框概览
date: 2019-04-12 15:15:04
tags:
- macOS
- Swift
- NSTextField
- NSControl
---

> 这是 macOS 开发系列的第四篇文章 —— NSTextField。
> 最常见的控件之一，却不一定是你最熟悉的控件之一。

<!-- more -->

将近半年没发文了，这段时间过得真是充实得过分，以至于完全没有时间好好整理一下手边可以写的内容。最近好不容易有点时间可以把存货整理整理，发现当时写的好多东西都已经过时了！赶紧收拾干净先发一篇上来，不然指不定哪天连整个主题都没用了...

咱们直接进入正题！


## NSTextField
一个完整的 TextField 是由两个类组成的：[NSTextFieldCell](https://developer.apple.com/documentation/appkit/nstextfieldcell)，干了绝大多数脏活累活的一个类，和 [NSTextField](https://developer.apple.com/documentation/appkit/nstextfield)，作为 NSTextFieldCell 的容器而存在。所有 NSTextFieldCell 里的方法在 NSTextField 里面都有对应的存在（有点像 UIView 对 Layer 的封装）。

对于绝大多数情况来说，我们直接用 NSTextField 就足够了（那这篇文章不久没什么作用了吗？！）。如果你想要对自己的输入框有更多的掌控权，那可能还需要了解一个叫 [NSControl](https://developer.apple.com/documentation/appkit/nscontrol) 的家伙。

## NSControl
正如 iOS 里的 [UIControl](https://developer.apple.com/documentation/uikit/uicontrol)，[NSControl](https://developer.apple.com/documentation/appkit/nscontrol) 是一个抽象类，必须通过子类继承来使用。但是跟比较纯粹的 UIControl 不同，NSControl 除了支持 Target/Action 机制和一些常见的属性设置之外，还加上了支持文字编辑的一系列代理方法。

举个两组最常用的例子：

### controlTextDidxxx(_:)
这么多方法里面，比较好用的当属 `did` 系列方法了：
1. [`controlTextDidBeginEditing(_:)`](https://developer.apple.com/documentation/objectivec/nsobject/1428934-controltextdidbeginediting?language=objc)
2. [`controlTextDidEndEditing(_:)`](https://developer.apple.com/documentation/objectivec/nsobject/1428847-controltextdidendediting?language=objc)
3. [`controlTextDidChange(_:)`](https://developer.apple.com/documentation/objectivec/nsobject/1428982-controltextdidchange?language=objc)

这三个方法虽然已经在官方文档里被标记为 “macOS 10.0-10.14 Deprecated”了，~~但它们仍然在勤勤恳恳地工作着~~。~~鉴于 macOS 的 10.14 还没出来~~（2018.8），~~我们但用无妨~~。2019年4月再看，系统版本已经到10.14以上了，是时候考虑正式换成下面的方法了。

> 上面三个方法的链接都是 Objective-C 版本的，被 Deprecated 的也是这个版本的方法。在 Swift 版的文档里，在 NSControlTextEditingDelegate 里已经加入这三个方法~~的 Beta 版~~了（2019.4 Beta 标识已经去掉了）：[controlTextDidBeginEditing(_:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/3005176-controltextdidbeginediting)、 [controlTextDidChange(_:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/3005177-controltextdidchange)、 [controlTextDidEndEditing(_:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/3005178-controltextdidendediting)。方法签名看起来是一样的，估计新方法的正式版出来之后也可以无缝迁移。

顾名思义，它们代表了输入过程中的三个状态，不过有一点要注意的是：每次出现 didEnd 并不一定会有一个对应的 didBegin。因为 didBegin 表示的是用户**开始输入**的状态，也就是说，单单是光标在控件上面闪烁着是不算数的，一定要用户敲下第一个字符的时候才会回调 `controlTextDidBeginEditing(_:)`。

而相对的，didEnd 表示**结束编辑**，只要用户选中输入框之后点击了输入框以外的地方，都会被算作“结束”，即使他从头到尾都没有输入过一个字。

这三个方法传入的参数都是 `Notification` 类型，说明它们其实都是系统通知的回调方法，只要实现了这个方法，系统就会自动帮你注册这三个消息的监听器。

> 虽然参数是个 `Notification` ，但它会把触发消息的输入框作为 object 属性一起传进来，可以做的事情就相当多了。

### NSControlTextEditingDelegate
这个代理是专门为了编辑操作而设计的，除了~~还在 Beta 版的~~三个 did 系列方法外（2019.4 Beta 标识已经去掉了），还有分工明确的另外7个方法，一共10个。这部分在现在的项目里还没怎么接触，就只是把文档搬过来方便大家参考。

验证：
* [control(_:isValidObject:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/1428873-control)
* [control(_:didFailToValidatePartialString:errorDescription:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/1428941-control)

格式化文本：
* [control(_:didFailToFormatString:errorDescription:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/1428883-control)

文本编辑响应：
* [control(_:textShouldBeginEditing:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/1428865-control)
* [control(_:textShouldEndEditing:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/1428984-control)

自动补全：
* [control(_:textView:completions:forPartialWordRange:indexOfSelectedItem:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/1428925-control)

按键事件响应：
* [control(_:textView:doCommandBy:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/1428898-control)

成员方法 ~~Beta~~:
* [controlTextDidBeginEditing(_:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/3005176-controltextdidbeginediting)
* [controlTextDidChange(_:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/3005177-controltextdidchange)
* [controlTextDidEndEditing(_:)](https://developer.apple.com/documentation/appkit/nscontroltexteditingdelegate/3005178-controltextdidendediting)



## 总结

没有错！到这里就结束了！（因为实在是没什么存货...）希望这篇文章能起到入门和索引的作用。

有了这些内容，应该大概能知道怎么去控制输入框的内容，也可以避免一些简单的坑了。

如果没人注意到标题里的“(1)”的话，我就在这里打住了…不然我可能会把打造一个真实情景下使用的输入框的过程讲一讲。