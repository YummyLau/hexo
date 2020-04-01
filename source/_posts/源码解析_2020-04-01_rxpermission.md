---
title: 系统源码解析——RxPermissions
layout: post
date: 2020-04-01
comments: true
categories: Android
tags: [源码解析] 
---

> 最近在定位一个 魅族 机型的权限请求问题，刚好又走了一遍之前基于 RxPermission 封装的基础库。顺便就把整个库的逻辑重新看了一遍，这里就留作技术文档及解析，后面给团队的人了解并熟悉这个库底层是如何实现的。

[RxJava](https://github.com/ReactiveX/RxJava) 一个用于使用Java VM的可观察序列来组成异步和基于事件程序的库。换成人话就是 **“通过使用该库，用同步的代码编写方式实现异步的功能场景”** 。 我对它可算是一见钟情。

“扯远了喂”。

[RxPermissions](https://github.com/tbruyelle/RxPermissions) 是基于 [RxJava](https://github.com/ReactiveX/RxJava) 编写的一套用于 Android 6.0 及以上版本的系统用于协助请求系统权限的库。Android 官方在 [请求应用权限](https://developer.android.com/training/permissions/requesting?hl=zh-cn) 的篇章详细描述了运行时用户权限的各种场景及如何请求这些权限。 我们的项目也经历过自主在每个页面通过官方 api 编写请求，到后续基于官方 api 封装一套代理请求。虽然请求的代码被聚合统一了，但是十分不优雅。后来 [RxPermissions](https://github.com/tbruyelle/RxPermissions) 的接触让我感受了 **“智商的碾压”**。 

“同样是封装，基于 rxjava 实力碾碎了我们”。

[RxPermissions](https://github.com/tbruyelle/RxPermissions) 就三个类

* Permission 权限信息类
* RxPermisssionsFragment 权限代理请求类
* RxPermissions 封装了权限操作的集合类，对外开放操作 api

类信息如下图。

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200401/rxpermission_1.png" align=center  />

**RxPermissions** 在 [RxPermissions](https://github.com/tbruyelle/RxPermissions) 框架中，用户每一个场景都需要构建一个 *RxPermissions* 对象来完成请求工作。 (当然，自己的项目可以封装一个 **Manager** 来作为 **RxPermissions** 的工厂及 api 二次暴露也未尝不可，这里不做展开)。

单一场景请求时序如下。

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200401/rxpermission_2.png" align=center  />

这里以页面场景第一次请求权限为例子。当前页面初始化一个 *RxPermissions* 对象并调用 *request* 方法（框架提供了很多此类方法), *RxPermissions* 内部会在第一次触发权限交互时初始化一个 *rxPermisssionsFragment* 对象并把请求委托给他。同时 **RxPermissions#request** 的调用会返回一个 *Observable* 对象，而该对象实际上就是代理 *rxPermisssionsFragment* 所持有并返回的 *publicSubject* 对象。当代理收到系统的权限通知结果时，使用
*publicSubject* 进行事件触发，此时页面通过监听返回的  *Observable* 对象即可完成交互。

> 库的源码实际上比较简单，但是却实现了复杂的功能。灵活借助 rxjava 的库并把系统权限交互的过程流式化，大大简便了原业务场景的权限交互流程。并且使用不可见的 Fragment 来间接操控系统 api 的想法着实优秀!









