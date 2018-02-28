---
title: Android中的设计模式之代理模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。Android中的系统service都是采用代理模式来向上层提供服务的。本文以ActivityManager为例，讲解代理模式在Android中的应用。"
date: 2016-12-21 19:15:06
---
## 简介
代理模式是对象的结构模式。代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。

进行交互的时候，不和原本的对象直接交互，而是通过代理的方式，用代理来代替真正的对象进行交互，这样做的好处是降低了耦合性。

![image](/assets/img/pattern-proxy/proxy.png)

　　在代理模式中的角色：

　　●　　抽象对象角色：声明了目标对象和代理对象的共同接口，这样一来在任何可以使用目标对象的地方都可以使用代理对象。

　　●　　目标对象角色：定义了代理对象所代表的目标对象。

　　●　　代理对象角色：代理对象内部含有目标对象的引用，从而可以在任何时候操作目标对象；代理对象提供一个与目标对象相同的接口，以便可以在任何时候替代目标对象。代理对象通常在客户端调用传递给目标对象之前或之后，执行某个操作，而不是单纯地将调用传递给目标对象。

[示例代码](http://www.cnblogs.com/java-my-life/archive/2012/04/23/2466712.html)

## 代理模式的分类
[示例代码](http://m.blog.csdn.net/article/details?id=53645189)
### 普通代理
创建一个和真实对象相同的类，将真实的对象当作参数传入到代理中，当外界通过代理进行操作的时候，代理中的真实对象也会进行相应的操作。

操作：

1. 创建接口，定义共同方法。
2. 创建真实类，实现接口
3. 创建代理类实现接口，但是增加一步，在构造方法中将真实对象的引用当作参数，拿到真实对象。
4. 得到代理对象，通过对代理对象操作，实现真实对象的相应操作

### 虚拟代理
也可以叫做延时代理，他的思想就是在真实对象数据量非常大的时候通过代理先代替真实对象，当真正操作真实对象的时候在加载真实对象，这样做能够在很大程度上提高效率。

### 强制代理
强制代理是反其道而行之的代理模式,一般情况下代理模式都是通过代理来找到真实的对象,而强制代理则是通过真实对象才能找到代理也就是说由真实对象指定代理,当然最终访问还是通过代理模式访问的.

### 动态代理
动态代理是针对当需要的代理非常多的情况下，如果使用普通代理的话，难免需要创建多个相同的代理类，因为普通代理都必须创建一个和真实类相同的类，所以动态代理使用动态的方式减少了大量冗余类的出现，给代理增添了扩展性。

## Android中的代理模式
Android中的各种service大多是通过代理来向应用层提供服务的。

以ActivityManager为例
![image](/assets/img/pattern-proxy/ActivityManager.jpg)

IActivityManager(frameworks/base/core/java/android/app/ActivityManagerNative.java)作为ActivityManagerProxy和ActivityManagerNative的公共接口，所以两个类具有部分相同的接口，可以实现合理的代理模式；

ActivityManagerProxy代理类是ActivityManagerNative的内部类；

查看代码：
frameworks/base/core/java/android/app/ActivityManager.java
```
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
...
}

class ActivityManagerProxy implements IActivityManager
{
...
}
```
上面的IActivityManager相当于代理模式中的抽象对象，ActivityManagerProxy相当于代理对象。而目标对象则是ActivityManagerNative。

然而ActivityManagerNative是个抽象类，真正发挥作用的是它的子类ActivityManagerService（系统Service组件）。ActivityManagerService是真正意义上的目标对象。所以可以说ActivityManagerService实现了IActivityManager的所有功能，比如启动Activity，绑定服务，注册广播等等，这些逻辑最终都是交给ActivityManagerService实现的。

从图中可以看出代理类：使用ActivityManagerProxy代理类，来代理ActivityManagerNative类的子类ActivityManagerService；

ActivityManagerService是系统统一的Service，运行在独立的进程中；通过系统ServiceManger获取；

ActivityManager运行在一个进程里面，ActivityManagerService运行在另一个进程内，

对象在不同的进程里面，其地址是相互独立的；实现跨进程的对象访问，需要对应进程间通信的规则，

此处是采用Binder机制实现跨进程通信；所以此处的Proxy模式的运用属于：远程代理（RemoteProxy）。

### 代理实现过程
这里涉及到两个过程：
1. 代理对象建立：ActivityManagerProxy代理对象的创建；

2.程序执行过程：如何通过代理对象来执行真正对象请求；

#### 代理对象建立
是在ActivityManager的执行时就需要代理类来执行；
```
    public List<RunningServiceInfo> getRunningServices(int maxNum)
            throws SecurityException {
        try {
            return ActivityManagerNative.getDefault()
                    .getServices(maxNum, 0);
        } catch (RemoteException e) {
            // System dead, we will be dead too soon!
            return null;
        }
    }
```

继续查看ActivityManagerNative.getDefault(),实际上是关乎到Singleton<IActivityManager>类型的gDefault对象创建；
```
    static public IActivityManager getDefault() {
        return gDefault.get();
    }
    
    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
```
ServiceManager.getService("activity");获取系统的“activity”的Service,所有的Service都是注册到ServiceManager进行统一管理。

这样就创建了一个对ActivityManagerService实例的本地代理对象ActivityManagerProxy实例。Singleton是通用的单例模板类。

ActivityManagerNative.getDefault就返回一个此代理对象的公共接口IActivityManager类型，就可以在本地调用远程对象的操作方法。

#### 执行过程
执行过程就涉及到ActivityManager框架的执行流程

![image](/assets/img/pattern-proxy/ActivityManagerService.jpg)

此图表明整个Client对Service的访问是通过Service的代理对象Proxy进行访问的。

Android中对Service访问的模式都是以Client/Server模式进行；

Client实际上访问Service是通过对Service的建立代理的Proxy对象进行访问的——代理模式。

此处也可以看到如果ActivityManager相关的Remote端的Service组件可以任意进行改变替换，依然不会影响到Local端的使用。