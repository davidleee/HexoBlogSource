---
title: SwiftUI 系列教程（4）—— UIKit 老相好在 SwiftUI 下的实现
date: 2019-08-15 08:44:32
tags:
- Swift
- SwiftUI
- List
- NavigationView
- NavigationButton
- Form
- Picker
- Toggle
- Section
- Stepper
---

咱们最有意思的第四篇 SwiftUI 教程来啦！为什么说是“最有意思”的呢？因为按照约定，在这篇文章里我们会一起来看看用 SwiftUI 开发界面的快捷便利体现在什么地方。相信这会让许多苹果开发者们耳目一新。

> 信了苹果教之后，每次有什么更新，我最期待的都是隐藏在大功能下的小细节，不知道有多少人跟我一样？  

<!--more-->

## UITableView & UITableViewCell

首先要说的是 UI 中最常见的列表：在萌新们刚开始学习 iOS 开发的时候，列表的实现也许就是其中一个劝退点。虽然 UIKit 的 API 已经做了比较友好的封装，但在这个前提下，开发者还必须要了解 UITableViewDelegate、UITableViewDataSource、Cell 与列表的关系，如果想要构造一个高性能的列表，还需要了解 Cell 的重用等等等等。

那么在 SwiftUI 里构造一个列表的操作是怎样的呢？我们分三步走：

1. 单个列表项的布局
2. 准备数据
3. 全部塞到一个列表里显示出来

### 列表项布局

假设我们就构造一个最基础的列表项布局好了，一种 UIKit 直接就支持的显示模式：图片 + 标题 + 详情描述。

经过前面几篇文章的训练，我们对这种类型的 UI 构造应该熟门熟路了：

```swift
HStack() {
	Image(systemName: "photo") // 1
	VStack(alignment: .leading) {
		Text("Title")
		Text("Detail description")
			.font(.subheadline)
			.foregroundColor(.secondary) // 2
	}
}
```

1. 我们这里用了一张名为“photo”的图片，这个图片是 iOS 13 送给我们的，就是那个常见的图片占位符图标
2. UIKit 默认的详情描述用的是一种灰色的偏小一点的字体，我们直接用系统提供的属性就能模仿出来

完成这一步之后，预览会是这个样子的：
![](/uploads/swiftui-serial-tutorial-4/B1BE86B5-7B16-46DF-8DE8-C6AD6B23565D.png)

OK，学习完前面三篇文章之后，这里并没有什么新鲜的知识。下一步是准备数据。

### 准备数据

这一步跟直接用 Swift 来实现没有太多区别，我就直接用 WWDC 视频里的数据结构了：

```swift
import Foundation

struct Room {
    var id = UUID()
    var name: String
    var capacity: Int
    var hasVideo: Bool = false
    
    var imageName: String { return name }
    var thumbnailName: String { return name + "Thumb" }
}

#if DEBUG
let testData = [
    Room(name: "Observation Deck", capacity: 6, hasVideo: true),
    Room(name: "Executive Suite", capacity: 8, hasVideo: false),
    Room(name: "Charter Jet", capacity: 16, hasVideo: true),
    Room(name: "Dungeon", capacity: 10, hasVideo: true),
    Room(name: "Panorama", capacity: 12, hasVideo: false),
    Room(name: "Oceanfront", capacity: 8, hasVideo: false),
    Room(name: "Rainbow Room", capacity: 10, hasVideo: true),
    Room(name: "Pastoral", capacity: 7, hasVideo: false),
    Room(name: "Elephant Room", capacity: 1, hasVideo: true)
]
#endif
```

> 这是我照着手敲的…如果有人知道哪里能弄来 WWDC 视频里的 Demo 项目源文件，请务必告诉我！  

到这一步为止还没什么不同，唯一比较不常见的可能是下面的 `testData`，这是方便我们调试用的假数据。

为了让 SwiftUI 的 `List` 能正常使用这个数据结构，我们还需要进行一点点改造：

```swift
import SwiftUI

struct Room: Identifiable {
    var id = UUID()
	  ...
}
```

`List` 能使用的数据结构必须遵循 SwiftUI 里新加入的 `Identifiable` 协议，它要求这个数据结构里必须有一个符合 `Hashable` 协议的变量 `id`，我们这里用到的 `id` 是 `UUID` 类型的，本身已经满足这个要求，所以这里就不需要做更多修改了。

## 列表显示

现在我们把 UI 和数据都准备好了，接下来就让它们组合起来，显示成一个列表！

对于 `List` 来说，列表项的默认布局就是水平方向的，所以我们可以直接 Cmd + 鼠标左键点击列表项 UI 里的 `HStack`，然后选择“Convert to List…”：
![](/uploads/swiftui-serial-tutorial-4/D66F6EDE-FA01-489B-A8D3-4EDC81B5E18A.png)

> 也许是 Xcode 的版本问题，在我这里显示的是“Embed in List”，但是最后的效果也是把选中了的 `HStack` 替换成了 `List`  

得益于 `List` 构造方法里的数组类型参数，这段 UI 代码看起来就像是一个 for…in 的语句一样！回头看看预览，一个像模像样的列表已经出来了：
![](/uploads/swiftui-serial-tutorial-4/38B4C8E5-0363-4674-AB1D-F078915AAF24.png)

接下来我们把准备好的数据对接上去：

```swift
struct ListView: View {
	var rooms: [Room] // 1
	var body: some View {
		List(rooms) { room in // 2
			Image(room.thumbnailName)
			VStack(alignment: .leading) {
				Text(room.name)
				Text("\(room.capacity) people")
					.font(.subheadline)
					.foregroundColor(.secondary)
			}
		}
	}
}

#if DEBUG
struct ListView_Previews: PreviewProvider {
    static var previews: some View {
        ListView(rooms: testData) // 3
    }
}
#endif
```

1. 先定义一个成员变量来保管数据，这一行为符合单一数据源的原则，这个数据是由调用方负责构建和传入的
2. 把 `List` 的参数缓存我们真实的数组，然后把内部的硬编码数据换成传进来的数据
3. 这里我们能看到预览视图的另一个好处，就是它完全模拟了真实情况下的代码逻辑，所以我们可以直接把上一步准备好的测试数据传进去

![](/uploads/swiftui-serial-tutorial-4/ACCBAF94-47D3-43B5-B924-5589A3F68C9D.png)

对比导入数据前后的界面，我们可以发现：默认情况下 Cell 的高度为 44pt，但是当塞进去的内容（比如图片）需要用到的高度大于这个值时，Cell 会自动调整自己的高度以适应内容的大小，并填充合适的间距来美化我们的列表项。这些小细节全都是免费的！

> 预览里的图片当然是要我们事先导入的，因为跟 SwiftUI 没什么关系，所以这里也就跳过了这一步；而且我也懒得找图片来代替它们了，所以截图里看到的还是之前默认图的样子…  

## NavigationViewController & NavigationBar

孤零零的一个列表可能还不够有意思，一般来说列表是罗列概要数据用的，为了看到更完整的信息，我们通常会在用户点击列表项的时候跳转到一个详情界面，这就涉及到了界面导航的概念。

在原来的开发过程里，这时候我们就会实现一个 `NavigationViewController` 去把我们列表的视图控制器包裹起来，然后通过 push & pop 这样的操作来实现界面的切换。

一起来看看 SwiftUI 是怎么做的：

```swift
NavigationView { // 1
    List(rooms) { 
		  // 列表项的布局 
	  }
	  .navigationBarTitle(Text("Rooms")) // 2
}
```

1. 我们用一个 `NavigationView` 把原本 `body` 里的全部内容包裹起来，就像我们原来用 `NavigationViewController` 把 `UIViewController` 包起来一个道理
2. 然后，我们给 `NavigationView` 里的最外层子视图（在这里就是 `List`）加一个修饰器来配置导航栏的标题；注意我们这里传的是一个 `View`，也就是说它不仅限于显示文字，还可以是各种各样遵循 `View` 协议的视图

完成这两步就可以在原有的视图之上显示一个导航栏了，不过还不够，我们要给列表项加上点击事件以便跳转到详情界面：

```swift
List(rooms) { 
    NavigationButton(destination: Text(room.name)) {
        // 列表项的布局
    }
}
```

这里就只有一步：把原先用于布局列表项的代码用一个 `NavigationButton` 包起来，同时通过参数来指定这个导航事件的目的地。如例子的代码所示，实现的效果是点击列表项之后跳转到一个新的界面，界面中会居中显示房间的名字：
![](/uploads/swiftui-serial-tutorial-4/IMG_E91EAE65D877-1.jpeg)

在加上 `NavigationButton` 之后，细心的童鞋们可能已经发现我们列表项的变化了：每一项的尾部都显示了一个小尖角，这是 iOS 里用于表示列表可以“点击查看更多”的常见元素了。

OK，用 Live Mode 运行一下预览，可以看到 SwiftUI 已经把转场过程中的标题动画处理好了，界面跳转、手势返回，全都丝般顺滑。

## 表单

两个典型的常用组件我们已经看完了，接下来我们要实现一个更复杂一些的界面。往常要实现这么一个界面，虽说没有太多的智力活，但体力活是肯定少不了的。

一图胜千言，先来看看效果图：
![](/uploads/swiftui-serial-tutorial-4/00BD9E2A-5EFD-4B06-86E1-E008B2FA3D55.png)
这种界面在用户注册的过程中也是挺常见的，借用一个前端的概念——它叫做“表单（Form）”。

### 环境准备

接下来的代码里，有些类需要在 Xcode11 beta 4 版本上才能使用（其实我只知道 Beta 1 用不了，但具体哪个版本变化的就不清楚了），建议大家先更新一下 Xcode，不然就只能在文章里过过眼瘾啦。虽说 WWDC 展示的时候那个讲师明明就已经用上了，果然发布会要用特供版本在哪里都是惯例啊。

顺带一提，这种变化除了更新软件之后上手试试之外，还能在哪里看到呢？答案就是：官方文档！
![](/uploads/swiftui-serial-tutorial-4/1BCC5384-E073-417C-A7AA-DF85A6253099.png)
在网页上找到你想看的 API 之后，把图示右上角的 “API Changes” 打开，选择要比较的版本（对于 Beta 版的软件，一般不会给出每个版本之间的变化，所以如上所述我也不知道 `Form` 这位兄弟是什么时候加进来的），然后就能在 “SDKs” 下面看到特定版本的需求是什么时候加进来的了。上面的截图就表示 macOS App 在 Xcode beta 4 版本上是无法使用 `Form` 的，因为它在 beta 5 才被正式加进来。

### 需求梳理

让我们用前三篇文章加上上面两个小节的知识，来判断一下要怎么实现它。

1. 我们要有一个大大的标题在界面左上角，也许是一个 `navigationBarTitle` 可以搞定的事情
2. 标题下面的内容样式看起来像是一个 `UITableView`
3. 这个列表分了好几个 Section，有些 Section 有属于自己的标题、有些却没有
4. 每一行都有自己的样式，可能要定制好几种 `UITableViewCell`，而最后一行起到的是按钮的作用，可以真的用按钮去实现，也可以用文字加上列表项点击事件的方式来做

### 那就动手吧！

看看上面的需求，每一点都不难，可它们胜在多啊！如果用 UIKit 去实现，光是把界面做出来就已经要花不少代码，更别说界面背后的数据逻辑了。

于是 SwiftUI 应运而生，这种简单元素组合而成的复杂页面正是它的强项！

我们直接看最终的界面代码，不过你也不用看得太仔细，我们会在下面把它一点点拆开来讲：

```swift
struct FormView : View {
    @State var userInfo = UserInfo() // 1
    var body: some View {
        Form { // 2
            Section(header: Text(“账号注册”).font(.title)) { // 3
                TextField(“请输入用户名”, text: $userInfo.name)
            }
            Section(header: Text(“种族”).font(.headline)) {
                Picker(selection: $userInfo.race, label: Text(“种族”)) { // 4
                    ForEach(Race.allCases) { race in // 4.1
                        Text(race.name).tag(race)
                    }
                }.pickerStyle(SegmentedPickerStyle())
            }
            Section(header: Text(“口味”).font(.headline)) {
                Toggle(isOn: $userInfo.loveSweet) { // 5
                    Text(“喜好甜食”)
                }
                Toggle(isOn: $userInfo.loveSpicy) {
                    Text(“爱吃辣”)
                }
            }
            Section(header: Text("年龄").font(.headline)) {
                Stepper(value: $userInfo.age, in: 1…120){ // 6
                    Text("年龄：\(userInfo.age)")
                }
            }
            Section {
                Button(action: {
                    print(“register with info: \(self.userInfo)”)
                }) {
                    Text(“Register”)
                }
            }
        }
    }
}
```

尽管有前几篇文章的知识作为铺垫，但这段代码里还是有不少的新面孔啊。

#### UserInfo

`UserInfo` 的类型定义是这样的：

```swift
struct UserInfo {
    var name: String = “”
    var loveSweet = false
    var loveSpicy = false
    var age: Int = 18
    var race: Race = .Yellow
}
```

它其实就是一个普通的结构体，用来表示这位用户的个人信息。

在上一篇文章里我们讲过，如果要把一个完整的自定义对象绑定到视图上，那么这个对象的类型就需要遵循 `BindableObject` 协议。但在这个例子里，我们只是将对象的属性绑定到了视图上，我们只在意这些属性而不是对象这个整体，所以可以直接通过 `@State` 来进行对象局部的绑定。

#### Form

如前面的文档截图里所述：`Form` 是一个用于组合一系列数据入口的视图，比如应用的设置页面，或者是例子里的用户信息录入界面。

话说回来，这么多新玩具，在开发过程中怎么知道有没有可以对症下药的呢？其实它们都整整齐齐码在了原来我们用于挑选 xib 或者 storyboard 组件的地方，我们可以用 Cmd + L 把这个界面弄出来：
![](/uploads/swiftui-serial-tutorial-4/%E6%88%AA%E5%B1%8F2019-08-13%E4%B8%8B%E5%8D%8810.18.33.png)

#### Section

顾名思义，这是一个用于分区的控件，它也可以被用在 `List` 里面，就组成了所谓的 “SectionList”，例如通讯录里按首字母分区的样式。

例子中的 `Section` 用于给 `Form` 里的不同数据进行分类，并提供统一样式的 Header，它显示的内容也可以通过构造方法里的 `header` 参数来进行修改。

#### Picker

这是目前为止碰到的最复杂的组件了，它有点类似于 UIKit 里的 `PickerView`，不过它是更广义的“选择器”，在样式上也会有更多的变化可供选择，比如例子里的 `SegmentedPickerStyle`（有些样式是平台相关的，在使用的时候要留意编译器的提醒）。

其实只要明白 `$` 变量的含义，这个构造方法也很好理解，就是把选择器里选中的内容与 `userInfo` 对象里的 `race` 属性绑定起来。

##### ForEach

这个写法本质上跟往 `List` 的构造方法里传一个数组的意思差不多，这样拆开来可以让列表型视图的内容样式更丰富，感兴趣的童鞋们可以详细看看[这个视频](https://developer.apple.com/videos/play/wwdc2019/204/)，这里就不展开讲了。

其中的 `Race` 是一个枚举类型，它的实现也很有意思：

```swift
enum Race : CaseIterable, Hashable, Identifiable { // 1
    case Yellow
    case Black
    case White
    case Mixed
    
    var id: UUID { UUID() }
    
    var name: String {
        switch self {
            case .Yellow: return "黄种人"
            case .Black: return "黑种人"
            case .White: return "白种人"
            case .Mixed: return "混血"
        }
    }
}

```

例子里的 `Race.allCase` 是遵循 `CaseIterable` 协议带给我们的一个福利，意思是“枚举所有可能的值”，于是我们就可以把它用在需要数组类型参数的地方了。

而 `Hashable` 和 `Identifiable` 是为了让这个枚举的对象可以被作为唯一标识符来使用。之所以这样做，是因为 `Picker` 的列表项必须要设置 `tag` 以区分用户的选择，于是例子中就直接用了枚举对象本身来作为选项的 `tag`。

#### Toggle

这个控件看名字就该知道是干嘛用的了，正如 Swift 4.2 之后的布尔值有了一个叫 `toggle` 的语法糖，这个控件就是用来标识开关状态的，也就是原来的 `UISwitch`。

#### Stepper

直译过来应该叫“步进器”。对于重视用户体验的苹果来说，这样看上去简单，但要正确实现逻辑还需要费点心思的小玩意儿，当然是选择封装成一个系统组件！相信看了前面那些五花八门的组件之后，这个 `Stepper` 的用法也是不需要再多加解释了。

### 小结

好了，这短短不到40行的代码（好吧，算上那个结构体和枚举也有60行左右了），就已经实现了我们这一小节开头截图里的那种表单效果。不仅如此，SwiftUI 还默默为我们做了大量的优化工作，我可以拍胸脯说这些代码在效率上也会是远超 UIKit 版本的。

## 总结

四篇文章下来，这个系列的 SwiftUI 教程也算是告一段落了。显然这仅仅是一个入门教程，更多有意思的新元素还在后面等着你。

文章里面的例子都是通过 Xcode 的新功能 Canvas 预览和截图出来的，目前它也只有个 iPhone 的外框样式，这难免有点限制了我们的想象，所以我必须要在最后提醒大家一句：

**用 SwiftUI 构建的界面天生就是跨全苹果平台的！**

也就是说，之前例子里的**所有代码放到 macOS 和 watchOS 上都是适用的**！（当然要除掉跟 UIKit 混合的那部分）而且所有界面的样式都会根据平台的不同而自动调整，几乎不需要开发者的介入就可以做到全平台适配！

写到这的时候我居然感觉挺兴奋的，就像是回到了刚刚接触 Xcode 和在 iOS 5 上做开发的那个年代。那个时候 Android 已经有好些机型要适配，而我只要把应用在我手里的 iPhone 上跑顺就八九不离十了:)

## 参考文章

- [Introducing SwiftUI: Building Your First App - WWDC 2019 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2019/204/)
- [SwiftUI Essentials - WWDC 2019 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2019/216/)
- [SwiftUI 的一些初步探索 (二)](https://onevcat.com/2019/06/swift-ui-firstlook-2/)

