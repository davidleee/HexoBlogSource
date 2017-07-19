---
title: 深入理解 Swift 中的问号感叹号
date: 2017-07-14 21:16:57
tags:
- Swift
- Optional
- nil
---

对于写惯了 OC 代码的程序员来说，不判空直接调用对象方法可能已经成为习惯了；而当方法的返回值是对象时，通常也是拿来就用。这些情况在 Swift 下都不存在了，因为 Swift 中出现了一个全新的概念：Optional（? & !）。

<!-- more -->

Optional 用于表示一种值可能为空的对象类型。一个 Optional 对象表示了两种可能性：要么对象有值，你可以通过 “unwrap” 去获取到这个值；要么对象里面没有任何东西。

> unwrap（解包）：在对象后加 “?” 或 “!” 称为将对象 “unwrap”，可以获取到 Optional 里面的关联值

Optional 这个概念在 C 语言或 Objective-C 里面并不存在。在 OC 中最接近的概念是：本来要返回对象的方法可能会返回 nil，这个 nil 表示“没有有效的对象可以返回”；然而，这只在对象身上有效，它不能作用在结构体、基础 C 类型或枚举上。这些类型的变量如果没有值，OC 会用 `NSNotFound` 来表示，它需要方法的调用者意识到这些特殊返回值的存在，并作出特殊的处理。

Optional 解决了上述问题，在 Swift 中，Optional 可以处理任何类型的空值，而不需要用一个特殊的常量去表示。

举个栗子：
当我们需要将字符串转换为数字时，在 Swift 中会使用 `Int` 的构造方法，如下：
```Swift
let possibleNumber = "123"
let convertedNumber = Int(possibleNumber)
```
此时，`convertedNumber` 就是一个 Optional，看一下文档可以发现，这个构造方法返回的是 `Int?`。

原因是这个构造器可能会失败： `possibleNumber` 也许并不能被转化为数字。这里的 `?` 表示返回的对象是一个可选值，它可能是某个 `Int` 类型的对象，也可能什么都没有。（它不可能是别的类型的对象，因为 Swift 是强类型的）

## nil
这里可以套用 OC 中的概念，`nil` 表示空值，但是在 Swift 中，它只能被赋值给 Optional 对象。当声明一个 Optional 的变量又没有给它赋值时，它会自动被赋值为 `nil` ：
```Swift
var surveyAnswer: String?
// surveyAnswer == nil
```

> 本质上 Swift 的 `nil` 跟 OC 的 `nil` 是不一样的。
> 在 OC 中，`nil` 是指向一个不存在对象的指针；在 Swift 中，`nil` 不是一个指针，它是一个带有特定类型的表示数值缺失的值，任何类型的 Optional 都可以设置为 `nil` 而不只是对象类型。

## If 和强制解包
可以使用 `if` 来判断一个 Optional 对象是否有值，就像常见的判空操作。在判空后，这个 Optional 对象可以使用 `!` 来强制解包，这相当于告诉编译器：“我确定这个 Optional 对象肯定有值，直接取出来用吧！”

举个栗子：
```Swift
if convertedNumber != nil {
    print("convertedNumber has an integer value of \(convertedNumber!).")
}
```

> 当 `!` 被用在一个空值时，你的程序就会“卡蹦”一声崩掉！

## Optional Binding
这个机制可以用来判断一个 Optional 对象是否有值，如果有值就将它复制给一个局部变量或常量，否则不执行任何操作。

我们用 Optional Binding 来改写上一小节中的例子：
```Swift
if let actualNumber = Int(possibleNumber) {
    print("\"\(possibleNumber)\" 是一个整型数字 \(actualNumber)")
} else {
    print("\"\(possibleNumber)\" 不能被转化为整型")
}
```

当 `Int()` 返回的对象有值，这个值就会被直接赋给前面的 `actualNumber` ，所以这个变量就不是一个 Optional，可以不需要解包而直接使用了。

在这种用法下，`if` 原来的作用还是存在的，可以用逗号分隔不同类型的判断，比如这样：
```Swift
if let firstNumber = Int("4"), let secondNumber = Int("42"), firstNumber < secondNumber && secondNumber < 100 {
    print("\(firstNumber) < \(secondNumber) < 100")
}
// 如果其中一个 Optional 没有值，或者最后那个判断的结果为 false，整个 if 判断会直接返回 false
```

> 通过 Optional Binding 声明的变量的作用于只在这个 `if` 之内，除非用 `guard` 去声明，详情参见官方文档 [Early Exit](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/ControlFlow.html#//apple_ref/doc/uid/TP40014097-CH9-ID525)

## 隐式解包
有时候，在特定的代码结构下，一个 Optional 对象可以被确保永远都有值（或者说理应永远都有值）。这种时候，每次使用这个对象都进行判空和解包就显得非常多余了，于是我们可以在声明这个对象的时候用隐式解包来处理：
```Swift
let forcedString: String!
print(forcedString) // 不需要写成 “forcedString!”
```

事实上，例子中的 `forcedString` 还是一个 Optional 没变，但是我们让它在使用的时候自动解包，不需要我们手动加 `!` 了。

## 链式调用
如果我们要取得的对象被包裹在了一层又一层的 Optional 之中，取得它的过程可能非常繁琐：
```Swift
var label: UILabel?
if label != nil {
	if let temp1 = label.text {
		if let temp2 = temp1.hashText {
			...
		}
	}
}
```

这时候，可以使用链式调用的方式改写：
```Swift
if let hashText = label?.text?.hashText {
	...
}
```

在这句话当中，`?` 表达的意思是：“如果这个对象有值就取出来，继续下面的步骤；如果没有值，就当我没写过这句话吧”。

## 默认值
在一些情况下，我们会想要 Optional 对象为 `nil` 的时候给出一个默认值。比如我们使用一个 `String?` 给 `label.text` 赋值时，我们并不希望设置一个 `nil` 上去，因为那会让 UILabel 的高度变为0。
一种很简便的写法是这样的：
```Swift
let s: String?
s = ...
label.text = s ?? "placeholder"
```

这样，当 `s` 为空时，`label.text` 的值就会是 “placeholder”。

## 总结
Swift 中的 Optional 其实是一个 enum：
```Swift
enum Optional<T> {
	case none
	case some(T)
}
```

而它现在所见到的使用方法都可以认为是 Swift 的语法糖：
```Swift
let x: String?
// 等价于
let x = Optional<String>.none

let x: String? = "hello"
// 等价于
let x = Optional<String>.some("hello")

let y = x!
// 等价于
switch(x) {
	case .some(let value): y = value
	case .none: // 抛个异常并整死你的应用:)
}

if let y = x {
	y.doSomething()
}
// 等价于
switch(x) {
	case .some(let y):	y.doSomething()
	case .none: break
}
```

> 什么？你想问这是哪门子的 enum？推荐你去看看官方文档 [Enumeration](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Enumerations.html#//apple_ref/doc/uid/TP40014097-CH12-ID145)

## 参考资料
* [The Swift Programming Language (Swift 4)](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID309)
* [Developing iOS 10 Apps with Swift](https://itunes.apple.com/us/course/developing-ios-10-apps-with-swift/id1198467120)
