---
title: Activity生命周期
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "详细分析Activity的生命周期。举例说明各种操作过程中，Activity生命周期变化的过程。分析横屏竖屏切换过程中的生命周期变化。分析onSaveInstanceState方法和onRestoreInstanceState方法的执行时机。"
date: 2016-09-22 20:24:09
---

[怎么理解Activity的生命周期？](http://www.jianshu.com/p/ae6e1d93cc8e)


[如何判断Activity是否在运行？](http://www.jianshu.com/p/f8a0c43b3dfe)

[自定义View的状态是如何保存的？](http://www.jianshu.com/p/1071b9c48f1e)

这个题目主要针对view的onSaveInstanceState方法

[通过new创建的View实例它的onSaveStateInstance会被调用吗?](http://www.jianshu.com/p/4f482548de59)

这个题目是view的onSaveInstanceState方法的一个延伸

## Activity生命周期
Activity是由Activity栈进管理，当来到一个新的Activity后，此Activity将被加入到Activity栈顶，之前的Activity位于此Activity底部。Acitivity一般意义上有四种状态：

1.当Activity位于栈顶时，此时正好处于屏幕最前方，此时处于运行状态；

2.当Activity失去了焦点但仍然对用于可见（如栈顶的Activity是透明的或者栈顶Activity并不是铺满整个手机屏幕），此时处于暂停状态；

3.当Activity被其他Activity完全遮挡，此时此Activity对用户不可见，此时处于停止状态；

4.当Activity由于人为或系统原因（如低内存等）被销毁，此时处于销毁状态；

在每个不同的状态阶段，Android系统对Activity内相应的方法进行了回调。因此，我们在程序中写Activity时，一般都是继承Activity类并重写相应的回调方法。

![image](/assets/img/android_activity/activity_life_cycle.png)

1. Activity实例是由系统自动创建，并在不同的状态期间回调相应的方法。一个最简单的完整的Activity生命周期会按照如下顺序回调：onCreate -> onStart -> onResume -> onPause -> onStop -> onDestroy。称之为entire lifetime
2. 当执行onStart回调方法时，Activity开始被用户所见（也就是说，onCreate时用户是看不到此Activity的，那用户看到的是哪个？当然是此Activity之前的那个Activity），一直到onStop之前，此阶段Activity都是被用户可见，称之为visible lifetime
3. 当执行到onResume回调方法时，Activity可以响应用户交互，一直到onPause方法之前，此阶段Activity称之为foreground lifetime

## 一些例子
在实际应用场景中，假设A Activity位于栈顶，此时用户操作，从A Activity跳转到B Activity。那么对AB来说，具体会回调哪些生命周期中的方法呢？回调方法的具体回调顺序又是怎么样的呢？

开始时，A被实例化，执行的回调有A:onCreate -> A:onStart -> A:onResume。

当用户点击A中按钮来到B时，假设B全部遮挡住了A，将依次执行A:onPause -> B:onCreate -> B:onStart -> B:onResume -> A:onStop。

此时如果点击Back键，将依次执行B:onPause -> A:onRestart -> A:onStart -> A:onResume -> B:onStop -> B:onDestroy。

至此，Activity栈中只有A。在Android中，有两个按键在影响Activity生命周期这块需要格外区分下，即Back键和Home键。我们先直接看下实验结果：

此时如果按下Back键，系统返回到桌面，并依次执行A:onPause -> A:onStop -> A:onDestroy。

此时如果按下Home键（非长按），系统返回到桌面，并依次执行A:onPause -> A:onStop。由此可见，Back键和Home键主要区别在于是否会执行onDestroy。

* 此外，需要注意，不能在onPause()中执行耗时的操作。从上面的流程可以看出，就得Activity必须先onPause，新的Activity才会开始执行onCreate等一些列的操作，所以，如果在旧的Activity的onPause中执行耗时的操作，就会影响界面切换的速度。

## 何时会调用onRestart()
* 当用户按下Home键然后又回到当前程序，就会执行这个方法
* 当用户从当前甲Activity打开一个新的Activity，然后又back键返回到甲Activity
* 用户按下任务列表，然后选择了刚刚打开过的那个程序，那么这个方法也会执行。

## 一个别人踩过的坑
**重点是：只要没有人为的调用A的finish()方法，虽然A执行了onDestroy，但Activity栈中依然保留有A**

由于Android本身的特性，使得现在不少应用都没有直接退出应用程序的功能，按照一般的逻辑，当Activity栈中有且只有一个Activity时，当按下Back键此Activity会执行onDestroy，那么下次点击此应用程图标将从重新启动，因此，当前不少应用程序都是采取如Home键的效果，当点击了Back键，系统返回到桌面，然后点击应用程序图标，直接回到之前的Activity界面，这种效果是怎么实现的呢？

通过重写按下Back键的回调函数，转成Home键的效果即可。
```
@Override
public void onBackPressed() {
    Intent home = new Intent(Intent.ACTION_MAIN);
    home.addCategory(Intent.CATEGORY_HOME);
    startActivity(home);
}
```
当然，此种方式通过Home键效果强行影响到Back键对Activity生命周期的影响。注意，此方法只是针对按Back键需要退回到桌面时的Activity且达到Home效果才重写。

或者，为达到此类效果，Activity实际上提供了直接的方法。
```
activity.moveTaskToBack(true);
```
moveTaskToBack()此方法直接将当前Activity所在的Task移到后台，同时保留activity顺序和状态。

在之前的项目开发过程中，当时遇到一个很奇怪的问题：手机上的“开发者选项”中有一个“不保留活动”的设置，当开启此设置，手机上的设置提示是“用户离开后即销毁每个活动”，开启后，对于其他的应用程序是从A Acticity到B Activity，然后Back键回到A，此时，其他应用程序只是先白屏（有可能黑屏等，取决于主题设置）一下，然后A开始可见，但是我的应用程序中出现的一个结果却是直接返回到了桌面。一开始百思不得其解。最后终于定位出问题。首先，我们需要明确开启此设置项后对Activity生命周期的影响。开启此设置项后，当A到B时，假设B全部遮挡住了A，将依次执行A:onPause -> B:onCreate -> B:onStart -> B:onResume -> A:onStop -> A:onDestroy。是的，A在系统原本的生命周期回调中增加了onDestroy。此即“用户离开后即销毁每个活动”的含义。但此时需要注意的是，只要没有认为的调用A的finish()方法，虽然A执行了onDestroy，但Activity栈中依然保留有A，此时B处于栈顶。那么在B中按Back键回到A时，将依次执行：B:onPause -> A:onCreate -> A:onStart -> A:onResume -> B:onStop -> B:onDestroy。没错，A从onCreate开始执行了。此处也就解释了为什么A可能会出现白屏（或黑屏等）一下的原因了。

那么为什么我的应用程序会跟其他应用程序出现不一样呢?最后定为出问题在于当时我的应用程序中为了做到完全退出应用程序效果，专门使用了一个Activity栈去维护Activity（当时是借鉴了网上的此类实现方案，现在想想，实在没必要，且不说Android本身特性决定了没必要通过如此方法去达到退出效果，仅仅是此方法本身也存在很大的问题，现在在网上依然能见到有不少文章说到应用程序退出可以使用此方法，哎。。），在onCreate中入栈，onDestroy出栈，调用了如下方法：
```
// 结束Activity&从堆栈中移除
AppManager.getAppManager().finishActivity(this);
```
其中，AppManager中finishActivity函数具体定义是：
```
/**
 * 结束指定的Activity
 */
public void finishActivity(Activity activity) {
    if (activity != null) {
        activityStack.remove(activity);
        activity.finish();
        activity = null;
    }
}
```
至此，相信大家应该看出问题的所在了吧。

没错，问题在于执行了activity的finish()方法！！ activity的finish()方法至少有两个层面含义，1.将此Activity从Activity栈中移除，2.调用了此Activity的onDestroy方法。对于不开启“不保留活动”的设置项，实际上也没什么影响，但是一旦开启此设置，问题显露无疑。开启此此设置后，正常情况下离开A，即使执行了A的onDestroy，Activity栈中还是有A的，但是我这样写后，finish()方法一执行，Activity栈中就没有A了，因此，当点击Back键时，Activity栈中已经没有此应用的任何Activity了，直接来到了手机桌面。

可能，有些人会说，我就是要通过此种方法想去完全退出应用程序，同时希望自己的Activity栈和系统中Activity栈保持一致，怎么办呢？

在此，可以通过如下改写去实现：
```
/**
* 结束指定的Activity
 */
public void finishActivity(Activity activity) {
    if (activity != null) {
    // 为与系统Activity栈保持一致，且考虑到手机设置项里的"不保留活动"选项引起的Activity生命周期调用onDestroy()方法所带来的问题,此处需要作出如下修正
    if(activity.isFinishing()){
        activityStack.remove(activity);
        //activity.finish();
        activity = null;
    }
    }
}
```

## 横屏竖屏切换
1、新建一个Activity，并把各个生命周期打印出来

2、运行Activity，得到如下信息

onCreate-->
onStart-->
onResume-->

3、按crtl+f12切换成横屏时

onSaveInstanceState-->
onPause-->
onStop-->
onDestroy-->
onCreate-->
onStart-->
onRestoreInstanceState-->
onResume-->

4、再按crtl+f12切换成竖屏时，发现打印了两次相同的log

onSaveInstanceState-->
onPause-->
onStop-->
onDestroy-->
onCreate-->
onStart-->
onRestoreInstanceState-->
onResume-->
onSaveInstanceState-->
onPause-->
onStop-->
onDestroy-->
onCreate-->
onStart-->
onRestoreInstanceState-->
onResume-->

5、修改AndroidManifest.xml，把该Activity添加 android:configChanges="orientation"，执行步骤3

onSaveInstanceState-->
onPause-->
onStop-->
onDestroy-->
onCreate-->
onStart-->
onRestoreInstanceState-->
onResume-->

6、再执行步骤4，发现不会再打印相同信息，但多打印了一行onConfigChanged

onSaveInstanceState-->
onPause-->
onStop-->
onDestroy-->
onCreate-->
onStart-->
onRestoreInstanceState-->
onResume-->
onConfigurationChanged-->

7、把步骤5的android:configChanges="orientation" 改成 android:configChanges="orientation|keyboardHidden"，执行步骤3，就只打印onConfigChanged

onConfigurationChanged-->

8、执行步骤4

onConfigurationChanged-->
onConfigurationChanged-->

 总结：

1、不设置Activity的android:configChanges时，切屏会重新调用各个生命周期，切横屏时会执行一次，切竖屏时会执行两次

2、设置Activity的android:configChanges="orientation"时，切屏还是会重新调用各个生命周期，切横、竖屏时只会执行一次

3、设置Activity的android:configChanges="orientation|keyboardHidden"时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged方法


#### 补充
1. 当前Activity产生事件弹出Toast和AlertDialog的时候Activity的生命周期不会有改变

2. Activity运行时按下HOME键(跟被完全覆盖是一样的)：onSaveInstanceState --> onPause --> onStop       onRestart -->onStart--->onResume

3. Activity未被完全覆盖只是失去焦点：onPause--->onResume

4. 在一些特殊的情况中，你可能希望当一种或者多种配置改变时避免重新启动你的activity。你可以通过在manifest中设置android:configChanges属性来实现这点。

	你可以在这里声明activity可以处理的任何配置改变，当这些配置改变时不会重新启动activity，而会调用activity的
onConfigurationChanged(Resources.Configuration)方法。如果改变的配置中包含了你所无法处理的配置（在android:configChanges并未声明），
你的activity仍然要被重新启动，而onConfigurationChanged(Resources.Configuration)将不会被调用。

## Activity/View的状态保存机制
Android中为Actvity的实例状态保存和恢复提供了相应的机制，通过提供相应的实例状态保存和恢复回调，将状态数据存储到系统中的Bundle中。相应的回调函数分别为：onSaveInstanceState(Bundle)和onRestoreInstanceState(Bundle)。当然，对于实例状态的恢复，也可以直接通过onCreate(Bundle)中的Bundle参数进行。onCreate(Bundle)和onRestoreInstanceState(Bundle)都能恢复实例状态，且一般情况下，两种方式恢复实例状态功能相同，唯一比较有差别的地方，在于恢复实例状态的时机，onRestoreInstanceState(Bundle)回调时机更加靠后。

![image](/assets/img/android_activity/saveInstanceState.png)

当某个activity变得“容易”被系统销毁时，该activity的onSaveInstanceState就会被执行，除非该activity是被用户主动销毁的，例如当用户按BACK键的时候。何为“容易”？言下之意就是该activity还没有被销毁，而仅仅是一种可能性。这种可能性有这么几种情况： 
1. 当用户按下HOME键时。 
这是显而易见的，系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activity A是否会被销毁，故系统会调用onSaveInstanceState，让用户有机会保存某些非永久性的数据。以下几种情况的分析都遵循该原则 
2. 长按HOME键，选择运行其他的程序时。 
3. 按下电源按键（关闭屏幕显示）时。 
4. 从activity A中启动一个新的activity时。 
5. 屏幕方向切换时，例如从竖屏切换到横屏时。 
在屏幕切换之前，系统会销毁activity A，在屏幕切换之后系统又会自动地创建activity A，所以onSaveInstanceState一定会被执行。 

总而言之，onSaveInstanceState的调用遵循一个重要原则，即当系统“未经你许可”时销毁了你的activity，则onSaveInstanceState会被系统调用，这是系统的责任，因为它必须要提供一个机会让你保存你的数据.它会在调用 onStop之前，但是是不是在onPuase之前就不能确认了，要看情况，官方文档在说明这个执行顺序时用了“可能”这个词。

Activity类的onSaveInstanceState默认实现会恢复Activity的状态，默认实现会为布局中的每个View调用相应的 onSaveInstanceState方法，让每个View都能保存自身的信息。

这里需要注意一个细节：想要保存View的状态，需要在XML布局文件中提供一个唯一的ID（android:id），如果没有设置这个ID的话，View控件的onSaveInstanceState是不会被调用的。

至于onRestoreInstanceState方法，需要注意的是，

**onSaveInstanceState方法和onRestoreInstanceState方法“不一定”是成对的被调用的。**

onRestoreInstanceState被调用的前提是，activity A“确实”被系统销毁了，而如果仅仅是停留在有这种可能性的情况下，则该方法不会被调用，例如，当正在显示activity A的时候，用户按下HOME键回到主界面，然后用户紧接着又返回到activity A，这种情况下activity A一般不会因为内存的原因被系统销毁，故activity A的onRestoreInstanceState方法不会被执行。
 
另外，onRestoreInstanceState的bundle参数也会传递到onCreate方法中，你也可以选择在onCreate方法中做数据还原。 

## 通过new创建的View实例它的onSaveStateInstance会被调用吗

自定义View控件的状态被保存需要满足两个条件：
1. View有唯一的ID；
2. View的初始化时要调用setSaveEnabled(true) 

View的状态保存和读取的调用过程：
![image](/assets/img/android_activity/savestate.png)

里面的SparseArray(完整的参数是：SparseArray<Parcelable> )是一个KEY-VALUE的Map，KEY当然就是View的ID了。

所以，只要设置了ID就会。其实我们在XML文件中配置的布局和属性最终都是通过LayoutInflater中的inflate方法去加载，由它去创建各个View的实例（还是用new），并根据XML文件中的属性设置相关的值。

## 如何判断Activity是否在运行？
这个问题直接对应着各种弹dialog的时候，没有activity的情况。

比如，Activity开启了一个线程，去执行一个耗时操作，并在操作完成后，使用handler来通知Activity弹出通知dailog。但如果在线程执行的过程中，按了back键，activity就被销毁了，当线程执行结束，需要弹出dialog时，就会报错，进而引起崩溃。

* 为什么activity销毁了，仍然能执行到弹出dialog的代码？

	Activity销毁只是不再受系统的AMS控制，但Activity这个对象的实例还是存在于内存中的，具体什么时候真正把这个对象实例也销毁（回收）了，就要看内存回收机制了，哪怕是这个实例没有可达的引用了也不一定会马上回收。

### 错误的解决方法
isFinishing()

源码中
```
public boolean isFinishing() {
     return mFinished;
}
```
而mFinished是在finish()中被赋值的，也就是说只有通过调用finish()结束的Activity，mFinished的值才会被置为true。所以有时候Activity的生命周期没有按我们预想的来走时（如内存紧张时），会出现判断出错的情况。

### 正确的解决方法
该方法来自官方应用Dialer

packages/apps/Dialer/src/com/android/dialer/calllog/ClearCallLogDialog.java
```
if (activity == null || activity.isDestroyed() || activity.isFinishing()) {
                            return;
}

if (progressDialog != null && progressDialog.isShowing()) {
                            progressDialog.dismiss();
}
```
可以看到，判断条件有三个：activity == null || activity.isDestroyed() || activity.isFinishing()