---
title: markdowm语法记录
comments: true
date: 2016-10-21 15:32:26
categories: Markdowm
tags: [Markdowm]
---
<!--more-->
# 理解
* 标题用 <code>#</code>，共6级
* 列表分有无序，无论用 <code>*</code> 或 <code>-</code>，有序用 <code>1.</code>等，后追加一个空格
* 引用用 <code>></code>，支持嵌套
* 超链接 <code>[文本]\(网站\)</code>
* 图片需图传 <code>![文本]\(图片地址\)</code>
* 斜体一对 <code>\*</code> 或 <code>\_</code>包裹,粗体用一堆 <code>**</code> 或 <code>__</code>包裹
* 文章内锚点跳转

	```
	在标题写，比如 <h2 id="1">协议模型</h2> 标示一个id，然后在需要点击跳转的时候写 [协议模型](#1) 就可以了。
	```
* 文章外锚点跳转

	```
	在标题写，比如 <h2 id="1">协议模型</h2> 标示一个id，页面链接为 “https://myblog.com/article/one.html”，然后在需要点击跳转的时候写 [协议模型](https://myblog.com/article/one.html#1) 就可以了。
	```


## 参考文章
>[认识与入门Markdowm](http://sspai.com/25137)  
