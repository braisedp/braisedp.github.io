---
layout: post
title: "Binder 内核驱动与架构"
date:   2025-6-17
tags: [Android, Binder]
comments: true
author: braisedp
toc : true
---

<!-- more -->

## Binder 内核数据结构

对于用户空间进程间方法调用（IPC），需要内核空间提供一种数据交换的机制，这个机制在内核空间中开辟一段缓冲区，当一个进程想要向另一个进程传输数据时，他将数据发送到这个节点上，而另一个进程也从这个节点上读取数据或者订阅改节点以接受buffer的消息通知。

![ipc](../images/2025-6-17-binder_driver/buffer.svg)

一个服务端可以提供多种不同的服务和不同的方法，一种做法是在传输的协议中附带上服务名+方法名，然后由服务端根据服务名+方法名处理方法调用，另一种方法是在内核中，对于每个service都创建一个对应的实例，在内核中，客户端的service引用与服务端的service实例成一一对应关系。

![ipc 2](../images/2025-6-17-binder_driver/services.svg)

Binder采取了第二种方式，对于每个服务端的服务，Binder都通过一个`binder_node`结构体对应，同样，在客户端，也对应一个`binder_ref`结构体。

IPC需要处理客户端与服务端之间的数据传输，在每一个时间段，多个客户端会调用不同的服务，这样，不同服务节点的状态也会不同，有的节点上会传输数据，有的节点则会等待，于是，需要通过多线程的方式处理这些并发请求。在Binder中，通过`binder_proc`描述一个服务端进程在内核空间中对应的一个binder进程，并通过`binder_thread`描述执行不同操作的binder线程。通过`binder_work`，Binder描述了不同的binder操作。对于IPC的提供者而言，一次IPC操作只有成功和失败两种状态，也就是说一次IPC操作是原子的，要么是成功的，要么其中其中的所有操作都不执行。对于IPC，通过事务化的方式管理IPC操作。在Binder中，通过`binder_transaction`描述一个binder事务。

![ipc proc](../images/2025-6-17-binder_driver/proc.svg)

现在考虑IPC提供给用户空间的api：
- 一方面，IPC需要提供一个协议，用于用户空间指定数据传输的一些细节，比如传输的数据的长度，传输的对象的类型，还需要提供对象的序列化方式。
- 另一方面，IPC需要提供一些接口方法，用于用户空间向内核空间读取/写入数据。

在第一点上，Binder通过binder object描述在buffer中传输的对象，有如下结构体：
```cpp
struct binder_object{
    union{
        struct binder_object_header hdr;
        struct flat_binder_object fbo;
        struct binder_fd_object fdo;
        struct binder_buffer_object bbo;
        struct binder_fd_array_object fdao;
    }
}
```
，该结构体描述了在内核buffer中的几种对象类型。其中：
```cpp
struct binder_object_header{
    __u32 type;
}
```
，为所有binder object都有的一个对象头，用于描述binder 对象的类型。
```cpp
struct 
```

## Binder 数据结构

| 数据结构 | 成员变量 |
| --- | --- |
| binder_ref | - rb_node_desc 按句柄值排序的红黑树节点<br>- rb_node_node 按binder实体地址排序的红黑树节点<br>- proc binder引用对象的宿主进程<br>- node binder实体<br>- desc 句柄值<br>- death 死亡接受通知|
| binder_ref_death   | - work  工作状态<br>- cookie（__user） 负责接收死亡通知的对象的地址|
| binder_node        | - work 工作状态<br>- rb_node/ dead_node 宿主维护的binder实体红黑树节点/ 全局宿主进程死亡的hash列表<br>- proc 宿主进程<br>- ptr（\_\_user）Service组件的引用计数对象(weakref_impl)<br>- cookie (\_\_user) Service组件地址<br>- has_async_transaction 是否有异步事务<br>- accept_fds 是否接受文件描述头<br>- min_priority 最低优先级<br>- async_todo 异步事务队列 |
| binder_buffer      | - entry 内核缓冲区节点<br>- rb_node 进程的空闲 entry 红黑树节点（大小）/ 已分配的buffer entry 红黑树节点（地址）<br>- free 是否空闲<br>- allow_user_free 是否允许用户空间释放buffer<br>- transaction 处理的事务<br>- target_node 处理buffer的binder<br>- data_size buffer中的数据大小<br>- offsets_size buffer中传输的binder的偏移量数组<br>- data 指向数据缓冲区 |
| binder_proc        | - proc_node 全局binder_proc散列表中的节点<br>- buffer_size/buffers 全部缓冲区的大小/缓冲区列表的头部指针<br>- free_buffers/ allocated_buffers/ free_async_space 空闲内存红黑树/ 分配内存红黑树/ 用来保存异步事务数据的内核缓冲区大小<br>- vma/ user_buffer_offset 其中保存buffer在用户空间中地址/ 保存用户空间地址与内核空间地址之间的差值<br>- pages struct pages\*的数组，每个元素指向物理页面<br>- threads/ max_threads/ ready_threads 以红黑树（线程ID）组织的Binder线程池/ Binder驱动程序最多可以主动请求进程注册的线程数量/ 进程当前的空闲Binder线程数量<br>- requested_threads/ requested_threads_started Binder正在主动请求注册的线程的数量/ Binder已经主动请求注册的线程数量<br>- todo/ wait 进程的待处理工作项队列/ 空闲Binder线程<br>- default_priority 默认优先级<br>- nodes/ refs_by_desc/ refs_by_node Binder实体的红黑树/ Binder引用句柄的红黑树/ Binder引用按binder实体组织的红黑树<br>- deferred_work_node 进程可以延迟执行的工作项的散列表（`BINDER_DEFERRED_PUT_FILES` 在进程不再需要Binder通信时，关闭相应的文件描述符并释放内核缓冲区时创建的工作项 \| `BINDER_DEFERRED_FLUSH` 唤醒wait中的线程创建的工作项 \| `BINDER_DEFERRED_RELEASE`  进程通过close关闭设备文件`/dev/binder` 时创建 ）<br>- delivered_death Service组件死亡通知 |
| binder_thread      | - proc 宿主进程<br>- rb_node proc中thread红黑树的节点<br>- pid/ looper 标识binder的状态<br>- todo 请求线程处理的client进程请求<br>- transaction_stack Binder交给线程的事务的栈（binder_transaction类型）<br>- wait 线程执行事务时等待其依赖的其他事务完成时所在的等待队列<br>- return_error/ return_error2 错误|
| binder_transaction | - from/ to_proc/ to_thread 发起事务的线程/ 处理事务的进程/ 处理事务的线程<br>- priority / sender_id/ saved_priority 源线程优先级/ 用户ID/ 保存被修改的线程优先级<br>- buffer 事务的内核缓冲区<br>- from_parent/ to_parent 事务依赖的另一个事务/ 目标线程中当前事务后需要处理的事务 |

## Binder与用户程序通信的数据结构

| 数据结构| 成员变量  | 作用 |
| --- | ---| --- |
| binder_write_read         | - write_size/write_consumed/write_buffer <br>输入数据的长度/ 驱动已读的长度/ 输入缓冲区的起始地址<br>- read_size/ read_consumed/ read_buffer <br>返回的数据长度/ 用户空间已读的长度/ 输出缓冲区的其实地址 | 通信所传输的数据|
| BinderDriverCommandProtocol | | 命令协议|
| BinderDriverReturnProtocol  | | 返回协议 |
| binder_transaction_data     | | 进程间传输的数据|
| flat_binder_object          | | 描述Binder实体，Binder引用对象或一个文件描述符 |

## Binder初始化

1. `binder_init` 在/proc下创建/binder/proc目录，并创建`state, stats, transaction_log, failed_transaction_log`五个文件，通过`misc_register`创建Binder设备
2. 指定binder的`file_operations`，定义binder设备`miscdevice`

## Binder文件操作

### 打开Binder设备文件

1. 创建`binder_proc`结构体`proc`，初始化`tsk, default_priority, pid`，并加入到`binder_procs`中
2. 将`proc`保存到filp的private_data中，用于`binder`驱动程序获取`proc`
3. 创建以PID为名的文件，用于获取进程对应的Binder信息

### 内存映射

1. 对`vma`指向的用户地址空间进行检查，包括其范围(`vma_end-vma_start`)截断为4M，是否可写(`vm_flags & FORBIDDEN_MMAP_FLAGS`需为0) ，并设置`VM_DONTCOPY`为1，`VM_MAYWRITE`位设置为0，判断`proc->buffer`是否已经分配
2. 调用`get_vm_area`在内核中分配空间，并将其地址与大小保存在`proc`中，指定其打开和关闭函数
3.  分配一个空间保存`proc->pages`指向的数组，分配一个物理页面给`proc->buffer`，（通过`binder_update_page_range`）创建链表`proc->buffers`并将`buffer`加入，设置`buffer`空闲并加入到空闲`buffer`红黑树中，设置给异步事务的空间大小为`buffer_size/2`

## 内核缓冲区管理

### 内核缓冲区分配 

发送`BC_TRANSACTION, BC_REPLY`后

`binder_alloc_buf` 
1. data_size, offsets_size 的合法性检验，异步事务缓冲区大小检验
2. 在红黑树中查找第一个大于等于需要的size的buffer
3. 对buffer进行裁剪（若剩余部分大于等于一个`struct binder_buffer`+4字节，则进行裁剪，否则不进行裁剪），并为其分配物理页面
4. 将buffer从空闲内核缓冲区删除，并加入到已分配物理页面的内核缓冲区红黑树中
5. 将裁剪后剩余的空间封装为另一个new_buffer并加入到内核缓冲区列表和空闲缓冲区红黑树中
6. 设置buffer的数据缓冲区和偏移数组缓冲区大小，并设置其是否用于异步事务

### 内核缓冲区释放

进程处理完`BR_TRANSACTION, BR_REPLY`后

`binder_free_buf`
1. 计算需要释放的buffer的大小
2. 释放物理页面，并从已分配物理页面的buffer红黑树中删除：
3. 合并前后两个空闲buffer，方法为删除buffer，将其所占用的空间追加到前一个空闲内核缓冲区中

### 内核缓冲区查询

`binder_buffer_lookup`

