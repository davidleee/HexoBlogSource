---
title: CSS 是怎么运作的
date: 2018-10-19 19:28:35
tags:
- css
- html
- web
- dom
---

CSS 全称 Cascading Style Sheets，网页内容（HTML）会被浏览器转换为 DOM（Document Object Model）以供显示，而 CSS 就是作用在 DOM 上以改变它们的样式、布局或行为等。对于前端工程师来说，这是很常见的基本操作了，但是对其他不常敲网页代码的程序员来说，却可能会有些陌生。

借着最近前端同事事务繁忙的机会，我这客户端工程师赶鸭子上架怼了一个静态网页出来，顺便产下了两篇副产品，都与 CSS 基础相关，这是第一篇，希望能对其他可能有相同需要的同志送上一些帮助。

<!--more-->

> 文章的内容基本是翻译 MDN 文档来的，怕有什么遗漏的同学可以直接翻到文末看官文，也欢迎指出本文的错误 :)

## CSS 是怎么作用到 HTML 上的？

网页浏览器会把 CSS 规则应用到文档上，以改变文档内容的表现形式，一个单一的 CSS 规则是由下面这两个东西组成的：

1. 一组属性（Properties）：这些参数会更新 HTML 的内容，让它在显示的时候与众不同
2. 一个选择器（Selector）：用来挑选要作用到哪个元素上

一个 CSS 规则约定了某个元素长什么样，一个包含了一组 CSS 规则（Rulesets/Rules）的 `stylesheet` 就定义了一个网页的长相。

### 举个例子

来看一个简单的 HTML 文档，这个例子里包含了 `<h1>` 和 `<p>` 标签，而 `stylesheet` 则是通过 `<link>` 元素实现的：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>My CSS experiment</title>
    <link rel="stylesheet" href="style.css">
  </head>
  <body>
    <h1>Hello World!</h1>
    <p>This is my first CSS example</p>
  </body>
</html>
```

然后看两个 CSS 的规则：

```css
h1 {
  color: blue;
  background-color: yellow;
  border: 1px solid black;
}

p {
  color: red;
}
```

> 这两个 CSS 规则是写在 *style.css* 文件里的，跟上面的 *.html* 文件放在一起就可以通过相对路径引用到

大括号签名的标签（`h1` 和 `p`）就是选择器，它告诉浏览器这些规则要作用在什么标签上；而大括号里的键值对就约定了这些标签的内容的显示规则。

## 原理呢？

浏览器在处理网页的时候，会分两步走：

1. 把 HTML 和 CSS 转换为 DOM，DOM 会把内容和样式融合到一起
2. 把 DOM 的内容显示出来
   ![DOM](/uploads/How-CSS-works/D90AF498-F58B-4AEF-91B9-9E38F4863B92.png)

## 介绍一下 DOM

一个 DOM 的内容是以树状结构保存的。每个通过 markup 语言表述的元素、属性、文字等会变成 DOM 节点保存在树上。

### DOM 的真面目

假设我们有一段 HTML 代码：

```html
<p>
  Let's use:
  <span>Cascading</span>
  <span>Style</span>
  <span>Sheets</span>
</p>
```

将它转换为 DOM 之后，这个 DOM 会是这样的：

```
P
├─ "Let's use:"
├─ SPAN
|  └─ "Cascading"
├─ SPAN
|  └─ "Style"
└─ SPAN
   └─ "Sheets"
```

现在来加一个 CSS 约束试试：

```css
span {
  border: 1px solid black;
  background-color: lime;
}
```

emmm…还是跟刚刚一样的 DOM，不过这些 CSS 约束会被加到 `span` 选择器上面去，于是渲染出来的样子就不一样了。

## 三种使用 CSS 的方法

### 外部 stylesheet

就是上面例子里用到的方法，CSS 约束是写在一个单独的 *.css* 文件里的

### 内部 stylesheet

即直接通过 `<style>` 标签来定义元素的长相，这个 `<style>` 需要写在 `<head>` 标签下才会生效：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>My CSS experiment</title>
    <style>
      h1 {
        color: blue;
        background-color: yellow;
        border: 1px solid black;
      }

      p {
        color: red;
      }
    </style>
  </head>
  <body>
    <h1>Hello World!</h1>
    <p>This is my first CSS example</p>
  </body>
</html>
```

这个例子跟之前的例子是同样效果的。

### 内联样式

对于只想改变单独一个标签元素的情况下，可以通过标签的 `style` 属性实现：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>My CSS experiment</title>
  </head>
  <body>
    <h1 style="color: blue;background-color: yellow;border: 1px solid black;">Hello World!</h1>
    <p style="color:red;">This is my first CSS example</p>
  </body>
</html>
```

然而这种做法并没有很受待见，因为这样定义的样式没办法复用，你可能需要在好几个文档里面写上同样的几个样式，维护成本大大升高了。

另一方面，把 HTML 语法跟 CSS 语法混合在一起，看起来就不那么清晰易懂了，建议是把不同类型的代码分开，保持纯粹。

## 小结

到这里，我们已经不止能写 HTML 网页了，还能通过 CSS 给这简陋的网页披上华丽丽的外衣。在下一篇文章里，我们会继续深入学习 CSS 的句法，距离踏入前端世界的大门又要近一些了！

## 参考文章

[How CSS works - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/CSS/Introduction_to_CSS/How_CSS_works)