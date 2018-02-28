---
title: activity启动模式
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "详细分析Activity的四种启动模式，"
date: 2016-12-13 18:30:14
---
## 任务栈与启动模式
指定一个Activity的任务栈模式，也就是启动模式。
任务栈有以下四种：
* standard 标准模式，默认使用该模式
* singletop 单一顶部模式 （如果当前activity就在栈顶，就不新建）
* singleTask 单一任务栈，干掉头上的其他Activity，自己变成栈顶
* singleInstance 单一实例(单例)，任务栈里面只有自己

## 两种指定任务栈的方式
### manifest中指定
使用manifest里面的Activity的launchMode属性来制定启动模式
```
android:name=".MainActivity"
android:launchMode="standard"
```

### 代码中指定intent.addFlag
```
Intent intent  = new Intent();
intent.setClass(FirstActivity.this,SecondActivity.class);
// 通过Intent的addFlag指定
intent.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);
startActivity(intent);
```
### 对比
1. 代码中指定的优先级更高
2. manifest无法设定 FLAG_ACTIVITY_CLEAR_TOP 标识
3. 代码指定无法指定为 singleInstance启动模式

## 启动模式详解
### standard 标准模式 默认启动模式
每开启一个Activity，就会在栈顶添加一个Activity实例。多次间隔或者直接启动一个甲Activity会添加多个甲的示例，可重复添加。（间隔 ABA, 直接 ACC或者AAA）

### singletop 单一顶部模式
如果开启的Activity已经存在一个实例在任务栈的顶部（仅限于顶部）,再去开启这个Activity，任务栈不会创建新的Activity的实例了，而是复用已经存在的这个Activity，onNewIntent方法被调用；之前打开过，但不是位于栈顶，那么还是会产生新的实例入栈，不会回调onNewIntent方法。

当我们把一个Activity B 设置为singleTop，当我们打开B页面，会出现几种情况：

说明：当前A和C都是Standard，B是singleTop

1. 之前没打开过：
假设我们首先打开了A，此时任务栈S1里面只有A。然后打开singleTop的B，B入S1栈（谁打开它进入谁的栈）。此时S1的情况是BA，B为栈顶。

2. 之前打开过，且位于栈顶：
复用这个栈，不会有新的实例压入栈中。同时 onNewIntent 方法会被回调，我们可以利用这个方法获得页面传过来的消息或者其他操作。

3. 之前打开过，但是不是位于栈顶：
还是会产生新的实例入栈。

### singleTask 单一任务（整个任务栈只有一个实例）
如果开启的Activity已经存在一个实例在任务栈S1，当再次去开启这个Activity时
1. 位于栈顶则直接复用，回调onNewIntent方法
2. 位于栈中，也会复用，回调onNewIntent方法，但复用的同时会自己上方的全部Activity都干掉。

### singleInstance 单一实例，任务栈中只存在自己
当启动一个启动模式为singleInstance的Activity时（之前没启动过），这时系统将开辟出另外一个任务栈，用于存放这个Activity，而且这个新的任务栈只能存放自身这唯一一个Activity。

singleInstance页面作为前台任务打开自己，则复用，任务栈顺序无变化

singleInstance页面作为后台任务栈，则切换成为前台任务栈，无新实例产生，复用。

复用就会调用onNewIntent方法。

## FLAG
在代码中指定任务栈，其实就是在代码中执行intent.addFlag
在代码中指定flag可以改变activity的启动方式，且最后指定的Flag是activity最终任务栈的启动模式。
### Intent.FLAG_ACTIVITY_NEW_TASK
* intent.FLAG_ACTIVITY_NEW_TASK单独使用并无任何效果，需要和taskAffinity一起配合着使用。
* 当和taskAffinity一起配合着使用时：Intent.FLAG_ACTIVITY_NEW_TASK + android:taskAffinity 指定非程序包名的任务栈名，与清单文件里面指定 android:launchMode="singleInstance"效果一致 
* 这个Flag常用于在服务中启动Activity，因为服务没有Activity的任务栈，所以只能用一个新的任务栈来启动这个Activity创建实例。

### Intent.FLAG_ACTIVITY_SINGLE_TOP
与在manifest.xml中指定 android:launchMode="singleTop"效果一致

### Intent.FLAG_ACTIVITY_NEW_TASK
首先会查找是否存在和被启动的Activity具有相同的亲和性的任务栈（即taskAffinity，注意同一个应用程序中的activity的亲和性一样，所以下面的例子a会在同一个栈中），如果有，刚直接把这个栈整体移动到前台，并保持栈中的状态不变，即栈中的activity顺序不变，如果没有，则新建一个栈来存放被启动的activity

#### 例子a
Activity A和Activity B在同一个应用中.

Activity A启动开僻Task堆栈(堆栈状态: A), 在Activity A中启动Activity B, 启动Activity B的Intent的Flag设为FLAG_ACTIVITY_NEW_TASK, Activity B被压入Activity A所在堆栈(堆栈状态: AB).

原因是默认情况下同一个应用中的所有Activity拥有相同的关系(taskAffinity).

#### 例子b
 Activity A在名称为"TaskOne应用"的应用中, Activity B和Activity C在名称为"TaskTwo应用"的应用中.
 
在Launcher中单击"TaskOne应用"图标, Activity A启动开僻Task堆栈, 命名为TaskA(TaskA堆栈状态: A),在Activity A中启动Activity B, 启动Activity C的Intent的Flag设为FLAG_ACTIVITY_NEW_TASK,Android系统会为Activity B开僻一个新的Task, 命名为TaskB(TaskB堆栈状态: B), 长按Home键, 选择TaskOne, Activity A回到前台, 再次启动Activity B（两种情况1.从桌面启动；2.从Activity A启动，两种情况一样）, 这时TaskB回到前台, Activity B显示, 供用户使用, 即:包含FLAG_ACTIVITY_NEW_TASK的Intent启动Activity的Task正在运行, 则不会为该Activity创建新的Task,而是将原有的Task返回到前台显示.

#### 例子c
Activity A在名称为"TaskOne应用"的应用中, Activity B和Activity C在名称为"TaskTwo应用"的应用中.

在Launcher中单击"TaskOne应用"图标, Activity A启动开僻Task堆栈, 命名为TaskA(TaskA堆栈状态: A),在Activity A中启动Activity B,启动Activity B的Intent的Flag设为FLAG_ACTIVITY_NEW_TASK, Android系统会为Activity B开僻一个新的Task, 命名为TaskB(TaskB堆栈状态: B),  在Activity B中启动Activity C(TaskB的状态: BC) 长按Home键, 选择TaskOne, Activity A回到前台, 再次启动Activity B(从桌面或者ActivityA启动，也是一样的),这时TaskB回到前台, Activity C显示,供用户使用.说明了在此种情况下设置FLAG_ACTIVITY_NEW_TASK后，会先查找是不是有Activity B存在的栈，根据亲和性(taskAffinity)，如果有，刚直接把这个栈整体移动到前台，并保持栈中的状态不变，即栈中的顺序不变

### Intent.FLAG_ACTIVITY_CLEAR_TOP
* 如果activity在栈中，则activity启动时，同一个任务栈里面位于它上方的activity会全部被清空。


	例子:Activity A启动开僻Task堆栈(堆栈状态: A), 在Activity A中启动Activity B(堆栈状态: AB), 在Activity B中启动Activity C(堆栈状态: ABC), 在Activity C中启动Activity D(堆栈状态: ABCD), 在Activity D中启动Activity B,启动Activity B的Intent的Flag设置为FLAG_ACTIVITY_CLEAR_TOP, (堆栈状态: AB).
	

* 如果activity在栈顶，再次打开自己（并且也设置为Intent.FLAG_ACTIVITY_CLEAR_TOP）时，会摧毁旧的，重建新的。虽然从任务栈的内容看来，没有变化，但其实对于activity的生命周期来说，已经经历了摧毁重建的过程。

* 这个标记位通常是和FLAG_ACTIVITY_NEW_TASK配合着使用的。在这种情况下被启动的Activty如果已经存在，那么系统就会调用它的onNewIntent方法。如果被调用的Activity使用默认的standrad模式，那么它连同它之上的Activity都要出栈，系统会创建新的实例并入栈置为栈顶。

### Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS
具有这个标记的Activity不会出现在Activity的历史列表中.
他等同于在xml里面设定 
android:excludeFromRecents="true" 





http://www.cnblogs.com/xiaoQLu/archive/2012/07/17/2595294.html