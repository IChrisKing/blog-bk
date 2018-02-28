---
title: 如何使用android源码来制作java的jar包
category:
  - Android
tags:
  - Android
  - Linux
date: 2015-04-20 18:02:16
description: "如何将android中使用的代码拆解出来，打包成java的jar包。这是为了解决开发某个android应用的pc版，将某些功能直接打包成jar包，然后丢给java使用。在这个过程中，需要注意两点：1.项目import进来的包，有些是android独有的，如何寻找java实现的，同样功能的包。2.编译时需要注意系统是32位还是64位，并在编译脚本中作相关处理。"
---

說是java的jar包，其實只是爲了區別于android的jar包

把android項目里，需要製成jar包的那部份代碼摳出來以後，要將所有android的方法和類，使用java里類似的東西替代掉。基本上都是一些import的包，這些包多數可以從網上找到，找不到的，可以試著查看android包的源碼，如果是java的class，直接自己製作一份就行了，比如這次遇到的Pair類，直接自己製作一個就行了。

遇到一個本地庫文件libkaliumjni.so，這需要針對java重新編譯。也是從git上找到源碼，然後按照readme的指示編譯。在過程中發現，用於java的和用於android的，是使用了不同的編譯方法的。其中用於android，居然有四種：arm，arm7，mips，x86. 說起來用於java的其實也要區分32位和64位。

編好的libkaliumjni.so放在項目的根目錄下一份，src下一份，用到他的類包下面一份，之所以放這麼多份，其實。。。因為沒確認到底應該放哪裡。

不過正式的做法應該是這樣的：在代碼里面調用的時候，使用	`System.loadLibrary("kaliumjni");`

這裡要注意名字，文件的名字是libkaliumjni.so，而調用的時候，只剩下了kaliumjni，簡直是。。。掐頭去尾。。。
然後將libkaliumjni.so拷貝到java.library.path的路徑下面。這個路徑其實包含了多個路徑，就像環境變量PATH一樣。要查看java.library.path包含的路徑，可以使用System.getProperty("java.library.path");
System.loadLibrary()方法會自動到java.library.path所包含的路徑下查找要load的library。這種自己將.so添加到正確位置的方法，才是代碼自動適配不同系統的標準做法。

在項目的根目錄下創建一個文件夾META-INF，其下創建一個文本MANIFEST.MF，添加內容
```
Manifest-Version: 1.0
Class-Path: libs/commons-codec-1.9.jar libs/commons-logging-1.2.jar libs/httpclient-4.4.1.jar libs/httpcore-4.4.1.jar libs/json.jar
Main-Class: mixun.platform.test.SDKTest
```
Class-Path後面跟這個項目要依賴的第三方jar，以空格隔開。這些第三方jar包的位置，要與未來這個jar包的位置平級。以libs/json.jar為例。假設這個jar打好為/xxx/xxx/a.jar 那麼第三方包就在/xxx/xxx/libs/json.jar
Main-Class後面跟要運行的main文件所在的類名，格式如上。
注意兩點：
1. 每行的冒號後面都是有空格的
2. 最後寫完了是要回車換行的，不然會報錯找不到Main-Class的

在eclipse的export里選擇JAR file打包吧。左邊窗口的libs目錄和右邊窗口的.classpath  .project  so文件都是可以不要的。
JAR Manifest Specification要選擇Use existing manifest from workspace項，點Browse..選擇剛才寫的MANIFEST.MF文件。

基本上，這樣打成的jar包就可以用了。記得要建好libs目錄。