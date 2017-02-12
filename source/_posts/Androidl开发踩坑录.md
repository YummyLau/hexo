---
title: Android开发踩坑录
layout: post
date: 2016-08-28
comments: true
categories: Android
tags: [经验]
---
<!--more-->
* 调用notifyItemRemoved方法移除RecyclerView的某个Item时，需要移除数据源，否则可能因为position的问题引起crash。
* 监听未获取焦点的Edittext的点击事件，第一次点击触发OnFocusChangeListener，在获取焦点的情况下才能响应onClickListener。
* 新起线程不要随便调用网络请求，一般的newThread没有looper队列，参考handlerThread。
* 使用listview或gridview的处理item的state_selected事件是无效的，可以在xml布局中对listview或gridview设置**Android:choiceMode="singleChoice"**,并使用state_activated状态来代替state_selected状态。（2016.12.10）
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
