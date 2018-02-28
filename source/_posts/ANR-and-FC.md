---
title: ANR与FC
date: 2016-08-23 21:42:08
category:
	- Android
	- Android开发工程师

tags:
	- Android开发工程师
	- Android
description: "ANR和FC发生的原因和处理方法"
---

## ANR(Application Not Responding)
### 引起ANR的直接原因
1. KeyDispatchTimeout(5 seconds) --主要类型按键或触摸事件在特定时间内无响应

2. BroadcastTimeout(10 seconds) --BroadcastReceiver在特定时间内无法处理完成

3. ServiceTimeout(20 seconds) --小概率类型 Service在特定的时间内无法处理完成

### 引起ANR的根本原因
1. UI线程中进行耗时操作，如网络操作，数据库操作，I/O
2. BroadcastReceiver中进行耗时操作

### 避免ANR的方法
1. UI线程不做耗时操作，将它们放到单独的线程中处理，并使用Handler来处理UIThread和耗时Thread之间的交互。耗时操作处理完成之后，通过handler.sendMessage()传递处理结果，在handler的handleMessage()方法中更新UI，或者使用handler.post()方法将消息放到Looper中


2. 使用AsyncTask，在doInBackground()方法中执行耗时操作，在onPostExecuted()更新UI
 
### 排查ANR的方法
1. 分析log
2. 从trace.txt文件查看调用stack，adb pull data/anr/traces.txt ./mytraces.txt
3. 看代码
4. 仔细查看ANR的成因(iowait?block?memoryleak?)

## FC（Force Close）
### 引起FC的原因
1. Error
2. OOM,内存溢出
3. StackOverFlowError
4. Runtime,比如NullPointExection，Class Not Found

### 避免弹出Force Close窗口的方法
实现Thread.UncaughtExceptionHandler接口的uncaughtException方法
```
import java.lang.Thread.UncaughtExceptionHandler;
import android.app.Application;
public class MyApplication extends Application implements UncaughtExceptionHandler {
@Override
public void onCreate() {
  // TODO Auto-generated method stub
  super.onCreate();
}

@Override
public void uncaughtException(Thread thread, Throwable ex) {
  thread.setDefaultUncaughtExceptionHandler( this);  
}

}
```

## OOM
OutOfMemoryError异常

除了程序计数器外，虚拟机内存的其他几个运行时区域都有发生OutOfMemoryError(OOM)异常的可能，

### Java Heap 溢出

一般的异常信息：java.lang.OutOfMemoryError:Java heap spacess

java堆用于存储对象实例，我们只要不断的创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象，就会在对象数量达到最大堆容量限制后产生内存溢出异常。

出现这种异常，一般手段是先通过内存映像分析工具(如Eclipse Memory Analyzer)对dump出来的堆转存快照进行分析，重点是确认内存中的对象是否是必要的，先分清是因为内存泄漏(Memory Leak)还是内存溢出(Memory Overflow)。

如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链。于是就能找到泄漏对象时通过怎样的路径与GC Roots相关联并导致垃圾收集器无法自动回收。

如果不存在泄漏，那就应该检查虚拟机的参数(-Xmx与-Xms)的设置是否适当。

### 虚拟机栈和本地方法栈溢出

如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常。

如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常

这里需要注意当栈的大小越大可分配的线程数就越少。

### 运行时常量池溢出

异常信息：java.lang.OutOfMemoryError:PermGen space

如果要向运行时常量池中添加内容，最简单的做法就是使用String.intern()这个Native方法。该方法的作用是：如果池中已经包含一个等于此String的字符串，则返回代表池中这个字符串的String对象；否则，将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。由于常量池分配在方法区内，我们可以通过-XX:PermSize和-XX:MaxPermSize限制方法区的大小，从而间接限制其中常量池的容量。

### 方法区溢出

方法区用于存放Class的相关信息，如类名、访问修饰符、常量池、字段描述、方法描述等。

异常信息：java.lang.OutOfMemoryError:PermGen space

方法区溢出也是一种常见的内存溢出异常，一个类如果要被垃圾收集器回收，判定条件是很苛刻的。在经常动态生成大量Class的应用中，要特别注意这点。
