---
title: Java集合框架
date: 2016-09-18 20:57:31
category:
	- Java
	- Android开发工程师
tags:
	- Android开发工程师
	- Android
	- Java
description: "Java集合框架相关内容。"
---

## Java集合框架概述
[Java的集合框架最全详解（图）](http://doc.okbase.net/DavidIsOK/archive/94766.html)

[java集合框架的讲解](http://www.cnblogs.com/xiohao/p/4309462.html)

## ArrayList
[ArrayList简介](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaSE/ArrayList%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.md)

[深入Java集合学习系列：ArrayList的实现原理](http://zhangshixi.iteye.com/blog/674856)

动态数组，默认长度10，扩容时默认 原长度*3/2+1，不够的话，直接增加到需要的长度。扩容代价大，应尽量避免，所以，ArrayList适合预知要保存的元素数量时使用。

非线程安全

查找效率高，插入删除效率低

它允许所有元素，包括null

## LinkedList
[LinkedList简介](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaSE/LinkedList%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.md)

双向循环链表，可以当栈，队列，双端队列。
因为是链表，所以扩容无压力，无参的构造函数，会返回一个表头。

非线程安全

查找效率低，插入删除效率高

允许元素为null

## Vector
[Vector简介](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaSE/Vector%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.md)

与ArrayList相似

默认容量10

线程安全

允许元素为null

## HashMap
[HashMap简介](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaSE/HashMap%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.md)

[深入Java集合学习系列：HashMap的实现原理](http://zhangshixi.iteye.com/blog/672697)

HashMap是基于哈希表的Map接口的非同步实现。此实现提供所有可选的映射操作，并允许使用null值和null键。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。

默认的初始容量（容量为HashMap中槽的数目）是16，且实际容量必须是2的整数次幂

非线程安全

初始容量是2的n次方，负载因子（loadFactor）默认0.75

HashMap中key和value都允许为null

key为null的键值对永远都放在以table[0]为头结点的链表中，当然不一定是存放在头结点table[0]中

## HashTable
[Hashtable简介](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaSE/HashTable%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.md)

线程安全

默认容量为11

key和value都不允许为null

## LinkedHashMap
[LinkedHashMap简介](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaSE/LinkedHashMap%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.md)

HashMap的子类，与HashMap有着同样的存储结构，但它加入了一个双向链表的头结点，将所有put到LinkedHashmap的节点一一串成了一个双向循环链表，因此它保留了节点插入的顺序，可以使节点的输出顺序与输入顺序相同。

非线程安全

允许key和value为null

accessOrder标志位，当它false时，表示双向链表中的元素按照Entry插入LinkedHashMap到中的先后顺序排序，即每次put到LinkedHashMap中的Entry都放在双向链表的尾部，这样遍历双向链表时，Entry的输出顺序便和插入的顺序一致，这也是默认的双向链表的存储顺序；当它为true时，表示双向链表中的元素按照访问的先后顺序排列，可以看到，虽然Entry插入链表的顺序依然是按照其put到LinkedHashMap中的顺序，但put和get方法均有调用recordAccess方法（put方法在key相同，覆盖原有的Entry的情况下调用recordAccess方法），该方法判断accessOrder是否为true，如果是，则将当前访问的Entry（put进来的Entry或get出来的Entry）移到双向链表的尾部（key不相同时，put新Entry时，会调用addEntry，它会调用creatEntry，该方法同样将新插入的元素放入到双向链表的尾部，既符合插入的先后顺序，又符合访问的先后顺序，因为这时该Entry也被访问了），否则，什么也不做。
