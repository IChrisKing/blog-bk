---
title: Android中的context详解
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "分析了context相关类的继承关系；分析了application，activity，service的context的创建时机；给出一个使用Application的例子"
date: 2016-10-08 07:07:28
---
[如何理解Android中的Context，它有什么用？](http://www.jianshu.com/p/0754e65a5744)

## Context相关类的继承关系
Activity类 、Service类 、Application类本质上都是Context子类
![image](/assets/img/android_context/context.png)
(上图中的mPackageInfo是一个LoadApk对象,frameworks/base/core/java/android/app/LoadedApk.java)
context的直系子类有两个，一个是ContextWrapper，一个是ContextImpl。那么从名字上就可以看出，ContextWrapper是上下文功能的封装类，而ContextImpl则是上下文功能的实现类。而ContextWrapper又有三个直接的子类，ContextThemeWrapper、Service和Application。其中，ContextThemeWrapper是一个带主题的封装类，而它有一个直接子类就是Activity。

## 相关类介绍
### context类
位置：/frameworks/base/core/java/android/context/Context.java

context是一个抽象类，提供一组通用的API
```
public abstract class Context {  
     ...  
     public abstract Object getSystemService(String name);  //获得系统级服务  
     public abstract void startActivity(Intent intent);     //通过一个Intent启动Activity  
     public abstract ComponentName startService(Intent service);  //启动Service  
     //根据文件名得到SharedPreferences对象  
     public abstract SharedPreferences getSharedPreferences(String name,int mode);  
     ...  
}  
```

### ContextImpl类
位置：frameworks/base/core/java/android/app/ContextImpl.java

该类实现了Context类的功能。请注意，该函数的大部分功能都是直接调用其属性mPackageInfo去完成，这点我们后面会讲到。   
```
/** 
 * Common implementation of Context API, which provides the base 
 * context object for Activity and other application components. 
 */  
class ContextImpl extends Context{  
    //所有Application程序公用一个mPackageInfo对象  
    /*package*/ ActivityThread.PackageInfo mPackageInfo;  
      
    @Override  
    public Object getSystemService(String name){  
        ...  
        else if (ACTIVITY_SERVICE.equals(name)) {  
            return getActivityManager();  
        }   
        else if (INPUT_METHOD_SERVICE.equals(name)) {  
            return InputMethodManager.getInstance(this);  
        }  
    }   
    @Override  
    public void startActivity(Intent intent) {  
        ...  
        //开始启动一个Activity  
        mMainThread.getInstrumentation().execStartActivity(  
            getOuterContext(), mMainThread.getApplicationThread(), null, null, intent, -1);  
    }  
}
```

### ContextWrapper类
位置：frameworks/base/core/java/android/content/ContextWrapper.java
该类只是对Context类的一种包装，该类的构造函数包含了一个真正的Context引用，即ContextIml对象。
```
public class ContextWrapper extends Context {  
    Context mBase;  //该属性指向一个ContextIml实例，一般在创建Application、Service、Activity时赋值  
      
    //创建Application、Service、Activity，会调用该方法给mBase属性赋值  
    protected void attachBaseContext(Context base) {  
        if (mBase != null) {  
            throw new IllegalStateException("Base context already set");  
        }  
        mBase = base;  
    }  
    @Override  
    public void startActivity(Intent intent) {  
        mBase.startActivity(intent);  //调用mBase实例方法  
    }  
}  
```

### ContextThemeWrapper类
位置：frameworks/base/core/java/android/view/ContextThemeWrapper.java
该类内部包含了主题(Theme)相关的接口，即android:theme属性指定的。只有Activity需要主题，Service不需要主题，所以Service直接继承于ContextWrapper类。
```
public class ContextThemeWrapper extends ContextWrapper {  
     //该属性指向一个ContextIml实例，一般在创建Application、Service、Activity时赋值  
       
     private Context mBase;  
    //mBase赋值方式同样有一下两种  
     public ContextThemeWrapper(Context base, int themeres) {  
            super(base);  
            mBase = base;  
            mThemeResource = themeres;  
     }  
  
     @Override  
     protected void attachBaseContext(Context newBase) {  
            super.attachBaseContext(newBase);  
            mBase = newBase;  
     }  
}  
```

## 何时创建Context实例
应用程序创建Context实例的情况有如下几种情况：
1. 创建Application 对象时， 而且整个App共一个Application对象
2. 创建Service对象时
3. 创建Activity对象时
 
因此应用程序App共有的Context数目公式为：
 
总Context实例个数 = Service个数 + Activity个数 + 1（Application对应的Context实例）

### 创建Application对象
应用程序在第一次启动时，都会首先创建Application对象。这个创建过程在frameworks/base/core/java/android/app/ActivityThread.java的handleBindApplication()方法中
```
private void handleBindApplication(AppBindData data) {
  ....
  Application app = data.info.makeApplication(data.restrictedBackupMode, null);
  ...     
}
```
函数调用了LoadedApk中的makeApplication方法：
```
public Application makeApplication(boolean forceDefaultAppClass, Instrumentation instrumentation) {  
    ...  
    try {
        java.lang.ClassLoader cl = getClassLoader();
        if (!mPackageName.equals("android")) {
                initializeJavaContextClassLoader();
            }
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);//创建并初始化一个ContextIml实例 
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);//将该Application实例传递给该ContextImpl实例
                 
    }   
    ...  
    return app;
}  
```

### 创建Activity对象
通过startActivity()或startActivityForResult()请求启动一个Activity时，如果系统检测需要新建一个Activity对象时，就会回调handleLaunchActivity()方法，该方法继而调用performLaunchActivity()方法，去创建一个Activity实例，并且回调onCreate()，onStart()方法等， 函数都位于 ActivityThread.java类
```
private final void handleLaunchActivity(ActivityRecord r, Intent customIntent) {  
    ...  
    Activity a = performLaunchActivity(r, customIntent);  //启动一个Activity  
} 

private final Activity performLaunchActivity(ActivityRecord r, Intent customIntent) {  
    ...  
    Activity activity = null;  
    try {  
        //创建一个Activity对象实例  
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();  
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);  
    ...
    }  
    if (activity != null) {  
    	Context appContext = createBaseContextForActivity(r, activity);
    	...
    	activity.attach(appContext, this, getInstrumentation(), r.token,
    	r.ident, app, r.intent, r.activityInfo, title, r.parent,
    	r.embeddedID, r.lastNonConfigurationInstances, config,
    	r.referrer, r.voiceInteractor);
        ...  
    }  
    ...      
}  
```

### 创建Service对象
 通过startService或者bindService时，如果系统检测到需要新创建一个Service实例，就会回调handleCreateService()方法，
 完成相关数据操作。handleCreateService()函数位于 ActivityThread.java类
```
private void handleCreateService(CreateServiceData data) {
    Service service = null;//创建一个Service实例 
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = (Service) cl.loadClass(data.info.name).newInstance();
      catch (Exception e) {
      	...
      }
      
    try {
	 ...
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);//创建并初始化一个ContextImpl对象实例 
        context.setOuterContext(service);//将该Service信息传递给该ContextImpl实例

        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManagerNative.getDefault());
        service.onCreate();
	...
    }
```

## Context的应用场景
![image](/assets/img/android_context/use_context.png)

NO1：启动Activity在这些类中是可以的，但是需要创建一个新的task。一般情况不推荐。(在Application或者Service需要给Intent设置Intent.FLAG_ACTIVITY_NEW_TASK才能正常启动Activity，这就会引出Activity的Task栈问题)
NO2：在这些类中去layout inflate是合法的，但是会使用系统默认的主题样式，如果你自定义了某些样式可能不会被使用。
NO3：在receiver为null时允许，在4.2或以上的版本中，用于获取黏性广播的当前值。（可以无视）
注：ContentProvider、BroadcastReceiver之所以在上述表格中，是因为在其内部方法中都有一个context用于使用。

看Activity和Application，可以看到，和UI相关的方法基本都不建议或者不可使用Application，并且，前三个操作基本不可能在Application中出现。实际上，只要把握住一点，凡是跟UI相关的，都应该使用Activity做为Context来处理；其他的一些操作，Service,Activity,Application等实例都可以。

## getApplication()与getApplicationContext()
Application本身就是一个Context，所以getApplication()和getApplicationContext()得到的结果都是MyApplication本身的实例。

但这两个方法在作用域上有比较大的区别。getApplication()方法的语义性非常强，一看就知道是用来获取Application实例的，但是这个方法只有在Activity和Service中才能调用的到。那么也许在绝大多数情况下我们都是在Activity或者Service中使用Application的，但是如果在一些其它的场景，比如BroadcastReceiver中也想获得Application的实例，这时就可以借助getApplicationContext()方法了

## 使用Application的正确方法
Application全局只有一个，它本身就已经是单例了，无需再用单例模式去为它做多重实例保护了，代码如下所示：
```
public class MyApplication extends Application {
	
	private static MyApplication app;
	
	public static MyApplication getInstance() {
		return app;
	}
	
	@Override
	public void onCreate() {
		super.onCreate();
		app = this;
	}
	
}
```
getInstance()方法可以照常提供，但是里面不要做任何逻辑判断，直接返回app对象就可以了，而app对象又是什么呢？在onCreate()方法中我们将app对象赋值成this，this就是当前Application的实例，那么app也就是当前Application的实例了。
