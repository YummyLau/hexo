---
title: git的一些关键配置
layout: post
date: 2017-02-12 19:49:57
comments: true
categories: Git
tags: [版本控制]
---
<!--more-->

> 经常更换/迁移开发环境，有时git的一些细节配置会对开发造成影响。这里记录下开发过程中遇到的一些问题或者小Tips。
> (文章持续更新...)

* 经常会在IDE(Android Studio)遇到修改不了“大写命名的文件”名称，可配置git使文件名对大小写敏感。
```
git config core.ignorecase false
```
