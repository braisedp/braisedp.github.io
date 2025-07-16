---
layout: post
title: "Android权限机制"
date:   2025-07-16
tags: [Android, Perssions]
comments: true
author: braisedp
toc : true
---

## Android权限分类

![permission level](../images/2025-7-16-android_permission/permissons.svg)

## Android权限工作流

```mermaid
flowchart LR
A{是否需要使用权限}--是-->B[在manifest中声明权限]
A--否-->C[执行相应功能]
B--是-->D{是否使用运行时权限}
D--否-->C
D--是-->F[请求用户授权权限]
F-->C
```
## Android权限声明

