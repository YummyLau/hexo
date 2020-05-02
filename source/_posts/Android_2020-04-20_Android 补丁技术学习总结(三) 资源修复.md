--- 
title: Android 补丁技术学习总结(三) 资源修复
date: 2020-05-02 18:00:03
comments: true
categories: Android进阶
tags: [热修复]
---

资源修复是很常见的操作，热修复方案中的资源修复很多参考了 **Instant Run** 的实现，**Instant Run** 的资源修复核心流程大致如下：

1. 构建一个新的 **AssetManager** 对象，并调用 *addAssetPath* 添加新的资源包；
2. 修改所有 **Activity** 的 *Activity.mAssets(AssetManager实例)* 引用指向新构建的 **AssetManager** 对象；
3. 修改所有 **Resource** 的 *Resource.mAssets(AssetManager实例)* 引用指向新构建的 **AssetManager** 对象.

对于任意的资源包，被 *AssetManager#addAssetPath* 添加之后，解析 **resourecs.asrc** 并在 native *mResources* 侧保存起来。可参考 [AssetManager.h](https://android.googlesource.com/platform/frameworks/base/+/master/libs/androidfw/include/androidfw/AssetManager.h) 的实现，实际上 *mResources*  是一个 **ResTable** 结构体,存放 **resourecs.asrc** 信息用的。而且一个进程只会有一个  **ResTable**。

* **ResTable** 可加载多个资源包
* 一个资源包都包含一个 **resourecs.asrc** 
* 每一个 **resourecs.asrc**  记录了该包的所有资源信息，每一个资源对应一个 **ResChunk**
* 每一个 **ResChunk** 都有一个唯一的编号，由该编号由三部分构成，比如 **0x7f0e0000**. 可以随便找一个 apk 解包查看 *resourecs.asrc* 文件。
	* 前两位 *0x7f* 为 package id，用于区分是哪个资源包
	* 接着两位 *0x0e* 为 type id，用于区分是哪类型资源，比如 drawable，string 等
	* 最后四位 *0x0000* 为 entry id，用于表示一个资源项，第一个为 *0x0000*，第二个为 *0x0001* 依次递增。

> 值得注意的是，系统的资源包的 package id 为 0x01，我们的 apk 为 0x7f

在应用启动之后，**ResourceManager** 在构建 **AssetManager** 的时候就已经加载了 apk 包的资源和系统的资源。如果补丁下发的资源 package id 也会 *0x7f* ，同时我们使用已有的 **AssetManager** 进行加载，在 **Android L** 版本之后会继续追加到已经解析资源的后面，但是由于相同的 package id 的原因，有可能在获取某个资源的时候，由于原 apk 已经存在近而忽略了补丁的新资源。故类 **Instant Run** 方案下，**AssetManager** 需要被完全替换才有效。

假如完整的替换 **AssetManager** ，则需要完整的资源包。则补丁包需要通过修复前后的资源包经过差异计算之后下发，再到客户端合成完整的新资源包，运行时耗费较多的时间和内存。

**Sophix** 给出了一种可以不用重新合成资源包的方案。该方案应用到 **Android L 及后续版本** 。同样是比较新旧资源包得到补丁资源包，然后通过修改补丁资源包的 package id 为 *0x66* ，并利用已有的 *AssetManager* 直接使用。这个补丁资源包要遵循一下规则：**“补丁包只包含新增的资源，包含纯新增的资源和修改旧包的资源，不包含旧包需要删除的资源”**。

* 纯新增的资源，代码处直接引用该资源；
* 旧包需要修改的资源，则新增修改后的对应资源，然后把代码处的资源引用指向修改后的资源；
* 旧包需要删除的资源，则代码处不引用该资源就好。（虽然会占着坑）

使用新资源包进行编译，则代码中可能出现资源 ID 的偏移，需要修正代码处的资源引用。

举个例子，比如原来有一个 **Drawable** 在代码的引用为 *0x7f0002*，由于新资源包新增了一个 **Drawable**，导致原 **Drawable** 在代码的引用为 *0x7f0003*，这个时候就需要把代码引用更改会原来的 *0x7f0002*。因为**Sophix** 采用的是 package id *0x66* 的补丁包加载而不是重新合成新的资源包。 同时，对于使用到补丁包涉及的资源，其引用也需要改成对应补丁资源引用 *0x66????* 。

这导致构建补丁资源变得非常复杂，需要懂得分析新旧资源包的 *resources.asrc* 及对系统资源加载的流程十分了解。


针对 **Android KK 及以下版本** ，为了避免和 **Instant Run** 一样创建新的 **AssetManager** 并做大量反射修改的工作，对原 **AssetManager** 对象析构和重构，具体就是让 native 层的 *AssetManager* 释放之前所有加载的资源然后及把 java 层的 *AssetManager* 对其的引用设置为空。之后 java 层的 *AssetManager* 在重新调用 *AssetManager#init* 方法驱动 native 创建一个没有加载过资源的 *AssetManager*。这样一来，java 层上层代码对 *AssetManager* 的引用就不需要修改了。然后在对其调用 *AddAssetPath* 添加所有资源包就可以了。

资源修复整体是围绕 **AssetManager** 展开的，本文只是记录了大体的思路，用来学习著名框架的设计思路及解决问题思考方法。中间细节自然存有一些难点兼容点需被攻克，感兴趣可查看参考资料一书。

**参考资料** —《深入探索 Android 热修复技术原理》
