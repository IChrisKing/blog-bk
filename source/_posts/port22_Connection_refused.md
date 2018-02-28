---
title: "port 22: Connection refused" 
category:
  - Linux
tags:
  - Linux
  - 填坑
date: 2016-03-16 19:01:21
description: "在使用winSCP的过程中,发现自己的linux突然怎么都连不上ssh了，於是記錄下來填坑的過程"
---

在使用winSCP的过程中,发现自己的linux突然怎么都连不上ssh了.

当windows系统中使用winSCP试图连接linux的时候,始终提示port 22: Connection refused

经过初步排查,应该是linux这边出了问题,百度及google了一些解决方法,并排除了一些出错的可能原因:
1. 没装openssh_server 和openssh_client   解决方法:sudo apt-get install openssh_server openssh_client
2. 没装ssh  解决方法:sudo apt-get install ssh
3. 没有开启ssh服务   解决方法:sudo service ssh start     解决后现象:ps -e|grep ssh         显示有sshd    和   ssh-agent  
4. 还可以尝试重启ssh服务    sudo service ssh restart
5. 查看文件/etc/ssh/sshd_config     查看Port  是否是22,或者说,是不是跟scp设置的端口符合.   PermitRootLogin 这一项要设置为 yes  
6. 查看防火墙,因为本台linux系统没装,所以...


以上都排查过后,查看Port 22的状态    
```
netstat -an|grep 22
```
发现大量ipv6的地址,而没有ipv4地址,怀疑是配置有问题,只监听了ipv6,查看/etc/ssh/sshd_config  发现以下内容
```
ListenAddress ::
ListenAddress fc9e:3b75:735a:596b:a229:6390:badb:258
```
果然是硬性绑定了监听ipv6地址,将這幾行注销,重启ssh服务   
```
sudo service ssh restart
```
再次尝试从windows中使用winSCP连接linux,成功了.同时在终端运行ssh "linux的ipv4地址"  也成功了.
