---
title: 简易 Core Data 入门
date: 2016-05-09 09:59:27
tags:
- iOS
- Core Data
---

也不知道是我搜索的方法不对还是别的什么问题，网络上很少见 Core Data 的入门教程，所以这篇东西就这样定位了：比看官方文档轻松得多，却可以马上在自己的 App 里用上 Core Data。
不求深入到可以拿出去显摆，但是起码要可以用上手，并且能分享一篇入门指南，就像现在这样:)

<!-- more -->

## Launch our Xcode!
不管怎么样，先把咱熟悉的 Xcode 召唤起来，创建一个新项目。本项目的完整代码可以在[这里](https://github.com/davidleee/CoreDataTryout)找到。对了，创建项目的时候记得把下面这个小勾给勾上，让 Xcode 帮我们创建模型文件。
{% img center /uploads/easy-core-data/use_core_data.png 创建项目 %}

然后就可以看到项目中出现一个后缀为 xcdatamodeld 的文件，这个文件相当于 Core Data 的 storyboard （不知道 [storyboard](https://developer.apple.com/videos/wwdc/2012/?id=407)？）。选中它，你会看到这样几个东西：
{% img center /uploads/easy-core-data/model_document.png xcdatamodeld %}

在这篇文章里，我们就只关注 Entities 部分。

## 浅入浅出
想要详细了解 Core Data 里面的结构关系的话，可以去翻翻[官方文档](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreData/cdProgrammingGuide.html#//apple_ref/doc/uid/TP40001075)。但是一开始接触的时候，不外乎就是这么几个类：

* NSManagedObjectModel

这玩意儿是什么呢？说白了，你可以认为它就是我们的 xcdatamodeld 。它包含了我们自己定义的实体( Entitiy )、实体内部的属性( Attribute )和实体之间的关系( Relationship )，就像是我们在设计数据库的时候画的E-R图。原则上，这个模型越是完善，你的 App 对 Core Data 的支持就越好。

* NSManagedObjectContext

按照官方文档的[说法](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreData/Articles/cdBasics.html#//apple_ref/doc/uid/TP40001650-SW3)，`NSManagedObjectContext` 就像是一个智能的工作台。当你想从数据库中获取数据的时候，这些数据会被暂时复制一份到这个工作台上，然后你就可以对它们进行操作了。这样，除非你确实进行了一次保存，否则数据库里的真实数据是不会被改变的。

而在实际使用中，我们最频繁接触的也是这个 `NSManagedObjectContext`。

* NSPersistentStoreCoordinator

从名字大概可以猜到，这个类是不同数据库存储之间的协调者。

在 App 里面创建出来的数据对象和实际保存着的数据之间，存在着一个持久栈( Persistence Stack )。这个栈的最顶层是 `NSManagedObjectContext`，而它的最底层则是一个个数据文件( Persistent Object Store )，在它们之间的就是 `NSPersistentStoreCoordinator`。我们因为 `NSManagedObjectContext`的存在，而不需要直接操作这些数据文件；而且 Persistent Store Coordinator 的存在，是基于外观模式的设计，这就使得 `NSManagedObjectContext` 不需要面对多个数据文件，只和 Coordinator 打交道就可以了。

整个持久栈看起来就是这个样子：
￼{% img center /uploads/easy-core-data/persistence_stack.png  Persistence Stack %}

好，铺垫了那么久，也该用实践来检验一下真理了。

## 准备工作
什么？上面讲了那么久的三个东西，Xcode 已经帮我们在 AppDelegate 里都准备好了？
嗯，没错，很贴心嘛。
￼{% img center /uploads/easy-core-data/r_u_kidding.png  250 250 %}

不过我们还是来简单理一理这些代码里面的关系吧。

可以发现，这几个对象的创建顺序是这样的 NSManagedObjectModel -> NSPersistentStoreCoordinator -> NSManagedObjectContext，跟持久栈的关系很一致。
不过在此之前，我们还是需要拿到我们的 xcdatamodeld 才可以创建我们的模型：

```objc
	NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"CoreDataTryout" withExtension:@"momd"];
	_managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
```

然后用这个模型来创建 `NSPersistentStoreCoordinator`：

```objc
	_persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:[self managedObjectModel]];
  NSURL *storeURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:@"CoreDataTryout.sqlite"];

  ...

  if (![_persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:storeURL options:nil error:&error]) {...}
```

程序会在 App 的 document 目录中创建一个名为 CoreDataTryout.sqlite 的文件，`NSSQLiteStoreType` 是我们选择的数据库类型，我们还可以选择 XML（ iOS 上不支持）、Atomic 或者 In-Memory，如果这些类型你都不喜欢，你还可以创建[自定义存储类型](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreData/Articles/cdPersistentStores.html#//apple_ref/doc/uid/TP40002875-SW6)。

最后，在 `managedObjectContext` 的 setter 里面，我们给它装上准备好的 `persistentStoreCoordinator` 就大功告成了：

```objc
  _managedObjectContext = [[NSManagedObjectContext alloc] init];
  [_managedObjectContext setPersistentStoreCoordinator:coordinator];
```

注意到在 `application:didFinishLaunchingWithOptions:` 的最后还调用了一个 `saveContext` 方法。

Core Data 里面的数据都是自动保存的，也就是说，你完全可以在代码里把实体的对象创建出来，乱改一通，然后留下一个帅气的背影，奔向下一段代码。
 `managedObjectContext` 在发现有数据被改动了之后，会在一个*合适的时机*保存这些更改。这个保存也许不是数据变化后马上进行的，但是也足够智能可以让我们免去时不时保存一下的烦恼。有些时候（比如你暂停了你运行中的程序） `managedObjectContext` 会来不及做这些工作，或者你想要确保当下的数据被写进了数据库里，那么这个 `saveContext` 就会派上用场了。

## 创建实体
一切准备就绪，我们终于要真正开始操作我们的数据库了！

先在我们的 xcdatamodeld 文件里面添加一个 Entity ，随便起个名字叫 User 好了，然后在右边 Attributes 那一栏里面添加一些属性。
{% img center /uploads/easy-core-data/attributes.png Attributes %}

Attributes 都是强类型的，所以如果你不手动给它们指定类型，编译器就会直接报错。能够写入数据库的类型还是挺丰富的，值得一提的是，数字和布尔值最终都会被转换成 `NSNumber` 来处理，而图片这一类比较大的文件就必须转换成 `NSData` 再写入了。

接下来，让 Xcode 帮我们生成这个实体的子类：
{% img center /uploads/easy-core-data/create_subclasses.png 创建子类 %}


选择一下 Model 和想要子类化的 Entity （有的项目会使用多个 Model，我们这里就只有一个）：
{% img center /uploads/easy-core-data/choose_model.png 选择 Model %}
{% img center /uploads/easy-core-data/choose_entity.png 选择 Entity %}


然后点 create ，你会发现项目文件夹中多两个文件 User.h/m。这两个文件由 Xcode 生成也由 Xcode 管理，所以如果我们改动了里面的内容，Xcode 可能就会跟我们发牢骚了。（如果确实需要添加某些功能，可以使用 category ）

如今这个类已经全权代表了我们在模型里面创建的实体，它的 property 和我们添加的 attributes 是一一对应的关系，也就是可以直接从这些属性访问到实体的变量了。

多说无益，赶紧来动动手！
{% img /uploads/easy-core-data/accept_challenge.png 250 250 %}

## Write the code, change the world!
首先拉一个简陋的界面出来给我们的用户输入信息，下面放一个输出信息的地方，省的我们每次都跑到 Debug Area 里面看：
{% img center /uploads/easy-core-data/main_vc.png 500 500 Main VC %}


把所有控件都 hook 起来，然后在“写进去”和"读出来"两个按钮的 IBAction 里写代码。
首先把数据写进数据库：

```objc
	- (IBAction)writeAction:(id)sender
	{
	    AppDelegate *appDelegate = [[UIApplication sharedApplication] delegate];
	    NSManagedObjectContext *context = appDelegate.managedObjectContext;

	    User *newUser = [NSEntityDescription insertNewObjectForEntityForName:@"User" inManagedObjectContext:context];
	    newUser.name = self.nameTextField.text;
	    newUser.age = @([self.ageTextField.text intValue]);
	    newUser.height = @([self.heightTextField.text intValue]);
	    newUser.weight = @([self.weightTextField.text intValue]);

	//    [context save:NULL];
	}
```

Core Data 用起来还是挺直观的吧？
我们从 AppDelegate 里面拿到了老早就创建好了的 `managedObjectContext`，通过 `NSEntityDescription` 新建了一个准备插入到数据库中的 User；
然后把用户输入的数据填充到 User 里面，保存一下。

等等！ Core Data 是会自动保存的，所以最后一句注释掉也没有影响咯。

接下来是读取数据：

```objc
	- (IBAction)readAction:(id)sender
	{
	    AppDelegate *appDelegate = [[UIApplication sharedApplication] delegate];
	    NSManagedObjectContext *context = appDelegate.managedObjectContext;

	    NSFetchRequest *fetchRequest = [[NSFetchRequest alloc] initWithEntityName:@"User"];
	    NSError *error;
	    NSArray *results = [context executeFetchRequest:fetchRequest error:&error];
	    if ([results count]) {
	        self.consoleTextView.text = [results description];
	    } else {
	        self.consoleTextView.text = [error localizedDescription];
	    }
	}
```

认识一个新伙伴—— `NSFetchRequest`，数据库的读取基本都是靠这家伙了，你还可以对它加上各式各样的条件和约束，进行更精确的数据库查询。查询的结果会保存在一个 NSArray 里面，不管它，直接输出给我们看看。

这就够了！让我们的程序跑起来吧！
￼{% img /uploads/easy-core-data/let_run.png  250 250 %}

## 结果
随便录入点数据，按下“写进去”，嗯，和预期的一样，什么反应也没有。
然后按下"读出来"：
{% img center /uploads/easy-core-data/result.png 500 500 Result %}

嗯？输出的结果看起来有点奇怪？
那是因为我们直接把数组的 description 打印了出来。这个数组里面其实就是我们之前创建好的 User 对象，直接拿出来当Model用就可以了。
