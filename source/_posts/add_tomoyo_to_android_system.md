---
title: 移植TOMOYO的步驟
category:
  - Anrom
tags:
  - Anrom
  - Android
date: 2014-02-14 11:10:14
description: "TOMOYO和SELinux一樣，都是android系統支持的，內核級別的加密方案。在基於android4.x系統下開發ROM的過程中，選擇了使用TOMOYO加密方法來保護系統安全和限制用戶及應用的操作。在後來的android5及後續系統中，android開始提供SELinux來保護系統。"
---

## 移植代碼
kernel目录下的security/tomoyo内容copy来


system/tomoyo内容copy来

## 尋找內核編譯配置文件
device目录下，BoardConfig.mk文件中有两行：
```
# Define kernel config for inline building
TARGET_KERNEL_CONFIG := cyanogenmod_hammerhead_defconfig  A文件   
TARGET_KERNEL_SOURCE := kernel/lge/hammerhead     B文件
```
locate A文件，它就是编译内核的文件，通常在kernel目录下的arch/arm/configs/下

## 修改內核編譯配置文件，添加內核編譯選項
在內核編譯配置文件中添加
```
CONFIG_SECURITY_TOMOYO=y
CONFIG_SECURITY_TOMOYO_MAX_ACCEPT_ENTRY=2048
CONFIG_SECURITY_TOMOYO_MAX_AUDIT_LOG=1024
CONFIG_SECURITY_TOMOYO_POLICY_LOADER="/sbin/tomoyo-init"
CONFIG_SECURITY_TOMOYO_ACTIVATION_TRIGGER="/init"
CONFIG_DEFAULT_SECURITY_TOMOYO=y
CONFIG_DEFAULT_SECURITY="tomoyo"
```

同时确保
```
CONFIG_KEYS=y
CONFIG_SECURITY=y
CONFIG_SECURITYFS=y
CONFIG_NET=y 
#CONFIG_SECURITY_SELINUX is not set
#CONFIG_DEFAULT_SECURITY_SELINUX is not set
```

## 修改commom.mk文件
vender/cm/config/common.mk 里添加一段
```
 # rsync ----这一行定位用的
PRODUCT_PACKAGES += \
    tomoyo-init \
    tomoyo-conf \
    macd \
    mac \
    tomoyo-editpolicy-agent \
    rsync 
```

## 修改init.rc文件
system/core/rootdir/init.rc 里添加macd的相关内容，不过暂时macd还没用上呢

```
 service macd /system/bin/macd
 class core
 socket macd stream 0660 root system 
```

## 驗證移植效果
編譯系統，並刷機

如果手机里出现/security/目录，里面有策略文件

/sys/kernel/security/目录下出现tomoyo目录，里面有策略文件

则tomoyo移植成功


## 排錯方法
* 檢查內核編譯選項是否打開
cp A文件 B/.config

在kernel目录，该设备的目录下，make menuconfig，查看tomoyo选项是否选中，如果没有，请手动去内核编译文件查错，不要在make menuconfig的时候直接选中tomoyo，因为会出很多奇怪的编译错误。

* 查看内核中是否已经编译了tomoyo
adb shell到手机上

grep -r tomoyo /proc/kallsyms

如果有结果，那么证明tomoyo已经成功编译到内核中了
这时考虑是否common.mk中有问题了