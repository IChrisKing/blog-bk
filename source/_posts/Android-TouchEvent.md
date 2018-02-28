---
title: Android事件分发机制
category:
  - Android
  - Android开发工程师
tags:
  - Android开发工程师
  - Android
date: 2016-08-24 20:31:03
description: "Android Touch事件的分发过程，从Activity到ViewGroup到View"
---

[原文链接](http://www.jianshu.com/p/e99b5e8bd67b)
## ACTION_DOWN的传递过程
### ACTION_DOWN的传递图
![image](/assets/img/Android-TouchEvent/touch_event_1.png)

这张图中的true，false，super是指函数的返回值。

传递过程经过三个层次：activity -> viewgroup -> view

对于每一层的分发函数dispatchTouchEvent和响应函数onTouchEvent来说：
1. 返回true：消费掉该事件，不再向下一层传递。
2. 返回false：开始回溯，把它递交给上一层的onTouchEvent
3. 返回super：按照默认的流程传递事件

对于onInterceptTouchEvent来说：

它本身是一个拦截装置，相当于给viewgroup一个跳过下层viewgroup和view，直接将事件递交给本层onTouchEvent的机会。
1. 返回true：递交给本层的onTouchEvent
2. 返回false：不做拦截，按照默认的流程传递事件
3. 返回super：按照默认的流程传递事件

### 默认的传递过程图
![image](/assets/img/Android-TouchEvent/touch_event_super.png)

所有函数的默认实现就是返回super，也就是按照默认的流程，将事件依次传递给所有相关的函数。这个传递过程是一个U型的方向。

### onInterceptTouchEvent方法
![image](/assets/img/Android-TouchEvent/touch_event_onInterceptTouchEvent.png)
这个方法是viewgroup独有的。每个viewgroup进行事件分发的时候，会先调用onInterceptTouchEvent方法，问一问拦截器，是否需要将这个事件拦截下来，递交给本层的onTouchEvent来处理。如果要自己处理那就在onInterceptTouchEvent方法中 return true就会交给本层的onTouchEvent的处理，如果不拦截就是继续往下传。默认是不会去拦截的，因为子View也需要这个事件，所以onInterceptTouchEvent拦截器return super.onInterceptTouchEvent()和return false是一样的，是不会拦截的，事件会继续往子View的dispatchTouchEvent传递。

## ACTION_MOVE和ACTION_UP的传递过程

ACTION_MOVE和ACTION_UP的传递过程，和ACTION_DOWM事件被消费掉的函数所处的层有关。

下列图中，红色箭头代表

红色的箭头代表ACTION_DOWN 事件的流向

蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向
![image](/assets/img/Android-TouchEvent/move-up-1.png)
![image](/assets/img/Android-TouchEvent/move-up-2.png)
![image](/assets/img/Android-TouchEvent/move-up-3.png)
![image](/assets/img/Android-TouchEvent/move-up-4.png)
![image](/assets/img/Android-TouchEvent/move-up-5.png)
![image](/assets/img/Android-TouchEvent/move-up-6.png)
![image](/assets/img/Android-TouchEvent/move-up-7.png)
![image](/assets/img/Android-TouchEvent/move-up-8.png)
![image](/assets/img/Android-TouchEvent/move-up-9.png)
![image](/assets/img/Android-TouchEvent/move-up-10.png)
![image](/assets/img/Android-TouchEvent/move-up-11.png)
