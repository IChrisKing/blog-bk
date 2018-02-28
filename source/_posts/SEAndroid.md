---
title: SEAndroid in Android5.x
category:
  - SEAndroid
tags:
  - SEAndroid
  - SELinux
date: 2015-12-01 16:39:15
description: "分析Android5.x源碼中，SEAndroid的部分，原來寫在新浪博客，現在把鏈接整理一下"
---
## SEAndroid in Android5.x
分析Adroid 5.x源碼中的SEAndroid部分，提出一些修改方法
[概述](http://blog.sina.com.cn/s/blog_67521b8c0102wp7s.html)

[用户和角色](http://blog.sina.com.cn/s/blog_67521b8c0102wp82.html)

[文件的安全上下文](http://blog.sina.com.cn/s/blog_67521b8c0102wpo1.html)

[进程的安全上下文](http://blog.sina.com.cn/s/blog_67521b8c0102wpoj.html)

[设置文件安全上下文的代码实现](http://blog.sina.com.cn/s/blog_67521b8c0102wpv6.html)

[安全策略的生成](http://blog.sina.com.cn/s/blog_67521b8c0102wq57.html)

[设置进程安全上下文的代码实现](http://blog.sina.com.cn/s/blog_67521b8c0102wqvj.html)

[对属性的保护](http://blog.sina.com.cn/s/blog_67521b8c0102ws62.html)

## SEAndroid Policy分析
對於SEAndroid策略的分析，基於andriod4.x系統，不過策略的格式在android5.x以及後續的版本中都沒有變化，所以對於策略的分析依然適用於android後續的版本

[SEAndroid策略分析（一）：概述SEAndroid](http://blog.sina.com.cn/s/blog_67521b8c01017iz7.html)

[SEAndroid策略分析（二）：类和许可](http://blog.sina.com.cn/s/blog_67521b8c01017izg.html)

[SEAndroid策略分析（三）：类型强制和角色声明](http://blog.sina.com.cn/s/blog_67521b8c01017izj.html)

