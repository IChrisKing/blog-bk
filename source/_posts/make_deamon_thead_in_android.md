---
title: 爲android編寫一個deamon進程
category:
  - Anrom
tags:
  - Anrom
  - Android
date: 2013-06-09 07:55:51
description: "在基於android2.x系列開發的anrom中，有一個分配磁盤空間，並加密的項目，叫做密盤。本文是密盤的底層支持文件，mipand的開發過程。他是一個由.c文件編譯而成的可執行文件，被放置在/system/bin/目錄下，並且作爲一個deamon進程，在android系統中提供服務。"
---

## 編寫deamon進程
在vendor/cm/prebuilt/common/etc/init.d目录下建立02mipan（必须以两位数开头），在里面写要执行的操作
比如，输出点什么
```
L="log -p i -t cm"
$L "____ _   _ ____ _  _ ____ ____ ____ _  _ _  _ ____ ___";
$L "|     \\_/  |__| |\\ | |  | | __ |___ |\\ | |\\/| |  | |  \\";
$L "|___   |   |  | | \\| |__| |__] |___ | \\| |  | |__| |__/";
$L "Welcome to Android `getprop ro.build.version.release` / CyanogenMod-`getprop ro.cm.version`";
```

比如，运行点什么脚本
```
/system/bin/mipand 
```
都是shell脚本，具体写法，可以模仿目录下面那两个文件00banner和90userinit

在/vendor/cm/config/common.mk 中，加上
```
# mipaninit support
PRODUCT_COPY_FILES += \
 vendor/cm/prebuilt/common/etc/init.d/02mipan:system/etc/init.d/02mipan
```
这样，02mipan就会开机过程中自动启动了。

adb shell后可以通过pstree来查看到它

## 把.c文件編譯成android用的可執行文件
然后，mipand是怎么来的捏？

首先pc上建个文件夹，mipand。最好在全英文路径下
mipand里面建文件夹jni

jni里面建一个弱弱的.c文件，要执行的动作放在里面
想要进程常驻的话，可以在main函数最后加上
```
while(1)
{
    sleep(100);
}
```

为了让mipand能够被编译，还需要一个弱弱的Android.mk文件
```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE_TAGS :=optional

LOCAL_SRC_FILES:= mipand.c

LOCAL_MODULE:= mipand

LOCAL_C_INCLUDES := $(KERNEL_HEADERS)

LOCAL_PRELINK_MODULE := false

include $(BUILD_EXECUTABLE)
```
使用ndk把它编译成.d文件，这里使用的是`android-ndk-r8e`

需要把它的路径加到环境变量里。
在~/.profile里加入
```
PATH="/xxxx/xxxx/android-ndk-r8e:$PATH"
NDK="/xxxx/xxxx//android-ndk-r8e"
```
source一下这个.profile，然后，在mipand路径下运行ndk-build

正常的话，会在mipand/libs/armeabi下出现mipand可执行文件

这个可执行文件是android可以用的了
可以运行
```
adb root
remount -o rw,remount -t rootfs /system
adb push /xxx/mipand/libs/armeabi/mipand /system/bin/
```
把这可执行文件发到手机的/system/bin/目录下

## 可執行文件自動copy到/system/bin/
比如，我放到了vendor/cm/prebuilt/mipan/目录下，mipan是自己建的，这不重要，只要这个放置mipand的路径跟后面的操作对应上就可以了

在/vendor/cm/config/common.mk 中，加上
```
PRODUCT_COPY_FILES +=  \
    vendor/cm/prebuilt/mipan/mipand:system/bin/mipand
```
