---
title: AsyncTask
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "首先给出了一个使用AsyncTask的例子，然后根据源码，分析了AsyncTask的执行过程。最后列出AsyncTask的常见问题"
date: 2016-10-31 19:07:51
---
[在項目中使用AsyncTask會有什麼問題麼](http://www.jianshu.com/p/c925b3ea1444)
## 使用AsyncTask的例子
```  
import android.app.Activity;  
import android.app.ProgressDialog;  
import android.os.AsyncTask;  
import android.os.Bundle;  
import android.util.Log;  
import android.widget.TextView;  
  
public class MainActivity extends Activity  
{  
  
    private static final String TAG = "MainActivity";  
    private ProgressDialog mDialog;  
    private TextView mTextView;  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState)  
    {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
  
        mTextView = (TextView) findViewById(R.id.id_tv);  
  
        mDialog = new ProgressDialog(this);  
        mDialog.setMax(100);  
        mDialog.setProgressStyle(ProgressDialog.STYLE_HORIZONTAL);  
        mDialog.setCancelable(false);  
  
        new MyAsyncTask().execute();  
  
    }  
  
    private class MyAsyncTask extends AsyncTask<Void, Integer, Void>  
    {  
  
        @Override  
        protected void onPreExecute()  
        {  
            mDialog.show();  
            Log.e(TAG, Thread.currentThread().getName() + " onPreExecute ");  
        }  
  
        @Override  
        protected Void doInBackground(Void... params)  
        {  
  
            // 模拟数据的加载,耗时的任务  
            for (int i = 0; i < 100; i++)  
            {  
                try  
                {  
                    Thread.sleep(80);  
                } catch (InterruptedException e)  
                {  
                    e.printStackTrace();  
                }  
                publishProgress(i);  
            }  
  
            Log.e(TAG, Thread.currentThread().getName() + " doInBackground ");  
            return null;  
        }  
  
        @Override  
        protected void onProgressUpdate(Integer... values)  
        {  
            mDialog.setProgress(values[0]);  
            Log.e(TAG, Thread.currentThread().getName() + " onProgressUpdate ");  
        }  
  
        @Override  
        protected void onPostExecute(Void result)  
        {  
            // 进行数据加载完成后的UI操作  
            mDialog.dismiss();  
            mTextView.setText("LOAD DATA SUCCESS ");  
            Log.e(TAG, Thread.currentThread().getName() + " onPostExecute ");  
        }  
    }  
}  
```
MyAsyncTask繼承自AsyncTask,需要重寫的函數主要有四個：onPreExecute,doInBackground,onProgressUpdate,onPostExecute

## AsyncTask源碼
以下源碼基於android sdk 21版本，源碼位置：android/os/AsyncTask.java

### execute
在AsyncTask的使用中，只要寫好繼承於AsyncTask的類之後，就可以直接使用`new MyAsyncTask().execute();`來執行調用。所以，對於源碼的分析，也從execute()方法開始。
```
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```
分析executeOnExecutor這個函數，
* 首先，判斷了AsyncTask的狀態mStatus，如果不是PENDING狀態，就根據當前狀態拋出對應的exception,由此可見，每個AsyncTask在完成之前只能執行一次。
* 然後將mStatus設置爲RUNNING
* 執行onPreExecute()，這是依然在UI線程，可以根據需要執行一些準備工作
* 接下來兩行，出現了兩個沒見過的變量：一個是mWorker,我們將params賦值給了mWorker.mParams。第二個是mFuture，我們執行了exec.execute(mFuture)。分別來分析這兩個變量和這兩行代碼。

### mWorker和mFuture
在源碼上部，可以看到這兩個變量的定義
```
    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;
```
WorkerRunnable是Callable的子类，且包含一个mParams用于保存我们传入的参数。
```
  private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {  
        Params[] mParams;  
}  
```
mWorker和mFuture的初始化都在AsyncTask的構造函數中：
```
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occured while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```
先看mWorker
```
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };
```
new了一個WorkerRunnable的實現類，並且實現了call方法，在call方法中，
* 首先設置了mTaskInvoked爲true，這個在後面會用到
* 最終調用到了`postResult(doInBackground(mParams));`查看這個函數
```    
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```
handler，message那一套東西。在這裏send了一個消息。
* 這個消息的message.what爲MESSAGE_POST_RESULT
* message.object是一個AsyncTaskResult類的對象，查看這個類的源碼
```
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```
AsyncTaskResult就是一个简单的携带参数的对象。

有message的發出方，就肯定有接收方，查看sHandler的初始化過程
```
private static final InternalHandler sHandler = new InternalHandler();
```
查看InternalHandler類：
```
    private static class InternalHandler extends Handler {
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
InternalHandler接收兩種message，其中對MESSAGE_POST_RESULT消息的處理調用了finish()方法，查看這個方法：
```
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```
調用了onPostExecute方法，並且把mStatus設置爲FINISHED。

mWorker分析完了，接下來看mFuture，在AsyncTask的構造方法中初始化
```
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occured while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
```
new了一個FutureTask的實現類，並把mWorker作爲參數傳入。重寫了done方法。在任務執行結束後會調用postResultIfNotInvoked(get())，其中get()方法獲取mWorker的call的返回值，也就是result。

查看postResultIfNotInvoked方法：
```
    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
```
這裏首先判斷了wasTaskInvoked，如果是false，則調用postResult方法。然而在前面mWorker的初始化過程中，就已經將mTaskInvoked設爲true了，所以，postResult方法並不會調用到。

### 繼續execute方法
mWorker和mFuture的初始化已經分析完，接下來繼續看execute方法中的exec.execute(mFuture)

exec，也就是sDefaultExecutor，查看它的初始化：
```
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```
它是一個SerialExecutor類的實例：
```
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
SerialExecutor內部維護着一個任務隊列ArrayDeque<Runnable> mTasks。

查看execute（Runnable runnable）方法，將Runnable r放到mTask的隊尾，然後調用scheduleNext，依次取出隊首的任務，執行任務。其中，THREAD_POOL_EXECUTOR是一個線程池。
```
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;
    
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
                    
                        private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

```
這個線程池最大支持CPU_COUNT * 2 + 1個線程，再加上長度爲128的阻塞隊列。

然而在執行的時候，是依次取出任務，線性執行的。

## 整個過程
AsyncTask內部維護了一個線程池，一個任務隊列，從任務隊列中依次取出任務，並使用線程池中的線程來執行。執行完成後，使用handler，message機制更新狀態。
```
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```
* 狀態判斷
* 設置mStatus爲RUNNING
* onPreExecute()進行準備操作
* 將參數賦值給mWorker.mParams，mWorker是一个Callable的子類，並且在內部的call()方法中，調用了doInBackground(mParams)，然后得到的返回值作为postResult的参数进行执行；postResult中通过sHandler发送消息，最终sHandler的handleMessage中完成onPostExecute的调用。
* exec.execute(mFuture)，mFuture为真正的执行任务的单元，将mWorker进行封装，然后由sDefaultExecutor交给线程池进行执行。

## publishProgress
AsyncTask還有一個重要的方法：更新進度條的publishProgress方法。
```
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            sHandler.obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```
發送了一條message
```
    private static class InternalHandler extends Handler {
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
InternalHandler對其進行處理，調用`result.mTask.onProgressUpdate(result.mData);`

## AsyncTask的缺陷
### 3.0之前的崩潰
Android 3.0之前（1.6之前的版本不再关注）规定线程池的核心线程数为5个（corePoolSize），线程池总大小为128（maximumPoolSize），还有一个缓冲队列（sWorkQueue，缓冲队列可以放10个任务），当我们尝试去添加第139个任务时，程序就会崩溃。

### 內存泄漏

如果AsyncTask被声明为Activity的非静态的内部类，那么AsyncTask会保留一个对创建了AsyncTask的Activity的引用。如果Activity已经被销毁，AsyncTask的后台线程还在执行，它将继续在内存里保留这个引用，导致Activity无法被回收，引起内存泄露。
```
new AsyncTask<String, Void, Void>() {
	@override
	protected Void doInBackground(String... params) {
		//do something
		return null;
	}
}.execute("hello");
```
上述代碼使用了匿名內部類創建AsyncTask實例，非靜態內部類會隱式持有外部類的實例引用，即上面的AsyncTask會隱式持有Activity的實例引用。

而在AsyncTask的源碼中，mFuture也使用了匿名內部類來創建對象，而mFuture會被放入到一個靜態成員變量SERIAL_EXECUTOR指向的對象SerialExecutor的一個ArrayDeque類型的集閤中。

當任務處於排隊狀態時，Activity實例引用被靜態常量SERIAL_EXECUTOR間接持有。

此時，如果Activity被銷燬（比如旋轉屏幕），因爲AsyncTask的引用，該Activity無法被回收，造成內存泄漏。

爲了避免內存泄漏，可以使用靜態內部類和弱引用的方式解決。

### cancel的問題
AsyncTask是支持cancel()方法取消提交的，然而cancel並非真的能起作用。
```
    public final boolean cancel(boolean mayInterruptIfRunning) {
        mCancelled.set(true);
        return mFuture.cancel(mayInterruptIfRunning);
    }
```
cancel方法有一個boolean型的參數mayInterruptIfRunning，表示是否可以打斷正在執行的任務。

1. 當調用cancel(false)，不打斷正在執行的任務
* 處於doInBackground的任務不受影響，繼續執行
* 任務結束後調用onCancelled方法，而非onPostExecute方法

2. 當調用cancel(true),打斷正在執行的任務
* 如果doInBackground方法處於阻塞狀態（如調用Thread.sleep,wait方法等），則拋出InterruptedException
* 正在執行的任務並非都能打斷，有時會打斷失敗
