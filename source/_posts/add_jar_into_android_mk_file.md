---
title: 給Android系統應用加jar包
category:
  - Android
tags:
  - Android
date: 2016-05-31 19:27:38
description: "開發系統應用AnromManager的過程中，涉及到一些網絡操作，所以使用了okhttp。這就需要把相關的jar包加入到AnromManager的Android.mk當中"
---

## jar放置位置
jar包放在libs/下面

## 修改Android.mk
### 增加jar包到LOCAL_STATIC_JAVA_LIBRARIES
```
LOCAL_STATIC_JAVA_LIBRARIES := \
    android-support-v4 \
    android-support-v13 \
    libphonenumber \
    org.cyanogenmod.platform.sdk \
    eventbus \
    okhttp-2.5.0 \
    okhttp-apache \
    okio \
    oss-android-sdk
```
其中，eventbus \
    okhttp-2.5.0 \
    okhttp-apache \
    okio \
    oss-android-sdk
是新增加的包，放在libs下面，okhttp因为和系统中的某个okhttp重名，所以改名加了版本号。

### 增加jar包到LOCAL_JAVA_LIBRARIES
```
LOCAL_JAVA_LIBRARIES := telephony-common \
    org.apache.http.legacy
```
org.apache.http.legacy 不是libs下面的，但也用到了，加到LOCAL_JAVA_LIBRARIES项，并在项目的AndroidManifests.xml中加入声明，具体位置在：
```
    <application android:label="@string/app_name"
                 android:icon="@drawable/icon"
                 android:theme="@style/Theme.Setup"
                 android:uiOptions="none"
                 android:name=".AnromMgrApp">

        <uses-library android:name="org.apache.http.legacy" android:required="false" />
```

### 增加LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES
```
include $(CLEAR_VARS)

LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := \
    eventbus:libs/eventbus-2.4.0.jar \
    okhttp-2.5.0:libs/okhttp-2.5.0.jar \
    okhttp-apache:libs/okhttp-apache-2.5.0.jar \
    okio:libs/okio-1.6.0.jar \
    oss-android-sdk:libs/oss-android-sdk-1.4.0.jar

include $(BUILD_MULTI_PREBUILT)
```

### 完整的Android.mk文件
```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_STATIC_JAVA_LIBRARIES := \
    android-support-v4 \
    android-support-v13 \
    libphonenumber \
    org.cyanogenmod.platform.sdk \
    eventbus \
    okhttp-2.5.0 \
    okhttp-apache \
    okio \
    oss-android-sdk

LOCAL_SRC_FILES := $(call all-java-files-under, src)

LOCAL_MODULE_TAGS := optional

LOCAL_PACKAGE_NAME := AnromManager
LOCAL_CERTIFICATE := platform
LOCAL_PRIVILEGED_MODULE := true

LOCAL_PROGUARD_FLAG_FILES := proguard.flags


LOCAL_JAVA_LIBRARIES := telephony-common \
    org.apache.http.legacy

LOCAL_RESOURCE_DIR := $(addprefix $(LOCAL_PATH)/, $(res_dir))
LOCAL_AAPT_FLAGS := --auto-add-overlay

include $(BUILD_PACKAGE)

include $(CLEAR_VARS)

LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := \
    eventbus:libs/eventbus-2.4.0.jar \
    okhttp-2.5.0:libs/okhttp-2.5.0.jar \
    okhttp-apache:libs/okhttp-apache-2.5.0.jar \
    okio:libs/okio-1.6.0.jar \
    oss-android-sdk:libs/oss-android-sdk-1.4.0.jar

include $(BUILD_MULTI_PREBUILT)
```
