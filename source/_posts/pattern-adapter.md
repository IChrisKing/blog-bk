---
title: Android中的设计模式之适配器模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "介绍了适配器模式。然后以android中的ListAdapter为例，介绍了适配器模式在android中的运用。"
date: 2016-11-21 19:03:26
---
## 简介
适配器模式把一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口不匹配而无法在一起工作的两个类能够在一起工作。

### 模式中的角色
需要适配的类（Adaptee）：需要适配的类。

适配器（Adapter）：通过包装一个需要适配的对象，把原接口转换成目标接口。

目标接口（Target）：客户所期待的接口。可以是具体的或抽象的类，也可以是接口。

### 类的适配器模式
类的适配器模式把适配的类的API转换成为目标类的API。

![image](/assets/img/adapter_pattern/adapter_class.png)
在上图中可以看出，Adaptee类并没有sampleOperation2()方法，而客户端则期待这个方法。为使客户端能够使用Adaptee类，提供一个中间环节，即类Adapter，把Adaptee的API与Target类的API衔接起来。Adapter与Adaptee是继承关系，这决定了这个适配器模式是类的：

　　模式所涉及的角色有：

　　●　　目标(Target)角色：这就是所期待得到的接口。注意：由于这里讨论的是类适配器模式，因此目标不可以是类。

　　●　　源(Adapee)角色：现在需要适配的接口。

　　●　　适配器(Adaper)角色：适配器类是本模式的核心。适配器把源接口转换成目标接口。显然，这一角色不可以是接口，而必须是具体类。

[源代码](http://www.cnblogs.com/java-my-life/archive/2012/04/13/2442795.html)

### 对象的适配器模式
与类的适配器模式一样，对象的适配器模式把被适配的类的API转换成为目标类的API，与类的适配器模式不同的是，对象的适配器模式不是使用继承关系连接到Adaptee类，而是使用委派关系连接到Adaptee类。

![image](/assets/img/adapter_pattern/adapter_object.png)

　　从上图可以看出，Adaptee类并没有sampleOperation2()方法，而客户端则期待这个方法。为使客户端能够使用Adaptee类，需要提供一个包装(Wrapper)类Adapter。这个包装类包装了一个Adaptee的实例，从而此包装类能够把Adaptee的API与Target类的API衔接起来。Adapter与Adaptee是委派关系，这决定了适配器模式是对象的。

### 缺省适配模式
在很多情况下，必须让一个具体类实现某一个接口，但是这个类又用不到接口所规定的所有的方法。通常的处理方法是，这个具体类要实现所有的方法，那些有用的方法要有实现，那些没有用的方法也要有空的、平庸的实现。

　　这些空的方法是一种浪费，有时也是一种混乱。除非看过这些空方法的代码，程序员可能会以为这些方法不是空的。即便他知道其中有一些方法是空的，也不一定知道哪些方法是空的，哪些方法不是空的，除非看过这些方法的源代码或是文档。

　　缺省适配模式可以很好的处理这一情况。可以设计一个抽象的适配器类实现接口，此抽象类要给接口所要求的每一种方法都提供一个空的方法。此抽象类可以使它的具体子类免于被迫实现空的方法。

![image](/assets/img/adapter_pattern/adapter_empty.png)

可以看到，接口AbstractService要求定义三个方法，分别是serviceOperation1()、serviceOperation2()、serviceOperation3()；而抽象适配器类ServiceAdapter则为这三种方法都提供了平庸的实现。因此，任何继承自抽象类ServiceAdapter的具体类都可以选择它所需要的方法实现，而不必理会其他的不需要的方法。

　　适配器模式的用意是要改变源的接口，以便于目标接口相容。缺省适配的用意稍有不同，它是为了方便建立一个不平庸的适配器类而提供的一种平庸实现。

　　在任何时候，如果不准备实现一个接口的所有方法时，就可以使用“缺省适配模式”制造一个抽象类，给出所有方法的平庸的具体实现。这样，从这个抽象类再继承下去的子类就不必实现所有的方法了。

## Android中的适配器模式
![image](/assets/img/adapter_pattern/adapter_android.png)
以ListView为例，它需要的目标接口就是ListAdapter，继承自Adapter，其中包含一些基本方法，getCount,getView,getItem，getItemId等。

数据源包括List<T>,Cursor等，为了兼容这些不同类型的数据源，需要定义不同的适配器：ArrayAdapter，CursorAdapter。这些adapter间接继承自ListAdapter，满足了ListView的需求。将数据以合适的方式提供给目标。

以ArrayAdapter为例，查看一下源代码：
### Adapter
frameworks/base/core/java/android/widget/Adapter.java
```
public interface Adapter {
    .....
    
    int getCount();   

    Object getItem(int position);

    long getItemId(int position);
    
    .......
}
```

### ListAdapter
/frameworks/base/core/java/android/widget/ListAdapter.java
```
public interface ListAdapter extends Adapter {
......
    boolean isEnabled(int position);
}
```

### BaseAdapter
/frameworks/base/core/java/android/widget/BaseAdapter.java
```
public abstract class BaseAdapter implements ListAdapter, SpinnerAdapter {
    ......
    
    public void notifyDataSetChanged() {
        mDataSetObservable.notifyChanged();
    }
    
    ......
    
}
```

### ArrayAdapter
/frameworks/base/core/java/android/widget/ArrayAdapter.java
```
public class ArrayAdapter<T> extends BaseAdapter implements Filterable, ThemedSpinnerAdapter {
....
private List<T> mObjects;
.....

    @Override
    public void notifyDataSetChanged() {
        super.notifyDataSetChanged();
        mNotifyOnChange = true;
    }
    
    .....
    
        public int getCount() {
        return mObjects.size();
    }

    public T getItem(int position) {
        return mObjects.get(position);
    }

    public long getItemId(int position) {
        return position;
    }
    
    ......
}
```