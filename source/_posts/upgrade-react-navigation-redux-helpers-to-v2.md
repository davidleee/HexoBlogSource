---
title: react-navigation-redux-helpers 从 v1 到 v2
date: 2018-09-10 16:40:21
tags:
- react-native
- react-navigation
- redux
- react-navigation-redux-helpers
---

在使用 [React Navigation](https://reactnavigation.org) 的项目中，想要集成 redux 就必须要引入 [react-navigation-redux-helpers](https://github.com/react-navigation/react-navigation-redux-helpers) 这个库。最近整理第三方库的时候，发现这两个库的版本都比较旧了，在尝试更新的时候踩了一些坑，于是就有了这篇文章。

<!--more-->

## Navigator
升级之后，配置上唯一的不同在于 v2 版本中干掉了 `createReduxBoundAddListener(Key)` 方法，取而代之的是 `reduxifyNavigator(Navigator, Key)`。

在 v1 版本中，我们需要把前者构造出来的 `addListener` 作为参数传给 AppNavigator：

```javascript
import {
  StackNavigator,
  addNavigationHelpers,
} from 'react-navigation'
import {
  createReduxBoundAddListener
} from 'react-navigation-redux-helpers'

const AppNavigator = StackNavigator(AppRouteConfigs)
const addListener = createReduxBoundAddListener("root")

class App extends React.Component {
  render() {
    return (
      <AppNavigator navigation={addNavigationHelpers({
        dispatch: this.props.dispatch,
        state: this.props.nav,
        addListener, // --> 就是这里
      })} />
    );
  }
}
```

而在 v2 版本中，使用新方法可以简化上述步骤：
```javascript
import {
  createStackNavigator, // new creator of StackNavigator
} from 'react-navigation'
import {
  reduxifyNavigator
} from 'react-navigation-redux-helpers'

const AppNavigator = createStackNavigator(AppRouteConfigs)
const App = reduxifyNavigator(AppNavigator, "root") // --> 干净清爽！
```

## mapStateToProps
原来 `state.nav` 对应的 `props` 键叫 “nav”，现在改为 “state” 了：
```javascript
// v1
const mapStateToProps = (state) => ({
  nav: state.nav
})

// v2
const mapStateToProps = (state) => ({
  state: state.nav, // nav -> state
})
```

另外，之前为了处理 Android 返回按钮的问题，可能会自定义一个类包裹着上面构造出来的 AppNavigator，然后通过 react-redux 的 `connect` 方法把 `mapStateToProps` 给作用到这个自定义的类上去，比如：

```javascript
class ReduxNavigation extends React.Component {

  /* additional setup for back button handling */

  render() {
    /* more setup code here! this is not a runnable snippet */*
    return <AppNavigator navigation={navigation} />;
  }
}

export default connect(mapStateToProps)(ReduxNavigation)
```

经测试发现，在 v2 版本里，这种操作会报 “undefined is not an object(evaluating ‘state.routes’)” 的错误，猜测可能跟 Props 的键值变化有关。把 `connect` 的调用提前，让它先作用到 AppNavigator 再包裹到自定义类里面即可：
```javascript
const ConnectedNavigator = connect(mapStateToProps)(AppNavigator)
class ReduxNavigation extends React.Component {

  /* additional setup for back button handling */

  render() {
    /* more setup code here! this is not a runnable snippet */*
    return <ConnectedNavigator navigation={navigation}/>;
  }
}

export default ReduxNavigation
```

## Full Example
对于同一个 Navigator， `reduxifyNavigator` 如果在 `connect` 之后调用，会报重复定义`navigation` 属性的错误。所以加上前面的配置过程，完整的例子应该长这样：
```javascript
/* importing the whole world */

const AppNavigator = createStackNavigator(AppRouteConfigs)
// 下面两句顺序不能变
const App = reduxifyNavigator(AppNavigator, "root")
const ConnectedNavigator = connect(mapStateToProps)(App)

class ReduxNavigation extends React.Component {

  /* additional setup for back button handling */

  render() {
    /* more setup code here! this is not a runnable snippet */*
    return <ConnectedNavigator/> // 不需要 `navigation` 参数了
  }
}

export default ReduxNavigation
```

## 参考文章
[Redux integration · React Navigation](https://reactnavigation.org/docs/en/redux-integration.html)
[Redux integration · React Navigation (v1)](https://v1.reactnavigation.org/docs/redux-integration.html)