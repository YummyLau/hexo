---
title: android 自定义view记录
comments: true
date: 2016-12-19 12:05:04
categories: Android
tags: [经验]
---
<!--more-->
# 构造器的理解
1. 第一个构造器，针对代码中new一个view对象
2. 第二个构造器，针对在xml布局中配置view属性
3. 第三个构造器，针对使用defStyleAttr属性资源时使用
4. 第四个构造器，针对使用defStyleRes资源时使用  

**针对以上2/3/4点中对view属性的定义，综合理解与参考网上博文，有以下优先级：**
<code>xml直接定义 > xml中引用style > defStyleAttr(可见于自定义theme中) > defStyleRes > theme直接定义</code>

