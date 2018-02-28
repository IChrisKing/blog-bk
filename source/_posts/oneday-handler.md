---
title: android的Handler机制详解
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "详细分析了Handler，Looper，Message，MessageQueue之间的关系，和使用Handler过程中的程序运行流程。然后简要分析了HandlerThread类的原理和使用方法。"
date: 2016-09-26 21:13:00
---

[能讲讲Android的Handler机制吗？](http://www.jianshu.com/p/108db0240a34)

## 用来解决什么问题
我们知道在android的主线程（UI线程）中，是不适合作耗时操作的。而界面的刷新，又必须在UI线程当中。也就是说，当我们需要做一个耗时的操作，并且在操作完成后更新界面时，一种比较常规的解决方法是使用Handler和Message。

开启一个子线程，将耗时的操作放在子线程中进行，并使用Message和主线程中的Handler进行通信。当子线程完成耗时操作后，使用sendMessage函数，向主线程发送message，这些message会放入主线程的message队列中，并由主线中的handlerMessage函数进行处理。

## 一个简单的例子
```
public class MyHandlerActivity extends Activity { 
    Button button; 
    MyHandler myHandler; 
    
    int UPDATE_UI = 100;
    
    private Handler myThreadHandler = new Handler() {
        public void handleMessage(Message msg) {
        switch(msg.what) {
        	case UPDATE_UI
		//更新UI
		//。。。
		//。。。
		break;
	default:
		break;
	}
     }

    };
 
    protected void onCreate(Bundle savedInstanceState) { 
        super.onCreate(savedInstanceState);
        setContentView(R.layout.handlertest); 

        MyThread m = new MyThread(); 
        new Thread(m).start(); 
    }  
 
    class MyThread implements Runnable { 
        public void run() { 
 
            try { 
                Thread.sleep(10000); 
            } catch (InterruptedException e) { 
                // TODO Auto-generated catch block 
                e.printStackTrace(); 
            } 
 
            Log.d("thread......."， "mThread........"); 
            Message msg =  myThreadHandler.obtainMessage(UPDATE_UI);
            myThreadHandler.sendMessage(msg); // 向Handler发送消息，更新UI 
 
        } 
    } 
} 
```

可以看到，UI线程在onCreate中初始化了MyThread，并start了MyThread。我们也可以调用其他类中的函数来实现耗时操作，只需将myThreadHandler传递过去就可以。

MyThread中休眠1秒来模拟耗时操作。之后使用Handler的sendMessage方法向UI线程发送消息，通知UI线程更新界面。这个消息是由myThreadHandler进行处理的。myThreadHandler是一个Handler类，并实现了handlerMessage方法。

## 四個類 Looper MessageQueue Handler Message 
在主线程中，使用handler很简单，new一个Handler对象实现其handleMessage方法，在handleMessage中
提供收到消息后相应的处理方法即可。

那么，当子线程调用sendMessage(msg);发送一个message对象时，Handler是如何接收该message对象并处理的呢？这个过程主要涉及到四个类：

（未特別指出的源码，位置都在/frameworks/base/cre/java/android/os/目录下）

1. Message：消息分为硬件产生的消息(如按钮、触摸)和软件生成的消息；
2. MessageQueue：消息队列的主要功能向消息池投递消息(MessageQueue.enqueueMessage)和取走消息池的消息(MessageQueue.next)；
3. Handler：消息辅助类，主要功能向消息池发送各种消息事件(Handler.sendMessage)和处理相应消息事件(Handler.handleMessage)；
4. Looper：不断循环执行(Looper.loop)，按分发机制将消息分发给目标处理者。

### Looper
Looper是一个循环器，他不断循环执行Looper.loop来查看当前是否有消息，如果有，将消息分发出去。

**一个线程里面，只能有一个Looper，**他负责管理此线程中的MessageQueue.
```
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

Looper的主要函数有两个：prepare和loop
```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

final MessageQueue mQueue;
final Thread mThread;
    
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

可见prepare的核心，就是sThreadLocal.set(new Looper(quitAllowed));并且可以看出，prepare只能执行一次，sThreadLocal.set也只能set一次，否则就会报错。
sThreadLocal.get()方法和set方法的源码已经深入到SDK当中，在/java/lang/ThreadLocal.java中，可以大致看一下。
```
    public T get() {
        // Optimized for the fast path.
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values != null) {
            Object[] table = values.table;
            int index = hash & values.mask;
            if (this.reference == table[index]) {
                return (T) table[index + 1];
            }
        } else {
            values = initializeValues(currentThread);
        }

        return (T) values.getAfterMiss(this);
    }
    
	public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }
```
这个方法其实是返回了当前线程存储的Looper实例。
（[关于ThreadLocal](http://blog.csdn.net/qjyong/article/details/2158097)）

```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
而loop则不断的调用queue.next来取出MessageQueue中的消息来处理。

### Handler
Handler负责往MessageQueue中添加消息（sendMessage）和处理消息（handleMessage）。这两个过程，是异步的。handler创建时会关联一个looper，默认的构造方法将关联当前线程的looper。
```
    public Handler(Callback callback, boolean async) {
        .........
        .........

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

**一个线程可以有多个Handler，但是只能有一个Looper！**

#### 发送消息
在Handler的源码文件中，可以发现，无论使用哪一种发送消息的方式，最终都会调用到
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
再看MessageQueue的源码
```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
这个函数的核心功能，是将该消息插入到MessageQueue中。

#### 处理消息
处理消息的函数是dispatchMessage，首选handlerCallback(msg),当msg.callback为空时，使用handleMessage(msg)处理。也就是说，子类只需要实现handleMessage(msg)方法，就可以实现处理消息的功能。
```
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
    
private static void handleCallback(Message message) {
        message.callback.run();
    }
    
//由子类实现
public void handleMessage(Message msg) {
    }
```

### Handler的特点
1. handler可以在任意线程中发送消息，这些消息会被放入该handler关联的looper的messageQueue当中。而每个线程只有一个looper和一个messageQueue，也就是说，当一个线程有多个handler时，所有handler发出的message都会被放到同一个messageQueue中，等着同一个looper来分发处理它们。

2. handler是在与其关联的looper中进行消息处理的。
Looper.loop不断取出messageQueue中的消息，并分发给对应的handler进行处理。

这就达到了子线程进行耗时操作，主线程（UI线程）进行界面更新的需求。

总结一下：
1. 每一个线程都只有一个Looper，维护一个MessageQueue。Looper通过prepare()和loop()初始化并保证自己具有收到和分发message的功能，维护MessageQueue。
2. Handler会关联到一个Looper，默认关联当前线程的Looper。也可以说，通常Handler和Looper在同一个线程中，比如UI线程。Handler需要实现处理消息的方法，通常是handleMessage()
3. handler可以在任何线程中发送message给Looper，比如，UI线程可以将handler实例作为参数给子线程，并在子线程中通过handler发送message给UI线程。

再看文章开始的那个简单的例子，很明显，它实现了handleMessage()方法；新开了子线程；在子线程中通过handler实例发送了message。那么，Looper在哪里？Looper的prepare()和loop()在哪里？

## 例子中的整个过程
### 1.UI线程初始化Handler

其实，android已经为ActivityThread实现了Looper，并运行了prepare()和loop()，同时还有一个sMainThreadHandler与这个looper关联。查看/framework/base/core/java/android/app/ActivityThread.java
```
final Looper mLooper = Looper.myLooper();
...
static Handler sMainThreadHandler;  // set once in main()
.........

public static void main(String[] args) {
        ....

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

.....

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
查看Looper中相关函数的实现
``` 
public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
    
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
    
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
UI主线程初始化时会通过ThreadLocal创建一个Looper，该Looper与UI主线程一一对应
。使用ThreadLocal的目的是保证每一个线程只创建唯一一个Looper。

之后我们创建自己的Handler时，直接获取第一个Handler（即sMainThreadHandler）创建的Looper。并共同使用Looper初始化的时候创建的消息队列MessageQueue。至此，主线程、消息循环、消息队列之间的关系是1:1:1。
Handler、Looper、MessageQueue的初始化流程如图所示:

![image](/assets/img/android_handler/new_handler.png)

Handler持有对UI主线程消息队列MessageQueue和消息循环Looper的引用，子线程可以通过Handler将消息发送到UI线程的消息队列MessageQueue中。

### 2. Handler创建消息
每一个消息都需要被指定的Handler处理，通过Handler创建消息便可以完成此功能。Android消息机制中引入了消息池。Handler创建消息时首先查询消息池中是否有消息存在，如果有直接从消息池中取得，如果没有则重新初始化一个消息实例。使用消息池的好处是：消息不被使用时，并不作为垃圾回收，而是放入消息池，可供下次Handler创建消息时使用。消息池提高了消息对象的复用，减少系统垃圾回收的次数。消息的创建流程如图所示。
![image](/assets/img/android_handler/new_message.png)

查看源代码
Handler.java
```
public final Message obtainMessage(int what)
    {
        return Message.obtain(this, what);
    }
```

Message.java
```
public static Message obtain(Handler h, int what) {
        Message m = obtain();
        m.target = h;
        m.what = what;

        return m;
    }


public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

### 3. Handler发送消息
关于Handler发送消息的内容，上面已经提到过了。主要就是将一条消息放到Looper维护的MessageQueue中去。

### 4. Handler处理消息
UI主线程通过Looper.loop查询消息队列MessageQueue，当发现有消息存在时会将消息从消息队列中取出。首先分析消息，通过消息的参数判断该消息对应的Handler，然后将消息分发到指定的Handler进行处理。
子线程通过Handler、Looper与UI主线程通信的流程如图所示。
![image](/assets/img/android_handler/handle_message.png)

## 既然一个线程只能有一个Looper，但可以有多个Handler，那么Looper是如何将消息发送给正确的Handler处理的呢？
** 依靠msg.target **
在例子当中可以看到，发送消息的过程主要是这两行代码：
```
            Message msg =  myThreadHandler.obtainMessage(UPDATE_UI);
            myThreadHandler.sendMessage(msg); // 向Handler发送消息，更新UI 
```
这两行代码其实都有设置msg.target的过程，先看obtainMessage()
### obtainMessage()
```
    public final Message obtainMessage(int what)
    {
        return Message.obtain(this, what);
    }
```
Handler的obtainMessage方法，调用到了Message的obtain方法，并传入参数this，也就是当前的myThreadHandler。再看Message的obtain(Handler h, int what)方法：
```
    public static Message obtain(Handler h, int what) {
        Message m = obtain();
        m.target = h;
        m.what = what;

        return m;
    }
```
可以看到m.target = h;将当前msg的target设为myThreadHandler。

###sendMessage()
上面讲到，所有发送消息的函数，最终都会调用到Handler的enqueueMessage方法
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
该方法的第一行msg.target = this;，就将msg的target设置为了当前的handler。

###  loop() 处理消息
上面讲到，消息的处理是依靠loop方法不断的取出消息，并发送给对应的handler进行处理，在loop()方法中，负责这一工作的是`msg.target.dispatchMessage(msg);`可以看到，loop方法，正是依靠msg.target来确定调用哪一个handler来处理消息的。


## HandlerThread
应用程序当中为了实现同时完成多个任务，所以我们会在应用程序当中创建多个线程。为了让多个线程之间能够方便的通信，我们可以使用HandlerThread实现线程间的通信。

HandlerThread继承于Thread，所以它本质就是个Thread。与普通Thread的差别就在于，主要的作用是建立了一个线程，并且创立了消息队列，有来自己的looper,可以让我们在自己的线程中分发和处理消息。

HandlerThread类可以很方便地创建一个带有looper的新线程。该looper可以被用来创建hanlder对象。需要注意的是start方法必须要调用。

源码位置：/framework/base/core/java/android/os/HandlerThread.java
HandlerThread的代码非常简洁，核心函数包括：

```
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```
可以看到，在该run方法中也是先调用了Looper.prepare()方法，然后通过Looper.myLooper()方法得到该线程所关联的looper对象，最后会调用Looper.loop()方法让消息队列循环起来。由此可以看出，HandlerThread的run方法主要就是将我们上面给出的正常情况下在新线程中创建Handler的代码做了一些封装而已。 在创建HandlerThread对象并调用其start方法之后，该HandlerThread线程就已经关联了looper对象（通过Looper.prepare()方法关联），并且该线程内部的消息队列循环了起来（通过Looper.loop()方法）。 

```
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
```
通过handlerThread.getLooper()可以得到handlerThread线程所关联的looper对象

```
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
```
用来退出looper。

### HandlerThread的例子
1. 创建一个HandlerThread，即创建了一个包含Looper的线程。

HandlerThread handlerThread = new HandlerThread("leochin.com");
handlerThread.start(); //创建HandlerThread后一定要记得start()

2. 获取HandlerThread的Looper

Looper looper = handlerThread.getLooper();

3. 创建Handler，通过Looper初始化

Handler handler = new Handler(looper);

通过以上三步我们就成功创建HandlerThread。通过handler发送消息，就会在子线程中执行。

如果想让HandlerThread退出，则需要调用handlerThread.quit();。
