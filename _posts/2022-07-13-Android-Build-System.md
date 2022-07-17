---
title: android build system
author: Mista
date: 2022-07-13 21:55:00 +0800
categories: [Android, build]
tags: [android build system]
---

最近，集成杜比音效用到了很多Android编译知识，记录一下

1. Android编译系统，Android.bp、Soong以及Android.mk
2. Android.bp语法记录，vintf-fragments字段等
3. Android HIDL使用实例

## Android 构造系统

AOSP源码[`build/README.md`](https://android.googlesource.com/platform/build/+/refs/heads/master),

> This is the Makefile-based portion of the Android Build System.
>
> For documentation on how to run a build, see [Usage.txt](https://android.googlesource.com/platform/build/+/refs/heads/master/Usage.txt)
>
> For a list of behavioral changes useful for Android.mk writers see [Changes.md](https://android.googlesource.com/platform/build/+/refs/heads/master/Changes.md)
>
> For an outdated reference on Android.mk files, see [build-system.html](https://android.googlesource.com/platform/build/+/refs/heads/master/core/build-system.html). Our Android.mk files look similar, but are entirely different from the Android.mk files used by the NDK build system. When searching for documentation elsewhere, ensure that it is for the platform build system -- most are not.
>
> This Makefile-based system is in the process of being replaced with [Soong](https://android.googlesource.com/platform/build/soong/+/master), a new build system written in Go. During the transition, all of these makefiles are read by [Kati](https://github.com/google/kati), and generate a ninja file instead of being executed directly. That's combined with a ninja file read by Soong so that the build graph of the two systems can be combined and run as one.

Android 构造系统演进历史大概：

* Android 6.0（包括6.0）之前采用的是纯 Makefile 编译，
* Android 7.0 开始引入Soong，希望加速 Make；但由于改动太大，所以到目前为止 Soong 并没有完全取代 Make，直到 
* Android 13.0 为止还是 Make 和 Soong 两者兼容的系统。但是 Soong 是主流，未来也会全面取代 Make。

![img](/Users/cai/Documents/GitHub/longan/assets/img/aosp_build_procedure.png)
