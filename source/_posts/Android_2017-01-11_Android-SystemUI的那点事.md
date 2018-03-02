---
title: Android-SystemUI的那点事
date: 2017-01-11 16:15:48
comments: true
categories: Android
tags: [Android进阶]
---
<!--more-->

关于Android系统蓝（状态栏、导航栏）透明的处理，可参考我的另一篇文章【Android系统栏开发实践】，链接见文章尾部。
这篇文章主要记录在开发直播间或点播场景下，视频展示场景与输入法显示的冲突及解决方案。读完这篇文章，你能了解到以下几个方面：
> 1. SystemUI实践心得
> 2. 如何设计横竖屏直/点播良好的交互
> 3. 如何解决输入法Resize模式在部分场景下失效的问题

# 关于SystemUI

这里不对SystemUI的基础用法做介绍，详情可见[官网SystemUI介绍](https://developer.android.com/training/system-ui/index.html)。  
说一下自己的拙见。开发过程中涉及**SystemUI**操作主要是状态栏和导航栏，市场上大多数应用实现了状态栏透明操作，部分还会处理导航栏透明（bilibili安卓客户端首页）。无论哪种场景，处理的问题无非三种：
1. 着色处理（透明色）
2. 隐藏处理
3. 布局是否layout到SystemUI区域。

## 着色处理

在【Android系统栏开发实践】文章中可知，可配置主题或者代码实现对**SystemUI**进行着色，而这种两种方式的有效期伴随着组件的生命周期的结束而解决，这里不做过多介绍，如下。

```
若主题配置 
<item name="android:windowTranslucentStatus">true</item>
上述配置会导致界面布局默认入侵到状态栏，对此暂时在界面根布局android:fitsSystemWindows="true"解决。

若代码设置
/**
 * 对状态栏进行着色
 * 可以在主题中设置4.4以上的状态栏透明，但是5.0以上的则为半透明。
 * 为了兼容4.4以上的版本都可以全透明，并根据业务需求对状态栏进行着色
 * @param activity
 * @param color
 */
public static void initSystemBar(Activity activity, int color) {
	if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT &&
			Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
		Window mWindow = activity.getWindow();
		mWindow.setFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS, 
		WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
		SystemBarTintManager mTintManager = new SystemBarTintManager(activity);
		mTintManager.setStatusBarTintEnabled(true);
		mTintManager.setStatusBarTintColor(color);
	} else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
		//兼容5.0及以上支持全透明
		Window window = activity.getWindow();
		window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
		activity.getWindow().getDecorView().setSystemUiVisibility(
				View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
				| View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
		window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
		window.setStatusBarColor(color);
	}
}
其实原理同第一种，暂时也需在界面根布局android:fitsSystemWindows="true"解决。
```

## 隐藏处理

除了可以配置**FULLSCREEN**主题或者对**Window**设置**FLAG_FULLSCREEN**的**flags**达到全屏效果,进而实现隐藏**SystemUI**外，全屏沉浸模式也同样可以达到效果。android4.4及以上版本通过**setSystemUiVisibility**设置**SYSTEM_UI_FLAG_HIDE_NAVIGATION**和**SYSTEM_UI_FLAG_FULLSCREEN**方法来达到隐藏状态栏和导航栏效果。后者的这种做法会伴随着用户在状态栏与导航栏原有区域的边缘向内滑动而抹除**flags**从而重新显示状态栏和导航栏。处理如下。

```
若配置全屏主题
任何涉及 FULLSCREEN 的主题或者设置 <item name="android:windowFullscreen">true</item>。
这种方式只能隐藏状态栏，对于有导航栏的手机，仍然不是全屏模式，这种模式下用户从顶部下滑唤出状态栏后会自动隐藏回去。

若设置window flags    
getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN,  WindowManager.LayoutParams.FLAG_FULLSCREEN);
效果完全和第一种一样。

若设置SYSTEM_UI_FLAG_HIDE_NAVIGATION或SYSTEM_UI_FLAG_FULLSCREEN
getWindow().getDecorView().getRootView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_FULLSCREEN);
理论上根据官网提供说明可以达到任何机型的全屏，不过当用户从系统栏向内侧方向滑动唤出系统栏后Flags就会被清除掉，用户可通过setOnSystemUiVisibilityChangeListener
监听SystemUI的变化来进行操作即可。（华为部分rom 荣耀6.0版本这种方案会对在状态栏留下白条区域 = 。=，第三方的真是伤不起）
```

## layout到SystemUI区域

一般来说，我们可以通过设置**setFitsSystemWindows(boolean)**来告诉系统为我们的app预留出**SystemUI**空间，进而通过**fitSystemWindows(rect)**允许appUI适应Window变化。此时如果Window是全屏则app布局会layout到SystemUI区域。实际上**setFitsSystemWindows**为**false**也可通过拿到*android.R.id.content*对应的布局并设置Margin来**侵占**控件，但强烈不建议这么做。还是乖乖遵循api标准吧。值得注意的是，输入法等系统窗口的唤出可能导致窗口重绘，比如非全屏模式下的**SOFT_INPUT_ADJUST_RESIZE**模式，如果**setFitsSystemWindows**为**false**意味着应用窗口不会随输入法的弹出而绘制布局。除了设置**setFitsSystemWindows**的属性来layout到SystemUI外，还可以使用**SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN**和**SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION**来达到效果，同第二点。总结如下。

```
<item name="android:windowTranslucentStatus">true</item>
允许我们的布局入侵到状态栏并默认入侵

<item name="android:windowTranslucentNavigation">true</item>
允许我们的布局入侵到状态栏和导航栏并默认入侵

手动设置padding或者margin，这个需要获取状态栏和系统栏高度，建议通过以下方法获取
private static final String STATUS_BAR_HEIGHT_RES_NAME = "status_bar_height";
private static final String NAVIGATION_BAR_HEIGHT_RES_NAME = "navigation_bar_height";
/**
 * 选择性获取状态栏、导航栏告诉
 *
 * @param res context.getResources()
 * @param key NAVIGATION_BAR_HEIGHT_RES_NAME 或 STATUS_BAR_HEIGHT_RES_NAME
 * @return
 */
private static int getInternalDimensionSize(Resources res, String key) {
	int result = 0;
	int resourceId = res.getIdentifier(key, "dimen", "android");
	if (resourceId > 0) {
		result = res.getDimensionPixelSize(resourceId);
	}
	return result;
}	

getWindow().getDecorView().getRootView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN);
设置SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN或SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION且（setFitsSystemWindows不配置||setFitsSystemWindows为false），同样默认入侵我们的状态栏和导航栏。
```

网上常看到讲述沉浸式状态栏大多都是设置把状态栏着色成布局背景色或状态栏着透明色且布局入侵系统栏同时设置布局的paddindTop为状态栏高度。

# 设计横竖屏交互
了解SystemUI的上述三种处理场景，基本可以满足现在市场上直点播app的良好交互。在开发实际项目中，综合主流视频类app（b站、腾讯、优酷、斗鱼等异步点播视频界面），基本需要满足场景：
>1. 竖屏状态
>	1. 顶部视频区域需入侵状态栏
>	2. 视频浮层（控制进度、发送弹幕等）的唤出与隐藏需要和状态栏同步
>	3. 导航栏透明（可选）
>2. 横屏状态
>	1. 粘性沉浸式
>	2. 视频浮层（控制进度、发送弹幕等）的唤出与隐藏需要和导航栏（如有）同步

考虑上述设计主要是竖屏状态下，保持视频区域沉浸风格能给用户带来更好的观看体验，而导航栏（部分机型比如华为荣耀6等）能给用户返回上一界面，不建议进行隐藏，但可选择性让其透明，根据底部展示的内容而定。而横屏状态下，考虑到app的视频格式为16:9，满屏的观看体验更优，需设置粘性沉浸式。而这里为什么不选择【隐藏处理】的前两个方法呢？原因是前两个配置会影响竖屏的配置，在横竖屏切换的时候需要动态修改Window布局配置且无法兼容有导航栏的手机，不建议如此，并且伴随着视频浮层的唤出需要对系统栏做显示，更不方便。所以选择**setSystemUiVisibility**来操作更灵活。

## Demo演示

这里以点播为例子：

<video id="video" controls="" preload="none" height=640 width=360 poster=""  autoplay muted >
      <source id="mp4" src="http://ojyx20l6a.bkt.clouddn.com/system_ui_test_demo.mp4" type="video/mp4">
    </video>

这里我们有一个权衡的问题：对于华为系用户随时可隐藏导航栏而言，如果底部有内容且可以点击（比如评论条），如果选择隐藏导航栏，则有导航栏的手机导航栏就消失了，失去了返回（其实还有Home等）的入口，当然我们可在视频左上添加一个返回上一级的按钮，但用户体验可能不太好。因为大多数使用有导航栏手机的用户都会习惯使用导航栏来返回，会因为我们app的导航栏隐藏而感觉操作不方便。因此，最多只能透明处理。如果选择透明化，要不要把内容入侵进去，不建议这么做，原因刚才说了，如果存在评论条，那么就破坏了原本界面的设计，只能通过设置padding或者margin来让布局贴在导航栏上面。那可恶心的事就发生了，如果用户突然自主隐藏导航栏，那么我们的布局下突然多了一块空白的区域...因此，我们在竖屏的时候舍弃了导航栏隐藏或者透明化的想法。

## 配置实现
下面配置是我们实际项目中的配置：

```
整个Activity的根布局设置 android:fitsSystemWindows="false" （默认）。

主题设置
<!-- values 视频播放页主题-->
<style name="VideoActivityTheme" parent="AppTheme">
	<item name="colorPrimary">@color/colorPrimary</item>
	<item name="colorPrimaryDark">@color/main_theme_text_color</item>
	<item name="colorAccent">@color/colorAccent</item>
	<item name="android:textColor">@color/main_theme_text_color</item>
	<item name="android:windowBackground">@color/main_theme_window_bg_color</item>
	<item name="actionBarSize">@dimen/actionBarSize</item>
	<item name="windowActionBar">false</item>
	<item name="windowNoTitle">true</item>
</style>

<!-- values-19 视频播放页主题-->
<style name="VideoActivityTheme" parent="AppTheme">
	<item name="colorPrimary">@color/colorPrimary</item>
	<item name="colorPrimaryDark">@color/main_theme_text_color</item>
	<item name="colorAccent">@color/colorAccent</item>
	<item name="android:textColor">@color/main_theme_text_color</item>
	<item name="android:windowBackground">@color/main_theme_window_bg_color</item>
	<item name="actionBarSize">@dimen/actionBarSize</item>
	<item name="windowActionBar">false</item>
	<item name="windowNoTitle">true</item>
	<item name="android:windowTranslucentStatus">true</item>
	<item name="android:windowTranslucentNavigation">false</item>
</style>

<!-- values-21 视频播放页主题-->
<style name="VideoActivityTheme" parent="AppTheme">
	<item name="colorPrimary">@color/colorPrimary</item>
	<item name="colorPrimaryDark">@color/main_theme_text_color</item>
	<item name="colorAccent">@color/colorAccent</item>
	<item name="android:textColor">@color/main_theme_text_color</item>
	<item name="android:windowBackground">@color/main_theme_window_bg_color</item>
	<item name="actionBarSize">@dimen/actionBarSize</item>
	<item name="windowActionBar">false</item>
	<item name="windowNoTitle">true</item>
   <item name="android:windowTranslucentStatus">true</item>
	<item name="android:windowTranslucentNavigation">false</item>
	<item name="android:statusBarColor">@android:color/transparent</item>
</style>

利用着色处理中对initSystemBar根据场景进行着色，在demo中比如竖屏下一般为透明，当用户点击暂停并向上滚动式会变成黄色。
```

处理完上述之后，状态栏已透明且布局入侵状态栏，接下来处理点击视频区域同步SystemUI的显示与隐藏。这里我写了一个处理工具类，先看场景代码:

```
package com.netease.bolo.android.helper;

import android.annotation.TargetApi;
import android.os.Build;
import android.support.annotation.NonNull;
import android.view.View;

import static android.view.View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN;
import static android.view.View.SYSTEM_UI_FLAG_LAYOUT_STABLE;

/**
 * 针对视频播放页SystemUI处理
 * Created by yummyLau on 2017-01-12 15:33.
 */

public class VideoSystemUIHelper {

    public static final int P_HIDE_STATUS = 0x0001;
    public static final int P_SHOW_STATUS = 0x0002;
    public static final int L_HIDE_STATUS = 0x0003;
    public static final int L_SHOW_STATUS = 0x0004;
    public static final int DEFAULT_P_STATUS = P_HIDE_STATUS;
    public static final int DEFAULT_L_STATUS = L_HIDE_STATUS;
    private int mSystemUIStatus = P_SHOW_STATUS;

    private View mRoot;
    private IOnSystemUIChangeListener mListener;


    public VideoSystemUIHelper(View root,@NonNull IOnSystemUIChangeListener listener){
        mRoot = root;
        mListener = listener;
    }

    /**
     * 处理横竖屏的systgemUi变化
     *
     * @param status
     */
    @TargetApi(16)
    public void handleSystemUI(int status) {
        mSystemUIStatus = status;
        int systemUiVisibility = 0;
        switch (status) {
            case P_SHOW_STATUS: {
                systemUiVisibility =
                        SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | SYSTEM_UI_FLAG_LAYOUT_STABLE;
                mListener.onSystemUIChange(P_SHOW_STATUS);
                break;
            }

            case P_HIDE_STATUS: {
                systemUiVisibility =
                        View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_FULLSCREEN;
                mListener.onSystemUIChange(P_HIDE_STATUS);
                break;
            }

            case L_SHOW_STATUS: {
                systemUiVisibility =  View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                        | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                        | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                        | View.SYSTEM_UI_FLAG_FULLSCREEN
                        | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
                mListener.onSystemUIChange(L_SHOW_STATUS);
                break;
            }

            case L_HIDE_STATUS: {
                systemUiVisibility =
                        View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                                | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                                | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                                | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                                | View.SYSTEM_UI_FLAG_FULLSCREEN
                                | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
                mListener.onSystemUIChange(L_HIDE_STATUS);
                break;
            }
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
            mRoot.setSystemUiVisibility(systemUiVisibility);
        }
    }

    public interface  IOnSystemUIChangeListener {
        void onSystemUIChange(int status);
    }
}

```

在这里，我把异步点播系统栏变化的场景分为4中：

>1. 竖屏状态栏（显示）导航栏（不处理）；
>2. 竖屏状态栏（隐藏）导航栏（不处理）；
>3. 横屏状态栏（隐藏）导航栏（显示）；
>4. 横屏状态栏（隐藏）导航栏（隐藏）。

第**4**种场景下选择*粘性沉浸式*的原因是 “可能存在用户不想操控视频浮层而只是想看看状态栏信息” 的场景，这种场景下系统栏能自动隐藏。在具体业务中，我们只需要响应视频浮层的显示与隐藏之后调用上述**handleSystemUI**方法设置flags即可。到此，我们能满足[设计横竖屏交互]中横竖屏的剩余场景。
实际上，当前已经产生一个间接的问题：
**如果采用上述方法实现观看端，那么你将无法监听输入法的高度。**
这就棘手了。一般直播或异步点播观看端，交互基本是必须的，会涉及用户输入，这会导致输入法的弹出。为了避免布局跳动，输入法模式建议不适用**SOFT_INPUT_ADJUST_PAN**，哪怕你适用**ScrollView**来回滚，遇到其他滑动的场景经常会有冲突；而为了平滑地显示适用**SOFT_INPUT_ADJUST_NOTHING**却无法计算**Keyboard**的高度；最终选择**SOFT_INPUT_ADJUST_RESIZE**模式，正常情况下采用该模式，界面窗口会随着输入法的弹出而重新绘制调整。但是**fitSystemWindow**为**false**时，界面是不会重绘的。实际上，**Android**有个**bug**，全屏模式下**SOFT_INPUT_ADJUST_RESIZE**模式同样失效。全屏模式？fitSystemWindow为false？都会？为什么？如果你认真看本文，因果关系应该会明白的。
那是不是没有一种输入法模式可以选择？不急，我们再看看观看端的其他需求，发评论？弹幕？最好都能支持。更人性化的呢？支持发表情？快捷回复？是的！竖屏的表情、快捷回复面板等样式可参考传统的IM（QQ或微信）软件，实现自由切换且输入布局不跳动；但是横屏呢？bilibili安卓客户端异步视频观看页面的做法是适用**SOFT_INPUT_ADJUST_PAN**但输入框位于界面顶部避免布局跳动（给人的感觉不太好），而直播间却不是！输入框能无缝贴在输入法上沿（这才是我们追求的姿势嘛），它展开的延迟动画给我的直觉是“那是一个弹窗啊！”Good idea！

## 兼容方案

如果要是能满足以下条件：

>1. 需要测量输入法高度；
>2. 需要支持表情、快捷回复等面板与输入法快速切换；
>3. 需要兼容全屏或fitSystemWindow为false的场景。

那就太完美了！有，就是新启一个**Window**来完成！那就用**PopupWindow**来做吧！写到这里有点多，这个Window怎么做到横竖屏通用，下篇文章吧~

>[Android系统栏开发实践](http://yummylau.com/2016/10/10/android%E7%B3%BB%E7%BB%9F%E6%A0%8F%E5%BC%80%E5%8F%91%E5%AE%9E%E8%B7%B5/)  
