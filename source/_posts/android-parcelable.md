---
title: activity之间传递数据的方法
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "分析了四种在activity之间传递数据的方法，并重点分析了Parcelable和Serializable的区别"
date: 2016-09-27 19:44:03
---
[两个Activity之间如何传递参数？](http://www.jianshu.com/p/be593134eeae)

## 基于消息的通信机制 intent
使用bundle和extra来传递数据
1. 直接使用putExtra传递简单的数据

```
//存
Intent intent = new Intent(LoginActivity.this, MainActivity.class);

intent.putExtra("flag", flag);

startActivity(intent);

//取
String flag = "   ";

Intent intent1 = this.getIntent(); 

flag = intent1.getStringExtra("flag");
```

2.使用bundle传递简单的数据包
Bundle就是一个简单的数据包，它可以传递Serializable数据。

例如，我们创建一个Serializable的类Person
```
public class Person implements Serializable {

	private static final long serialVersionUID = 1L;
	private String name;
	private String passwd;
	private String gender;

	public Person(String name, String passwd, String gender) {
		this.name = name;
		this.passwd = passwd;
		this.gender = gender;
	}

	public String getName() {
		return name;
	}

	public String getPasswd() {
		return passwd;
	}

	public String getGender() {
		return gender;
	}
}

```
该类就可以使用bundle来传递
```
Person person = new Person(name, passwd, gender);
Bundle bundle = new Bundle();
					bundle.putSerializable("person", person);
Intent intent = new Intent(RegisterActivity.this,							ResultActivity.class);
intent.putExtras(bundle);
// 启动另一个Activity。					
startActivity(intent);
```

此外，Bundle对象包含了多个方法来存入、取出数据
```
putXxx(String key, Xxx data)：向Bundle放入Int、Long等各种类型的数据；

putSerializable(String key, Serializable data)：向Bundle放入一个可序列化的对象；

getXxx(String key)：从Bundle取出Int、Long等各种类型的数据；

getSerializable(String key)：从Bundle取出一个可序列化的对象。					
```

## 利用static静态数据，public static成员变量
注意，static的变量不能太大，不然容易引起异常
ERROR/AndroidRuntime(4958): Caused by: java.lang.OutOfMemoryError: bitmap size exceeds VM budget

## 基于外部存储的传输
比如File SharedPreference Sqlite，如果针对第三方应用，可以使用Content Provider

## 基于全局变量，使用application的context
```
package net.blogjava.mobile1;

import android.app.Application;
import android.graphics.Bitmap;

public class MyApp extends Application
{
    private Bitmap mBitmap;

    public Bitmap getBitmap()
    {
        return mBitmap;
    }

    public void setBitmap(Bitmap bitmap)
    {
        this.mBitmap = bitmap;
    }
    
}
<application android:name=".MyApp" android:icon="@drawable/icon" android:label="@string/app_name">

</application>
获得Bitmap对象的代码：
    ImageView imageview = (ImageView)findViewById(R.id.ivImageView);
        
    MyApp myApp = (MyApp)getApplication();
        
    imageview.setImageBitmap(myApp.getBitmap());
```

## 使用Parcelable
### 对比Parcelable和Serializable
1. Serializalbe会使用反射，序列化和反序列化过程需要大量I/O操作，Parcelable自已实现封送和解封（marshalled &unmarshalled）操作不需要用反射，数据也存放在Native内存中，效率要快很多。

	Parcelable 接口定义在封送/解封送过程中混合和分解对象的契约。Parcelable接口的底层是Parcel容器对象。Parcel类是一种最快的序列化/反序列化机制，专为Android中的进程间通信而设计。该类提供了一些方法来将成员容纳到容器中，以及从容器展开成员。

2. 在使用内存的时候，Parcelable比Serializable性能高，所以推荐使用Parcelable。

3. Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC。

4. Parcelable不能使用在要将数据存储在磁盘上的情况，因为Parcelable不能很好的保证数据的持续性在外界有变化的情况下。尽管Serializable效率低点，但此时还是建议使用Serializable 。

### Parcelable接口定义
```
public interface Parcelable {

    public int describeContents();
    //写入接口函数，打包
    public void writeToParcel(Parcel dest, int flags);

    //读取接口，目的是要从Parcel中构造一个实现了Parcelable的类的实例处理。因为实现类在这里还是不可知的，所以需要用到模板的方式，继承类名通过模板参数传入
    //为了能够实现模板参数的传入，这里定义Creator嵌入接口,内含两个接口函数分别返回单个和多个继承类实例
    /**
     * Interface that must be implemented and provided as a public CREATOR
     * field that generates instances of your Parcelable class from a Parcel.
     */
    public interface Creator<T> {

        public T createFromParcel(Parcel source);

        public T[] newArray(int size);
    }


    public interface ClassLoaderCreator<T> extends Creator<T> {

        public T createFromParcel(Parcel source, ClassLoader loader);
    }
```

### 实现Parcelable步骤
1. implements Parcelable

2. 重写writeToParcel方法，将你的对象序列化为一个Parcel对象，即：将类的数据写入外部提供的Parcel中，打包需要传递的数据到Parcel容器保存，以便从 Parcel容器获取数据

3. 重写describeContents方法，内容接口描述，默认返回0就可以

4. 实例化静态内部对象CREATOR实现接口Parcelable.Creator
```
public static final Parcelable.Creator<T> CREATOR
```
注：其中public static final一个都不能少，内部对象CREATOR的名称也不能改变，必须全部大写。需重写本接口中的两个方法：createFromParcel(Parcel in) 实现从Parcel容器中读取传递数据值，封装成Parcelable对象返回逻辑层，newArray(int size) 创建一个类型为T，长度为size的数组，仅一句话即可（return new T[size]），供外部类反序列化本类数组使用。

### Parcelable示例
```
public class MyParcelable implements Parcelable 
{
     private int mData;

     public int describeContents() 
     {
         return 0;
     }

     public void writeToParcel(Parcel out, int flags) 
     {
         out.writeInt(mData);
     }

     public static final Parcelable.Creator<MyParcelable> CREATOR = new Parcelable.Creator<MyParcelable>() 
     {
         public MyParcelable createFromParcel(Parcel in) 
         {
             return new MyParcelable(in);
         }

         public MyParcelable[] newArray(int size) 
         {
             return new MyParcelable[size];
         }
     };
     
     private MyParcelable(Parcel in) 
     {
         mData = in.readInt();
     }
 }
```

### Serializable实现与Parcelabel实现的区别
1. Serializable的实现，只需要implements  Serializable 即可。这只是给对象打了一个标记，系统会自动将其序列化。

2. Parcelabel的实现，不仅需要implements  Parcelabel，还需要在类中添加一个静态成员变量CREATOR，这个变量需要实现 Parcelable.Creator 接口。

### 代码比较
#### 创建Person类，实现Serializable
```
public class Person implements Serializable
{
    private static final long serialVersionUID = -7060210544600464481L;
    private String name;
    private int age;
    
    public String getName()
    {
        return name;
    }
    
    public void setName(String name)
    {
        this.name = name;
    }
    
    public int getAge()
    {
        return age;
    }
    
    public void setAge(int age)
    {
        this.age = age;
    }
}
```
#### 创建Book类，实现Parcelable
```
public class Book implements Parcelable
{
    private String bookName;
    private String author;
    private int publishDate;
    
    public Book()
    {
        
    }
    
    public String getBookName()
    {
        return bookName;
    }
    
    public void setBookName(String bookName)
    {
        this.bookName = bookName;
    }
    
    public String getAuthor()
    {
        return author;
    }
    
    public void setAuthor(String author)
    {
        this.author = author;
    }
    
    public int getPublishDate()
    {
        return publishDate;
    }
    
    public void setPublishDate(int publishDate)
    {
        this.publishDate = publishDate;
    }
    
    @Override
    public int describeContents()
    {
        return 0;
    }
    
    @Override
    public void writeToParcel(Parcel out, int flags)
    {
        out.writeString(bookName);
        out.writeString(author);
        out.writeInt(publishDate);
    }
    
    public static final Parcelable.Creator<Book> CREATOR = new Creator<Book>()
    {
        @Override
        public Book[] newArray(int size)
        {
            return new Book[size];
        }
        
        @Override
        public Book createFromParcel(Parcel in)
        {
            return new Book(in);
        }
    };
    
    public Book(Parcel in)
    {
        bookName = in.readString();
        author = in.readString();
        publishDate = in.readInt();
    }
}
```
