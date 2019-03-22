---
title: 在树莓派上搭建私有git
category:
  - 树莓派
tags:
  - 树莓派raspberry
  - git
description: '如何在树莓派上搭建git，并远程clone，pull，push'
date: 2017-04-21 10:50:04
---

## 在树莓派上搭建git
### 准备工作
1. 安装ssh服务
```
sudo apt-get install ssh
```
2. 安装git服务
```
sudo apt-get install git-core
```

### 搭建git
#### 创建git用户
1. 新建用户
```
sudo adduser --system --shell /bin/bash --gecos 'git version control by pi' --group --home /home/git git
```
2. 修改passwd
```
sudo passwd git
```
3. 赋su权限
如果不赋su权限，会出现无法ssh 登录，无法git clone
```
sudo vim /etc/sudoers
添加一行
git ALL=(ALL) ALL
```
4. 切换到git
```
su git
```

#### 新建git项目
1. 进入git目录
```
cd /home/git
```
2. 创建git项目并初始化
```
mkdir test.git
cd test.git
git --bare init 
```
创建完成后，目录长这样
```
git@raspberrypi_chris:~/test.git$ ls
branches  config  description  HEAD  hooks  info  objects  refs
```
3. 新增远程主机
```
git remote add pi git@RASPBERRY'S IP:/home/git/test.git
```
其中RASPBERRY‘S IP是树莓派的ip地址，支持ip6。
查看一下：
```
git@raspberrypi_chris:~/test.git$ git remote
pi

git@raspberrypi_chris:~/test.git$ git remote show pi
git@pi-raspberry's password: 
* 远程 pi
  获取地址：git@pi-raspberry:/home/git/test.git
  推送地址：git@pi-raspberry:/home/git/test.git
  HEAD分支：master
  远程分支：
    master 新的（下一次获取将存储于 remotes/pi）
  为 'git push' 配置的本地引用：
    master 推送至 master (最新)
```

### 如何使用
git搭建完毕，接下来找一个别的机器，来使用它。都是常规的git操作，本人使用linux，所以windows小伙伴自行摸索。
#### clone项目
```
git clone git@PASPBERRY'S IP:/home/git/test.git
```
#### 与服务器同步
```
git pull
```
#### 对项目做修改并递交
```
chris@chris-office ~/respberry/git/test $ echo "test test test git" > test
chris@chris-office ~/respberry/git/test $ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   test

no changes added to commit (use "git add" and/or "git commit -a")
chris@chris-office ~/respberry/git/test $ git add test
chris@chris-office ~/respberry/git/test $ git commit -m "test git"
[master 1132719] test git
 1 file changed, 1 insertion(+), 1 deletion(-)
chris@chris-office ~/respberry/git/test $ git push origin master 
git@pi-raspberry's password: 
Counting objects: 3, done.
Writing objects: 100% (3/3), 250 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@pi-raspberry:/home/git/test.git
   c60ed7b..1132719  master -> master
```
