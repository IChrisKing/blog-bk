---
title: 树莓派摄像头没有/dev/video0设备节点的问题
category:
  - 树莓派
  - 配置
tags:
  - 树莓派raspberry
  - raspberry配置
description: "解决非USB摄像头在树莓派3上无法使用motion，mjpg-streamer等视频应用的问题。此外，使用nginx将视频数据流转发，以解决浏览器不支持ipv6的问题。"
date: 2017-03-15 19:15:13
---

## 安装mjpg-streamer
### 安装依赖库
```
sudo apt-get update
sudo apt-get install libjpeg8-dev imagemagick libv4l-dev cmake
```

### 下载mjpg-streamer
```
mkdir mjpg-streamer
cd mjpg-streamer
wget https://codeload.github.com/jacksonliam/mjpg-streamer/zip/master
mv master mjpg-streamer-master.zip
unzip mjpg-streamer-master.zip
```

### 安装mjpg-streamer
```
cd mjpg-streamer-master/mjpg-streamer-experimental
sudo make clean all
sudo cp mjpg_streamer /usr/local/bin
sudo cp output_http.so input_uvc.so /usr/local/lib/
sudo cp -R www /usr/local/www
```

### 配置PATH
```
sudo vim /etc/profile
```
加上两行
```
#mjpg-streamer
export LD_LIBRARY_PATH=/opt/mjpg-streamer/
```
```
source profile
```

### 启动mjpg-streamer
* 方法1：LD_LIBRARY_PATH=/usr/local/lib mjpg_streamer -i "input_uvc.so" -o "output_http.so -w /usr/local/www"
* 方法2：配置好PATH之后，执行 mjpg_streamer -i "/usr/local/lib/input_uvc.so" -o "/usr/local/lib/output_http.so -w /usr/local/www"

### 查看效果
浏览器打开 192.168.2.58:8080/stream.html  (ip换成自己的)

### 填坑1：找不到/dev/video0
```
MJPG Streamer Version.: 2.0
 i: Using V4L2 device.: /dev/video0
 i: Desired Resolution: 640 x 480
 i: Frames Per Second.: -1
 i: Format............: JPEG
 i: TV-Norm...........: DEFAULT
ERROR opening V4L interface: No such file or directory
 Init v4L2 failed !! exit fatal 
 i: init_VideoIn failed
```
这个报错，是因为使用直接插到树莓派视频CSI接口的摄像头造成的。如果使用usb摄像头，则会在/dev目录下出现相关的设备。而直接插到树莓派视频CSI接口的摄像头，使用的是树莓派中的camera module，它是放在/boot/目录下以固件的形式加载的，不是一个标准的v4l2的摄像头ko驱动，所以加载起来之后会找不到/dev/video0的设备节点，这是因为这个驱动是在底层的，v4l2这个驱动框架还没有加载，所以要在/etc/modules里面添加一行bcm2835-v4l2,这句话意思是在系统启动之后会加载这个文件中模块名，这个模块会在树莓派系统的/lib/modules/xxx/xxx/xxx下面，添加之后重启系统，就会在/dev/下面发现video0设备节点了。这个文件名可能不是叫/etc/modules，较早的版本是/etc/modules-load.d/rpi-camera.conf.

然后，重启一下。

## 配置nginx来转发视频数据流
### 安装nginx
```
sudo apt-get install nginx
```

### 编写nginx配置文件
在/etc/nginx/sites-available/ 目录下新建配置文档mjpg，添加如下内容：
```
upstream backend{
        ip_hash;
        server 127.0.0.1:8080; ##mjpg-streamer监听的端口是8080
}
server {
        listen 8000 default_server;##另外找一个端口来转发，比如8000
        listen [::]:8000 default_server ipv6only=on; ##记得配上ipv6的8000端口
        root /home/pi/;##配好root地址
        # Make site accessible from http://localhost/
        charset utf-8;
        access_log  /var/log/nginx/mjpg.access.log;
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}

```

```
cd /etc/nginx/sites-enabled
sudo ln -s /etc/nginx/sites-available/mjpg mjpg
sudo service nginx restart
```

### 查看效果
首先，浏览器要支持ipv6
查看 [fb72:e349:2cd8:52e0:f9d0:e1bf:fb76:1def]:8080/stream.html 
应该可以看到监控图像。
如果看不到，检查mjpg-streamer和nginx是否都打开了。