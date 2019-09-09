---
title: SwiftUI 系列教程（1）—— 初识 SwiftUI
date: 2019-06-12 09:07:19
tags:
- Swift
- SwiftUI
- Catalina
- VStack
- HStack
---

> 可能是全网最早的 SwiftUI 中文教程？

这篇文章来源于苹果官方的教程，相当于是我自己学习过程的一个记录。这个系列教程会跟着官方教程构造一个新的项目，还会加入一些 WWDC 的东西作为补充，可能偶尔会有一些自由发挥的部分。（不过我这里是做不出官方教程那种酷炫的动画了…）

<!-- more -->

## 全新的编码体验！

目前（2019.6.5）要体验到这个新东西，需要用到 **Xcode 11 beta**，而如果要体验新的预览机制和对画布上的预览进行操作，还需要把系统更新到 **macOS Catalina 10.15 beta**， 看来这个系列的新功能提供了系统层面的支持。

> 话说回来，下了 iPadOS 之后用 iTunes 死活升不上去，报错说 macOS 有软件要更新，但总更新失败，最后下载 Xcode 11 beta 让它跑完 install components 那一步就可以成功升级 iPad 了…告诉我不是一个人…  

让我们先用新鲜热辣的 Xcode 11 beta 创建一个项目吧！

前面的步骤身为 iOS 开发的同学们应该很熟悉了，这里要创建的是 Xcode project -> Single View App（playground 我还没试过，也许也被“强化”过了？），然后在取名字的那一步稍作停留，瞧瞧我在打勾区发现了什么：**Use SwiftUI**！当然选择钩上它！

![](/uploads/swiftui-serial-tutorial-1/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72019-06-10%E4%B8%8B%E5%8D%889.55.44.png)

完事之后我们就可以看到熟悉又带点陌生的编辑器界面了。点开默认提供给我们的 *ContentView.swift*，可以看到里面已经写好了两个 `struct`：
![](/uploads/swiftui-serial-tutorial-1/F31CF147-5F01-422B-8ECB-C9AEFC274150.png)

---

先打个岔，看看编辑区域右边新加入的 Minimap 界面，这虽然是一个在代码编辑器中比较常见的功能，但苹果一出手，还是给改进了一番：

1. 当鼠标在 Minimap 移动时，会像上图那样高亮并在左边把对应方法或变量名凸显出来，此时点击任意一块高亮区域都可以跳转到对应的代码位置
2. 鼠标放在 Minimap 上时，按住 Command 键，所有的方法和变量名都会凸显出来（上图其实是按下了 Command 后的状态），重要的代码结构一目了然

> 苹果的每次大更新，着重宣传的主要变化，在我都尝试一遍之后就没什么感觉了，反而是这些小地方特别打动我  

---

言归正传，看回代码。

这两个是 SwiftUI 默认提供的结构体，其中遵循 `View` 协议的那个定义了我们的界面内容和布局，而遵循 `PreviewProvider` 协议的那个则负责处理这个界面的预览。

那什么是预览？我们可以先把旁边的所谓预览窗口跑起来，点击右上角的 *Resume*（这个版本的 Xcode 默认会显示下图这个界面，没有的话，也可以在 *Editor* 里面打开它）：
![](/uploads/swiftui-serial-tutorial-1/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72019-06-10%E4%B8%8B%E5%8D%8810.22.22.png)

这时，Xcode 会把我们的项目运行起来，就跟平时点 *Run* 跑到模拟器上一样。有所不同的是，这次的模拟器直接显示成了 Xcode 的一个子界面，我们甚至可以直接操作这个模拟器里的视图，就像操作 xib 和 storyboard 一样！

![](/uploads/swiftui-serial-tutorial-1/38B1BE43-FECA-471C-A845-FBFB8E5FA2C0.png)

> 仔细看，点选了界面上的文案之后，左侧编辑器里相应的视图代码也被高亮了起来，是不是有一种根据按钮事件找 IBAction 代码的感觉？  

尝试修改一下代码里 `Text` 中的内容，会发现模拟器里的显示也实时更新了！对这个更加强大的模拟器，苹果给它起了个名字叫 **画布（Canvas）**。

## 自定义文本视图

那既然 **画布** 有着跟 Storyboard 相似的体验，那是不是意味着我们也可以直接改动界面上的元素？答案是肯定的，所有功能都隐藏在 Command + 左键点击里：
![](/uploads/swiftui-serial-tutorial-1/AA6AA015-7832-46DD-83E7-063BC1631D83.png)

点击后会出现一个内容丰富的弹出框，这些操作会根据点击的视图不同而不同。

我们可以通过 *Inspect…* 来修改视图的一些基本元素：
![](/uploads/swiftui-serial-tutorial-1/D0D9CEBC-7A12-4479-9541-34C80D177817.png)

> 图上应该能看出来，这个弹出框是可以滚动的，它已经可以取代原来我们常用的 Attributes Inspector。实际上，如果你在这个时候点开右侧边栏，会发现 Attributes Inspector 的内容跟这里是完全一致的  

我们来把它的字体改为 *Large Title*，可以看到代码部分也跟着界面一起改变了：
![](/uploads/swiftui-serial-tutorial-1/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72019-06-10%E4%B8%8B%E5%8D%8810.38.09.png)

按照这个规律，我们通过手写代码来改个字体颜色试试：
![](/uploads/swiftui-serial-tutorial-1/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72019-06-10%E4%B8%8B%E5%8D%8810.40.22.png)

这种链式调用的语法是不是跟用 OC 实现的 AutoLayout 开源库 [Masonry]( https://github.com/SnapKit/Masonry) 很像呢？苹果把这些方法叫做 *修饰器(modifiers)*，它们会在旧视图的基础上构造一个新视图返回出来，这使得上述的链式调用成为可能。

> 如果我们再从刚才的 *Inspect…* 里把字体颜色改为 *inherited*，就会发现 Xcode 把我们刚加上的 `.color(.red)` 又给删掉了，这波操作让写代码有意思了不少啊。  

## 把视图都叠起来吧

在前面的内容里，我们通过 SwiftUI 来描述了我们想要的视图样式，但这只是单一的视图。当视图多起来的时候，我们可以通过 *stacks* 把视图在竖直方向、水平方向或从前往后组合起来。

注意力继续回到我们的新朋友画布上，这次我们加快一点速度，先对刚刚的文本进行 *Embed in VStack* 的操作：
![](/uploads/swiftui-serial-tutorial-1/32BB01AF-9406-41A2-8CA6-61F10F511D3A.png)

然后通过 Command + shift + L 调出视图库界面：
![](/uploads/swiftui-serial-tutorial-1/73CBCF80-9712-48AA-9049-AD8641FC0269.png)

从里面拖一个 `Text` 到编辑器里（对，没错，就是编辑器，它会自动变成代码），放到我们之前操作的 `Text` 之下。现在我们的代码应该变成这个样子了：
![](/uploads/swiftui-serial-tutorial-1/54973458-B739-467A-A6E9-B28081FEC640.png)

> 从视图库拖组件出来这一步，我们有两种选择：一种是拖到代码里，另一种是拖到 Canvas 上。大家可以尝试一下拖到界面上会有什么样的效果。  

稍后我们再回过头来看这个 `VStack` 是什么。现在让我们继续快进，加入两个没见过的新组件 `HStack` 和 `Spacer`，通过给 `VStack` 加上参数来进行布局，还要再通过修饰器美化一下界面，最后 `ContentView` 的内容应该是这个样子的：

```swift
struct ContentView : View {
    var body: some View {
        VStack(alignment: .leading) {
            Text(“Hello SwiftUI”)
                .font(.largeTitle)
                HStack {
                    Text(“First Description”)
                        .font(.subheadline)
                    Spacer()
                    Text(“Second Description”)
                        .font(.subheadline)
                }
        }
        .padding()
    }
}
```

相应的，我们的界面也成了这样：
![](/uploads/swiftui-serial-tutorial-1/33A162A9-B3E2-4938-80DD-A7B752AC2098.png)

## 幕后发生了什么？

刚才的代码里，起到容器作用的是 `VStack` 和 `HStack`，顾名思义，它们分别是竖直方向上和水平方向上的层叠视图（Vertical & Horizontal），用法跟我们早就认识的 `StackView` 相同。

到目前为止，有过前端开发经验的同学们应该能发现，这不就是 JSX 的语法吗？

我们知道，在实现一个新界面的时候，通常包含着“用基本组件就能实现”的常规部分和“要把奇技淫巧发挥到极致”的出彩部分。SwiftUI 的出现就是为了简化常规 UI 的开发过程，让开发者能够把精力都放在激动人心的部分。

> —— 摘自 WWDC

为了达到这个目的，一个首要的改变就是：**把命令式的视图逻辑转变为声明式的**。这样做的好处在于：

1. 提高了组件的复用性
2. 代码更清晰易懂，在编写的过程中也更符合直觉
3. 隐藏了背后复杂的部分，让 SwiftUI 帮我们料理好一切

这样转变之后，整个视图层级的代码看起来就清晰了许多（对比一下用单纯的 Swift 来实现会有多少代码），然而这种转变的背后其实都是我们熟悉的 Swift 语法。举个例子，`VStack` 本身就是一个 `View`，它的实现是这样的：

```swift
public struct VStack<Content: View> : View {
    public init(
        alignment: HorizontalAlignment = .center,
        spacing: Length? = nil,
        @ViewBuilder content: () -> Content
    )
}
```

其中，`alignment` 和 `spacing` 是两个布局用的属性，我们在之前的例子里就设置了 `VStack(alignment: .leading)`，这可以让内部的元素左对齐；而 `content` 属性则是一个闭包，它将返回另外一个视图，里面包含了所有要显示在 `VStack` 里的子视图。

`VStack` 和 `HStack` 在 SwiftUI 里被用到时候，其实就是调用了它的构造方法，因为前两个参数要么是有默认值的，要么是可选的，所以只需要关注最后一个闭包参数；而这个闭包参数作为参数列表里的最后一员，可以写作结尾闭包的样子，于是就有了我们上面例子中的写法。

剩下的 `Spacer()` 和 `.padding()` 就只不过是 SwiftUI 提供给我们的又一个常规组件和修饰器而已了。

## 小结

写到这里，仅仅涵盖了官方教程第一章里的前3部分，外加一点来自 WWDC 视频里的内容。我还会继续补充，努力把整个教程都覆盖掉。

不过这就足以让我们看到它的好玩之处了，至少写了一段时间 React Native 的我，看到这似曾相识的语法，着实感觉欣喜。苹果在为开发者打造工具上下的功夫，恐怕在历史所有科技型企业上也是数一数二的了。

## 参考链接

[SwiftUI | Creating and Combining Views](https://developer.apple.com/tutorials/swiftui/creating-and-combining-views)
[SwiftUI Essentials - WWDC 2019 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2019/216/)