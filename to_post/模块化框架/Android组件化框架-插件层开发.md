---
title: Kotlin开发踩坑录(持续更新...)
layout: post
date: 2019-09-14
comments: true
categories: Android
tags: [Android经验,kotlin]
---
<!--more-->




### Kotlin-Java互相调用
* 使用 object 编写 kotlin 工具类是需要对方法添加 @JvmStatic 使得 Java 端能够通过引用到工具类的静态方法

