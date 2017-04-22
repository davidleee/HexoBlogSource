---
title: 基于 Redux 的 React Native 应用架构（2/2）
date: 2017-04-21 11:49:28
tags:
- React Native
- Redux
- react-redux
- redux-action
- selector
---

在上一篇文章里，我们了解了 Redux 的核心思想和基本用法。不过从例子中可以发现，如果按照这个势头继续开发下去，我们将会写出一个庞大的 actions.js 和 reducers.js，这是我们不想看到的。

<!-- more -->

这篇文章会介绍一些好用的第三方框架，有了它们的帮助，我们的项目结构才能更加清晰，也能帮我们省下不少写重复代码的时间。d(`･∀･)b

> P.S. 文中提到的所有第三方库都是可以被替代的，这样的选择是因为它们适合我们现在的项目框架，欢迎大家在评论区给出自己的见解。

---

# 问题在哪？

光使用 Redux 来构建项目，虽然比起纯粹的 React Native 要规整一些，不过仍然会存在许多不尽人意的地方。下面列出了一些我们项目中的前辈已经遇到过的问题：

* Q1 对于一个 Component 来说，只会关注自身的 state 变化，无法监听到外部的 state 变化
* Q2 所有 Action 和 Reducer 的格式都是类似的，导致有大量的重复代码产生

---

# 怎么办？

## A1: react-redux

通过 react-redux 的 `connect` 方法，我们可以将所需的 `state` 参数包装为一个叫 `props` 的变量传给 Component。

举个栗子：

在上一篇文章中我们通过发起一个 Action 修改了 `state.username`，因为是 `UserInfo` 自己发起动作然后自己接收变化，所以一切看起来都很美好。然而，如果另外一个叫做 AuthInfo 的地方也需要知道 `state.username` 变化了，那怎么办呢？使用 react-redux 我们可以这样做：

```javascript
// AuthInfo.js

import { connect } from 'react-redux'
import ...

const mapStateToProps = (state) => { // 1
  return {
    username: state.username
  }
}

const mapDispatchToProps = (dispatch) => {} // 2

class AuthInfoView extends Component {
  // ...
}

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(AuthInfoView) // 3
```

1. 将 `state` 中的参数映射到 props 变量上
2. 将我们自己定义的方法映射到 `props` 变量上
3. 经过 1 & 2 两个方法的洗礼，`props` 变量已经准备就绪，在这里传给我们自定义的视图

`connect` 方法其实只做了两件事情：

1. 将 `state` 和 `dispatch` 解开，并封装成 `props` 参数（为什么要解开？还记得纯函数吗？）
2. 根据传入的 `AuthInfoView` 重新构造一个 View，并将上面封装好的 `props` 传递过去

## A2 redux-actions

先来复习一下上一篇文章中 `actions.js` 的代码：

```javascript
// actions.js

export const CHANGE_USERNAME = 'CHANGE_USERNAME' 

export function changeUsername(username) {
  return { type: CHANGE_USERNAME, username}
}
```

如果用 redux-actions 的话，将会变成这样子：

```javascript
// actions.js

import { createAction } from 'redux-actions'

export const CHANGE_USERNAME = 'CHANGE_USERNAME'
export let changeUsername = createAction(CHANGE_USERNAME) // 1
```

1. 按照 Flux 的标准 Action 形态，Action 里面应该只有两个属性，一个是 `type` ，另一个是 `payload`；所以一个规范的 `changeUsername` 应该是这样的：`{ type: CHANGE_USERNAME, payload: username }`。而 `createAction` 就是用这种方式构建 Action 的。

`createAction` 还可以传入其他的参数，这里就不展开了。它的作用主要是省去了我们写对象构造的那一系列代码，当 Action 中带的 `payload` 越复杂的时候，它的优点就会越发地体现出来。

如果你以为 redux-actions 只是优化了 Action 那就太低估它了！让我们先来复习一下 `reducers.js`：

```javascript
// reducers.js

import { CHANGE_USERNAME } from './actions'

const initialState = {
  username: ""
}

function demoApp(state = initialState, action) {
  switch (action.type) {
    case CHANGE_USERNAME:
      return {
        ...state,
        username: action.username
      }
      default:
      	return state
  }
}

export default demoApp
```

按照上面这个情况，我们只有 `CHANGE_USERNAME` 这个 Action 需要处理，于是我们可以用 redux-actions 这样改写：

```javascript
// reducers.js

import { handleAction } from 'redux-actions'
import ...

const initialState = {
  username: ""
}

export default handleAction(CHANGE_USERNAME, (state, action) => initialState, initialState) // 1
```

1. 这里的三个参数分别是：Action 的 `type`、Reducer 中的处理方法和默认返回值

---

参考资料：

* [How does connect work in React Redux](https://medium.com/@juinchiu/how-does-connect-work-in-react-redux-a25c68692335)
* [connect源码](https://github.com/reactjs/react-redux/tree/master/src/connect)