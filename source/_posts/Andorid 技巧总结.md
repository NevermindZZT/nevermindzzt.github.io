---
layout:     post
title:      Android 技巧总结
subtitle:   
date:       2019-10-215
author:     Letter
header-img:
catalog:    true
tags: 
    - Android
---

## ADB

- 隐藏虚拟键及顶部状态栏：

  adb shell settings put global policy_control immersive.full=*

- 隐藏顶部状态栏（底部虚拟键会显示）：

  adb shell settings put global policy_control immersive.status=*

- 隐藏虚拟键（顶部状态栏会显示）：

  adb shell settings put global policy_control immersive.navigation=*

- 恢复原来的设置：

  adb shell settings put global policy_control null
