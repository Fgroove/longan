---
title: audio volume
author: Mista
date: 2021-11-03 21:55:00 +0800
categories: [audio, framework]
tags: [audio volume]
---

简单记录下 Android 原生音量管理，主要包括：

1. Audio 初始化阶段加载音量曲线：Android 音量管理在 audioserver 的 AudioPolicyManager 模块中实现，在 audioserver 初始化时会创建并初始化 AudioPolicyManager，APM 初始化会从 `audio_policy_volumes.xml` 和 `default_volume_tables.xml` 中加载音量曲线
2. 音量调节流程：音量调节（音量键/音量条）最终也是在 APM 根据音量曲线计算出音量增益，在 AudioFlinger 的播放线程重采样是生效



## 音量曲线加载

在 APM 初始化中，会加载音量曲线，主要实现在loadVolumeConfig()，代码如下

```C++
engineConfig::ParsingResult EngineBase::loadAudioPolicyEngineConfig()
{
    auto loadVolumeConfig = [](auto &volumeGroups, auto &volumeConfig) {
        // Ensure name unicity to prevent duplicate
        LOG_ALWAYS_FATAL_IF(std::any_of(std::begin(volumeGroups), std::end(volumeGroups),
                                     [&volumeConfig](const auto &volumeGroup) {
                return volumeConfig.name == volumeGroup.second->getName(); }),
                            "group name %s defined twice, review the configuration",
                            volumeConfig.name.c_str());

        sp<VolumeGroup> volumeGroup = new VolumeGroup(volumeConfig.name, volumeConfig.indexMin,
                                                      volumeConfig.indexMax);
        volumeGroups[volumeGroup->getId()] = volumeGroup;

        for (auto &configCurve : volumeConfig.volumeCurves) {
            device_category deviceCat = DEVICE_CATEGORY_SPEAKER;
            if (!DeviceCategoryConverter::fromString(configCurve.deviceCategory, deviceCat)) {
                ALOGE("%s: Invalid %s", __FUNCTION__, configCurve.deviceCategory.c_str());
                continue;
            }
            sp<VolumeCurve> curve = new VolumeCurve(deviceCat);
            for (auto &point : configCurve.curvePoints) {
                curve->add({point.index, point.attenuationInMb});
            }
            volumeGroup->add(curve);
        }
        return volumeGroup;
    };

		......

    for (auto &volumeConfig : result.parsedConfig->volumeGroups) {
				......
        loadVolumeConfig(mVolumeGroups, volumeConfig);
    }

    ......

    return result;
}
```

### 数据类型：VolumeGroup、VolumeCurve 和 CurvePoint

代码涉及三个关键的数据类型：`VolumeGroup`、`VolumeCurve` 和 `CurvePoint`，详见 [VolumeCurve.h](https://cs.android.com/android/platform/superproject/+/master:frameworks/av/services/audiopolicy/engine/common/include/VolumeCurve.h) 和 [VolumeGroup.h](https://cs.android.com/android/platform/superproject/+/master:frameworks/av/services/audiopolicy/engine/common/include/VolumeGroup.h)。

音量曲线从 `audio_policy_volumes.xml` 和 `default_volume_tables.xml` 文件中加载，下面是两个文件的片段，在 [`frameworks/av/services/audiopolicy/config/`](https://cs.android.com/android/platform/superproject/+/master:frameworks/av/services/audiopolicy/config/) 目录下，

```xml
// audio_policy_volumes.xml
<volumes>
    <volume stream="AUDIO_STREAM_VOICE_CALL" deviceCategory="DEVICE_CATEGORY_HEADSET">
        <point>0,-4200</point>
        <point>33,-2800</point>
        <point>66,-1400</point>
        <point>100,0</point>
    </volume>
    <volume stream="AUDIO_STREAM_VOICE_CALL" deviceCategory="DEVICE_CATEGORY_SPEAKER">
        <point>0,-2400</point>
        <point>33,-1600</point>
        <point>66,-800</point>
        <point>100,0</point>
    </volume>
    <volume stream="AUDIO_STREAM_VOICE_CALL" deviceCategory="DEVICE_CATEGORY_EARPIECE">
        <point>0,-2400</point>
        <point>33,-1600</point>
        <point>66,-800</point>
        <point>100,0</point>
    </volume>
    <volume stream="AUDIO_STREAM_VOICE_CALL" deviceCategory="DEVICE_CATEGORY_EXT_MEDIA"
                                             ref="DEFAULT_MEDIA_VOLUME_CURVE"/>
    <volume stream="AUDIO_STREAM_VOICE_CALL" deviceCategory="DEVICE_CATEGORY_HEARING_AID"
                                             ref="DEFAULT_NON_MUTABLE_HEARING_AID_VOLUME_CURVE"/>
		......
</volumes>

// default_volume_tables.xml
<volumes>
  	......
    <reference name="DEFAULT_MEDIA_VOLUME_CURVE">
    <!-- Default Media reference Volume Curve -->
        <point>1,-5800</point>
        <point>20,-4000</point>
        <point>60,-1700</point>
        <point>100,0</point>
    </reference>
  	......
  	<reference name="DEFAULT_NON_MUTABLE_HEARING_AID_VOLUME_CURVE">
    <!-- Default non-mutable Hearing Aid Volume Curve -->
    <!--     based on DEFAULT_HEARING_AID_VOLUME_CURVE -->
        <point>0,-12700</point>
        <point>20,-8000</point>
        <point>60,-4000</point>
        <point>100,0</point>
    </reference>
</volumes>
```

* VolumeGroup: xml 文件中 stream 类型相同的一组 `<volume>` 为一个 VolumeGroup，比如 `stream="AUDIO_STREAM_VOICE_CALL"` 的 5 个 `<volume>` 对应 `VolumeGroup(AUDIO_STREAM_VOICE_CALL)`。可以看到，**VolumeGroup 以 stream 类型划分，不指定设备**。
* VolumeCurve: xml 文件中每一个`<volume>` 为一个 VolumeCurve，比如 `<volume stream="AUDIO_STREAM_VOICE_CALL" deviceCategory="DEVICE_CATEGORY_HEADSET">` 对应 `VolumeCurve(DEVICE_CATEGORY_HEADSET)`。**VolumeCurve 已经指定了 stream 类型和设备**。
* CurvePoint: xml 文件中每一个 `<point>` 为一个 CurvePoint，比如 `<volume stream="AUDIO_STREAM_VOICE_CALL" deviceCategory="DEVICE_CATEGORY_HEADSET">`  节点下的 4 个 `<point>` 分别对应 `CurvePoint(0, -4200)`、 `CurvePoint(33, -2800)`、 `CurvePoint(66, -1400)` 和 `CurvePoint(100, 0)`。

### 加载音量曲线

1. 解析 xml 文件并按 stream 类型分为若干组 VolumeGroup，
2. 每个 stream 类型/VolumeGroup 根据 device 类型有分为多个 VolumeCurve，
3. 每个 VolumeCurve 又包含若干个 CurvePoint，解析完成后保存到 mVolumeGroups。

音量曲线，通常所说的音量曲线就是 VolumeCurve 及其包含的一组 CurvePoint。每个 <stream, volume> 对都对应着一条音量曲线。

在音量调节时，根据 <stream, volume> 拿到对应的 VolumeCurve，然后根据 index 拿到对应的 CurvePoint，取得 attenuationInMb 计算增益值。

以上代码片段加载后的音量曲线对应关系如下，

![](https://raw.githubusercontent.com/Fgroove/longan/master/assets/img/audio_framework/VolumeCurve.png)

>adb shell dumpsys media.audio_policy，可以查看音量曲线参数，
>
>adb pull \<product>/etc/audio_policy_volumes.xml 和 default_volume_tables.xml，可以直接查看音量配置文件

## 音量调节

音量调节有两个入口，音量键（adjustSuggestedStreamVolume）和音量条（setStreamVolume）调节，其底层实现原理一样。

1. 音量键：音量键调节主要使用的是 `adjustSuggestedStreamVolume()` 方法来进行音量的调节。这个方法会**自动更新建议的音量类型**，以便用户可以正确地使用音量调节按键来调整音量，并且它会产生音量变化的反馈（例如声音和 UI 的变化）。

   当用户按下音量键时，系统会检测当前的音量类型，并将音量调整方向（减小/放大）传递给 `adjustSuggestedStreamVolume()` 方法。然后，该方法会根据调整方向自动增加或减少指定音量类型的音量，并自动更新建议的音量类型，以便用户可以继续使用音量调节按键来调整音量。

2. 音量条：设置和音量控制面板都是通过拖动音量条调节音量，使用的是 `setStreamVolume()` 方法来进行音量的调节。在设置中，用户可以直接拖动滑块或者按下加减按钮来调节音量大小。这些操作会触发系统调用 `setStreamVolume()` 方法，以直接设置指定音量类型的音量。

   与 `adjustSuggestedStreamVolume()` 不同，`setStreamVolume()` **不会尝试自动更新建议的音量类型，并且它通常不会产生音量变化的反馈**。因此，如果需要在应用中进行音量调节，建议使用 `adjustSuggestedStreamVolume()` 方法。

### 音量键：adjustSuggestedStreamVolume

`adjustSuggestedStreamVolume()` 方法是 Android 系统中用于调整音量的一个重要 API，它可用于在应用程序中动态调整音量。

下面将详细介绍该方法及其使用方法。

#### 1. adjustSuggestedStreamVolume() 方法概述

`adjustSuggestedStreamVolume()` 方法是在 `AudioManager` 类中定义的，它有如下两种不同的调用方式：

```java
public void adjustSuggestedStreamVolume(int direction, int suggestedStreamType, int flags)

public void adjustSuggestedStreamVolume(int direction, int suggestedStreamType, int flags, String packageName, int uid)
```

第一种调用方式是 Android 5.0 及以上版本新增的 API，该方法允许应用程序在调整音量时，指定调整的音量流类型（stream type）。

第二种调用方式是 Android 9.0 及以上版本新增的 API，该方法在第一种调用方式的基础上增加了应用程序包名（packageName）和 UID（uid）参数，以更好地保护用户隐私和安全。

#### 2. adjustSuggestedStreamVolume() 方法参数说明

| 参数名      | 参数类型 | 参数说明                                                     |
| :---------- | :------- | :----------------------------------------------------------- |
| direction   | int      | 指定音量调整的方向。可以使用以下常量值：`ADJUST_LOWER`（降低音量）、`ADJUST_RAISE`（增加音量）、`ADJUST_SAME`（不改变音量）。 |
| streamType  | int      | 指定音量调整的音频流类型。可以使用以下常量值：`STREAM_ALARM`（闹钟音量）、`STREAM_DTMF`（DTMF 音量）、`STREAM_MUSIC`（媒体音量）、`STREAM_NOTIFICATION`（通知音量）、`STREAM_RING`（铃声音量）、`STREAM_SYSTEM`（系统音量）。 |
| flags       | int      | 指定音量调整的标志参数。可以使用以下常量值： `FLAG_SHOW_UI`（显示音量调节 UI 界面）、`FLAG_PLAY_SOUND`（播放调节音量的提示音效）、`FLAG_REMOVE_SOUND_AND_VIBRATE`（移除调节音量和振动的提示效果）。 |
| packageName | String   | 指定请求音量调整的应用程序的包名。                           |
| uid         | int      | 指定请求音量调整的应用程序的 UID（User ID）。                |

`adjustSuggestedStreamVolume()` 方法的参数如下所示：

- `direction`：表示用户希望将音量调整增加还是减少，它可以是以下三种值之一：

  - `ADJUST_LOWER`：表示将音量调整降低（减小）。
  - `ADJUST_RAISE`：表示将音量调整增加（放大）。
  - `ADJUST_SAME`：表示将音量保持不变。

- `suggestedStreamType`：表示要调整的音量流类型，它可以是以下常量之一：

  - `STREAM_ALARM`：表示警报音量流类型。

  - `STREAM_DTMF`：表示 DTMF（双音多频）音量流类型。

  - `STREAM_MUSIC`：表示音乐音量流类型。

  - `STREAM_NOTIFICATION`：表示通知音量流类型。

  - `STREAM_RING`：表示来电铃声音量流类型。

  - `STREAM_SYSTEM`：表示系统音量流类型。

  - `STREAM_VOICE_CALL`：表示语音通话音量流类型。

  - `STREAM_ACCESSIBILITY`：表示辅助功能音量流类型。

    在 Android 5.0 及以上版本中，可以使用 `AudioManager.STREAM_*` 常量来指定要调整的音量流类型，例如 `AudioManager.STREAM_MUSIC`。

- `flags`：表示在音量调整完成后需要执行的操作，它可以包含以下值之一或多个值之和：

  - `FLAG_SHOW_UI`：表示在调整音量时需要显示音量调节 UI 界面。
  - `FLAG_PLAY_SOUND`：表示在调整音量时需要播放调节音量的提示音效。
  - `FLAG_REMOVE_SOUND_AND_VIBRATE`：表示在调整音量时需要停止正在播放的提示音和振动。

- `packageName`：表示调用该方法的应用程序的包名。

- `uid`：表示调用该方法的应用程序的 UID（User ID）。

#### 3. adjustSuggestedStreamVolume() 方法的使用示例

以下是使用 `adjustSuggestedStreamVolume()` 方法来调整音量的示例代码：

```java
AudioManager audioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);

// 调整媒体音量的大小
audioManager.adjustSuggestedStreamVolume(
  			AudioManager.ADJUST_LOWER, 
  			AudioManager.STREAM_MUSIC, 
  			AudioManager.FLAG_PLAY_SOUND);

// 调整铃声音量的大小
audioManager.adjustSuggestedStreamVolume(
  			AudioManager.ADJUST_RAISE, 
  			AudioManager.STREAM_RING, 
  			AudioManager.FLAG_SHOW_UI | AudioManager.FLAG_PLAY_SOUND);
```

上述代码中，我们

1. 首先通过 `getSystemService()` 方法获取 `AudioManager` 实例，然后分别使用 `adjustSuggestedStreamVolume()` 方法来调整媒体音量和铃声音量的大小。
2. 在第一次调用中，我们将媒体音量调整减小一个单位，并播放调节音量的提示音效；
3. 而在第二次调用中，我们将铃声音量调整增加一个单位，并显示音量调节 UI 界面，并播放调节音量的提示音效。

需要注意的是，由于不同的设备和系统版本可能会有所不同，因此调用 `adjustSuggestedStreamVolume()` 方法可能会有一些不同的行为和效果。因此，我们应该在应用程序中仔细测试和调试该方法，以确保它能够正确地工作和达到预期的效果。

#### 4. adjustSuggestedStreamVolume() 方法的调用流程

![img](https://raw.githubusercontent.com/Fgroove/longan/master/assets/img/audio_framework/adjustSuggestedStreamVolume.png)

在上述UML图中，

- `App`表示应用程序，
- `SystemService`表示Android系统中的系统服务，
- `AudioFlinger`表示Android系统中的AudioFlinger音频服务，
- `AudioHAL`表示Android系统中的音频HAL库。

流程从应用程序中调用`AudioManager.adjustSuggestedStreamVolume()`方法开始，然后通过系统服务和AudioFlinger服务调用底层的Audio HAL库实现音量调节操作。

调用过程中有一些数据的传递和返回，但并不在本图中显示。

### 音量条：setStreamVolume

```java
public void setStreamVolume (int streamType, int index, int flags)
```

其中，参数含义如下：

- streamType：要调整音量的音频流类型，如STREAM_MUSIC、STREAM_RING等等。
- index：要设置的音量大小，可以是一个介于0到getStreamMaxVolume(streamType)之间的整数，其中getStreamMaxVolume(streamType)获取了当前音频流类型的最大音量值。
- flags：标识符，可以是0或者PLAY_SOUND、REMOVE_SOUND_AND_VIBRATE等。

在调用setStreamVolume方法时，会先通过AudioService.getAudioManager()方法获取到一个AudioManager实例，然后调用AudioManager对象的setStreamVolume方法，传入对应的参数即可完成音量调整操作。

注意，由于不同的手机厂商和系统版本可能存在差异，因此在使用setStreamVolume方法时需要仔细查看官方文档或源码，以确保代码的正确性。

以下是一个使用`setStreamVolume`方法将媒体音量设置为50%的示例代码：

```java
scssCopy codeAudioManager audioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
int maxVolume = audioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC);
int volume = (int) (maxVolume * 0.5); // 50% of max volume
audioManager.setStreamVolume(AudioManager.STREAM_MUSIC, volume, 0);
```

在上述示例中，我们

1. 首先获取了`AudioManager`实例，并使用`getStreamMaxVolume`方法获取了当前设备媒体音量的最大值。
2. 然后，我们将该最大值与0.5相乘，得到我们想要的音量值（50%的最大音量）。
3. 最后，我们使用`setStreamVolume`方法将媒体音量设置为该值。其中，第一个参数为要设置音量的流类型（这里为`STREAM_MUSIC`即媒体音量），第二个参数为要设置的音量值，第三个参数为可选的标志位，此处为0。

## 参考

[AOSP源码搜索](https://cs.android.com/android/platform/superproject/+/master:frameworks/av/)

[安卓原生音量控制](https://cailiuzhuang.atlassian.net/wiki/spaces/~825683595/pages/44105729)
