---
title: Swift 中的幽灵类型
date: 2022-04-20 10:22:36
tags:
- Swift
- 类型
- type system
- Translation
---

在日常开发中，数据内容的歧义可能是最常见的一种引发 Bug 的原因了。虽然 Swift 是一个强类型的语言，但是开发者自行构建的数据结构在使用时却很可能绕开了编译器的检测，这就带来了潜在的数据内容歧义问题。

<!-- more -->
> 原文链接：[Phantom types in Swift](https://www.swiftbysundell.com/articles/phantom-types-in-swift/)

## 结构漂亮，却意义模糊
举个例子，假设我们正在开发一个文本编辑器，它除了支持纯文字的内容之外，还提供了 HTML 编辑和预览 PDF 文件的功能。

为了能复用文件处理逻辑的代码，我们统一使用结构体 `Document` 来表示我们的文件，并且通过 `format` 属性来区分文件的类型：
```swift
struct Document {
    enum Format {
        case text
        case html
        case pdf
    }

    var format: Format
    var data: Data
    var modificationDate: Date
    var author: Author
}
```
尽管这样做避免了很多重复代码，而且枚举的使用也很好地区分了不同数据，但是当我们要针对特定文件类型进行操作时，这样的写法就会带来不小的歧义问题。

比如，我们要针对纯文本文件实现一个打开编辑器的 API，它会假设传进来的参数类型都是纯文本的：
```swift
func openTextEditor(for document: Document) {
    let text = String(decoding: document.data, as: UTF8.self)
    let editor = TextEditor(text: text)
    ...
}
```
这个方法在传入 HTML 文件时可能不会有什么大问题，但如果传进来了一个 PDF 文件就可能让我们的 App 崩溃了。

在针对特定格式的文件实现逻辑时，我们会不断遇到这样的问题。再打个比方，我们现在要实现一个 HTML 的编辑器了：
```swift
func openHTMLEditor(for document: Document) {
    // 就像上面那个方法那样，这个方法也假设了传进来的文件都是 HTML 格式的
    let parser = HTMLParser()
    let html = parser.parse(document.data)
    let editor = HTMLEditor(html: html)
    ...
}
```
那我们来尝试解决一下这个问题。首先想到的可能是用 `switch` 对传入的参数的 `format` 进行判断，然后再调用对应的方法。这种方式对纯文本和 HTML 文件很友好（因为我们在上面实现过它们的打开编辑器 API），然而对于 PDF 文件来说，我们可能就需要抛一个错误或者运行某条断言了：
```swift
func openEditor(for document: Document) {
    switch document.format {
    case .text:
        openTextEditor(for: document)
    case .html:
        openHTMLEditor(for: document)
    case .pdf:
        assertionFailure("Cannot edit PDF documents")
    }
}
```
上述方案依然有问题，因为它还是要求开发者自己去跟踪某种类型的文件和特定代码分支的关系，而且只有运行时才能发现某些分支里的潜在问题。

所以即便这个结构体看起来还比较优雅，但在实际使用时还是能感觉到不妥。

## 看来该轮到协议出场了
其中一种解决这个问题的方式是，把 `Document` 从一个实际的类型改为一个协议：
```swift
protocol Document {
    var data: Data { get }
    var modificationDate: Date { get }
    var author: Author { get }
}
```
这样改了之后，我们就可以针对不同类型的文件实现单独的类型：
```swift
struct TextDocument: Document {
    var data: Data
    var modificationDate: Date
    var author: Author
}
```
这种改法的优点在于，它让我们具备了同时对通用类型和特定类型进行操作的能力：
```swift
// 这个方法用于保存文件（不论类型），所以它的参数可以是任何实现了 Document 协议的对象：
func save(_ document: Document) {
    ...
}

// 现在我们可以做出这样的约束：打开文本编辑器的方法只接受纯文本类型的文件
func openTextEditor(for document: TextDocument) {
    ...
}
```
现在编译器就能帮我们检查方法调用的参数是否符合要求了，也就是说，我们最终把运行时对文件类型的判断提前到了编译时。这已经前进了一大步！

然而，这种做法也降低了我们的代码复用程度，因为我们现在是用协议来表示文件的，所以与文件相关的所有方法都需要我们在每一个具体的类型里写一遍。这甚至会波及我们未来可能支持的更多文件类型。

## 引入幽灵类型
如果能找到一种方法能在编译时进行类型检查，同时还能保障我们的代码复用，那就太棒了。其实我们前面写过的其中一行代码里就给了我们提示：
```swift
let text = String(decoding: document.data, as: UTF8.self)
```
在进行 `Data` 和 `String` 的转换时，我们把期望的字符串编码格式传了过去，但我们传递的不是值，而是这个类型本身的引用。我们再深挖一层，在 Swift 标准库里对 `UTF8` 的声明是这样的：
```swift
enum Unicode {
    enum UTF8 {}
    ...
}

typealias UTF8 = Unicode.UTF8
```
> 实际上，`UTF8` 枚举里有一个私有的 case，这是为了向后兼容 Swift3 而存在的。

这就是所谓的**幽灵类型**，即把类型当作一个标记来使用，而不会用它来声明对象。不过上面那些枚举里没有定义任何公开的 case，所以它们根本就不能被用来声明对象。

这对我们的文本类型困境有什么帮助呢？回到最初的结构体的实现方式，这次我们把 `format` 属性去掉，改成泛型：
```swift
struct Document<Format> {
    var data: Data
    var modificationDate: Date
    var author: Author
}
```
类似 `Unicode` 那样，我们会定义一个 `DocumentFormat` 枚举，用它来充当命名空间的作用，然后分别定义三个没有 case 的枚举来表示文件类型：
```swift
enum DocumentFormat {
    enum Text {}
    enum HTML {}
    enum PDF {}
}
```
到目前为止，我们都没有用到协议。`Format` 只是用来充当一个运行时标记，任何类型都能约束我们的 `Document`。接下来，我们就能把之前针对特定文件类型所写的 API 改成这样：
```swift
func openTextEditor(for document: Document<DocumentFormat.Text>) {
    ...
}

func openHTMLEditor(for document: Document<DocumentFormat.HTML>) {
    ...
}

func openPreview(for document: Document<DocumentFormat.PDF>) {
    ...
}
```
当然了，不指定任何约束也是可以的，比如之前用于保存文件的方法就可以这样写：
```swift
func save<F>(_ document: Document<F>) {
    ...
}
```
我们还可以进一步给不同类型定义一个别名，就像 `UTF8` 那样：
```swift
typealias TextDocument = Document<DocumentFormat.Text>
typealias HTMLDocument = Document<DocumentFormat.HTML>
typealias PDFDocument = Document<DocumentFormat.PDF>
```
幽灵类型在我们需要给特定文件类型写扩展的时候也很好用，比如我们要给纯文本文件加一个设置字体的方法：
```swift
extension Document where Format == DocumentFormat.Text {
    func makeAttributedString(withFont font: UIFont) -> NSAttributedString {
        let string = String(decoding: data, as: UTF8.self)

        return NSAttributedString(string: string, attributes: [
            .font: font
        ])
    }
}
```
而且，因为幽灵类型只是普通的类型，所以我们还能让它们遵循其他的协议。比如在想要实现打印功能的时候，我们可以让 `DocumentFormat` 遵循 `Printable` 协议，在这个基础下再实现我们自己的代码。

## 一种标准化的模式
尽管幽灵类型看起来不太像 Swift 本身的语法，然而，虽然 Swift 不像其他纯函数式语言（比如 Haskell）那样把幽灵类型当作语言里的一等公民来对待，但这种模式已经在标准库和其他苹果平台的 SDK 里被广泛使用了。

比方说 `Foundation` 里的 `Measurement` API 用就幽灵类型来确保类型安全：
```swift
let meters = Measurement<UnitLength>(value: 5, unit: .meters)
let degrees = Measurement<UnitAngle>(value: 90, unit: .degrees)
```
这样就严格区分了两种计量单位，避免了开发者在一个需要长度单位的地方用了角度，就像我们在上文中区分文件类型那样。

## 总结
幽灵类型是一种能让我们更好利用类型机制来区分变量的神奇技术。虽然幽灵类型会让代码看起来更冗长，而且泛型的使用看起来也更复杂，但是它却能帮我们把运行时才能发现的问题提前到了编译期间，让编译器能发挥出更大的作用。

不过，就像我们之前认为 `Document` 结构体看起来很美好那样，幽灵类型如果用错了地方，也可能是杀鸡用牛刀。也许本来很简单的流程，会被幽灵类型搞得很复杂。

到头来，选择合适趁手的工具才是最重要的。