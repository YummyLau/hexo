---
title: 提交本地项目到GitHub  
layout: post  
date: 2018-03-07 14:21:57  
comments: true  
categories: Git  
tags: [Git经验]  
---

### GitHub 新建仓库
* New repository 创建一个仓库。
	* 必填：`Repository name`
	* 选填：`Description`  

### 初始化本地仓库并提交到远端
* 初始化本地仓库

```
cd <项目文件夹>
git init
git add *
git commit -m "首次提交"
```

* 关联远端仓库并提交

```
git remote add origin <远端仓库地址>
git push -u origin master
```





