---
title: raspberry-lirc
category:
  - 树莓派
tags:
  - 树莓派raspberry
  - lirc
  - 红外控制
description: "使用树莓派来模拟红外发射器，远程控制空调。分成三个部分：1.如何使用树莓派模拟空调遥控器。2.如何使用nginx,将网页访问操作变成红外命令发送。3.如何通过端口映射，将访问路由地址的流量转到树莓派上。"
date: 2019-12-19 11:36:23
---

## 使用树莓派模拟红外发射器 ##
### 硬件设备 ###
购买地址：[http://item.taobao.com/item.htm?id=37526197545]（http://item.taobao.com/item.htm?id=37526197545）

功能说明：

1. 红外线接收功能

工作频率：38K HZ

接收距离：18-20m

接收角度：+/-45度

2. 红外线发射功能

波长：940nm

发射距离：7-8m

3. 支持红外线双LED发射，发射效果更强（需要用户自行焊接备用发射管D2,并断开SJ1）

4. 支持强大的LIRC软件，利用LIRC和扩展板，用户几乎可以用来复制所有的红外线遥控器功能（电视，功放，DVD等等电器遥控器），并通过命令来控制你的各种电器设备。

5. 支持XBMC系统，用户可以在XBMC环境下使用扩展板的红外功能

6. 支持双个GPIO按键，用户可以通过编程配置按键功能

扩展板接口图：
 ![IR-Remote-Shield-v1.0](/assets/img/raspberry-lirc/IR-Remote-Shield-v1.0.jpg  "IR-Remote-Shield-v1.0")

管脚对应关系图：
![树莓派 40Pin 引脚对应表](/assets/img/raspberry-lirc/rpi-pins-40-0.jpg  "树莓派 40Pin 引脚对应表")

### 红外接收与发送 ###
#### lirc的安装 ####
LIRC (Linux Infrared remote control)是一个linux系统下开源的软件包。这个软件可以让你的Linux系统能够接收及发送红外线信号。
1. 安装lirc
```
sudo apt-get install lirc
```
#### 红外设备的配置 ####
##### config.txt的配置语法 #####
树莓派因为没有BIOS，所以Raspbian对设备的加载都是依赖在/boot/config.txt中的配置来加载。当Linux内核加载时，会读取/boot/config.txt中的设备配置和设备参数配置来把设备动态加载到Device Tree(DT)中
```
dtoverlay=<device>
dtparam=<param1>,<param2>,...
```
* dtoverlay=要加载设备
这些设备都必须是Raspbian支持的，它们位于/boot/overlays下。设备配置说明位于/boot/overlays/README，可以在这里查看到Raspbian支持的每个设备的具体信息和参数(也可以直接在官方Github查阅最新的设备支持)。在其中找到关于lirc设备的描述
```
Name:   lirc-rpi
Info:   Configures lirc-rpi (Linux Infrared Remote Control for Raspberry Pi)
        Consult the module documentation for more details.
Load:   dtoverlay=lirc-rpi,<param>=<val>
Params: gpio_out_pin            GPIO for output (default "17")

        gpio_in_pin             GPIO for input (default "18")

        gpio_in_pull            Pull up/down/off on the input pin
                                (default "down")

        sense                   Override the IR receive auto-detection logic:
                                 "0" = force active-high
                                 "1" = force active-low
                                 "-1" = use auto-detection
                                (default "-1")

        softcarrier             Turn the software carrier "on" or "off"
                                (default "on")

        invert                  "on" = invert the output pin (default "off")

        debug                   "on" = enable additional debug messages
                                (default "off")

```
文档中关于dtoverlay的例子，恰好使用了lirc-rpi
```
    dtoverlay=lirc-rpi,gpio_out_pin=17,gpio_in_pin=13
```
在本项目中，需要找到lirc的设备名，并为其选择一对引脚来发送和接收红外信号。在/boot/overlays/目录下找到了lirc-rpi.dtbo。
选择一对引脚GPIO.0和GPIO.1，BCM编码为17和18
编辑config.txt，增加lirc相关配置
```
sudo vim /boot/config.txt
```
添加一下一行内容到文档最后
```
dtoverlay=lirc-rpi,gpio_in_pin=18,gpio_out_pin=17
```
* dtparam是设备的参数，具体信息可根据/boot/overlays/README中的说明来配置。这次暂时用不到。

##### 编辑lirc配置文件 #####
编辑 /etc/lirc/hardware.conf，修改以下行。或者直接新建该文件，并添加以下内容
```
LIRCD_ARGS="--uinput --listen"
LOAD_MODULES=true
DRIVER=”default”
DEVICE=”/dev/lirc0″
MODULES=”lirc_rpi”
```
编辑 /etc/lirc/lirc_options.conf， 修改以下行。或者新建该文件，并添加以下内容
```
[lircd]
nodaemon        = False
driver          = default
device          = /dev/lirc0
output          = /var/run/lirc/lircd
pidfile         = /var/run/lirc/lircd.pid
plugindir       = /usr/lib/arm-linux-gnueabihf/lirc/plugins
permission      = 666
allow-simulate  = No
repeat-max      = 600
#effective-user =
#listen         = 0.0.0.0:8766
#connect        = host[:port]
#loglevel       = 6
#uinput         = Yes
#release        = Yes
#logfile        = ...

[lircmd]
uinput          = False
nodaemon        = False
```

**重启树莓派，另lirc生效**

#### 红外接收功能 ####
##### 打开/关闭lirc命令 #####
```
sudo /etc/init.d/lircd stop
sudo /etc/init.d/lircd start
sudo /etc/init.d/lircd restart
或者
sudo service lirc stop
sudo service lirc start
sudo service lirc restart
```

##### 红外接收 #####
```
sudo service lirc stop
mode2  -m -d /dev/lirc0
```
使用空调遥控器，对这红外接收器摁下任意键，屏幕应答打印处类似下面的内容，则证明红外接收功能正常。
下面是摁下on后，收到的编码
```
  2997409

     4330     4495      472     1707      475      606
      473     1707      475     1707      474      604
      472      606      472     1708      473      605
      474      604      473     1708      474      604
      473      613      464     1708      474     1709
      471      605      473     1708      474      609
      468      605      474     1707      472     1710
      472     1709      474     1708      473     1708
      473     1708      474     1707      475     1709
      472      610      468      604      475      602
      473      606      472      604      473      606
      473     1707      475      602      474     1708
      475     1707      474     1707      474     1711
      470      604      474      603      474      604
      474     1708      475      602      473      605
      451      627      446      632      473     1707
      448     1734      448     5307     4339     4492
      473     1708      475      603      447     1734
      471     1711      471      607      447      631
      447     1734      447      631      450      627
      471     1711      448      630      446      633
      445     1734      449     1734      470      607
      446     1736      446      633      444      658
      420     1745      436     1736      472     1711
      445     1739      441     1762      420     1762
      429     1753      418     1762      420      658
      419      663      416      659      418      661
      445      607      467      609      470     1748
      433      635      417     1763      419     1762
      418     1763      420     1761      347      758
      393      659      417      687      391     1790
      392      686      366      713      391      713
      364      687      391     1764      443     1765
      390
```
如果打印内容类似下面内容，则要注意是否忘记了在mode2 后增加了-m参数。
```
space 1815503
pulse 4380
space 4469
pulse 470
space 1709
pulse 473
space 604
pulse 474
space 1707
pulse 483
space 1700
pulse 471
space 607
pulse 470
space 604
pulse 476
space 1705
pulse 449
space 630
pulse 447
space 630
pulse 447
space 1736
pulse 447
space 630
pulse 447
space 630
pulse 447
space 1734
pulse 473
space 1708
pulse 447
space 631
pulse 471
space 1710
pulse 473
space 606
pulse 454
space 623
pulse 447
space 1735
pulse 448
space 1733
pulse 471
space 1709
pulse 448
space 1735
pulse 447
```

下面的命令可以列出可用的按键名
```
irrecord -l
```

##### 红外编码录制 #####
* 对于风扇之类的简单的红外设备，其命令简单，可以使用下面的方式进行红外编码录制
```
irrecord -d /dev/lirc0 ~/lircd.conf
```
根据软件的提示操作即可，这个程序会自动算出你按下的遥控器按键的编码和时长，并录制下来记录在~/lircd.conf文件中

* 对于空调这种复杂的设备，可以使用下面的模板作为lircd.conf
```
begin remote

  name  aircon
  flags RAW_CODES
  eps            30
  aeps          100

  gap          19991

      begin raw_codes

          name on
	      4330     4495      472     1707      475      606
	      473     1707      475     1707      474      604
	      472      606      472     1708      473      605
	      474      604      473     1708      474      604
	      473      613      464     1708      474     1709
	      471      605      473     1708      474      609
	      468      605      474     1707      472     1710
	      472     1709      474     1708      473     1708
	      473     1708      474     1707      475     1709
	      472      610      468      604      475      602
	      473      606      472      604      473      606
	      473     1707      475      602      474     1708
	      475     1707      474     1707      474     1711
	      470      604      474      603      474      604
	      474     1708      475      602      473      605
	      451      627      446      632      473     1707
	      448     1734      448     5307     4339     4492
	      473     1708      475      603      447     1734
	      471     1711      471      607      447      631
	      447     1734      447      631      450      627
	      471     1711      448      630      446      633
	      445     1734      449     1734      470      607
	      446     1736      446      633      444      658
	      420     1745      436     1736      472     1711
	      445     1739      441     1762      420     1762
	      429     1753      418     1762      420      658
	      419      663      416      659      418      661
	      445      607      467      609      470     1748
	      433      635      417     1763      419     1762
	      418     1763      420     1761      347      758
	      393      659      417      687      391     1790
	      392      686      366      713      391      713
	      364      687      391     1764      443     1765
	      390

          
          name off
	     4362     4469      471     1710      448      630
	      447     1734      447     1737      468      608
	      509      570      472     1710      446      630
	      469      607      447     1735      448      630
	      470      606      447     1734      447     1737
	      446      629      448     1734      471      607
	      446     1734      448     1734      447     1735
	      446     1735      447      631      447     1733
	      447     1773      409     1734      447      631
	      445      632      470      608      446      632
	      446     1736      470      618      437      629
	      447     1735      446     1734      475     1711
	      444      631      446      632      446      633
	      446      631      458      621      444      634
	      446      636      445      625      447     1736
	      446     1734      471     1712      445     1735
	      446     1735      473     5282     4337     4494
	      446     1736      447      631      445     1736
	      470     1712      447      630      446      638
	      441     1734      446      631      445      631
	      447     1736      447      633      443      631
	      447     1735      455     1727      471      605
	      447     1735      446      631      447     1733
	      447     1735      447     1734      451     1730
	      447      632      450     1739      437     1735
	      447     1737      445      634      442      633
	      445      632      446      632      449     1732
	      446      630      448      629      471     1711
	      447     1734      446     1736      446      632
	      446      631      479      598      448      631
	      451      625      472      606      448      630
	      447      630      472     1711      446     1735
	      473     1708      471     1710      448     1733
	      473

      end raw_codes

end remote
```
文件的首尾内容固定，中间的name xxx部分，就是红外命令的编码。
以name on为例，他的编码就是上面在进行红外接收测试时，摁下on键后，收到的红外信号编码。将编码第一行的那个大数字删除，余下部分复制过来，并调整格式。
其他命令编码以此为例获取。得到完整的命令文件。
将这个文件复制到/etc/lirc/lircd.conf.d/目录下
```
sudo cp ./lircd.conf  /etc/lirc/lircd.conf.d/lircd.conf
```
重启lirc
```
sudo service lirc restart
```

##### 红外命令发送 #####
```
irsend SEND_ONCE aircon on
```

* 如果出现提示
```
irsend: could not connect to socket
irsend: Connection refused
```
运行
```
sudo lircd -d /dev/lirc0
```

* 如果出现
```
irsend: command failed: SEND_ONCE aircon on
irsend: unknown command: "on"
```
说明没有on这个name，仔细查看/etc/lirc/lircd.conf中的name们