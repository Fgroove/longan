---
title: AudioTrack::start/stop
author: Mista
date: 2021-11-03 21:32:00 +0800
categories: [audio, framework]
tags: [audio playback]
---

AudioTrack::start/stop，开始/停止播放，主要是：

1. start、startOutput、start_output_stream
2. stop、stopOutput、stop_output_stream
3. pause、flush、release

### AudioTrack::start

startOutput

### AudioTrack::stop

stopOutput

### AudioTrack::pause

remove活跃的track

### AudioTrack::flush

清空buffer

### AudioTrack::release

removeTrack_l
