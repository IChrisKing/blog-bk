---
title: Binder
category:
  - Android
  - Android开发工程师
tags:
  - Android开发工程师
  - Android
date: 2016-08-31 21:33:11
description: "主要分析了Service Manager成为binder守护进程的过程，简要分析了Server和Client获取Service Manager的过程。分析仅到framework层，kernel层的代码只讲解原理，给出源码位置。同时，提供了一些binder的参考文件"
---

## Binder
### 框架分析
Binder是Android进程间通信的方式之一。采用Server，Client通信方式。

![image](/assets/img/Binder/binder.png)
Binder的架构，分为四块：
1. BinderDriver 位于内核空间，源码在/kernel/drivers/staging/android/目录下，Binder驱动提供设备文件/dev/binder与用户空间交互
2. Service Manager 位于用户空间，源码在/frameworks/native/cmds/servicemanager/目录下。Service Manager是一个守护进程，用来管理Server，并向Client提供查询Server接口的功能。
3. Server
4. Client

### Service Manager成为Binder守护进程
* Service Manager是整个Binder机制的守护进程，负责管理系统中的所有Server，并向Client提供查询Server远程接口的功能。

* Service Manager与Server和Client通信，也属于进程间通信，也采用的Binder机制。所以说，Service Manager在充当Binder机制的守护进程的同时，也是一个Server。他是一个特殊的Server。

Service Manager的入口位于/frameworks/native/cmds/servicemanager/service_manager.c中的main函数
```
int main(int argc, char **argv)
{
    struct binder_state *bs;

//第一步
    bs = binder_open(128*1024);
    if (!bs) {
        ALOGE("failed to open binder driver\n");
        return -1;
    }

//第二步
    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

//一些selinux相关操作
    selinux_enabled = is_selinux_enabled();
    sehandle = selinux_android_service_context_handle();
    selinux_status_open(true);

    if (selinux_enabled > 0) {
        if (sehandle == NULL) {
            ALOGE("SELinux: Failed to acquire sehandle. Aborting.\n");
            abort();
        }

        if (getcon(&service_manager_context) != 0) {
            ALOGE("SELinux: Failed to acquire service_manager context. Aborting.\n");
            abort();
        }
    }

    union selinux_callback cb;
    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

//第三步
    binder_loop(bs, svcmgr_handler);

    return 0;
}
```
main函数主要使用三个函数，完成三个步骤：
#### 1.打开Binder设备文件
`bs = binder_open(128*1024);`返回一个binder_state类型的bs
binder_open的实现在/frameworks/native/cmds/servicemanager/binder.c中。查看这个函数之前，首先来看一下结构体binder_state，
```
struct binder_state
{
    int fd;
    void *mapped;
    size_t mapsize;
};
```
fd是文件描述符，即表示打开的/dev/binder设备文件描述符；

mapped是把设备文件/dev/binder映射到进程空间的起始地址；

mapsize是上述内存映射空间的大小。

查看函数的binder_open实现
```
struct binder_state *binder_open(size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }

    bs->fd = open("/dev/binder", O_RDWR);
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open device (%s)\n",
                strerror(errno));
        goto fail_open;
    }

    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        fprintf(stderr,
                "binder: kernel driver version (%d) differs from user space version (%d)\n",
                vers.protocol_version, BINDER_CURRENT_PROTOCOL_VERSION);
        goto fail_open;
    }

    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```
这个函数的内容，围绕着给binder_state *bs赋值展开，根据上面binder_state结构体的内容，这个函数的主要操作包括：
1.`bs->fd = open("/dev/binder", O_RDWR);`

这就调用到了kernel层的Binder驱动了，这里暂时不对驱动层的代码作详细分析，仅指出原理和源码位置。具体分析可以参考[参考文档](http://blog.csdn.net/luoshengyang/article/details/6621566)

源码位置在/kernel/drivers/staging/android/binder.c
中的static int binder_open(struct inode *nodp, struct file *filp)函数。
	
这个函数的主要作用是创建一个struct binder_proc数据结构来保存打开设备文件/dev/binder的进程的上下文信息，并且将这个进程上下文信息保存在打开文件结构struct file的私有数据成员变量private_data中，这样，在执行其它文件操作时，就通过打开文件结构struct file来取回这个进程上下文信息了。这个进程上下文信息同时还会保存在一个全局哈希表binder_procs中，驱动程序内部使用。

2.`bs->mapsize = mapsize;`
设置内存映射空间的大小

3.`bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);`

对打开的设备文件进行内存映射操作mmap，这调用到了/kernel/drivers/staging/android/binder.c
中的static int binder_mmap(struct file *filp, struct vm_area_struct *vma)函数

这个函数将一个物理地址，同时映射到进程虚拟地址空间和内核虚拟地址空间。这就是Binder进程间通信机制的精髓所在了，同一个物理页面，一方映射到进程虚拟地址空间，一方面映射到内核虚拟地址空间，这样，进程和内核之间就可以减少一次内存拷贝了，提到了进程间通信效率。举个例子如，Client要将一块内存数据传递给Server，一般的做法是，Client将这块数据从它的进程空间拷贝到内核空间中，然后内核再将这个数据从内核空间拷贝到Server的进程空间，这样，Server就可以访问这个数据了。但是在这种方法中，执行了两次内存拷贝操作，而采用我们上面提到的方法，只需要把Client进程空间的数据拷贝一次到内核空间，然后Server与内核共享这个数据就可以了，整个过程只需要执行一次内存拷贝，提高了效率。

#### 2.向Binder驱动声明自己是Binder上下文管理者，即Binder守护进程
`binder_become_context_manager(bs)`

查看/frameworks/native/cmds/servicemanager/binder.c，发现这个函数的实现非常简单
```
int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```
这里通过调用ioctl文件操作函数来通知Binder驱动程序自己是守护进程，命令号是BINDER_SET_CONTEXT_MGR，没有参数。再次到驱动程序中来查看实现过程，调用到了/kernel/drivers/staging/android/binder.c
中的`static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)`函数，只需查看switch中的BINDER_SET_CONTEXT_MG代码段即可。
这段代码的实现类似一种先到先得的机制，将第一个调用到此的线程设置为Binder机制的守护进程，并为他创建Binder实体。


#### 3. 进入一个无穷循环，充当Server的角色，等待Client请求
`binder_loop(bs, svcmgr_handler);`
该函数定义在/frameworks/native/cmds/servicemanager/binder.c中
```
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }

        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
```
首先是通过binder_write函数执行BC_ENTER_LOOPER命令告诉Binder驱动程序， Service Manager要进入循环了
```
int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;

    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    if (res < 0) {
        fprintf(stderr,"binder_write: ioctl failed (%s)\n",
                strerror(errno));
    }
    return res;
}
```
熟悉的ioctl，又进入kernel层的驱动程序的，这次只看BINDER_WRITE_READ代码块

随后进入 for (;;)循环，再次调用了ioctl的BINDER_WRITE_READ代码块

### Server和Client获得Service Manager接口
Service Manager在Binder机制中既充当守护进程的角色，同时它也充当着Server角色，然而它又与一般的Server不一样。对于普通的Server来说，Client如果想要获得Server的远程接口，那么必须通过Service Manager远程接口提供的getService接口来获得，这本身就是一个使用Binder机制来进行进程间通信的过程。而对于Service Manager这个Server来说，Client如果想要获得Service Manager远程接口，却不必通过进程间通信机制来获得，因为Service Manager远程接口是一个特殊的Binder引用，它的引用句柄一定是0。

 获取Service Manager远程接口的函数是defaultServiceManager，这个函数在frameworks/native/libs/binder/IServiceManager.cpp文件中：
 ```
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
    
    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
            if (gDefaultServiceManager == NULL)
                sleep(1);
        }
    }
    
    return gDefaultServiceManager;
}
```
 gDefaultServiceManagerLock和gDefaultServiceManager是全局变量，定义在frameworks/native/libs/binder/Static.cpp文件中：
```
Mutex gDefaultServiceManagerLock;
sp<IServiceManager> gDefaultServiceManager;
```
gDefaultServiceManager是单例模式，调用defaultServiceManager函数时，如果gDefaultServiceManager已经创建，则直接返回，否则通过`interface_cast<IServiceManager>(ProcessState::self()->getContextObject(NULL))`来创建一个，并保存在gDefaultServiceManager全局变量中。本质上是创建了一个BpServiceManager，包含了一个句柄值为0的Binder引用。
具体创建过程分析参照
[http://blog.csdn.net/luoshengyang/article/details/6627260](http://blog.csdn.net/luoshengyang/article/details/6627260)
[http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)

这两篇文章详细的分析了创建过程。

获取到这个Service Manager之后，Server可以调用IServiceManager::addService这个接口来和Binder驱动程序交互了，即调用BpServiceManager::addService 。而BpServiceManager::addService又会调用通过其基类BpRefBase的成员函数remote获得原先创建的BpBinder实例，接着调用BpBinder::transact成员函数。在BpBinder::transact函数中，又会调用IPCThreadState::transact成员函数，这里就是最终与Binder驱动程序交互的地方了。IPCThreadState有一个PorcessState类型的成中变量mProcess，而mProcess有一个成员变量mDriverFD，它是设备文件/dev/binder的打开文件描述符，因此，IPCThreadState就相当于间接在拥有了设备文件/dev/binder的打开文件描述符，于是，便可以与Binder驱动程序交互了。

Client获得Service Manager之后，调用IServiceManager::getService这个接口来和Binder驱动程序交互了。

## 参考文章
[Binder机制](https://github.com/GeniusVJR/LearningNotes/blob/master/Part1/Android/Binder%E6%9C%BA%E5%88%B6.md)

[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363/)

[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)

[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)

[ Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)

[Android系统进程间通信（IPC）机制Binder中的Client获得Server远程接口过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6633311)

## 其他讲解Binder的文章
[Binder牌胶水,在Android中无处不在](http://www.open-open.com/lib/view/open1464181227898.html)

[Binder学习指南](http://www.open-open.com/lib/view/open1452649194089.html)
