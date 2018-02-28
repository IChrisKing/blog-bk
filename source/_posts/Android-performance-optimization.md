---
title: Android性能优化
date: 2016-08-23 19:12:15
category:
	- Android
	- Android开发工程师

tags:
	- Android开发工程师
	- Android
description: "Android性能优化方式，主要包括内存优化，编码优化，布局优化"
---
[原文地址](https://github.com/GeniusVJR/LearningNotes/blob/master/Part1/Android/Android%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96.md)
### 管理内存
#### IntentService 代替 Service
使用IntentService，当后台任务执行结束后会自动停止，避免了Service的内存泄漏

#### 界面不可见时释放内存
重写Activity的onTrimMemory()方法，然后在这个方法中监听TRIM_MEMORY_UI_HIDDEN这个级别，一旦触发说明用户离开了程序，此时就可以进行资源释放操作了。

#### 使用SparseArray系列
使用SparseArray，SparseBooleanArray、LongSparseArray等代替HashMap<Integer,Object>

HashMap工具类会相对比较低效，因为它需要为每一个键值对都提供一个对象入口，而SparseArray就避免掉了基本数据类型转换成对象数据类型的时间。

#### 谨慎使用多进程
多进程比较适用于哪些需要在后台去完成一项独立的任务，和前台是完全可以区分开的场景。比如音乐播放，关闭软件，已经完全由Service来控制音乐播放了，系统仍然会将许多UI方面的内存进行保留。在这种场景下就非常适合使用两个进程，一个用于UI展示，另一个用于在后台持续的播放音乐。关于实现多进程，只需要在Manifast文件的应用程序组件声明一个android:process属性就可以了。进程名可以自定义，但是之前要加个冒号，表示该进程是一个当前应用程序的私有进程。

### 分析内存使用情况
#### 获取手机堆大小
```
ActivityManager manager = (ActivityManager)getSystemService(Context.ACTIVITY_SERVICE);
int heapSize = manager.getMemoryClass();
```
结果以MB为单位进行返回，我们开发时应用程序的内存不能超过这个限制，否则会出现OOM。

#### GC操作及内存泄漏

### 编码优化
#### 换掉String
如果有需要拼接的字符串，那么可以优先考虑使用StringBuffer或者StringBuilder来进行拼接，而不是加号连接符

当一个方法的返回值是String的时候，通常需要去判断一下这个String的作用是什么，如果明确知道调用方会将返回的String再进行拼接操作的话，可以考虑返回一个StringBuffer对象来代替，因为这样可以将一个对象的引用进行返回，而返回String的话就是创建了一个短生命周期的临时对象。

#### 使用基本数据类型
在没有特殊原因的情况下，尽量使用基本数据类型来代替封装数据类型，int比Integer要更加有效，其它数据类型也是一样。

基本数据类型的数组也要优于对象数据类型的数组。另外两个平行的数组要比一个封装好的对象数组更加高效，举个例子，Foo[]和Bar[]这样的数组，使用起来要比Custom(Foo,Bar)[]这样的一个数组高效的多。

#### 静态代替抽象
如果并不需要访问一个类中的某些字段，只是想调用它的某些方法来去完成一项通用的功能，那么可以将这个方法设置成静态方法，调用速度提升15%-20%，同时也不用为了调用这个方法去专门创建对象了，也不用担心调用这个方法后是否会改变对象的状态(静态方法无法访问非静态字段)。

#### 对常量使用static final修饰符
这种优化方式只对基本数据类型以及String类型的常量有效，对于其他数据类型的常量是无效的。

#### 正确使用for循环
首选
```
public void for_example() {  
    int sum = 0;  
    for (Counter a : mArray) {  
        sum += a.mCount;  
    }  
} 
```
次选
```
public void for_example() {  
    int sum = 0;  
    Counter[] localArray = mArray;  
    int len = localArray.length;  
    for (int i = 0; i < len; ++i) {  
        sum += localArray[i].mCount;  
    }  
} 
```

#### 使用系统封装好的API
如：实现数组拷贝的功能，使用循环的方式来对数组中的每一个元素一一进行赋值当然可行，但是直接使用系统中提供的System.arraycopy()方法会让执行效率快9倍以上。

#### 避免在内部调用Getters/Setters方法

### 布局优化
#### 重用布局文件
标签可以允许在一个布局当中引入另一个布局，那么比如说我们程序的所有界面都有一个公共的部分，这个时候最好的做法就是将这个公共的部分提取到一个独立的布局中，然后每个界面的布局文件当中来引用这个公共的布局。

#### 需要时再加载布局
```
public void onMoreClick() {  
    ViewStub viewStub = (ViewStub) findViewById(R.id.view_stub);  
    if (viewStub != null) {  
        View inflatedView = viewStub.inflate();  
        editExtra1 = (EditText) inflatedView.findViewById(R.id.edit_extra1);  
        editExtra2 = (EditText) inflatedView.findViewById(R.id.edit_extra2);  
        editExtra3 = (EditText) inflatedView.findViewById(R.id.edit_extra3);  
    }  
}  
```
