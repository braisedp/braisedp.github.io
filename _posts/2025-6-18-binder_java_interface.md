---
layout: post
title: "Binder学习-Java接口"
date:   2025-6-18
tags: [Android]
comments: true
author: braisedp
toc : true
---

<!-- more -->

## Binder学习-Java接口

### 用于使用Binder机制的Java API

| 接口\类  | 作用  | Native | 与C++的对应  |
| -| - | -| - |
| IBinder| 定义进程间调用基本接口|   |          |
| Binder      | 模板类，继承IBinder，提供一些模板方法                   | JavaBBinder | BnBinder |
| Stub        | 用户自定义类，继承Binder，实现其模板方法，与Java服务通信，实现服务接口 |             | BnXXX    |
| Proxy       | 用户自定义类，实现服务接口，负责将client对服务的调用发送给binder驱动 |             | BpXXX    |
| BinderProxy |                                          |             | BpBinder |


### 类关系：
 ```mermaid
 classDiagram
	 IInterface <|-- XXXService
	 IBinder <|-- Binder
	 Binder <|-- Stub
	 XXXService <|-- Stub
	 XXXService <|-- Proxy
	 IBinder<|--BinderProxy
	 Proxy *-- BinderProxy : mRemote
	 
	class IInterface{
		 + asBinder(): IBinder
	 }
	 
	class IBinder{
		 + transact(code,data,reply,flags): boolean
	 }

	class Binder{
		+ transact()
		+ onTransact(code,data,reply,flags): boolean
	}
	
	class Stub{
		+ asInterface()$: XXXService
		+ onTransact()
	}

	class Proxy{
		- IBinder mRemote$
		~ int TRANSACTION_XXX$
		+ XXXMethod()
	}
	
	class XXXService{
		- XXXMethod()
	}
	
	class BinderProxy{
		- long mObject
		+ transact()
	}
```


