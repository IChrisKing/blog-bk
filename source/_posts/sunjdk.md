---
title: 如何安装sunjdk，并用它替代openjdk
category:
  - Java
tags:
  - Java
  - tool
  - 填坑
date: 2016-09-21 20:44:20

description: "为了使用Android Studio，需要将openjdk换成sunjdk，但是为了编译，还需要保留openjdk。本文记录了下载安装sunjdk的过程，使用sunjdk替换openjdk的方法，以及过程中遇到的问题和解决方法。"

---

## 目的
为了使用Android Studio，需要将openjdk换成sunjdk

## 过程
1. oracle上下载最新的sunjdk
[JavaPlatform(JDK)](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
下载了8u102版本的Linux x64位的jdk-8u101-linux-x64.rpm
下载完成后
```
rpm -ivh jdk-8u101-linux-x64.rpm
```
点击解压出的.deb，安装程序

2. 配置环境变量
```
sudo vim /etc/profile
```
增加
```
JAVAHOME=/usr/java/jdk1.8.0_102
export PATH=$PATH:/usr/java/jdk1.8.0_102/bin/
export CLASSPATH=$CLASSPATH:/usr/java/jdk1.8.0_102/lib
```

3. 更改默认jdk

```
sudo update-alternatives --install /usr/bin/java java /usr/java/jdk1.8.0_102/bin/java 400

sudo update-alternatives --install /usr/bin/javac javac /usr/java/jdk1.8.0_102/bin/javac 400

sudo update-alternatives --config java
```

显示
```
有 2 个候选项可用于替换 java (提供 /usr/bin/java)。

  选择       路径                                          优先级  状态
------------------------------------------------------------
  0            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      自动模式
* 1            /usr/java/jdk1.8.0_102/bin/java                  400       手动模式
  2            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      手动模式

要维持当前值[*]请按<回车键>，或者键入选择的编号：
```
选择1
查看修改效果
```
java -version
```
显示
```
java version "1.8.0_102"
Java(TM) SE Runtime Environment (build 1.8.0_102-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.102-b14, mixed mode)

```

## 遇到的坑
```
java -version
```
时，报错
```
Error occurred during initialization of VM

java/lang/NoClassDefFoundError: java/lang/Object
```
检查PATH和CLASSPATH都设置没问题，查看安装目时居然发现lib目录下没有tools.jar但有tools.pack;jre/lib下没有rt.jar，但有rt.pack

解决方法：
```
cd /usr/java/jdk1.8.0_102/lib
sudo ../bin/unpack200 tools.pack tools.jar

cd /usr/java/jdk1.8.0_102/jre/lib
sudo ../bin/unpack200 rt.pack rt.jar
```
