---
layout: post
title: "Android 编译"
date:   2025-9-12
tags: [Android, bp, soong, blueprint]
comments: true
author: braisedp
toc : true
---

<!-- more -->

## Android.bp详解

一个Android.bp是一个json类型的文件，内容通常具有如下格式：
```go
module_name{
    "name" : "XXXX",
    ...
}
```

### Android.bp常见用法

#### 声明一个C++默认配置
```go
cc_defaults {
   name: "test_defaults", //模块名称
   shared_libs: ["XXX"], //设置使用共享库
   stl: "none", //设置stl library
}
```

#### 创建一个C++ 动态/静态库
```go
cc_library(_shared|_static){
    name : "test_lib",
    srcs: ["*/test.cpp"],

}
```

#### 创建一个C++ 应用

```go
cc_binary {
   name: "test",
   defaults: ["test_defaults"], //默认配置
   srcs: ["*/minigzip.cpp"], //文件路径
   static_libs: ["test_lib",]
   vendro: true //是否打包到vendor分区
}
```

### 声明一个java默认配置
```go
java_defaults {
    ...
}
```

### 创建一个java .jar包
```go
java_library{
    name: "XXX",
    srcs:["*/test.java"],
    static_libs:["xxx",],
}
```

### 创建一个apk
```go
android_app{
    ...
}
```

### Android.bp 常见字段总结

<table>
<tr><td rowspan="7" > <b>打包目标</b> </td> <td> <code>java_library</code></td> <td>打包成.jar包</td><td/><td/><td/></tr>
<tr><td> <code>cc_library(_static/_shared/_header)</code> </td><td>打包成.lib/.so/头文件库</td><td/><td/><td/></tr>
<tr><td> <code>cc_binary</code> </td><td>打包成native程序</td><td/><td/><td/></tr>
<tr><td> <code>java_binary</code> </td><td>打包成java程序</td><td/><td/><td/></tr>
<tr><td> <code>android_app</code> </td><td>打包成Android应用程序</td><td/><td/><td/></tr>
<tr><td> <code>aidl_interface</code> </td><td>hidl打包成.jar/.so</td><td/><td/><td/></tr>
<tr><td> <code>hidl_interface</code> </td><td>aidl打包成.jar/.so</td><td/><td/><td/></tr>
<tr><td rowspan="2"> <b>文件路径</b></td><td> <code>srcs</code> </td><td>需要打包的文件path</td><td/><td/><td/></tr>
<tr><td> <code>excluded_srcs</code> </td><td>在打包时不可包含的文件path</td><td/><td/><td/></tr>
<tr><td rowspan="2"> <b>依赖的库</b></td><td> <code>static_libs</code> </td><td>依赖的静态库/.jar包</td><td/><td/><td/></tr>
<tr><td> <code>shared_libs</code> </td><td>依赖的动态库</td><td/><td/><td/></tr>
<tr><td rowspan="2"> <b>分区选项</b></td><td> <code>vendor</code> </td><td>是否打包到vendor分区</td><td/><td/><td/></tr>
<tr><td> <code>system_ext</code> </td><td>是否打包到ext分区</td><td/><td/><td/></tr>
<tr><td rowspan="5"> <b>语言选项</b></td><td rowspan="4"> aidl </td><td rowspan="4"><code>back_end</code> </td><td rowspan = "2"> <code>java</code></td><td><code>enabled</code></td><td>是否开启</td></tr>
<tr><td><code>sdk_version</code></td><td>sdk版本，默认system_current</td></tr>
<tr><td rowspan = "2"> <code>ndk</code></td><td><code>enabled</code></td><td>是否开启</td></tr>
<tr> <td><code>sdk_version</code></td><td>sdk版本，默认current</td></tr>
<tr><td> hidl </td><td><code>gen_java</code></td> <td>是否生成.jar库</td><td/><td/></tr>
</table>

上表给出了一些bp文件中的常见字段，所有字段的示意可见[soong docs](../files/2025-9-12-android_pack/soong_build.html)。