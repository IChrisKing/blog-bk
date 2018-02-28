---
title: Android中的设计模式之备忘录模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "备忘录模式在Android源码中的运用还有待继续确认和补充"
date: 2016-11-09 17:09:58
---
## 简介
备忘录模式又叫做快照模式(Snapshot Pattern)或Token模式，是对象的行为模式。

　　备忘录对象是一个用来存储另外一个对象内部状态的快照的对象。备忘录模式的用意是在不破坏封装的条件下，将一个对象的状态捕捉(Capture)住，并外部化，存储起来，从而可以在将来合适的时候把这个对象还原到存储起来的状态。备忘录模式常常与命令模式和迭代子模式一同使用。

![image](/assets/img/memento_pattern/memento.png)
备忘录模式所涉及的角色有三个：备忘录(Memento)角色、发起人(Originator)角色、负责人(Caretaker)角色。

### 备忘录(Memento)角色

　　备忘录角色又如下责任：

　　（1）将发起人（Originator）对象的内战状态存储起来。备忘录可以根据发起人对象的判断来决定存储多少发起人（Originator）对象的内部状态。

　　（2）备忘录可以保护其内容不被发起人（Originator）对象之外的任何对象所读取。

　　备忘录有两个等效的接口：

　　●　　窄接口：负责人（Caretaker）对象（和其他除发起人对象之外的任何对象）看到的是备忘录的窄接口(narrow interface)，这个窄接口只允许它把备忘录对象传给其他的对象。

　　●　　宽接口：与负责人对象看到的窄接口相反的是，发起人对象可以看到一个宽接口(wide interface)，这个宽接口允许它读取所有的数据，以便根据这些数据恢复这个发起人对象的内部状态。

### 发起人（Originator）角色

　　发起人角色有如下责任：

　　（1）创建一个含有当前的内部状态的备忘录对象。

　　（2）使用备忘录对象存储其内部状态。

### 负责人（Caretaker）角色

　　负责人角色有如下责任：

　　（1）负责保存备忘录对象。

　　（2）不检查备忘录对象的内容。

## 白箱模式和黑箱模式
代码查看：[http://www.cnblogs.com/java-my-life/archive/2012/06/06/2534942.html](http://www.cnblogs.com/java-my-life/archive/2012/06/06/2534942.html)
### 白箱模式
备忘录角色对任何对象都提供一个接口，即宽接口，备忘录角色的内部所存储的状态就对所有对象公开。因此这个实现又叫做“白箱实现”。

　　“白箱”实现将发起人角色的状态存储在一个大家都看得到的地方，因此是破坏封装性的。但是通过程序员自律，同样可以在一定程度上实现模式的大部分用意。因此白箱实现仍然是有意义的。
![image](/assets/img/memento_pattern/white.png)

### 黑箱模式
　备忘录角色对发起人（Originator）角色对象提供一个宽接口，而为其他对象提供一个窄接口。这样的实现叫做“黑箱实现”。

　　在JAVA语言中，实现双重接口的办法就是将备忘录角色类设计成发起人角色类的内部成员类。

　　将Memento设成Originator类的内部类，从而将Memento对象封装在Originator里面；在外部提供一个标识接口MementoIF给Caretaker以及其他对象。这样，Originator类看到的是Menmento的所有接口，而Caretaker以及其他对象看到的仅仅是标识接口MementoIF所暴露出来的接口。
![image](/assets/img/memento_pattern/black.png)

## Android中的备忘录模式
在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态，这样以后就可以将该对象恢复到先前保存的状态。

Android中的Canvas是一个黑箱的备忘录模式。组织者，把原发器的状态State（全部或者部分状态，一般是变量的值），通过CreateMemento()方法保存起来，继续运行后，等待合适的时机，在通过SetMemento()方法可以再次恢复到之前的状态。在这个过程中，我们并没有对这些状态做任何的访问和设置，实际上这些状态都是私有的，对外是禁止访问的，我们只是通过Memento对象的两个最简单的方法就达到了这个效果。Memento经常写成Originator的内部类。

查看他的源码：
```
    public int save() {
        return native_save(mNativeCanvasWrapper, MATRIX_SAVE_FLAG | CLIP_SAVE_FLAG);
    }
    
    public int save(@Saveflags int saveFlags) {
        return native_save(mNativeCanvasWrapper, saveFlags);
    }
    
        public void restore() {
        boolean throwOnUnderflow = !sCompatibilityRestore || !isHardwareAccelerated();
        native_restore(mNativeCanvasWrapper, throwOnUnderflow);
    }
```
　　从上面的两个方法中，它们实现了自我状态的恢复，实际上我们只是执行了两个没有接触任何内部信息的方法，实际上这两个方法就是在操作我们看不到的这些内部状态信息。