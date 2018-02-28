---
title: Android基础知识
date: 2016-08-18 18:16:31
category:
	- Android
	- Android开发工程师
tags:
	- Android开发工程师
	- Android
description: "零碎的Android基础知识点"
---
[链接地址](https://github.com/GeniusVJR/LearningNotes/blob/master/Part1/Android/Android%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86.md "地址")

### 五种界面布局

** **FrameLayout 、 LinearLayout 、 AbsoluteLayout 、 RelativeLayout 、 TableLayout

### Activity生命周期

### Activity launchMode

`standard|singleInstance|singleTask|singleTop`

### 使用onSaveInstanceState\(Bundle outState\)和onRestoreInstanceState \(Bundle outState\)缓存activiry

### Fragment+Activity生命周期

创建时

Fragment先做一些准备工作，包括onAttach,onCreate,onCreateVIew,onViewCreated

Activity开始onCreate,onStart,onResume,Fragment跟随

销毁时

Fragment先onPause,onStop,onDestory,Activity跟随

![](https://raw.githubusercontent.com/GeniusVJR/LearningNotes/master/Part1/Android/FlowchartDiagram.jpg)

### Intent传递数据

### Service两种启动方式

bindService    startService

### 广播的静态注册和动态注册

### ContentProvider使用方法

### Service保活

### Context区别

Context的数量等于Activity的个数 + Service的个数 + 1，这个1为Application

### IntentService

## 新增
### 图片三级缓存，优化多图应用
适用与有大量从网上下载图片在本地展示的应用
#### 三级缓存
1. 内存缓存，优先加载，速度最快

内存缓存需要小心内存溢出，所以要避免使用HashMap<String,Bitmap>这种键值对的方式，无论使用强引用，软引用还是弱引用，都容易引起内存溢出。

官方推荐使用LruCache<String,Bitmap>，内存控制在一定的大小内, 超出最大值时会自动回收, 这个最大值开发者自己定

2. 本地缓存，次优先加载，速度较快，图片存在本地存储设备中
3. 网络获取，最后选择，速度慢，使用流量

#### 两个例子
[线程池管理下载任务+LruCache缓存+本地缓存+GridView滑动取消下载，静止进行下载](http://blog.csdn.net/xiaanming/article/details/9825113)

[照片墙](http://blog.csdn.net/guolin_blog/article/details/9526203#comments)
