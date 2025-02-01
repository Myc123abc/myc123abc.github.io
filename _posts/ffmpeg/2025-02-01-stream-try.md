---
title: "ffmpeg Stream Try"
excerpt: "ffmpeg流式播放的尝试"
date: 2025-02-01T22:52:00+08:00
categories:
  - blog
tags:
  - ffmpeg
---

哎，又碰壁了。虽然比上次学习ffmpeg有进步了，至少能解码音频并播放，但现在又卡在如何流式解码音频并同步播放。

目前尝试了有一天半进展不佳，已经有些气馁了。

还是对ffmpeg了解的太浅了。唯一有什么收获的话就是知道怎么用avformat去获取各个音频所需要的数据，以至于在正式解码前能把`WAVEFORMATEX`给初始化了（`WAVEFORMATEX`是xaudio2创建`source voice`所必要的结构体）。

目前打算放缓一些，还是要深入学习ffmpeg才能更好的使用它，所以打算花一段时间，把ffplay的源码搞明白吧。