---
title: Android中的设计模式之策略模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "介绍了策略模式的定义。分析了Android中的Animation时如何使用策略模式依靠Interpolator的不同而实现的。"
date: 2016-11-23 19:55:33
---
## 简介
策略模式属于对象的行为模式。其用意是针对一组算法，将每一个算法封装到具有共同接口的独立的类中，从而使得它们可以相互替换。策略模式使得算法可以在不影响到客户端的情况下发生变化。

策略模式的结构
　　策略模式是对算法的包装，是把使用算法的责任和算法本身分割开来，委派给不同的对象管理。策略模式通常把一个系列的算法包装到一系列的策略类里面，作为一个抽象策略类的子类。用一句话来说，就是：“准备一组算法，并将每一个算法封装起来，使得它们可以互换”。


### 策略模式的优点
　　（1）策略模式提供了管理相关的算法族的办法。策略类的等级结构定义了一个算法或行为族。恰当使用继承可以把公共的代码移到父类里面，从而避免代码重复。

　　（2）使用策略模式可以避免使用多重条件(if-else)语句。多重条件语句不易维护，它把采取哪一种算法或采取哪一种行为的逻辑与算法或行为的逻辑混合在一起，统统列在一个多重条件语句里面，比使用继承的办法还要原始和落后。

### 策略模式的缺点
　　（1）客户端必须知道所有的策略类，并自行决定使用哪一个策略类。这就意味着客户端必须理解这些算法的区别，以便适时选择恰当的算法类。换言之，策略模式只适用于客户端知道算法或行为的情况。

　　（2）由于策略模式把每个具体的策略实现都单独封装成为类，如果备选的策略很多的话，那么对象的数目就会很可观。

![image](/assets/img/strategy_pattern/strategy-pattern.png)

[源代码](http://www.cnblogs.com/java-my-life/archive/2012/05/10/2491891.html)

这个模式涉及到三个角色：

　　●　　环境(Context)角色：持有一个Strategy的引用。

　　●　　抽象策略(Strategy)角色：这是一个抽象角色，通常由一个接口或抽象类实现。此角色给出所有的具体策略类所需的接口。

　　●　　具体策略(ConcreteStrategy)角色：包装了相关的算法或行为。

## Android中的策略模式
在android中不同Animation动画的实现，主要是依靠Interpolator的不同而实现的。动画实现的流程，要先从View入手，而View一般不会单独存在，都是存在于某个ViewGroup中，所以Android动画绘制的地方选择了在ViewGroup中的drawChild(Canvas canvas, View child, long drawingTime)方法中进行调用子View的绘制。
```
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }
```
再查看View中的draw方法：
```
    boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
        
        .......
//获取设置的动画信息
        final Animation a = getAnimation();
        if (a != null) {
        //绘制动画
            more = applyLegacyAnimation(parent, drawingTime, a, scalingRequired);
            concatMatrix = a.willChangeTransformationMatrix();
            if (concatMatrix) {
                mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
            }
            transformToApply = parent.getChildTransformation();
        } else {
        	......
        }

......
    }
```

查看applyLegacyAnimation方法：
```
Transformation invalidationTransform;
        final int flags = parent.mGroupFlags;
        //动画是否已经初始化过
        final boolean initialized = a.isInitialized();
        if (!initialized) {
            a.initialize(mRight - mLeft, mBottom - mTop, parent.getWidth(), parent.getHeight());
            a.initializeInvalidateRegion(0, 0, mRight - mLeft, mBottom - mTop);
            if (mAttachInfo != null) a.setListenerHandler(mAttachInfo.mHandler);
            onAnimationStart();
        }

	//View是否需要进行缩放
        final Transformation t = parent.getChildTransformation();
        boolean more = a.getTransformation(drawingTime, t, 1f);
        if (scalingRequired && mAttachInfo.mApplicationScale != 1f) {
            if (parent.mInvalidationTransformation == null) {
                parent.mInvalidationTransformation = new Transformation();
            }
            invalidationTransform = parent.mInvalidationTransformation;
            a.getTransformation(drawingTime, invalidationTransform, 1f);
        } else {
            invalidationTransform = t;
        }

        if (more) {
        	//根据具体实现，判断当前动画类型是否需要进行调整位置到小，然后刷新不同的区域
            if (!a.willChangeBounds()) {
                .......
            } else {
            	  .....
            }
        }
        return more;
    }
```
其中主要的操作是动画始化、动画操作、界面刷新。动画的具体实现是调用了Animation中的getTransformation(long currentTime, Transformation outTransformation,float scale)方法。
```
    public boolean getTransformation(long currentTime, Transformation outTransformation,
            float scale) {
        mScaleFactor = scale;
        return getTransformation(currentTime, outTransformation);
    }
    
    public boolean getTransformation(long currentTime, Transformation outTransformation) {
        .....
		//计算当前动画的时间点
            final float interpolatedTime = mInterpolator.getInterpolation(normalizedTime);
            //应用动画效果
            applyTransformation(interpolatedTime, outTransformation);
        }
	 ......
        return mMore;
    }
```
根据上面一系列的分析，Android系统中在处理动画的时候会调用插值器中的getInterpolation(float input)方法来获取当前的时间点，依次来计算当前变化的情况。

Android中的插值器Interpolator，它的作用是根据时间流逝的百分比来计算出当前属性值改变的百分比，系统预置的有LinearInterpolator（线性插值器：匀速动画）、AccelerateDecelerateInterpolator（加速减速插值器：动画两头慢中间快）和DecelerateInterpolator（减速插值器：动画越来越慢）等。
![image](/assets/img/strategy_pattern/interpolator.png)

查看INterpolator的定义：
```
public interface Interpolator extends TimeInterpolator {
    // A new interface, TimeInterpolator, was introduced for the new android.animation
    // package. This older Interpolator interface extends TimeInterpolator so that users of
    // the new Animator-based animations can use either the old Interpolator implementations or
    // new classes that implement TimeInterpolator directly.
}
```
它只是继承了TimeInterpolator，没有做任何改变，查看TimeInterpolator。
```
public interface TimeInterpolator {

    /**
     * Maps a value representing the elapsed fraction of an animation to a value that represents
     * the interpolated fraction. This interpolated value is then multiplied by the change in
     * value of an animation to derive the animated value at the current elapsed animation time.
     *
     * @param input A value between 0 and 1.0 indicating our current point
     *        in the animation where 0 represents the start and 1.0 represents
     *        the end
     * @return The interpolation value. This value can be more than 1.0 for
     *         interpolators which overshoot their targets, or less than 0 for
     *         interpolators that undershoot their targets.
     */
    float getInterpolation(float input);
}
```
只是定义了一个getInterpolation方法。

接下来查看不同的Interpolator是如何实现这个方法的。

### Accelerate	Interpolator
```
    public float getInterpolation(float input) {
        if (mFactor == 1.0f) {
            return input * input;
        } else {
            return (float)Math.pow(input, mDoubleFactor);
        }
    }
```

### CycleInterpolator
```
    public float getInterpolation(float input) {
        return (float)(Math.sin(2 * mCycles * Math.PI * input));
    }
```

### LinearInterpolator
```
    public float getInterpolation(float input) {
        return input;
    }
```