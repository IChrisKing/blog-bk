---
title: 爲android添加一個新的service函數
category:
  - Anrom
tags:
  - Anrom
  - Android
date: 2015-01-06 15:07:56
description: "利用android現有的service，向其中添加新的函數來實現自己的需求。本文記錄的從上到下的一系列修改過程"
---

## 需求 
frameworks層的keyguard需要探測一個/data/system/user/10/下的文件gesture.key的存在性。
對於frameworks層的應用也好，package/apps下的應用也好，都是沒有su權限的。系統也不允許申請su權限。
這時需要使用service來調用底層那些有su權限的進程來幫忙探測了。
因爲前期已經有些其他的類似需求，前人使用了IMountService來添加新的函數，那現在也添加到這條線路好了。

## keguard的KeyguardViewMediator.java
### 1. 獲取名爲“mount”的service
```
private IMountService getMountService() {
        final IBinder service = ServiceManager.getService("mount");
        if (service != null) {
            return IMountService.Stub.asInterface(service);
        }
        return null;
    }
```
當然，爲了讓這個函數能用，文件頭部需要包含
```
import android.os.storage.IMountService;
import android.os.ServiceManager;
```

### 2. 探測底層的gesture.key的存在性
下面的函數用來探測底層的gesture.key的存在性。這裏也是這次添加過程的最高一層。他調用了mountservice的isGestureKeyExist( )函數，這個函數返回一個int值，爲0說明文件存在，-1說明不存在。
```
private void doKeyguardLocked(Bundle options) {
       
        //by chris
       try {
            IMountService service = getMountService();//條用函數，獲取MountService
            boolean isGestureKeyExist = (service.isGestureKeyExist() == 0)? true:false;//這個isGestureKeyExist就是我們後面要添加到IMountService.java的函數
            Log.d("chris", "isGestureKeyExist " + service.isGestureKeyExist());
            if (ActivityManagerNative.getDefault().getCurrentUser().getUserHandle().getIdentifier() == UserHandle.USER_WORK && !isGestureKeyExist) {
                if (DEBUG)
                    Log.d(TAG, "doKeyguard: not showing because user work");
                Log.d("chris", "startAnromLock");
                startAnromLock();
                return;
            }
        } catch (RemoteException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
```

## /frameworks/base/core/java/android/os/storage/IMountService.java

現在進入添加過程的第二層，一共4個添加點。
### 1.聲明這個函數
```
public int isGestureKeyExist() throws RemoteException;
```
### 2.這個函數的實現
```
public int isGestureKeyExist() throws RemoteException {
                Slog.d("chris","isGestureKeyExist imountservice");
                Parcel _data = Parcel.obtain();
                Parcel _reply = Parcel.obtain();
                int _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    //_data.writeString(password);
                    mRemote.transact(Stub.TRANSACTION_isGestureKeyExist, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readInt();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
```
### 3.針對2中的mRemote.transact(Stub.TRANSACTION_isGestureKeyExist, _data, _reply, 0)函數，添加一個static的值對應。
```
static final int TRANSACTION_isGestureKeyExist = IBinder.FIRST_CALL_TRANSACTION + 41;
```

### 4.添加實現函數

onTransact函數是對應着mRemote.transact(Stub.TRANSACTION_isGestureKeyExist, _data, _reply, 0)的實現方法，所以需要在onTransact函數中添加對應的實現方法。可以看到添加代碼的核心是 isGestureKeyExist()函數，這個 isGestureKeyExist()和本文件中的 isGestureKeyExist()可不是一回事了，這個 isGestureKeyExist()是MountService.java中的 isGestureKeyExist()函數。
```
@Override
public boolean onTransact(int code, Parcel data, Parcel reply,
                int flags) throws RemoteException {
            switch (code) {
                 //這裏省略很多本來就有的case
                 case TRANSACTION_isGestureKeyExist: {
                    data.enforceInterface(DESCRIPTOR);
                    //String password = data.readString();
                    int result = isGestureKeyExist();
                    reply.writeNoException();
                    reply.writeInt(result);
                    return true;
                }
            //這裏省略一些無關代碼
         }
```

## /frameworks/base/services/java/com/android/server/MountService.java

進入第三層了，這裏已經開始直接調命令了，mConnector.execute("cryptfs", "isgesturekeyexist")，這個調用將會被CommandListener.cpp接收到並響應。
```
    @Override
    public int isGestureKeyExist()throws RemoteException{
        Slog.d("chris","isGestureKeyExist mountservice");
        final NativeDaemonEvent event;
        try{
            event = mConnector.execute("cryptfs", "isgesturekeyexist");
            Slog.d("chris","isGestureKeyExist mountservice. result is " + Integer.parseInt(event.getMessage()));
            return Integer.parseInt(event.getMessage());
        }catch(NativeDaemonConnectorException e){
            return e.getCode();
        }
    }
```

## /system/vold/CommandListener.cpp

前面的調用：mConnector.execute("cryptfs", "isgesturekeyexist")，這裏包含兩個部分：cryptfs 和 isgesturekeyexist。這就對應着兩次參數的判斷，首先第一個參數爲cryptfs，找到針對他的處理函數：
```
CommandListener::CryptfsCmd::CryptfsCmd() :
                 VoldCommand("cryptfs") {
}
```
在他下面，是所有與他相關的命令的處理方式，在其中添加關於isgesturekeyexist的處理方法，調用函數cryptfs_is_gesturekey_exist()，這已經是一個c級別的函數了。
```
if (!strcmp(argv[1], "isgesturekeyexist")) {
        if (argc != 2) {
            cli->sendMsg(ResponseCode::CommandSyntaxError, "Usage: cryptfs isgesturekeyexist", false);
            return 0;
        }
        dumpArgs(argc, argv, -1);
        rc = cryptfs_is_gesturekey_exist();
    }
```

## /system/vold/cryptfs.h   &&   /system/vold/cryptfs.c

對於這兩個文件的修改，已經完全進入到C的層面了.首先在cryptfs.h裏面添加函數聲明
```
  int cryptfs_is_gesturekey_exist(void);
```
順便爲我們要判斷的文件gesture.key 定義一個靜態的對應，以方便以後調用：

```
#define GESTUREKEYFILE "/data/system/users/10/gesture.key"
```
最後的最後的，真正的實現是在cryptfs.c文件當中，在這裏才真正的去探測gesture.key文件的存在性。
```
int cryptfs_is_gesturekey_exist(void)
{
    int rc = -1;  
    rc = access(GESTUREKEYFILE, 0 ); //access函數用來測試一些與文件相關的權限。函數的第一個參數爲文件名，第二個參數爲0時，代表測試文件存在與否。返回值爲0時，代表文件存在。爲-1時代表文件不存在。  
    return rc;
}
```

走完這一套流程，添加過程就完成了，回頭看下，其實是最底層提供了接口，然後經過一層一層的包裝，最後被應用層所調用.
