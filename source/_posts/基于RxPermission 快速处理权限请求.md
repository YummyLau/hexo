---
title: 基于 RxPermission 构建权限交互
date: 2020-04-01 18:24:00
comments: true
categories: Android
tags: [快速开发]
---

# Android 4.3
Application Operations出现

# Android 4.4
Api支持获取App Ops实例
```
Context.getSystemService(Context.APP_OPS_SERVICE)
```


# Android 6.0
如果操作需要某一项权限而App却没有被授予该权限，则会抛出异常。

1. 提高targetSdkVersion到23
意味着6.0以下的设备表现得和以往一样。而6.0及以上设备则会接受权限检测。
2. 用于的targetSdkVersion<23,则默认App会在安装的时候授权，但是对于6.0以上的设备依然可以主动撤销权限。
但是，如果我们需要用户授予某种权限来调用某个函数，在targetSdkVersion<23不会出现异常，如果对于需要返回值的函数，可能会返回0或者null。但是仍然可能因为我们函数的返回值而导致程序crash。


>系统权限分为`正常权限`和`危险权限`  
>正常权限不会直接给用户隐私权带来风险。如果您的应用在其清单中列出了正常权限，系统将自动授予该权限。  
>危险权限会授予应用访问用户机密数据的权限。如果您的应用在其清单中列出了正常权限，系统将自动授予该权限。如果您列出了危险权限，则用户必须明确批准您的应用使用这些权限。

正常权限包括  
从Api23起，包含以下正常权限

| 权限名称 | 权限说明 |
| --------------- | --------------- |
| ACCESS_LOCATION_EXTRA_COMMANDS | 允许应用程序访问额外的位置提供命令 |
| ACCESS_NETWORK_STATE | 允许程序访问有关网络的信息 |
| ACCESS_NOTIFICATION_POLICY | 为应用标记希望访问通知策略的权限 |
| ACCESS_WIFI_STATE | 允许应用程序访问Wi-Fi网络的信息 |
| BLUETOOTH | 允许应用程序连接到已配对的蓝牙设备 |
| BLUETOOTH_ADMIN | 允许应用程序发现和配对蓝牙设备 |
| BROADCAST_STICKY | 允许应用程序使用粘性广播，广播数据由系统完成并保持，客户端可以快速地检索数据，而不用等待下一个广播 |
| CHANGE_NETWORK_STATE | 允许程序修改网络连接状态 |
| CHANGE_WIFI_MULTICAST_STATE | 允许程序进入多播状态 |
| CHANGE_WIFI_STATE | 允许程序修改Wifi连接状态|
| DISABLE_KEYGUARD | 允许应用程序访问额外的位置提供命令 |
| EXPAND_STATUS_BAR | 允许应用程序访问额外的位置提供命令 |
| GET_PACKAGE_SIZE | 允许应用程序访问额外的位置提供命令 |
| INSTALL_SHORTCUT | 允许应用程序访问额外的位置提供命令 |
| INTERNET | 允许应用程序访问额外的位置提供命令 |
| KILL_BACKGROUND_PROCESSES | 允许应用程序访问额外的位置提供命令 |
| MODIFY_AUDIO_SETTINGS | 允许应用程序访问额外的位置提供命令 |
| NFC | 允许应用程序访问额外的位置提供命令 |
| READ_SYNC_SETTINGS | 允许应用程序访问额外的位置提供命令 |
| READ_SYNC_STATS | 允许应用程序访问额外的位置提供命令 |
| RECEIVE_BOOT_COMPLETED| 允许应用程序访问额外的位置提供命令 |
| REORDER_TASKS| 允许应用程序访问额外的位置提供命令 |
| REQUEST_IGNORE_BATTERY_OPTIMIZATIONS| 允许应用程序访问额外的位置提供命令 |
| REQUEST_INSTALL_PACKAGES| 允许应用程序访问额外的位置提供命令 |
| SET_ALARM| 允许应用程序访问额外的位置提供命令 |
| SET_TIME_ZONE| 允许应用程序访问额外的位置提供命令 |
| SET_WALLPAPER| 允许应用程序访问额外的位置提供命令 |
| SET_WALLPAPER_HINTS |允许应用程序访问额外的位置提供命令 |
| TRANSMIT_IR| 允许应用程序访问额外的位置提供命令 |
| UNINSTALL_SHORTCUT| 允许应用程序访问额外的位置提供命令 |
| VIBRATE| 允许应用程序访问额外的位置提供命令 |
| WAKE_LOCK| 允许应用程序访问额外的位置提供命令 |
| SET_WALLPAPER| 允许应用程序访问额外的位置提供命令 |
| WRITE_SYNC_SETTINGS| 允许应用程序访问额外的位置提供命令 |


权限通过组别来申请

* 通过`Activity#checkSelfPermission`来检测权限  
* 通过`Activity#requestPermissions`来请求权限，并通过 `Activity#onRequestPermissionsResult` 回调操作结果
* 通过 *shouldShowRequestPermissionRationale* 是一个提升用户体验的信息，字面上的意思是 **“是否应该显示请求权限的根本原因”**。
	* 在 6.0 版本之后，用户第一次请求权限，系统弹窗是没有 *checkbox* 可以勾选不再询问，所以 *shouldShowRequestPermissionRationale*  的值自然是 false；
	* 如果用户第一次拒绝权限之后，那么往后系统弹窗则会出现 *checkbox* 可以勾选不再询问。如果用户请求权限之后收到 onRequestPermissionsResult 反馈时，*shouldShowRequestPermissionRationale = true* 则意味着用户曾经拒绝权限且勾选了不再询问，此时你应该弹窗告诉用户 **“为什么我们需要你授权并引导用户去授权”**。

权限申请结果

* PackageManager.PERMISSION_GRANTED = 0 表示通过
* PackageManager.PERMISSION_DENIED = 0 表示未通过

v4包下的三个函数用于替换activity的三个对应函数

* ContextCompat.checkSelfPermission(),无论在哪个版本，该函数都会返回PERMISSION_GRANTED 或者 PERMISSION_DENIED
* ActivityCompat.requestPermissions()在6.0以前版本调用，则OnRequestPermissionsResultCallback will be suddenly called with correct PERMISSION_GRANTED or PERMISSION_DENIED result.
* ActivityCompat.shouldShowRequestPermissionRationale()在6.0之前一直返回 flase

### >=Android6.0 的终极方案

> 善于利用别人的轮子来提升效率，才是编程的最佳手段。

一开始，我们维护了一套针对不同 context 场景的权限请求，后面发现处理 Result 的回调的时候很麻烦。后来发现 [RxPermissions](https://github.com/tbruyelle/RxPermissions) 处理得非常完美。结合我们自己的项目使用场景，基于 RxPermissions 基础封装我们自己的权限库，针对 6.0 以上的版本进行权限动态判断。

关于 RxPermissions 的原理解析,可以参考 [系统源码解析——RxPermissions]()

参考项目维护的旧权限请求模块,我们希望对 RxPermissions 进一步改进,基于 **RxPermissions** 封装了 PermissionManager用于代理 *rxPermissions* 功能,并收集请求返回的 *Disposable* 对象,统一解绑避免内存泄漏. 同时针对项目一次/多次请求权限的场景,封装了 **ResultCall** 和 **MultiResultCall** 用于简化一次/多次请求结果的回调。

则从 **RxPermissions** 扩展的类图结构变为

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200401/rxpermission_3.png" align=center  />

时序图结构更改为

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200401/rxpermission_4.png" align=center  />

目前整套流程稳定服务于项目场景。


# 参考文章
* [Everything every Android Android Developer must konw about new Android's Runtime Permission](https://inthecheesefactory.com/blog/things-you-need-to-know-about-android-m-permission-developer-edition/en)
* [RxJava](https://github.com/ReactiveX/RxJava)
* [RxPermission](https://github.com/vanniktech/RxPermission)


