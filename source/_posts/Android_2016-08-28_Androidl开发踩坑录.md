---
title: Android开发踩坑录
layout: post
date: 2016-08-28
comments: true
categories: Android
tags: [Android经验]
---
<!--more-->
# View
* `Recyclerview`调用notifyItemRemoved方法移除某个Item之后可能因为引用position引起crash。  
解决思路：`notifyItemRemoved`方法并不会移除列表的数据源的数据项导致数据源中的数据与列表Item数目不一致，需要同步刷新数据源。

* `Recyclerview`局部刷新Item时会因为默认动画导致闪烁。  
解决思路：因为`recyclerview`存在ItemAnimator，且在删除/更新/插入Item时会触发，可设置不支持该动画即可。
```
((SimpleItemAnimator)recyclerView.getItemAnimator()).setSupportsChangeAnimations(false);
```

* `Edittext`监听未获取焦点的Edittext的点击事件，第一次点击触发OnFocusChangeListener，在获取焦点的情况下才能响应onClickListener

* 使用listview或gridview的处理item的state_selected事件是无效的，可以在xml布局中对listview或gridview设置**Android:choiceMode="singleChoice"**,并使用state_activated状态来代替state_selected状态。（2016.12.10）

* 解决5.0以上Button自带阴影效果  
在xml定义的Button中，添加以下样式定义
```
style="?android:attr/borderlessButtonStyle"
```

#线程
* 新起线程不要随便调用网络请求，一般的newThread没有looper队列，参考handlerThread。

#编码

* 关于Java中字符与字节的编码关系 （2017.02.12）
```
unicode编码下1个中文字符或英文字符都占2个字节；
GBK编码下测试：
“java测试”.length() 返回 6，“GBK测试”.getBytes().length() 返回 8
总结：1个中文字符占2个字节  1个英文字符占1个字节
utf-8编码下测试：
“java测试”.length() 返回 6，“GBK测试”.getBytes().length() 返回 10
总结：1个中文字符占3个字节  1个英文字符占1个字节
utf-16编码下测试：
“java测试”.length() 返回 6，“GBK测试”.getBytes().length() 返回 14
总结：1个中文字符占3个字节  1个英文字符占2个字节
utf-32编码下测试：
“java测试”.length() 返回 6，“GBK测试”.getBytes().length() 返回 24
总结：1个中文字符或英文字符都占4个字节
```
