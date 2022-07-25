---
title: audio volume
author: Mista
date: 2021-11-03 21:55:00 +0800
categories: [audio, framework]
tags: [audio volume]
---

简单记录下 Android 音量管理，主要包括：

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

![]()

>adb shell dumpsys media.audio_policy，可以查看音量曲线参数，
>
>adb pull \<product>/etc/audio_policy_volumes.xml 和 default_volume_tables.xml，可以直接查看音量配置文件
