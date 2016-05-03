---
title: 老生常谈的自适应高度UITableViewCell
date: 2016-04-29 15:44:39
tags:
- iOS
- AutoLayout
- UITableViewCell
---

UITableViewCell 自适应内容高度已经是一个老生常谈的问题了，网络上也随处可以找到相关的资料。从 iOS 8 开始，强大的 AutoLayout 甚至已经可以直接接管 Cell 高度的计算，于是这一话题也就慢慢淡出了人们的视野，包括我...
最近整理笔记的时候发现了这篇文章，想了想还是把它弄到新博客上来了。这篇文章是做毕业设计的时候（2014）总结下来的，当时因为要兼容到 iOS 7 ，所以用的都是比较原始的方法，不过在那个时候用起来还是比较顺手的（只是对比起 iOS 8 的用法，那真叫一个酸爽）。随着 iOS 后面的版本号越来越大，兼容 iOS 7 的应用应该也不多了，现在把它搬到这上面来，主要是为了记录以往走过的路，这些知识或许再也用不上，但积累它们的过程还是非常值得保留的。~~另一个原因是为了扩充新博客的文章数目。~~（喂！）

<!-- more -->

## 告诉我需求！
文中的界面制作使用的是 Storyboard，也会略微对比使用 xib 和纯代码的情况。

首先假设这么一种情况，我们的 UITableViewCell 里面只使用到一种布局：
![cell-UI](/uploads/using-autolayout-in-uitableviewcells/cell-ui.png)
其中， State 只显示一行； Name 最多只能显示3行，字体大小不改变； Additional 可以显示任意多行，字体大小不改变。
如果 cell 的布局变化比较大，比如说文字下面可能还有图片，4张图片和9张图片的布局要不同，那么还是建议把相去甚远的几种布局分别设计成几个不同的 cell，用 Cell Identifier 来区分彼此，然后再分别对每一个 cell 进行自适应内容高度的处理。

## 最喜欢IB了 - 约束设置
既然使用的是 Storyboard，那么一切的约束当然都是在 Storyboard 里面手动进行。
想要让 cell 根据内容自动调整高度，一个极其重要也似乎是唯一的原则是：约束自上而下贯通整个 contentView。

还是拿上面的截图做例子：
1. State 必须要有一个 Top Space to: Superview，这个 Superview 应该是 cell 的 contentView 而不是 cell 本身。
2. Name 要有一个 Top Space to: State，而这个 constant 是多少就可以随意了，它的作用主要是把二者的垂直空间连起来，怎么连、连多长都是看心情的事。
3. Additional 除了要有一个 Top Space To: Name 之外，还需要有一个 Bottom Space To: Superview（这个也是 contentView）。

其实，如果将旁边的数字 1 连到了 contentView 的 top 上，再把 State 跟数字 1 的 center Y 等同一下，也可以达到同样的效果。只是如果使用上面的构建方法，数字 1 就可以自己一边玩儿去了，因为三个 UILabel 已经完成了“确定 cell 所需高度”的关键任务。

## Write the code, change the world!
下面将分别针对 UITableView 中需要进行修改的不同几处分开来进行说明。

### 主角 - CustomCell.m
相对来说最忙的应该是 CustomCell 这个自定义类了，所以放到最前面来说。
在这个地方，需要关注的其实也只有两件事：

1. 为 cell 填充内容。
2. layoutSubviews。

#### 填充内容
其实就是属性的设置而已，但是需要注意的是，对于 UILabel 来说，我们想要看到的是，当显示的内容很多的时候，它可以有一个确定的宽度，然后多出来的内容自动往下接着排列下去。比较好的办法是，继承 UILabel，然后在子类中自己实现。这种做法也不麻烦，放到后面再说，现在我们这里就只是简单的设置几个 text 。
> 代码呢？

这么简单的赋值代码，我不贴！我不贴！

#### layoutSubviews
如果没有用继承 UILabel 的那套方案，那么这一步就是必须的，否则就是多余的。这里还是用到了继承的方案。
我们需要重写 cell 的 layoutSubviews 方法，并让它为我们的 UILabel 设定一些规矩：
![layoutSubviews-code](/uploads/using-autolayout-in-uitableviewcells/layoutSubviews-code.png)
代码非常简单，因为当运行到这里的时候，AutoLayout 已经帮我们处理好了一大堆布局的问题，我们只需要告诉 UILabel ，我想要你显示多宽就可以了。

### 主角诞生的地方 - cellForRowAtIndexPath
除了常规的获取和配置 cell 之外，我们最好在最后面补上两句简短的代码：
![cellForRowAtIndexPath-code](/uploads/using-autolayout-in-uitableviewcells/cellForRowAtIndexPath-code.png)
这两句的作用是告诉我们能者多劳的 cell，如果你发现自己的约束改变了，那么就马上更新一下自己。不写这两句问题也不大，只是为了防止某些特殊情况下 cell 的显示异常。

### 到底要我长多高你才满意？！ - heightForRowAtIndexPath
最关键的地方来了，告诉 tableView 每个 cell 到底该是多高。这里有三个关键步骤：

1. 创建一个offscreen cell。
2. 模拟真实的cell填充数据。
3. 计算出这个offscreen cell的高度。

实际上，我们要做的就是制作一个和当前位置需要显示的 cell 一模一样但却不会显示出来的 cell （好拗口），然后告诉 delegate 这个 cell 的高度即可。

#### Offscreen Cell
实验发现，“创建一个 offscreen cell ”是非常讲究的：

* 如果使用 `[tableView dequeueReusableCellWithIdentifier:CellIdentifier atIndexPath:indexPath]` 来获得一个 cell ，会造成内存的泄漏，因为上面这个方法在调用之前会先调用 heightForRowAtIndexPath 这个方法本身，有点像循环引用的概念。
* 直接用 `[[CustomCell alloc] init]` 来创建一个 cell 呢？如果使用的是 xib 或者纯代码的方式来创建布局的，那么当然没问题，不过千万不要忘了，在 VC 的某个地方调用一个 tableView 的 `registerClass:forCellReuseIdentifier:` 方法，把这个 CustomCell 给注册一下，否则会拿不到 cell 对应的属性。
* 对于 Storyboard 来说，因为 CustomCell 是直接放在 tableView 里面的，IB 已经帮你完成了这一切，所以不需要 registerClass 。如果你执意要注册一下的话，不仅没有任何好处，反而会破坏了 IB 的和谐，导致 `tableView:cellForRowAtIndexPath:` 里面的 dequeue 方法无法返回一个正确的 cell。

综上所述，用 Storyboard 就没有办法完成“创建一个 offscreen cell ”这一步了么？非也。
TableView 还有一个方法叫 `[dequeueReusableCellWithIdentifier:]` ，注意到它没有了 indexPath 这个参数，所以它不会去调用 `tableView:heightForRowAtIndexPath:` ，这给我们提供了一线希望。于是，完整的代码是这样的：
![offscreen-cell-code](/uploads/using-autolayout-in-uitableviewcells/offscreen-cell-code.png)

这里用到了 static 和 dispatch_once 来使得这个 offscreen cell 只需要被创建一次，然后就可以循环利用了，环保？

#### 填充数据
这里又是常规的填充数据，再啰嗦一次：如果使用了子类化 UILabel，那么这一步做的事会比看起来强大得多。不过，目前为止，这里看起来也只是填充一下数据而已。

#### 计算高度
这里我们要用到 cell.contentView 里一个很强大的方法：`[systemLayoutSizeFittingSize:]`，后面跟的参数有两个可选项： `UILayoutFittingCompressedSize` 和 `UILayoutFittingExpandSize` ，前者会得到满足所有约束的最小 size ，后者得到的是最大 size 。

这个方法看似方便，但也是因为我们前面做了许许多多的铺路：我们设置了贯通上下的约束，让 cell 能知道自己的内容总共有多高；我们设置了 UILabel 的 preferredMaxLayoutWidth ，让它们知道内容太多的时候要往下排列，等等等等。

在调用这个方法之前，为了保险起见，还是要让 cell 自己适当排列一下自己的 subviews ，完整代码如下：
![return-height-code](/uploads/using-autolayout-in-uitableviewcells/return-height-code.png)
最后的`+1`只是为了让 cell 之间有 1pt 的空间容纳 separator。

## 写在最后
到这里，一个可以自适应内容高度的 cell 就已经做好了。虽然整篇文章看起来挺长，但是实际写起来代码量也并不大。

其实如果还是觉得麻烦的话，让应用直接支持 iOS 8 以上不就好了:)
