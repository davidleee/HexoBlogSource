---
title: 【译】符号化iOS崩溃报告
date: 2018-03-08 15:09:39
tags:
- iOS
- Crash
---

> 原文链接：[Symbolicating Your iOS Crash Reports]( https://possiblemobile.com/2015/03/symbolicating-your-ios-crash-reports/)

你拿到一份 App 的崩溃报告，结果发现里面全是难以理解的内存地址。作为一名合格的开发，应该做些什么呢？简单来说，你需要把调试符号应用到崩溃日志上，让它变得更可读，这个过程叫做符号化（Symbolication）。

<!-- more -->

崩溃报告通常是一个 `.crash` 格式的文件。要拿到这个玩意儿有这几种方式：

1. 从 iTunes Connect 上获取
2. 通过 Xcode 从连接的设备上直接获取（Windows -> Devices）
3. 直接从连接设备上获取（ iOS 11 上的位置是：设置 -> 隐私 -> 分析）
4. 通过第三方的框架去获取

如果你已经在用第三方的崩溃收集服务的话，崩溃日志应该已经是符号化之后的可读格式了。



然后我们还需要拿到下面两个文件之一，来帮我们定位问题：

1. 崩溃应用的 `.app` 文件。这个包里面存有应用的二进制文件，可能还直接保存了调试符号（如果你手上的是 `.ipa` 文件，你可以直接把它当成一个压缩包去解压，里面会找到 `.app` 文件的）
2. 编译崩溃应用时生成的 `.dSYM` 文件。如果你编译应用的时候，没有指定把调试符号添加到 `.app` 文件里，那它们就会把这些符号单独放到这个 `.dSYM` 文件里



到底要拿哪一个呢？到 Xcode 项目里的 Build Setting 里面找一个叫 “Strip Debug Symbols During Copy”（`COPY_PHASE_STRIP`） 的字段，如果它是 `YES` 的话，调试符号就会从 `.app` 文件里剔除，并放到 `.dSYM` 文件里面。

> 默认情况下，出于代码混淆的考虑，打包 release 版本的时候调试信息会被放到 `.dSYM` 文件里。



## 等等，调试符号是什么鬼？

从程序员的角度看，调试符号就相当于我们给方法起的那个可读性高的名字。为了提高代码的混淆度，编译器会在编译过程中把我们的调试符号替换成它自己的。而且编译器每一次编译都可能会改变它自己的符号，即使我们的代码完全没有变化。



## 开始调试崩溃

如果你是从 Xcode 的 Organizer 里拿到的崩溃日志，那它里面与 iOS 框架（UIKit 等）相关的部分可能已经被符号化了，而且如果 Xcode 还记得这一次编译的话，整个崩溃日志的符号化过程也可以省掉了。

那在没那么好运的情况下呢？



### Symbolicatecrash 工具

最简单的方式就是使用苹果官方提供的 Symbolicatecrash 工具了，本质上它就是一个脚本，可以帮我们拿到调试符号并应用到指定的崩溃日志上。



在 Xcode 7.3 之后，这个工具在这个位置：

```
/Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash
```



在 Xcode 6 到 7.2 的版本下，它在这里：

```
/Applications/Xcode.app/Contents/SharedFrameworks/DTDeviceKitBase.framework/Versions/Current/Resources/symbolicatecrash
```



如果是更早的版本，可以到这里碰碰运气：

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/PrivateFrameworks/DTDeviceKitBase.framework/Versions/Current/Resources/symbolicatecrash
```



用这个工具之前，要先配置一个环境变量：

```
export DEVELOPER_DIR=”/Applications/Xcode.app/Contents/Developer”
```



然后把 `.crash`、`.app` 和 `.dSYM` 文件都放到同一个目录下，并执行：

```
symbolicatecrash <#YourAppName>.crash > Symbolicated.crash

// 如果需要明确指定应用二进制文件的话...
// symbolicatecrash <#YourAppName>.crash ./<#YourAppName>.app/<#YourAppName> > Symbolicated.crash
```



### 验证文件是否正确

如果符号化的过程碰到了问题，可以加上 `-v` 参数让 Symbolicatecrash 告诉我们更多信息。大多数情况下是因为 `.dSYM` 文件或者 `.app` 文件拿错了，可以用下面的指令来验证一下 UUID 是否正确：

```
// App 文件
dwarfdump -u <#YourAppName>.app/<#YourAppName>

// dSYM 文件
dwarfdump -u <#YourAppName>.app.dSYM/Contents/Resources/DWARF/<#YourAppName>
```

对比看看两次输出的 UUID 是不是一致的，然后看看崩溃日志里的 UUID 是不是输出的这一个。



### 定位 Symbolicatecrash 的问题

经过上面的步骤之后，如果还是没法输出有效的信息，那就要仔细看看符号化之后的日志了。官方符号化的工具会尝试寻找与崩溃 App 的 UUID 相匹配的文件和动态库，如果符号化失败的话，从输出的日志里确认它寻找的 App 名称和 UUID 是不是你要的。一般来说，会有这样的输出：

```
......fetching symbol file for Crasher[K–[undef] Searching []…– NO MATCH Searching in Spotlight for dsym with UUID of b00cdf0c29653095b1e86078b12d79e5 ... Number of symbols in /Users/You/Workspace/Crasher.app/Crasher: 1 + 106 = 107 Found executable /Users/You/Workspace/Crasher.app/Crasher — MATCH
```



如果 Spotlight 找不到 `.dSYM` 文件，输出是这样子的：

```
Did not find executable for dsym Warning: Can’t find any unstripped binary that matches version of /private/var/mobile/Containers/Bundle/Application/956755E3-6C66-4E87-A8BC-352FD4BE3711/Crasher.app/Crasher
```



如果 `.dSYM` 文件有问题：

```
Number of symbols in ./Crasher: + = 0 ./Crasher appears to be stripped, skipping.
```



非法输入的情况：

```
fatal error: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/lipo: can’t figure out the architecture type of: ./Crasher.app.dSYM.zip ./Crasher.app.dSYM.zip doesn’t contain armv7 slice
```



Xcode 6 上的 `symbolicatecrash` 会尝试修复 Xcode 5 上没办法解决的 Spotlight 问题，如果你的 `symbolicatecrash` 版本比较旧，可以尝试手动修复一下 Spotlight 的索引问题：

```
mdimport -g /Applications/Xcode.app/Contents/Library/Spotlight/uuid.mdimporter
```



### 命令行工具链

让我们再深入一步，使用命令行工具去一行行符号化堆栈信息。

先看一行崩溃日志里的堆栈信息：

```
... 13 Crasher 0x000aeef6 0xa8000 + 28406 ...
```



第一段十六进制数（`0x000aeef6`）是栈地址。第二段十六进制数（`0xa8000`）是程序加载地址。接下来的运算操作（`+ 28406`）是一个十进制的加法操作，这三个信息表示了一个等式：0x000aeef6 = 0xa8000 + 0x6EF6（== 28406）。

顺着崩溃日志往下看，会发现 "Binary Images" 这个字段的内容里包含了我们的程序加载地址，它代表了崩溃应用里加载了的一系列动态库占用的内存地址：

```
Binary Images: 0xa8000 – 0xaffff Crasher armv7 /var/mobile/Containers/Bundle/Application/956755E3-6C66-4E87-A8BC-352FD4BE3711/Crasher.app/Crasher
```



接下来，我们还需要看看崩溃应用的可执行文件的编译架构，可以用 `file` 或 `lipo -info` 命令查看：

```
file Crasher.app/Crasher
```



输出会是这个样子的：

```
Crasher.app/Crasher: Mach-O universal binary with 2 architectures Crasher.app/Crasher (for architecture armv7): Mach-O executable arm Crasher.app/Crasher (for architecture arm64): Mach-O 64-bit executable
```



现在我们知道所有需要的信息了。使用 `atos` 指令，可以把地址信息转化为调试符号：

```
atos -arch armv7 -o Crasher.app/Crasher -l 0xa8000 0x000aeef6
```

这里我们需要知道的参数是：应用的编译架构、应用位置、加载地址和栈地址。



输出会是这样的：

```
main (in Crasher) (main.m:14)
```



完成！如果你还有兴趣继续深入，可以了解一下 Mach-O 对象文件的格式，并尝试使用以下 Mach-O 相关的命令行工具，比如 `otool` 和 `lipo`。



## 延伸阅读

- [Technical Note TN2151: Understanding and Analyzing iOS Application Crash Reports](https://developer.apple.com/library/ios/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184)
- [Technical Q&A QA1765: How to Match a Crash Report to a Build](https://developer.apple.com/library/ios/qa/qa1765/_index.html#//apple_ref/doc/uid/DTS40012196)
- [Mach-O Programming Topics](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/MachOTopics/0-Introduction/introduction.html)
- [Objc.io on Mach-O Executables](http://www.objc.io/issue-6/mach-o-executables.html)
