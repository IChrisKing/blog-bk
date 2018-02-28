---
title: 源碼中實現禁止手機鏈接usb
category:
  - Anrom
tags:
  - Anrom
  - Android
date: 2014-03-21 16:21:05
description: "一種徹底禁止手機鏈接usb的方法，代碼實現，需要system權限"
---
使用SystemProperties.set("sys.usb.config","none");
这个函数要起作用，需要三个条件
* import android.os.SystemProperties
```
import android.hardware.usb.UsbManager; 
```

* 在AndroidManifest.xml里设置
```
android:sharedUserId="android.uid.system"
```
换言之，需要system权限

* 在Android.mk里设置
```
LOCAL_CERTIFICATE := platform
```

## 具体代码实现：
    private UsbManager mUsbManager;
    
    public void enableADB() {
        SystemProperties.set("sys.usb.config",mUsbManager.USB_FUNCTION_ADB);
    }

    public void disableADB() {
        SystemProperties.set("sys.usb.config","none");
    }  

关于UsbManager的代码，在frameworks/base/services/java/com/android/server/usb
