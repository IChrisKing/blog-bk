---
title: 使用火焰图（FlameGraph）来分析程序性能
category:
  - Linux
tags:
  - Linux
  - C/C++
description: "使用perf生成火焰图，并使用火焰图分析程序的运行状态，包括CPU使用状态等"
date: 2018-03-08 16:30:03
---
## 安装FlameGraph ##
```
wget https://github.com/brendangregg/FlameGraph/archive/master.zip
unzip master.zip
sudo mv FlameGraph-master/ /opt/FlameGraph
```
添加到环境变量 编辑/etc/profile，增加
```
#FlameGraph
export PATH=$PATH:/opt/FlameGraph
```
## 查找程序的pid ##
```
$ ps -aux|grep anet
nobody   31779  0.2  0.0  19208  4000 pts/37   S    15:35   0:10 /home/name/anet/cjdns/anet core /tmp client-core-puux8w0hdr7y5kdq9u12qqz7s7cgw5
```
pid为31779

## 生成CPU采样文件 ##
```
sudo perf record -F 99 -p 31779 -g -o in-fb.data -- sleep 60
sudo perf script -i in-fb.data > in-fb.perf
```
首先使用99HZ的采样频率，对pid为31779的进程进行采样，采样输出到in-fb.data中，采样时长为60秒

## 生成CPU火焰图 ##
```
stackcollapse-perf.pl in-fb.perf >in-fb.folded
flamegraph.pl in-fb.folded >in-fb-cpu.svg
```
![火焰图](/assets/img/FlameGraph/in-fb-cpu.svg)
火焰图可以使用浏览器来打开

## 一个简单的脚本 ##
```
#!/bin/sh
DIR=/your-path/
sudo perf record -F 99 -p $1 -g -o $DIR/cpu.data -- sleep $2
sudo perf script -i $DIR/cpu.data > $DIR/cpu.perf
stackcollapse-perf.pl $DIR/cpu.perf > $DIR/cpu.folded
flamegraph.pl $DIR/cpu.folded > $DIR/cpu1.svg
```
使用方法：对进程1234采样60秒
```
bash perf 1234 60
```
