---
title: Android audio framework
author: Mista
date: 2021-08-20 00:32:00 +0800
categories: [audio, framework]
tags: [audio framework]
pin: true
---

![](https://raw.githubusercontent.com/Fgroove/longan/master/assets/img/audio_framework/audio_framework.png)

Android 音频服务框架的内容说多不多，说少也不少，网上也有各种的教程。我看过的比较系统性的描述就是林学森老师的《深入理解*Android* 内核设计思想》，里面分为 AudioTrack、AudioFlinger 和 AudioPolicyService 三部分来讲解 Audio 模块，但是随着 Android 的演变，现在已经不能涵盖遇到的所有场景，所以从自身实际了解出发，总结下这部分内容：

* Audio data processing: 有点类似于 AudioTrack 及其扩展，音频数据流处理，主要包括音频播放录音框架，音量、音效和变声等分别在重采样、混音等阶段完成等。
* Output & Device Routing: 相当于 AudioFlinger 和 AudioPolicyManager 部分，主要是设备/通路的选择流程、切设备音量设置流程、AudioFocus以及MediaSession机制。
* 快稳省：音频导致的稳定性问题（native crash/SWT/ANR）、性能问题（原神用aaudio引起的杂音问题等）等。

## Audio Data Processing

- Audio Playback: 音频播放，AudioTrack::set/start/write，prepareTracks_l 和 threadLoop_l 等函数。
- Audio Capture：录音，类似于播放机制，数据流向反过来。
- Audio Effect：音效机制。
- Magic Voice：变声，类似机制的还有系统内录、耳返等。

## Output & Device Routing

- Output
- Device Routing：setSpeakerphoneOn/setBluetoothScoOn，getDeviceRoleForStrategy/setForceUse，getDeviceForStrategyInt
- Audio Volume
- AudioFocus
- MediaSession

## Perf&Stability&Save

- SWT/ANR/Native crash
- Audio low latency
- Systrace/Performance
- Binder Performance
