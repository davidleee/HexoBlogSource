---
title: 在 macOS 中如何使用 XPC 实现跨进程通讯？
date: 2020-07-20 12:45:38
tags:
- macOS
- XPC
- launchd
- SMJobBless
---

最近需要在 Electron 项目上引入一个比较吃性能的大头功能，因为已经用 Objective-C 实现过一套稳定且性能也可接受的带 UI 方案了，所以计划看看能不能将这套现成的方案直接用到 Electron 里。但想要这么做就必须解决原生 UI 与 Electron 通讯的问题，再进一步，能不能让 Electron 以多进程的方式调起这个大头功能的 Demo 以节省掉绝大部分的重复工作呢？

> 本文只研究了原生 XPC 通讯的部分，关于集成到 Electron 里还有哪些坑会在下一篇文章里讲讲  

<!--more-->

## 什么是 XPC
> 选型的过程不是这次要讨论的重点，就当作我们经过一番挣扎然后选择了原生的 XPC 实现吧：）  

XPC 是苹果官方提供的一种进程间通讯的手段，是一种苹果特有的 IPC 技术。

在 NSHipster 的[一篇文章](https://nshipster.com/inter-process-communication/)里，作者说 XPC 是官方 SDK 内跨进程通讯的最优解决方案（2014）。从 2011 年被提出的时候，XPC 就持续在“体制内”发光发热，比如 macOS 的沙盒、iOS 的 Remote View Controller 和两个平台上都有的应用扩展（App Extensions）里都用到了 XPC 的技术。

对于开发者来说，使用 XPC 技术我们就能做到像这样的事情：
1. 模块 A 负责 UI 展示，它**不需要申请任何系统权限**，用到网络图片时就向模块 B 获取
2. 模块 B 拥有**网络权限**，能从网络或缓存中获取图片，但操作文件系统的工作由模块 C 负责
3. 模块 C 拥有**文件读写权限**，负责将数据写成文件或读取文件数据
4. 这三个模块都在同一个应用中，它们所需要的权限相互独立，功能单一，而且即使崩溃了也不会相互影响，只需要重启相应的模块就又可以恢复正常使用

看完是不是已经迫不及待了呢？别着急，在使用这个强大工具前，我们还需要了解两个关键技术。

### 题外话1 - launchd
`launchd` 负责管理 macOS 上的守护进程，在构建 XPC 方案的过程中，我们会用它来配置一个我们自己的守护进程。

这个守护进程会一直潜伏在系统里（只占用非常少的资源），当我们的应用需要它的时候就可以被随时唤醒。

更多 `launchd` 的信息和用法可以在它的[man 页面](x-man-page://5/launchd.plist)找到。

### 题外话2 - SMJobBless
字面意思是“给任务加上祝福”，任何应用都不能跟一个没有被系统祝福的任务愉快地玩耍。  

这是一组协助开发者安全地安装守护进程的 API，长这个样子：
```objectivec
Boolean SMJobBless(CFStringRef domain, CFStringRef executableLabel, AuthorizationRef auth, CFErrorRef *outError);
```

苹果似乎也认为这组 API 的用法只可意会不可言传，所以在[SMJobBless 的方法说明](https://developer.apple.com/documentation/servicemanagement/1431078-smjobbless?language=objc)里写了很多，还给出了一个很完整的[示例工程](http://developer.apple.com/library/mac/#samplecode/SMJobBless/)并通过一个 [Python 脚本](https://developer.apple.com/library/archive/samplecode/SMJobBless/Listings/SMJobBlessUtil_py.html#//apple_ref/doc/uid/DTS40010071-SMJobBlessUtil_py-DontLinkElementID_8)把安装守护进程的前置条件给配置好了。

> 脚本这个动作，虽然让整个流程变得更加完善，但却将原本只要几句命令就能解决的事情复杂化了，少了一些苹果味。  

## 架起通讯的桥梁
写了这么多，其实都还在 **Prerequisites** 阶段打转转。接下来才要正式开始跨应用通讯的实现！

不过在此之前，我们还是先把上文题外话里提到的前置条件准备好，让后面的过程更顺畅一些。

### 前置准备
通过 `launchd` 安装守护进程是个需要很高安全性的动作，所以应用签名是必不可少的。而对于一个跨应用通讯的系统来说，安全性主要涉及到两个部分：
* 通讯发起方
* XPC 应用

> 在这篇文章中，通讯的接收方不负责 XPC 应用的安装，所以它只要管好自己的签名就够了  

这里我们就要用上前面提到的 Python 脚本里的一句关键命令：
```shell
codesign -d -r - /path/to/file.app
```

> 虽然官方 Demo 里的这个脚本还做了许多其他的检验来确保信息的完整和正确，但对于我们这样成熟的（嘿嘿）开发者来说，当然要直接薅最珍贵的羊毛啦。  

把这个命令的路径参数改为我们已经签好名的应用，会得到像这样子的输出：
```
Executable=/path/to/file.app
designated => anchor apple generic and identifier "com.example.apple-samplecode.EBAS.App" and (certificate leaf[field.1.2.840.113635.100.6.1.9] /* exists */ or certificate 1[field.1.2.840.113635.100.6.2.6] /* exists */ and certificate leaf[field.1.2.840.113635.100.6.1.13] /* exists */ and certificate leaf[subject.OU] = SKMME9E2Y8)
```
其中，`designated =>` 后面的部分（例子里是从 “anchor” 开始，我们自己签名的话开头可能是“identifier”，这个顺序并不要紧）就是我们需要的“签名需求”（Code Signing Requirement）。

把签名需求放到我们自己的 XPC 应用的 Info.plist 里，如此一来这个 XPC 应用就只能被拥有这个签名的应用启动了：
```xml
<key>SMAuthorizedClients</key>
<array>
    <string>anchor apple generic and identifier "com.example.apple-samplecode.EBAS.App" and (certificate leaf[field.1.2.840.113635.100.6.1.9] /* exists */ or certificate 1[field.1.2.840.113635.100.6.2.6] /* exists */ and certificate leaf[field.1.2.840.113635.100.6.1.13] /* exists */ and certificate leaf[subject.OU] = SKMME9E2Y8)</string>
</array>
```
这里的 value 是数组格式的，意味着如果想允许多个 App 启动这个 XPC 应用的话，就需要把这些 App 的签名需求都写上。

同理，还要取到 XPC 应用的签名需求并配到我们客户端的 Info.plist 里：
```xml
<key>SMPrivilegedExecutables</key>
<dict>
    <key>com.example.apple-samplecode.EBAS.HelperTool</key>
    <string>anchor apple generic and identifier "com.example.apple-samplecode.EBAS.HelperTool" and (certificate leaf[field.1.2.840.113635.100.6.1.9] /* exists */ or certificate 1[field.1.2.840.113635.100.6.2.6] /* exists */ and certificate leaf[field.1.2.840.113635.100.6.1.13] /* exists */ and certificate leaf[subject.OU] = SKMME9E2Y8)</string>
</dict>
```
注意这个 “dict” 里的 “key” 要填的是我们的 XPC 应用的 label，不过因为 label 通常会定成跟 Bundle Identifier 一致，所以写上它的 Bundle Identifier 也就可以了。

> 上一段啰嗦了一下是因为 label 其实可以跟 Bundle Identifier 不同的，但这会给开发的过程带来许多麻烦，所以建议还是统一。这个 label 具体是什么鬼会在下一个小节里讲到。  

### 创建 & 安装 XPC 应用
首先来添加一个 Target 并选择 XPC Service，让 Xcode 帮我们生成一些默认代码：
![](/uploads/ipc-for-macOS/create_target.png)

然后为我们的 XPC 应用再创建一个 plist，这个文件会在 XPC 应用被安装的时候自动拷贝到 */Library/LaunchDaemons* 目录下，这是统一存放守护进程配置文件的地方。

为了与默认的 Info.plist 区分开来，在文件的名字里加上个 “Launchd”，文件内容是这样的：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>com.example.apple-samplecode.EBAS.HelperTool</string>
	<key>MachServices</key>
	<dict>
		<key>com.example.apple-samplecode.EBAS.HelperTool</key>
		<true/>
	</dict>
</dict>
</plist>
```
看，前面埋的坑—— **Label** 出现了！这是系统用来唯一标识守护进程的值，下面 MachServices 中的 key 是我们 XPC 应用的 Bundle Identifier。在建立连接的时候，系统就会根据这张配置表去寻找正确的 XPC 应用。

完成后我们的目录结构是这样的：（例子来自官方的 [EvenBetterAuthorizationSample](https://developer.apple.com/library/archive/samplecode/EvenBetterAuthorizationSample/Introduction/Intro.html#//apple_ref/doc/uid/DTS40013768-Intro-DontLinkElementID_2)）
![](/uploads/ipc-for-macOS/9B514455-AC1D-454F-8681-34076024982E.png)

> 如图，官方例子中还给 plist 加上了项目名前缀，但名字不重要，重要的是别忘了把签名需求写对。  

最后，因为我们要用到的产物是 .xpc 包里的二进制文件，所以必须把这两个 plist 也打进二进制文件里去，这就要在 Build Settings 的 Other Linker Flags 里配置一下：
![](/uploads/ipc-for-macOS/29AEBF92-9331-41E1-8EAB-04718FEFDB6C.png)

配置内容如下，把最后的路径改成自己的 plist 就可以了（这也是为什么前面说文件名不重要）：
```
-sectcreate __TEXT __info_plist HelperTool/HelperTool-Info.plist
-sectcreate __TEXT __launchd_plist HelperTool/HelperTool-Launchd.plist
```

完成了这些配置后打出来的包会是一个完整的 .xpc 文件了，但我们需要的只是它里面的二进制文件。在用上它之前，让我们把安装 XPC 应用的代码写好，这里的代码是在官方例子的基础上改的，个人感觉比例子里的更易懂一些：
```objectivec
// 1
AuthorizationItem authItem = { kSMRightBlessPrivilegedHelper, 0, NULL, 0 };
AuthorizationRights authRights = { 1, &authItem };
AuthorizationFlags flags = kAuthorizationFlagDefaults | kAuthorizationFlagInteractionAllowed | kAuthorizationFlagPreAuthorize | kAuthorizationFlagExtendRights;

AuthorizationRef authRef = NULL;
// 2
OSStatus status = AuthorizationCreate(&authRights, kAuthorizationEmptyEnvironment, flags, &authRef);
if (status != errAuthorizationSuccess) {
    NSLog(@"Failed to create AuthorizationRef, return code %i", status);
    return NO;
} else {
    CFErrorRef error;
    // 3
    BOOL success = SMJobBless(kSMDomainSystemLaunchd, (__bridge CFStringRef)@"com.example.apple-samplecode.EBAS.HelperTool", authRef, &error);
    if (success) {
        NSLog(@"job bless success");
        return YES;
    } else {
        NSLog(@"job bless error: %@", (__bridge NSError *)error);
        CFRelease(error);
        return NO;
    }
}
```
1. 构造申请权限所需要的参数，官方例子中没有这一步
2. 申请权限，如果这一步失败了，那我们的应用就不能做任何需要用户授权的操作了；官方例子中因为少了构造参数的步骤，所以这里会变成 `AuthorizationCreate(NULL, NULL, 0, &authRef)`
3. 使用题外话里讲到的 API 来安装我们的 XPC 应用，其中，`kSMDomainSystemLaunchd` 表示我们要使用 launchd 服务（这也是目前仅有的可选项），第二个参数是我们之前设置的 XPC 应用的 label

当执行到上面的逻辑时，我们从两个角度来看看会发生什么：
* 用户角度：界面上弹出一个授权框，提示用户输入解锁密码
* 系统角度：系统会进入申请授权的应用内部寻找这个待安装的 XPC 应用二进制包，如果找到了会将它 *存起来* 以便下一次可以直接唤起，并把其中的 launchd 配置拷贝的统一的位置

> 通过 SMJobBless 安装的 XPC 应用会存在 /Library/PrivilegedHelperTools 下面，一旦授权完成过一次，后续只要配置文件和这里的二进制文件还对得上就不会再弹授权框了  

为了让系统方便地找到 XPC 应用，要把它的**二进制文件**放到应用的 /Contents/Library/LaunchServices 路径下，我们可以在客户端的 Build Phases 里面加一个步骤来做这件事：
![](/uploads/ipc-for-macOS/03FE8C66-DDBB-4A23-9D0B-5C0FE1535F30.png)

> 千万记得这里要放的是 .xpc 包里的二进制文件，在 xxx.xpc/Contents/MacOS 目录下  

OK，万事具备，接下来我们真的要写代码了。

### 与 XPC 应用通讯
首先我们来实现 XPC 应用的连接监听逻辑，在创建 Target 之后的 .m 文件里已经有连接处理的模版和丰富的注释了：
```objectivec
self.listener = [[NSXPCListener alloc] initWithMachServiceName:@"这里改成上面设置的 Label"];

...

- (BOOL)listener:(NSXPCListener *)listener shouldAcceptNewConnection:(NSXPCConnection *)newConnection {
    // This method is where the NSXPCListener configures, accepts, and resumes a new incoming NSXPCConnection.
    assert(listener == self.listener);
    assert(newConnection != nil);
    
    // Configure the connection.
    // First, set the interface that the exported object implements.
    newConnection.exportedInterface = [NSXPCInterface interfaceWithProtocol:@protocol(HelperToolProtocol)];
    
    // Next, set the object that the connection exports. All messages sent on the connection to this service will be sent to the exported object to handle. The connection retains the exported object.
    newConnection.exportedObject = self;
    
    // Resuming the connection allows the system to deliver more incoming messages.
    [newConnection resume];
    
    // Returning YES from this method tells the system that you have accepted this connection. If you want to reject the connection for some reason, call -invalidate on the connection and return NO.
    return YES;
}
```

需要特别说明的一点是，XPC 连接建立起来之后，连接发起方就能获取到上面的逻辑里的 `exportedObject`，而再上一行的 `exportedInterface` 是声明这个对象在这次 XPC 通讯中会遵循的协议。

换句话说，连接的发起方会把连接上的 XPC 应用直接当作一个对象来操作。这个对象的消息传递是异步的，所以在调用的时候要小心避免卡主线程。

> 因为协议需要连接双方自行约定统一，所以上面 `HelperToolProtocol` 的定义建议放到一个公共的文件里，让我们的应用项目和 XPC 应用项目都能访问到

XPC 应用这边先说这么多，大多数情况下模版代码就够了，只需要自己定义一下 `exportedInterface` 就能实现例如心跳机制这样的功能。

接下来实现客户端发起连接的逻辑，我们直接参考官方例子里的代码：
```objectivec
- (void)connectToHelperTool
    // Ensures that we're connected to our helper tool.
{
    assert([NSThread isMainThread]);
    if (self.helperToolConnection == nil) {
        // 1
        self.helperToolConnection = [[NSXPCConnection alloc] initWithMachServiceName:kHelperToolMachServiceName options:NSXPCConnectionPrivileged];
        // 2
        self.helperToolConnection.remoteObjectInterface = [NSXPCInterface interfaceWithProtocol:@protocol(HelperToolProtocol)];
        // 3
        #pragma clang diagnostic push
        #pragma clang diagnostic ignored "-Warc-retain-cycles"
        // We can ignore the retain cycle warning because a) the retain taken by the
        // invalidation handler block is released by us setting it to nil when the block 
        // actually runs, and b) the retain taken by the block passed to -addOperationWithBlock: 
        // will be released when that operation completes and the operation itself is deallocated 
        // (notably self does not have a reference to the NSBlockOperation).
        self.helperToolConnection.invalidationHandler = ^{
            // If the connection gets invalidated then, on the main thread, nil out our
            // reference to it.  This ensures that we attempt to rebuild it the next time around.
            self.helperToolConnection.invalidationHandler = nil;
            [[NSOperationQueue mainQueue] addOperationWithBlock:^{
                self.helperToolConnection = nil;
                [self logText:@"connection invalidated\n"];
            }];
        };
        #pragma clang diagnostic pop
        // 4
        [self.helperToolConnection resume];
    }
}
```
1. 通过 label 找到特定 XPC 应用并建立连接，建议把这个连接实例保存起来，避免重复创建带来别的问题
2. 这一步参数里的协议就是我们在 XPC 应用中声明的协议，两边的协议要对得上才能拿到 XPC 应用中暴露出来的正确对象
3. 大段注释是在解释为什么这里不需要担心循环引用的问题；要注意的是如果我们把连接实例存了起来，最好是像这样在 `invalidationHandler` 里置空，在其他地方通过 `[connection invalidate]`来实现断连
4. 手动调用 `resume` 来建立连接，调用后 XPC 应用那边才会收到 `-[listener:shouldAcceptNewConnection:]` 回调

Done！如果前面的一系列配置都正确的话，这个方法就能搭起客户端与 XPC 应用之间连接桥梁了！

### 与其他进程通讯
除了与 XPC 应用建立连接之外，NSXPCConnection 还提供了另一组 API 用于直接跟其他客户端建立连接：
```objectivec
- (instancetype)initWithListenerEndpoint:(NSXPCListenerEndpoint *)endpoint;
```

一次完整的连接建立流程是这样的：
1. 客户端 A 与 XPC 应用建立连接
2. 客户端 A 生成一个 NSXPCListenerEndpoint 并存放到 XPC 应用里
3. 客户端 B 与 XPC 应用建立连接并取到这个 NSXPCListenerEndpoint
4. 客户端 B 通过 NSXPCListenerEndpoint 与客户端 A 建立连接

在上个小节中我们完成了第一步，而第四步跟第一步其实挺像的，所以第二三步就是我们现在要处理的了。

之前我们声明了一个空的 `HelperToolProtocol`，现在就给它加一些内容，向外界提供对象读写的能力：
```objectivec
@protocol HelperToolProtocol
- (void)setEndpoint:(NSXPCListenerEndpoint *)endpoint withReply:(void (^)(BOOL))reply;
- (void)getEndpointWithReply:(void (^)(NSXPCListenerEndpoint *))reply;
@end
```
> 因为 `exportedObject` 的消息传递是异步的，所以在需要返回值的时候要改用回调的方式实现。

然后在 XPC 应用里声明一个成员变量并实现上面的两个方法就完成了：
```objectivec
@property (strong, nonatomic) NSXPCListenerEndpoint *endpoint;

...

- (void)setEndpoint:(NSXPCListenerEndpoint *)endpoint withReply:(void (^)(BOOL))reply {
    self.endpoint = endpoint;
    reply(YES);
}

- (void)getEndpointWithReply:(void (^)(NSXPCListenerEndpoint *))reply {
    reply(self.endpoint);
}
```

接下来回到客户端的代码里（现在还没实现客户端 B，所以这里讲的都是客户端 A）：
```objectivec
// 1
[self connectToHelperTool];

// 2
self.listener = [NSXPCListener anonymousListener];
self.listener.delegate = self;

// 3
id<HelperToolProtocol> service = [self.connection remoteObjectProxyWithErrorHandler:^(NSError * _Nonnull error) {
    NSLog(@"get remote object proxy error: %@", error);
}];

// 4
[service setEndpoint:self.listener.endpoint withReply:^(BOOL result) {
    NSLog(@"set endpoint result: %@", result ? @"success" : @"failed");
}];
```
1. 先 XPC 应用建立连接
2. 准备一个监听器来处理其他客户端的连接
3. 获取到 XPC 应用的 `exportedObject`，因为方法返回的是实现了这个协议的对象，所以协议的匹配很关键
4. 调用协议中的方法把匿名监听器的端点设置过去，因为我们在 XPC 应用里写死了返回 `YES`，所以这里肯定会成功，实际使用的过程中可能要加上安全性的处理

监听器有了，就差监听到连接后的回调了：
```objectivec
// 1
- (BOOL)listener:(NSXPCListener *)listener shouldAcceptNewConnection:(NSXPCConnection *)newConnection {
    assert(listener == self.listener);
    assert(newConnection != nil);
    
    // 2
    __weak typeof(self) weakSelf = self;
    self.clientConnection.invalidationHandler = ^{
        weakSelf.clientConnection.invalidationHandler = nil;
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            weakSelf.clientConnection = nil;
        }];
    };
    
    // 3
    self.clientConnection.remoteObjectInterface = [NSXPCInterface interfaceWithProtocol:@protocol(ClientBProtocol)];
    self.clientConnection.exportedInterface = [NSXPCInterface interfaceWithProtocol:@protocol(ClientAProtocol)];
    
    self.clientConnection.exportedObject = self;
    [self.clientConnection resume];
    return YES;
}
```
1. 这个熟悉的回调方法其实跟 XPC 应用里的那个一样，客户端 A 已经具备了一个 XPC 应用的基本功能了
2. 官方例子的另一种写法，逻辑上是一样的（可能这种还亲切一些呢😬）
3. 除了 `exportedInterface` 之外，还要设置 `remoteObjectInterface`，因为这是一条双向通讯的连接，所以要让其他客户端知道我们期望它们能遵循什么协议

好的，流程走完一半了，第三四步需要在客户端 B 里面实现：
```objectivec
// 1
[self connectToHelperTool];
__weak typeof(self) weakSelf = self;
[[self.connection remoteObjectProxyWithErrorHandler:nil] getEndpointWithReply:^(NSXPCListenerEndpoint *endpoint) {
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        if (endpoint) {
            [weakSelf connectWithEndpoint:endpoint];
        }
    }];
}]

- (void)connectWithEndpoint:(NSXPCListenerEndpoint *)endpoint {
    // 2
    self.serverConnection = [[NSXPCConnection alloc] initWithListenerEndpoint:endpoint];
    self.serverConnection.remoteObjectInterface = [NSXPCInterface interfaceWithProtocol:@protocol(ServerCommunicationProtocol)];
    
    // 3
    self.serverConnection.exportedInterface = [NSXPCInterface interfaceWithProtocol:@protocol(ClientCommunicationProtocol)];
    self.serverConnection.exportedObject = self;
    
    __weak typeof(self) weakSelf = self;
    self.serverConnection.invalidationHandler = ^{
        weakSelf.serverConnection.invalidationHandler = nil;
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            weakSelf.serverConnection = nil;
        }];
    };

    [self.serverConnection resume];

    // 4
    // self.serverConnectionEndpoint = endpoint;
}
```
1. 客户端 B 从 XPC 应用中拿到客户端 A 设置的端点，这跟设置端点的代码差不多
2. 通过端点来构造 `NSXPCConnection`，这样在调用 `resume` 之后对方会收到 `-[listener:shouldAcceptNewConnection:]` 的回调
3. 与上文的回调处理类似，因为我们要做的是双向通讯，所以客户端 B 在发起连接时也要把 `exportedInterface` 和 `exportedObject` 设置好，之后的代码就跟其他地方看到的差不多了
4. 有必要的话可以把这个端点存起来用于一些判断或重连的逻辑

## 常见错误
> 这一段总结了我在实现过程中踩的坑，也许我们的情况不太一样，但希望能给大家一个排查的思路  

涉及到多端通讯的逻辑调试起来比较绕，错误通常会发生在以下两个部分：
1. 客户端部分：确认代码逻辑没漏的话，可以把各种 `handler` 的结果都打印一下，一般都会带有比较明确的错误域和错误码
2. 连接部分：这类错误信息不会出现在客户端日志里，也分两类
	1. 没有任何反应但就是连不上：可以用自带的控制台工具去捞日志
	2. 调用 API 导致崩溃：也是控制台捞，macOS 的系统崩溃上报弹窗里可能会有更多信息

下面是我碰过的一些错误和处理方式：
### CFErrorDomainLaunchd Code=2
安装 XPC 应用时在客户端内找不到 XPC 应用的二进制文件，检查一下二进制包是不是放到了正确的路径下，格式是否正确（记得要取 .xpc 后缀的文件里的二进制文件）。

### CFErrorDomainLaunchd Code=4 or 8
签名匹配不上。大概率是 Info.plist 里配置的签名需求不正确，回头看看 *前置准备* 那个小节，检查内容是否跟 `codesign -d -r - /path/to/app`  和 `codesign -d -r - /path/to/xpc`  的一致。

### Error Domain=NSCocoaErrorDomain Code=4097
> 出自 FoundationErrors.h - NSXPCConnectionInterrupted

连接被打断（interrupted），约等于 connection.interruptionHandler 被触发了。
如果发生在连接建立的过程中，那意味着它发现连接已经被占用了，多见于调试过程中重启了其中一端，但是另一端没有把连接释放掉。

在正常运行的过程中发生的话，可能是系统 XPC 服务发现我们的连接长时间没有使用而挂起了它，这种情况一般不需要处理，系统会在我们下次使用这条连接的时候自动帮我们处理好。

### Error Domain=NSCocoaErrorDomain Code=4099
> 出自 FoundationErrors.h - NSXPCConnectionInvalid

同样分两种情况，一连接就出事的话，可能是 XPC 应用没有安装成功，排查方式是看 plist 和二进制文件有没有出现在它们该出现的路径里。
另一种情况，可能是客户端因为沙盒的原因而无法建立这条连接，控制台日志里会看到类似 *deny mach-lookup* 的信息，可以选择把 App Sandbox 关上（会没法上 Mac App Store 但不影响其他渠道的分发），真要打开沙盒的话有两条可以尝试的路径：

1. 想办法搞定 entitlements 的配置，可以参考[这篇文章](https://christiantietze.de/posts/2015/01/xpc-helper-sandboxing-mac/)
2. 应用内置另一个不在沙盒内的 XPC 应用，通过它去跟安装到系统里的 XPC 应用建立连接，具体的方式在官方的 [EvenBetterAuthorizationSample](https://developer.apple.com/library/archive/samplecode/EvenBetterAuthorizationSample/Introduction/Intro.html#//apple_ref/doc/uid/DTS40013768-Intro-DontLinkElementID_2)中有实现

## 总结
XPC 是 macOS 跨应用通讯中不得不面对的一种方案，可能出于各种原因最终的选择并不是它，但它确实是目前最简单可靠的实现了。

尽管我在网上已经查了非常多的资料，也还是在动手的过程中频频踩坑。写下这篇长文也是希望能把这条路尽可能填平，只是这个文章长度就有些一发不可收拾了😅。

## 参考资料
* [通过ServiceManagement注册LaunchdDaemon | 老谭笔记](http://www.tanhao.me/pieces/1623.html/)
* [Inter-Process Communication - NSHipster](https://nshipster.com/inter-process-communication/)
* [Read Me About EvenBetterAuthorizationSample.txt](https://developer.apple.com/library/archive/samplecode/EvenBetterAuthorizationSample/Listings/Read_Me_About_EvenBetterAuthorizationSample_txt.html#//apple_ref/doc/uid/DTS40013768-Read_Me_About_EvenBetterAuthorizationSample_txt-DontLinkElementID_17)
* [XPC · objc.io](http://www.objc.io/issue-14/xpc.html)
* [Creating a Launch Agent that provides an XPC service on macOS using Swift](https://rderik.com/blog/creating-a-launch-agent-that-provides-an-xpc-service-on-macos/)
* [Creating Launch Daemons and Agents](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html#//apple_ref/doc/uid/10000172i-SW7-BCIEDDBJ)