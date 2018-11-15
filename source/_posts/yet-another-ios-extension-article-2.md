---
title: 又一篇 iOS Extension 入门（2/3）— 与容器沟通
date: 2018-11-14 18:01:12
tags:
- ios
- App Groups
- Extension
- UserDefaults
- FileManager
- CoreData
---

在{% post_link yet-another-ios-extension-article 上一篇文章 %}里，我们了解到了 iOS Extension 的基础和怎么制作一个简单的分享扩展，然而，限于篇幅原因，这个分享操作止于用户点下 “Post” 的那一刻了。
接下来，就让我们一起看看怎么把用户分享的数据给到载体应用，让这次分享溜得飞起。

<!--more-->

## App Groups
我们都知道 iOS 的应用是跑在一个属于自己的沙盒里面的，为了实现应用间的数据共享，苹果提供了一个叫 App Groups 的概念。只有当应用属于同一个 App Groups 的时候，才能访问到共享的数据存储区域。

我们可以在载体应用的项目配置 Capabilities -> App Groups 里创建一个应用分组：
![](/uploads/yet-another-ios-extension-article-2/77534A33-46E6-4381-B6EC-4AA09E726A6A.png)

然后在应用扩展的项目配置 Capabilities -> App Groups 里会出现我们刚刚新建的应用分组，直接钩上就可以了。

这样我们就等于分配了一个共享空间给这哥俩，为我们接下来的数据共享做好准备了。

## 共享空间
做完上面的准备之后，我们就可以通过三种方式去访问共享空间，它们分别是 `UserDefaults`、`FileManager` 和 `CoreData`。

### UserDefaults
`UserDefaults` 有一个带参数的初始化方法，通过这个方法我们可以访问到一个共享的用户配置空间。在上一篇文章里，我们成功把 Safari 分享出来的一个 URL 打印了出来，现在我们把它放到共享空间去，让载体应用也可以获取到这个链接：
```swift
...

attachment.loadItem(forTypeIdentifier: kUTTypeURL as String, options: nil) { (item, error) in
                        if error == nil {
                            let userDefaults = UserDefaults(suiteName: "group.com.davidleee.SharePlayground")
                            userDefaults?.set(item, forKey: "share-url")
							   print("url from userdefault: \(userDefaults?.value(forKey: "share-url"))")
                        }
                    }

...
```

通过传入之前设置好的应用分组 ID，我们告诉 `UserDefault` 接下来要访问一个特定的共享空间，接着就像平常那样使用它即可。

> 上面的打印输出的是一堆 data，以为 `URL` 在保存到 `UserDefaults` 的时候会被序列化，想看到原来的 `URL` 对象的话还要再反序列化一下才行。  

### FileManager
与 `UserDefaults` 类似，`FileManager` 也有一个特殊的获取方法，我们看看把刚刚的 URL 写到一个文本文件里应该是什么样子：
```swift
...

let groupPath = FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "group.com.davidleee.SharePlayground")
                            if let url = item as? URL, let filePath = groupPath?.appendingPathComponent("url.txt")  {
                                try? url.absoluteString.write(to: filePath, atomically: true, encoding: .utf8)
                            }
                            
                            if let filePath = groupPath?.appendingPathComponent("url.txt") {
                                try? print("content of file: \(String(contentsOf: filePath, encoding: .utf8))")
                            }

...
```

### CoreData
好吧，CoreData 的共享空间其实跟 `FileManager` 是同一个，只是从写文件变成写数据库，再把数据库的文件放到共享空间而已。这个就不贴代码了，CoreData 里的类名是真的长…

## 总结一下
感觉这篇文章跟应用扩展都没什么关系了…毕竟 App Goups 是 iOS 平台上一个比较通用的数据共享技术。

App Groups 的引入让 iOS 应用间数据共享成为可能，这不仅可以用在应用扩展和载体应用之间，还可以用在自家的多个独立应用之间，真可谓是沙盒墙上透过来的一道亮光。

## 参考文章
* [iOS扩展开发攻略(一) - Share Extension - vimfung的开源部落 - 开源中国](https://my.oschina.net/vimfung/blog/707448)