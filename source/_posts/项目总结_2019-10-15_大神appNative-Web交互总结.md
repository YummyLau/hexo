---
title: 网易大神JsSdk-移动端侧的设计开发
layout: post
date: 2019-10-15
comments: true
categories: Android
tags: [项目,总结,Android]
---
<!--more-->

### 背景

现在基本的移动端项目都离不了于 Web 侧页面的交互。在前期大神刚开始起步迭代过程中，我们使用前后端约定 json 格式及方法名，用原生的 **JavaScriptEnabled** 方法来于 h5 交互。随着版本及用户量的上涨，我们的方法越来越多，业务越来越复杂。和前端同时沟通之后统一集成开发 h5 及 移动端侧的 Js-Sdk 用于大神与 h5 的所有交互。

### Android侧设计与实现

#### 使用 X5的WebView 替换原生WebView

在我们的线上统计中，部分机型的原生 WebView 在显示 h5 的时候出现显示错位/加载慢的场景。根据官方文档及众多渠道的归纳，至少存在一下问题：

* addJavascriptInterface 4.2以下存在漏洞
* WebView 密码明文存储漏洞 /data/data/com.package.name/databases/webview.db
* webView耗电的问题，我们之前发现的一个情况是，webView切换到后台时，如果当前页面有JS代码仍在不时的run, 就会导致比较严重的耗电，所以必须确保切换到后台后暂停JS执行，同时切回来的时候恢复它
* 闪屏问题，和硬件加速有关
* 选择文件存在兼容
* ...

故集成 X5 的Webview替换原生 WebView。[x5.tencent链接](https://x5.tencent.com/tbs/index.html)

#### 集成JsBridge

[JsBridge](https://github.com/lzyzsd/JsBridge) 是一个非常优秀的桥库，android侧使用 Java 与之交互。但初期我们在使用该库的时候发现很多问题，基本都存在于 Issues 中。所以 clone 下了源码之后，直接集成到项目，与X5的 WebView 统一开发城X5JsWebView。 代码经过测试之后存放在 [X5JsWebView 模块](https://github.com/YummyLau/AndroidModularArchiteture/tree/master/libWebview) 中可直接使用。这里就不贴代码了。

#### X5JsWebview模块类图/流程图

<img src="https://github.com/YummyLau/hexo/blob/master/source/pics/20191015/2_1.png?raw=true"  alt="类图" align=center />

<img src="https://github.com/YummyLau/hexo/blob/master/source/pics/20191015/2_2.png?raw=true"  alt="流程图" align=center />

#### 一次完整的Js-Native 交互流程

<img src="https://github.com/YummyLau/hexo/blob/master/source/pics/20191015/2_3.png?raw=true"  alt="交互流程" align=center />

### 思考

在业务迭代过程中优先选择使用官方提供的api解决业务问题。随着业务迭代及问题的产生，要搞懂问题的原因及找到解决问题的方法。针对解决问题的策略，优先选择目前已有的优秀方案，懂得站在巨人的肩膀上学习，才能看得更远。