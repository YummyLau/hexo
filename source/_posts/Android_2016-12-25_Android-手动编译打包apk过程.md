---
title: Android-手动编译打包apk
layout: post
date: 2016-12-25
comments: true
categories: Android
tags: [Android打包]
---
<!--more-->
>　　现在项目组的打包工作基本借助gradle化集成打包。最近想熟悉下gradle的自动化构建原理以及后续想学习下apk包体优化及热修复，过程中了解了这个打包过程。故自己动手打包体验一下，加深细节了解。

# 概念了解
* **aar**，即Android Archive，是Android库项目的二进制归档文件。
* **aapt**，即Android Asset Packaging Tool，SDK的build-tools目录的可执行文件，可查看，创建， 更新ZIP格式的文档附件(zip, jar, apk)，也可将资源文件编译成二进制文件。
* **aidl**，即Android Interface Definition Language Android接口定义语言，定义android跨进程通信接口文件。

# apk结构
* AndroidMainfest.xml:编译后的压缩文件，包括了一些应用的信息如包名、版本号、权限、组件注册。
* resources.arsc :通过AAPT编译后的资源索引表文件。
* res目录: 存放APP的资源，包括图片，字符串、布局等等文件。
* classes.dex:java源码编译后生成的java字节码文件被转化为android虚拟机可执行文件
* META-INF目录：存放的是签名信息，用来保证apk包的完整性和系统的安全。

# 构建打包过程
* 测试项目目录为：**E:\AndroidArea\AndroidWorkspace\PackageTest**
* 打包过程中产生的中间文件都存放在 **E:\AndroidArea\AndroidWorkspace\PackageTest\topack**中
* aapt版本/目录：**E:\AndroidArea\AndroidSdk\build-tools\23.0.3\aapt.exe**
* dx工具目录：**E:\AndroidArea\AndroidSdk\build-tools\23.0.3\dx**
* zipalign工具目录：**E:\AndroidArea\AndroidSdk\build-tools\23.0.3\zipalign**
* 安装过jdk1.8版本

## 生成R.java文件

```
执行命令：
E:\AndroidArea\AndroidSdk\build-tools\23.0.3\aapt.exe package -f 
-m -J .\topack\gen 
-S app\src\main\res 
-M app\src\main\AndroidManifest.xml 
-I E:\AndroidArea\AndroidSdk\platforms\android-23\android.jar
参数说明：
-f 如编译生成的文件已存在则覆盖。
-m -J E:\AndroidArea\AndroidWorkspace\topack\gen 在gen目录下生成带包路径的R.java
-S app\src\main\res  指定项目res文件夹的路径
-M app\src\main\AndroidManifest.xml  指定项目AndroidManifest.xml路径
-I 指定某个版本平台的android.jar文件的路径
如果项目中还存在assert文件夹，则还需要-A 指定assert文件夹的路径（这里没有用到）
执行结果：
在.\topack\gen 生成R.java文件，该文件实际上是为Android项目资源定义了很多ID引用。
```

## 生成.class文件

```
执行命令：
javac -source 1.6 -target 1.6 
-bootclasspath E:\AndroidArea\AndroidSdk\platforms\android-23\android.jar 
-d .\topack\bin .\app\src\main\java\com\example\yummylau\packagetest\*.java topack\gen\com\example\yummylau\packagetest\R.java 
参数说明：
javac -source 1.6 -target 1.6 制定生成的.class文件能在vm1.6版本中运行
-bootclasspath E:\AndroidArea\AndroidSdk\platforms\android-23\android.jar  制定使用到的android源码文件
-d <.class文件存放目录> <项目源码目录> <上步骤生成的R.java文件>
执行结果：
在.\topack\bin 生成许多.class文件，实际上步骤需要编译的源码包括：android jar、项目源码、第三方依赖包源码。
```

## 生成dex文件

```
执行命令：
E:\AndroidArea\AndroidSdk\build-tools\23.0.3\dx --dex --output=.\topack\bin\classes.dex .\topack\bin
参数说明：
--dex --output=.\topack\bin\classes.dex .\topack\bin 利用.\topack\bin目录下的.class文件在.\topack\bin目录生成classes.dex。
注意点：
1. Caused by: com.android.dx.cf.iface.ParseException: bad class file magic (cafebabe) or version (0034.0000)错误
原因是我jdk版本使用的是1.8版本太高，后面在生成.class文件步骤中指定生成1.6版本可解决
2. class name (com/example/yummylau/packagetest/R$style) does not match path (R$style.class)错误
原因是生成class文件自带目录记录，我的项目包名为:**com.example.yummylau.packagetest**,在指定class文件目录应该是
.\topack\bin 而不是.\topack\bin\com\example\yummylau\packagetest
执行结果：
在.\topack\bin下生成classes.dex文件，项目的核心代码都在此，分dex场景这里暂时不讨论，先走完流程先。
```

## 打包项目资源

```
执行命令：
E:\AndroidArea\AndroidSdk\build-tools\23.0.3\aapt.exe package -f 
-S app\src\main\res 
-M app\src\main\AndroidManifest.xml 
-I E:\AndroidArea\AndroidSdk\platforms\android-23\android.jar 
-F .\topack\bin\resources.ap_  
参数说明：
-F .\topack\bin\resources.ap_ 
注意点：
1. Caused by: com.android.dx.cf.iface.ParseException: bad class file magic (cafebabe) or version (0034.0000)错误
原因是我jdk版本使用的是1.8版本太高，后面在生成.class文件步骤中指定生成1.6版本可解决
2. class name (com/example/yummylau/packagetest/R$style) does not match path (R$style.class)错误
原因是生成class文件自带目录记录，我的项目包名为:**com.example.yummylau.packagetest**,在指定class文件目录应该是
.\topack\bin 而不是.\topack\bin\com\example\yummylau\packagetest
执行结果：
在.\topack\bin下生成resources.ap_ ，该文件包括了res资源、编译后的AndroidManifest.xml,resources.arsc(资源映射表)。
```

## 生成未签名apk

目前sdk tools目录下apkbuilder已经被移除，可以把tools目录下android.bat进行修改
“复制android.bat一份并重新命名为apkbuilder.bat,把apkbuilder.bat中**change com.android.sdkmanager.Main**替换为**com.android.sdklib.build.ApkBuilderMain**”
```
执行命令：
E:\AndroidArea\AndroidSdk\tools\apkbuilder E:\AndroidArea\AndroidWorkspace\PackageTest\topack\bin\unsign_app.apk -v -u 
-z E:\AndroidArea\AndroidWorkspace\PackageTest\topack\bin\resources.ap_ 
-f E:\AndroidArea\AndroidWorkspace\PackageTest\topack\bin\classes.dex 
-rf E:\AndroidArea\AndroidWorkspace\PackageTest\app\src\main\java
参数说明：
-v 构建过程显示log信息
-u 创建一个无签名的包
-z 指定resources.ap_资源路径
-f 指定dex文件路径
-rf 指定源码路径
执行结果：
在.\topack\bin生成unsign_app.apk，该apk还不能安装，提示解析失败，需要签名。
```

## 生成密钥

```
执行命令：
keytool -genkey -alias release -keyalg RSA -validity 20000 -keystore release.keystore 
参数说明：
-genkey 生成公钥、私钥和证书
-alias release 指定别名为release
-keyalg RSA 指定加密算法
-validity 20000 指定创建证书的有效天数20000
-keystore release.keystore 指定密钥库的名称
执行结果：
在项目根目录生成release.keystore。
```

## 对apk签名

```
执行命令：
7. 进行签名
jarsigner -verbose 
-keystore release.keystore 
-storepass ******
-keypass ******
-signedjar .\topack\bin\signed_app.apk .\topack\bin\unsign_app.apk release 
参数说明：
-verbose  签名过程显示log信息
-keystore release.keystore  指定密钥库的路径
-storepass <密钥库密钥>
-keypass <密钥密码>
-signedjar <生成已签名apk路径> <未签名apk路径> 密钥别名
执行结果：
在.\topack\bin生成已签名的app signed_app.apk 
```

## 4字节对齐优化

```
执行命令：
E:\AndroidArea\AndroidSdk\build-tools\23.0.3\zipalign 
-v 4 .\topack\bin\signed_app.apk .\topack\bin\signed_zip_app.apk
参数说明：
-v <align> 未对齐apk路径 对齐apk路径 align为字节数
验证是否对齐成功：
E:\AndroidArea\AndroidSdk\build-tools\23.0.3\zipalign -c -v 4 .\topack\bin\signed_zip_app.apk
执行结果：
在.\topack\bin生成已对齐的app signed_app.apk 
```

自此，整个apk的打包过程基本完成。



