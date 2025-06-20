---
layout: post
title: "Android学习-Android架构"
date:   2025-6-17
tags: [Android]
comments: true
author: braisedp
toc : true
---

<!--more-->

## Android架构

**Android系统架构图**

![android arch](../images/2025-6-20-android_architecture/android-stack.svg)

> 图片来源：https://source.android.google.cn/docs/core/architecture?hl=zh-cn

从Android的架构图可以看出，Android架构中从上至下通常包含如下层级：

|层级 | 作用|
|--|--|
|App层|厂商或者开发者提供的应用程序|
|Framework层| 运行Zygote、System Server，Media Server等进程|
|Android Runtime 层|虚拟机,提供AOT和JIT编译以及垃圾回收功能|
|HAL层|提供硬件操作的标准接口|
|Linux Kernel|提供最基本的进程管理、内存管理等功能以及Android对Linux内核的拓展|

### Kernel

![kernel](../images/2025-6-20-android_architecture/Kernel.svg)

启动内核时，首先启动Swapper进程，Swapper进程会初始化Driver、进程管理、内存管理等

kthreadd进程会创建内核工作线程kworkder，软中断线程ksoftirqd，thermal等内核守护进程，其他所有内核线程也都由kthreadd创建

### HAL

HAL层为硬件访问提供了标准化的接口，向上屏蔽了驱动程序具体的实现细节，编写硬件访问服务时通过HAL层提供的接口访问硬件

### Android Runtime (ART)

ART是Android用来运行Java程序的虚拟机，Zygote进程是Android的第一个Java进程，由`init`进程创建，是所有Java进程的父进程

### Framework

![framework](../images/2025-6-20-android_architecture/Framework.svg)

Framework层提供的API给APP层进行调用并运行在应用程序当中，而其提供的服务运行在System Server进程中

比如ActivityManager，其通过`Activity.java`向应用程序提供一组API，而其`ActivityManagerService.java`运行在`system_server`进程中，对应用程序，系统资源等进行管理

### App层

Laucher（第一个创建的App层进程）, Browser, Phone等进程都是由Zygote进程创建的

