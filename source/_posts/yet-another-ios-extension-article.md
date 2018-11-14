---
title: 又一篇 iOS Extension 入门（1/3） — 基础 & 分享扩展 
date: 2018-11-14 16:00:50
tags:
---

应用扩展（App Extension）让你应用的功能和内容都得到了更大的延伸，这让用户在使用其他应用的时候有机会与你的应用发生交互。在这个大家都极力争夺注意力的时代，应用扩展无疑为我们打开了一扇新的大门。

<!--more-->

## 什么是应用扩展？
应用扩展与应用本身是有不同的。尽管在上架应用扩展的时候，你必须以一个普通的应用为载体（Containing App），但是它实际上是一个独立的二进制文件，而且并不依赖于载体应用来运行。

具体来说，应用扩展分为了十多个类别，你可以通过它们来实现各种各样的功能。下面是官方文档里介绍扩展点（Extension Point）的表格，我调整了一下格式：
![](/uploads/yet-another-ios-extension-article/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-11-13%20%E4%B8%8B%E5%8D%8812.05.50.png)

我们创建的每一个应用扩展都必须与上表的其中一个扩展点相对应，不允许出现一个应用扩展对应多个扩展点的情况。换句话说，每个应用扩展的职责都应该是单一的，我们应该给用户提供快速、线性、聚焦的体验。

## 应用扩展是怎么工作的？
### 生命周期
应用扩展的生命周期有别于一般的应用。应用扩展通常会在另一个应用的使用过程中被唤起，这个应用被称为宿主应用（Host App），宿主应用定义了与应用扩展交流的上下文，并通过发送请求的方式把应用扩展给启动起来。一般来说，应用扩展在完成宿主应用请求的任务之后，生命周期就结束了。
![](/uploads/yet-another-ios-extension-article/app_extensions_lifecycle_2x.png)

> 注意区分“载体应用”和“宿主应用”。载体应用是这个应用扩展的容器，在我们实现应用扩展的时候一并开发出来的；“宿主应用”指的是实际使用过程中调起我们的应用扩展的那个应用。  

在上图第2步里，系统在启动了我们的应用扩展之后，会在应用扩展和宿主应用之间建立一条通信通道，用于传递宿主应用定义好的上下文和相关信息。

应用扩展根据宿主应用发来的请求，执行相应的任务，这些任务可能是立即返回的，也可能通过一个后台进程去完成。但无论是哪种方式，在应用扩展跑完自己的代码逻辑之后，系统就会立马把它结束掉。

### 通信
上面提到应用扩展和宿主应用之间的通信方式，一个完整的通信关系是这样的：
![](/uploads/yet-another-ios-extension-article/simple_communication_2x.png)

应用扩展不会直接跟载体应用打交道，因为大多数情况下，应用扩展在工作的时候，载体应用甚至都还没有被启动。

在特殊情况下，比如 Today 小组件，扩展可以向系统提出启动载体应用的申请（通过调用 `NSExtensionContext` 的 `openURL:completionHandler:` 方法）。这时，应用扩展与载体应用就可以通过一个私有的共享容器来传递数据了，如下图所示：
![](/uploads/yet-another-ios-extension-article/detailed_communication_2x.png)

> 从系统层面上看，这已经涉及到进程间通信了，但苹果提供的高级 API 很好地屏蔽了这一点，所以我们完全不用考虑这些事情。  

### 应用扩展的“禁忌”
因为应用扩展与一般应用的设计是不同的，所以虽然开发起来差不多，但有些 API 是应用扩展无法使用的：
* 不能访问 `sharedApplication`
* 不能使用头文件里宏定义了 `NS_EXTENSION_UNAVAILABLE` 的框架，比如 HealthKit 和 EventKit UI 框架
* 不能访问摄像头和麦克风，除非它是 iMessage 应用
* 不能执行耗时过长的任务，具体限制与平台相关，但是它可以通过 `NSURLSession` 对象来实现数据上传和下载，最终的结果会给到载体应用
* 不能接收 AirDrop 数据，但是它可以发送

## 创建应用扩展
因为每一个扩展点都对应了一个特定的应用场景，所以创建应用扩展的第一步是选择正确的扩展点（可以回到文章开头部分查看扩展点表格）。

在 File -> New -> Target 里面，找到 Application Extension 模块，在里面选择想要实现的扩展点。这里我选了分享扩展作为例子：
![](/uploads/yet-another-ios-extension-article/7AA6705E-C543-4011-A638-E171E66F8E54.png)

给你的应用扩展起个美美的名字之后，它就会出现在项目配置的侧边栏里了，同时，Xcode 还为我们新建的应用扩展添加了一个 Scheme，让我们可以直接调式扩展而不用启动载体应用：
![](/uploads/yet-another-ios-extension-article/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-11-13%20%E4%B8%8B%E5%8D%885.45.59.png)

直接运行应用扩展，Xcode 会让你选一个应用来作为应用扩展的宿主：
![](/uploads/yet-another-ios-extension-article/A1A7864C-EA2B-4823-A33D-91656F91569A.png)

分享扩展的兼容性很好，我们选 Safari 来尝试分享一个网页好了：
![](/uploads/yet-another-ios-extension-article/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-11-13%20%E4%B8%8B%E5%8D%885.48.58.png)
随便打开一个网页，点击下面的分享按钮，可以看到我们的应用扩展已经出现在分享列表里面了！

> 应用扩展的图标会跟载体应用的图标一致  

## Talk is Cheap!
创建了应用扩展之后，会发现项目结构里多了一个属于应用扩展的位置：
![](/uploads/yet-another-ios-extension-article/65CB2068-CB5B-4443-A46B-3D8DC3DD5F30.png)

看起来就像一个普通的应用项目的结构，但 *Info.plist* 里有一个不同点，就是这里的 `NSExtension` 字典：
![](/uploads/yet-another-ios-extension-article/125ED1F7-CCA8-4D23-9218-5D8212E2D5D8.png)
看名字都挺好懂的，其中的 `NSExtensionAttributes` 用来配置一些通用参数，比如支持的媒体类型等等。默认情况下，`NSExtensionActivationRule` 是一个 `String` 类型，这个值就是让系统在所有分享场景里都显示我们的应用扩展（我全都要！）。更真实的场景应该是只支持特定的文件类型，这时可以把它改成 `Dictionary` 类型：
![](/uploads/yet-another-ios-extension-article/D7E348AB-33CD-4DC6-9F54-2005AE2BA66E.png)

上图的设置表示：我们支持分享图片、视频、文件和网页链接，后面的数字表示：一次分享中支持带上多少个这种类型的附件。

> 除图片和视频外的文件类型，都包括在 File 的范围里面，所以上面的配置几乎涵盖了所有的文件分享场景了  

### 响应请求
在 Xcode 创建好的 `ShareViewController` 里，我们可以通过 `extensionContext` 来拿到宿主应用想要传达给我们的信息：
```swift
let items = self.extensionContext?.inputItems
```

这是一个 [NSExtensionItem](https://developer.apple.com/documentation/foundation/nsextensionitem) 数组，每一个 `NSExtensionItem` 都带有一系列属性，例如标题、内容、附件、用户信息。

系统会回调 `didSelectPost` 或 `didSelectCancel` 以通知我们用户操作的结果，在这两个回调方法里，我们需要调用 `completeRequest(returningItems:completionHandler:)` 返回一系列 `NSExtensionItem` 对象给宿主应用，或者调用 `cancelRequest(withError:)` 返回一个错误。

### 获取附件
从 `NSExtensionItems` 里能直接获取到的信息是远远不够的，真正的大部头都在 `attachments` 这个属性里。这是一个 [NSItemProvider](https://developer.apple.com/documentation/foundation/nsitemprovider) 类型的数组，自此我们就基本看到了整个 `NSExtensionContext` 的构成了，借用一张其他博客的图片：
![](/uploads/yet-another-ios-extension-article/221150_8LYD_222120.png)

好，回到正题。拿到 `NSItemProvider` 之后，会发现要从这个类里面拿东西并不简单。

继续上面的例子，我们打算在用户点击 “Post” 按钮之后，获取从 Safari 分享过来的 URL，完整的代码是这样的：
```swift
override func didSelectPost() {
        // This is called after the user selects Post. Do the upload of contentText and/or NSExtensionContext attachments.
		  // 1
        guard let items = extensionContext?.inputItems as? [NSExtensionItem] else {
            return
        }
		  // 2
        for item in items {
            for attachment in item.attachments ?? [] {
				  // 3
                if attachment.hasItemConformingToTypeIdentifier(kUTTypeURL as String) {
					  // 4
                    attachment.loadItem(forTypeIdentifier: kUTTypeURL as String, options: nil) { (item, error) in
                        if error == nil {
                            print("found an url: \(item)")
                        }
                    }
                }
            }
        }
        
        // Inform the host that we're done, so it un-blocks its UI. Note: Alternatively you could call super's -didSelectPost, which will similarly complete the extension context.
        self.extensionContext!.completeRequest(returningItems: [], completionHandler: nil)
    }
```

1. `inputItems` 是一个 `[Any]` 类型的数组，所以在使用之前需要转换一下
2. 如果允许用户分享的时候多选的话，需要逐层遍历 `items` 和 `attachments`
3. 判断附件的类型，附件类型使用 [UTI（Uniform Type Identifier）](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/UTIRef/Articles/System-DeclaredUniformTypeIdentifiers.html#//apple_ref/doc/uid/TP40009259-SW1) 来表示，在 iOS 平台使用的时候需要先执行 `import MobileCoreService`
4. 读取指定类型的附件内容，回调过来的 `item` 是 `NSSecureCoding?` 类型的，这里只简单打印了一下，真正使用的时候还需要补充一些额外处理

## 性能要求
应用扩展对于用户来说应该是敏捷而且轻量的小工具，所以它的启动速度务必要保持在1秒以内，系统会自动关闭启动耗时太长的应用扩展。

对于 Widget 来说，界面上通常会一次显示多个，所以 Widget 的内存使用要求是最严格的（大概是16 MB），一旦超出了限制，会显示 “Unable to Load” 的字样：
![](/uploads/yet-another-ios-extension-article/TodayWidgetUnableToLoad.jpg)

其他类型的应用扩展对内存的要求会松一点，但还是比一般应用要严格，比如自定义键盘要求 48 MB 以下，分享扩展要求 120 MB 以下，实际情况可能跟设备相关。

另外，应用扩展是公用同一个主线程的，所以不要在应用扩展的逻辑里做可能会阻塞主线程的操作。同理，GPU 也是这样一个共享资源，如果一个应用扩展需要执行大量绘图逻辑，系统会倾向于把它结束掉。

总而言之，开销大的操作都应该在载体应用里做，而不是让应用扩展去负责。

## 总结一下
本文介绍了什么是应用扩展，并介绍了一个简单的分享扩展是怎么实现的。文章大体是来源于官方的文档，虽然文档已经被苹果归档了，但是文中的代码都是我写完用模拟器验证后得来的（2018年11月14日），大家可以直接拿走按需服用 :)

> 没想到只是介绍了一些基础就写了这么多。其实我还打算讲讲分享之后怎么跟载体应用交互，还想要看看今日小组件（Today Widget）怎么整...只好放到后面的文章里去了。  我发誓在这周之内把这两部分内容都给补上来！

## 参考文章
* [App Extension Programming Guide: App Extensions Increase Your Impact](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/index.html#//apple_ref/doc/uid/TP40014214-CH20-SW1)
* [App Extension Programming Guide: Share](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/Share.html#//apple_ref/doc/uid/TP40014214-CH12-SW1)
* [iOS扩展开发攻略(一) - Share Extension - vimfung的开源部落 - 开源中国](https://my.oschina.net/vimfung/blog/707448)