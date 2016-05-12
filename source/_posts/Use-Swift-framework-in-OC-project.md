---
title: 在 OC 项目中使用基于 Swift 的 CocoaPods 库
date: 2016-05-12 16:57:28
tags:
- iOS
- Swift
- CocoaPods
---

随着 Swift 的流行，各种神奇的库也开始有对应的 Swift 版本了，而其中一些更神奇的库却只有 Swift 版本...
正巧接手了一个前人用 Swift 写的项目，里面有一个非常关键的图表库，找了半天硬是没有发现类似的 OC 版开源库。出于不想造轮子的心态，就让我们这些"落后"的 OC 党想办法兼容这些库吧！好在苹果为了推广这门新语言已经做好了准备工作，虽然还是需要绕个路，但是比起造轮子来，还是简单了不少。

<!-- more -->

因为项目是用 CocoaPods 来管理第三方库的，所以这次的兼容工作也会在 CocoaPods 上展开。不过道理还是那个道理，如果没有用到 CocoaPods 的话，直接跳过下面关于 Podfile 的那一步就好了。

## Podfile
要用 CocoaPods，首先要修改的当然是 Podfile，这是最简单的一步，只需要在文件开始加上这一句：

```
use_frameworks!
```

这是告诉 CocoaPods：“请把我要用到的第三方库用动态框架的形式集成进来”。
因为 Apple 不允许开发者构建内含 Swift 代码的静态库，所以要往 OC 项目中集成第三方 Swift 代码的时候就只能通过动态框架（ framework ）的形式了。

而 CocoaPods 还不能很好地将 framework 和静态库混编到一起，所以要么不用 framework，要用就要全部用上。关于这一点，[CocoaPods 官博](https://blog.cocoapods.org/CocoaPods-0.36/)上的原话是这样说的：
> This is an all or nothing approach per integrated targets, because we can't ensure to properly build frameworks, whose transitive dependencies are static libraries.

## Xcode配置
这一步的操作比较绕，但总体来说还是简单的。

首先在你的项目中任意创建一个 Swift 文件，这时候聪明的 Xcode 会问你需不需要它帮助你创建一个 Bridging 文件。
![bridging-header](/uploads/Use-Swift-framework-in-OC-project/bridging-header.png)￼

嘛，这当然是最好不过了，然而如果（像我这样）手贱点了 *Don't create* ，那以后不管你创建再多的 Swift 文件，它都不会问你了。不过，这当然是有手动操作的途径：

1. 手动创建一个头文件，名字叫 `Your_Product_Module_Name-Bridging-Header.h`，注意不是 `Project_Name`。
2. 确保你的项目目录下至少有一个 Swift 文件。
3. 确保在 **Targets** 的 Build Settings 里，**Product Module Name** 是有值的。（如果没有，直接设置为 `$(PRODUCT_NAME)` 就可以了）
4. 将 **Project** 的 Build Settings 里的 **Defines Modules** 设置为 `Yes`。（如果项目里没有创建过 Swift 文件的话，这个设置可能是不可见的）

配置完成！进入代码环节！

## 代码
其实也不需要什么代码啦。

完成了上面的所有步骤之后，Xcode 会自动生成一个名为 Your_Product_Module_Name-Swift.h 的文件，以后只要在需要使用到 Swift 代码的地方 import 这个文件就可以了。
现在已经可以直接按照 OC 的语法去调用 Swift 里的属性和方法了，开始愉快地 coding 吧 :)

> P.S. 为了避免循环引用，`-Swift.h` 文件只能在 `.m` 文件中 import。如果需要在 `.h` 文件中使用，就只能用 @class 来前向声明。


## 参考文章
* [苹果官方文档：Mix and Match](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/MixandMatch.html#//apple_ref/doc/uid/TP40014216-CH10-ID122)
* [Importing Project-Swift.h into a Objective-C class…file not found](http://stackoverflow.com/a/30202319)
* [CocoaPods 0.36 - Framework and Swift Support](https://blog.cocoapods.org/CocoaPods-0.36/)
