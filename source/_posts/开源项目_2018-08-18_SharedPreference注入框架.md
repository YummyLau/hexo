---
title: 开源项目之 SharedPreferencesInjector
layout: post
date: 2018-08-18
comments: true
categories: Android
tags: [开源项目]
---

#### 用于做什么
在日常开发中你可能会定义一个 SharedPreferencesUtils 助手用于处理轻量级数据存储 ， 但项目中依然会存在大量静态变量标识某一数据的 Key 或者零散分布着处理存储的代码。 希望某一类别的轻量级数据可被聚合并统一处理，并减少编写大量重复代码 。

*SharedPreferencesInjector* 通过 APT 及 javaPoet 技术在编译期获取自定义注解生成对应 SharedPreferences Helper操作类，帮助解决上述问题 。

* 定义一个聚合某些配置的类

```
@SharedPreferencesFile(fileName = "filename_if_you_want")       //可申明对应存储的SharedPreferences文件名
public class Config {

    @SharedPreferencesField(key = "key_if_you_want")            //定义你所需要的key，不设置则默认取变量名
    public int readCount;

    @SharedPreferencesField()
    public String configKey;

    @SharedPreferencesField()
    public Set<String> setStringData;
}
```

* 编译生成的类对应 {类名}_SPHelper，可直接调用操作

```
//全局初始化一次，建议在application#create 中处理
SharedPreferenceInjector.init(this);

//操作 Config 数据
boolean result = Config_SPHelper.put_configKey("newKey");       //同步操作
Config_SPHelper.put_configKey_async("newKey");                  //异步操作
Config_SPHelper.get_configKey("defaultKey");                    //获取

//清除 Config 数据
Config_SPHelper.clear();
Config_SPHelper.clear_async();
```

#### 如何引用
在对应模块下 `build.gradle` 添加
```
annotationProcessor 'com.effective.android:sp-compiler:1.0.1'
compile 'com.effective.android:sp-core:1.0.1'
compile 'com.effective.android:sp-annotation:1.0.1'
```

#### 期望
编写该项目只是希望能提高日常开发的效率，专注于处理业务 。 javapoet 是个好东西，也可以参考 apt 文件下的三个模块，自行利用注解扩展满足自身项目需求 。

如果更好的做法或者意见建议，欢迎写信到 yummyl.lau@gmail.com 。                                