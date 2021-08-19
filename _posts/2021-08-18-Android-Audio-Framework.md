---
title: Android audio framework
author: Mista
date: 2021-08-20 00:32:00 +0800
categories: [audio framework]
tags: [audio framework]
pin: true
---

安卓音频框架，主要理清：

1. audiotrack播放（start、pause、stop）、通路（output）和设备（setOutputDevice）选择，音量和audiofocus使用
2. 稳定性，audioserver crash及引起的anr、watchdog
3. 杂音卡顿，性能、通路强转、音效等原因

## 音频播放过程

### audiotrack

客户端AudioTrack::start

服务端audioflinger、audiopolicymanager::startoutput

hal/driver: audio_hw_primary:start_output_stream

### 通路设备选择

听筒：setMode，setPhoneState，异常守护

免提：setSpeakerPhoneOn

蓝牙：startBluetoothSco，setBluetoothSco

### 音量

VolumeGroupState，Stream音量，Track音量

## 稳定性

由于Google TimeCheck机制的引入，audioserver crash大部分都是这里引起的。

* audioserver crash，会发送debuggerd signal给audiohal，一起打印tombstone方便分析；

* audiohal crash，上层来不及打印tombstone，只有haldeath first打印。

## 杂音卡顿

由于性能原因引起声音卡顿、杂音，可以通过systrace信息确认，

*  三方APP写数据不及时，音频服务得不到足够的数据，导致sleep写入补0数据，
* audioserver向audiohal层写数据时被阻塞，也会引入杂音，体现在0x1531节点，
