---
title: Android Unit Test 框架比较
date: 2017-07-04 14:50:08
tags:
- Android
- 单元测试
- JUnit
- Mock
---

这篇文章列举了现有常见的 Android 单元测试框架，并进行了简单的比较，方便用来进行框架的选型和收藏（毕竟只要收藏了本文，就相当于收藏了各大单元测试框架的主页，是不是很棒棒？）。
> 框架对比的部分会带有一定的偏向性，不过光是把本文当做一个单元测试的工具箱也是蛮顺手的。

<!-- more -->

## Java Tests
### JUnit
> [JUnit](http://junit.org/junit4/) 是基于 xUnit 架构实现的单元测试框架。雏形是1998年在 SmallTalk 上实现的 SUnit，后来原作者将它迁移到 Java 上，从此声名大噪。  

JUnit 在 Android 官网上出现的频率相当高，是 Android 单元测试的基础框架之一，后面提到的 mock 框架都相当于是在丰富这些基础框架的功能。
要做单元测试的开发者们通常都会在 JUnit 和 TestNG 之间选择一个，个人感觉还是选 JUnit 好一些。（理由在 TestNG 的介绍中）

### TestNG
> [TestNG](http://testng.org/doc/index.html)  是一名软件工程师（Cédric Beust）对 JUnit 的改进，他将对 JUnit 的不满写在了 [这里](http://beust.com/weblog/2004/02/08/junit-pain/) 和 [这里](http://beust.com/weblog/2004/08/25/testsetup-and-evil-static-methods/)，感兴趣的可以去看看。  

TestNG 对 JUnit 的改进主要在两个方面：
1. 在 JUnit 中，同一个测试类中的不同测试方法是相互独立的，也就是说，每个测试方法开始前都会执行一次测试类的构造方法，这就让测试方法之间共享某些状态变得不可能
2. 后来 JUnit 对问题1给出了解决方案—— `TestSetup`，但是它必须通过静态方法去使用（JUnit 的 `TestSetup` 因为年代久远，已经没有文档了，估计 JUnit 也觉得这个解决方案不妥，赶紧修复了吧）

事实上，TestNG 要改进的问题在新版本的 JUnit 上已经不是问题了（毕竟那次抱怨都发自2004年）。从官网最后一次更新是2015年12月来看，TestNG 社区的活跃程度远不如它的改进对象 JUnit，所以还是乖乖滚回 JUnit 的怀抱好了。

### Mockito
> [Mockito](http://site.mockito.org/) 最初是在 [EasyMock](http://easymock.org/) 的基础上实现的，持续更新到现在，而始祖 EasyMock 已经在2015年停更了。  

Mockito 的优势正如它官网说的：
* 测试代码和验证提示的良好可读性
* 良好的社区支持
* Java 世界中排名能到达前十的明星框架
* [Features And Motivations](https://github.com/mockito/mockito/wiki/Features-And-Motivations)

但劣势也同样明显，其实不应该叫做“劣势”，而应该说是因为实现方式所带来的“限制”：[What are the limitations of Mockito](https://github.com/mockito/mockito/wiki/FAQ#what-are-the-limitations-of-mockito)
不过这些限制也因为 PowerMock 的出现而被化解，所以搭配上 PowerMock 之后，Java 单元测试这边几乎所向披靡。

### jMock
> [jMock](http://www.jmock.org/) 是一款爷爷级的 mock 框架。  

在2008年底就已经更新到 2.6.0-RC1 版了，结果在2012年底发布 2.6.0-released 版本之后就再也没有了消息，花费4年打磨了一个版本，估计作者是气死了吧。

### PowerMock
> [PowerMock](https://github.com/powermock/powermock) 的出现是为了弥补常规 mock 框架的限制问题。Mockito 与 PowerMock 合作紧密，但毕竟不是一伙人，所以后者的大版本更新往往会落后前者一点。  

它绕开了 CGLib，直接修改类的字节码，以实现 mock 某个类的目的。这种风骚的操作让 PowerMock 能够做到上面提到的框架做不到的事情：修改静态或私有方法等等。
然而，它把自己定位为高层次框架的插件，所以用它的时候就不可避免要带上其他的框架（在 gradle 里引入 PowerMock 的时候会自动引入相关框架的依赖）。如果已经使用了 Mockito 之流，那在后期引入 PowerMock 的时候必须让二者的版本相对应，或者抛弃原来的 Mockito，直接使用 PowerMock 依赖的版本。

### JMockit
> [JMockit](http://jmockit.org/index.html) 从2014年提交第一个 commit，到现在已经迭代了将近3年的时间。虽然从 Github 上面看它的社区并不太活跃，但是却保持着良好的更新频率，现在还在持续更新着。  

研究 Android 单元测试好几天了才看到这个框架，真是惭愧。
它相当于一个非插件化的 PowerMock，集成和使用的方式都非常简单。语法上虽然不及 `when/thenReturn` 这样口语化，但也是一目了然。
所以，一个 JMockit 能搞定的事情，为什么要用 Mockito&PowerMock 组合呢？

关于这类修改字节码的框架是如何实现的，可以看这篇文章：
[浅谈jmockit中mock机制的实现](https://segmentfault.com/a/1190000003718149)



## Android Test
### Robolectric
> [Robolectric](http://robolectric.org/) 重写了许多 Android SDK 里的类，使得在 JVM 上进行 Android 测试成为了可能。  

按照[文档](http://robolectric.org/extending/)中的描述，Shadow class 是用来修改和扩充 Android OS 下的类的行为的，除了可以 shadow 构造方法外，它和 Mockito 的 mock 没有太大区别，所以它并不能作为 Mockito 的扩充来使用。

官网的介绍中也提及了 Mockito，按照它的说法，我们完全可以用 Mockito 来实现 Robolectric 的功能，只不过要我们自己将 Android SDK 和一些 native 方法一个个 mock 掉而已。这部分工作正是 Robolectric 的价值所在。

### Espresso & UI Automator
> [Espresso](https://developer.android.com/topic/libraries/testing-support-library/index.html#Espresso) & [UI Automator](https://developer.android.com/topic/libraries/testing-support-library/index.html#UIAutomator) 这两兄弟都是 Google 官方推荐的 UI 测试框架，其中前者更适合于白盒测试，后者更适合黑盒测试。  

根据官方的说法，它们还是有一些区别的：
Espresso：适合应用中的功能性 UI 测试
UI Automator：适合跨系统和已安装应用的跨应用功能性 UI 测试

但因为目前还没有涉及到这一块的测试，没有深入研究，所以推荐大家还是去官网看看比较好。



## Others
### Hamcrest
> [Hamcrest](https://github.com/hamcrest) 是一个提供更灵活的 Assertion 的 API 的第三方库  
