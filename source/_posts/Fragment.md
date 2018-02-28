---
title: Fragment
category:
  - Android
  - Android开发工程师
tags:
  - Android开发工程师
  - Android
date: 2016-08-24 22:09:33
description: "Android Fragment的生命周期以及一些与Activity交互的方式"
---

[原文链接](https://github.com/GeniusVJR/LearningNotes/blob/master/Part1/Android/Fragment.md)
## Fragment依附于Activity的生命状态
![image](/assets/img/Fragment/FragmentWithActivity.jpg)

## Fragment生命周期方法含义
* public void onAttach(Context context)
 * onAttach方法会在Fragment于窗口关联后立刻调用。从该方法开始，就可以通过Fragment.getActivity方法获取与Fragment关联的窗口对象，但因为Fragment的控件未初始化，所以不能够操作控件。


* public void onCreate(Bundle savedInstanceState)
 * 在调用完onAttach执行完之后立刻调用onCreate方法，可以在Bundle对象中获取一些在Activity中传过来的数据。通常会在该方法中读取保存的状态，获取或初始化一些数据。在该方法中不要进行耗时操作，不然窗口不会显示。


* public View onCreateView(LayoutInflater inflater, ViewGroup container,Bundle savedInstanceState)
 * 该方法是Fragment很重要的一个生命周期方法，因为会在该方法中创建在Fragment显示的View，其中inflater是用来装载布局文件的，container是<fragment>标签的父标签对应对象，savedInstanceState参数可以获取Fragment保存的状态，如果未保存那么就为null。


* public void onViewCreated(View view,Bundle savedInstanceState)
 * Android在创建完Fragment中的View对象之后，会立刻回调该方法。其种view参数就是onCreateView中返回的view，而bundle对象用于一般用途。


* public void onActivityCreated(Bundle savedInstanceState)
 * 在Activity的onCreate方法执行完之后，Android系统会立刻调用该方法，表示窗口已经初始化完成，从这一个时候开始，就可以在Fragment中使用getActivity().findViewById(Id);来操控Activity中的view了。


* public void onStart()
 * 当系统调用该方法的时候，fragment已经显示在ui上，但还不能进行互动，因为onResume方法还没执行完。


* public void onResume()
 * 该方法为fragment从创建到显示Android系统调用的最后一个生命周期方法，调用完该方法时候，fragment就可以与用户互动了。


* public void onPause()
 * fragment由活跃状态变成非活跃状态执行的第一个回调方法，通常可以在这个方法中保存一些需要临时暂停的工作。如保存音乐播放进度，然后在onResume中恢复音乐播放进度。


* public void onStop()
 * 当onStop返回的时候，fragment将从屏幕上消失。


* public void onDestoryView()
 * 该方法的调用意味着在 onCreateView 中创建的视图都将被移除。


* public void onDestroy()
 * Android在Fragment不再使用时会调用该方法，要注意的是~这时Fragment还和Activity藕断丝连！并且可以获得Fragment对象，但无法对获得的Fragment进行任何操作。


* public void onDetach()
 * 为Fragment生命周期中的最后一个方法，当该方法执行完后，Fragment与Activity不再有关联

 
## Fragment与Activity传递参数
Fragment.setArguments(Bundle args)

Fragment.getArguments()

## Fragment状态持久化
* 方法一：

 protected void onSaveInstanceState(Bundle outState)

 protected void onRestoreInstanceState(Bundle savedInstanceState) 

* 方法二：这个方法仅仅能够保存Fragment中的控件状态，比如说EditText中用户已经输入的文字，并且，空间必须设置id，否则无法保存控件状态

 FragmentManager.putFragment(Bundle bundle, String key, Fragment fragment) 

 FragmentManager.getFragment(Bundle bundle, String key)
 
 * 方法三：保存变量
 ```
      /** Activity中的代码 **/
     FragmentB fragmentB;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.fragment_activity);
        if( savedInstanceState != null ){
            fragmentB = (FragmentB) getSupportFragmentManager().getFragment(savedInstanceState,"fragmentB");
        }
        init();
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        if( fragmentB != null ){
            getSupportFragmentManager().putFragment(outState,"fragmentB",fragmentB);
        }

        super.onSaveInstanceState(outState);
    }

    /** Fragment中保存变量的代码 **/

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        AppLog.e("onCreateView");
        if ( null != savedInstanceState ){
            String savedString = savedInstanceState.getString("string");
            //得到保存下来的string
        }
        View root = inflater.inflate(R.layout.fragment_a,null);
        return root;
    }

    @Override
    public void onSaveInstanceState(Bundle outState) {
        outState.putString("string","anAngryAnt");
        super.onSaveInstanceState(outState);
    }
 ```
 
 ## complete activity/fragment lifecycle
 ![image](/assets/img/Fragment/complete_android_fragment_lifecycle.png)
