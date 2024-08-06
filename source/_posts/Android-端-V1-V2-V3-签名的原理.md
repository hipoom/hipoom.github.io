---
title: Android 端 V1/V2/V3 签名的原理
date: 2024-08-06 23:59:43
tags:
---

Android 的安装包签名方案到目前有3个版本，分别是：  

* 最初签名方案V1；
* 为了提高验证速度和覆盖度在 7.0 引入的 V2；
* 以及为了实现密钥轮转在 9.0 引入的 V3。

让我们分别了解一下这些签名的原理：

---

# 一、 V1 签名方案

## 1. 签名相关的文件
apk 本质是个 zip 文件，解压缩后，在 `META-INFO` 文件夹中可以看到有 `MANIFEST.MF`、`CERT.SF`、`CERT.RSA` 三个文件。这三个文件在签名时创建，在安装时用于验证签名。下面让我们看一下这三个文件各自的作用：

### 1.1 MANIFEST.MF文件

**文件的作用：**
记录 apk 中每一个文件对应的摘要信息，防止某个文件被篡改。

**文件的内容：**
打开 MANIFEST.MF 文件可以看到文件内容是这种格式：
```
Manifest-Version: 1.0
Built-By: Generated-by-ADT
Created-By: Android Gradle 2.3.1

Name: res/drawable-hdpi-v4/tracepoint_tip.png
SHA1-Digest: UqNwQcd9oLGpVfILjkVOtNQmySA=

Name: res/layout/activity_new_base_layout.xml
SHA1-Digest: Uw3jXiCR9Msf9C6P0Mjcmh2/A/E=

...
```
前三行记录了基础信息，后面每一块都对应了 apk 中一个原始文件的数据摘要，摘要算法是 SHA-1。
在 MANIFEST.MF 文件没被篡改的情况下，可以用于保证 apk 中的其他文件不被篡改。
那怎么保证 MANIFEST.MF 文件本身不被篡改呢？ 就是靠下面的 CERT.SF 文件了：


---

### 1.2 CERT.SF文件
**文件的作用：**
记录 **MANIFEST.MF** 文件的摘要，以及 **MANIFEST.MF** 中，每个数据块的摘要。防止 **MANIFEST.MF** 被篡改。

**文件的内容：**
CERT.SF 的文件内容如下：
```
Signature-Version: 1.0
SHA1-Digest-Manifest: m4hofJv2im9b2HQo/h6VPKRnzqE=
Created-By: 1.0 (Android)

Name: res/drawable-hdpi-v4/tracepoint_tip.png
SHA1-Digest: UqNwQcd9oLGpVfILjkVOtNQmySA=

Name: res/layout/activity_new_base_layout.xml
SHA1-Digest: Uw3jXiCR9Msf9C6P0Mjcmh2/A/E=
...
```

第一行 **Signature-Version** 记录了签名版本；
第二行 **SHA1-Digest-Manifest** 记录了整个 MANIFEST.MF 文件的摘要；

后面每一块都是 MANIFEST.MF 中对应数据块的摘要；
例如，CERT.SF 中对应的这一段：
```
Name: res/drawable-hdpi-v4/tracepoint_tip.png
SHA1-Digest: UqNwQcd9oLGpVfILjkVOtNQmySA=
```
其中「UqNwQcd9oLGpVfILjkVOtNQmySA=」就是 MANIFEST.MF 中这一段的摘要（包含换行符）：
```properties
Name: res/drawable-hdpi-v4/tracepoint_tip.png
SHA1-Digest: UqNwQcd9oLGpVfILjkVOtNQmySA=
\r\n
```


CERT.SF 如果没被篡改，就能用于验证清单文件 MANIFEST.MF 是否被篡改。但又怎么验证 CERT.SF 是否被篡改呢？ 靠的就是签名文件 CERT.RSA 了：

---

### 1.3 CERT.RSA文件

**文件的作用：**
这个文件是为了验证 CERT.SF 文件有没有被篡改。

**文件的内容：**
它包含了 「对 **CERT.SF** 文件的签名」以及「包含公钥的开发者证书」。

如果不了解 证书和签名 是如何用于数据验证的，可以看[《摘要、签名与数字证书都是什么？》](https://www.hipoom.com/2024/08/05/%E6%91%98%E8%A6%81%E5%92%8C%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%95%B0%E5%AD%97%E8%AF%81%E4%B9%A6%E9%83%BD%E6%98%AF%E4%BB%80%E4%B9%88/)

---


## 2. V1的签名机制
![V1签名的文件生成过程](./1.webp)

签名的流程如下：
1. 计算每一个原始文件的 SHA-1 摘要，写入到 **MANIFEST.MF** 中；
2. 计算整个 **MANIFEST.MF** 文件的 SHA-1 摘要，写入到 **CERT.SF** 中；
3. 计算 **MANIFEST.MF** 中，每一块的 SHA-1 摘要，写入到 **CERT.SF** 中；
4. 计算整个 **CERT.SF** 文件的摘要，使用开发者私钥计算出摘要的签名；
5. 将签名和开发者证书(X.509)写入  **CERT.RSA** 。

---

## 3. V1签名是怎么校验的？
校验的流程如下：
1. 取出 **CERT.RSA** 中包含的开发者证书；
2. 通过系统的根证书（CA证书）验证这个开发者证书是否可信；
3. 如果开发者证书可信，用证书中的公钥解密 **CERT.RSA** 中包含的签名。
4. 计算 **CERT.SF** 的签名；
5. 对比 (3) 和 (4) 的签名是否一致；
6. 如果一致，用 **CERT.SF** 去校验 **MANIFEST.MF** 是否被修改；
7. 如果没有被修改，再用 **MANIFEST.MF** 中的每一块数据去校验每一个文件是否被修改。


---

## 4. V1签名如何防止篡改
假如攻击者修改了其中某一个文件，那么他必须修改 **MANIFEST.MF** 中对应文件的摘要，否则这个文件校验不通过；
接着还得修改 **CERT.SF** 中的摘要，否则摘要校验不过；
还得重新计算 **CERT.SF** 的签名，否则签名校验不通过；
但是计算签名需要私钥，私钥在开发者手中，攻击者没有私钥，所以无法签名。

---

## 5. V1签名存在的问题
校验速度慢：需要对 apk 中的每个文件都计算摘要并验证，如果文件很多，校验时间会很长。
完整性不够：V1 签名只会校验 Zip 文件中的部分文件，例如 **META-INFO** 文件夹就不会参与校验。



----


# 二、 V2 签名方案

V2 签名是在 Android7.0 之后引入的，它解决了 V1 签名校验时速度慢的问题，同时对完整性的校验扩展到整个安装包。

了解 V2 签名原理之前，我们先了解一下 Zip 文件：

## 1. Zip 文件

### 1.1 Zip 文件的格式和解析过程

![Zip文件格式](./2.webp)

1. 先从文件尾部查找 0x06054b50，确定 `End Of Central Directory Record` 区域的起始位置；
2. 解析 EoCD 区域，并获得中央目录的起始位置；
3. 根据起始位置，逐个解析文件。

**从解析过程可以看出，如果在 「文件信息部分」 和 「中央目录部分」之间插入了其他数据，是不会影响 Zip 文件的解压缩的。**

---

## 2. V2 签名数据块的格式
V2 签名时，会将 签名信息块 插入到 Zip 文件的「文件信息」和「中央目录」之间，如图：
![apk V2 签名前后对比](./3.webp)

「Apk Signing Block」的具体结构：

![V2签名数据块的结构](./4.webp)

签名块的前8个字节记录了所有键值对数据块的大小。其后紧接着键值对数据块，数据块由一个个的键值对块组成。
每个键值对块的开始8字节记录了「键值对的ID」和「键值对的Value」的大小。接下来4字节是键值对的ID，后面跟着对应的值。
**ID = 0x7109871a 的键值对块就是保存签名信息的地方。**
键值对数据块的后面还有8个字节，也是用于记录「键值对块」的大小，它的值和最开始的8字节相同。
签名块的末尾是一个魔数，也就是‘APK Sig Block 42’的 ASCII 码。

下一篇文章我们会讲到如何在这里动态插入其他信息，例如渠道信息等。

签名信息的具体结构：
![签名信息的结构](./5.webp)

---

## 3. V2 摘要计算方式
V2 签名摘要的计算就不是按照文件计算的了，而是按照 1MB 为单位计算：

![apk按照1MB大小计算摘要](./6.webp)

步骤：
1. 对原始apk文件的 文件信息部分、中央目录部分、EoCD部分，按照 1MB 大小分割为多个小块（Chunks）;
2. 分别对每一个小块计算其摘要，类似于 V1 签名中的 MANIFEST.MF 文件；
3. 对(2)中所有摘要计算其摘要，类似于 V1 签名中的 CERT.SF 文件；


## 4. V2 签名的校验
Android 7.0 及以上在校验时，会先判断是否具有 V2 签名，如果有 V2 签名，会走 V2 签名的校验流程，不再验证V1签名了。

如何判断是否有V2签名？
根据Zip文件格式的规则，我们可以找到中央目录区的起始位置。
读取从起始位置开始往回的16个字节，判断这16个字节的值是否为 "Apk Sig Block 42"，如果是，则对应上了魔数，说明有 V2 签名。后续就是解析 V2 签名块的流程了。

用公钥和签名对蓝色区域验证，验证通过后，用 「APK数据摘要集」对APK每一块做验证。

----


# 三、 V3 签名方案
V3 签名方案的签名块格式和V2完全一样，只是 V2 的签名块信息存放在 ID = 0x7109871a 的数据块中，而 V3 的签名信息存放在 ID = 0xf05368c0 的数据块中。

在这个新的数据块中，记录了旧的签名信息和新的签名信息，以密钥转轮的方案，做签名的替换和升级。这意味着我们可以更改 APK 的签名。

V3 签名块的大小必须是 4096 的整数倍，否则在安装时会出现如下异常：
```
adb: failed to install xxx.apk: Failure [INSTALL_PARSE_FAILED_NO_CERTIFICATES:
Failed to collect certificates from /data/app/xxxx.apk using APK Signature Scheme v3:
Size of APK Signing Block is not a multiple of 4096: xxxx]
```