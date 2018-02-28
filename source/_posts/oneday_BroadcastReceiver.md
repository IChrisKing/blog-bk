---
title: BroadcastReceiver
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "BroadcastReceiver的使用方法，生命周期，及其衍生出的LocalBroadcastManager类"
date: 2016-09-22 07:43:52
---
[用广播来更新UI界面好吗？](http://www.jianshu.com/p/df7af437e766)

## 简介
Android四大组件之一。
1. 广播接收器是一个专注于接收广播通知信息，并做出对应处理的组件。很多广播是源自于系统代码的──比如，通知时区改变、电池电量低、拍摄了一张照片或者用户改变了语言选项。应用程序也可以进行广播──比如说，通知其它应用程序一些数据下载完成并处于可用状态。
2. 应用程序可以拥有任意数量的广播接收器以对所有它感兴趣的通知信息予以响应。所有的接收器均继承自BroadcastReceiver基类。
3. 广播接收器没有用户界面。然而，它们可以启动一个activity来响应它们收到的信息，或者用NotificationManager来通知用户。通知可以用很多种方式来吸引用户的注意力──闪动背灯、震动、播放声音等等。一般来说是在状态栏上放一个持久的图标，用户可以打开它并获取消息。

Android中的广播事件有两种，一种就是系统广播事件，比如：ACTION_BOOT_COMPLETED（系统启动完成后触发），ACTION_TIME_CHANGED（系统时间改变时触发），ACTION_BATTERY_LOW（电量低时触发）等等。另外一种是我们自定义的广播事件。

## 源码位置
frameworks/base/core/java/android/content/BroadcastReceiver.java

BroadcastReceiver是一个抽象类，继承它的子类，需要实现它的抽象方法：`public abstract void onReceive(Context context, Intent intent);`

## 注册广播
### 两种方法
* 静态注册
1. 创建一个类继承BroadcastReceiver
```
public class MyReceiver extends BroadcastReceiver{
    private static final String MY_ACTION = "com.testBroadcastReceiver.myaction";
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        if(action.equal(MY_ACTION)) {
            Toast.makeText(context, "静态:"+action, 1000).show();
        }    
    }
}
```

2. 到AndroidManifest.xml 添加receive标签
```
<receiver android:name="MyReceiver">  
    <intent-filter>  
        <action  
            android:name="com.testBroadcastReceiver.myaction"/>  
    </intent-filter>  
</receiver> 
```

* 动态注册
```
IntentFilter intentFilter  = new IntentFilter();  
intentFilter .addAction(MY_ACTION); //指定action 
registerReceiver(myReceiver, intentFilter);  //注册监听
unregisterReceiver(myReceiver); //取消监听
```
**一般在onStart中注册，onStop中取消unregisterReceiver**

## 发送广播
发送广播消息，把要发送的信息和用于过滤的信息(如Action、Category)装入一个Intent对象，然后通过调用 Context.sendBroadcast()、sendOrderBroadcast()或sendStickyBroadcast()方法，把 Intent对象以广播方式发送出去。

```
Intent intent = new Intent(MY_ACTION);
intent.putExtra("Name", "abc");
intent.putExtra("Msg", "hello");
sendBroadcast(intent);//传递过去
```

## 广播分类
### Normal broadcasts（普通广播）
Normal broadcasts是完全异步的可以同一时间被所有的接收者接收到。消息的传递效率比较高。但缺点是接收者不能将接收的消息的处理信息传递给下一个接收者也不能停止消息的传播。

### Ordered broadcasts（有序广播）
Ordered broadcasts的接收者按照一定的优先级进行消息的接收。如：A,B,C的优先级依次降低，那么消息先传递给A，在传递给B，最后传递给C。优先级别声明在中，取值为[-1000,1000]数值越大优先级别越高。优先级也可通过filter.setPriority(10)方式设置。 另外Ordered broadcasts的接收者可以通过abortBroadcast()的方式取消广播的传播，也可以通过setResultData和setResultExtras方法将处理的结果存入到Broadcast中，传递给下一个接收者。然后，下一个接收者通过getResultData()和getResultExtras(true)接收高优先级的接收者存入的数据。

### StickyBroadcast （异步广播）
当处理完之后的Intent ，依然存在，这时候registerReceiver(BroadcastReceiver, IntentFilter) 还能收到他的值，直到你把它去掉 , 不能将处理结果传给下一个接收者 , 无法终止广播。

在发送广播时Reciever还没有被注册，但它注册后还是可以收到在它之前发送的那条广播

## 生命周期
Receiver的生命周期很短，当它的onReceive方法执行完成后，它的生命周期就结束了。这时BroadcastReceiver已经不处于active状态，被系统杀掉的概率极高，也就是说如果你在onReceive去开线程进行异步操作或者打开Dialog都有可能在没达到你要的结果时进程就被系统杀掉。因为这个进程可能只有这个Receiver这个组件在运行，当Receiver也执行完后就是一个empty进程，是最容易被系统杀掉的。替代的方案是用Notificaiton或者Service（这种情况当然不能用bindService，因为bindService启动的service会在启动者被销毁后跟着被销毁）。

## 题目解答
如果界面并不需要频繁更新，那么可以使用广播。但如果时频繁的刷新界面，则不适合使用广播。因为广播的发送和接收是有一定的延迟的，它的传输是通过Binder进程间通信机制来实现的，系统需要为广播的传递做一些进程间通信的准备。

ActivityManagerService有一个专门的消息队列来接收发送出来的广播，sendBroadcast执行完后就立即返回，但这时发送来的广播只是被放入到队列，并不一定马上被处理。当处理到当前广播时，又会把这个广播分发给注册的广播接收分发器ReceiverDispatcher，ReceiverDispatcher最后又把广播交给接Receiver所在的线程的消息队列去处理（就是你熟悉的UI线程的Message Queue）。

整个过程从发送--ActivityManagerService--ReceiverDispatcher进行了两次Binder进程间通信，最后还要交到UI的消息队列，如果基中有一个消息的处理阻塞了UI，当然也会延迟你的onReceive的执行。

## LocalBroadcastManager
### 简介
LocalBroadcastManager是Android Support包提供了一个工具，是用来在同一个应用内的不同组件间发送Broadcast的。

使用LocalBroadcastManager有如下好处：

* 发送的广播只会在自己App内传播，不会泄露给其他App，确保隐私数据不会泄露
* 其他App也无法向你的App发送该广播，不用担心其他App会来搞破坏
* 比系统全局广播更加高效

### 使用方法
1. 通过getInstance获取实例
```
LocalBroadcastManager lbm = LocalBroadcastManager.getInstance(this); ```
2. 通过函数 registerReceiver来注册监听器
```
lbm.registerReceiver(new BroadcastReceiver() {  
        @Override  
        public void onReceive(Context context, Intent intent) {  
        // TODO Handle the received local broadcast  
        }  
    }, new IntentFilter(LOCAL_ACTION)); 
```
3. 通过 sendBroadcast 函数来发送广播
```
lbm.sendBroadcast(new Intent(LOCAL_ACTION));  
```
