---
title: Android文件系统总结——从 Android Q “开始”
date: 2020-04-08 15:24:00
comments: true
categories: Android进阶
tags: [Android进阶]
---

> Android 系统的文件结构相对复杂，目录繁多。对于上层应用开发者来说，针对系统的分区操作及各个分区的掌握可按照下面几个分点掌握。对于boot， bootloader，recoverty相对底层的分区可按需了解即可。

AndroidStudio 提供了 **Device File Explorer** 可实时查看设备文件系统。部分分区要求 **Root** 才能查看，可自行刷机或者使用可 **Root** 配置的模拟器。其中 system 分区可做了解，data 和 storage 分区需要重点掌握。同时，随着 Android 10 推出 [scoped-storage](https://developer.android.com/training/data-storage#scoped-storage)，以往通过文件路径读取 storage 分区可能失效。并且往后随着系统更新，存储框架的约束并定越收越紧。有必要对整个安卓应用所涉及的存储空间有熟悉的了解，同时做好兼容 Android10 [scoped-storage](https://developer.android.com/training/data-storage#scoped-storage) 的准备工作。

### system分区

始终存在且只读挂载，刷 ROM 的分区.

查看方式

```
/system
Environment.getRootDirectory().getAbsolutePath()
```
主要目录

* `/system/{packageName}` 预装 apk 目录，系统应用
* `/system/lib` 自带 apk 用到的库文件
* `/system/bin` 存放一些可执行文件，基本上都是由 C/C++ 编写，比如 app_process 用于启动 Zygote
* `/system/xbin` 存放一些可执行文件，目录可为空
* `/system/framework` 启动系统所用到的框架



### data分区（Internal Storage）

Internal Storage,操作该区域存储文件，需要 root 权限操作，用于存放应用内的重要信息，

查看方式

```
/data
Environment.getDataDirectory().getAbsolutePath()
```
主要目录

* `/data/app` 用于包含许多 apk 的文件列表
* `/data/cache` 用于保存临时缓存或者下载的文件
* `/data/data` 用于存储 app 的数据
* `/data/data/{packageName}` 以包名形式区分，为app的私有存储空间，app卸载之后会删除该包下的文件
	* `/database`用于存储数据库文件
	* `/shared_prefs` 用于存储 shared 文件
	* `/lib`用于存储 so 库
	* `/code_cache `优化过的代码缓存
	* ...
	* `/cache`缓存空间， context.getCacheDir() 获取
	* `/files` 数据存储空间，contet.getFilesDir() 获取

	一般的，一个应用的内部存储路径未 `/data/data/{packageName}/` 。但是对于特殊机型比如华为，小米可能为 `/data/user/0/{packageName}/`
	
	```
	/data/data/{packageName}/files/
	context.getFilesDir().getAbsolutePath();
	
	/data/data/{packageName}/cache/
	context.getCacheDir().getAbsolutePath();
	```


### storage 分区（External Storage/Shared Storage）

External Storage/Shared Storage,不需要 root 权限就可以操作。可能包含可移除的存储介质，在使用之前需要判断是否挂载（mounted）

> 对于 4.4 以前的手机，设备存储很小，存在一个内置的存储空间，这部分空间就是内部存储。另外，还支持一个可以移除的存储介质，就是外部存储，比如 SD 卡。随着硬件升级，大部分大于 Android 4.4 的设备内置的存储空间可以划分为 “内部存储” 和 “外部存储”。同时，若此时还支持插入 SD 卡，则外部存储空间等于 “外部存储” 和 “SD卡存储” 之和。

查看方式

```
/storage/emulated/0
@Deprecated  Android10 版本上不再推荐使用该 Api
Environment.getExternalStorageDirectory() 
```
主要目录

* `storage/emulated/0/Android/`
   * `media/{packageName}`,以包名的形式区分，app的私有多媒体空间,5.0 Api可用
   * `obb/{packageName}`,以包名的形式区分，游戏 obb 数据文件
   * `data/{packageName}` 以包名的形式区分，app的私有存储空间
		* `/cache` 缓存空间，可通过 `context.getExternalCacheDir()` 获取
		* `/files` 数据存储空间，可通过 `context.getExternalFilesDir()` 获取, Android 10 通过以下方法进一步操作
			* `/Music`, 通过 `context.getExternalFilesDirs(Environment.DIRECTORY_MUSIC)` 获取
			* `/Podcasts`, 通过 `context.getExternalFilesDirs(Environment.DIRECTORY_PODCASTS)` 获取
			* `/Ringtones`, 通过 `context.getExternalFilesDirs(Environment.DIRECTORY_RINGTONES)` 获取
			* `/Alarms`, 通过 `context.getExternalFilesDirs(Environment.DIRECTORY_ALARMS)` 获取
			* `/Notifications`, 通过 `context.getExternalFilesDirs(Environment.DIRECTORY_NOTIFICATIONS)` 获取
			* `/Pictures`, 通过 `context.getExternalFilesDirs(Environment.DIRECTORY_PICTURES)` 获取
			* `/Movies`, 通过 `context.getExternalFilesDirs(Environment.DIRECTORY_MOVIES)` 获取
	* `media`，`obb`,`data` Android10及以上 按包名为应用划分 **沙盒目录**，跟随 app 卸载而删除，外部无法访问
* `storage/emulated/0/Music/`  Android 10及以上 无法通过路径访问，SAF，MediaStore 可行
* `storage/emulated/0/Pictures/` 访问同上
* `storage/emulated/0/Ringtones/` 访问同上
* `storage/emulated/0/Alarms/` 访问同上
* `storage/emulated/0/Notifications/` 访问同上
* `storage/emulated/0/Podcasts/` 访问同上
* `storage/emulated/0/Movies/` 访问同上
* `storage/emulated/0/Download/` 访问同上
* `storage/emulated/0/DCIM/` 访问同上
* `storage/emulated/0/Documents/` 访问同上
* `storage/emulated/0/Screenshots/` 访问同上
* `storage/emulated/0/Audiobooks/` 访问同上


### Android 逻辑角度看

* **App-specific storage**
	* 存储类型：应用专用存储，私有目录
	* 使用方法：`getFilesDir()`,`getCacheDir()`,`getExternalFilesDir()`,`getExternalCacheDir()`,`getExternalMediaDirs()`
	* 操作权限：内部存储不需要权限，外部存储从 Android4.4 之后也不需要
	* 外部应用访问：无法访问内部存储，Android 10及以后无法外部存储
	* 卸载是否移除：**移除**

* **Preferences**
	* 存储类型：内部私有存储，键值对存在
	* 使用方法：Jetpack Preferences library
	* 操作权限：不需要
	* 外部应用访问：不可以访问
	* 卸载是否移除：**移除**
	
* **Databases**
	* 存储类型：内部私有存储，持久化结构
	* 使用方法：Room persistence library
	* 操作权限：不需要
	* 外部应用访问：不可以访问
	* 卸载是否移除：**移除**
	
* **Shared storage - Media**
	* 存储类型：共享存储，比如一些图片，视音频
	* 使用方法：MediaStore API
	* 操作权限：Android9或者更低版本都需要 `READ_EXTERNAL_STORAGE` 和 `WRITE_EXTERNAL_STORAGE`。Android10或更高版本在访问外部app才需要
	* 外部应用访问：可以访问，但是需要  `READ_EXTERNAL_STORAGE` 权限
	* 卸载是否移除：不移除

* **Shared storage - Documents、files**
	* 存储类型：共享存储，比如文档，文件
	* 使用方法：Storage Access Framework
	* 操作权限：不需要
	* 外部应用访问：可以访问，文件选择器可以扫描到
	* 卸载是否移除：不移除

**一些重要的建议**

1. 存储作用域 [scoped-storage](https://developer.android.com/training/data-storage#scoped-storage)
 针对 Android10 及以上设备，可通过 [requestLegacyExternalStorage](https://developer.android.com/training/data-storage/compatibility) 来控制，默认 `false` 。低版本设备可以通过设置为 `false` 来开启这个特性。以下观点均默认 Andriod 10及以上版本默认支持该特性。
2. 考虑到专用存储空间会随着 app 的卸载而被移除，针对那些用户希望独立于 app 而存在的文件，比如照片，多媒体等，不希望卸载 app 之后被删除则就应该使用共享存储来保留。
3. 如果希望外部应用可以访问到内部存储的文件，应该使用 **FileProvider** 结合 *FLAG_GRANT_READ_URI_PERMISSION* 使用。
4. `getCacheDir()` 可能因为设备内部存储空间不足而删除部分缓存文件，确保在读取之前检测文件是否存在
5. 如果 app 仅在应用内部为用户提供有价值的多媒体信息，则应使用 `getExternalFilesDir` 来保存这些多媒体信息,例如选择 `getExternalFilesDir(Environment.DIRECTORY_PICTURES)` 。
6. 使用 MediaStore 时，系统会自动扫描外部存储并针对媒体进行，目录为 `/storage/emulated/0/`
	* **MediaStore.Images**，被存储在 `DCIM/` 和 `Picture/`
	* **MediaStore.Video**,被存储在 `DCIM/`，`Picture/` 和 `Movies/`
	* **MediaStore.Audio**,被存储在 `Alarms/`，`Audiobooks/`，`Music/`，`Notifications/`，`Podcasts/`和`Ringtones/`
	* **MediaStore.Downloads**，被存储在 `Download/`, 这类型是 Android10 及以上可用
	* **MediaStore.Files**,只有 Andriod10及以上可用
7. 如果尝试使用外部存储的文件原始路径去访问，即使你有 `READ_EXTERNAL_STORAGE` 权限，在 android 10 及以上也会收到 `FileNotFoundException` 。应该改成使用 **MediaStore API** 来访问。
8. Android 10及以上版本的设备，每一个多媒体文件都有一个应用归属属性，非归属应用访问时需要授权 `READ_EXTERNAL_STORAGE` 权限。当应用卸载重装之后，访问之前保存的文件也会需要权限，原因是该保存的文件属于之前安装的应用。

### Android10及以上实践建议
* 通过控制 [android:requestLegacyExternalStorage] (https://developer.android.com/training/data-storage/compatibility) 属性来决定要不要开启新特性 [scoped-storage](https://developer.android.com/training/data-storage#scoped-storage)

	```
	<manifest ... >
	  <!-- This attribute is "false" by default on apps targeting
	       Android 10 or higher. -->
	    <application android:requestLegacyExternalStorage="true" ... >
	      ...
	    </application>
	</manifest>
	```

* 读取非共享外部存储已经不需要 `READ_EXTERNAL_STORAGE` 和 `WRITE_EXTERNAL_STORAGE`,因为整体是一个 **沙箱** 涉及，减少权限请求。只有读取其他app的共享外部存储数据才需要，代码向下兼容。
* 读取共享数据需要请求必要的权限，使用 **MediaStore API** 进行文件访问
* 针对其他应用创建的共享文件
	* 访问：请求必要权限 -> ContentResoler 查找并打开文件
	* 写入：当且仅当你的应用是系统某个功能的默认应用才可以，比如照片管理应用，默认音乐应用等。被设置默认角色之后 -> ContentResoler 查找并打开文件,执行编辑、变更操作。
* 当应用需要进入兼容 [scoped-storage](https://developer.android.com/training/data-storage#scoped-storage) ，面向用户的是允许或拒绝应用程序访问整个外部存储的权限，而不是像图片、视频或音乐等单独的共享集合。
* 应用无法在沙盒目录之外的路径新建文件，包括使用 `Environment.getExternalStorageDirectory()`.
* 应用无法直接使用路径访问沙盒之外的文件，`MediaStore` 只能访问公共目录下的多媒体文件`
* 应用无法分享文件，需要 `file://` 类型的 Uri 转换成 `content://` 类型

细节编码可直接查看 [Android Q 独立存储](https://shoewann0402.github.io/2019/03/17/android-q-beta-scoped-storage/) 一文
	
### 参考文档
* [官方 -- 应用数据和文件](https://developer.android.google.cn/guide/topics/data)
* [AndroidQ 适配-存储空间篇](https://juejin.im/post/5dc23ade6fb9a04ab745b74e)
* [Android Q 独立存储](https://shoewann0402.github.io/2019/03/17/android-q-beta-scoped-storage/)
* [Android Q版本应用兼容性适配指导](https://open.oppomobile.com/wiki/doc#id=10432)

	









