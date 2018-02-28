---
title: Java中的值传递和引用传递
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "Java中使用值传递的情况和使用引用传递的情况"
date: 2016-09-22 19:16:06
---
[Java的值传递和引用传递问题](http://www.jianshu.com/p/c0c5e0540928)

值传递，传递的是值的拷贝，也就是说传递后就互不相关了。

引用传递，指的是在方法调用时，传递的参数是按引用进行传递，其实传递的引用的地址，也就是变量所对应的内存空间的地址。

简单的说，基本类型是按值传递的，方法的实参是一个原值的复本。类对象是按对象的引用地址（内存地址）传递地址的值，那么在方法内对这个对象进行修改是会直接反应在原对象上的（或者说这两个引用指向同一内存地址）。

三句话总结一下：
1. 对象就是传引用
2. 原始类型（byte,char,short,int,long,float,double，boolean
）就是传值
3. String，Integer, Double等immutable类型因为没有提供自身修改的函数，每次操作都是新生成一个对象，所以要特殊对待。可以认为是传值。
Integer 和 String 一样。保存value的类变量是Final属性，无法被修改，只能被重新赋值／生成新的对象。 当Integer 做为方法参数传递进方法内时，对其的赋值都会导致 原Integer 的引用被 指向了方法内的栈地址，失去了对原类变量地址的指向。对赋值后的Integer对象做得任何操作，都不会影响原来对象。

## 例子
1. 按值传递
```
public class TempTest {  
    private void test1(int a){  
        a = 5;  
        System.out.println("test1方法中的a="+a);  
}  
public static void main(String[] args) {  
    TempTest t = new TempTest();  
    int a = 3;  
    t.test1(a);//传递后，test1方法对变量值的改变不影响这里的a  
    System.out.println(”main方法中的a=”+a);  
    }  
}  
```
运行结果：
```
test1方法中的a=5  
main方法中的a=3  
```

2. 按引用传递
```
public class TempTest {  
    private void test1(A a){  
    a.age = 20;  
    System.out.println("test1方法中的age="+a.age);  
}  
public static void main(String[] args) {  
    TempTest t = new TempTest();  
    A a = new A();  
    a.age = 10;  
    t.test1(a);  
    System.out.println(”main方法中的age=”+a.age);  
    }  
}  
class A{  
    public int age = 0;  
} 
```
运行结果
```
test1方法中的age=20  
main方法中的age=20  
```
3. String的测试
```
class test{
	public static void main(String[] args) {
    	String x = new String("xxxx");
		String y = "yyyy";    
		change(x,y);
    	System.out.println("main: "+ x);
		System.out.println("main: " + y);
	}

	public static void change(String x,String y) {
    	x = "x";
		y = "y";
    	System.out.println("change: " + x);
		System.out.println("change: " + y);
	}
}
```
运行结果
```
change: x
change: y
main: xxxx
main: yyyy
```

