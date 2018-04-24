---
title: 小心避开 RxSwift 里的坑（Top Mistakes in RxSwift you want to avoid）
date: 2018-04-24 08:56:05
tags:
- Reactive
- RxSwift
- Swift
---

每当我们要学习一样新的语言或者框架时，总是会犯下这样那样的错误。这就是人类学习新知识的方法。下文列出了一些使用 RxSwift 过程中常见的错误，供大家参考。

<!-- more -->

原文链接在文末。

## combineLatest vs withLatestFrom
前者会在内部的任意一个 Observable 发出消息时发出一个总的消息，所以把两个按钮的 tap 事件 combineLatest 不是一个合理的做法。这种情况就要看看后者的使用方式了。

## Observable 应该延迟初始化
当一个 Observable 是为了把耗时操作的结果通知出去时，这个 Observable 本身应该被延迟初始化，这样才能避免在有人 subscribe 之前就在做那个耗时操作。
举个栗子：
```swift
func rx_myFunction() -> Observable<Int> {
    let someCalculationResult: Int = calculate()
    return .just(someCalculationResult)
}
```

`calculte()` 是一个耗时操作，这样用户在调用这个方法的时候就已经在跑真正的计算了，而我们写这个方法的本意应该是有人 subscribe 的时候才去执行操作。所以上面的方法应该做一下这样的小修改：
```swift
func rx_myFunction() -> Observable<Int> {
    return Observable.deferred {
        let someCalculationResult: Int = self.calculate()
        return .just(someCalculationResult)
    }
}
```

## DisposeBag 的错误使用
这玩意儿的作用是把一堆 `Disposable` 在某个对象 `deinit` 的时候全部结束掉，所以这个管理对象的选择就尤为重要。

比如我们在 tableViewCell 里面进行了一些订阅，该用的 DisposeBag 是 cell 本身声明的一个属性，而不应该直接用 VC 里面的那个，因为 cell 会发生重用，所以这里的 Disposable 的管理应该更积极一点：
```swift
dataSource.configureCell = { _, tableView, indexPath, cellViewModel in
	let cell: TheCell = tableView.dequeueCell(at: indexPath)
	cellViewModel.image
		.drive(cell.avatarView.image)
		.disposed(by: cell.disposeBag)
	return cell
}
        
//somewhere in TheCell.swift file
class TheCell: UITableViewCell {
	private(set) var disposeBag = DisposeBag()
    
	override func prepareForReuse() {
		disposeBag = DisposeBag()
	}
}
```

## 没有在 UI 层使用 drivers
Driver 的设计是为了避免线程的混乱，对于 Driver 的订阅的通知都会发生在主线程上，所以可以降低线程问题的概率。具体要看看这个东西的详细用法。

##  异常处理
当 Observable 抛出异常的时候，它会终止整个流程。如果使用了 `flatMap` 这类转换方法，那抛出异常的时候被终止的是源头的主流程。

也就是说，如果把一个可能抛出异常的流程绑定到了按钮的点击事件上，一旦这个流程抛出了异常，按钮的点击事件就再也不会响应了。

解决方法是使用 `Observable<Result<User>>` 或者 `materialize()` 之类的方法。

## 同一个 Observable 订阅多次
Observable 是不可变的类。每一个处理方法都是返回一个新的 Observable 而不会对原来的那个做任何变动。

在需要共享某些流程的结果时，可能会对某个 Observable 进行分别处理和订阅，这时候就应该用 `share` 或者 `shareReplay(1)` 来避免事件的重复发出：
```swift
let items: Observable<[Item]> = itemsProvider.items
	.share()
    
let count: Observable<Int> = items
	.map { $0.count }
    
items.subscribe(onNext: { items in 
	//do something with items
}) 
    
numberOfItems.subscribe(onNext: { count in 
	//do something with count
})
```

> 如果不加 `share()`，下面的两次 `subscribe` 就会触发两次 `itemProvider.items` 的 `get` 方法，但是显然我们只需要获取一次就可以满足下面两个订阅了

## 过度使用 subjects & variables

Rx 的世界应该是由“不可变变量”组成的，一旦一个事件已经形成，那么我们不应该对它本身做出任何改动，而是操作事件的流向最终得到我们想要的结果。

而 Subjects & Variables 正是 Rx 世界里的“可变变量”。

不是说不能用它们，而是说我们在大多数时候并不需要用上它们。在更多情况下，我们可以用 `merge` 、`concat`、`publish`&`refCount` 、`defer` 和其他一些方法去替代这两个玩意儿。



* 原文链接：[Top mistakes in RxSwift you want to avoid - Code in a suit](http://adamborek.com/top-7-rxswift-mistakes/)