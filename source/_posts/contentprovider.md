---
title: Android中的ContentProvider,ContentResolver,ContentObserver
category:
  - Android
tags:
  - Android
description: "以一个使用ContentProvider的例子为切入点，根据使用过程，深入源码中分析了ContentProvider的运行原理。内容主要涉及到ContentProvider,ContentResolver,ContentObserver这三个类。"
date: 2017-01-04 11:34:00
---

## ContentProvider简介
ContentProvider是Android四大组件之一。主要实现应用程序之间共享数据的功能，为存储和获取数据提供统一的接口。

Android系统本身提供了一些默认的ContentProvider，包括音频，视频，图片，通讯录等。

ContentProvider提供一些方法，对数据进行操作：
　　 query：查询
　　 insert：插入
　　 update：更新
　　 delete：删除
　　 getType：得到数据类型
　　 onCreate：创建数据时调用的回调函数
　　 
每个ContentProvider拥有一个公开的URI，其他应用使用这个URI来进行数据获取和操作。

## URI类简介
通用资源标志符（Universal Resource Identifier, 简称"URI"）。

Uri代表要操作的数据，Android上可用的每种资源 - 图像、视频片段等都可以用Uri来表示。

### Android中的URI
Android的Uri由以下三部分组成：

 content://com.cj.mycontentprovider/contact

一：scheme
ContentProvider（内容提供者）的scheme已经由Android所规定，
 scheme为：content://

二：主机名
主机名（或叫Authority）用于唯一标识这个ContentProvider，比如这里com.cj.mycontentprovider，外部调用者可以根据这个标识来找到它。

三：path
路径（path）可以用来表示我们要操作的数据，路径的构建应根据业务而定，如上面程序:

Android中的URI： 

	所有联系人的Uri： content://contacts/people
	某个联系人的Uri: content://contacts/people/5
	所有图片Uri: content://media/external
	某个图片的Uri：content://media/external/images/media/4

我们很经常需要解析Uri，并从Uri中获取数据。

Android系统提供了两个用于操作Uri的工具类，分别为UriMatcher 和ContentUris 。

### UriMatcher

UriMatcher 类主要用于匹配Uri.

使用方法如下。

首先第一步，初始化：
UriMatcher matcher = new UriMatcher(UriMatcher.NO_MATCH);  

第二步注册需要的Uri:
matcher.addURI("com.yfz.Lesson", "people", PEOPLE);  
matcher.addURI("com.yfz.Lesson", "person/#", PEOPLE_ID);  

第三部，与已经注册的Uri进行匹配:
Uri uri = Uri.parse("content://" + "com.yfz.Lesson" + "/people");  
int match = matcher.match(uri);  
       switch (match)  
       {  
           case PEOPLE:  
               return "vnd.Android.cursor.dir/people";  
           case PEOPLE_ID:  
               return "vnd.android.cursor.item/people";  
           default:  
               return null;  
       }  

match方法匹配后会返回一个匹配码Code，即在使用注册方法addURI时传入的第三个参数。 

上述方法会返回"vnd.Android.cursor.dir/person". 

总结: 

--常量 UriMatcher.NO_MATCH 表示不匹配任何路径的返回码

--# 号为通配符

--* 号为任意字符 

### ContentUris

ContentUris 类用于获取Uri路径后面的ID部分

1)为路径加上ID: withAppendedId(uri, id)

比如有这样一个Uri
Uri uri = Uri.parse("content://com.yfz.Lesson/people")  

通过withAppendedId方法，为该Uri加上ID
Uri resultUri = ContentUris.withAppendedId(uri, 10);  
最后resultUri为: content://com.yfz.Lesson/people/10

2)从路径中获取ID: parseId(uri)

Uri uri = Uri.parse("content://com.yfz.Lesson/people/10")  
long personid = ContentUris.parseId(uri);  
最后personid 为 :10 

## 一个使用ContentProvider的例子
### 写提供数据的应用
首先是MyContentProvider类，它继承自ContentProvider类，提供上面提到的六个基本方法：

　　 query：查询
　　 insert：插入
　　 update：更新
　　 delete：删除
　　 getType：得到数据类型
　　 onCreate：创建数据时调用的回调函数

数据的处理实际是使用SELite来实现的。
```
package com.cj.providerdemo2;  
  
import android.content.ContentProvider;  
import android.content.ContentResolver;  
import android.content.ContentUris;  
import android.content.ContentValues;  
import android.content.Context;  
import android.content.UriMatcher;  
import android.database.Cursor;  
import android.database.sqlite.SQLiteDatabase;  
import android.database.sqlite.SQLiteOpenHelper;  
import android.net.Uri;  
  
public class MyContentProvider extends ContentProvider {  
    private final static int CONTACT = 1;  
  
    private static UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);  
    static {  
        uriMatcher.addURI("com.cj.mycontentprovider","contact",CONTACT);  
    }  
  
    private MyDBHelp dbHelp;  
  
    @Override  
    public int delete(Uri uri, String selection, String[] selectionArgs) {  
        int id =0;  
        if(uriMatcher.match(uri) == CONTACT){  
            SQLiteDatabase database = dbHelp.getWritableDatabase();  
            id= database.delete("contact", selection, selectionArgs);  
            contentResolver.notifyChange(uri,null);  
        }  
       return id;  
    }  
  
    @Override  
    public String getType(Uri uri) {  
        // TODO: Implement this to handle requests for the MIME type of the data  
        // at the given URI.  
        throw new UnsupportedOperationException("Not yet implemented");  
    }  
  
    @Override  
    public Uri insert(Uri uri, ContentValues values) {  
        Uri u = null;  
        if(uriMatcher.match(uri) == CONTACT){  
            SQLiteDatabase database = dbHelp.getWritableDatabase();  
  
            long d = database.insert("contact", "_id", values);  
            u = ContentUris.withAppendedId(uri,d);  
            contentResolver.notifyChange(u,null);  
        }  
        return u;  
  
    }  
    private ContentResolver contentResolver;  
    @Override  
    public boolean onCreate() {  
        Context context = getContext();  
        contentResolver = context.getContentResolver();  
        dbHelp = new MyDBHelp(context,"contact.db",null,1);  
        return true;  
    }  
  
    @Override  
    public Cursor query(Uri uri, String[] projection, String selection,  
                        String[] selectionArgs, String sortOrder) {  
        Cursor cursor = null;  
        if(uriMatcher.match(uri) == CONTACT){  
            SQLiteDatabase database = dbHelp.getReadableDatabase();  
            cursor = database.query("contact", projection, selection, selectionArgs, null, null, sortOrder);  
            cursor.setNotificationUri(contentResolver,uri);  
        }  
        return cursor;  
  
    }  
  
    @Override  
    public int update(Uri uri, ContentValues values, String selection,  
                      String[] selectionArgs) {  
        // TODO: Implement this to handle requests to update one or more rows.  
        throw new UnsupportedOperationException("Not yet implemented");  
    }  
  
    /** 
     * 
     */  
    private static  class MyDBHelp extends SQLiteOpenHelper{  
  
        public MyDBHelp(Context context, String name, SQLiteDatabase.CursorFactory factory, int version) {  
            super(context, name, factory, version);  
  
        }  
  
        @Override  
        public void onCreate(SQLiteDatabase db) {  
            String sql = "create table contact(_id integer primary key autoincrement," +  
                    "name text not null,number text not null);";  
            db.execSQL(sql);  
        }  
  
        @Override  
        public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {  
            onCreate(db);  
  
        }  
    }  
}  

```
可以看到这个类实现了ContentProvider要求的基本方法。

在onCreate方法中，使用了
```
contentResolver = context.getContentResolver();  
```

query方法中，
```
cursor.setNotificationUri(contentResolver,uri);  
```

在insert和delete方法中，调用了contentResolver.notifyChange(uri,null);  事实上，ContentResoler是联系ContentProvider和使用它的应用之间的纽带。使用它的应用程序，是通过ContentResoler来调用ContentProvider中的方法的。具体调用方式，后面再说。

### 声明这个ContentProvider
写好自己的ContentProvider类之后，需要在Manifest.xml文件中声明这个ContentProvider
```
<provider
	android:name=".MyContentProvider"
	android:authorities="com.cj.mycontentprovider"
	android:enabled="true"
	android:exported="true">
</provider>
```
我们知道，其他应用如果想要访问这个应用程序，使用MyContentProvider提供的那些方法，来获取和操作数据，需要知道对应的uri。

uri的三个组成部分中，scheme已经被Android固定设置为content://，而这里ContentProvider有一个属性authorities,就是uri的第二部分：主机名,为了保证唯一性,一般使用包名的方式命名。

剩下uri的第三部分path，上面关于UriMatcher类的使用方法中说过，使用分为三步，结合MyContentProvider类的代码
```
    private static UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);  
    static {  
        uriMatcher.addURI("com.cj.mycontentprovider","contact",CONTACT);  
    }  
  ```
  首先初始化：  `private static UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);  `
  然后注册uri：`uriMatcher.addURI("com.cj.mycontentprovider","contact",CONTACT);`
  最后在 增删改查的方法中进行uri匹配`if(uriMatcher.match(uri) == CONTACT)`
  
  ### 写使用数据的应用
  注意，这是另一个应用了。这个应用会通过content://com.cj.mycontentprovider/contact这个uri来使用上一个应用中提供的数据。
  ```
  package com.cj.providerdemo1;  
  
import android.app.Activity;  
import android.content.ContentResolver;  
import android.content.Context;  
import android.content.Intent;  
import android.database.ContentObserver;  
import android.database.Cursor;  
import android.net.Uri;  
import android.os.Bundle;  
import android.os.Handler;  
import android.support.v7.app.AppCompatActivity;  
import android.view.LayoutInflater;  
import android.view.View;  
import android.view.ViewGroup;  
import android.widget.AdapterView;  
import android.widget.Button;  
import android.widget.CursorAdapter;  
import android.widget.ListView;  
import android.widget.TextView;  
  
public class MainActivity extends AppCompatActivity {  
    private ListView lv;  
    private Button btn;  
    private ContentResolver resolver;  
    private MyAdapter myAdapter;  
    private Cursor cursor;  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        lv = (ListView) findViewById(R.id.lv);  
        btn = (Button) findViewById(R.id.btn);  
        resolver = getContentResolver();  
        cursor = resolver.query(Uri.parse("content://com.cj.mycontentprovider/contact"), null, null,  
                null, null);  
            myAdapter = new MyAdapter(this,cursor);  
            lv.setAdapter(myAdapter);  
        btn.setOnClickListener(new View.OnClickListener() {  
            @Override  
            public void onClick(View v) {  
                MainActivity.this.startActivityForResult(new Intent(MainActivity.this,AddActivity.class),  
                        1);  
            }  
        });  
        String[] s = new String[2];  
        lv.setOnItemClickListener(new AdapterView.OnItemClickListener() {  
            @Override  
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {  
                resolver.delete(Uri.parse("content://com.cj.mycontentprovider/contact"),  
                null,null);  
  
  
            }  
        });  
        resolver.registerContentObserver(Uri.parse("content://com.cj.mycontentprovider/contact"),  
                true,new MyContentObserver(new Handler()));  
    }  
  
    @Override  
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {  
        super.onActivityResult(requestCode, resultCode, data);  
  
  
    }  
    private class MyContentObserver extends ContentObserver{  
  
        /** 
         * Creates a content observer. 
         * 
         * @param handler The handler to run {@link #onChange} on, or null if none. 
         */  
        public MyContentObserver(Handler handler) {  
            super(handler);  
        }  
  
        @Override  
        public void onChange(boolean selfChange) {  
            myAdapter.notifyDataSetChanged();  
  
        }  
    }  
    private static class MyAdapter extends CursorAdapter{  
        private Cursor cursor;  
        private Context context;  
        public MyAdapter(Context context, Cursor c) {  
            super(context, c);  
            this.context = context;  
            this.cursor = c;  
        }  
  
        @Override  
        public View newView(Context context, Cursor cursor, ViewGroup parent) {  
            LayoutInflater layoutInflater = LayoutInflater.from(context);  
            View view = layoutInflater.inflate(R.layout.item, null);  
            if(cursor !=null){  
                view.setTag(cursor.getInt(cursor.getColumnIndex("_id")));  
            }  
            return view;  
        }  
  
        @Override  
        public void bindView(View view, Context context, Cursor cursor) {  
            TextView name = (TextView) view.findViewById(R.id.tv_name);  
            TextView num = (TextView) view.findViewById(R.id.tv_num);  
            if(cursor !=null){  
                name.setText(cursor.getString(cursor.getColumnIndex("name")));  
                num.setText(cursor.getString(cursor.getColumnIndex("number")));  
            }  
        }  
  
        @Override  
        public long getItemId(int position) {  
            return super.getItemId(position);  
        }  
    }  
  
  
}  
```

```
package com.cj.providerdemo1;  
  
import android.app.Activity;  
import android.content.ContentValues;  
import android.net.Uri;  
import android.support.v7.app.AppCompatActivity;  
import android.os.Bundle;  
import android.view.View;  
import android.widget.EditText;  
import android.widget.Toast;  
  
public class AddActivity extends AppCompatActivity {  
    private EditText et_name;  
    private EditText et_num;  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_add);  
        et_name = (EditText) findViewById(R.id.et_name);  
        et_num = (EditText) findViewById(R.id.et_num);  
    }  
    public void add(View view){  
        String name = et_name.getText().toString();  
        String num = et_num.getText().toString();  
        if(name == null || num == null){  
            Toast.makeText(this,"姓名和号码不能为空",Toast.LENGTH_SHORT).show();  
        }else {  
            ContentValues contentValues = new ContentValues();  
            contentValues.put("name",name);  
            contentValues.put("number",num);  
            getContentResolver().insert(Uri.parse("content://com.cj.mycontentprovider/contact"),contentValues);  
            setResult(Activity.RESULT_OK);  
            finish();  
        }  
  
    }  
}  
```
这个应用主要提供两个功能：1.查询数据并显示。2.新增数据并显示。都是使用ContentResolver，通过uri来使用第一个应用程序中的方法来实现的。

首先，获取ContentResolver
```
 resolver = getContentResolver();  
```
然后通过query方法来获得数据。
```
 cursor = resolver.query(Uri.parse("content://com.cj.mycontentprovider/contact"), null, null,  
                null, null);  
```
然而resolver是如何最终调用到我们的MyContentProvider中的query方法的呢？下面深入源码来分析这个过程。

## 源码分析
### 获取ContentResoler对象
一切都从getContentResoler()这个方法开始。
framerwork/base/core/java/android/content/ContextWrapper.java
```
    @Override
    public ContentResolver getContentResolver() {
        return mBase.getContentResolver();
    }
 ```
 mBase变量是ContextImpl类型,是在创建activity的时候,new 一个ContextImpl对象,赋值给activity的.
 
 frameworks/base/core/java/android/app/ContextImpl.java
 ```
     @Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
 ```
 这里直接返回了ContextImpl对象的成员变量mContentResolver;这个变量是在APP启动的时候赋值的
 frameworks/base/core/java/android/app/ActivityThread.java
 ```
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {  
       ....  
  
       Activity activity = null;  
     ...  
       }  
  
       try {  
       ...  
           if (activity != null) {  
               Context appContext = createBaseContextForActivity(r, activity);//创建contextimpl的地方  
               CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());  
               Configuration config = new Configuration(mCompatConfiguration);  
               if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "  
                       + r.activityInfo.name + " with config " + config);  
               activity.attach(appContext, this, getInstrumentation(), r.token,  
                       r.ident, app, r.intent, r.activityInfo, title, r.parent,  
                       r.embeddedID, r.lastNonConfigurationInstances, config);//这里将创建的contextimpl对象传给activity内部mBase变量  
  
            ....  
       return activity;  
   }  
```
   通过createBaseContextForActivity()函数创建contextimpl对象,然后通过attach()函数将创建的contextimpl对象赋值给activity的内部变量,对应第一步的mBase变量。接下来，查看创建contextimpl对象的地方,看createBaseContextForActivity()函数
```
       private Context createBaseContextForActivity(ActivityClientRecord r, final Activity activity) {
      ...
        ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, displayId, r.overrideConfig);
        appContext.setOuterContext(activity);
        Context baseContext = appContext;

        final DisplayManagerGlobal dm = DisplayManagerGlobal.getInstance();

        ...
        if (pkgName != null && !pkgName.isEmpty()
                && r.packageInfo.mPackageName.contains(pkgName)) {
            for (int id : dm.getDisplayIds()) {
                if (id != Display.DEFAULT_DISPLAY) {
                    Display display =
                            dm.getCompatibleDisplay(id, appContext.getDisplayAdjustments(id));
                    baseContext = appContext.createDisplayContext(display);
                    break;
                }
            }
        }
        return baseContext;
    }
```
它调用到了ContextImpl.createActivityContext(
                this, r.packageInfo, displayId, r.overrideConfig);方法
 frameworks/base/core/java/android/app/ContextImpl.java
```
    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, int displayId, Configuration overrideConfiguration) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        return new ContextImpl(null, mainThread, packageInfo, null, null, false,
                null, overrideConfiguration, displayId, null);
    }
```
调用到ContextImpl方法
```
private ContextImpl(ContextImpl container, ActivityThread mainThread,
            LoadedApk packageInfo, IBinder activityToken, UserHandle user, boolean restricted,
            Display display, Configuration overrideConfiguration, int createDisplayWithId,
            String themePackageName) {
        ...
        mContentResolver = new ApplicationContentResolver(this, mainThread, user);
        ...
}
```
无关代码全部隐去，可以看到mContentResolver就是在这里被赋值的。这样，就得到了ContentResolver对象。

### 获取IContentProvider对象
接下来，查看query方法。
frameworks/base/core/java/android/content/ContentResolver.java
```
    public final @Nullable Cursor query(@NonNull Uri uri, @Nullable String[] projection,
            @Nullable String selection, @Nullable String[] selectionArgs,
            @Nullable String sortOrder) {
        android.util.SeempLog.record_uri(13, uri);
        return query(uri, projection, selection, selectionArgs, sortOrder, null);
    }
    
        public final @Nullable Cursor query(final @NonNull Uri uri, @Nullable String[] projection,
            @Nullable String selection, @Nullable String[] selectionArgs,
            @Nullable String sortOrder, @Nullable CancellationSignal cancellationSignal) {
        ...
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }
        IContentProvider stableProvider = null;
        Cursor qCursor = null;
        try {
            ...
            try {
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);//使用IContentProvider对象的query方法,最终会调用ContentProvider对象的query()方法 
            } catch (DeadObjectException e) {
                unstableProviderDied(unstableProvider);
                stableProvider = acquireProvider(uri);
                if (stableProvider == null) {
                    return null;
                }
                qCursor = stableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);//使用IContentProvider对象的query方法,最终会调用ContentProvider对象的query()方法 
            }
            if (qCursor == null) {
                return null;
            }

            // Force query execution.  Might fail and throw a runtime exception here.
            qCursor.getCount();
                ...
            CursorWrapperInner wrapper = new CursorWrapperInner(qCursor,
                    stableProvider != null ? stableProvider : acquireProvider(uri));
            stableProvider = null;
            qCursor = null;
            return wrapper;
        } catch (RemoteException e) {
                ...
            return null;
        } finally {
           ...
        }
    }
```
无论最终使用的是unstableProvider还是stableProvider，都是调用了IContentProvider对象的query方法。
在查看这个方法之前，先来关注一下unstableProvider和stableProvider的获取。
frameworks/base/core/java/android/content/ContentResolver.java
```
    public final IContentProvider acquireUnstableProvider(Uri uri) {
        if (!SCHEME_CONTENT.equals(uri.getScheme())) {
            return null;
        }
        String auth = uri.getAuthority();
        if (auth != null) {
            return acquireUnstableProvider(mContext, uri.getAuthority());
        }
        return null;
    }
    
        public final IContentProvider acquireProvider(Uri uri) {
        if (!SCHEME_CONTENT.equals(uri.getScheme())) {
            return null;
        }
        final String auth = uri.getAuthority();
        if (auth != null) {
            return acquireProvider(mContext, auth);
        }
        return null;
    }
```
这两个函数的返回值都是一个IContentProvider类型。
frameworks/base/core/java/android/app/ContextImpl.java
```
        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }
        
                @Override
        protected IContentProvider acquireUnstableProvider(Context c, String auth) {
            return mMainThread.acquireProvider(c,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), false);
        }
```
mMainThread是一个ActivityThread类型的对象，这两个方法最终都调用了它的acquireProvider方法。其中第二个参数ContentProvider.getAuthorityWithoutUserId(auth),获得的就是在AndroidManifest.xml文件中配置的android:authorities的值。
```
    public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }

        IActivityManager.ContentProviderHolder holder = null;
        try {
            holder = ActivityManagerNative.getDefault().getContentProvider(
                    getApplicationThread(), auth, userId, stable);
        } catch (RemoteException ex) {
        }
        if (holder == null) {
            Slog.e(TAG, "Failed to find provider info for " + auth);
            return null;
        }

        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
```
这里就是查找ContentProvider的精髓所在。
1. 首先，acquireExistingProvider(c, auth, userId, stable);这个函数检查本地是否已经存在这个要获取的IContentProvider。本地已经存在的IContentProvider保存在ActivityThread类的mProviderMap成员变量中，以ContentProvider对应的URI的authority为键值保存。如果本地已经存在了这个IContentProvider，直接返回它。当本地没有时，进入步骤2.
2. 获取ContentProviderHolder对象holder。通过ActivityManagerNative.getDefault().getContentProvider来获取holder。这其中经过Binder驱动进行进程间通信,会调用到ActivityManagerService中的getContentProvider()函数。具体过程可以查看[这篇](http://blog.csdn.net/hehe26/article/details/51784355)。ActivityManagerService会把所有的ContentProvider都实例化出来，并且缓存在一个map里面，所以我们就可以从ActivityManagerService远程得到一个ContentProvider对象。
3. 调用installProvider将这个IContentProvider保存在本地中，以便下次要使用这个IContentProvider时，直接就可以通过getExistingProvider()函数获取。

获取到这个IContentProvider对象后，query等方法的最终调用就会指向MyContentProvider类中的方法了。

### 数据更新（ContentObserver）
ContentProvider使用ContentObserver进行数据更新，其中应用到了观察者模式。[Android中的设计模式之观察者模式](https://ichrisking.github.io/2016/11/07/pattern-observer/)
ContentProvider的数据更新机制划分为两个单元，第一个单元是监控数据变化的ContentObserver的注册过程，第二个单元是数据更新通知的发送过程。
#### ContentObserver的注册过程
在第二个程序的onCreate中，使用
```
resolver.registerContentObserver(Uri.parse("content://com.cj.mycontentprovider/contact"),  
                true,new MyContentObserver(new Handler()));
```
注册一个自定义的ContentObserver(MyContentObserver)来监控MyContentProvider这个Content Provider中的数据变化.
```
 private class MyContentObserver extends ContentObserver{  
  
        /** 
         * Creates a content observer. 
         * 
         * @param handler The handler to run {@link #onChange} on, or null if none. 
         */  
        public MyContentObserver(Handler handler) {  
            super(handler);  
        }  
  
        @Override  
        public void onChange(boolean selfChange) {  
            myAdapter.notifyDataSetChanged();  
  
        }  
    }  
```
从ContentObserver继承下来的子类必须要实现onChange函数。当这个ContentObserver子类负责监控的数据发生变化时，ContentService就会调用它的onChange函数来处理，参数selfChange表示这个变化是否是由自己引起的。在这个应用程序中，MyContentObserver继承了ContentObserver类，它负责监控的URI是"content://com.cj.mycontentprovider/contact".当以这个URI为前缀的URI对应的数据发生改变时，ContentService都会调用这个MyContentObserver类的onChange函数来处理。在MyContentObserver类的onChange函数中，执行的操作就是重新获取MyContentProvider中的数据来更新界面上的联系人信息列表。

在MyContentObserver类的构造函数中，有一个参数handler，它的类型为Handler，它是从MainActivity类的onCreate函数中创建并传过来的。这个handler是用来分发和处理消息用的。由于MainActivity类的onCreate函数是在应用程序的主线程中被调用的，因此，这个handler参数就是和应用程序主线程的消息循环关联在一起的。在后面我们分析数据更新通知的发送过程时，便会看到这个handler参数是如何使用的了。
        
frameworks/base/core/java/android/content/ContentResolver.java
```
    public final void registerContentObserver(@NonNull Uri uri, boolean notifyForDescendents,
            @NonNull ContentObserver observer) {
        Preconditions.checkNotNull(uri, "uri");
        Preconditions.checkNotNull(observer, "observer");
        registerContentObserver(
                ContentProvider.getUriWithoutUserId(uri),
                notifyForDescendents,
                observer,
                ContentProvider.getUserIdFromUri(uri, UserHandle.myUserId()));
    }

    /** @hide - designated user version */
    public final void registerContentObserver(Uri uri, boolean notifyForDescendents,
            ContentObserver observer, int userHandle) {
        try {
            getContentService().registerContentObserver(uri, notifyForDescendents,
                    observer.getContentObserver(), userHandle);
        } catch (RemoteException e) {
        }
    }
```
当参数notifyForDescendents为true时，表示要监控所有以uri为前缀的URI对应的数据变化。上面函数做了三件事情，一是调用getContentService函数来获得前面已经启动起来了的ContentService服务，二是调用从参数传进来的ContentObserver对象observer的getContentObserver函数来获得一个Binder对象，三是通过调用这个ContentService远程接口的registerContentObserver函数来把这个Binder对象注册到ContentService中去。
    
##### getContentService
frameworks/base/core/java/android/content/ContentResolver.java
```
    public static IContentService getContentService() {
        if (sContentService != null) {
            return sContentService;
        }
        IBinder b = ServiceManager.getService(CONTENT_SERVICE_NAME);
        ...
        sContentService = IContentService.Stub.asInterface(b);
        ...
        return sContentService;
    }
```
 在ContentResolver类中，有一个静态成员变量sContentService，开始时它的值为null。当ContentResolver类的getContentService函数第一次被调用时，它便会通过ServiceManager类的getService函数来获得前面已经启动起来了的ContentService服务的远程接口，然后把它保存在sContentService变量中。这样，当下次ContentResolver类的getContentService函数再次被调用时，就可以直接把这个ContentService远程接口返回给调用者了
 
#####  getContentObserver
frameworks/base/core/java/android/database/ContentObserver.java
```
    public IContentObserver getContentObserver() {
        synchronized (mLock) {
            if (mTransport == null) {
                mTransport = new Transport(this);
            }
            return mTransport;
        }
    }
```
ContentObserver类的getContentObserver函数返回的是一个成员变量mTransport，它的类型为ContentObserver的内部类Transport。ContentObserver类的成员变量mTransport是一个Binder对象，它是要传递给ContentService服务的，以便当ContentObserver所监控的数据发生变化时，ContentService服务可以通过这个Binder对象通知相应的ContentObserver它监控的数据发生变化了。

##### registerContentObserver
frameworks/base/services/core/java/com/android/server/content/ContentService.java
```
    public void registerContentObserver(Uri uri, boolean notifyForDescendants,
            IContentObserver observer, int userHandle) {
        ...

        synchronized (mRootNode) {
            mRootNode.addObserverLocked(uri, observer, notifyForDescendants, mRootNode,
                    uid, pid, userHandle);
            ...
        }
    }
```
它调用了ContentService类的成员变量mRootNode的addObserverLocked函数来注册这个ContentObserver对象observer。成员变量mRootNode的类型为ContentService在内部定义的一个类ObserverNode。

##### addObserverLocked
```
         public void addObserverLocked(Uri uri, IContentObserver observer,
                boolean notifyForDescendants, Object observersLock,
                int uid, int pid, int userHandle) {
            addObserverLocked(uri, 0, observer, notifyForDescendants, observersLock,
                    uid, pid, userHandle);
        }

        private void addObserverLocked(Uri uri, int index, IContentObserver observer,
                boolean notifyForDescendants, Object observersLock,
                int uid, int pid, int userHandle) {
            // If this is the leaf node add the observer
            if (index == countUriSegments(uri)) {
                mObservers.add(new ObserverEntry(observer, notifyForDescendants, observersLock,
                        uid, pid, userHandle));
                return;
            }

            // Look to see if the proper child already exists
            String segment = getUriSegment(uri, index);
            if (segment == null) {
                throw new IllegalArgumentException("Invalid Uri (" + uri + ") used for observer");
            }
            int N = mChildren.size();
            for (int i = 0; i < N; i++) {
                ObserverNode node = mChildren.get(i);
                if (node.mName.equals(segment)) {
                    node.addObserverLocked(uri, index + 1, observer, notifyForDescendants,
                            observersLock, uid, pid, userHandle);
                    return;
                }
            }

            // No child found, create one
            ObserverNode node = new ObserverNode(segment);
            mChildren.add(node);
            node.addObserverLocked(uri, index + 1, observer, notifyForDescendants,
                    observersLock, uid, pid, userHandle);
        }
```
可以看出，注册到ContentService中的ContentObserver按照树形来组织，树的节点类型为ObserverNode，而树的根节点就为ContentService类的成员变量mRootNode。
private final ObserverNode mRootNode = new ObserverNode("");
每一个ObserverNode节点都对应一个名字，它是从URI中解析出来的。
在这个应用程序例子中，uri为content://com.cj.mycontentprovider/contact，它最终解析成的树状结构为：
```
        mRootNode("")

            -- ObserverNode("com.cj.mycontentprovider")

                --ObserverNode("contact") , which has a ContentObserver in mObservers  
```
至此，ContentObserver的注册过程就完成了。

#### 数据更新通知的发送过程
在第二个应用程序中，在添加联系人的界面田添加一个联系人时：
```
ContentValues contentValues = new ContentValues();    
            contentValues.put("name",name);    
            contentValues.put("number",num);    
            getContentResolver().insert(Uri.parse("content://com.cj.mycontentprovider/contact"),contentValues);    
```
会调用到第一个应用程序的MyContentProvider类的insert函数
```
 public Uri insert(Uri uri, ContentValues values) {  
        Uri u = null;  
        if(uriMatcher.match(uri) == CONTACT){  
            SQLiteDatabase database = dbHelp.getWritableDatabase();  
  
            long d = database.insert("contact", "_id", values);  
            u = ContentUris.withAppendedId(uri,d);  
            contentResolver.notifyChange(u,null);  
        }  
        return u;  
  
    } 
```
从上面传来的参数uri的值为"content://com.cj.mycontentprovider/contact"。假设当这个函数把数据成功增加到SQLite数据库之后，返回来的id值为d，于是通过调用ContentUris.withAppendedId(uri,d);得到的newUri的值就为"content://com.cj.mycontentprovider/contact/d"。这时候就会调用下面语句来通知那些注册了监控"content://com.cj.mycontentprovider/contact/d"这个URI的ContentObserver，它监控的数据发生变化了：
contentResolver.notifyChange(u,null); 
下面就根据源码来分析这个数据变化通知的发送过程：ContentResolver.notifyChange()
frameworks/base/core/java/android/content/ContentResolver.java
```
    public void notifyChange(@NonNull Uri uri, @Nullable ContentObserver observer) {
        notifyChange(uri, observer, true /* sync to network */);
    }

    public void notifyChange(@NonNull Uri uri, @Nullable ContentObserver observer,
            boolean syncToNetwork) {
        Preconditions.checkNotNull(uri, "uri");
        notifyChange(
                ContentProvider.getUriWithoutUserId(uri),
                observer,
                syncToNetwork,
                ContentProvider.getUserIdFromUri(uri, UserHandle.myUserId()));
    }
    
     public void notifyChange(Uri uri, ContentObserver observer, boolean syncToNetwork,
            int userHandle) {
        try {
            getContentService().notifyChange(
                    uri, observer == null ? null : observer.getContentObserver(),
                    observer != null && observer.deliverSelfNotifications(), syncToNetwork,
                    userHandle);
        } catch (RemoteException e) {
        }
    }
```
调用了ContentService的远接程口来调用它的notifyChange()函数来发送数据更新通知。
frameworks/base/services/core/java/com/android/server/content/ContentService.java
```
    public void notifyChange(Uri uri, IContentObserver observer,
            ...
        try {
            ArrayList<ObserverCall> calls = new ArrayList<ObserverCall>();
            synchronized (mRootNode) {
                mRootNode.collectObserversLocked(uri, 0, observer, observerWantsSelfNotifications,
                        userHandle, calls);
            }
            final int numCalls = calls.size();
            for (int i=0; i<numCalls; i++) {
                ObserverCall oc = calls.get(i);
                try {
                    oc.mObserver.onChange(oc.mSelfChange, uri, userHandle);
                    ...
                    }
                } catch (RemoteException ex) {
                    ...
                }
            }
           ...
    }
```
这个函数主要做了两件事情，第一件事情是调用ContentService的成员变量mRootNode的collectObserverLocked()函数来收集那些注册了监控"content://com.cj.mycontentprovider/contact/d"这个URI的ContentObserver，第二件事情是分别调用了这些ContentObserver的onChange()函数来通知它们监控的数据发生变化了。

可以看到，calls的类型，是一个ObserverCall的ArrayList，查看ObserverCall的定义
frameworks/base/services/core/java/com/android/server/content/ContentService.java
```
    public static final class ObserverCall {
        final ObserverNode mNode;
        final IContentObserver mObserver;
        final boolean mSelfChange;

        ObserverCall(ObserverNode node, IContentObserver observer, boolean selfChange) {
            mNode = node;
            mObserver = observer;
            mSelfChange = selfChange;
        }
    }
```
其中包括一个成员 IContentObserver mObserver。这个就是注册了监控"content://com.cj.mycontentprovider/contact/d"这个URI的ContentObserver。
在本例当中，只有一个这样的ContentObserver，即在第二个应用程序中注册的MyContentObserver。

接下来，会调用这个ContentObserver的onChange方法，通知数据更新事件
```
oc.mObserver.onChange(oc.mSelfChange, uri, userHandle);
```
前面我们在分析ContentObserver的注册过程时，介绍到注册到ContentService服务中的mObserver是一个在ContentObserver内部定义的一个类Transport的对象的远程接口，于是这里调用这个接口的onChange()函数时，就会进入到ContentObserver的内部类Transport的onChange()函数中去。
frameworks/base/core/java/android/database/ContentObserver.java
```
    private static final class Transport extends IContentObserver.Stub {
        private ContentObserver mContentObserver;

        public Transport(ContentObserver contentObserver) {
            mContentObserver = contentObserver;
        }

        @Override
        public void onChange(boolean selfChange, Uri uri, int userId) {
            ContentObserver contentObserver = mContentObserver;
            if (contentObserver != null) {
                contentObserver.dispatchChange(selfChange, uri, userId);
            }
        }

        public void releaseContentObserver() {
            mContentObserver = null;
        }
    }
```
frameworks/base/core/java/android/database/ContentObserver.java
```
    private void dispatchChange(boolean selfChange, Uri uri, int userId) {
        if (mHandler == null) {
            onChange(selfChange, uri, userId);
        } else {
            mHandler.post(new NotificationRunnable(selfChange, uri, userId));
        }
    }
```
在前面分析MyContentObserver的注册过程时，我们以第二个应用程序的主线程的消息循环创建了一个Handler，并且以这个Handler来创建了这个MyContentObserver，这个Handler就保存在MyContentObserver的父类ContentObserver的成员变量mHandler中。因此，这里的mHandler不为null，于是把这个数据更新通知封装成了一个消息，放到第二个应用程序的主线程中去处理，最终这个消息是由NotificationRunnable类的run()函数来处理的。

```
    private final class NotificationRunnable implements Runnable {
        private final boolean mSelfChange;
        private final Uri mUri;
        private final int mUserId;

        public NotificationRunnable(boolean selfChange, Uri uri, int userId) {
            mSelfChange = selfChange;
            mUri = uri;
            mUserId = userId;
        }

        @Override
        public void run() {
            ContentObserver.this.onChange(mSelfChange, mUri, mUserId);
        }
    }
```
这个函数就直接调用ContentObserver的子类的onChange函数来处理这个数据更新通知了。在我们这个情景中，这个ContentObserver子类便是MyContentObserver了。
这里它要执行的操作便是更新界面上的ListView列表中的联系人信息了。