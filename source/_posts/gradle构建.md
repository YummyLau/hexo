---
title: gradle构建记录
layout: post
date: 2016-12-23
comments: true
categories: gradle
tags: [gradle]
---
<!--more-->

>基于目前AndroidStudio使用gradle构建基础，对部分配置或者构建原理还不够清楚，故留字记录以便过后温故

# 环境要求（window平台）

## 安装Java环境（网上教程众多自行参考）

```
以我jdk安装目录为**E:\Workspace\JavaArea\Java\jdk1.8.0_91**为参考配置环境变量
变量名：JAVA_HOME
变量值：E:\Workspace\JavaArea\Java\jdk1.8.0_91      
变量名：CLASSPATH
变量值：.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar; 
变量名：Path
追加值：%JAVA_HOME%\bin;%JAVA_HOME%\jre\bin;
```

## 安装Gradle

```
[Gradle各版本下载地址h](https://services.gradle.org/distributions )  
选择自己需要的版本下载，以我gradle包解压目录为**E:\Workspace\gradle\gradle-2.0**为参考配置环境变量
变量名：Path
追加值：E:\Workspace\gradle\gradle-2.0\bin;
```




