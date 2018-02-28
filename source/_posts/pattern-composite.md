---
title: Android中的设计模式之组合模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "介绍了组合模式，然后给出在Android源码中使用组合模式的例子，最经典的就是View和ViewGroup的组合。几乎所有在/frameworks/base/core/java/android/widget/目录下的类和布局类都会用到这对组合。"
date: 2016-11-04 19:33:32
---
## 简介
合成模式属于结构模式，有时又叫做“部分——整体”模式或者合成模式。组合模式将对象组织到树结构中，可以用来描述整体与部分的关系。组合模式可以使客户端将单纯元素与复合元素同等看待。

文件系统是一个典型的组合模式。它是一个树形的结构，树的节点有两种：一种是树枝节点，即目录，目录中还有文件，也就是该节点还有子树。一种是树叶节点，即文件，没有子树。

显然，可以把目录和文件当做同一种对象同等对待和处理，这也就是合成模式的应用。

合成模式可以不提供父对象的管理方法，但是合成模式必须在合适的地方提供子对象的管理方法，诸如：add()、remove()、以及getChild()等。

合成模式的实现根据所实现接口的区别分为两种形式，分别称为安全式和透明式。

具体实现代码可参考[《JAVA与模式》之合成模式](http://www.cnblogs.com/java-my-life/archive/2012/04/17/2453861.html)
### 安全式
安全模式的合成模式要求管理聚集的方法只出现在树枝构件类中，而不出现在树叶构件类中。

安全式合成模式的缺点是不够透明，因为树叶类和树枝类将具有不同的接口.

![image](/assets/img/composite_pattern/safe.png)
这种形式涉及到三个角色：

　　●　　抽象构件(Component)角色：这是一个抽象角色，它给参加组合的对象定义出公共的接口及其默认行为，可以用来管理所有的子对象。合成对象通常把它所包含的子对象当做类型为Component的对象。在安全式的合成模式里，构件角色并不定义出管理子对象的方法，这一定义由树枝构件对象给出。

　　●　　树叶构件(Leaf)角色：树叶对象是没有下级子对象的对象，定义出参加组合的原始对象的行为。

　　●　　树枝构件(Composite)角色：代表参加组合的有下级子对象的对象。树枝构件类给出所有的管理子对象的方法，如add()、remove()以及getChild()。

### 透明模式
透明式的合成模式要求所有的具体构件类，不论树枝构件还是树叶构件，均符合一个固定接口。

从客户端使用合成模式上，是否需要区分到底是“树枝对象”还是“树叶对象”。如果是透明的，那就不用区分，对于客户而言，都是Compoent对象，具体的类型对于客户端而言是透明的，是无须关心的。

![image](/assets/img/composite_pattern/transparent.png)

## Android中的组合模式
Android中的组合模式，最经典的就是View和ViewGroup类的使用。在android UI设计，几乎所有的widget和布局类都依靠这两个类。

View和ViewGroup类使用的时安全式的组合模式。
![image](/assets/img/composite_pattern/android.jpg)
上图是View和ViewGroup的大致架构。

可以看到，ViewGroup继承自View，作为树枝节点，它实现了addView(),removeView(),getChild(int)等管理聚集的方法。而Button这样的叶子节点则没有管理局及的方法（虽然Button不是直接继承自View，而是通过了TextView，间接继承自View）。这样的架构，正是安全式的组合模式。