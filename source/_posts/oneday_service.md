---
title: Android中的Service
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "分析了android系统中的Service。主要介绍其生命周期，不同启动方法，分析了源码。同时，分析了其子类IntentService与它的异同。"
date: 2016-09-21 21:50:19
---

[知道Service吗，它有几种启动方式？](http://www.jianshu.com/p/7a7db9f8692d)

题目：

## service生命周期
两张生命周期的图

![image](/assets/img/android_service/life_cycle_1.png)

![image](/assets/img/android_service/life_cycle_2.png)

当我们第一次使用startservice启动Service时，先后调用了onCreate(),onStart()这两个方法，当停止Service时，则执行onDestroy()方法，这里需要注意的是，如果Service已经启动了，当我们再次启动Service时，不会在执行onCreate()方法，而是直接执行onStart()方法

## 源码位置
frameworks/base/core/java/android/app/Service.java

## 源码分析
* Service是一个抽象类，它有一个抽象方法
```
public abstract IBinder onBind(Intent intent);
```
所有继承自Service的类，都要实现这个抽象方法。

* 继承service的子类在重写service的方法中，除了一个onStart()方法之外，还有一个onStartCommand()方法

```
public void onStart(Intent intent, int startId) {
    }

public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mStartCompatibility ? START_STICKY_COMPATIBILITY : START_STICKY;
    }
 ```
    
可以看到，onStartCommand内部调用了onStart方法，并且返回了一个int类型的值。

查看源码中的注释，可以看到这样的返回值有四个：

```
    /**
     * Constant to return from {@link #onStartCommand}: compatibility
     * version of {@link #START_STICKY} that does not guarantee that
     * {@link #onStartCommand} will be called again after being killed.
     */
     public static final int START_STICKY_COMPATIBILITY = 0;
    
    /**
     * Constant to return from {@link #onStartCommand}: if this service's
     * process is killed while it is started (after returning from
     * {@link #onStartCommand}), then leave it in the started state but
     * don't retain this delivered intent.  Later the system will try to
     * re-create the service.  Because it is in the started state, it will
     * guarantee to call {@link #onStartCommand} after creating the new
     * service instance; if there are not any pending start commands to be
     * delivered to the service, it will be called with a null intent
     * object, so you must take care to check for this.
     * 
     * <p>This mode makes sense for things that will be explicitly started
     * and stopped to run for arbitrary periods of time, such as a service
     * performing background music playback.
     */
    public static final int START_STICKY = 1;
    
    /**
     * Constant to return from {@link #onStartCommand}: if this service's
     * process is killed while it is started (after returning from
     * {@link #onStartCommand}), and there are no new start intents to
     * deliver to it, then take the service out of the started state and
     * don't recreate until a future explicit call to
     * {@link Context#startService Context.startService(Intent)}.  The
     * service will not receive a {@link #onStartCommand(Intent, int, int)}
     * call with a null Intent because it will not be re-started if there
     * are no pending Intents to deliver.
     * 
     * <p>This mode makes sense for things that want to do some work as a
     * result of being started, but can be stopped when under memory pressure
     * and will explicit start themselves again later to do more work.  An
     * example of such a service would be one that polls for data from
     * a server: it could schedule an alarm to poll every N minutes by having
     * the alarm start its service.  When its {@link #onStartCommand} is
     * called from the alarm, it schedules a new alarm for N minutes later,
     * and spawns a thread to do its networking.  If its process is killed
     * while doing that check, the service will not be restarted until the
     * alarm goes off.
     */
    public static final int START_NOT_STICKY = 2;
    
    /**
     * Constant to return from {@link #onStartCommand}: if this service's
     * process is killed while it is started (after returning from
     * {@link #onStartCommand}), then it will be scheduled for a restart
     * and the last delivered Intent re-delivered to it again via
     * {@link #onStartCommand}.  This Intent will remain scheduled for
     * redelivery until the service calls {@link #stopSelf(int)} with the
     * start ID provided to {@link #onStartCommand}.  The
     * service will not receive a {@link #onStartCommand(Intent, int, int)}
     * call with a null Intent because it will will only be re-started if
     * it is not finished processing all Intents sent to it (and any such
     * pending events will be delivered at the point of restart).
     */
    public static final int START_REDELIVER_INTENT = 3;
```

START_STICKY：如果service进程被kill掉，保留service的状态为开始状态，但不保留递送的intent对象。随后系统会尝试重新创建service，由于服务状态为开始状态，所以创建服务后一定会调用onStartCommand(Intent,int,int)方法。如果在此期间没有任何启动命令被传递到service，那么参数Intent将为null。


START_NOT_STICKY：“非粘性的”。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统将会把它置为started状态，系统不会自动重启该服务，直到startService(Intent intent)方法再次被调用;。


START_REDELIVER_INTENT：重传Intent。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统会自动重启该服务，并将Intent的值传入。


START_STICKY_COMPATIBILITY：START_STICKY的兼容版本，但不保证服务被kill后一定能重启。

## 启动service
### 两种方法
* 方法一：Context.startService()

在同一个应用任何地方调用 startService() 方法就能启动 Service 了，然后系统会回调 Service 类的 onCreate() 以及 onStart() 方法。这样启动的 Service 会一直运行在后台，直到 Context.stopService() 或者 selfStop() 方法被调用。另外如果一个 Service 已经被启动，其他代码再试图调用 startService() 方法，是不会执行 onCreate() 的，但会重新执行一次 onStart() 。

* 方法二：Context.bindService()

bindService() 方法是把这个 Service 和调用 Service 的客户类绑定起来，如果调用这个客户类被销毁，Service 也会被销毁。用这个方法的一个好处是，bindService() 方法执行后 Service 会回调onBind() 方法，你可以从这里返回一个实现了 IBind 接口的类，在客户端操作这个类就能和这个服务通信了，比如得到 Service 运行的状态或其他操作。当发现所有绑定都进行了unBind时才会销毁Service。

### 区别
再看一遍生命周期的图

![image](/assets/img/android_service/life_cycle_2.png)

两种启动方法的主要差异就在Active Lifetime期间。

启动时，startService调用onStartCommand,bindService调用onBind

停止时，startService直到service调用selfStop或者有组件调用stopService()才停止,bindService则在发现所有绑定都进行了unBind时才会销毁Service

## service的子类IntentService
### IntentService简介
IntentService是Service类的子类，用来处理异步请求。客户端可以通过startService(Intent)方法传递请求给IntentService。

与Service相比，IntentService有以下不同：
1. startService(Intent)方法启动，而非bindServie

2. IntentServie默认创建新的线程运行. 而Service是缺少这个特质的,因为Service和调用者( 比如某个Activity )是运行在同一个进程中,会直接和调用竞争有限的资源,所以Service不适合用于处理耗时耗资源的任务,否则极易阻塞主线程导致ANR错误.

3. IntentService自动实现多线程, 会新建工作线程专门处理任务, 同时使用了队列来管理请求启动IntentService的各个Intent. 比如当有3个intent都请求该IntentService,那么IntentService会按照队列顺序依次去新建线程,处理任务,保证在同一时间内只有一个intent在调用该intentService.

4. 当调用完毕后intentService会自动停止自身,并处理下一次的调用，直至最后处理完所有请求。所以如果使用intentService, 用户并不需要主动的使用stopService() 或者在intentService中使用stopSelf()来停止.

5. 继承IntentService必须实现onHandleIntent()方法, 将耗时的任务放在这个方法内即可. 

### 源码位置

frameworks/base/core/java/android/app/IntentService.java

### 源码分析
分析源码，来查看上面说到的不同点
1. 实现了onStart和onStartCommand，但只在onBind中返回了null
```
    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    /**
     * You should not override this method for your IntentService. Instead,
     * override {@link #onHandleIntent}, which the system calls when the IntentService
     * receives a start request.
     * @see android.app.Service#onStartCommand
     */
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    /**
     * Unless you provide binding for your service, you don't need to implement this
     * method, because the default implementation returns null. 
     * @see android.app.Service#onBind
     */
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
```

2. 查看onCreate函数
```
    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```
可以看到，在onCreate中，new了一个HandlerThread来处理后续的工作。IntentService在处理事务时，还是采用的Handler方式，创建一个名叫ServiceHandler的内部Handler，并把它直接绑定到HandlerThread所对应的子线程。 

3. IntentService内部实现了一个ServiceHandler类来依次处理任务。
```
private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

4. 在任务处理完成后，调用stopSelf(msg.arg1);来结束自己，这就不再需要用户主动调用stopService,或者在intentService中使用stopSelf()来停止.

5. ServiceHandler把处理一个intent所对应的事务都封装到叫做onHandleIntent的虚函数；因此我们直接实现虚函数onHandleIntent，再在里面根据Intent的不同进行不同的事务处理就可以了。
```
    @WorkerThread
    protected abstract void onHandleIntent(Intent intent);
```

6. 此外，IntentService的构造函数一定是参数为空的构造函数，然后再在其中调用super("name")这种形式的构造函数。
因为Service的实例化是系统来完成的，而且系统是用参数为空的构造函数来实例化Service的
```
    public IntentService(String name) {
        super();
        mName = name;
    }
```
举个栗子：
```
 public MyIntentService() {
  super("com.lenovo.robin.test.MyIntentService");
 }
```

### 使用IntentService的例子
使用IntentService需要两个步骤：
1. 写构造函数
2. 实现虚函数onHandleIntent，并在里面根据Intent
的不同进行不同的事务处理

```
public class MyIntentService extends IntentService {
final static String TAG="robin";
 public MyIntentService() {
  super("com.lenovo.robin.test.MyIntentService");
  Log.i(TAG,this+" is constructed");
 }
 @Override
 protected void onHandleIntent(Intent arg0) {
  Log.i(TAG,"begin onHandleIntent() in "+this);
  try {
   Thread.sleep(10*1000);
  } catch (InterruptedException e) {
     e.printStackTrace();
  }
  Log.i(TAG,"end onHandleIntent() in "+this);
 }
 public void onDestroy()
 {
  super.onDestroy();
  Log.i(TAG,this+" is destroy");
 }
}
```
