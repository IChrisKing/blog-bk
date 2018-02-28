---
title: Java基础知识
date: 2016-09-09 07:16:31
category:
	- Java
	- Android开发工程师
tags:
	- Android开发工程师
	- Android
	- Java
description: "零碎的java基础知识点"
---

## 八种基本数据类型的大小，以及他们的封装类
八种基本数据类型，int ,double ,long ,float, short,byte,character,boolean

对应的封装类型是：Integer ,Double ,Long ,Float, Short,Byte,Character,Boolean

## equals与==的区别
[http://www.importnew.com/6804.html](http://www.importnew.com/6804.html)

## Object
[Object有哪些公用方法](http://www.cnblogs.com/yumo/p/4908315.html)

## java的四种引用
[JAVA四种引用方式](http://blog.csdn.net/sbvfhp/article/details/43371487)

[Java的四种引用，强弱软虚，用到的场景](http://www.cnblogs.com/yumo/p/4908416.html)

## hashcode
[Hashcode的作用](http://c610367182.iteye.com/blog/1930676)

## String StringBuffer StringBuilder
[String,StringBuffer与StringBuilder的区别](http://blog.csdn.net/rmn190/article/details/1492013)

## Exception Error
[Excption与Error包结构。OOM你遇到过哪些情况，SOF你遇到过哪些情况](http://www.cnblogs.com/yumo/p/4909617.html)

## Java面向对象的三个特征与含义。

* 继承：继承是从已有类得到继承信息创建新类的过程。提供继承信息的类被称为父类（超类、基类）；得到继承信息的类被称为子类（派生类）。继承让变化中的软件系统有了一定的延续性，同时继承也是封装程序中可变因素的重要手段。

* 封装：通常认为封装是把数据和操作数据的方法绑定起来，对数据的访问只能通过已定义的接口。面向对象的本质就是将现实世界描绘成一系列完全自治、封闭的对象。我们在类中编写的方法就是对实现细节的一种封装；我们编写一个类就是对数据和数据操作的封装。可以说，封装就是隐藏一切可隐藏的东西，只向外界提供最简单的编程接口（可以想想普通洗衣机和全自动洗衣机的差别，明显全自动洗衣机封装更好因此操作起来更简单；我们现在使用的智能手机也是封装得足够好的，因为几个按键就搞定了所有的事情）。

* 多态：多态性是指允许不同子类型的对象对同一消息作出不同的响应。简单的说就是用同样的对象引用调用同样的方法但是做了不同的事情。多态性分为编译时的多态性和运行时的多态性。如果将对象的方法视为对象向外界提供的服务，那么运行时的多态性可以解释为：当A系统访问B系统提供的服务时，B系统有多种提供服务的方式，但一切对A系统来说都是透明的（就像电动剃须刀是A系统，它的供电系统是B系统，B系统可以使用电池供电或者用交流电，甚至还有可能是太阳能，A系统只会通过B类对象调用供电的方法，但并不知道供电系统的底层实现是什么，究竟通过何种方式获得了动力）。方法重载（overload）实现的是编译时的多态性（也称为前绑定），而方法重写（override）实现的是运行时的多态性（也称为后绑定）。运行时的多态是面向对象最精髓的东西，要实现多态需要做两件事：1. 方法重写（子类继承父类并重写父类中已有的或抽象的方法）；2. 对象造型（用父类型引用引用子类型对象，这样同样的引用调用同样的方法就会根据子类对象的不同而表现出不同的行为）。

## Override和Overload的含义与区别

Overload：顾名思义，就是Over(重新)——load（加载），所以中文名称是重载。它可以表现类的多态性，可以是函数里面可以有相同的函数名但是参数名、类型不能相同；或者说可以改变参数、类型但是函数名字依然不变。

Override：就是ride(重写)的意思，在子类继承父类的时候子类中可以定义某方法与其父类有相同的名称和参数，当子类在调用这一函数时自动调用子类的方法，而父类相当于被覆盖（重写）了。

方法的重写Overriding和重载Overloading是Java多态性的不同表现。重写Overriding是父类与子类之间多态性的一种表现，重载Overloading是一个类中多态性的一种表现。如果在子类中定义某方法与其父类有相同的名称和参数，我们说该方法被重写 (Overriding)。子类的对象使用这个方法时，将调用子类中的定义，对它而言，父类中的定义如同被“屏蔽”了。如果在一个类中定义了多个同名的方法，它们或有不同的参数个数或有不同的参数类型，则称为方法的重载(Overloading)。Overloaded的方法是可以改变返回值的类型。

## Interface与abstract类的区别
[abstract class和interface的区别](http://blog.csdn.net/b271737818/article/details/3950245)

## Static class 与non static class的区别
[static与non-static的区别](http://blog.csdn.net/luanxj/article/details/1451932)

## 反射机制
[ Java反射机制详解](http://blog.csdn.net/yongjian1092/article/details/7364451)

[Java动态代理详解](http://shensy.iteye.com/blog/1698197)

## String类内部实现
[String源码分析](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaSE/String%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)

[ Java中的String为什么是不可变的？ -- String源码分析](http://blog.csdn.net/zhangjg_blog/article/details/18319521)

## try catch 块，try里有return，finally也有return，如何执行
[关于Java中try-catch-finally-return的执行顺序](http://qing0991.blog.51cto.com/1640542/1387200)

## java泛型
[Java总结篇系列：Java泛型](http://www.cnblogs.com/lwbqqyumidi/p/3837629.html)

## 解析XML的几种方式的原理与特点：DOM、SAX、PULL。
[Java sax、dom、pull解析xml](http://www.cnblogs.com/HaroldTihan/p/4316397.html)

## java和c++的简单对比
[详细介绍JAVA和C++区别
](http://developer.51cto.com/art/201106/270422.htm)
