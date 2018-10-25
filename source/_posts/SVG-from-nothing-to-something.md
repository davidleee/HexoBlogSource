---
title: SVG 从“完全不懂”到“足够开个入门分享”
date: 2018-10-25 15:08:10
tags:
- svg
- xml
- markup
- react-native
- viewbox
- clippath
---

最近需要在 React-Native 项目里实现一个填满特定图形的效果，找很久都没发现能满足需求的开源库，于是就打算用 SVG 自己怼一个。好在这方面教程还不少，虽然不能一步到位，但几篇文章加一起也能把效果实现出来，于是在这里把入门过程记录一下，希望帮后来者省点功夫。

<!--more-->

## SVG 是个啥？

SVG（Scalable Vector Graphics） 是一种基于 XML 语法的图像格式。跟基于像素处理的图片格式不同，它是基于对图像形状的描述来实现的，本质上是一个文本文件，体积上较小，而且在放大的时候也不会失真。

因为 SVG 是基于 XML 语法的，所以对于前端开发者来说，写起来应该比较顺手；对于 React-Native 的项目，因为 JSX 的关系，用起 SVG 来也是没有“语言障碍”的。

## SVG 长什么样？

直接上源码：

```xml
<svg width='100' height="100" viewBox="0 0 100 100" version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
  <defs>
    <clipPath id="heart">
        <path d="M81.495,13.923c-11.368-5.261-26.234-0.311-31.489,11.032C44.74,13.612,29.879,8.657,18.511,13.923  C6.402,19.539,0.613,33.883,10.175,50.804c6.792,12.04,18.826,21.111,39.831,37.379c20.993-16.268,33.033-25.344,39.819-37.379  C99.387,33.883,93.598,19.539,81.495,13.923z"/>
    </clipPath>
  </defs>
  <rect x='0' y='0' fill='rgb(217,217,217)' width="100%" height="100%" clip-path="url(#heart)" />
  <rect x='0%' y="50%" fill='red' width="100%" height="100%" clip-path="url(#heart)" />
</svg>
```

这张 SVG 图渲染出来是这样的：
![](/uploads/SVG-from-nothing-to-something/57D524DA-6FD7-4538-A238-B002D099D91A.png)

> 直接把上面的代码保存为文本，就可以用浏览器打开并显示了。macOS 的用户还可以直接空格预览。  

配合一些参数的改变，做动画也是分分钟的事情。对于之前没有怎么用过 SVG 的我来说，简直是打开了新世界的大门。

接下来就让我们一起探究一下上面这个图是怎么来的。

## SVG 显示原理

根据上面的例子，我们可以大胆猜测一下： `<path>` 标签下 `d` 属性的值就是用来描绘这个心形的外框路径的。既然描绘路径已经是确定的了，那一张 SVG 图片是怎么实现缩放不失真的特性的呢？
要回答这个问题，就要让我们先了解一下 SVG 的一些基本显示原理。

### The SVG Canvas

假设我们要将一个 SVG 图形绘制到一张画布（Canvas）上，概念上这张画布应该是无限大的，这样我们的图形才可以是任意大小。然而，实际上 SVG 图片是显示在一个有限的区域里的，就像我们透过窗户看窗外的风景一样，这个有限区域被称为“观察孔”（Viewport）。

### The Viewport

“观察孔”指的是 SVG 图片可见的那一部分，想象我们透过窗户看窗外的风景，这个窗子就是外面风景的观察孔。

> 类似的，我们在浏览网页的时候面对的也是这种情况，网页的大小通常比浏览器的窗口要大，这时候浏览器就是这个网页的观察孔了。  

我们通过设置 `<svg>` 标签的 `width` 和 `height` 属性来确定这张 SVG 的观察孔大小，对于上面的心形来说，观察孔是 100x100 的正方形：

```svg
<!-- the viewport will be 100px by 100px -->
<svg width="100" height="100">
		<!-- SVG content drawn onto the SVG canvas -->
</svg>
```

> 在 SVG 里，数值的单位是可选的。在我们不主动提供的时候，默认会以 `px` 为单位。可选的单位有 `em`、`ex`、`px`、`pt`、`pc`、`cm`、`mm`、`in` 和百分比。  

### 坐标系统

在观察孔的大小确定下来之后，SVG 就会建立一套初始的坐标系统：以最左上角为 `(0, 0)` 点，x 轴和 y 轴分别向右和向下延伸（就像移动客户端和网页显示里那样）。在这个基础上，我们刚刚创建的观察孔也就有了属于自己的一套位置标识：`(x: 0, y: 0, width: 100, height: 100)`。

## The `viewbox`

在了解了上述知识之后，我们就可以来说说 `viewbox` 这个属性了。

我们可以把 `viewbox` 理解为“真正的坐标系统”，因为它决定了 SVG 图形是怎么绘制到画布上的。一个 SVG 图形的大小可以与观察孔不一样，它可能会完整地显示出来，也可能会被观察孔裁减掉一部分。

> 就像一张普通图片一样，当你需要把图形完整地放进一个视图里面时，可以调节图片的拉伸模式，一边让图片的大小更为合理。SVG 里对应的属性是 `preserveAspectRatio`。  

`viewbox`  会一次性设置4个参数：

```svg
<svg viewbox="<min-x> <min-y> <width> <height>"></svg>
```

其中，`<min-x>` 和 `<min-y>` 不能设置为负数，`<width>` 和 `<height>` 设置为 0 的话，元素就压根不会绘制了。

所以说，`viewbox="0 0 100 100"` 就做了下面几件事情：

1. 在画布上划分出来一个 100x100 大小的区域，放在 (0, 0) 点上
2. 把 SVG 图形缩放成合适这个区域的样子
3. 将整个区域（包括里面的图形）放大到铺满整个观察孔
4. 将这个坐标系统**按比例**映射到到初始坐标系统上

那对于上面的爱心图片来说，我们尝试调整一下 `viewbox` 的原点，将它设置为 `viewbox="50 50 100 100"` 试试：
![](/uploads/SVG-from-nothing-to-something/353B4442-E68D-4F8A-8D53-CE5EEED21F36.png)
可以看到，就像地图软件一样，镜头往画面的右下方移动了一段。这也相当于把整个画面往左上方推了过去一点，我们可以通过设置画布的 `transform` 属性来实现相同的效果：`transform="translate(-50 -50)"`。

> 当设计师给你一张 SVG 图片的时候，其中的图案路径可能是按照一定的大小和位移来绘制的，比如从(10, 10) 点开始画的一张 40x40 的图，这时候你的 `viewbox` 就应该设置为 `viewbox="10 10 40 40"` ，让图片放到最合适的坐标系统上。  

## 小结

上面的例子演示了一张 SVG 图的常规操作，限于篇幅原因，还有一些有意思的情况没有展示到，比如当 `viewbox` 里设置的宽高比与我们在 `<svg>` 标签里设置的宽高比不一样会发生什么呢？这种情况下，就需要我们去了解一下 `preserveAspectRatio` 属性了。

在参考文章 [Understanding SVG Coordinate Systems and Transformations (Part 1) ](https://www.sarasoueidan.com/blog/svg-coordinate-systems/) 里，有对这方面更详细的解释，而且作者还提供了非常直观的在线预览工具，相信会对偏向于使用图像思维的同学们更有帮助。

## 参考文章

- [Understanding SVG Coordinate Systems and Transformations (Part 1) ](https://www.sarasoueidan.com/blog/svg-coordinate-systems/)
- [SVG 图像入门教程 - 阮一峰的网络日志](http://www.ruanyifeng.com/blog/2018/08/svg.html)