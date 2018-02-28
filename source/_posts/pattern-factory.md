---
title: Android中的设计模式之工厂模式系列
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "工厂模式系列主要包括简单工厂模式，工厂方法模式，抽象工厂模式。在Android中主要用到的是抽象工厂模式。对于抽象工厂类ThreadFactory，Android中的AsyncTask,MultiTaskDealer,ActivityView各自有不同的实现方式。"
date: 2016-11-17 18:21:26
---
## 简介
关于工厂模式，主要包括简单工厂模式，工厂方法模式，抽象工厂模式。

关于他们的定义和异同，可以查看下面三篇文章。

[简单工厂模式](http://www.cnblogs.com/java-my-life/archive/2012/03/22/2412308.html)

[工厂方法模式](http://www.cnblogs.com/java-my-life/archive/2012/03/25/2416227.html)

[抽象工厂模式](http://www.cnblogs.com/java-my-life/archive/2012/03/28/2418836.html)

## Android中的抽象工厂模式
抽象工厂模式提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。 

就是把生产抽象成一个接口，每个实例类都对应一个工厂类（普通工厂只有一个工厂类），同时所有工厂类都继承这个生产接口。

Android中对抽象工厂模式的运用的典型例子就是对ThreadFactory这个接口类的实现。

工厂的抽象：
```
public abstract interface ThreadFactory
{
  public abstract Thread newThread(Runnable paramRunnable);
}
```

产品的抽象：
```
public abstract interface Runnable
{
  public abstract void run();
}
```

### AsyncTask工厂类的实现：
frameworks/core/java/android/os/AsyncTask.java
```
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
```

### MultiTaskDealer工厂类的实现
/frameworks/base/services/core/java/com/android/server/pm/MultiTaskDealer.java
```
        ThreadFactory factory = new ThreadFactory()
        {
            private final AtomicInteger mCount = new AtomicInteger(1);

            public Thread newThread(final Runnable r) {
                if (DEBUG_TASK) Log.d(TAG, "create a new thread:" + taskName);
                return new Thread(r, taskName + "-" + mCount.getAndIncrement());
            }
        };
```

### ActivityView工厂类的实现
frameworks/base/core/java/android/app/ActivityView.java
``
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "ActivityView #" + mCount.getAndIncrement());
        }
    };
```
   