---
title: Android内存泄漏
date: 2016-08-22 23:31:26
category:
	- Android
	- Android开发工程师

tags:
	- Android开发工程师
	- Android
description: "java内存分配策略，内存管理方式，内存泄漏的情景和一些防止内存泄漏的方式"
---

[原文地址](https://github.com/GeniusVJR/LearningNotes/blob/master/Part1/Android/Android%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E6%80%BB%E7%BB%93.md)
### java内存分配策略
* 静态存储区（方法区）：主要存放静态数据、全局 static 数据和常量。这块内存在程序编译时就已经分配好，并且在程序整个运行期间都存在。

* 栈区 ：当方法被执行时，方法体内的局部变量（其中包括基础数据类型、对象的引用）都在栈上创建，并在方法执行结束时这些局部变量所持有的内存将会自动被释放。因为栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。

* 堆区 ： 又称动态内存分配，通常就是指在程序运行时直接 new 出来的内存，也就是对象的实例。这部分内存在不使用时将会由 Java 垃圾回收器来负责回收。

### java内存管理
有向图，GC清理不可达的对象
java回收机制：判断一个内存空间是否符合垃圾收集标准有两个：一个是给对象赋予了空值null，以下再没有调用过，另一个是给对象赋予了新值，这样重新分配了内存空间

### java内存泄漏
无用的可达对象持续占有内存或无用对象的内存得不到及时释放

长生命周期的对象持有短生命周期对象的引用就很可能发生内存泄漏，尽管短生命周期对象已经不再需要，但是因为长生命周期持有它的引用而导致不能被回收

#### 主要场景
##### 静态集合类引起内存泄漏
静态变量的生命周期和应用程序一致，他们所引用的所有的对象Object也不能被释放
```
Static Vector v = new Vector(10);
for (int i = 1; i<100; i++)
{
Object o = new Object();
v.add(o);
o = null;
}
```
在这个例子中，循环申请Object 对象，并将所申请的对象放入一个Vector 中，如果仅仅释放引用本身（o=null），那么Vector 仍然引用该对象，所以这个对象对GC 来说是不可回收的。因此，如果对象加入到Vector 后，还必须从Vector 中删除，最简单的方法就是将Vector对象设置为null。

##### Set的remove方法失效
集合里面的对象属性被修改后，再调用remove()方法时不起作用

##### 监听器
通常一个应用当中会用到很多监听器，我们会调用一个控件的诸如addXXXListener()等方法来增加监听器，但往往在释放对象的时候却没有记住去删除这些监听器，从而增加了内存泄漏的机会。

##### 连接
比如数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接，除非其显式的调用了其close（）方法将其连接关闭，否则是不会自动被GC 回收的。对于Resultset 和Statement 对象可以不进行显式回收，但Connection 一定要显式回收，因为Connection 在任何时候都无法自动回收，而Connection一旦回收，Resultset 和Statement 对象就会立即为NULL。但是如果使用连接池，情况就不一样了，除了要显式地关闭连接，还必须显式地关闭Resultset Statement 对象（关闭其中一个，另外一个也会关闭），否则就会造成大量的Statement 对象无法释放，从而引起内存泄漏。这种情况下一般都会在try里面去的连接，在finally里面释放连接。
[Android下常见的内存泄露 经典](http://blog.csdn.net/bill_ming/article/details/7814841)

##### 非静态内部类
非静态内部类默认会持有外部类的引用，而如果该非静态内部类又创建了一个静态的实例，该实例的生命周期和应用的一样长。

##### 匿名内部类
android开发经常会继承实现Activity/Fragment/View，如果使用了匿名类，并被异步线程持有了，此线程和此Acitivity生命周期不一致的时候，就造成了Activity的泄露。

##### 外部模块的引用
例如程序员A 负责A 模块，调用了B 模块的一个方法如： public void registerMsg(Object b); 这种调用就要非常小心了，传入了一个对象，很可能模块B就保持了对该对象的引用，这时候就需要注意模块B 是否提供相应的操作去除引用。

##### 单例模式
单例对象在初始化后将在JVM的整个生命周期中存在（以静态变量的方式），如果单例对象持有外部的引用，那么这个对象将不能被JVM正常回收，导致内存泄漏

##### handler
[Handler内存泄漏分析及解决](https://github.com/GeniusVJR/LearningNotes/blob/master/Part1/Android/Handler%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E5%88%86%E6%9E%90%E5%8F%8A%E8%A7%A3%E5%86%B3.md)

Handler 属于 TLS(Thread Local Storage) 变量, 生命周期和 Activity 是不一致的。因此这种实现方式一般很难保证跟 View 或者 Activity 的生命周期保持一致，故很容易导致无法正确释放。

修复方法：在 Activity 中避免使用非静态内部类，比如将 Handler 声明为静态的，则其存活期跟 Activity 的生命周期就无关了。同时通过弱引用的方式引入 Activity，避免直接将 Activity 作为 context 传进去

在 Activity 的 Destroy 时或者 Stop 时应该移除消息队列 MessageQueue 中的消息。

下面几个方法都可以移除 Message：
```
public final void removeCallbacks(Runnable r);

public final void removeCallbacks(Runnable r, Object token);

public final void removeCallbacksAndMessages(Object token);

public final void removeMessages(int what);

public final void removeMessages(int what, Object object);
```

##### 资源未关闭
对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销，否则这些资源将不会被回收，造成内存泄漏。

Bitmap 没调用 recycle()方法，对于 Bitmap 对象在不使用时,我们应该先调用 recycle() 释放内存，然后才它设置为 null. 因为加载 Bitmap 对象的内存空间，一部分是 java 的，一部分 C 的（因为 Bitmap 分配的底层是通过 JNI 调用的 )。 而这个 recyle() 就是针对 C 部分的内存释放。 构造 Adapter 时，没有使用缓存的 convertView ,每次都在创建新的 converView。这里推荐使用 ViewHolder。

##### 避免使用static成员变量
如果成员变量被声明为 static，其生命周期将与整个app进程生命周期一样。不要在类初始时初始化静态成员。可以考虑lazy初始化。 

##### 避免 override finalize()
1. finalize 方法被执行的时间不确定，不能依赖与它来释放紧缺的资源。时间不确定的原因是： 虚拟机调用GC的时间不确定 Finalize daemon线程被调度到的时间不确定

2. finalize 方法只会被执行一次，即使对象被复活，如果已经执行过了 finalize 方法，再次被 GC 时也不会再执行了，原因是：
含有 finalize 方法的 object 是在 new 的时候由虚拟机生成了一个 finalize reference 在来引用到该Object的，而在 finalize 方法执行的时候，该 object 所对应的 finalize Reference 会被释放掉，即使在这个时候把该 object 复活(即用强引用引用住该 object )，再第二次被 GC 的时候由于没有了 finalize reference 与之对应，所以 finalize 方法不会再执行。

3. 含有Finalize方法的object需要至少经过两轮GC才有可能被释放。


#### 使用context的正确姿势
![](https://camo.githubusercontent.com/dee4aecb8a80c4e73337b56ee01cbffa2a8049dd/687474703a2f2f696d672e626c6f672e6373646e2e6e65742f32303135313132333134343232363334393f73706d3d353137362e3130303233392e626c6f67636f6e742e392e437455316334)
对 Activity 等组件的引用应该控制在 Activity 的生命周期之内； 如果不能就考虑使用 getApplicationContext 或者 getApplication，以避免 Activity 被外部长生命周期的对象引用而泄露。

#### java对象的几种引用类型
![](https://camo.githubusercontent.com/068950c506eddc68677d6a84b5d7ffe1c03c6cf9/68747470733a2f2f67772e616c6963646e2e636f6d2f7470732f5442315536544e4c565858585863685846585858585858585858582d3634342d3534362e6a7067)
如果只是想避免OutOfMemory异常的发生，则可以使用软引用。如果对于应用的性能更在意，想尽快回收一些占用内存比较大的对象，则可以使用弱引用。

另外可以根据对象是否经常使用来判断选择软引用还是弱引用。如果该对象可能会经常使用的，就尽量用软引用。如果该对象不被使用的可能性更大些，就可以用弱引用。