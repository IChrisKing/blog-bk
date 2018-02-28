---
title: 配置shadowsocks
category: Linux
date: 2016-08-05 10:43:19
tags:
	- shadowsocks/vps
description: "shadowsocks翻墙大法的搭建过程"
---

## 服务端配置 ##
### Ubuntu安装shadowsocks
```
sudo apt-get install python-pip
sudo pip install shadowsocks
```
### 配置
```
sudo vim /etc/shadowsocks.json
```
内容如下：
```
{
    // 服务器的IP
    "server": "xxx.xxx.xxx.xxx",
    // 本地的IP，默认为127.0.0.1
    "local_address": "127.0.0.1",
    // 本地端口
    "local_port": 1080,
    //服务器端口与密码，这是配置多个端口的方法
    "port_password": {
        "9001":"123456",
        "9002":"123456",
        "9003":"123456",
        "9004":"123456",
        "9005":"123456"
    },
    // 超时时间，单位是秒，不是毫秒
    "timeout": 200,
    // 加密方法
    "method": "aes-256-cfb"
}
```
### 服务器运行
```
ssserver -c /etc/shadowsocks.json
```
### 自动启动 ###
为服务器端配置开机自动启动shadowsocks
```
sudo vim /etc/rc.local
```
在 exit 0 这一行之前，加上
```
/usr/bin/python /usr/local/bin/ssserver -c /etc/shadowsocks.json
```

## 客户端配置 ##
### 安装shadowsocks
```
sudo apt-get install python-pip python-m2crypto
pip install --upgrade pip
sudo pip install setuptools
sudo pip install shadowsocks
```
### 配置
```
mkdir shadowsocks
cd shadowsocks
sudo vim shadowsocks.json
```
内容如下：
```
{
        "server": "服务器ip",
        "server_port": 选一个服务器端口,
        "local_port": 1080,
        "password": "与服务器端口对应的密码",
        "timeout": 200,
        "method": "aes-256-cfb"
}
```
### 运行shadowsocks
```
sslocal -c ~/shadowsocks/shadowsocks.json
```
### 设置全局PAC模式
安装genpac
```
sudo pip install genpac
```
生成pac文件
```
genpac -p "SOCKS5 127.0.0.1:1080" --gfwlist-proxy="SOCKS5 127.0.0.1:1080"  --gfwlist-url="https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt"  --output="~/shadowsocks/autoproxy.pac"
```
pac文件生成成功后，需要在系统设置中设置该文件
```
在 系统设置 -> 网络 -> 网络代理 中选则 方法 为 自动，然后在下面URL填入
file:///home/xxx/shadowsocks/autoproxy.pac
其中xxx为自己的用户名，完成之后，即可使用PAC文件进行科学上网。
```
### 为Chromium安装Proxy Switchymega插件
全局很方便，然而mint系统并不能如此愉快的玩耍，所以为浏览器单独安装插件吧

[download](https://github.com/FelisCatus/SwitchyOmega/releases/download/v2.4.2/SwitchyOmega.crx)

下载完成后，浏览器打开chrome://extensions/
直接把下载好的文件拖进去

配置方法：
1. 选择New profile
2. 起个名字，比如ss
3. Protocol选择SOCKS5
4. Server 填写127.0.0.1
5. Port填写 1080
6. 點擊左側的Apply changes 
7. 接下來訪問網頁的時候，點擊地址欄右邊的圓圈，選擇剛纔新建好的ss，就可以了
 

## vps购买相关
说了半天还不知道服务器在哪里。。。
来，https://www.kvmla.com

买个便宜的VPS，记得操作系统选Ubuntu系列的，最好最新版，还有32位，64位要看好。我选的Ubuntu-14.04-x86_64

支持支付宝付款，真是适合中国国情啊

