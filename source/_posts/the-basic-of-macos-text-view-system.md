---
title: 关于 macOS 输入框你需要了解的一些基础
date: 2018-09-11 14:48:40
tags:
- macOS
- AppKit
- Cocoa
- NSTextField
- NSTextView
- Text System
- Field Editor
---

> 这是 macOS 开发系列的第三篇文章 —— 文本输入系统基础。
>
> 电脑的文字编辑功能比手机上的强大（难搞）真不是吹！  

<!--more-->

## 认识输入框
> The Macintosh operating system has provided sophisticated text handling and typesetting capabilities from its beginning. In fact, these features sparked the desktop publishing revolution. —— Apple  

在 macOS 的世界里，要显示或者编辑文字主要会用到两个控件，一个是 [NSTextView](https://developer.apple.com/documentation/appkit/nstextview)，另一个是 [NSTextField](https://developer.apple.com/documentation/appkit/nstextfield)。

> 虽然控件库里面有个叫 Label 的东西，但是拖出来之后就会发现，它其实也是一个  [NSTextField](https://developer.apple.com/documentation/appkit/nstextfield)  

它们的继承关系是这样的：
![](/uploads/the-basic-of-macos-text-view-system/7004648D-1B33-476E-8788-8D894D8B08E9.png)

NSTextView 是苹果花了大心血打造的一个“满足几乎所有显示和管理文字需求”的一个控件，也是 macOS 引以为傲的文字编辑系统的主心骨；NSTextField 则相当于一个简化版的 NSTextView，在大多数情况下它可以满足字数较少的输入需求。

正如我们在 iOS 开发里常做的那样，在需要用户输入的地方，通常是直接展示一个 Text 相关的控件，然后通过 delegate 方法来控制输入的内容；又或者继承一个官方的控件，然后在内部直接实现想要的内容控制逻辑。

在 macOS 上，这个流程也是大致相同的。

> 以前写过一篇简单介绍 NSTextView 用法的文章（ {% post_link Windows-in-macOS 20分钟手把手教你写 macOS 文本编辑器 %}），等不及的童鞋们可以在这篇文章里过过瘾。  

## 文本输入的幕后玩家
虽然从使用上来看，跟开发者直接打交道的就是 NSTextView 和 NSTextField 这两个类，最多再加上它们带着的一些协议/代理，但是继续往深了看，会发现有一个未知的世界在支撑这这一切。

### Field Editor
macOS 上有一个叫 Field Editor 的概念。

在输入框获得焦点的时候，系统会实例化一个 NSTextView 作为 Field Editor，并把它作为 first responder 插入到这个输入框的事件响应链当中。如此一来，Field Editor 会负责处理所有的用户输入事件，在这个过程中，获得焦点的输入框会作为 Field Editor 的代理，以便对文本的内容进行控制处理。

![](/uploads/the-basic-of-macos-text-view-system/field_editor_2x.png)

> 这就是为什么 NSWindow 的 `firstResponder` 返回的是一个不可见的对象，而不是我们获取了焦点的输入框，因为这个对象就是上面说的 Field Editor。  

Field Editor 是同一个窗口里所有输入框共用的，所以在我们用 Tab 键切换输入框的时候，Field Editor 就会切换事件响应的对象。另一方面，这个机制也确保了同一个窗口中只能有一个控件去响应用户输入事件。不过，我们也可以实现自定义的 Field Editor 来推翻上面说的这些功能。

> 虽然 Field Editor 一般会是一个 NSTextView 的实例，但是它们对 Tab 和 Return 的事件处理是不同的。对于 Field Editor 来说，这两个键盘事件是“结束编辑”的意思。  

### NSTextInputContext
这是外界和文本输入系统之间沟通的桥梁。

在用户进行输入的时候，`keyDown` 消息会被传递到获取了焦点的输入框里，输入框接下来会调用 [NSTextInputContext](https://developer.apple.com/documentation/appkit/nstextinputcontext) 的 `handleEvent` 方法，以便让 NSTextInputContext 告诉自己需要怎么处理这个用户事件。而作为响应，NSTextInputContext 会把处理结果通过 [NSTextInputClient](https://developer.apple.com/documentation/appkit/nstextinputclient) 这个协议告知输入框。

![](/uploads/the-basic-of-macos-text-view-system/F200783C-8569-4396-BC36-4DAD92E810DF.png)

从上图可以看到，NSTextInputContext 会跟一个叫 Key-bindings dictionary 的字典保持密切联系。这个字典默认来自于 AppKit 内部的一个文件（*/System/Library/Frameworks/AppKit.framework/Resources/StandardKeyBinding.dict*），里面保存了系统默认定义好的所有快捷键，只有当用户输入的值在这个字典里找不到匹配的键值对时，这次输入才会作为普通字符回调给输入框，否则 NSTextInputContext 就会在这里把这次输入拦截下来，并让输入框执行相应的特殊操作。

> 我们可以通过修改 *~/Library/KeyBindings/DefaultKeyBinding.dict* 里的值来覆盖默认的快捷键  

## 小结
到此为止，macOS 文本编辑系统的一些内在机理已经了解地差不多了。在往后使用 NSTextView 和 NSTextField 的过程中，碰到一些不明觉厉的问题也能有个大概的问题排查方向了。（不过也仅限于“大概方向”了）

好吧，我知道这篇文章偏理论了一些，大多数情况下也不会用到。更多详细的说明可以在文章里提供的各种链接上找到。

在后续的文章里，还会讲到 NSTextField 实际应用上的一些内容。NSTextView 因为暂时没有用上，所以可能会等有机会研究清楚些再讲咯。

## 参考文章
[About the Cocoa Text System](https://developer.apple.com/library/archive/documentation/TextFonts/Conceptual/CocoaTextArchitecture/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009459-CH1-SW1)