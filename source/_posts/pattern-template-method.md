---
title: Android中的设计模式之模板方法模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "简单的介绍了模板方法的定义。在Android中运用模板方法模式的一个典型例子就是view和它的子类们，onDraw,onLayout等方法都是父类中定义钩子函数，然后在子类中进行实现。"
date: 2016-11-08 20:32:37
---
## 简介
模板方法模式是类的行为模式。准备一个抽象类，将部分逻辑以具体方法以及具体构造函数的形式实现，然后声明一些抽象方法来迫使子类实现剩余的逻辑。不同的子类可以以不同的方式实现这些抽象方法，从而对剩余的逻辑有不同的实现。这就是模板方法模式的用意。

　模板模式的关键是：子类可以置换掉父类的可变部分，但是子类却不可以改变模板方法所代表的顶级逻辑。

![image](/assets/img/template_method_pattern/template_method.png)
这里涉及到两个角色：

　　抽象模板(Abstract Template)角色有如下责任：

　　■　　定义了一个或多个抽象操作，以便让子类实现。这些抽象操作叫做基本操作，它们是一个顶级逻辑的组成步骤。

　　■　　定义并实现了一个模板方法。这个模板方法一般是一个具体方法，它给出了一个顶级逻辑的骨架，而逻辑的组成步骤在相应的抽象操作中，推迟到子类实现。顶级逻辑也有可能调用一些具体方法。

　　具体模板(Concrete Template)角色又如下责任：

　　■　　实现父类所定义的一个或多个抽象方法，它们是一个顶级逻辑的组成步骤。

　　■　　每一个抽象模板角色都可以有任意多个具体模板角色与之对应，而每一个具体模板角色都可以给出这些抽象方法（也就是顶级逻辑的组成步骤）的不同实现，从而使得顶级逻辑的实现各不相同。

代码实现参考：[《JAVA与模式》之模板方法模式](http://www.cnblogs.com/java-my-life/archive/2012/05/14/2495235.html)

### 模板方法模式中的方法
　　模板方法中的方法可以分为两大类：模板方法和基本方法。

#### 模板方法
　　一个模板方法是定义在抽象类中的，把基本操作方法组合在一起形成一个总算法或一个总行为的方法。

　　一个抽象类可以有任意多个模板方法，而不限于一个。每一个模板方法都可以调用任意多个具体方法。

#### 基本方法
　　基本方法又可以分为三种：抽象方法(Abstract Method)、具体方法(Concrete Method)和钩子方法(Hook Method)。

　　●　　抽象方法：一个抽象方法由抽象类声明，由具体子类实现。在Java语言里抽象方法以abstract关键字标示。

　　●　　具体方法：一个具体方法由抽象类声明并实现，而子类并不实现或置换。

　　●　　钩子方法：一个钩子方法由抽象类声明并实现，而子类会加以扩展。通常抽象类给出的实现是一个空实现，作为方法的默认实现。

　　在上面的例子中，AbstractTemplate是一个抽象类，它带有三个方法。其中abstractMethod()是一个抽象方法，它由抽象类声明为抽象方法，并由子类实现；hookMethod()是一个钩子方法，它由抽象类声明并提供默认实现，并且由子类置换掉。concreteMethod()是一个具体方法，它由抽象类声明并实现。

* 默认钩子方法
　　一个钩子方法常常由抽象类给出一个空实现作为此方法的默认实现。这种空的钩子方法叫做“Do Nothing Hook”。显然，这种默认钩子方法在缺省适配模式里面已经见过了，一个缺省适配模式讲的是一个类为一个接口提供一个默认的空实现，从而使得缺省适配类的子类不必像实现接口那样必须给出所有方法的实现，因为通常一个具体类并不需要所有的方法。

* 命名规则
　　命名规则是设计师之间赖以沟通的管道之一，使用恰当的命名规则可以帮助不同设计师之间的沟通。

　　钩子方法的名字应当以do开始，这是熟悉设计模式的Java开发人员的标准做法。在上面的例子中，钩子方法hookMethod()应当以do开头；在HttpServlet类中，也遵从这一命名规则，如doGet()、doPost()等方法。

## Android中的模板方法模式
![image](/assets/img/template_method_pattern/view.jpg)
以view为例，查看Android中的模板方法模式
### draw方法
draw方法，就是一个典型的模板方法，它是一个具体方法，给出了一个顶级逻辑的骨架，而逻辑的组成步骤在相应的抽象操作中，推迟到子类实现。
```
 public void draw(Canvas canvas) {
        ...

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
            ...
            if (!dirtyOpaque) onDraw(canvas);

            ...
            dispatchDraw(canvas);

            ...
        }

        ...
        
        // Step 2, save the canvas' layers
        ...
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        ...

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
    }
```
可以看到，在draw方法中调用了一些dispatchDraw和onDraw的方法，这些方法都是钩子方法。
```
    protected void onDraw(Canvas canvas) {
    }
    
    protected void dispatchDraw(Canvas canvas) {

    }
```

### layout方法：
layout方法也是一个模板方法。
```
public void layout(int l, int t, int r, int b) {
            ...
            onLayout(changed, l, t, r, b);
            ...
    }
```
layout方法中调用的onLayout也是一个钩子函数。
```
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }
```

* 具体的实现，将在view的子类当中，例如
### TextView的实现

```
@Override
    protected void onDraw(Canvas canvas) {
	//省略具体实现代码
    }
    
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        if (mDeferScroll >= 0) {
            int curs = mDeferScroll;
            mDeferScroll = -1;
            bringPointIntoView(Math.min(curs, mText.length()));
        }
    }
```

### ListView
```
@Override
    protected void dispatchDraw(Canvas canvas) {
    //省略具体实现代码
    }
```

### ProgressBar
```
    @Override
    protected synchronized void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        drawTrack(canvas);
    }
```

