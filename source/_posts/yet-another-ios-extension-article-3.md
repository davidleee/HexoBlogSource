---
title: 又一篇 iOS Extension 入门（3/3）— Today 小组件
date: 2018-11-15 16:12:27
tags:
- ios
- Today Extension
- Widget
- Universal Link
---

昨天通过两篇文章介绍了 iOS Extension 的基础，并尝试制作了一个分享扩展，让我们的应用可以接收到从其他应用分享过来的数据，还实现了跨沙盒的应用扩展与载体间的通信。

看上面这段话就觉得内容挺多的吧…所以专门把 Extension 界的当红选手 —— Today 小组件单独放在这一篇文章里面讲，作为这个 iOS Extension 入门系列的收尾～

让我们马上进入正题！

<!--more-->

## 啥是 Today 小组件
展示在 Today 界面（手机主页最左屏）里的应用扩展统称为“小组件”（Widget）。小组件存在的目的是向用户快速展示**当下**最重要的信息，并提供一些简易的任务处理功能，比如“把任务标记为完成”之类的。

> 官方建议： Today 小组件负责的任务最好在单次操作内就能完成，如果你发现这个任务需要多个步骤，那 Today 小组件也许不是最适合的扩展点。具体扩展点参见 [官方扩展点列表](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/index.html#//apple_ref/doc/uid/TP40014214-CH20-SW3) 或者这个系列文章的第一篇（{% post_link yet-another-ios-extension-article 又一篇 iOS Extension 入门（1/3）— 基础 & 分享扩展 %}）里的翻译版。

### 划重点
iOS 和 macOS 平台上都有 Today 小组件，在开发过程中需要注意的地方是相同的：
* 确保内容是最新的
* 谨慎地对待用户交互
* 性能调优

> 从交互上来说，务必避免在小组件里放滚动列表，因为用户很难区分小组件内部的滚动和整个小组件列表的滚动

**在 iOS 上**：小组件不允许键盘输入，所以一切针对小组件的设置都应该在载体应用内完成。以“股市”为例，用户可以直接在小组件上切换显示的单位，但是股票列表的编辑需要在载体应用里进行。
**在 macOS 上**：载体应用可以不做任何功能，小组件可以提供一个配置入口。还是以“股市”为例，小组件里可以直接搜索、添加和删除特定股票。

## 来做一个 Today 小组件吧
就像创建分享扩展那样，首先要在项目配置里添加一个 Target（复习{% post_link yet-another-ios-extension-article 系列文章第一篇 %}），如果想要共享数据的话，还需要配置一下 Capabilities -> App Groups（复习{% post_link yet-another-ios-extension-article-2 系列文章第二篇 %}）。

Xcode 依旧贴心地为我们创建了一个目录，随便点开看看，可以发现 *Info.plist* 里关于 `NSExtension` 的内容有所不同，其中的 `NSExtensionActivationRule` 字段已经没有了，因为 Today 小组件的开关是用户自己选择操作的，不需要我们开发者去判断。

### 界面
> 为了实现最好的效果，建议使用 AutoLayout 去做界面的布局。

Today 小组件的宽度是固定的，高度上有延伸的空间以显示更多的内容。Xcode 创建的 IB 模版里已经用上了 AutoLayout，并用上了标准的四周间隔，我们可以通过 `widgetMarginInsetsForProposedMarginInsets:` 方法来获取到这些间隔以便计算。
> 模版里的 VC 已经实现了 `NSWidgetProviding` 协议，上述方法就是在这个协议里定义的。

界面部分最值得一提的就是右上角的“展开/折叠”了。这个按钮默认情况下并不会显示，需要我们添加一些代码来实现：
```swift
override func viewDidLoad() {
    super.viewDidLoad()
	  // 1
    extensionContext?.widgetLargestAvailableDisplayMode = .expanded
}

// 2
func widgetActiveDisplayModeDidChange(_ activeDisplayMode: NCWidgetDisplayMode, withMaximumSize maxSize: CGSize) {
	  // 3
    if activeDisplayMode == .expanded {
        preferredContentSize = CGSize(width: maxSize.width, height: 200)
    } else {
        preferredContentSize = maxSize
    }
}
```

1. 告诉 `extensionContext` 我们的小组件是支持展开的，这个属性的默认值是 `.compact`
2. 用户点击“展开/折叠”的回调，如果不进行处理，小组件的高度不会发生变化
3. 修改小组件的高度，这里的 `maxSize` 是系统限制的当前模式下的最大尺寸，使用 iPhone XR 模拟器测试时，`.compact` 模式下是 `(398, 110)`，`.expanded` 模式下是 `(398, 748)`。可见，苹果限制了折叠状态下最大高度为 110，超出部分会直接截掉；而展开状态下，最大高度为设备的高度。

界面部分就没什么了，剩下的该是具体问题具体分析。接下来轮到功能逻辑的部分。

### 跳转到载体应用
实际上，小组件还是通过 Universal Link 的机制来唤起载体应用的，与应用间跳转没有什么区别，只不过需要通过 `extensionContext` 来调用：
```swift
// 唤起 URL Schemes 为 davidleee 的应用
extensionContext?.open(URL(string: "davidleee://")!, completionHandler: nil)
```

需要注意的是，如果你的小组件要打开第三方的应用，在提交 App Store 审核的时候需要特别说明一下，否则会被打回。

> 关于 Universal Link 的官方文档在这里：[Universal Links - Apple Developer](https://developer.apple.com/ios/universal-links/)，我也写过一篇文章记录了可能存在的一些坑，感兴趣的可以瞅瞅：{% post_link universal-link-problems Universal Link（iOS）踩坑 %}

### 数据更新
既然 Today 小组件的目的就是为用户提供最新鲜的数据，那么数据更新的部分就一定不能马虎。

在 Xcode 帮我们创建的 `TodayViewController` 里面，我们可以看到一个叫  `widgetPerformUpdate(completionHandler:)` 的方法，在它的描述里能看到这么一句话：

*This method is called to give a widget an opportunity to update its contents and redraw its view prior to an operation such as a snapshot.* 

苹果设计这个 API 是为了把数据的更新统一放到一个地方去。如果我们实现了这个方法，系统就会在合适的时候调用这个方法（比如系统想要给你的小组件进行 snapshot 之前），给我们一次更新数据的机会，并且这个机会不一定出现在小组件显示出来的时候，在后台的情况下也有可能触发这个回调。

于是我们就有两个拉数据的机会：
* 在 `viewDidLoad` 里面
* 在 `widgetPerformUpdate(completionHandler:)` 里面

> 实验发现，小组件只要不可见的时间稍微长一点点，比如滚动出了屏幕，或离开 Today 视图一小会，它就会被重新初始化，也就是说 `viewDidLoad` 的调用会比想象中更频繁。但这并不意味着我们可以完全依赖 `viewDidLoad` 来做数据更新。

在 SO 上的[这个讨论](https://stackoverflow.com/questions/25168950/what-is-the-purpose-of-widgetperformupdatewithcompletionhandler-in-ios-8-today-w)里，对数据更新的时机进行了更多的讨论，总结起来就是两点：
1. 苹果希望你在整个生命周期里尽可能早的地方进行数据更新，所以 `viewDidLoad` 要用
2. `viewDidLoad` 还不够，那就用上 `widgetPerformUpdate(completionHandler:)`，毕竟前者并不会在后台情况下被调用

## 总结一下
Today 小组件就是一个用来展示**小块**数据和处理**简单**任务的地方。

注意上面那句话加粗的两个词，这给小组件定下了一个主基调：敏捷，所以凡是逻辑越写越复杂的时候，都该停下来想一想：这些逻辑是不是应该挪到载体应用里面去做？

> 用这个理由去怼产品经理吧，就说是那个估值超万亿的苹果的产品经理说的～

## 参考文章
* [App Extension Programming Guide: Today](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/Today.html#//apple_ref/doc/uid/TP40014214-CH11-SW1)
* [iOS today extension (swift) 教學筆記 – 碼農勤耕田 – Medium](https://medium.com/nine9devtw/ios-today-extension-swift-%E6%95%99%E5%AD%B8%E7%AD%86%E8%A8%98-5361446d1950)
* [ios8 - What is the purpose of widgetPerformUpdateWithCompletionHandler in iOS 8 Today Widget? - Stack Overflow](https://stackoverflow.com/questions/25168950/what-is-the-purpose-of-widgetperformupdatewithcompletionhandler-in-ios-8-today-w)
