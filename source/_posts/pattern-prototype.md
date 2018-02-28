---
title: Android中的设计模式之原型模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "介绍了原型模式，深克隆，浅克隆的区别。在Android系统的源码中，存在着很多的clone方法，这些方法实现了原型模式。"
date: 2016-11-22 19:10:58
---
## 简介
原型模式属于对象的创建模式。通过给出一个原型对象来指明所有创建的对象的类型，然后用复制这个原型对象的办法创建出更多同类型的对象。这就是选型模式的用意。

原型模式要求对象实现一个可以“克隆”自身的接口，这样就可以通过复制一个实例对象本身来创建一个新的实例。这样一来，通过原型实例创建新的对象，就不再需要关心这个实例本身的类型，只要实现了克隆自身的方法，就可以通过这个方法来获取新的对象，而无须再去通过new来创建。

### 原型模式的优点
　　原型模式允许在运行时动态改变具体的实现类型。原型模式可以在运行期间，由客户来注册符合原型接口的实现类型，也可以动态地改变具体的实现类型，看起来接口没有任何变化，但其实运行的已经是另外一个类实例了。因为克隆一个原型就类似于实例化一个类。

### 原型模式的缺点
　　原型模式最主要的缺点是每一个类都必须配备一个克隆方法。配备克隆方法需要对类的功能进行通盘考虑，这对于全新的类来说不是很难，而对于已经有的类不一定很容易，特别是当一个类引用不支持序列化的间接对象，或者引用含有循环结构的时候。

　　原型模式有两种表现形式：（1）简单形式、（2）登记形式，这两种表现形式仅仅是原型模式的不同实现。
[源码位置](http://www.cnblogs.com/java-my-life/archive/2012/04/11/2439387.html)

### 简单形式的原型模式
![image](/assets/img/prototype_pattern/simple.png
)
这种形式涉及到三个角色：

　　（1）客户(Client)角色：客户类提出创建对象的请求。

　　（2）抽象原型(Prototype)角色：这是一个抽象角色，通常由一个Java接口或Java抽象类实现。此角色给出所有的具体原型类所需的接口。

　　（3）具体原型（Concrete Prototype）角色：被复制的对象。此角色需要实现抽象的原型角色所要求的接口。

### 登记形式的原型模式
![image](/assets/img/prototype_pattern/manager.png
)
它多了一个原型管理器(PrototypeManager)角色，该角色的作用是：创建具体原型类的对象，并记录每一个被创建的对象。

原型管理器角色保持一个聚集，作为对所有原型对象的登记，这个角色提供必要的方法，供外界增加新的原型对象和取得已经登记过的原型对象。

### 两种形式的比较
　　简单形式和登记形式的原型模式各有其长处和短处。

　　如果需要创建的原型对象数目较少而且比较固定的话，可以采取第一种形式。在这种情况下，原型对象的引用可以由客户端自己保存。

　　如果要创建的原型对象数目不固定的话，可以采取第二种形式。在这种情况下，客户端不保存对原型对象的引用，这个任务被交给管理员对象。在复制一个原型对象之前，客户端可以查看管理员对象是否已经有一个满足要求的原型对象。如果有，可以直接从管理员类取得这个对象引用；如果没有，客户端就需要自行复制此原型对象。

## java中的克隆
　　Java的所有类都是从java.lang.Object类继承而来的，而Object类提供protected Object clone()方法对对象进行复制，子类当然也可以把这个方法置换掉，提供满足自己需要的复制方法。对象的复制有一个基本问题，就是对象通常都有对其他的对象的引用。当使用Object类的clone()方法来复制一个对象时，此对象对其他对象的引用也同时会被复制一份

　　Java语言提供的Cloneable接口只起一个作用，就是在运行时期通知Java虚拟机可以安全地在这个类上使用clone()方法。通过调用这个clone()方法可以得到一个对象的复制。由于Object类本身并不实现Cloneable接口，因此如果所考虑的类没有实现Cloneable接口时，调用clone()方法会抛出CloneNotSupportedException异常。

### 克隆满足的条件
　　clone()方法将对象复制了一份并返还给调用者。所谓“复制”的含义与clone()方法是怎么实现的。一般而言，clone()方法满足以下的描述：

　　（1）对任何的对象x，都有：x.clone()!=x。换言之，克隆对象与原对象不是同一个对象。

　　（2）对任何的对象x，都有：x.clone().getClass() == x.getClass()，换言之，克隆对象与原对象的类型一样。

　　（3）如果对象x的equals()方法定义其恰当的话，那么x.clone().equals(x)应当成立的。

　　在JAVA语言的API中，凡是提供了clone()方法的类，都满足上面的这些条件。JAVA语言的设计师在设计自己的clone()方法时，也应当遵守着三个条件。一般来说，上面的三个条件中的前两个是必需的，而第三个是可选的。

### 浅克隆
只负责克隆按值传递的数据（比如基本数据类型、String类型），而不复制它所引用的对象，换言之，所有的对其他对象的引用都仍然指向原来的对象。

### 深克隆

除了浅度克隆要克隆的值外，还负责克隆引用类型的数据。那些引用其他对象的变量将指向被复制过的新对象，而不再是原有的那些被引用的对象。换言之，深度克隆把要复制的对象所引用的对象都复制了一遍，而这种对被引用到的对象的复制叫做间接复制。

　　深度克隆要深入到多少层，是一个不易确定的问题。在决定以深度克隆的方式复制一个对象的时候，必须决定对间接复制的对象时采取浅度克隆还是继续采用深度克隆。因此，在采取深度克隆时，需要决定多深才算深。此外，在深度克隆的过程中，很可能会出现循环引用的问题，必须小心处理。

#### 利用序列化实现深度克隆
　　把对象写到流里的过程是序列化(Serialization)过程；而把对象从流中读出来的过程则叫反序列化(Deserialization)过程。应当指出的是，写到流里的是对象的一个拷贝，而原对象仍然存在于JVM里面。

　　在Java语言里深度克隆一个对象，常常可以先使对象实现Serializable接口，然后把对象（实际上只是对象的拷贝）写到一个流里（序列化），再从流里读回来（反序列化），便可以重建对象。

这样做的前提就是对象以及对象内部所有引用到的对象都是可序列化的，否则，就需要仔细考察那些不可序列化的对象可否设成transient，从而将之排除在复制过程之外。

　　浅度克隆显然比深度克隆更容易实现，因为Java语言的所有类都会继承一个clone()方法，而这个clone()方法所做的正式浅度克隆。

　　有一些对象，比如线程(Thread)对象或Socket对象，是不能简单复制或共享的。不管是使用浅度克隆还是深度克隆，只要涉及这样的间接对象，就必须把间接对象设成transient而不予复制；或者由程序自行创建出相当的同种对象，权且当做复制件使用。


## Android中的原型模式
Android中实现原型模式的例子非常多，很多的类都实现了clone()方法。有深克隆，也有浅克隆。

### Intent
/frameworks/base/core/java/android/content/Intent.java
```
    public Intent(Intent o) {
        this.mAction = o.mAction;
        this.mData = o.mData;
        this.mType = o.mType;
        this.mPackage = o.mPackage;
        this.mComponent = o.mComponent;
        this.mFlags = o.mFlags;
        this.mContentUserHint = o.mContentUserHint;
        if (o.mCategories != null) {
            this.mCategories = new ArraySet<String>(o.mCategories);
        }
        if (o.mExtras != null) {
            this.mExtras = new Bundle(o.mExtras);
        }
        if (o.mSourceBounds != null) {
            this.mSourceBounds = new Rect(o.mSourceBounds);
        }
        if (o.mSelector != null) {
            this.mSelector = new Intent(o.mSelector);
        }
        if (o.mClipData != null) {
            this.mClipData = new ClipData(o.mClipData);
        }
    }

    @Override
    public Object clone() {
        return new Intent(this);
    }
```
Intent的clone方法返回了一个新的Intent对象，查看Intent()方法的代码可以发现，这个克隆方法除了将原对象所有的值和引用直接赋给新对象之外，还将对象的引用对象也重新创建一遍赋给了新对象。所以，这是一个深克隆。

### Bundle
frameworks/base/core/java/android/os/Bundle.java
```
    public Object clone() {
        return new Bundle(this);
    }
    
        public Bundle(Bundle b) {
        super(b);

        mHasFds = b.mHasFds;
        mFdsKnown = b.mFdsKnown;
    }
```
这是一个浅拷贝。

### Transition
frameworks/base/core/java/android/transition/Transition.java
```
    public Transition clone() {
        Transition clone = null;
        try {
            clone = (Transition) super.clone();
            clone.mAnimators = new ArrayList<Animator>();
            clone.mStartValues = new TransitionValuesMaps();
            clone.mEndValues = new TransitionValuesMaps();
            clone.mStartValuesList = null;
            clone.mEndValuesList = null;
        } catch (CloneNotSupportedException e) {}

        return clone;
    }
```
这是一个深拷贝

### Notification
frameworks/base/core/java/android/app/Notification.java
```
    public Notification clone() {
        Notification that = new Notification();
        cloneInto(that, true);
        return that;
    }
    
        public Notification()
    {
        this.when = System.currentTimeMillis();
        this.priority = PRIORITY_DEFAULT;
    }
    
        public void cloneInto(Notification that, boolean heavy) {
        that.when = this.when;
        that.mSmallIcon = this.mSmallIcon;
        that.number = this.number;

        // PendingIntents are global, so there's no reason (or way) to clone them.
        that.contentIntent = this.contentIntent;
        that.deleteIntent = this.deleteIntent;
        that.fullScreenIntent = this.fullScreenIntent;

        if (this.tickerText != null) {
            that.tickerText = this.tickerText.toString();
        }
        if (heavy && this.tickerView != null) {
            that.tickerView = this.tickerView.clone();
        }
        if (heavy && this.contentView != null) {
            that.contentView = this.contentView.clone();
        }
        if (heavy && this.mLargeIcon != null) {
            that.mLargeIcon = this.mLargeIcon;
        }
        that.iconLevel = this.iconLevel;
        that.sound = this.sound; // android.net.Uri is immutable
        that.audioStreamType = this.audioStreamType;
        if (this.audioAttributes != null) {
            that.audioAttributes = new AudioAttributes.Builder(this.audioAttributes).build();
        }

        final long[] vibrate = this.vibrate;
        if (vibrate != null) {
            final int N = vibrate.length;
            final long[] vib = that.vibrate = new long[N];
            System.arraycopy(vibrate, 0, vib, 0, N);
        }

        that.ledARGB = this.ledARGB;
        that.ledOnMS = this.ledOnMS;
        that.ledOffMS = this.ledOffMS;
        that.defaults = this.defaults;

        that.flags = this.flags;

        that.priority = this.priority;

        that.category = this.category;

        that.mGroupKey = this.mGroupKey;

        that.mSortKey = this.mSortKey;

        if (this.extras != null) {
            try {
                that.extras = new Bundle(this.extras);
                // will unparcel
                that.extras.size();
            } catch (BadParcelableException e) {
                Log.e(TAG, "could not unparcel extras from notification: " + this, e);
                that.extras = null;
            }
        }

        if (this.actions != null) {
            that.actions = new Action[this.actions.length];
            for(int i=0; i<this.actions.length; i++) {
                that.actions[i] = this.actions[i].clone();
            }
        }

        if (heavy && this.bigContentView != null) {
            that.bigContentView = this.bigContentView.clone();
        }

        if (heavy && this.headsUpContentView != null) {
            that.headsUpContentView = this.headsUpContentView.clone();
        }

        that.visibility = this.visibility;

        if (this.publicVersion != null) {
            that.publicVersion = new Notification();
            this.publicVersion.cloneInto(that.publicVersion, heavy);
        }

        that.color = this.color;

        if (!heavy) {
            that.lightenPayload(); // will clean out extras
        }
    }
```
这是一个可选的拷贝过程。cloneInto方法有一个参数boolean heavy，可以决定是深克隆还是浅克隆。