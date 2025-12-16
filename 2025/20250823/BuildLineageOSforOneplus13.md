---
title: 为一加13编译LineageOS-22.2
date: 2025-7-14 19:00:00

cover: https://tc.z.wiki/autoupload/f/s53jY0hzeXWnA2DBfRnx7Z_Kv4b_7Q93KIuY3QIXybyyl5f0KlZfm6UsKj-HyTuv/20250715/Qd5M/840X1200/60168840_p0.jpg

tags:

  - Oneplus13

  - LineageOS

  - Android

  - build

  - GSI

  - Treble Project
categories:

  - 踩坑实况

keywords: Oneplus13,LineageOS,Build,Android
description: Build LineageOS-22.2 GSI for Oneplus13 global aka dodge without official ROM release
---

## 前情提要
网上看到一加官方收紧Bootloader解锁政策，顿感不对劲，恰逢父亲生日，国补降价，骁龙8 Elite能打，果断某宝下单两部一加13，抢在新的装载ColorOS 16的答辩上架前拿到手，一部孝敬老爹一部换掉老1+9（傻逼888）。

当然了骁龙8 Elite和24+1T的配置再怎么流畅在ColorOS上也只能算作是流畅的窜稀，这B国产ROM全是广告和追踪器，用不了一点；个人工作原因又不方便用满血GMS的OxygenOS，所以视角投向了一年使用体验非常OK的lineageOS：AOSP，可以完全切割GMS（工作原因），Debug模式下默认给root权限到adb shell，如果说OxygenOS还算毛坯房，那这个算是烂尾楼，真的什么都没有，就一个空Android壳子和增强控制（当然也没有广告和原厂ROM的广告SDK的各种追踪器）。但是问题来了：没有官方支持的release镜像，要想摆脱ColorOS这坨就只能自己动手编译整个操作系统。

##