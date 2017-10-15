---
title: Android快速开发-权限处理
date: 2017-08-15 15:52:00
comments: true
categories: Android
tags: [快速开发]
---

# Android 6.0
如果操作需要某一项权限而App却没有被授予该权限，则会抛出异常。

1. 提高targetSdkVersion到23
意味着6.0以下的设备表现得和以往一样。而6.0及以上设备则会接受权限检测。
2. 用于的targetSdkVersion<23,则默认App会在安装的时候授权，但是对于6.0以上的设备依然可以主动撤销权限。
但是，如果我们需要用户授予某种权限来调用某个函数，在targetSdkVersion<23不会出现异常，如果对于需要返回值的函数，可能会返回0或者null。但是仍然可能因为我们函数的返回值而导致程序crash。


>系统权限分为`正常权限`和`危险权限`  
>正常权限不会直接给用户隐私权带来风险。如果您的应用在其清单中列出了正常权限，系统将自动授予该权限。  
>危险权限会授予应用访问用户机密数据的权限。如果您的应用在其清单中列出了正常权限，系统将自动授予该权限。如果您列出了危险权限，则用户必须明确批准您的应用使用这些权限。

权限通过组别来申请
通过`Activity#checkSelfPermission`来检测权限  
通过`Activity#requestPermissions`来请求权限  
如果第一次用户没有授权，则会回调`Activity#onRequestPermissionsResult`返回用于操作结果  
如果用于设置不再询问则选线对话框不会再弹出来，所有当用户检测到没有授权某一权限时，在请求权限之前需要弹出一个对话框告诉用户只有授权该权限才能支持某个功能。  
shouldShowRequestPermissionRationale方法


v4下的三个函数一直用于替换activity的三个对应函数

- ContextCompat.checkSelfPermission()  
无论在哪个版本，该函数都会返回PERMISSION_GRANTED 或者 PERMISSION_DENIED

- ActivityCompat.requestPermissions()在6.0以前版本调用，则OnRequestPermissionsResultCallback will be suddenly called with correct PERMISSION_GRANTED or PERMISSION_DENIED result.

- ActivityCompat.shouldShowRequestPermissionRationale()在6.0之前一直返回flase






# 参考文章
[Everything every Android Android Developer must konw about new Android's Runtime Permission](https://inthecheesefactory.com/blog/things-you-need-to-know-about-android-m-permission-developer-edition/en)
