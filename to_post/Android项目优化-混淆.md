---
title: Android项目优化-混淆
date: 2017-10-15 11:44:00
comments: true
categories: Android
tags: [项目优化,混淆]
---

* 关于混淆之后打包出现gson解析失败的问题。  
原因:项目中`Bean`类被混淆导致解析的字段与服务端返回的字段不一致。
解决方法：不混淆Gson jar包的类且不混淆`Bean`类所有属性名称

```
-keepattributes Signature

#Gson类
-keep class sun.misc.Unsafe { *; } 
-keep class com.google.gson.** { *; }

#Bean类
-keep class xxx.beans.** {*;}

#不要混淆注解类型
-keep class * extends java.lang.annotation.Annotation { *; }  
```