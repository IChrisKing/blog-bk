---
title: 对我的Blog做了点改动
date: 2016-07-24 21:35:36
category: hexo
tags:
	- hexo
description: "对blog的一些修改"
---

### 将整个blog项目的源码递交到github ###
修改了blog_backup的目录结构，本来只有source一个目录，现在相当于回退一级，将整个myblog项目都递交到github上

因为之前在source目录中已经有项目的.git，所以，直接将它拷贝到myblog下，然后直接递交了。

后来试了一下从头递交的流程，已经更新之前的使用hexo搭建blog那片文章里面

### 修改theme为maupassant ###
看了一天的各种theme，挑了一个好看的，yilia其实很好看，然则不知道为什么，头像就是无法设置，本地hexo server都能显示正常，一递交到网络，直接访问blog，就不显示头像，想了很多方法，找了资料，没有解决，还是不浪费时间了。

换了maupassant主题，看着还不错。

```
$ git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
$ npm install hexo-renderer-jade --save
$ npm install hexo-renderer-sass --save
```
这两行npm install是个坑，一不小心忘了执行，结果搞坏了代码，修复了好几个小时阿。。。

幸好rom项目的代码没有牵扯到换theme，幸免于难 
