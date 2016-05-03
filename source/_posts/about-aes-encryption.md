---
title: AES是个什么鬼？
date: 2016-04-26 10:38:18
tags:
- 加解密
- 安全
- AES
- DES
---

前段时间参加了部门的几次分享会，主题围绕着数字签名、数字证书和https相关的知识。这些方面的内容都不可避免的要涉及到数据加解密，于是趁热打铁，准备进行一次加解密相关基础的学习和分享。~~顺便扩充一下博客数量。~~
这一篇是针对AES的学习笔记，主要的知识来源是维基百科和各种网络资源。
因为时间有限，所以研究的不是非常深入，如果有不准确和错误的地方，希望能指出来一起讨论学习。

<!-- more -->

## 前世今生
AES 的出现就是为了取代原来的数据加密标准（DES），作为爷爷级的加密算法，DES在风光过后也是到了该退休的年纪了。

### 关于DES
在继续了解AES之前，不妨先看看被它取代的DES是什么。
它的全称为 **Data Encryption Standard** ，是一种对称密钥加密块算法，大致的加密流程长这个样子：
![des-encrypt](/uploads/AES是个什么鬼？/des-encrypt.png)￼
在进入到加密流程之前，64位的块被拆分为两个32位的子块，并作为 IP 的两个输入。中间的 F 是 Feistel function，算法中的密钥就是在这个函数中被用到的。

> 块加密：Block cipher， 也叫作分组加密，是将明文分成多个等长模块（block），使用确定的算法和对称密钥对每组分别加密解密的方式。

DES 在1976年曾经风光一时，被美国联邦政府的国家标准局定为 **联邦资料处理标准（FIPS）**。然而因为它只是用了56位的密钥，所以在当下已经不是一种安全的加密方法。在1999年1月，已经有组织在22小时15分钟内公开破解了一个DES密钥。

> 后来出现了一种改进的 DES，叫 TDES 或 3DES。它本质上就是把密钥个数增加到了3个，并没有算法上的改进。（感觉很儿戏的样子）

在2001年，DES已经不再是 **国际标准科技协会**（NIST，前 FIPS）的一个标准，而且也开始慢慢被AES所取代。

### 言归正传

AES，全称为 **Advanced Encryption Standard**，原名叫做 **Rijndael 加密法**。（还是新名字好念）
> 至于一开始为什么有个这么拗口的名字，因为两位作者的名字是 Joan Daemen 和 Vincent Rijmen，发现为什么了吗？这是不是密码学家约定俗成的某种命名方式呢？

2001年11月26日，美国的 NIST 公布了 AES 这一标准，并开始了长达5年的标准化进程，直到 Rijndael 被选为最适合的方法。
在2002年5月26日，AES 成为了一项联邦政府标准。它还是联邦安全局（NSA）批准的唯一一种用来加密顶级机密信息的公开加密方法。

> 也就是说，如果你想要黑 FBI，也许可以试试看 AES 解密 :)

严格来说，AES 和 Rijndael 并不完全一样。AES 使用的是固定128位大小的块，密钥的大小只能是128位、192位或256位；而 Rijndael 使用的块大小和密钥长度可以是在128位和256位之间能被32整除的任意值，相对来说灵活性高了很多。

## 主要过程
AES加密算法的组成可以分成4个主要部分：
1. AddRoundKey
2. SubBytes
3. ShiftRows
4. MixColumns

简单来说，就是将上面的几个部分组合起来形成三种不同的序列，然后把这些过程序列重复执行若干个回合，具体的循环次数由密钥的长度决定：
* 128位密钥：循环10次
* 192位密钥：循环12次
* 256位密钥：循环14次

这三种序列是：

* 首次循环：
	1. AddRoundKey


* 一般循环：
	1. SubBytes
	2. ShiftRows
	3. MixColumns
	4. AddRoundKey


* 末尾循环：
	1. SubBytes
	2. ShiftRows
	3. AddRoundKey


那么这几个部分到底是干了些什么呢？

### AddRoundKey
在每一次循环中，通过 [Rijndael 密钥生成方案](https://en.wikipedia.org/wiki/Rijndael_key_schedule)从主密钥中生成一个子密钥，这个子密钥的大小应该等同于块的大小，并且以列优先的方式排列在一个矩阵里（每个块也是以这样的方式排列在矩阵里的）。
接下来将这个子密钥的值与块上对应位置的值 XOR 起来，形成一个新的矩阵，到这里这一过程就算完成了。
![220px-AES-AddRoundKey.svg](/uploads/AES是个什么鬼？/AES-AddRoundKey.png)￼

### SubBytes
这一步会使用到一个叫做 Rijndael S-box 的东西，它其实就是一个8位的代换表，每一个字节的数据都可以在表中查到对应的代换结果。只要这个 S-box 在构建的时候足够好，就可以大大降低这次加密的线性关系。下面是一个6位 S-box 的例子，输入的值是011011，输出的值是1001。
![S-box-DES--input011011](/uploads/AES是个什么鬼？/S-box-DES-input011011.png)￼

将块矩阵中的每一个元素通过 S-box 进行代换，组成一个代换后的矩阵，就是 SubBytes 这一步的工作。
![320px-AES-SubBytes.svg](/uploads/AES是个什么鬼？/AES-SubBytes.png)￼

### ShiftRows
这一步容易理解，就是把块矩阵中的每一行都进行一个向左循环移位，最后的效果是要让输出矩阵的每一列上的元素都属于输入矩阵原本不同的列。
这样做可以保证每一列上的元素都是非线性相关的。
![320px-AES-ShiftRows.svg](/uploads/AES是个什么鬼？/AES-ShiftRows.png)￼

### MixColumns
这个部分会接受4个字节的输入，并输出4个字节，而且每一个字节输入的字节都会对输出造成影响，所以它跟上面的 ShiftRows 一起为加密算法提供了良好的扩散性.

> 扩散性（Diffusion）：如果改变了任意1位的原文，密文中一半以上的位也应该会跟着改变；反过来，改变了任意1位密文，得到的原文也应该有一半以上的位被改变。—— Stallings, William (2014). Cryptography and Network Security (6th ed.)

简单来说，这一步就是讲输入矩阵的每一列与一个固定的多项式在一定条件下相乘。最终得到的将会是一个与输入矩阵完全不一样的输出矩阵。
![320px-AES-MixColumns.svg](/uploads/AES是个什么鬼？/AES-MixColumns.png)￼

> 更多资料
> * [伽罗华域(Galois Field，GF，有限域)乘法运算](http://blog.csdn.net/mengboy/article/details/1514445)
> * [Confusion and diffusion](https://en.wikipedia.org/wiki/Confusion_and_diffusion)

## 填充算法
对于块加密算法来说，如果数据的长度不满一个块的大小，我们就需要主动填充一些数据，让这个块的大小可以满足要求，于是，一个合适的填充算法就显得尤为重要。
经过导师的提醒并且在网上读了一些博客之后发现，[Java端与iOS端使用的AES填充算法是不一样的](http://my.oschina.net/nicsun/blog/95632)，在 Java 端上使用的是 PKCS5Padding ，而在iOS端上使用的是 PKCS7Padding 。所以就会导致在其中一端上加解密没有问题，但是把密文发到另一端上解密就会得到完全不同的结果。
P.S. 这里说到的 *Java 端* 应该是指服务器端，Android 端上不知道有没有这个问题。

> PKCS5 相当于是 PKCS7 的一个子集，因为 PKCS7 理论上支持1~255字节的块大小填充，而 PKCS5 只支持8字节的块大小填充。其实 PKCS5 更多是应用在 DES/3DES 上。

具体的填充过程也非常好理解，直接举例子好了：比如说块大小为8字节的加密算法，现在有一串长度为9的数据：
`FF FF FF FF FF FF FF FF FF`（9个FF）
使用 PKCS7 算法去填充的话，结果就是这样的
`FF FF FF FF FF FF FF FF FF 07 07 07 07 07 07 07`（9个FF和7个07）
填充的目的就是把块给补满，所以这里填充的长度为7；而采用 PKCS7 算法的话，填充的每一个字节都是填充长度的十六进制数，那就也是7。

> 有趣的是，如果采用 PKCS5 去填充，因为它的目标块大小是8，所以这里会填充一个01。详情可以参考[Can AES use PKCS#5 padding](http://crypto.stackexchange.com/a/11274)里的最佳答案。

## 参考资料
* [Advanced Encryption Standard](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)
* [Rijndael 密钥生成方案](https://en.wikipedia.org/wiki/Rijndael_key_schedule)
* [伽罗华域(Galois Field，GF，有限域)乘法运算](http://blog.csdn.net/mengboy/article/details/1514445)
* [Confusion and diffusion](https://en.wikipedia.org/wiki/Confusion_and_diffusion)
* [关于AES256算法java端加密，ios端解密出现无法解密问题的解决方案](http://my.oschina.net/nicsun/blog/95632)
* [Can AES use PKCS#5 padding](http://crypto.stackexchange.com/questions/11272/can-aes-use-pkcs5-padding)
