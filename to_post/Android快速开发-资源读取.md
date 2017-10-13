---
title: Android快速开发-资源读取  
date: 2017-08-15 23:06:34
comments: true
categories: Android
tags: [Android源码]
---


### Tip1 拼接drawable名字获取图片
```
public Bitmap getBitmapByName(String name) {  
        ApplicationInfo appInfo = getApplicationInfo();  
        int resID = getResources().getIdentifier(name, "drawable", appInfo.packageName);  
        return BitmapFactory.decodeResource(getResources(), resID);  
    }  
```