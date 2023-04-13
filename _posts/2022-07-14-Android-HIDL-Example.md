---
title: Android HAL 与 HIDL 开发实例
author: Mista
date: 2022-07-14 00:55:00 +0800
categories: [Android, build]
tags: [android HIDL]
---

Android 8.0之后，`/dev/binder`拓展多出了两个域，即`/dev/hwbinder`和`/dev/vndbinder`，

* `/dev/hwbinder` 主要用于 HIDL 接口的通信，
* `/dev/vndbinder` 则是用于 vendor 进程之间的 AIDL 通信。

作为 OEM/ODM 厂商，需要了解 Android 硬件的开发和集成流程，把自己硬件添加到 ROM 中，比如集成杜比音效。

> 关于 ROM：国内的定制系统开发者，经常会陷入自己的产品究竟是应该称为 OS (ColorOS) 还是 UI (MIUI) 的争论，为了避免此类争论和表示谦虚，会自称为 ROM。应该就是 Android /system分区。

## [HAL 硬件抽象层](https://source.android.com/docs/core/architecture/hal)

HAL，Hardware Abstraction Layer，即硬件抽象层。

从碎片化角度看，系统设计者希望底层硬件按照类型整齐划一，而不是半导体厂商各自的接口标准不一；从商业角度看，OEMs 自家的硬件驱动是不愿意开源出去被人研究，所以要求 OS 可以无视底层实现，只需商定统一的交互协议。

对于 Android 而言，这层抽象就是HAL，简而言之，Android HAL 就是定义了`.h`接口，并由 OEMs 实现接口为动态链接库`.so`，并使用商定的方式去加载和调用。

现在已经是 Android 13，但早在 Android 8 之后就弃用了 HAL，不过由于碎片化的原因，目前还有 IoT 等设备仍在使用传统的 HAL 模式。

## [HIDL](https://source.android.com/docs/core/architecture/hidl)

HAL 是最初的硬件抽象方案，在 Android 8 之后废弃并被 HIDL 取代。

HIDL，HAL Interface Definition Language，和AIDL类似，是用来描述硬件接口的语言。HIDL 设计的初衷是更新 frameworks 时避免重新编译 HAL，HAL 可以由厂商单独编译并定义在 vendor 分区中单独更新，以及进行版本管理。

## HIDL 开发实例

... to be continued





reference:

[HAL 硬件抽象层](https://source.android.com/docs/core/architecture/hal)

[HIDL](https://source.android.com/docs/core/architecture/hidl)
