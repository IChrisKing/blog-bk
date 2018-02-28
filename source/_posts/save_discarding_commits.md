---
title: 找回差點丟失的commit
category:
  - Linux
tags:
  - Linux
  - 填坑
date: 2015-02-04 11:23:40
description: "本地修改commit之後沒有及時push，然後執行了repo sync操作，導致本地修改被覆蓋，引起discarding commits時的補救方法"
---

## `git reflog`
```
612952f HEAD@{0}: checkout: moving from 612952fbfad9e480d5cd3486034399644393c253 to anrom-4.0
612952f HEAD@{1}: checkout: moving from a873309d7579ccceff13a1881bbc23740d62fe83 to 612952fbfad9e480d5cd3486034399644393c253
a873309 HEAD@{2}: commit: add new functions to install packages in different zone  --這一條就是差點丟失的commit，編號a873309
612952f HEAD@{3}: merge 612952fbfad9e480d5cd3486034399644393c253: Fast-forward
```
在其中找到被discarding的commit，他對應的id是a873309，後面會用到

## `git branch tmp a873309`

## `git branch -av`
```
* anrom-4.0               612952f fix destory activity save downloadinfo sttus
  tmp                     a873309 add new functions to install packages in different zone
  remotes/anrom/anrom-4.0 612952f fix destory activity save downloadinfo sttus
  remotes/m/anrom-4.0     -> anrom/anrom-4.0
```

## `git checkout tmp` 
切换到分支 'tmp'

## `git remote -v`
```
anrom http://anrom.ancode.org:81/p/anrom/packages/apps/AnromMarket (fetch)
anrom http://anrom.ancode.org:81/p/anrom/packages/apps/AnromMarket (push)
point ssh://chris@anrom.ancode.org:29418/anrom/packages/apps/AnromMarket.git (fetch)
point ssh://chris@anrom.ancode.org:29418/anrom/packages/apps/AnromMarket.git (push)
```

## `git log`
```
commit a873309d7579ccceff13a1881bbc23740d62fe83
Author: Chris King
Date:   Fri Jan 30 18:26:38 2015 +0800

    add new functions to install packages in different zone
    
    Change-Id: If42919eebd83723f8b3bff54a715c5ec2d49eb5a

commit 612952fbfad9e480d5cd3486034399644393c253
Author: NancyZhang <1049754324@qq.com>
Date:   Tue Dec 30 10:43:14 2014 +0800

    fix destory activity save downloadinfo sttus
```

## `git checkout anrom-4.0`
## `git cherry-pick  a873309d7579ccceff13a1881bbc23740d62fe83`
```
[anrom-4.0 bf79eec] add new functions to install packages in different zone
 6 个文件被修改，插入 390 行(+)，删除 194 行(-)
 create mode 100644 src/org/ancode/anrommarket/InstallTask.java
 create mode 100644 src/org/ancode/anrommarket/Installer.java
 create mode 100644 src/org/ancode/anrommarket/PackageInstallObserver.java
```

## `git branch -D tmp` 
已删除 分支 tmp（曾为 a873309）

## `git branch -av`
```
* anrom-4.0               bf79eec add new functions to install packages in different zone
  remotes/anrom/anrom-4.0 612952f fix destory activity save downloadinfo sttus
  remotes/m/anrom-4.0     -> anrom/anrom-4.0
```
  
## `git status`
```
# 位于分支 anrom-4.0
无须提交（干净的工作区）
```

## `git push point HEAD:refs/for/anrom-4.0`

```
Counting objects: 22, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (11/11), done.
Writing objects: 100% (13/13), 5.36 KiB, done.
Total 13 (delta 6), reused 0 (delta 0)
remote: Resolving deltas: 100% (6/6)
remote: Processing changes: new: 1, refs: 1, done    
remote: 
remote: New Changes:
remote:   http://anrom.ancode.org:81/4818
remote: 
To ssh://chris@anrom.ancode.org:29418/anrom/packages/apps/AnromMarket.git
 * [new branch]      HEAD -> refs/for/anrom-4.0
```