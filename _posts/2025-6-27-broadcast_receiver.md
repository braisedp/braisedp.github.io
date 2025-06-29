---
layout: post
title: "BroadcastReceiver学习-Android广播机制"
date:   2025-6-27
tags: [Android, BroadcastReceiver]
comments: true
author: braisedp
toc : true
---


![broad_cast](../images/2025-6-27-broadcast_receiver/broadcast.svg)

Anroid提供了一种消息广播机制，用于向关心相应事件的组件发送通知，通过`ActivityMangerService`，`Activity`和`Service`都可以将`BroadcastReceiver`注册到`ActivityMangerService`中，并监听自己想要的事件，同时提供了静态注册、动态注册和广播优先级等机制


<!-- more -->

## Android的广播机制

