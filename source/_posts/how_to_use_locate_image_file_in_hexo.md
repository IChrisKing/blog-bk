---
title: 如何在hexo中使用本地图片
category: hexo
date: 2016-08-23 11:03:26
tags:
	- hexo
description: "在hexo中使用本地图片的步骤，需要安装相关的插件，将图片放在对应的位置，并正确的引用图片"
---
### 安装相关插件
```
npm install hexo-renderer-pug --save
npm install hexo-renderer-sass --save
```

### 创建相关目录
```
mkdir /path/to/your/blog/source/assets/img/
```

### 打开assets支持
```
vim _config.yml
修改这一行
post_asset_folder: true
```
### 放置图片文件
将要使用的图片文件放到/path/to/your/blog/source/assets/img/目录下
如head.png

### 使用方法
```
![image](/assets/img/head.png)
```

### 效果
![image](/assets/img/head.png)
