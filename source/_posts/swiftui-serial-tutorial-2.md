---
title: SwiftUI 系列教程（2）
date: 2019-07-08 11:15:32
tags:
- Swift
- SwiftUI
- Image
- UIView
- UIViewRepresentable
- MKMapView
- Spacer
---

在{% post_link swiftui-serial-tutorial-1 上一篇文章 %}中，我们了解了 SwiftUI 的 `Text`  组件，并通过 `Stack` 系列的组件对内容进行了一些简单的布局。在这篇文章里，我们会认识一个全新的图片组件，并且会尝试利用这两篇文章的知识，结合 MapKit 框架，来实现一个简单的地点详情界面。

<!--more-->

> 写完第一篇文章之后，本职的开发任务突然进入了紧张的预发布阶段，搞得早就写好的第二篇文章拖了这么久才完成润色和发布，看来“全网最早”要丢了...

## 自定义图片视图
首先把一张地标图片放到 *Assets.xcassets* 里去，我在百度找了张广州塔的照片：
![](/uploads/swiftui-serial-tutorial-2/BF6A8678-F10F-4633-A932-94B781BE73F9.png)

然后，我们要为新的图片视图创建一个新的类，就放在上一篇文章的 *ContentView.swift* 旁边好了。新建文件的时候，选择 **SwiftUI View**：
![](/uploads/swiftui-serial-tutorial-2/8A140752-2AE7-40E5-B320-F9DF7ADECE5C.png)

取个名字叫 CircleImage，因为我们将要在这里把广州塔裁剪成一个圆。新建的代码内容跟上一章看到的一样，我们把 `Text` 改成 `Image`，然后把图片的名字传进去，直接就可以通过预览在画布上看到我们的图片了：
![](/uploads/swiftui-serial-tutorial-2/03DBC394-DA28-4686-87CA-117FCAC32CC6.png)

接下来我们在代码里给它加上一个圆形的裁剪。原来的做法有很多了，最快速的做法应该是操作 `clipsToBounds` 和 `cornerRadius`，给图片加上长度等于一半宽高的圆角，这还得要求图片是正方形的才能达到满意的显示效果。

而在 SwiftUI 里，这就是一句话的事情：
![](/uploads/swiftui-serial-tutorial-2/D1D63EE5-2169-40D0-ACAB-F68AE4EED5EB.png)

`.clipShape()` 给图片加了个裁剪的形状，其中 `Circle` 类型是一个用来当作遮罩的图形，你也可以给它加上填充色或者描边来单独使用，类似于以往通过 `CALayer` 去实现的效果。

但这也太大了，我们的屏幕装不下，我们可以再加两行代码，把图片缩放到一个合适的大小：
![](/uploads/swiftui-serial-tutorial-2/E0470CA5-E465-4D53-8E1A-555C180DED64.png)

> 讲道理，这里设置的宽高应该是一样的，毕竟是个圆嘛…但是我懒得重新截图了，各位童鞋自己调整一下数值就可以了

为了让图片本身在不同背景下都能凸显出来，我们再给它加个描边，这要通过 `overlay()` 方法去实现；也许再加个阴影吧，用到的是 `shadow()` 方法；最后出来的效果是这样的：
![](/uploads/swiftui-serial-tutorial-2/1263812F-2159-4AD5-9B21-50A314CFA99C.png)

是不是醒目多啦？

> 每当做完一个新视图，我就想对比一下用老方法实现同样的效果有什么不同…  

## 在 SwiftUI 里使用 UIKit
不知道大家发现了没有，我们在 SwiftUI 里用到的视图全部都是 `struct`，这意味着它们跟我们原本熟悉的 UIKit 是两套不同的机制，那难道以前开发的视图就完全用不上了吗？

答案是可以的。

要在 SwiftUI 里使用 UIView 的子类，只需要把它用一个遵循 `UIViewRepresentable` 协议的 SwiftUI 视图包裹起来即可。

> 这里举的是 UIKit 的例子，但同样也适用于 AppKit 和 WatchKit。  

我们再来创建一个新的 SwiftUI View 来做我们的地图界面，但这一次，我们要改一下内容视图的协议：
```swift
import SwiftUI
import MapKit // 引入 MapKit

struct MapView : UIViewRepresentable { // 把这里的协议改掉
    var body: some View {
        Text(“Hello World!”)
    }
}
```

然后你会发现 Xcode 在 `UIViewRepresentable` 这里报错了，因为这个协议下有两个必须实现的方法：
1. `makeUIView(context:)` 用来创建我们的 `MKMapView`
2. `updateUIView(_:context:)` 用来进行视图的配置，并响应后续可能的变化

那下面我们就来实现一下。再加新代码之前，可以把已经用不上的 `body` 部分先删掉了。

对于 `makeUIView(context:)`，只需要声明它返回的是 `MKMapView` 然后直接通过构造方法返回一个空对象就可以了：
```swift
func makeUIView(context: UIViewRepresentableContext<MapView>) -> MKMapView {
    MKMapView(frame: .zero)
}
```

`updateUIView(_:context:)` 要做的事情就比较多了，我们一步步说：
```swift
    func updateUIView(_ uiView: MKMapView, context: UIViewRepresentableContext<MapView>) {
		  // 1
        let coordinate = CLLocationCoordinate2D(latitude: 23.112223, longitude: 113.331084)
		  // 2
        let span = MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
		  // 3
        let region = MKCoordinateRegion(center: coordinate, span: span)
        uiView.setRegion(region, animated: true)
    }
```
1. 先把表示广州塔坐标的对象给构造出来（这是我在网上查的，等会预览的时候看看准不准）
2. 构造一个用来标识地图的缩放等级的对象，数值越小地图拉得越近
3. 构造坐标区域，并把这个区域设置到我们的地图视图上

赶紧预览一下看看效果吧！你会发现画布上空白一片…

那是因为预览默认是静态模式的，它只能完整渲染 SwiftUI 的视图。因为我们现在用到了 UIView 的子类，所以要把预览切换到实时模式（右下角红框里的按钮）：
![](/uploads/swiftui-serial-tutorial-2/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72019-06-16%E4%B8%8B%E5%8D%887.28.49.png)

emmmm…塔呢？这定位看起来也不是很准啊。

## 把一切都组装起来吧！
先看看预期实现的效果图：
![](/uploads/swiftui-serial-tutorial-2/973ba702-85db-4852-851f-86a94cfca002.png)

然后花一点时间思考一下怎么弄，我们这两篇文章的知识是完全足够了的。

好啦！公布答案！

我们会在上一篇文章实现的 `ContentView` 里直接进行组装，下面来看看分解动作：
```swift
struct ContentView: View {
    var body: some View {
        VStack { // 1
            MapView() 
                .edgesIgnoringSafeArea(.top) // 2.1
                .frame(height: 300) // 2.2

            CircleImage()
                .offset(y: -100)
                .padding(.bottom, -100) // 3

            VStack(alignment: .leading) {
                Text(“Hello SwiftUI”)
                    .font(.title)
                HStack(alignment: .top) {
                    Text(“First Description”)
                        .font(.subheadline)
                    Spacer()
                    Text(“Second Description”)
                        .font(.subheadline)
                }
            }
            .padding()

            Spacer() // 4
        }
    }
}
```
1. 先用一个 `VStack` 把所有内容包裹起来，默认情况下 `VStack` 的内容布局是居中的，所以我们不需要做修改
2. 针对 `MapView` 我们要做两个修改
     1.  `edgesIgnoringSafeArea` 可以让我们的视图把系统预留给刘海和状态栏的区域也用掉，这样看起来会更自然一点
     2. 对于只设置了 `height` 的视图，它的宽度会默认占满所有父视图里的可用空间
3. 我们的图片视图本身已经实现了所有效果了，现在只需要调整一下位置即可。需要注意的是，因为这里是通过 `offset` 移动的，所以为了保持与底部文字的间距，特意加上了一个负的 `padding` 来抵消掉位移导致的差距
4. 做完前三步的操作后，这个 `VStack` 整体是在竖直方向上居中的，所以加上一个 `Spacer` 把整体有内容的部分顶到最上面（其实也可以通过 `HStack` 的 `alignment`来实现，不过那样代码就没有现在的简单优雅了）

最终成果：
![](/uploads/swiftui-serial-tutorial-2/EF653793-4B81-42D7-9B53-00790EE31E8F.png)

> 如果你发现照着实现出来之后，中间圆形部分特别大的话，别担心，你是对的！ 
> 因为文章前面的 `CircleImage` 确实是为了展示而特意设置得比较大的，所以调整一下里面的宽高即可。  

## 小结
到这里大家应该对 SwiftUI 的使用比较上手了，但目前为止涉及到的组件还比较少，SwiftUI 光是各种强大的组件就已经够玩很久了。不过我打算在第四篇文章里再集中讲各种有意思的组件使用方式，因为下一篇文章我们要先解决数据来源的问题。

既然我们的视图组件已经是通过声明式的写法来构建的了，那我们的数据是不是也该换一种方式绑定到视图上呢？在 JS 上我们可以用 react-redux 这样的数据绑定手段，那 SwiftUI 是不是该搭配 RxSwift 来使用了？

这些问题都将在下一篇文章里为大家解答！

## 参考链接
[SwiftUI | Creating and Combining Views](https://developer.apple.com/tutorials/swiftui/creating-and-combining-views)
[SwiftUI Essentials - WWDC 2019 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2019/216/)