---
title: 重装系统后，修复hexo
category: hexo
date: 2016-08-23 10:38:50
tags:
	- hexo
description: "如何在重装系统后，再次配置使用hexo，对重新迁出代码后的hexo修复也有参考意义"
---
1. 添加PATH，
```
sudo vim /etc/profile
添加
#node.js
export PATH=$PATH:/path_to_nodejs/bin
```
2. 备份_config.yml 和 .git 和 .deploy_git/.git
3. 依次执行
```
npm install hexo -g
npm update hexo -g
hexo init ~/myblog
 
npm install hexo-deployer-git --save
npm install hexo-renderer-pug --save
npm install hexo-renderer-sass --save
```
4. 还原_config.yml 和 .git 和 .deploy_git/.git

