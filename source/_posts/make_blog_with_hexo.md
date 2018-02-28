---
title: '使用hexo搭建blog'
date: 2016-07-19 22:44:08
category: hexo
tags:
	- hexo
description: "使用hexo搭建本博客的过程"
---



## 安装node.js ##
由于node.js官网打不开，使用一个国内的地址替代
http://nodejs.cn

Linux(x64)版本node.js下载地址：https://nodejs.org/dist/v6.2.0/node-v6.2.0-linux-x64.tar.gz

```  
$ tar xvf node-v6.2.0-linux-x64.tar.gz)
$ cd node-v6.2.0-linux-x64.tar.gz
$ cd bin
$ ./node -v
v6.2.0 
```
将bin添加到PATH中

```
$ sudo vim /etc/profile

```
添加

```
#node.js
export PATH=$PATH:/path_to_nodejs/bin
```

## 安装git ##
## 申请github帐号 ##
## 添加SSH Keys ##
## 安装 hexo ##

首选官网攻略：[hexo官网教程](https://hexo.io/docs/)
官网打不开就：
```
sudo npm install -g hexo-cli 
npm update hexo -g
hexo init <folder>
```
如果指定 <folder>，便会在目前的资料夹建立一个名为 <folder> 的新资料夹；否则会在目前资料夹初始化，最好指定，不然可能安装到HOME下去
我安装到了myblog。

## 创建新博文 ##
```
hexo new "Hey,man"
```
## 创建新页面 ##

```
hexo new page "firstPage"
```
## 生成网站 ##

```
hexo generate
```
## 打开服务器 ##

```
hexo server
```
服务器会跑在 http://localhost:4000 (默认4000端口，可在_config.yml中设定)


# 配置github #
## 建立repository ##
建立与你github的用户名对应的仓库，仓库名必须为your_user_name.github.io，固定写法 然后建立关联
编辑本地hexo文件夹下的_config.yml文件
修改最底下的几行：

```
deploy:
  type: git
  repository: https://github.com/your_github_name/your_github_name.github.io.git
  branch: master
```
注意：将your_github_name 替换为你的github名，或者整个地址替换为github上该repository的地址
## 将本地的网站部署到git上 ##

```
npm install hexo-deployer-git --save
hexo deploy

```
## 查看部署结果 ##
在浏览器中访问 http://your_github_name.github.io
注意替换your_github_name

## 部署命令 ##

```
hexo clean
hexo generate
hexo deploy
```
## 常用命令 ##

```
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy #将.deploy目录部署到GitHub
hexo help  # 查看帮助
hexo version  #查看Hexo的版本
```
## 使用ssh替代https，避免每次deploy都要输入username和password ##
在github的该repository目录中，点击 Clone or download, 选择Use SSH，复制地址。

修改_config.yml,拖到最后，将repository: 后面的https地址，换为ssh的地址。

修改hexo/.deploy_git/.git/config 将remote = 后面的https地址，换为ssh地址。

再次运行hexo deploy，应该会直接递交成功，不需要输入username和password。

# 部分目录说明 #

source/ 目录下放着新建的页面

source/_posts/ 目录下放着发布的博文，使用markdown格式，可使用相关编辑器编写内容

public/ 目录对应着github上的目录结构


_config.yml 配置文件，可以在其中修改blog名之类的，注意每一项配置的:后面是有空格的。

# 将整个项目同步到Github #
在ithub上新建一个repository，命名blog_backup
git@github.com:IChrisKing/blog_backup.git
根据github的提示，在myblog下，

```
git init
git status
git add XXXXXXXXXXXX
git commit "upload all hexo source"
git remote add origin git@github.com:IChrisKing/blog_backup.git
git push -u origin master

```
以后每次修改都需要在myblog中进行一次递交，递交到github的blog_backup项目

并且在修改了_config.yml中的message内容有，使用

```
hexo deploy -g
```
进行一次递交，这次递交是到github中的blog项目



