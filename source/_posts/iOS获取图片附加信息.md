---
title: iOS获取图片附加信息
date: 2016-04-21 14:17:35
tags:
- iOS
- Exif
---

为了提供更好的体验，iOS上的App通常都会在条件允许的情况下尽可能多地获取用户信息（说好的隐私呢？！）。比如说，在用户选取了相册中的某张图片后，自动补充图片的拍摄日期、时间、地点等等，给用户一个非常智能的感觉。当然，前提是用户给予了我们这样做的权限。

<!-- more -->

## 那么问题来了
如何获得这些信息呢？

稍微查一下可以知道，有一种叫做[EXIF](http://en.wikipedia.org/wiki/Exchangeable_image_file_format)(Exchangeable image file format)的东西可以用来在图片中保存一定的数据（像素、ISO等）。这种标准通常应用在数码相机上，在用户照相同时，默默把当前的日期和时间给记录下来。而如今的智能手机在这个基础上还加入了地理位置等信息，包装成了一大个MetaData，信息数据应有尽有，可谓是给各位开发者提供了大大的方便啊。

可是，照片上的这些信息我们要怎样读取出来呢？

## 前提
首先你的这张照片一定要存有这些信息才能被读取出来吧？
![focus](/uploads/iOS获取图片附加信息/focus.png)


喂！这真的不是废话！

虽然现在的手机已经非常智能了，但是可以照相的应用却是五花八门，并不是每一位开发者都会考虑那么多，有相当一部分应用不管这部分数据的“死活”，它们想要的只是那一张图片而已。

所以，手机相册中的照片也不是百分百的带有我们想要的数据。有可能数据是完整的，但更多情况下，数据是残缺的，甚至完全没有这些数据。碰上这种情况，而你又迫切需要这些信息，那就只好求我们的用户大发慈悲地给我输入一下了。

特殊情况，特殊处理，这里不处理...
就先把我们的用户当成一个非常有耐心、对手机非常信任、照相时网络状况良好的上帝吧:)

## 照片哪里来
对于直接从照相机获取的照片来说，因为过程太短，所以并不能指望这张照片上面有多少我们想要的信息。如果条件允许的话，我们可以自己在后台采集（记录时间、定位等）。

从相册里面拿出来的照片就好多了。因为照片有一个保存的过程，耗时相对比较长，该采集的信息也采集得差不多了，没采集的我们也没有什么办法，所以就在这些已有的数据上下功夫就好了。

题外话：在用`UIImagePickerController`的时候，不小心遇到了一个问题，虽然和这篇文章的主题没有太大关系，不过我还是在[完整代码](https://github.com/davidleee/GetExifInfoDemo)里把解决方案写了下来。

## 动手吧！
用`UIImagePickerController`来获取相册照片的过程就不多说了，我们主要来看它的一个代理方法`-[imagePickerController:didFinishPickingMediaWithInfo:]`。

这个代理方法会给我们一个字典info，如果用户选择的是一张图片的话，这个info里面除了有这张图片以外，还有图片的URL等信息。
这张图片对我们接下来要做的事情作用不大，要关注的是这个URL。

在继续之前，我们还要加入这两个库：（下面的代码其实不需要ImageIO库，看[完整代码](https://github.com/davidleee/GetExifInfoDemo)就知道怎么回事了）

```objc
#import <AssetsLibrary/AssetsLibrary.h>
#import <ImageIO/ImageIO.h>
```

在UIImagePickerController.h里面，对这个info字典的key给出了定义：

```objc
// info dictionary keys
UIKIT_EXTERN NSString *const UIImagePickerControllerMediaType;      // an NSString (UTI, i.e. kUTTypeImage)
UIKIT_EXTERN NSString *const UIImagePickerControllerOriginalImage;  // a UIImage
UIKIT_EXTERN NSString *const UIImagePickerControllerEditedImage;    // a UIImage
UIKIT_EXTERN NSString *const UIImagePickerControllerCropRect;       // an NSValue (CGRect)
UIKIT_EXTERN NSString *const UIImagePickerControllerMediaURL;       // an NSURL
UIKIT_EXTERN NSString *const UIImagePickerControllerReferenceURL        NS_AVAILABLE_IOS(4_1);  // an NSURL that references an asset in the AssetsLibrary framework
UIKIT_EXTERN NSString *const UIImagePickerControllerMediaMetadata       NS_AVAILABLE_IOS(4_1);  // an NSDictionary containing metadata from a captured photo
```

而我们需要的只有`UIImagePickerControllerReferenceURL`对应的值。

因为UIKit都已经帮我们封装好了，所以真正要做的只剩下很少一部分。
起作用的代码就只有下面这几行：

```objc
ALAssetsLibrary *library = [[ALAssetsLibrary alloc] init]; // 1
[library assetForURL:[info objectForKey:UIImagePickerControllerReferenceURL] resultBlock:^(ALAsset *asset) { // 2

    NSDictionary *imageInfo = [asset defaultRepresentation].metadata;

} failureBlock:^(NSError *error) {

    ...

}];
```

这里做了什么呢？

1. 初始化了一个`ALAssetsLibrary`，可以把它想象成当前应用与系统相册之间的一个通道，通过它我们就可以对系统的相册进行一定程度上的数据存取；
2. 根据`UIImagePickerController`提供的URL，找到这张照片在相册中对应的对象（ALAsset），并返回给我们操作。

剩下的事情就一目了然了，metadata就是我们想要的数据字典。

## 结语
读取照片中的信息并没有我们想象中的复杂，网上有许多实现用到了一些更底层的方法，我在代码里也有写下来。更完整的叙述可以看Stackoverflow上的[这个回答](http://stackoverflow.com/a/9767129/4177374)。

<br/>
[完整代码传送门](https://github.com/davidleee/GetExifInfoDemo)
