---
title: Android中的设计模式之装饰者模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "装饰模式以对客户端透明的方式扩展对象的功能，是继承关系的一个替代方案。Android中的ContextWrapper的设计，是对装饰者模式的一个经典用例。"
date: 2016-12-20 19:19:47
---
## 简介
装饰模式又名包装(Wrapper)模式。装饰模式以对客户端透明的方式扩展对象的功能，是继承关系的一个替代方案。

装饰模式以对客户透明的方式动态地给一个对象附加上更多的责任。换言之，客户端并不会觉得对象在装饰前和装饰后有什么不同。装饰模式可以在不使用创造更多子类的情况下，将对象的功能加以扩展。
![image](/assets/img/pattern-decorator/decorator.png)
　在装饰模式中的角色有：

　　●　　抽象构件(Component)角色：给出一个抽象接口，以规范准备接收附加责任的对象。

　　●　　具体构件(ConcreteComponent)角色：定义一个将要接收附加责任的类。

　　●　　装饰(Decorator)角色：持有一个构件(Component)对象的实例，并定义一个与抽象构件接口一致的接口。

　　●　　具体装饰(ConcreteDecorator)角色：负责给构件对象“贴上”附加的责任。

[java中的装饰模式](http://www.cnblogs.com/java-my-life/archive/2012/04/20/2455726.html)

## 装饰者模式的优点
装饰者模式实现的是从一个对象外部给对象添加功能，相当于改变了对象的外观，装饰过的对象，从外部系统来看已经不再是原来的对象，而是经过一系列装饰器装饰过的对象。
装饰者模式最大的好处就是灵活，它能够灵活的改变一个的对象的功能，并且是动态组合形式。另外好处就是代码复用，因为每个装饰器是独立的，可以给一个对象多次增加同一个装饰器，也可以同一个装饰器装饰不同对象。
在面向对象设计中：有一条基本规则

尽量使用对象组合，而不是对象继承来扩展和复用功能。

另外说明：

各个装饰器之间最好是完全独立的功能，不要依赖，这样在进行装饰组合的时候，才没有先后调用限制。否则会降低装饰器组合的灵活性。

装饰器模式的退化形式：
当仅仅只是添加一个功能，就没有必要再设计装饰器的抽象父类，直接在装饰器中实现组件接口，然后实现相应的拓展功能。

## 一个例子理解装饰者模式
[一个例子理解装饰者模式](http://www.2cto.com/kf/201512/455764.html)

## Android中的装饰者模式
Android中的装饰者模式，最经典的就是由Context抽象类扩展出的ContextWrapper的设计。
![image](/assets/img/pattern-decorator/context.png)

1. Context就是抽象构件，它提供了应用运行的基本环境，是各组件和系统服务通信的桥梁，隐藏了应用与系统服务通信的细节，简化了上层应用的开发。所以Contex就是“装饰模式”里的Component。
2. Context类是个抽象类，android.app.ContextImpl派生实现了它的抽象接口。ContextImpl对象会与Android框架层的各个服务（包括组件管理服务、资源管理服务、安装管理服务等）建立远程连接，通过对Android进程间的通信机制（IPC）和这些服务进行通信。所以ContextImpl就是“装饰模式”里的ConcreteComponent。

3. 如果上层应用期望改变Context接口的实现，就需要使用android.content.ContextWrapper类，它派生自Context，其中具体实现都是通过组合的方式调用ContextImpl类的实例（在ContextWrapper中的private属性mBase）来完成的。这样的设计，使得ContextImpl与ContextWrapper子类的实现可以单独变化，彼此独立。所以可以看出ContextWrapper就是“装饰模式”里的Decorator。

4. Android的界面组件Activity、服务组件Service以及应用基类Application都派生于ContextWrapper，它们可以通过重载来修改Context接口的实现。所以可以看出Activity、服务组件Service以及应用基类Application就是“装饰模式”里的具体装饰角色A、B、C。


[设计模式-装饰者模式（Decorator）理解和在Android中的应用](http://blog.csdn.net/card361401376/article/details/51222351)

详细的代码分析可以参考[Android源码学习之装饰模式应用](http://www.cnblogs.com/yemeishu/archive/2012/12/30/2839489.html)
