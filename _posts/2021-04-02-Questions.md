---
title: Question list
author: Mista
date: 2021-04-02 00:32:00 +0800
categories: [Question]
tags: [Summary]
pin: true
---


这篇博文目的是记录学习工作总遇到的问题，作为博客写作的一个方向，解决这些问题。

## CPP

### 1 [字符串拼接数字](https://fgroove.github.io/longan/posts/how-to-concatenate-a-stdstring-and-an-int/)

对于Java和Python来说，操作符`+`就足够了，但是C++不支持直接拼接字符串和数字。

### 2 [调用其他类的属性方法]()

对于Java和Python来说，`import`对应包/类就可以，但是C++不行，`extern`外部变量声明也不能在类里面。

那么，AudioFlinger怎么才能使用AudioDetectionManager里面的mTrackInfo呢？

* updateAudioDetectState函数里面调用了AudioDetectionManager里面的函数，
* AudioDetect特性的函数应该都和Audiodetection有关联，可以参考
* AudioDetection::mTrackInfo试一试，

### 3 [回调函数](https://fgroove.github.io/longan/posts/what-is-a-callback-function/)

回调函数，因为名字没起好，经常让人摸不着头脑；回调函数作为参数传给caller，在caller函数执行完调用。

可以理解为call at the back，或者call after function。

## Java

### 1 [public static final array 安全漏洞](https://fgroove.github.io/longan/posts/why-are-public-static-final-array-a-security-hole/)

Effective Java中提到的潜在安全漏洞，`public static final array`，