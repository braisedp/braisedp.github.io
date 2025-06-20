---
layout: post
title: "Binder学习-实践"
date:   2025-6-19
tags: [Android, Binder]
comments: true
author: braisedp
toc : true
---

<!-- more -->

## Binder学习-实践

### 环境

创建一个AIDL HelloService

``` java
interface HelloService {  
  
    String sayHello();  
}
```

创建一个Service MyService

``` java

public class MyService extends Service {  

	//通过Stub与Binder通信
    private final HelloService.Stub mBinder = new HelloService.Stub() {  
  
        @Override  
        public String sayHello() throws RemoteException {  
            return "hello";  
        }  
    };  
  
    @Override  
    public IBinder onBind(Intent intent) {  
        return mBinder;  
    }  
}

```

创建一个Activity ClientActivity

```java
public class ClientActivity extends Activity {  
  
    HelloService helloService;  
  
    ServiceConnection connection = new ServiceConnection() {  
        @Override  
        public void onServiceConnected(ComponentName name, IBinder service) {
	        //通过proxy和binder通信  
            helloService = HelloService.Stub.asInterface(service);  
            try{  
                String  s = helloService.sayHello();  
                Log.d("Client", s);  
            }catch (RemoteException e){  
                e.printStackTrace();  
            }  
        }  
  
        @Override  
        public void onServiceDisconnected(ComponentName name) {  
            helloService = null;  
        }  
    };  
  
  
    @Override  
    protected void onCreate(@Nullable Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_client);  
        Log.d("ClientActivity", "Attempting to bind service...");  
        Intent intent = new Intent(getBaseContext(), MyService.class);  
        bindService(intent,connection, Context.BIND_AUTO_CREATE);  
    }  
  
    @Override  
    protected void onDestroy() {  
        super.onDestroy();  
        unbindService(connection);  
    }  
}
```
### 不同进程间的服务调用流程

1. 观察`ClientActivity`中`onServiceConnected`对象，查看其`service`参数类型如下：

![service对象](../images/2025-6-19-binder_test/BinderProxy.png)
可以看到，接收到的是一个BinderProxy类型的对象。通过
``` java
helloService = HelloService.Stub.asInterface(service);
```
将返回一个Proxy对象 helloService，根据`HelloService.Stub.asInterface`的源码可以看到
``` java
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);  
if (((iin!=null)&&(iin instanceof com.example.mybindertest.HelloService))) {  
  return ((com.example.mybindertest.HelloService)iin);  
}  
return new com.example.mybindertest.HelloService.Stub.Proxy(obj);
```
如果没有找到一个本地的HelloService接口的实例，则返回一个`HelloService.Stub.Proxy`的对象，如下：
![helloSerivce](../images/2025-6-19-binder_test/helloService.png)
 执行`helloService.sayHello`方法：
```java
  public java.lang.String sayHello() throws android.os.RemoteException  
  {  
    android.os.Parcel _data = android.os.Parcel.obtain();  
    android.os.Parcel _reply = android.os.Parcel.obtain();  
    java.lang.String _result;  
    try {  
      _data.writeInterfaceToken(DESCRIPTOR);  
      boolean _status = mRemote.transact(Stub.TRANSACTION_sayHello, _data, _reply, 0);  
      _reply.readException();  
      _result = _reply.readString();  
    }  
    finally {  
      _reply.recycle();  
      _data.recycle();  
    }  
    return _result;  
  }  
}
```
可以看到其调用了`BinderProxy`的`transact`方法将请求的`Parcel`类型对象`_data, _reply`发送给`binder`。查看`BinderProxy`的`transact`方法，可以看到如下代码：
```java
final boolean result = transactNative(code, data, reply, flags);
```
其调用了`transactNative`方法：
```java
/**  
 * Native implementation of transact() for proxies */
public native boolean transactNative(int code, Parcel data, Parcel reply,  
        int flags) throws RemoteException;
```
可以看到该方法为一个native方法
2. 给MyService的sayHello方法打上断点，观察其调用栈，可以看到：
![sayHello.png](../images/2025-6-19-binder_test/sayHello.png)
查看`Binder.execTransact`方法
```java
// Entry point from android_util_Binder.cpp's onTransact.  
@UnsupportedAppUsage  
private boolean execTransact(int code, long dataObj, long replyObj,  
        int flags) {  
  
    Parcel data = Parcel.obtain(dataObj);  
    Parcel reply = Parcel.obtain(replyObj);  
  
    final long origWorkSource = callingUid == -1  
            ? -1 : ThreadLocalWorkSource.setUid(callingUid);  
  
    try {  
        return execTransactInternal(code, data, reply, flags, callingUid);  
    } finally {  
        reply.recycle();  
        data.recycle();  
  
        if (callingUid != -1) {  
            ThreadLocalWorkSource.restore(origWorkSource);  
        }  
    }  
}
```
可以看到其对应C++ native `android_util_Binder`的`onTransact`方法，并调用了自己的`execTransactInternal`方法
```java
private boolean execTransactInternal(int code, Parcel data, Parcel reply, int flags,  
        int callingUid) {
    ...
	if ((flags & FLAG_COLLECT_NOTED_APP_OPS) != 0 && callingUid != -1) {  
	AppOpsManager.startNotedAppOpsCollection(callingUid);  
	try {  
		res = onTransact(code, data, reply, flags);  
	} finally {  
		AppOpsManager.finishNotedAppOpsCollection();  
	}  
	} else {  
	res = onTransact(code, data, reply, flags);  
	}
}
```
可以看到，其调用了`onTransact`方法，而`onTransact`方法由`HelloServie$Stub`实现：
```java
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException  
{  
 //...
  switch (code)  
  {  
    case TRANSACTION_sayHello:  
    {  
      java.lang.String _result = this.sayHello();  
      reply.writeNoException();  
      reply.writeString(_result);  
      break;  
    }  
    default:  
    {  
      return super.onTransact(code, data, reply, flags);  
    }  
  }  
  return true;  
}
```
对于创建的`HelloService`的`AIDL`，自动创建了一个`HelloService`的接口，并为其创建了`Stub`和`Stub.Proxy`两个内部类。可以看到收到`TRANSACTION_sayHello`的`code`后，调用其`sayHello`方法，并将结果写入`reply`中
```java
private final HelloService.Stub mBinder = new HelloService.Stub() {  
  
    @Override  
    public String sayHello() throws RemoteException {  
        return "hello";  
    }  
};
```
在`MyService`中重写了`Stub`的`sayHello`方法

### 当Service和Activity运行在一个进程中的调用流程

1. 删除`AndroidManifest.xml`中将`MyService`和`ClientActivity`设置到两个进程的设置
2. 调试`onServiceConnected`方法，可以看到，传入的`service`类型变为`HelloService$Stub`类型
![Stub](../images/2025-6-19-binder_test/service_instance_of_Stub.png)
根据`HelloService.Stub.asInterface()`方法，此时`queryLocalInterface`返回该`Stub`类型对象
```java
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);  
if (((iin!=null)&&(iin instanceof com.example.mybindertest.HelloService))) {  
  return ((com.example.mybindertest.HelloService)iin);  
}  
```
   于是，当调用`helloService.sayHello`方法时，实际上是直接调用了在`MyService`中创建的`Stub`类型对象`mbinder`中重写的`sayHello`方法


### 问题记录

| 问题                                                                                            | 原因                                              | 解决方法                                                        |
| --------------------------------------------------------------------------------------------- | ----------------------------------------------- | ----------------------------------------------------------- |
| Activity不能运行                                                                                  | 未指定layout                                       | 指定layout                                                    |
| IllegalArgumentException                                                                      | bindService时未显式使用intent                         | 使用`Intent(this,HelloService.class)`的方式显式使用intent            |
| MyService没有启动                                                                                 | 在intent中设置Service的class时设置为`HelloService.class` | 修改为`MyService.class`                                        |
| onServiceConnected(ComponentName name, IBinder service) 没有返回BinderProxy对象而是返回MyService.Stub对象 | `service`和`activity`运行在同一个进程中                   | 在AndroidManifest.xml中配置`android:process`选项，设置两者在不同的process中 |
