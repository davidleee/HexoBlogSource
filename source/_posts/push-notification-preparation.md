---
title: 移动应用消息推送的准备工作
date: 2016-06-06 09:47:34
tags:
- iOS
- Android
---
每个移动应用开发者都会幻想自己的 App 被成千上万人每天不断地使用着，然而现实总是冰冷的：用户们总是会有意无意地把你的 App 关掉，并且有意无意地将它们晾在某个角落里。
然而聪明的程序员们总有各种办法引起用户们的注意，而消息推送（ Push Notifications ）无疑是这些办法中最直接了当的一种。

<!-- more -->

说了那么多，就是为了讲讲移动应用上常用的消息推送的**准备工作（不涉及代码）** ，内容将会涵盖 iOS 和 Android 两个平台。（你说 Win... 什么来着？）因为这方面做的工作实在算不上多，所以难免有错误和纰漏，欢迎大家提出来交流和讨论。

## 写在开头
一听到要实现消息推送机制，首先会想到的应该是对服务器的需求了。想要定制化程度更高的推送机制，当然是要有自己的服务器比较稳妥，但在某些追求快捷方便的情况下，服务器倒也不是必要。

对于追求开发速度的情况，已经有许多非常优秀的第三方服务商[^1]可以提供跨平台的解决方案，你压根就不需要考虑服务器的事情，甚至都不需要头疼集成的问题，因为这些产品的文档上都已经说明得很详细了。

所以，本文要提到的将是 iOS 和 Android 两方自家的推送服务。

## 两家的区别
对于生态环境封闭的 iOS 来说，想要推送消息就必须走苹果的服务器。你需要从自己的服务器向苹果的推送服务器发起推送请求，再由它替你将消息推向*用户的设备*。整个过程中，你自己的服务器是完全接触不到用户设备的。

而对于开放的 Android，虽然该 Google 有提供官方的服务，但你完全可以自己搭建一个推送服务器，将消息直接从你自己的服务器推送到*你自己的 App* 上。

注意上面两段文字中推送目标的差异：

* iOS 的推送服务器只是负责把消息推送到设备。每台 iOS 设备都有一个类似 IM 的程序跑在后台，统一接收来自服务器的推送，再分派到对应的 App 上去。
* Android 的推送服务通常是把消息直接推送到应用。每个需要接收推送的应用都会跑一个常驻后台的消息接收器，从这个方面看，Android 的设备会比iOS 设备稍微更耗电一些。

## iOS
想要在 iOS 系统中完成一次推送，就必须要借助 APNs[^2] 的力量。这一步是无论如何都绕不开的，所以市面上所有的第三方 iOS 推送服务都相当于是在官方的渠道上封装了一层，不过这比起自己架设服务器来说还是便捷了不少。

于是，各种配置还是必不可少的，区别只在最后生成的证书是你自己管还是让第三方托管。那么具体需要做些什么呢？

### Enabling Push Notification Service
这一步在 Xcode 中操作非常简单，就只需要两步：

1. 在 Xcode 应用设置的 General 标签栏下设置 App 的 Bundle Identifier
2. 在 Capabilities 标签栏下打开消息推送的开关
![push-off](/uploads/push-notification-preparation/push-off.jpg)￼

等它加载一会，成功后会变成这个样子：
![push-on](/uploads/push-notification-preparation/push-on.jpg)

这是为了告诉苹果：“我这个 ID 对应的 App 是要接收消息推送的”。

### SSL Certificate
完成了上面那一步之后，去到 Member Center 里找到刚刚填的那个 App ID，会看到下面 Push Notifications 的状态是两个黄黄的 "Configurable"，那说明现在已经万事俱备，只欠证书啦！

点编辑进去，果真如此：
![apple-push-notification](/uploads/push-notification-preparation/apple-push-notification.jpg)

如果是在开发环境下，点 Development SSL Certificate 里面的 "Create Certificate..." 会有教程教你怎么申请自己的 CSR 文件，其实就是下面这张图这里：
![request-csr](/uploads/push-notification-preparation/request-csr.jpg)

在弹出窗口的 "Request is" 后面选 "Saved to disk"，然后就可以在指定位置找到一个 ".certSigningRequest" 文件，把它上传到刚刚的网页上就可以了。

传好之后点击 "Generate" 就可以生成属于你的 SSL 证书。把它下载下来，双击安装，然后就可以在 Keychain Access 里面找到对应的 Push Services 证书，在开发的过程中，Xcode会自动找到这里的证书并打包到对应的 App 里面去。

### 小结
如果使用的是第三方的服务，在生成了 SSL 证书之后，通常还需要将它的私钥导出为一个 .p12 后缀的文件，上传到第三方的服务器上。这就相当于授权了第三方使用你的身份去发推送消息了。如果你自己搭建了服务器，那就还需要将 .p12 文件转为 .pem 文件才能使用。这些都是后话了。

P.S. 在应用要发布之前，还需要再走一遍上面的证书申请流程，不过这次要申请的是 Production SSL Certificate。

## Android
得益于其开放的生态环境，对比于 iOS 端，Android 这边几乎看不到什么证书相关的操作。

类似于 APNs，Android 也有一套推送服务，叫 FCM[^3]。它的强大之处在于：它还提供了对 iOS 设备的推送服务。如果想要选择一个有背景的、可媲美官方可靠性的（人家本来就是 Google 官方...）第三方推送服务商，FCM 绝对是一个不错的选择。

Google 已经把 FCM 很好地封装成 SDK 了，再加上已经不需要考虑证书的问题，你要做的就只是在网页上把 FCM 配置好，并把它的 SDK 下载下来，放到项目中。

> 可以参考 [Set Up a Firebase Cloud Messaging Client App on Android
](https://firebase.google.com/docs/cloud-messaging/android/client#set-up-firebase-and-the-fcm-sdk)和 [Add Firebase to your Android Project](https://firebase.google.com/docs/android/setup#prerequisites)。

## 写在结尾
没想到抛开了证书问题，Android 的准备工作就可以浓缩到一个小节里面了（虽然那两个网页也够看一会了），iOS可是洋洋洒洒分了三个部分呢。
上面也只是介绍了开发消息推送的前期准备工作，做完这些之后才是我们程序员老本行的开始。

## 参考资料
* [Push Notifications Tutorial: Getting Started](https://www.raywenderlich.com/123862/push-notifications-tutorial)
* [About Local and Remote Notifications](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/Introduction.html)
* [Firebase](https://firebase.google.com/)

[^1]: 国外： [Parse](https://www.parse.com/)、[FCM（原来的 GCM ）](https://firebase.google.com/)；国内： [极光推送（ JPush ）](https://www.jpush.cn/)、 [个推](http://www.getui.com/)
[^2]: 即 [Apple Push Notification service](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/ApplePushService.html)
[^3]: 即 [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging/)，前身是 Google Cloud Messaging ( GCM )
