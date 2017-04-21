---
title: 基于 Redux 的 React Native 应用架构（1/2）
date: 2017-04-20 09:47:03
tags:
- React Native
- Redux
- Flux
- JavaScript
---

> 自从 Facebook 把 [React Native](https://facebook.github.io/react-native/) 给开源了之后，使用前端技术进行移动应用开发的趋势已经越来越明显，现在连微软爸爸都用上了 React Native（[ReactXP](https://github.com/Microsoft/reactxp)），我们还有什么理由不来一点前端技术尝尝呢？

<!-- more -->

这篇文章是学习了公司现有项目框架、外加查阅一些官方文档之后的总结，受限于我仅有的一丢丢前端知识，如果有错误或没讲清楚的地方，欢迎在评论里给我留言 :)

因为要把现有项目讲清楚内容较多，所以分为两个部分来总结。这篇是第一个部分，主要涉及 Redux 思想的简单应用。

---
# 试想一下...

我们从一个实际可能会遇到的需求出发，考虑一下 Native 的实现和 React Native 的实现有什么不同。

## 来需求了

> 用户在登录之后，在首页要显示用户的头像和名字，然后还应该可以在某个地方看到自己的全部信息。

( ºωº ) 那还不简单！用户登录之后拉一下服务器上的用户信息，找个合适的界面把信息一股脑显示出来就好了，done！

## 简单地改一下

> 用户想要在我们的 App 上编辑自己的信息，我们简单点来，直接在信息显示的界面提供个编辑的入口就好了。

∑(￣□￣;) ......嗯，这个要求也合情合理。

编辑之后发个消息通知其他地方，比如首页的头像和名字就可能要更新...某个列表似乎会根据用户性别来决定显示的内容，不过什么人会用着用着修改性别？！

## 来，这个更加简单

> 果然修改一下用户信息很简单嘛，那再改回去也是一个道理吧，提供一个撤销修改的按钮吧。

Σ(°Д°; 这！

---
# 为什么用 Redux

如何优雅地实现上面的需求呢？如果我们的界面是与数据绑定的，并且数据的每一个状态我们都可以追踪到，那我们就不需要理会界面的更新，而可以专注于数据的修改了。

React Native 已经为我们提供了视图与数据绑定的机制，那么状态要怎么追踪呢？

## 什么是 Redux

> [Redux](https://github.com/reactjs/redux) 是一个为 JavaScript 应用设计的可预测的状态容器。

简单来说，Redux 使用了一种叫做 Action 的东西，每一次对状态的变更（也就是对数据的变更）都需要通过 Action 来进行，然后通过另一个叫做 Reducer 的东西将 Action 和数据联系起来。下面这张流程图应该可以帮助你理解：

{% img center /uploads/Build-App-With-Redux/flux.png 600 600 Flux %}

> 这张图来自 [Flux](https://facebook.github.io/flux/docs/in-depth-overview.html#content)，Redux 可以被理解为是 Flux 的具体实践。 

顾名思义，Action 是一个操作，也可以理解为将要发生的事件，例如“修改用户名”、“撤销修改”；而 Reducer 则位于上图 Dispatcher 和 Store 之间，负责将 Action 反应到对 Store 的修改上。

## 初衷

在上面的需求中，如果可以记录下用户的每一个操作和操作后的状态，那么当用户想要撤销某项操作时，只需要将上一个操作给无效掉就可以了。更暴力一些，我们把应用启动至今的所有用户操作都重播一遍，唯独不执行最后一次操作，就实现了“撤销”的功能了吧？

Redux 提供的机制正好满足了这个需求。

## 不止如此

React Native 实现的应用可以类比为使用 JavaScript 实现的单页应用，随着开发日趋复杂，我们的应用需要管理的状态（state）会变得越来越多，管理的难度也将成倍增加。

Redux 提供的机制，通过限制数据更新发生的时间和方式，使得 state 的变化变得可以预测，后续的开发也可以放手施展了。

---
# 那该怎么做呢？

接下来我们分别看看 Redux 中主要的三部分是怎么实现的。

## Action

为了描述一个动作，我们需要给这个动作一个名字和这个动作要操作的数据。举个栗子，对于用户想要改变用户名的情况，我们可以这样声明这个动作：

```javascript
// actions.js

export const CHANGE_USERNAME = 'CHANGE_USERNAME' // 1

// 2
export function changeUsername(username) {
  return { type: CHANGE_USERNAME, username}
}
```

1. 首先我们将这个动作称为 `CHANGE_USERNAME`，理论上每一个 Action 都应该有一个独一无二的名字
2. 通过一个创建函数，创建这个 Action 对象返回给调用方

例子里的动作名和创建函数都是 `export` 的，因为一般会把同一个模块的动作声明放到一起，方便外部的调用。

## Reducer

需要注意的是，Reducer 必须是一个[纯函数](https://en.wikipedia.org/wiki/Pure_function)。它不能修改传入的参数并且不能有任何副作用，它必须要保证只要两次传入的参数相同，得出的结果也要相同。

先看代码：

```javascript
// reducers.js

import { CHANGE_USERNAME } from './actions'

// 1
const initialState = {
  username: ""
}

function demoApp(state = initialState, action) { // 2
  switch (action.type) { // 3
    case CHANGE_USERNAME:
      return { // 4
        ...state,
        username: action.username
      }
      default:
      	return state // 5
  }
}

export default demoApp
```

1. 在 Redux 应用中，所有的 `state` 都被保存在一个单一的对象中
2. 这里用到了 ES6 中的参数默认值语法，省去了初始化 `state` 的过程
3. 当需要处理多个 Action 的时候，可以用 `switch`、`if/else` 等方式区分不同的处理方式
4. 再次强调，Reducer 必须是一个纯函数，所以使用对象展开运算符把传入的 `state` 展开，避免修改到里面的参数
5. 还是因为纯函数的原因，在不处理任何 Action 的时候，需要将传入的 `state` 原封不动地再传回去

通过上面的 Reducer，当有 `CHANGE_USERNAME` 这个动作发生时，应用会通过 `demoApp` 方法来更新 `state` 中的 `username` 参数，如果有视图绑定在这个参数上，则界面也会发生相应的改变。

## Store

有了 Action 和 Reducer，我们还需要将这二者联系起来的途径，称为 Store。

Store 还具有维持应用 `state` 的职责，所以它在应用中也是单一存在的。当需要拆分数据处理逻辑的时候，应该做的是拆分 Reducer 而不是创建多个 Store。

继续来看代码：

```javascript
// store.js

import { createStore } from 'redux'
import demoApp from './reducers'
let store = createStore(demoApp)
```

没啦！我们需要用到的是 `store` 里面的方法，所以主要还是来看看怎么使用它们吧。

## 调用

直接来看代码，我们假设已经存在一个用户信息界面，用户将在这里修改自己的名字：

```javascript
// UserInfo.js

import { changeUsername } from './actions'
import { store } from './store'

store.dispatch(changeUsername("Lee")) 
```

在这里，我们通过 `changeUsername` 方法构造了一个 Action，并告诉它我们要把名字改成什么；通过 `store` 中的 `dispatch` 方法，这个 Action 就会被发送 Reducer 中进行处理。

只是写到这里，得益于纯函数的实现，我们就已经可以给上述代码中的所有方法写单元测试了，就是这么简单。

---
# 结语

上面的例子只涉及到了 Redux 最基本的使用方式，到目前为止还不能做出什么有用的东西来，不过用来让我们感受 Redux 的主要思想应该是足够了。

在下一篇文章中，才会讲到 Redux 应用框架的搭建。其中会用到一些能有效减少代码量的第三方框架（从上面的例子应该不难想象，当程序变复杂之后，这几个类会变得多么庞大）。如果在学习了基础知识之后还是看不懂一些现有项目的代码的话，看完这些第三方框架的使用说不定就懂了呢！

更多更详细的资料可以看参考资料中的网站。关于 React Native 的部分建议上官网看原版，我发现中文网的翻译稍有缺失，更新也可能不那么及时。

---
参考资料：
[React Native](https://facebook.github.io/react-native/)
[React Native 中文网](http://reactnative.cn/)
[Redux 中文文档](http://cn.redux.js.org/)
[RNPlayground](https://github.com/luhui/RNPlayground)