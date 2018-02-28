---
title: ShardPreferences
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "给出了一个使用SharedPreferences的例子，并从例子出发，分析SharedPreferences创建，获取，写入的源码。"
date: 2016-11-01 20:35:00
---
[修改SharedPreferences后两种提交方式有什么区别？](http://www.jianshu.com/p/4dd53e1be5ba)

## 使用SharedPreferences的一个例子
```
SharedPreferences sp = context.getSharedPreferences("name",Context.MODE_PRIVATE);

int val1 = sp.getInt("val1",0);

SharedPreferences.Editor editor = sp.edit();

editor.putInt("val1", val1+1);

editor.apply();//异步写入

// editor.commit();//同步写入
```

## SharedPreferences源码
以上面的使用为例，分析SharedPreferences的相关实现过程。

### 获取SharedPreferences：getSharedPreferences
```
SharedPreferences sp = context.getSharedPreferences("name",Context.MODE_PRIVATE);
```
我们通过context的getSharedPreferences方法来获取SharedPreferences，context的实现类是ContextImpl，查看这个函数(源码位置frameworks/base/core/java/android/app/ContextImpl.java)：
```
    public SharedPreferences getSharedPreferences(String name, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            if (sSharedPrefs == null) {
                sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
            }

            final String packageName = getPackageName();
            ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
            if (packagePrefs == null) {
                packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
                sSharedPrefs.put(packageName, packagePrefs);
            }

            // At least one application in the world actually passes in a null
            // name.  This happened to work because when we generated the file name
            // we would stringify it to "null.xml".  Nice.
            if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                    Build.VERSION_CODES.KITKAT) {
                if (name == null) {
                    name = "null";
                }
            }

            sp = packagePrefs.get(name);
            if (sp == null) {
                File prefsFile = getSharedPrefsFile(name);
                sp = new SharedPreferencesImpl(prefsFile, mode);
                packagePrefs.put(name, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
```
首先可以看到的就是，整段代码是被一个synchronized (ContextImpl.class)同步锁包裹起来的。

然后代码依次做了以下操作：
```
if (sSharedPrefs == null) {
                sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
            }

            final String packageName = getPackageName();
            ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
            if (packagePrefs == null) {
                packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
                sSharedPrefs.put(packageName, packagePrefs);
            }
```
1. 判断sSharedPrefs是否为空，如果为空则执行初始化操作。从初始化代码可以看到，sSharedPrefs是一个用来缓存SharedPreferences的ArrayMap，它的key为包名，它的value为ArrayMap，这个ArrayMap保存的键值对是SharedPreferences文件名和对应的SharedPreferencesImpl（是SharedPreferences的实现类）。

```
            // At least one application in the world actually passes in a null
            // name.  This happened to work because when we generated the file name
            // we would stringify it to "null.xml".  Nice.
            if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                    Build.VERSION_CODES.KITKAT) {
                if (name == null) {
                    name = "null";
                }
            }
```
2. 对于KITKAT以前的版本做一个name的处理，不用分析了，可以忽略。

```
            sp = packagePrefs.get(name);
            if (sp == null) {
                File prefsFile = getSharedPrefsFile(name);
                sp = new SharedPreferencesImpl(prefsFile, mode);
                packagePrefs.put(name, sp);
                return sp;
            }
```
3. 根据参数传入的name，获取SharedPreferencesImpl。如果SharedPreferencesImpl已经存在，它会直接返回已经存在的SharedPreferencesImpl。

其中调用到getSharedPrefsFile(name)这个方法来获取存储数据的xml文件，查看与之相关的函数：
```
    public File getSharedPrefsFile(String name) {
        return makeFilename(getPreferencesDir(), name + ".xml");
    }
    
    private File makeFilename(File base, String name) {
        if (name.indexOf(File.separatorChar) < 0) {
            return new File(base, name);
        }
        throw new IllegalArgumentException(
                "File " + name + " contains a path separator");
    }
    
        private File getPreferencesDir() {
        synchronized (mSync) {
            if (mPreferencesDir == null) {
                mPreferencesDir = new File(getDataDirFile(), "shared_prefs");
            }
            return mPreferencesDir;
        }
    }
    
        private File getDataDirFile() {
        if (mPackageInfo != null) {
            return mPackageInfo.getDataDirFile();
        }
        throw new RuntimeException("Not supported in system context");
    }
```
可以看到getSharedPrefsFile主要是一个文件路径的拼接过程，根据拼接的文件路径来打开文件。

**至此，同步锁内的代码结束**

```
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
```
4. 如果是在多进程模式下，或者目标版本低于HONEYCOMB的时候，会检查是否需要重新从磁盘中加载文件。但是需要说的是MODE_MULTI_PROCESS模式已经被deprecated了，官方建议使用ContentProvider来处理多进程访问.

### SharedPreferencesImpl构造函数
根据上面的代码，可以得知，SharedPreferencesImpl是SharedPreferences的实现类。源码位置：frameworks/base/core/java/android/app/SharedPreferencesImpl.java
```
    SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        startLoadFromDisk();
    }
```
除了赋值操作外，构造函数主要完成了以下操作：
1. makeBackupFile(file)创建了一个.bak的备份文件
2. 调用startLoadFromDisk()方法读出xml文件，这个读取过程是使用DOM方式解析的（一开始就把整个XML给读取出来）。
```
    private void startLoadFromDisk() {
        synchronized (this) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                synchronized (SharedPreferencesImpl.this) {
                    loadFromDiskLocked();
                }
            }
        }.start();
    }
    
        private void loadFromDiskLocked() {
        if (mLoaded) {
            return;
        }
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }

        // Debugging
        if (mFile.exists() && !mFile.canRead()) {
            Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
        }

        Map map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    map = XmlUtils.readMapXml(str);
                } catch (XmlPullParserException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (FileNotFoundException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (IOException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
        }
        mLoaded = true;
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtime;
            mStatSize = stat.st_size;
        } else {
            mMap = new HashMap<String, Object>();
        }
        notifyAll();
    }
```
startLoadFromDisk()方法使用了一个异步线程来读取xml，同时使用了同步锁调用loadFromDiskLocked()来执行真正的读取操作。读取操作最终调用了XmlUtils.readMapXml(str)方法，读取整个xml的内容，放到mMap当中。

## 读取：getxxx
以getInt为例
```
    public int getInt(String key, int defValue) {
        synchronized (this) {
            awaitLoadedLocked();
            Integer v = (Integer)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
```
首先获取锁awaitLoadedLocked()
```
    private void awaitLoadedLocked() {
        if (!mLoaded) {
            // Raise an explicit StrictMode onReadFromDisk for this
            // thread, since the real read will be in a different
            // thread and otherwise ignored by StrictMode.
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                wait();
            } catch (InterruptedException unused) {
            }
        }
    }
```
然后根据key获取value，因为在构造函数中已经完整的读出了xml文件，所以这个获取过程很简单。

最后根据获取到的值判空并返回Value。

### editor的获取
SharedPreferences的写入过程主要分三步，获取editor；put操作（以putInt为例）；apply/commit。

首先分析获取editor的过程
```
    public Editor edit() {
        // TODO: remove the need to call awaitLoadedLocked() when
        // requesting an editor.  will require some work on the
        // Editor, but then we should be able to do:
        //
        //      context.getSharedPreferences(..).edit().putString(..).apply()
        //
        // ... all without blocking.
        synchronized (this) {
            awaitLoadedLocked();
        }

        return new EditorImpl();
    }
```
new了一个EditorImpl的实例。

### putInt
```
        private final Map<String, Object> mModified = Maps.newHashMap();
        
        public Editor putInt(String key, int value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
```
可以看出执行putxxx操作时，数据只是写入到了一个名为mModified的map当中，写入到xml文件的过程将在下一步进行。

### apply
```
        public void apply() {
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }
```

### commit
```
        public boolean commit() {
            MemoryCommitResult mcr = commitToMemory();
            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException e) {
                return false;
            }
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }
```
apply和commit的主要差别在于apply是异步执行的，而commit是同步执行的。两者都调用了几个核心的函数：commitToMemory；mcr.writtenToDiskLatch.await()；
SharedPreferencesImpl.this.enqueueDiskWrite；notifyListeners
依次分析核心函数：
#### commitToMemory
```
private MemoryCommitResult commitToMemory() {
    MemoryCommitResult mcr = new MemoryCommitResult();
    synchronized (SharedPreferencesImpl.this) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            mMap = new HashMap<String, Object>(mMap);
        }
        mcr.mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            mcr.keysModified = new ArrayList<String>();
            mcr.listeners =
                            new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (this) {
            if (mClear) {
                if (!mMap.isEmpty()) {
                    mcr.changesMade = true;
                    mMap.clear();
               }
              mClear = false;
            }

           for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    if (!mMap.containsKey(k)) {
                       continue;
                    }
                    mMap.remove(k);
                } else {
                    if (mMap.containsKey(k)) {
                        Object existingValue = mMap.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mMap.put(k, v);
                }

                 = true;
                if (hasListeners) {
                    mcr.keysModified.add(k);
                }
            }

            mModified.clear();
        }
    }
    return mcr;
}
```
* 首先是一个SharedPreferencesImpl.this的同步锁，所有代码都被这个同步锁包裹。
* 对写操作的数量做了判断，如果大于0，就做一个mMap的copy，并增加计数器。
* 对监听数量做了判断。
* 对EditorImpl加同步锁，在这两重锁的作用下，同一个线程中如果有后续的get操作，都会被阻塞。
* 先处理clear的情况，所以这次clear不会情调本次写操作的数据，只会清除掉之前的数据
* 遍历mModified，处理各个key，value，将mModified中的值更新到mMap当中。
* 标记mcr.changesMade为true，表示有更新
* 清空mModified，并返回MemoryCommitResult mcr。

#### 异步写入MemoryCommitResult
这是apply函数中的一部分。
```
final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };
```
这一段的主要作用是将mcr.writtenToDiskLatch.await放入线程队列中异步执行。在执行的过程中会阻塞住线程，并且在执行结束后从队列中删除这个任务。

#### SharedPreferencesImpl.this.enqueueDiskWrite
将mcr写到磁盘中。
```
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr);
                    }
                    synchronized (SharedPreferencesImpl.this) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };

        final boolean isFromSyncCommit = (postWriteRunnable == null);

        // Typical #commit() path with fewer allocations, doing a write on
        // the current thread.
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (SharedPreferencesImpl.this) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }

        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
    }
```
writeToDiskRunnable是真正执行写操作的runnable，其中首先使用同步锁保护，执行writeToFile操作，然后使用同步锁保护，将写操作的计数减一。

postWriteRunnable是apply时才有的，如果时apply调用了enqueueDiskWrite，则执行传过来的postWriteRunnable。

如果是commit调用了enqueueDiskWrite，执行if (isFromSyncCommit) 包裹的内容，直接调用writeToDiskRunnable.run()，将数据写入磁盘；如果是apply调用了enqueueDiskWrite，将其交给单线程的thread executor去执行。

#### 
```
    private void writeToFile(MemoryCommitResult mcr) {
        // Rename the current file so it may be used as a backup during the next read
        if (mFile.exists()) {//如果文件已经存在
            if (!mcr.changesMade) {//并且changesMade为false，说明没有什么更新和变化
                // If the file already exists, but no changes were
                // made to the underlying map, it's wasteful to
                // re-write the file.  Return as if we wrote it
                // out.
                mcr.setDiskWriteResult(true);//写入文件结果为true
                return;//直接返回
            }
            if (!mBackupFile.exists()) {//如果备份文件不存在，试着将mFile重命名，作为备份文件。这样如果本次写操作失败了，还有备份文件可以恢复。
                if (!mFile.renameTo(mBackupFile)) {//如果重命名失败了
                    Log.e(TAG, "Couldn't rename file " + mFile
                          + " to backup file " + mBackupFile);
                    mcr.setDiskWriteResult(false);//报错，并表示写入文件结果为false
                    return;
                }
            } else {
                mFile.delete();//如果重命名成功了，删除mFile
            }
        }

        // Attempt to write the file, delete the backup and return true as atomically as
        // possible.  If any exception occurs, delete the new file; next time we will restore
        // from the backup.
        try {
            FileOutputStream str = createFileOutputStream(mFile);//创建新的mFile
            if (str == null) {
                mcr.setDiskWriteResult(false);//如果失败的话，将写入文件的结果设为false，并返回
                return;
            }
            XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);//将mcr的mapToWriteToDisk写入到新的mFile当中
            FileUtils.sync(str);
            str.close();//关闭文件流
            ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);//根据mMode设置权限
            try {
                final StructStat stat = Os.stat(mFile.getPath());
                synchronized (this) {//处理相关的变量
                    mStatTimestamp = stat.st_mtime;
                    mStatSize = stat.st_size;
                }
            } catch (ErrnoException e) {
                // Do nothing
            }
            // Writing was successful, delete the backup file if there is one.
            mBackupFile.delete();//删除备份文件
            mcr.setDiskWriteResult(true);//标记写入成功
            return;//返回
        } catch (XmlPullParserException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        } catch (IOException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        }
        // Clean up an unsuccessfully written file
        if (mFile.exists()) {//如果在以上的写入过程中除了问题，删除mFile
            if (!mFile.delete()) {
                Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
            }
        }
        mcr.setDiskWriteResult(false);//标记写入失败
    }
```
至此，写入操作就分析完了。可以看出，每次调用put的时候并没有真的将数据写入磁盘中，而是在调用commit或者apply时，将整个文件重写一遍。所以，在修改多个条目时，应该将所有的put操作执行完，最后调用依次commit或者apply。

## 总结
SharedPreferences的读取，是一个异步的过程，在context.getSharedPreferences时，已经将整个xml文件都读取出来了。另外，会使用ArrayMap对SharedPreferences进行缓存，以SharedPreferences的name作为key。

SharedPreferences的写入是通过editor完成的。有两种方式写入，异步的apply和同步的commit。所以如果是在主线程执行写入操作，要考虑到ANR的问题，最好使用apply。

所有的线程读取的时候都会加SharedPreferencesImpl.this锁，editor写入内存的时候（写入SharedPreferencesImpl.this.mMap）也会加SharedPreferencesImpl.this锁，另外editor调用put，clear, remove方法的时候都会加上EditorImpl.this锁，这些是线程安全的保证，只有在commit/apply后才会写入内存（mMap, xml内容缓存的map变量）和磁盘。