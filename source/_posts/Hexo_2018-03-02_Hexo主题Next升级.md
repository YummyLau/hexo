---
title: Hexo主题Next升级
layout: post
date: 2018-03-02 10:31
comments: true
categories: Hexo
tags: [主题升级]
---
### 同步主仓库
Github上Hexo仓库地址
```
https://github.com/YummyLau/hexo
```
直接`git clone`下载到本地。

### 删除subtree
由于之前 [hexo实现多端写blog问题记录](http://yummylau.com/2016/12/20/hexo%E5%AE%9E%E7%8E%B0%E5%A4%9A%E7%AB%AF%E5%86%99bolg%E9%97%AE%E9%A2%98%E8%AE%B0%E5%BD%95/) 中我们使用subtree引用`hexo-theme-next`仓库。如果可以直接使用：

```
git subtree pull --prefix=<目录> <remote added name> <分支> --squash
```
指令更新则直接更新`subtree`就行了。试了几次都出现`hash`校验的失败，简单粗暴的方法：

```
git rm -r themes/next   //themes/next为hexo目录下存放主题的目录
git add *
git commit -m "删除旧主题"
git push -u origin master
```
切记一定要把本地旧`themes/next`文件**彻底删除**，最好重新拉取远端分支。

### 下载新主题
新建主题文件，下载最新版本的`hexo-theme-next`并删除.git仓库信息。
在hexo目录下,新建`themes/next`目录  

```
mkdir themes  
cd themes
mkdir next
cd ../
```

```
git clone https://github.com/iissnan/hexo-theme-next themes/next
rm -r ./themes/next/.git/
```

### 同步远端仓库

```
git add *
git commit -m "更新hexo-theme-next主题"
git push
```
完成主题升级！