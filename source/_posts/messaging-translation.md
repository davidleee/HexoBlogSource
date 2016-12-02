---
title: 【翻译】消息传递
date: 2016-12-02 21:58:58
tags:
- iOS
- Translation
- runtime
---

> 本文翻译自苹果官方文档[《Messaging》](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html#//apple_ref/doc/uid/TP40008048-CH104-SW1)。如有不正确的地方，欢迎在评论内指出。

<!-- more -->

### 传说中的 objc_msgSend 函数

在 OC 内，消息（messages）会一直拖到运行时（runtime）才会绑定到方法实现（implementations）上。编译器会将下面这样的语句：

```
[receiver message]
```

转化为对 objc_msgSend 这个消息传递函数的调用。

这个函数会将消息的接收者（上面的 receiver）和方法名（上面的 message，这个“方法名”也就是 method selector）作为它的两个主要参数：

```
objc_msgSend(receiver, selector)
```

任何传给方法的参数，都会原封不动地转交给 objc_msgSend：

```
objc_msgSend(receiver, selector, arg1, arg2, ...)
```

这个强大的消息传递函数真是为动态绑定（dynamic binding）操碎了心：

* 它首先要找到 selector 对应的方法实现。因为同一个方法可能会在不同的类里面有不同的实现，如何判断要使用哪一种实现取决于消息接收者是哪个类。
* 然后它要调用那个方法实现，并把方法实现所需要的接收者对象（一个指向对象数据的指针）和所有参数都传递过去。
* 最后，它会把方法实现的返回值作为自己的返回值给出去。

> 嘛，objc_msgSend 已经辣么忙了，你就不要在你自己写的代码里面调用它了吧？嗯？—— Apple.

能实现消息传递这一特性的关键，在于编译器在编译每一个类和对象时所使用的结构（structure）。类的结构包括了两个必备的元素：

* 一个指向父类的指针。
* 一个类的分派表（dispatch table）。从这张表上，你可以查到方法的 selector 在这个类里对应的地址（class-specific addresses）。比如说，根据 `setOrigin::` 的 selector，你可以找到这个方法实现的地址（address of the procedure that implements `setOrigin::`）。

在一个对象被创建出来的时候，它会被分配一块内存，而且它的成员变量也会进行相应的初始化。在它这些变量当中，排头的就是一个指向它类结构（class structure）的指针。这个指针叫做 `isa`，通过`isa`，对象可以访问它自己的类，而通过它自己的类，它还可以访问到它继承下来的所有父类。

> 虽然严格来说，`isa` 不是 Objective-C 语言的一部分，但它是对象与 Objective-C 运行时系统打交道所必备的东西。一个对象必须与一个 `struct objc_object`（在 objc/objc.h 里声明）等价。尽管你几乎不会有需要创建一个根对象（root object），但是你要知道，所有继承自 `NSObject` 或 `NSProxy` 的对象都会自动带上一个 `isa` 变量。

关于消息传递的框架，可以参考下面这个图：

![message](/uploads/messaging-translation/messaging1.gif)￼


当有消息被发送到对象的时候，消息传递函数会顺着 `isa` 指针找到类结构，在这个地方它可以从分派表里找到对应的方法实现。
如果很不幸的没找着，`objc_msgSend` 会继续顺着指向父类的指针找上去，企图从父类的分派表里找到对应的方法实现。不断地失败让 `objc_msgSend` 越战越勇，直到它来到了 `NSObject` 这里。
如果它找到了，就像上面提到的那样，它会调用对应的方法，并把所有参数一股脑丢给方法实现。

这就是运行时选择方法实现的一种途径，或者，用面向对象编程的行话来说，这是将方法动态绑定到了消息上。

为了加速消息传递的过程，运行时系统缓存了它用到的 selectors 和方法的地址。每一个类都有一个独立的缓存，这个缓存会将继承下来的方法也一并保存了，就像保存自己声明的方法一样。在搜索分派表之前，消息传递会惯例先查一下消息接收对象的类的缓存（因为理论上，一个被起码用过一次的方法，会更有可能被再用一次）。如果方法的 selector 在缓存里，那么消息传递就只会比方法调用慢那么一丢丢。一旦程序运行了足够久，缓存已经被充分调动起来了，那几乎所有发送的消息都可以在缓存里找到对应的方法。缓存的大小会随着程序的运行而慢慢地增长，以便让新的消息可以有地方妥当地安置下来。

### 使用隐藏参数
当 `objc_msgSend` 找到了想要的方法实现之后，它会调用那个方法实现并把所有传进来的参数都丢过去，除此之外它还会传过去两个不为人知的隐藏参数：

* 接收者对象
* 方法的 selector

这两个参数告诉了我们调用每一个方法实现的消息表达式的主要信息。为什么说是“隐藏”的呢？因为它们不会出现在声明方法的源码中，它们只在编译的时候才会被加入到方法实现里面。

尽管这两个参数没有被显式声明，但是在代码里还是可以引用它们（就像代码里可以引用对象里的其他实例变量那样）。在方法内，可以用 `self` 来引用接收消息的对象，用 `_cmd` 来引用它本身的 selector。举个栗子，下面的代码里，`_cmd` 表示 `strange` 方法的 selector，而 `self` 则表示接收 `strange` 这个方法的对象。

```
- strange
{
    id  target = getTheReceiver();
    SEL method = getTheMethod();

    if ( target == self || method == _cmd )
        return nil;
    return [target performSelector:method];
}

```

这两个参数里面，`self` 是比较有用的哪那一个。它其实是成员变量可以在方法里被调用到的关键。

### 获取方法地址
唯一可以绕开动态绑定的方法，是获取到方法的地址，然后把它当做一个函数那样直接调用。这种做法用到的机会不多，但是如果一个方法会被一连串地反复调用，这样做可以避免每次都进行消息传递从而节省一些时间。

你可以用 `NSObject` 里声明的方法 `methodForSelector:` 来获取到实现一个方法的程序的指针，然后用这个指针来直接调用这段程序。`methodForSelector:` 返回的指针在使用前要小心地转换成正确的方法类型，在这个转换里，返回值和参数类型都是必不可少的。

再举个栗子，它将会告诉你 `setFilled:` 方法的执行程序是怎么被调用的：

```
void (*setter)(id, SEL, BOOL);
int i;

setter = (void (*)(id, SEL, BOOL))[target
    methodForSelector:@selector(setFilled:)];
for ( i = 0 ; i < 1000 ; i++ )
    setter(targetList[i], @selector(setFilled:), YES);
```

前两个传递给执行程序的参数就是接收对象（`self`）和方法 selector（`_cmd`）。这些变量对于方法的语法来说是隐藏的，但在方法作为函数调用的时候，它们必须显式地传递过去。

用 `methodForSelector:` 来规避动态绑定可以节省消息传递过程所浪费的大多数时间。然而，这种节俭只有特定方法被频繁调用很多次的时候才有意义，比如像上面例子中的 `for` 循环那样。

需要注意的是，`methodForSelector:` 是 Cocoa 运行时系统所提供的，它并不是来自 Objective-C 这门语言本身。
