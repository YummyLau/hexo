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
