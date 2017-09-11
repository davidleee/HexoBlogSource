---
title: macOS 下的 Window 和 WindowController
date: 2017-09-11 15:22:15
tags:
- macOS
- Window
- Controller
- TextView
- Modal
- Document
- Menu
---

相较于 iOS 上火热的开发势头，macOS 开发简直就是一片蓝海。让人不禁有些好奇，本是同根生的 macOS 开发究竟是一番怎样的光景？在略微接触之后发现，除了 UIKit 被 AppKit 替换之外，最明显的是 macOS 对待 Window 的态度转变，想想也是，毕竟桌面端应用的效率优势很大一部分就是体现在窗口多开上。然而找了找现有的资料，关于 macOS 开发的实在不多，于是就在学习的过程中翻译一篇国外教程，为社区做点贡献。

本文是 [RayWenderlich](https://www.raywenderlich.com/) 上的一篇翻译文，里面带着读者从无到有地构建了一个简单的文本编辑器，内容会涉及到 macOS 上 Window 相关的一些使用基础。翻译里加入了一点个人理解，但是技术部分是忠于原文的，不放心的可以直接到官网上看：

原文链接：[Windows and WindowController Tutorial for macOS](https://www.raywenderlich.com/159287/windows-windowcontroller-tutorial-macos)

说太多了，来看正文啦！

<!-- more -->

Window 是一切 macOS 应用的界面载体，它定义了一个专属于某个应用的区域，并作为多任务处理的标识展现给用户。

一切 macOS Apps 都不外乎是下面三种类型之一：
* 单一窗口工具型应用（一个界面就完成所有功能），比如**计算器**
* 单一窗口图书馆式应用（一个窗口完成所有功能，但这个窗口里的界面可能有许多个），比如**照片**
* 多窗口的基于文档的应用，比如**文本编辑**

这篇教程将会涵盖下列知识：
* Windows 和 windowControllers
* 文档（Document）架构
* NSTextView
* 模态窗口
* 菜单栏和菜单项

而看这篇文章的读者们可能需要提前掌握这些知识：
* Swift 3 或更高版本的 Swift 语法
* Xcode 和 Storyboards 的基本操作
* 在 Mac App 上实现一个 Hello world
* 控件的响应链

## 那么就开始吧
同创建一个计算器 App 不同的是，我们将要创建的是一个基于文档的应用，在创建工程项目的时候，Xcode 会给出提示让你选择应用的类型：

{% img center /uploads/Windows-in-macOS/create-project.png Create Project %}

上面的内容可以自由发挥，唯独红框的部分要注意一下：
* Create Document-Based Application 要勾上，此时 Xcode 会为你生成基于文档型应用的示例代码，能省去我们不少工作量
* Document Extension 是告诉 Xcode 我们这个应用要操作的文档的后缀，我这个 Demo 的名字叫 “MyTextEditor”，所以我就取首字母 “mte” 作为后缀了
* 下面是关于数据库和测试的部分，同样是勾了就会有示例代码，但我们这里不需要，所以不勾选它们以排除一些干扰

项目创建好后马上就可以运行了，原始的 MyTextEditor 应该是这个样子的：

{% img center /uploads/Windows-in-macOS/raw-window.png Empty Window %}

而且它已经具有一些基本功能了，比如你已经可以新建很多个窗口（不过这些窗口是重叠在一起的，你可能要拖动一下才能看到后来的窗口）：
{% img center /uploads/Windows-in-macOS/Open-Many.png New Windows %}


## 文档（Documents）
在继续之前，我们要先来了解一下文档类型应用是怎么工作的。

### 文档（Document）架构
一个文档对应的是一个 `NSDocument` 类型的对象，它相当于这个文档的控制器。通过它，我们可以读取文件的内容或往里面写东西，而且它既可以是本地硬盘上的文件，也可以是存在 iCloud 上的。

NSDocument 是一个抽象类，也就是说你需要用一个子类去实现具体功能。在文档架构中还有两个很主要的类：`NSWindowController` 和 `NSDocumentController`，它们作用分别是：
* `NSDocument`：创建和保管文档数据
* `NSWindowController`：管理用来展示文档的窗口
* `NSDocumentController`：管理一个应用中的所有文档对象

{% img center /uploads/Windows-in-macOS/DocArchitecture.png Document Architecture %}

### 文档操作
还记得创建工程的时候，我们告诉了 Xcode 这个 App 是一个文档型应用吗？聪明的 Xcode 知道了这一点之后，会给我们的应用內建许多文档操作，但一些具体的逻辑还是需要我们继承 `NSDocument` 去实现。（Xcode 其实已经给了我们一个叫 `Document` 的子类作为例子了）

打开 **Document.swift**，可以看到已经有用于文件读写的空方法了（`data(ofType:)` 和 `read(from:ofType:)`）。运行这个 App 时，你可能已经发现顶部菜单栏里很多功能都是已经实现了的，比如新建、打开、保存等等，不过我们这个 Demo 里面不涉及“保存”，所以我们需要删除相关的逻辑，也借此看看 Xcode 帮我们做了些什么。

打开项目里唯一的 Storyboard，然后像下图这样取消菜单项和实际逻辑之间的关联：

{% img center /uploads/Windows-in-macOS/TargetAction.png Target-Action %}

把 **Open**、**Save**、**Save As**、**Revert to Saved** 的关联都干掉，因为这些我们都不会用到。

接下来打开 **Document.swift**，添加以下代码，我们要在用户尝试保存的时候弹一个提示：
```swift
override func save(withDelegate delegate: Any?, didSave didSaveSelector: Selector?, contextInfo: UnsafeMutableRawPointer?) {
        let userInfo = [NSLocalizedDescriptionKey: "Sorry, no saving for you, sir! Click \"Don't save\" to quit."]
        let error = NSError(domain: NSOSStatusErrorDomain, code: unimpErr, userInfo: userInfo)
        NSAlert(error: error).runModal()
    }
```

{% img center /uploads/Windows-in-macOS/no-saving-for-you.png No saving for you %}

现在重新运行项目，你会发现菜单栏里我们刚才取消关联的选项已经无法选中了：

{% img center /uploads/Windows-in-macOS/menuDisabled.png Menu Disabled %}

好了，清除掉了障碍，现在要开始真正的开工了！

## 窗口位置
首先我们要修复的一个问题是：新建的窗口都是死死盖在原来的窗口上面的。我们会通过继承一个窗口控制器来实现这部分逻辑。

### 继承一个 `NSWindowController`
新建一个 `NSWindowController` 的子类，确保语言选了 Swift，并且不要勾选创建 xib 的那个选项：

{% img center /uploads/Windows-in-macOS/WindowController.png New WindowController %}

然后打开 Storyboard ，将里面的 Window Controller 的 Custom Class 配置为我们刚刚新建的 `WindowController`：

{% img center /uploads/Windows-in-macOS/windowcontroller-class.png Set Custom Class %}

然后开始解 Bug！在我们的 WindowController.swift 里面，重写父类的一个初始化方法：
```swift
required init?(coder: NSCoder) {
  super.init(coder: coder)
  shouldCascadeWindows = true
}
```

运行！

{% img center /uploads/Windows-in-macOS/new-window.png New Windows %}

除了第二个窗口不是很听话，后续的窗口都已经会排好队了。

### 用 Tabs 绕过这个问题
这第二个窗口是怎么回事呢？我们后面将初始化窗口位置的时候再回答，现在我们先用另一种新建窗口的方式去绕过这个问题~（正式开发中千万不能绕开问题啊！）

其实将新建窗口变为新建 Tabs 超级简单，只需要在 Storyboard 里面配置一下就好了：

{% img center /uploads/Windows-in-macOS/tabbing-mode-preferred.png Tabbing Mode %}

重新运行，这次当你新建窗口时，这些窗口就会以一个个 Tab 的形式出现了：

{% img center /uploads/Windows-in-macOS/raw-windows-with-tabs.png New Windows in Tabbing Mode %}

### 在 IB 中设置窗口的位置
回到我们绕过的问题本身：窗口的位置。

在 Storyboard 里，当 Window Controller 里的 Window 被选中时，可以在右侧的 Size Inspector 中看到对窗口位置和大小的配置，其中 “Initial Position” 里设置的就是窗口的初始化位置：

{% img center /uploads/Windows-in-macOS/windowposition.png Initial Position %}

> 在 macOS 中，坐标轴的原点在左下角，横轴是 X 轴，纵轴是 Y 轴，跟 iPhone 上的坐标系要区别开来。

你也可以直接拖动那个小界面里的灰色窗口来设置初始位置。注意小界面下面的两个下拉框的内容变化：

* Proportional Horizontal/Vertical：初始化位置会根据屏幕的大小按比例来设置
* Fixed From Left/Right/Top/Bottom：写死一个固定的初始化位置

在这个 Demo 里，我们会让窗口固定在左下角 (200,200) 的位置出现：
* 设置下拉框内容为 Fixed From Left 和 Fixed From Bottom
* 将初始值设置为 X:200 和 Y:200

> macOS 会记录每次应用启动后的窗口位置，所以为了看到这里的设置引起的变化，你要先把所有的窗口的关掉，再编译运行项目。

### 用代码设置窗口的位置
这一小节其实就是把上一节做的事情用代码重新做一遍，以防你们以为 Swift 程序员不会写代码。

用代码设置还是有它的好处的，比如你可以在应用运行的过程中决定窗口要在哪里出现。

打开 **WindowController.swift**，把里面的 `windowDidLoad` 方法的实现改成下面这样：
```Swift
override func windowDidLoad() {
  super.windowDidLoad()
  //1.
  if let window = window, let screen = window.screen {
    let offsetFromLeftOfScreen: CGFloat = 100
    let offsetFromTopOfScreen: CGFloat = 100
    //2.
    let screenRect = screen.visibleFrame
    //3.
    let newOriginY = screenRect.maxY - window.frame.height - offsetFromTopOfScreen
    //4.
    window.setFrameOrigin(NSPoint(x: offsetFromLeftOfScreen, y: newOriginY))
  }
}
```

1. 取到窗口和屏幕对象（还记得 Swift 是怎么安全的获取 Optional 对象的值吗？就是这样~）
2. 取得屏幕的可视范围
3. 计算 Y 坐标的值（别忘了坐标轴原点是左下角，这里计算的是底边的高度）
4. 更新窗口的坐标

`visibleFrame` 这个属性不包含 Dock 和菜单栏的范围，如果不用这个参数来计算的话，可能会出现窗口被这两个控件挡住的情况。

重新编译运行，窗口应该就会出现在距离屏幕（不算菜单栏高度）左上角 (100,100) 的位置了。

## 打造一个迷你文字编辑器
Cocoa 自带了一些很神奇的 UI 和功能，就等着你把它们用起来了。接下来我们会接触到多才多艺的 `NSTextView`，但首先我们要先了解一下 `NSWindow` 自带的 content view。

### The Content View
`contentView` 是一个窗口中所有视图层级中的根视图，`NSWindow` 里的 content view 由它带着的 `ViewController` 来体现。

{% img center /uploads/Windows-in-macOS/contentVIew.png Content View %}

喏，那个蓝的发亮的就是 `contentView`

### 添加 Text View
打开 **Main.storyboard**，从右边栏拖一个 `NSTextView` 到上面说到的 `contentView` 里面去，把它调节到一个舒服的大小，然后点击右下角的小三角形，选择 **Reset to Suggested Constraints**。

{% img center /uploads/Windows-in-macOS/autolayout.png AutoLayout %}

这里我们让系统自动帮我们布局这个视图，省点事儿。

编译运行，你应该能看到我们简陋的文字编辑器了。尝试拉伸一下窗口，Text View 会跟着窗口一起变大变小，这是 AutoLayout 的功劳，这个 Demo 里面不会讲咯。

{% img center /uploads/Windows-in-macOS/mini_text_editor.png Mini Text Editor %}

好好探索一下我们的第一个文字编辑器吧，你会发现 Format - Font - Show Font 功能并不可用，我们接下来就解决这个问题。

### 打开字体设置框
在 **Main.storyboard** 里的 Main Menu 上找到 Show Font 这个菜单项，按住 Ctrl 把 Show Font 拖到 First Responder 上，然后在随之出现的弹框中找到 `orderFrontFontPanel:` 并选择它：

{% img center /uploads/Windows-in-macOS/font-to-first-responder.png Show Fonts %}

然后重新编译运行项目，Show Font 功能就被打开啦！

在不写一行代码的前提下，你实现了改变字体的功能，这是怎么做到的呢？其实是 `NSFontManager` 和 `NSTextView` 把所有的脏活累活都给干掉了。
* `NSFontManager` 是一个管理字体变化系统的类，它就是刚刚弹框中 `orderFrontFontPanel:` 方法的实际实现的地方，我们刚才的操作是把响应链上的信息发送（forward）给了它，然后它负责展示系统默认的字体设置框
* 当我们在字体设置框中对字体进行操作，`NSFontManager` 会发送一个 `changeFont` 消息给当前的第一响应对象（First Responder）
* `NSTextView` 实现了 `changeFont` 方法，当我们操作它里面的文字时（比如选中某个单词），它就自动成为了第一响应对象，然后一切就联系起来了

### 富文本
要看到 `NSTextView` 的真正实力，你可以先从[这里](https://koenig-media.raywenderlich.com/uploads/2017/04/BabyScript.rtfd_.zip)下载一段富文本，然后把它设置为 `NSTextView` 的默认文字，编译运行！

{% img center /uploads/Windows-in-macOS/rich_text.png RichText %}

嗯？文本里的图片哪里去了呢？

因为 IB 里面的设置默认文字的地方不能保存图片，所以图片就被丢弃掉了。不过我们还是可以通过复制粘贴或者拖拽的方式，把图片添加到 Text View 里面去。

玩耍过后，在你想要关闭这个窗口的时候，你会发现我们在文章开头设置的弹窗生效了！（我们在前面禁用了保存功能）

### 把帅气的刻度尺显示出来
打开 **ViewController.swift**，把 `viewDidLoad` 附近的代码替换成下面这段：
```Swift
@IBOutlet var text: NSTextView!
  
override func viewDidLoad() {
  super.viewDidLoad()
  text.toggleRuler(nil)
}
```

然后回到 **Main.storyboard**，把我们手写的 `text` 和 IB 里的 Text View 关联起来：按住 Ctrl，把代表 ViewController 的蓝色小圆圈拖向 Text View，选择 text。

{% img center /uploads/Windows-in-macOS/connect_outlet.png Connect Outlets %}

再跑一遍，看起来是不是高大上了一些：

{% img center /uploads/Windows-in-macOS/RulerShowing.png Show Ruler %}

## 模态窗口
模态窗口是 Window 世界中最霸道的存在，一旦出现，它会吃掉所有的事件，知道它们被主动 dismiss 掉。保存和打开文件的弹窗就是模态窗口的范例，总的来说，有三种方式展示模态窗口：
1. 当做一个常规窗口使用，通过调用 `NSApplication.runModal(for:)` 显示
2. 当做一个表单（Sheet）用，通过调用 `NSWindow.beginSheet(_:completionHandler:)` 显示
3. 通过一个模态会话来展示，这是一个高级用法，这里不讲

前面尝试关闭窗口时弹出的保存提醒框就是一个表单型的模态窗口：

{% img center /uploads/Windows-in-macOS/sheet-modal.png Save Window %}

嘛，这玩意儿就这样，我们在这里也不会接着深入了。但是我们会看看一个分离式的模态窗口怎么出现的。

### 添加一个新 Window
打开 **Main.storyboard**，从右边栏拖一个 Window Controller 到画面上，这会生成两个东西，一个 Window Controller Scene 和一个 View Controller Scene：

{% img center /uploads/Windows-in-macOS/newwindowcontroller.png New Window with IB %}

选中 Window Controller Scene 下面的 Window，把它的 Content Size 改为宽300高150，顺带也把 View Controller Scene 下面的 view 也做这样的修改：

{% img center /uploads/Windows-in-macOS/wc-window-frame.png Set Content Size %}

{% img center /uploads/Windows-in-macOS/wc-view-frame.png Set View Size %}

然后我们要禁用这个窗口左上角的那些控制按钮，让用户必须沿着我们设定的交互走：

{% img center /uploads/Windows-in-macOS/wordcount-props.png  Window Controls %}

### 新窗口里的界面布局
就像上文对 Text View 的布局那样，把我们的新窗口也进行一番鼓捣，留给大家自由发挥啦。Demo 里使用了4个 Label 和一个 Button，最终长这个样子：

{% img center /uploads/Windows-in-macOS/word_count_ui.png WordCountWindow UI %}

### 创建对应的 View Controller 类
为了控制这个窗口的内容，我们要从 `NSViewController` 继承一个子类：

{% img center /uploads/Windows-in-macOS/new_word_count_vc.png New WordCountViewController %}

接着在 IB 里关联一下界面和这个子类：

{% img center /uploads/Windows-in-macOS/wc-configure-custom-class-368x320.png Set Custom Class for WordCountViewController %}

### 界面与数据绑定
接下来，我们用 macOS 上的一个神奇功能，实现界面与数据的直接绑定。

打开 **WordCountViewController.swift**，给这个类添加两个属性：
```Swift
dynamic var wordCount = 0
dynamic var paragraphCount = 0
```

`dynamic` 关键字使得这两个属性可以被用作 **Cocoa Bindings** 的绑定对象。

回到我们的 **Main.storyboard**，选中上面添加的 “Word Count” 后面的那个 “0”，然后在右边栏进行设置，将这个 Text View 的内容跟 `wordCount` 属性的值进行绑定：

{% img center /uploads/Windows-in-macOS/bind-wordcount.png Cocoa Bindings %}

对 **Paragraph Count** 后面跟着的 Text View 也进行相同的操作，但是这次要绑定到 `paragrahCount` 属性上去。

> **Cocoa Bindings** 是一个很有用的 UI 技巧，在文章[《Cocoa Bindings on macOS》](https://www.raywenderlich.com/141297/cocoa-bindings-macos)中会有更详细的介绍。（如果有机会，这里也会尝试翻译一下这篇文章）

最后，给我们的 WordCountViewController 所属的 Window 加上一个 Storyboard ID，方便我们后续在代码里面找到它：

{% img center /uploads/Windows-in-macOS/WCControllerStoryboardId.png Setting Storyboard ID %}

> 这个 ID 是可以带有空格的，但是个人习惯不喜欢有空格，这张图是原文章里的。

## 模态窗口的显示与隐藏
有了前面的准备工作，这个可以显示字数和段落数的窗口已经是可用的了。接下来我们就找个合适的地方把它显示出来。

### 显示
打开 **ViewController.swift**，添加一个按钮事件：
```Swift
@IBAction func showWordCountWindow(_ sender: AnyObject) {
  
  // 1
  let storyboard = NSStoryboard(name: "Main", bundle: nil)
  let wordCountWindowController = storyboard.instantiateController(withIdentifier: "Word Count Window Controller") as! NSWindowController
  
  if let wordCountWindow = wordCountWindowController.window, let textStorage = text.textStorage {
    
    // 2
    let wordCountViewController = wordCountWindow.contentViewController as! WordCountViewController
    wordCountViewController.wordCount = textStorage.words.count
    wordCountViewController.paragraphCount = textStorage.paragraphs.count
    
    // 3
    let application = NSApplication.shared()
    application.runModal(for: wordCountWindow)
    // 4
    wordCountWindow.close()
  }
}
```

分解动作：
1. 通过刚刚设置的 Storyboard ID 初始化一个 WordCountWindowController（注意是 **WindowController**，不是我们自己创建的 **ViewController**）
2. 从我们的内容里取得字数和段落数，设置到 **WordCountViewController** 的属性里
3. 显示一个模态窗口
4. 关闭一个模态窗口

你可能会觉得奇怪，我们在显示之后立马就把它给关闭了，那还看什么？

事实上，当一个窗口以 `runModel(for:)` 的方式显示出来之后，应用会进入一个**模态过程**。这个动作相当于启动了一个阻塞的线程，启动方法调用之后的所有代码都会被阻塞住，只有在调用了 `stopModel` 停止 **模态过程** 后，代码才会继续执行。

### 隐藏
关闭模态窗口的代码需要由模态窗口本身去调用，因为其他地方都被阻塞住了呀。在 **WordCountViewController.swift** 里加上这段代码：
```Swift
@IBAction func dismissWordCountWindow(_ sender: NSButton) {
  let application = NSApplication.shared()
  application.stopModal()
}
```

这里只是让应用退出了 **模态过程**，真正关闭窗口的代码我们已经在上面写好了（就是 `close` 那个方法）

好咯，这个按钮事件怎么关联到那个 “OK” 按钮呢？留给你们自己去发现~

### 触发窗口的显示
显示的代码写好了，但是在哪里地方调用它好呢？不如在菜单里加一个选项吧。

增加菜单项跟之前添加视图没有扫描区别，菜单项的名字叫 “Menu Item”，直接拖到菜单里面去就好了，然后在右边栏里进行一下设置：

{% img center /uploads/Windows-in-macOS/add-word-count-window.png Add Menu Item %}

> Key Equivalent 设置的是快捷键

然后 Ctrl + 拖动，把这个菜单项跟我们写的方法联系起来：

{% img center /uploads/Windows-in-macOS/connect-menu.png Menu Action %}

完成了！编译运行！我们的统计功能就上线了！

## 接下来呢？
不知不觉，你其实已经学到了蛮多东西了：
* MVC 设计模式的一点应用
* 创建一个多窗口 app
* macOS app 的常见结构
* 通过 IB 和代码改变窗口的布局
* 将 UI 中的事件传递到响应链上
* 用模态窗口来展示附加信息
* …

不过这都只是 macOS app 的冰山一角。
如果你对窗口感兴趣，可以看看苹果的官方文章 [Window Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/WinPanel/Introduction.html)。如果想要继续深入研究 Mac 应用开发，则推荐 [Mac App Programming Guide](https://developer.apple.com/library/content/documentation/General/Conceptual/MOSXAppProgrammingGuide/Introduction/Introduction.html)。

这个 Demo 完整的代码在[这里](https://koenig-media.raywenderlich.com/uploads/2017/05/BabyScriptWithSave.zip)。这是原文里的链接，连保存的功能也实现好了，虽然没什么注释，但是代码很好懂。
