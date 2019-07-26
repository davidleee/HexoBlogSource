---
title: SwiftUI 系列教程（3）
date: 2019-07-25 16:41:24
tags:
- Swift
- SwiftUI
- MVC
- React
- Redux
- Property Wrappers
- State
- Binding
---

通过前两篇文章（SwiftUI 系列教程 {% post_link swiftui-serial-tutorial-1 （1） %} 和 {% post_link swiftui-serial-tutorial-2 （2） %}），我们已经看到了 SwiftUI 是怎么运作的了，对于常规的界面元素来说，使用 SwiftUI 确实能带来不小的生产力提升。但是在前面的例子里，我们用到的数据全都是写死的，这跟复杂多变的真实需求可不大一样。这篇文章我们就来了解一下，SwiftUI 里用到的全新的数据流模型。

> 相比起前两篇实操文，这篇文章可能会比较干，请大家看文章之前先访问一下饮水机。  

<!--more-->

## 先看看别人家是怎么做的

### 老东家

学习过斯坦福公开课的小伙伴们应该对下面这张图片很有印象了：
![](/uploads/swiftui-serial-tutorial-3/mvc1.png)

这是 iOS 自出道以来就非常推崇的，可谓是“官方建议”的数据流模型，也就是大家都熟知的 MVC 模式。

从 GUI 开始兴起以来，基于职责分离的思想，工程师们慢慢把管理用户界面的 View 和管理用户数据的 Model 给区分了开来；而从 Smalltalk 的某个版本开始，为了进一步降低图形应用程序的管理难度，设计出了 MVC 模式。MVC 的出现主要是为了解决这两个问题：

1. 如何管理响应用户操作的业务逻辑
2. View 如何同步 Model 的变化

解决这两个问题也是大多数为现代图形界面应用程序而诞生的设计模式们的目标，比如 MVVM、MVP。

### 隔壁前端家

因为有着一段不长不短的 React Native 开发经历（目前还在做着），所以从这个角度看过去，比较成熟的方案是 Redux + React Redux。

Redux 是专为 JavaScript 软件打造的一个可预测状态容器。听起来很厉害的样子，其实主要是做了三个事情：

1. 把 Model 层的数据统一放到一个地方
2. 约定只能通过特定的手段去改变数据，除此之外，数据就是只读的
3. 约定改变数据的操作必须是纯函数

这样做了之后，我们就可以确定这个数据源是可以真实反应我们的应用状态的，所以叫做“可预测状态容器（Predictable State Container）”。

然后 React Redux 就好理解了，它的任务是建立一套机制，让上述的状态一一绑定到视图上，实现一条双向更新的通道。

> Redux 和 React Redux 都是 Redux 官方出品，所以质量还是比较有保证的。下面这两篇文章应该可以给大家技术选型的时候提供一些支持：  
> [Motivation · Redux](https://redux.js.org/introduction/motivation)  
> [Why Use React Redux? · React Redux](https://react-redux.js.org/introduction/why-use-react-redux)  

## Swift 的实现

如果你觉得 SwiftUI 在构造界面时用到的声明式语法跟 JSX 的相似度很高的话，那在介绍完它的数据绑定逻辑之后，你肯定会再一次把它拿来跟 JavaScript 做比较了。

SwiftUI 中引入了一个关键字 `@State` 来作为数据绑定的标识。当一个被绑定的数据被改变时，相关联的视图会重新计算它自己的 `body` 内容；反过来，当视图主动去改变绑定在数据上的属性时，这个数据也会随之变化。这种双向绑定的机制就像 JSX + Redux + React Redux 的组合拳，只不过 SwiftUI 自己就把这些事情给做了。

但是，凭什么 SwiftUI 用几个关键字就实现了别人整整两个开源库的功能？其实这得益于 Swift 5.1 的新功能——属性包装（Property Wrappers）。

### 深挖一点点

在 2019 年3月份的时候，Swift 核心团队里的成员已经透露出了一个作用类似于 `lazy` 关键字的新功能，那个时候它被称为“属性代理”（Property Delegates）。

举个例子，延迟初始化可谓是编程里的一种美德，在 Swift 的世界里，除了直接用 `lazy`，我们也可以用一个私有属性加上一个作为访问器的 Computed Property 来实现：

```swift
private var _lazyProp: Prop?
var lazyProp: Prop {
    get {
        if let value = _lazyProp { return value }
        let initialValue = ... // 初始化数据
        _lazyProp = initialValue
        return initialValue
    }
    set {
        _lazyProp = newValue
    }
}
```

如今，Property Wrappers 为我们提供了第三条路可走，不仅如此，它还承诺会为开发者们提供了一种实现类似 `lazy` 关键字用法的途径。

在 SwiftUI 的功能提议 [SE-0258](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md) 里可以看到，Property Wrappers 的主要目的就是为了避免开发者重复写出上面那种固定模式的代码。既然这种写法是比较固定的，那么就应该定义出一种机制，来把各种固定写法定义成一个个的工具库。

### 官方的解决方案

还是用 `lazy` 的例子，怎么用 Property Wrappers 来实现一个同样的功能呢？比如实现一个作用相同的 `@Lazy` 属性？

官方给出的解决方案是这样的：

```swift
@propertyWrapper
enum Lazy<Value> {
  case uninitialized(() -> Value)
  case initialized(Value)

  init(wrappedValue: @autoclosure @escaping () -> Value) {
    self = .uninitialized(wrappedValue)
  }

  var wrappedValue: Value {
    mutating get {
      switch self {
      case .uninitialized(let initializer):
        let value = initializer()
        self = .initialized(value)
        return value
      case .initialized(let value):
        return value
      }
    }
    set {
      self = .initialized(newValue)
    }
  }
}
```

如此一来，下面这种变量声明

```swift
@Lazy var foo = 123
```

就会被展开成这样：

```swift
private var _foo: Lazy<Int> = Lazy<Int>(wrappedValue: 123) // 1
// 2
var foo: Int {
  get { return _foo.wrappedValue }
  set { _foo.wrappedValue = newValue }
}
```

1. 调用了 `Lazy` 的 `init` 方法来进行初始化一个私有变量，它的类型是 `Lazy<Int>`
2. 通过把原变量包装成一个 Computed Value 的方式来对接 `wrappedValue` 里提供的真正的逻辑实现

不仅如此，既然 `@Lazy` 是一个 `enum` ，那它本身就可以定义五花八门的公开方法，而每一个被 `@propertyWrapper` 标记的类型都可以通过定义一个 `projectedValue` 属性来实现一些骚操作：

```
@propertyWrapper
enum Lazy<Value> {
    // ... 内容跟前面一样 ...

    // 定义一个重置的方法，把 @Lazy 标记的变量还原为一个不同的初始值
    public mutating func reset(_ newValue:  @autoclosure @escaping () -> Value) {
        self = .uninitialized(newValue)
    }
    
    public var projectedValue: Self {
        get { self }
        set { self = newValue }
    }
}
```

声明了 `projectedValue` 之后 ，我们就自动获得了一个带 `$` 符号的分身用来访问我们 `projectedValue` 的 getter，从而调用到里面的方法：

```swift
@Lazy var foo = 123
$foo.reset(456)
```

上面那句声明变量的语句，会展开成这样：

```swift
private var _foo: Lazy<Int> = Lazy<Int>(wrappedValue: 123)
var foo: Int {
  get { return _foo.wrappedValue }
  set { _foo.wrappedValue = newValue }
}

// ... 上面跟原来一样，下面是添加了 projectedValue 之后新增的 ...

var $foo: Lazy<Int> {
  get { _foo.projectedValue }
  set { _foo.projectedValue = newValue }
}
```

> 其实给 `Lazy` 添加 `extension` 也可以达到类似的目的，不过这时候的方法调用就要通过 `_foo` 来进行，而不是 `$foo` 了  

## 说回 SwiftUI

SwiftUI 的数据流模型是基于下面两点原则来构建的：

1. Data Access as a Dependency
2. Source of Truth

我们展开来看：

### Data Access as a Dependency

在多数情况下，我们的视图是需要根据某些状态来动态变化显示样式的，比如对于 `Switch` 来说，改变它的 `on` 属性可以让它显示当前的开关状态。

对于这种情况，`on` 属性就应该作为 `Switch` 的依赖而存在，否则这个控件除了长得好看就一无是处了。所以在 SwiftUI 里，属性会被描述为视图的依赖，这意味着我们的注意力可以从属性和视图的关联里抽身出来，集中在建立更好的用户体验上。

### Source of Truth

同一组视图里的数据都是来自于同一个数据源的（甚至整个应用的数据都来自于同一个数据源，Redux 就是这么做的）。

对于开发者来说，数据源的不唯一意味着视图状态的不唯一。可以想象，位于同一视图层级的两个视图要共用某些参数时，数据来源的不唯一会为编程带来多大的麻烦。
![](/uploads/swiftui-serial-tutorial-3/D1B04594-7FAA-4391-AC05-CBC2BF839DD8.png)

SwiftUI 对这种情况的处理是，让父视图作为子视图的唯一数据源：
![](/uploads/swiftui-serial-tutorial-3/4CDDC5EC-0093-46EE-A17A-F38972875AF8.png)

> 做过前端 UI 开发的童鞋们应该很熟悉这套操作了，这就是把 State 上提成 Props 嘛，目的是让子视图尽可能的简单，最好的情况下子视图本身应该是无状态的。  

于是我们可以得出，基于这两个原则来实现的数据流模型已经完全不同于我们以往的理解，我们需要重新定义我们所认识的视图：**视图要体现的是一个个独立的状态，而不是一系列连续的事件** 。

## 实践出真知

说了这么多，我们来实际改造一段代码试试。

假设我们要实现一个播放器的播放按钮，需求是它要能反应播放状态：

1. 没有音频播放时，它表现为播放按钮
2. 在有音频播放时，它表现为暂停按钮

我们通过给按钮设置不同的图片来区分这两个状态，按照之前的知识，我们能轻松写下这样的代码：

```swift
struct PlayerView: View {
    private var isPlaying: Bool = false
    var body: some View {
        Button(action: {
            self.isPlaying = !self.isPlaying
        }) {
            self.isPlaying ? Text("Pause") : Text("Play")
        }
    }
}
```

等等，我们这个 `PlayerControl` 是 `struct` 类型的，不能这样直接改变属性的值：
![](/uploads/swiftui-serial-tutorial-3/29C7C479-1EBA-47FD-B9C5-1037A6FC8E17.png)

其中一个安抚编译器的方法是，用一个临时变量来替代 `self`，我们顺便把布尔值取反的操作也简化一下：

```swift
struct PlayerView: View {
    private var isPlaying: Bool = false
    var body: some View {
        Button(action: {
            var tempSelf = self // 安抚编译器
            tempSelf.isPlaying.toggle() // Implemented in Swift 4.2
        }) {
            self.isPlaying ? Text("Pause") : Text("Play")
        }
    }
}

```

好了，这样一改，那句临时变量赋值语句就成为了夜空中最亮的星，怎么看怎么别扭…

那接下来就轮到 SwiftUI 里定义的 Property Wrappers 出场了，这段代码可以改写成这样：

```swift
struct PlayerView: View {
    @State private var isPlaying: Bool = false // 加上 `@State` 标记
    var body: some View {
        Button(action: {
            self.isPlaying.toggle() // 可以直接改动 `self` 里的变量了
        }) {
            self.isPlaying ? Text("Pause") : Text("Play")
        }
    }
}

```

似乎…也没太大变化啊，代码量也不见少，只不过是省掉了临时变量了？
这是因为，这样的写法还是根据我们的惯常思维来走，回忆一下上面讲到的 SwiftUI 数据模型原则：

1. 以依赖的形式访问数据
2. 单一数据源

例子里的写法貌似符合了“单一数据源”，但是却把“以依赖的形式访问数据”晾在了一边。我们来进一步改造这个例子：

```swift
struct PlayerView: View {
    @State private var isPlaying: Bool = false
    var body: some View {
        PlayButton(isPlaying: $isPlaying) // 1
    }
}

struct PlayButton: View {
    @Binding var isPlaying: Bool // 2
    var body: some View {
        Button(action: {
            self.isPlaying.toggle()
        }) {
            Text(isPlaying ? "Pause" : "Play") // 3
        }
    }
}

```

1. 使用 `@State` 标记的变量会自动生成一个以 `$` 作为前缀的新变量，这个新变量本质上是一个 Computed Value，实现了双向绑定的机制，也就是说当 `PlayButton` 内部改变了这个变量之后，`PlayerView` 里的 `isPlaying` 也会发生相同的变化
2. 通过声明变量为 `@Binding`，我们就告诉了编译器这个变量是从外部传入的可以被绑定的参数，相当于 React 里的 `Props` 声明
3. 我们将 `isPlaying` 作为 `Text` 的依赖来使用，对于 `PlayButton` 来说，变量的声明和使用都是在一个结构体里面完成的，这就意味着这个视图与 `PlayerView` 是解偶的

> `@Binding` 具有以下两种特性：
>
> 1. 在不持有变量的前提下进行变量的读写  
> 2. 可以从 `@State` 变量中推导出来  

现在回忆一下，在 SwiftUI 之前我们是怎么实现类似逻辑的？在不知不觉中，我们已经舍弃了 ViewController，让视图直接成为了数据的载体。甚至可以说，在 SwiftUI 里，视图就是为数据服务的。

> `@State` 标记的属性一旦变化，会引起依赖它的视图、这个视图的父视图和它的同级视图一起做**必要的**变化。为什么要强调**必要的**？因为相较于繁重的渲染工作来说，对声明式语法描述出来的数据结构进行比较并不消耗什么性能，SwiftUI 会在重新渲染前对视图状态进行比较，尽可能地去避免无谓的绘制，所以不需要担心性能的问题。  
> 类似于 `React.PureComponent` 提供的逻辑。  

## 小结

基于这套数据模型实现出来的数据流可以用下面这张图片来表示：
![](/uploads/swiftui-serial-tutorial-3/870E1C27-D2D8-41AD-BBF2-66AABA4BD04A.png)
要知道，Action 不只可以来自于用户交互，它还可以来自我们自己实现的触发器、消息推送等等，而不管来源是什么，我们实现的逻辑都可以理解并做出同样的处理。

这样的数据流模式确保了数据的流动永远是单向的，而 State 在这里充当了视图变化的唯一数据源，让视图的更新是可预测和易懂的。

> 当然，`@State` 也有它的局限性，比如它无法正确处理我们自己定义的对象类型的属性变化，所以我们还需要 BindableObject 协议来从旁辅助，这里就不继续展开了。  

了解了视图基础和数据流模型，相信大家都已经看到了 SwiftUI 的魅力，余下的细节就需要各位开发者在实际应用中发掘了。下一讲就让我们来把这个魅力继续扩大，一起来实际看看 SwiftUI 还给我们的开发带来了哪些好处。

## 参考文章

- [界面之下：还原真实的 MVC、MVP、MVVM 模式](https://www.linuxidc.com/Linux/2015-10/124622.htm)
- [Motivation · Redux](https://redux.js.org/introduction/motivation)
- [Why Use React Redux? · React Redux](https://react-redux.js.org/introduction/why-use-react-redux)
- [State and Data Flow | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/state_and_data_flow)
- [Swift Property Wrappers - NSHipster](https://nshipster.com/propertywrapper/)
- [swift-evolution/0258-property-wrappers.md at master · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md)
- [Data Flow Through SwiftUI - WWDC 2019 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2019/226/)