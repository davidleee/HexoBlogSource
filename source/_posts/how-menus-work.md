---
title: 让人眼花缭乱的 macOS 菜单
date: 2018-08-22 17:44:37
tags:
- macOS
- Swift
- AppKit
- NSMenus
- NSPopover
- NSPopupButton
- Dock
- NSDockTile
- NSStatusBar
---

> 这是 macOS 开发系列的第二篇文章——让人眼花缭乱的 macOS 菜单。
>
> 这有什么好说的，你不要骗我！手机上的菜单都是我用自定义视图撸出来的！  

<!-- more -->

## 菜单的类型

在 macOS 开发中，所谓“菜单”并不只是一个自定义视图了（虽然自定义视图也可以实现），在 AppKit 里面名字直接叫“菜单”的类就占了两席之地，分别是 [NSMenu](https://developer.apple.com/documentation/appkit/nsmenu) 和 [NSMenuItem](https://developer.apple.com/documentation/appkit/nsmenuitem)。

在实际应用中，菜单对应了 5 种表现形式：

* 应用的菜单栏，在屏幕的最上方
  ![](/uploads/how-menus-work/D54DBE49-05EB-4638-AEC3-FA37D500ECC7.png)

* 弹出菜单，可以出现在当前窗口中的任何地方
  ![](/uploads/how-menus-work/3D50179E-F77C-4A53-85E5-9296B53895D7.png)

* 状态栏，从屏幕上方的菜单栏右边开始向左延伸
  ![](/uploads/how-menus-work/WX20180821-170602@2x.png)

* “上下文菜单”（Contextual Menus），点击右键或 “control + 左键” 触发
  ![](/uploads/how-menus-work/56863C68-CAB1-4BCE-B0DD-82C7E4C03F5F.png)

* Dock 菜单，对程序坞（Dock）的应用图片点击右键或 “control + 左键” 触发
  ![](/uploads/how-menus-work/4948EFFA-1E19-4922-AB52-5FD4518F4268.png)

看着挺多，但是用起来倒是挺简单方便的。下面就把这几个新玩具都拉出来溜一溜～

## 应用菜单栏（Application Menu）
这个菜单栏在上一篇文章（{% post_link storyboard-in-macos 不一样的 macOS Storyboard %}）里已经纠结过了。

概括一下：每个应用初始化的时候就自带了一个应用菜单栏，如果是使用 Storyboard 开发的项目，在 “Main.storyboard” 里面就可以直接对这个菜单栏进行各种各样功能上的调整了（也就限于逻辑，样子大概是改不动了，只能用系统控件…）

## 弹出菜单（Pop-up Menu）
这种菜单的含义比较宽泛，所有在当前窗口里面出现的、带有“弹出”感觉的菜单都可以属于这一类。从视觉上大致可以分成两种：对话框型 & 按钮型。

### 对话框型
这中文名是我自己取的…因为它长得像呀！它就是上面类型介绍里的图片所示的样子，对应的类是 [NSPopover](https://developer.apple.com/documentation/appkit/nspopover)。

这玩意儿在 iOS 9 和更老的版本中有类似的用法叫 [UIPopoverController](https://developer.apple.com/documentation/uikit/uipopovercontroller)，在 iOS 9 之后就变成了 UIViewController 的一种展现方式了，具体参见 [UIPopoverPresentationController](https://developer.apple.com/documentation/uikit/uipopoverpresentationcontroller)。

NSPopover 用起来跟上面说的几个类也是差不多的，只是它本身不是一个 ViewController，所以在展示之前需要先设置 `contentViewController` 以负责界面的显示。

更多风骚的用法还是参考官方文档为好 -  [NSPopover](https://developer.apple.com/documentation/appkit/nspopover)。

### 按钮型
顾名思义，这是个看起来很像按钮的菜单，打开系统偏好设置 -> 通用，就能见到大把的按钮型弹出菜单，它们对应的类是 [NSPopUpButton](https://developer.apple.com/documentation/appkit/nspopupbutton)。

在 xib 或者 storyboard 里面拖一个出来，能看到它跟普通按钮相比多出了这么一些独有的配置：
![](/uploads/how-menus-work/A7088EFE-96C5-4124-BA69-05E917716EB1.png)

其中 “Type” 部分有两个可选值：Pop up & Pull Down，它们最直观的区别在于按钮后面跟着的蓝色部分，前者是上下箭头，后者则只有一个向下的箭头：
![](/uploads/how-menus-work/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-08-22%20%E4%B8%8B%E5%8D%884.40.57.png)

当然，它们在列表展开方式和使用场景上也是不一样的，想要追究其中细节的童鞋们，推荐一篇官方文档：[Managing Pop-Up Buttons and Pull-Down Lists](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MenuList/Articles/ManagingPopUpItems.html#//apple_ref/doc/uid/20000274-BAJDEEJA)

## 状态栏（Status Bar）
作为一个中规中矩的菜单，状态栏菜单也是由两个部件组成的：[NSStatusBar](https://developer.apple.com/documentation/appkit/nsstatusbar) & [NSStatusItem](https://developer.apple.com/documentation/appkit/nsstatusitem)，从名字就能看出它们的关系了，具体用法也是看文档咯。

> 额外推荐一篇很详细的文章：[Menus and Popovers in Menu Bar Apps for macOS | Ray Wenderlich](https://www.raywenderlich.com/450-menus-and-popovers-in-menu-bar-apps-for-macos)  

根据官方的说法，状态栏的位置稀缺，不保证你应用的菜单在上面一直是可用的，所以建议把它放在最后考虑（是的，甚至在 Dock Menu 之后）。做事比较克制的微信只把它用来显示未读消息条数，点击回调也只是打开微信主窗口而已。

而且苹果还建议我们提供一个隐藏的选项，在必要时给用户隐藏掉我们状态栏图标的机会。

Emmm….虽然苹果的话我们也不一定听就是了…

## “上下文菜单”（Contextual Menus）
也就是俗称的右键菜单？

> macOS 上还可以用 control + 左键触发  

它跟应用菜单栏一样，是通过最常见的菜单样式来展现的，对应的类是： [NSMenu](https://developer.apple.com/documentation/appkit/nsmenu) & [NSMenuItem](https://developer.apple.com/documentation/appkit/nsmenuitem)。

唯一不同的是，它不是通过点击一个什么按钮去触发的，而是通过重载 NSView 的 `defaultMenu` 属性来实现的，我们只需要定义好菜单的样子和内部逻辑，打开/收起菜单这样的琐事就交给系统去做好了。

在 xib 或 storyboard 里，把菜单链接到其他视图的 Outlets -> menu 上面，同样能实现右键触发的效果：
![](/uploads/how-menus-work/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-08-22%20%E4%B8%8B%E5%8D%885.16.55.png)

话说上图这个界面本身也是一个 Contextual Menu 呢。

## Dock 菜单（Dock Menu）
阿哈！这也是个普普通通的 Menu，通过实现 [NSDockTilePlugIn](https://developer.apple.com/documentation/appkit/nsdocktileplugin) 这个协议里的 `dockMenu()` 方法就可以返回一个我们自定义的菜单啦。

我们还可以通过它来自定我们的应用图标在 Dock 上面的样子，比如加个角标或者改一下图标颜色什么的，不过在这样做之前，我们还需要看看这个类 [NSDockTile](https://developer.apple.com/documentation/appkit/nsdocktile)。这就超纲了啊，不说了不说了。

## 总结
虽说自定义视图和显隐逻辑也可以实现菜单的功能，但是 AppKit 已经为我们封装了好几个类，让我们可以方便快捷地怼出一个功能丰富的应用了，它们是：

* [NSMenu](https://developer.apple.com/documentation/appkit/nsmenu) & [NSMenuItem](https://developer.apple.com/documentation/appkit/nsmenuitem)
* [NSPopover](https://developer.apple.com/documentation/appkit/nspopover) & [NSPopUpButton](https://developer.apple.com/documentation/appkit/nspopupbutton)
* [NSStatusBar](https://developer.apple.com/documentation/appkit/nsstatusbar) & [NSStatusItem](https://developer.apple.com/documentation/appkit/nsstatusitem)
* [NSDockTile](https://developer.apple.com/documentation/appkit/nsdocktile) & [NSDockTilePlugIn](https://developer.apple.com/documentation/appkit/nsdocktileplugin)

除了一些特殊的应用之外，我们的主要功能应该是在窗口里面提供的，而菜单通常都是些锦上添花的东西。不过在有余力的时候，为我们的用户增添一分“意外之喜”也是极好的吧。