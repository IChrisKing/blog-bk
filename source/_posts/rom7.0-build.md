---
title: Android7.0Rom编译相关
category:
  - Anrom
  - rom7.0
tags:
  - Android
  - Anrom
description: "基于魔趣android7.x系统。记录编译相关的内容。"
date: 2017-08-25 10:21:40
---

## 踩过的坑 ###
### ？？？ ###
错误描述
没记下来
解决方法
执行
```
art/runtime/interpreter/mterp/rebuild.sh
```

### Cannot allocate memory ###
错误描述
```
ninja: fatal: fork: Cannot allocate memory
build/core/ninja.mk:151: recipe for target 'ninja_wrapper' failed
make: *** [ninja_wrapper] Error 1
```
解决方法
#### 第一种解决方法：内存不够用了，首先执行free，查看内存使用状况
```
              total        used        free      shared  buff/cache   available
Mem:       16382208     9798008      314756      734804     6269444     5714252
Swap:       8098808      276960     7821848

```
如果本身硬件内存够大，优先考虑降低内存使用量，运行top，shift+m，按照内存排序，把内存使用量大的进程看情况杀掉一些。

* 清除buff/cache
执行free后发现buff/cache异常的高，占了快一半的内存。
```
echo 3 > /proc/sys/vm/drop_caches
```

#### 第二种方法，修改JACK_SERVER_VM_ARGUMENTS
1. 第一种解决方法，这种起作用了
编辑~/.profile  增加
```
export JACK_SERVER_VM_ARGUMENTS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4g"
```
运行
```
 jack-admin kill-server 
 jack-admin start-server
 source ~/.profile
```
然后再编译

2. 修改vim prebuilts/sdk/tools/jack-admin 
将
```
      JACK_SERVER_COMMAND="java -XX:MaxJavaStackTraceDepth=-1 -Djava.io.tmpdir=$TMPDIR $JACK_SERVER_VM_ARGUMENTS -cp $LAUNCHER_JAR $LAUNCHER_NAME"
```
改成
```
      JACK_SERVER_COMMAND="java -XX:MaxJavaStackTraceDepth=-1 -Djava.io.tmpdir=$TMPDIR $JACK_SERVER_VM_ARGUMENTS -Xmx8g -cp $LAUNCHER_JAR $LAUNCHER_NAME"
```
运行
```
 jack-admin kill-server 
 jack-admin start-server
```
然后再编译

#### 第三种方法：修改swap大小

	1.创建交换分区的文件:增加1G大小的交换分区，则命令写法如下，其中的 count 等于想要的块大小。
	```
	# dd if=/dev/zero of=/home/swapfile bs=1M count=1024
	```
	2.设置交换分区文件:
	```
	# mkswap /home/swapfile  #建立swap的文件系统
	```
	3.立即启用交换分区文件:
	```
	# swapon /home/swapfile   #启用swap文件
	```
	4.使系统开机时自启用，在文件/etc/fstab中添加一行：
	```
	/home/swapfile swap swap defaults 0 0
	```

### subcommand failed ###
错误描述：
```
/bin/bash: xmllint: 未找到命令
.........
[ 32% 1076/3288] Building with Jack: /home/...mework_intermediates/with-local/classes.dex
ninja: build stopped: subcommand failed.
build/core/ninja.mk:151: recipe for target 'ninja_wrapper' failed
make: *** [ninja_wrapper] Error 1

```
解决方法：
```
sudo apt-get install libxml2-utils
```

错误描述：
```
FAILED: /bin/bash -c "prebuilts/misc/linux-x86/flex/flex-2.5.39 -o/home/chris/rom7.0/out/host/linux-x86/obj/STATIC_LIBRARIES/libaidl-common_intermediates/aidl_language_l.cpp system/tools/aidl/aidl_language_l.ll"
flex-2.5.39：严重内部错误，exec of /usr/bin/m4 failed
[  0% 8/49773] Yacc: aidl <= system/tools/aidl/aidl_language_y.yy
FAILED: /bin/bash -c "prebuilts/misc/linux-x86/bison/bison -d  --defines=/home/chris/rom7.0/out/host/linux-x86/obj/STATIC_LIBRARIES/libaidl-common_intermediates/aidl_language_y.h -o /home/chris/rom7.0/out/host/linux-x86/obj/STATIC_LIBRARIES/libaidl-common_intermediates/aidl_language_y.cpp system/tools/aidl/aidl_language_y.yy"
[  0% 8/49773] host C++: ijar <= build/tools/ijar/classfile.cc
ninja: build stopped: subcommand failed.
build/core/ninja.mk:151: recipe for target 'ninja_wrapper' failed
make: *** [ninja_wrapper] Error 1
```
解决方法：
```
sudo apt-get install m4
```

### Oracle JDK 7 is NOT installed. ###
错误描述
```
Oracle JDK 7 is NOT installed.
dpkg: 处理软件包 oracle-java7-installer (--configure)时出错：
 子进程 已安装 post-installation 脚本 返回错误状态 1
正在设置 libxml2-utils (2.9.3+dfsg1-1ubuntu0.2) ...
在处理时有错误发生：
 oracle-java7-installer
E: Sub-process /usr/bin/dpkg returned an error code (1)
```
解决方法：
```
sudo update-alternatives --config java
有 2 个候选项可用于替换 java (提供 /usr/bin/java)。

  选择       路径                                          优先级  状态
------------------------------------------------------------
  0            /usr/lib/jvm/java-8-oracle/jre/bin/java          1081      自动模式
  1            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      手动模式
* 2            /usr/lib/jvm/java-8-oracle/jre/bin/java          1081      手动模式

要维持当前值[*]请按<回车键>，或者键入选择的编号：1
update-alternatives: 使用 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java 来在手动模式中提供 /usr/bin/java (java)
```