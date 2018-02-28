---
title: Android中的设计模式之单例模式
category:
  - 设计模式
  - Java
tags:
  - 设计模式
  - Java
description: "综合了一下各种singleton的写法和分析。饿汉式，懒汉式，匿名内部类，双重校验锁。给出了单例模式在Android源码中使用的例子，主要是像InputMethodManager这样全局唯一的对象。在frameworks/base/core/java/android/view目录下有很多例子"
date: 2016-10-26 07:57:24
---
## 簡介
创建模式的一种，单例模式确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例。这个类称为单例类。

由于单例模式在内存中只有一个实例，减少了内存开销。

单例模式可以避免对资源的多重占用，例如一个写文件时，由于只有一个实例存在内存中，避免对同一个资源文件的同时写操作。

单例模式可以在系统设置全局的访问点，优化和共享资源访问。

## 实现
### 1.饿汉式  
```
/*  
 * 饿汉式，这种方式基于classloder机制避免了多线程的同步问题  
 * 不过，instance在类装载时就实例化，虽然导致类装载的原因有很多种，在单例模式中大多数都是调getInstance方法 
 * 但是也不能确定有其他的方式（或者其他的静态方法）导致类装载，这时候初始化instance显然没有达到lazy loading的效果。 
 */  
public class Singleton {  
	private static Singleton instance=new Singleton();
	private Singleton(){}   
	public static Singleton getInstance() {
	   return instance;   
	   	}  
	   } 
```

### 2.饿汉变形式  
```
/*   
 * 饿汉变形式 
 */  
public class Singleton1 {  
	private  static Singleton1 instance = null;
	static {
	   instance = new Singleton1();   		
	}    
	private Singleton1() {  }    		
	public static Singleton1 getInstance() {   
		return instance;  
	}  
}
```

### 3懒汉式  
```
/*   
 * 懒汉式,线程不安全  
 */  
public class Singleton2 {     
	private static Singleton2 instance;      
	private Singleton2(){} 
	
	public static Singleton2 getInstance(){   
		if(instance==null){
		    instance=new Singleton2();    
		}    
		return instance;   
	}   
}
```

### 4.懒汉变形式  
```
/*  
 * 懒汉式，线程安全，但是效率很低，99%情况下不需要同步  
 */  
public class Singleton3 {  
	private static Singleton3 instance;    
	private Singleton3() {  }    
	public static synchronized Singleton3 getInstance() {
	   if (instance == null) {
	       instance = new Singleton3();
	   }    
	   return instance;
	}
}
```

### 5.静态内部类  

```
/*  
 * 静态内部类  
 * 这种方式同样利用了classloder的机制来保证初始化instance时只有一个线程，  
 * 它跟第三种和第四种方式不同的是（很细微的差别）：第三种和第四种方式是只要Singleton类被装载了，  
 * 那么instance就会被实例化（没有达到lazy loading效果），  
 * 而这种方式是Singleton类被装载了，instance不一定被初始化。因为SingletonHolder类没有被主动使用，
 * 只有显示通过调用getInstance方法时，才会显示装载SingletonHolder类，从而实例化instance。  
 * 想象一下，如果实例化instance很消耗资源，我想让他延迟加载，   
 * 另外一方面，我不希望在Singleton类加载时就实例化，因为我不能确保Singleton类还可能在其他的地方被主动使用从而被加载，   
 * 那么这个时候实例化instance显然是不合适的。这个时候，这种方式相比第三和第四种方式就显得很合理。  
 */ 

public class Singleton4 {  
	private static class SingletonHolder{
	   private static Singleton4 INSTANCE=new Singleton4();
       }   
       private Singleton4(){      }   
       public static Singleton4 getInstance(){
          return SingletonHolder.INSTANCE;  
       }  
}
```

### 6.枚举式  
```
/*   
 * 枚举式，这种方式是Effective Java作者
Josh Bloch 提倡的方式 
 * 它不仅能避免多线程同步问题，而且还能防止反序列化重新创建新的对象  
 */  
public enum Singleton5 {  
	INSTANCE;   
	public void wheverMethod(){      }  
}
```

### 7.双重校检锁式  
```
/*  
 * 双重校检锁  
 */  
public class Singleton6 {  
	private volatile static Singleton6 singleton;  
	private Singleton6(){}   
	public static Singleton6 getInstance(){   
		if(singleton==null){    
			synchronized(Singleton.class){     
				if(singleton==null){      
					singleton=new Singleton6();      
				}     
			}    
		}    
		return singleton;   
	}  
}
```

## Android中的单例模式
### InputMethodManager
frameworks/base/core/java/android/view/inputmethod/InputMethodManager.java
```
    public static  getInstance() {
        synchronized (InputMethodManager.class) {
            if (sInstance == null) {
                IBinder b = ServiceManager.getService(Context.INPUT_METHOD_SERVICE);
                IInputMethodManager service = IInputMethodManager.Stub.asInterface(b);
                sInstance = new InputMethodManager(service, Looper.getMainLooper());
            }
            return sInstance;
        }
    }
```

### TextServicesManager
frameworks/base/core/java/android/view/textservice/TextServicesManager.java
```
    public static TextServicesManager getInstance() {
        synchronized (TextServicesManager.class) {
            if (sInstance != null) {
                return sInstance;
            }
            sInstance = new TextServicesManager();
        }
        return sInstance;
    }
```

### WindowManagerGlobal
frameworks/base/core/java/android/view/WindowManagerGlobal.java
```
    public static WindowManagerGlobal getInstance() {
        synchronized (WindowManagerGlobal.class) {
            if (sDefaultWindowManager == null) {
                sDefaultWindowManager = new WindowManagerGlobal();
            }
            return sDefaultWindowManager;
        }
    }
```
