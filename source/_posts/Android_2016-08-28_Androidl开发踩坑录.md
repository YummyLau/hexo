---
title: Android开发踩坑录(持续更新...)
layout: post
date: 2019-06-13
comments: true
categories: Android
tags: [Android经验]
---
<!--more-->
# View
### RecyclerView

* `Recyclerview`调用notifyItemRemoved方法移除某个Item之后可能因为引用position引起crash。

	`notifyItemRemoved`方法并不会移除列表的数据源的数据项导致数据源中的数据与列表Item数目不一致，需要同步刷新数据源。

* `Recyclerview`局部刷新Item时会因为默认动画导致闪烁。
  
	因为`recyclerview`存在ItemAnimator，且在删除/更新/插入Item时会触发，可设置不支持该动画即可。
```
((SimpleItemAnimator)recyclerView.getItemAnimator()).setSupportsChangeAnimations(false);
```

### AppBarLayout

* 如何控制 appbarLayout 随时定位到某个位置

	```
	CoordinatorLayout.Behavior behavior =((CoordinatorLayout.LayoutParams)mAppBarLayout.getLayoutParams()).getBehavior();
	if (behavior instanceof AppBarLayout.Behavior) {
	      AppBarLayout.Behavior appBarLayoutBehavior = (AppBarLayout.Behavior) behavior;
	      int topAndBottomOffset = appBarLayoutBehavior.getTopAndBottomOffset();
	      if (topAndBottomOffset != 0) {
	             appBarLayoutBehavior.setTopAndBottomOffset(0);
	      }
	```

* 如何禁止 appbarLayout 滚动

	```
	CoordinatorLayout.LayoutParams params = (CoordinatorLayout.LayoutParams) appBarLayout.getLayoutParams();
	AppBarLayout.Behavior behavior = (AppBarLayout.Behavior) params.getBehavior();
	behavior.setDragCallback(new AppBarLayout.Behavior.DragCallback() {
	    @Override
	    public boolean canDrag(@NonNull AppBarLayout appBarLayout) {
	        return false;
	    }
	});
	```



### Edittext

* edittext未能响应onClickListener事件

	`Edittext`监听未获取焦点的Edittext的点击事件，第一次点击触发OnFocusChangeListener，在获取焦点的情况下才能响应onClickListener
	
	


### 其他view

* 使用listview或gridview的处理item的state_selected事件是无效的

	在xml布局中对listview或gridview设置**Android:choiceMode="singleChoice"**,并使用state_activated状态来代替state_selected状态。（2016.12.10）

* 解决5.0以上Button自带阴影效果  

	在xml定义的Button中，添加以下样式定义
	```
	style="?android:attr/borderlessButtonStyle"
	```
	
# 系统适配
* 如何解决 NatigationBar 被 PopupWindow 遮挡的问题

	```
	popupWindow.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);
	```
	
* 如何解决MIUI系统后台无法 Toast 的问题

	参考 https://github.com/zhitaocai/ToastCompat_Deprecated 项目，但是在小米3或者小米Note（4.4.4）手机上
	```
	mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
	```
	mContext 需要使用 ApplicationContext 才能生效。
	
* 如何解决关闭通知栏权限无法弹出toast的问题

	由于谷歌把 Toast 设置为系统消息权限，可以参考 [Android关闭通知消息权限无法弹出Toast的解决方案](http://w4lle.com/2016/03/27/Android%E5%85%B3%E9%97%AD%E9%80%9A%E7%9F%A5%E6%B6%88%E6%81%AF%E6%9D%83%E9%99%90%E6%97%A0%E6%B3%95%E5%BC%B9%E5%87%BAToast%E7%9A%84%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/) 维护自己的消息队列.
	
* 如何适配 vivo 等双面屏幕
	在 AndroidManifest.xml 中声明一下 meta-data
	
	```
	<meta-data
	android:name="andriod.max_aspect" android:value="ratio_float"/>
	```
	或者使用 `android:maxAspectRatio="ratio_float"(API LEVEL 26)`
	ratio_float 一般为屏幕分辨率高宽比。
	其他比如凹槽区域，圆角切割等问题可以参考市面上最新的vivo机型对应的 [vivo开放平台](https://www.jianshu.com/p/4ed3b38d888d) 文档。
	
* 华为设备产生太多 broadcast 导致crash的问题

	由于部分华为中，如果app注册超过500个BroadcastReceiver就会抛出 “ *Register too many Broadcast Receivers* ” 异常。通过分析发现其内部有一个白名单，自己可以通过创建一个新的app，使用微信包名进行测试，发现并没有这个限制。通过反射 LoadedApk 类拿到 mReceiverResource 中的 mWhiteList 对象添加我们的包名就可以了。 可以参考 https://github.com/llew2011/HuaWeiVerifier 这个项目。



# 线程

* 新起线程不要随便调用网络请求，一般的newThread没有looper队列，参考handlerThread。

# 网络

* Retorfit get 请求参数出现错误

	```
	@GET( BASE_URL + "index/login" )
	Observable< LoginResult > requestLogin( @QueryMap(encoded = true) Map< String, String > params );
	 
	final Map< String, String > paramsMap = new ParamsProvider.Builder().
	        append( "username", account ).
	        append( "password",URLEncoded(password) ).
	
	```
	比如登陆，encode = true 表示未对特殊字符进行url编码，默认是false。


# Git

* 修改 commit 记录

	```
	git reset --soft HEAD^
	```
	撤销当前的commit，如果只是修改提示，则使用
	
	```
	git commit --amend
	```

* git ignore 忽略文件不生效
	
	```
	git rm -r --cached .
	git add .
	git commit -m 'update .gitignore'
	
	```

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
