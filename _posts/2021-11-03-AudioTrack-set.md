---
title: AudioTrack::set
author: Mista
date: 2021-11-03 00:32:00 +0800
categories: [audio, framework]
tags: [audio playback]
---

AudioTrack::set，AudioTrack的初始化，主要是：

1. 采样率、声道数、**flags和framecount**等参数设置
2. 创建track、createTrack_l，选择通路getOutputForAttr

### AudioTrack::set

native层的set可以设置更多的参数，所以SoundPool、OpenSL ES和AAudio等直接跳过了Java层API，直接使用的native的AudioTrack构造函数。

### set 参数

Java层JNI调用

Soundpool

OpenSL ES

AAudio

### createTrack_l

## getOutputForAttr
