---
title: 如何解决报错“prior sync failed; rebase still in progress”
category:
  - AndroidRom
tags:
  - rom
  - git
description: “如何解决在repo sync过程中遇到的prior sync failed; rebase still in progress错误。引起这类错误的原因可能是该项目由自己维护，同时又要与其他分支保持同步，所以在项目下执行了merge操作。而repo sync时，项目指向的位置并不是自己同步的那个。”
date: 2017-12-04 13:19:46
---
## 问题描述 ##
在android系统的源码中，有一些项目需要自己维护，所以添加了新的remote和branch。
但是在.repo中，这些项目的源还在原本的remote上，当执行repo sync时，会出现报错
```
prior sync failed; rebase still in progress
```
通常情况下，网上能找到的解决方法是
```
git rebase --abort
```
然而这并没有用
删除这个项目再执行repo sync有时是管用的。但是如果遇上例如frameworks/base这个巨大的项目，根本舍不得删除，再牵出一次耗时太久，成功率还低

## 解决方法 ##
在出问题的项目中，（比如frameworks/base，）新建一个符合repo中remote的branch
查看.repo/manifests/default.xml
```
  <project path="frameworks/base" name="MoKee/android_frameworks_base" groups="pdk-cw-fs,pdk-fs" />

```
可见这是一个mokee的项目
进入frameworks/base项目下
```
$ git branch -av
* （非分支，正变基 dev）   bc9e0ec Merge branch 'remote' into mkn-mr1
  remotes/anrom/dev           fcff150 Merge branch 'dev' of github.sbu:ROM/android_frameworks_base into dev
  remotes/anrom/mkn-mr1       cc29ef0 [Xposed] Add XposedBridge.removeFinalFlagNative()
  remotes/m/mkn-mr1           -> mokee/mkn-mr1
  remotes/mokee/jb-mr1_mkt    139c903 code cleanup and update license
  remotes/mokee/jb-mr2_mkt    c72c0b3 Merge branch 'cm-10.2' into martincz
  remotes/mokee/jb_mkt        f6abc38 fix 24hours bind bug
  remotes/mokee/kk_mkt        ea794d2 Merge branch into kk_mkt
  remotes/mokee/mkl           1032156 Update Translations
  remotes/mokee/mkl-mr1       2ab30d1 Merge branch 'remote' into mkl-mr1
  remotes/mokee/mkl-mr1-viper e833fe9 Merge branch 'remote' into mkl-mr1
  remotes/mokee/mkm           e52c73e Automatic translation import
  remotes/mokee/mkn-mr1       35e8c9b Automatic translation import
  remotes/mokee/mko           3bac848 base: mokee cloud interface
```
可以看到，这个正变基就是出问题的branch。
解决问题需要做的操作是，新建一个mkn-mr1分支。upstream设置为mokee/mkn-mr1.这样，以后执行repo sync时，会往这个分支同步。至于自己使用的分支，后期再自行手动merge。

进入frameworks/base，执行以下操作：
```
git checkout -f remotes/mokee/mkn-mr1
git branch -b mkn-mr1
git branch --set-upstream-to=mokee/mkn-mr1 
```
之后，再执行repo sync时，不会再报错了