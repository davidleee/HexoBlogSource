---
title: CSS 句法
date: 2018-10-19 19:54:42
tags:
- css
- html
- web
---

CSS 是声明型语言，这让它的句法（syntax）非常直白易懂。

除此之外，它还有很好的错误恢复机制，它能避免在错误发生时把所有东西都弄得一团乱：比如说在它碰到不认识的声明时，它会直接忽略掉这个东西。但从另一方面来说，这也让错误更难被发现了。

借着最近前端同事事务繁忙的机会，我这客户端工程师赶鸭子上架怼了一个静态网页出来，顺便产下了两篇副产品，都与 CSS 基础相关，这是第二篇（第一篇在此： {% post_link How-CSS-works CSS 是怎么运作的 %}），希望对其他可能有相同需要的同志送上一些帮助。

<!--more-->

## 名词解释

正如{% post_link How-CSS-works 上一篇文章 %}里说到的，CSS 是由选择器和一组属性组成的。其中，属性部分是由一系列的键值对组成，在 CSS 的世界里，它们有着自己的名字：

- 属性（Properties）：以“说人话”的方式表明这个玩意儿是干什么的
- 值（Values）：每个属性都会有对应的值，表示你想要怎么修改这个东西

这样的一组“属性-值”的组合，我在前面直接称呼为“键值对”了，但它在 CSS 世界里的本名其实是 **CSS 声明（CSS declaration）**。
被一对大括号包裹起来的一组 CSS 声明被称为 **CSS 声明块（CSS declaration blocks）**。
最后，一个 CSS 声明块会跟一个选择器搭配起来，称为 **CSS 规则（CSS Rulesets/Rules）**。

### CSS 声明

把 CSS 属性设置为一个特定的值，可以说是 CSS 这门语言的最核心功能了。需要注意的是，属性和值都是区分大小写的，它们之间用 “:” 来分隔。

目前，CSS 世界里一共有[300 种不同的属性](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference)，每种属性都有其对应的可选值。

> 在 CSS 语法里（包括其他 web 标准中），美式拼写是唯一的拼写标准。比如说，在需要设置颜色的时候，`color` 永远优于 `colour`。  

### CSS 声明块

CSS 声明以代码块的形式存在，用一对大括号括起来。

> CSS 声明块可以是空的（里面不带任何声明）  

CSS 声明块里的不同声明是通过 “;” 来分隔的。

> 实际上，最后一组声明是可以不用分号结尾的，但是好好的干嘛要逼死强迫症呢？

### CSS 选择器和规则

在写好了声明块之后，我们还需要告诉浏览器这些属性要用到哪里去，这就需要在这个声明块前面加上一个前缀——选择器了。

选择器可以是非常复杂的：你可以把一个声明块应用到好几个选择器上，通过逗号分隔；你还可以链式地构造一个指向性更明确的选择器，比如：选择一个类名是 “abc” 的元素，它要在 `<article>` 标签下，而且只有鼠标移动到它上面的时候才生效。

一个元素可能被多个选择器看上，所以同一个属性可能会被改变多次，CSS 会通过层叠算法（cascade algorithm）来判断这些属性修改的优先级。

> 对于同一个声明块，在使用复杂选择器的时候（比如存在多个选择器），如果其中的某一项选择有误，那么其他的选择器是不会被影响的，该怎么工作还是怎么工作。  

### CSS 语句

除了上面看到的声明块之外， CSS 里还有一些其他类型的语句：

- **At-规则** 用来传达元数据、条件信息或其他描述性信息。比如说：
  - [@charset](https://developer.mozilla.org/en-US/docs/Web/CSS/@charset)  和  [@import](https://developer.mozilla.org/en-US/docs/Web/CSS/@import)  (元数据)
  - [@media](https://developer.mozilla.org/en-US/docs/Web/CSS/@media)  或者  [@document](https://developer.mozilla.org/en-US/docs/Web/CSS/@document)  (条件信息，也叫内部语法)
  - [@font-face](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face)  (描述性信息)
    完整的写法是这样的：

```css
@import 'custom.css'; /* 从另一个 css 文件中引入规则 */
```

- **内部语法（Nested statements）** 是 at-规则 的一个子集，这类规则只会在特定条件下才会生效：
  - `@media` 运行设备符合某些条件时才执行
  - `@supports` 浏览器支持某些测试特性的时候才执行
  - `@document` 当前页面符合某些条件时才执行
    举个例子：

```css
@media (min-width: 801px) {
  body {
    margin: 0 auto;
    width: 800px;
  }
}
```

上述针对 `body` 的规则，只在设备宽度大于 800px 的时候才会生效。

## 小结

分两篇叙述的 CSS 相关知识就讲完了。这两篇文章的主要目的是把我们领进前端世界的大门，读完之后，我们应该可以实现一些简单的静态页面了！~（小声说：虽然具体怎么用还需要自己去谷歌百度一下）~当然，前端的魅力还远不止如此，要想把 CSS 玩出花儿来，还需要持续的磨练。

我这个半吊子的前端工程师总算是把整个静态页面的需求给怼出来啦！接下来如果有时间的话，我会再把页面里用到的一些 JS 实现的逻辑也拉出来溜一溜。要是这下一篇文章真的有诞生之日的话，那读到完整三个部分的同学们就会在前端界六得飞起~（假的）~了！

## 参考文章

[CSS syntax - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/CSS/Introduction_to_CSS/Syntax)