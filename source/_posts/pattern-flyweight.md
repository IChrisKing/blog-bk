---
title: Android中的设计模式之享元模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "介绍了享元模式的定义，给出了简单享元模式和复合享元模式的例子。Android中享元模式的运用主要突出的时其高效利用细粒度对象的特点。"
date: 2016-11-10 10:52:58
---
## 简介
享元模式是对象的结构模式。享元模式以共享的方式高效地支持大量的细粒度对象。

在JAVA语言中，String类型就是使用了享元模式。String对象是final类型，对象一旦创建就不可改变。在JAVA中字符串常量都是存在常量池中的，JAVA会确保一个字符串常量在常量池中只有一个拷贝。

享元模式采用一个共享来避免大量拥有相同内容对象的开销。这种开销最常见、最直观的就是内存的损耗。享元对象能做到共享的关键是区分内蕴状态(Internal State)和外蕴状态(External State)。

　　一个内蕴状态是存储在享元对象内部的，并且是不会随环境的改变而有所不同。因此，一个享元可以具有内蕴状态并可以共享。

　　一个外蕴状态是随环境的改变而改变的、不可以共享的。享元对象的外蕴状态必须由客户端保存，并在享元对象被创建之后，在需要使用的时候再传入到享元对象内部。外蕴状态不可以影响享元对象的内蕴状态，它们是相互独立的。

### 单纯享元模式
　在单纯的享元模式中，所有的享元对象都是可以共享的。
![image](/assets/img/flyweight_pattern/simple.png)
　　单纯享元模式所涉及到的角色如下：

　　●　　抽象享元(Flyweight)角色 ：给出一个抽象接口，以规定出所有具体享元角色需要实现的方法。

　　●　　具体享元(ConcreteFlyweight)角色：实现抽象享元角色所规定出的接口。如果有内蕴状态的话，必须负责为内蕴状态提供存储空间。

　　●　　享元工厂(FlyweightFactory)角色 ：本角色负责创建和管理享元角色。本角色必须保证享元对象可以被系统适当地共享。当一个客户端对象调用一个享元对象的时候，享元工厂角色会检查系统中是否已经有一个符合要求的享元对象。如果已经有了，享元工厂角色就应当提供这个已有的享元对象；如果系统中没有一个适当的享元对象的话，享元工厂角色就应当创建一个合适的享元对象。

[源码查看](http://www.cnblogs.com/java-my-life/archive/2012/04/26/2468499.html)

### 复合享元模式
在单纯享元模式中，所有的享元对象都是单纯享元对象，也就是说都是可以直接共享的。还有一种较为复杂的情况，将一些单纯享元使用合成模式加以复合，形成复合享元对象。这样的复合享元对象本身不能共享，但是它们可以分解成单纯享元对象，而后者则可以共享。
![image](/assets/img/flyweight_pattern/complex.png)
复合享元角色所涉及到的角色如下：

　　●　　抽象享元(Flyweight)角色 ：给出一个抽象接口，以规定出所有具体享元角色需要实现的方法。

　　●　　具体享元(ConcreteFlyweight)角色：实现抽象享元角色所规定出的接口。如果有内蕴状态的话，必须负责为内蕴状态提供存储空间。

　　●　  复合享元(ConcreteCompositeFlyweight)角色 ：复合享元角色所代表的对象是不可以共享的，但是一个复合享元对象可以分解成为多个本身是单纯享元对象的组合。复合享元角色又称作不可共享的享元对象。

　　●　  享元工厂(FlyweightFactory)角色 ：本角 色负责创建和管理享元角色。本角色必须保证享元对象可以被系统适当地共享。当一个客户端对象调用一个享元对象的时候，享元工厂角色会检查系统中是否已经有 一个符合要求的享元对象。如果已经有了，享元工厂角色就应当提供这个已有的享元对象；如果系统中没有一个适当的享元对象的话，享元工厂角色就应当创建一个 合适的享元对象。

### 享元的优缺点
享元模式的优缺点
　　享元模式的优点在于它大幅度地降低内存中对象的数量。但是，它做到这一点所付出的代价也是很高的：

　　●　　享元模式使得系统更加复杂。为了使对象可以共享，需要将一些状态外部化，这使得程序的逻辑复杂化。

　　●　　享元模式将享元对象的状态外部化，而读取外部状态使得运行时间稍微变长。

## Android中的享元模式
享元模式是一种针对大量细粒度对象有效使用的一种模式。享元模式在Android中的运用过程中明显剥离和简化了享元工厂这一部分。主要突出的是享元对对象复用的特点。

Android中的Message、Parcel和TypedArray都利用了享元模式。以Message为例，Message中的obtain方法和recycle方法共同维护一个message Pool实现了message对象的复用。

Message通过next成员变量保有对下一个Message的引用，从而构成了一个Message链表。Message Pool就通过该链表的表头管理着所有闲置的Message，一个Message在使用完后可以通过recycle()方法进入Message Pool，并在需要时通过obtain静态方法从Message Pool获取。
```
public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

 void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```