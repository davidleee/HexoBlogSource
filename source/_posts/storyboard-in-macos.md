---
title: 不一样的 macOS Storyboard
date: 2018-08-21 09:12:40
tags:
- macOS
- Swift
---

从 iOS 转来 macOS 阵营已经有几个月了，断断续续踩了一些这样那样的坑，特此写一个系列记录下来。这个系列碰到的问题都是已经找到了解决方法的，希望对其他人也能有些帮助。

> 这是 macOS 开发系列的第一篇文章。  
>
> 从项目开始的地方讲起 —— Storyboard。  

<!-- more -->

## Main.Storyboard

新建 macOS 项目的时候，如果勾上了 “Use Storyboards” 的话，工程里会自带了一个 “Main.storyboard”。

打开看看，发现跟 iOS 项目中的样子大同小异，不同的地方在于，原来的 “Navigation Controller” 在这里变成了 “Window Controller”，而且上面还多了一个没见过的 “Main Menu”。（关于菜单的内容会在第三篇文章里讲到）

![](/uploads/storyboard-in-macos/4BB6559C-64F4-421D-A8D5-6427C96EF774.png)

如果把应用跑起来，会发现 “Window Controller” 对应的是应用的启动窗口，而 “Main Menu” 对应的是显示在顶部的系统菜单栏。

![](/uploads/storyboard-in-macos/6D230E15-056A-495D-8063-96FA6F4E9792.png)

到目前为止，一切都还是合乎逻辑的，直到某天我手贱把 Main.storyboard 删掉了…

## 自定义 Storyboard

按照我们在 iOS 里学来的经验，这没什么大不了的嘛！

让我们重新创建一个 Storyboard
![](/uploads/storyboard-in-macos/AAA78593-47E1-4E34-A6FE-34D08C3D5328.png)

拖一个 “Window Controller” 出来，程序入口也要设置上（就是图片中间那个小箭头，位置在 Window Controller -> Attributes Inspector -> “Is Initial Controller”）
![](/uploads/storyboard-in-macos/7BCBD120-A01E-48A9-9146-9A07CD956C81.png)

然后到项目配置里设置一下
![](/uploads/storyboard-in-macos/35E29846-E2B3-4A94-A221-C9CB866FFC42.png)

Done！好像忘了点什么…不过先跑起来看看！
![](/uploads/storyboard-in-macos/7AC50D3E-EF1B-43AE-80CE-0D12D5520CFE.png)

Emmm….菜单栏你怎么了！我的菜单栏呢！

## 拯救被误删的 Main.storyboard

回到新建的 ”Main.storyboard“ 里瞅瞅，发现跟自带的 Storyboard 相比，少了 “Main Menu“ 这个玩意儿，相应的，在左边的场景列表里，应该要有一个叫 “Application Scene” 的场景。

![](/uploads/storyboard-in-macos/BCD3A74F-812C-47F1-992C-E8BA3E1B5EE1.png)

翻遍了控件库，都没有找到任何与 “Application” 和 “Main Menu” 相关的东西，最后在一个 StackOverflow 的问题里找到了答案

> [macos - Create Application Scene in blank OS X Storyboard - Stack Overflow](https://stackoverflow.com/questions/24418936/create-application-scene-in-blank-os-x-storyboard)  

简单总结一下就是：即使时隔多年，macOS 上还是不怎么支持让一个不是基于 Storyboard 开发的应用升级到使用 Storyboard 的版本，所以控件库里压根就没有提供 Application Scene。

难道就没有办法了？

虽然没有试过在非 Storyboard 项目上集成 Storyboard，但是对于误删的情况还是有救的：（答案也来自于上面的链接里）

1. 创建一个新的、带 Storyboard 的项目
2. 查看新项目里的 Main.storyboard 的源码
3. 将下图 `<!--Application-->` 注释下面的 `<Scene>...</Scene>` 的全部内容都拷贝到我们刚刚新创建的 Storyboard 里面
   ![](/uploads/storyboard-in-macos/35685F05-E093-4A08-93B4-80332BE704B0.png)

保存后重新打开 Storyboard，你会发现世界又恢复到原本的样子了！我们的菜单栏终于又回来了！

当然，因为是复制粘贴过来的，菜单栏里的一些信息还是新项目里的样子，在 Storyboard 稍加修改就可以了。

## 总结

虽说长得差不多，但 macOS 开发里用到的 Storyboard 和 iOS 上的还是有不少的出入。

比如有时候 IB 里修改了文案，但是跑起来却不一样了，那可能是 Storyboard 生成了国际化配置文件，在里面搜索一下也许会找到元凶：
![](/uploads/storyboard-in-macos/E490581B-53D5-4537-97B2-FC377985545E.png)

而更多的时候，在 IB 里调整了颜色之类的配置是不能直接看到效果的，还是要跑起来验证一下，以实际运行效果为准。

总而言之，在 macOS 开发过程中使用 Storyboard 需要更多的耐心和实际运行验证，这样的情况也许要到 macOS 支持 UIKit 的时候才能改善了。