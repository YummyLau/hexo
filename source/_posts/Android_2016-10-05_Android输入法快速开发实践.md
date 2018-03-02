---
title: Android输入法快速开发实践
layout: post
date: 2016-10-05
comments: true
categories: Android
tags: [Android输入法]
---
<!--more-->
# 常用属性
```
//activity和输入法是同时出现的，意味着activity启动后输入法也会启动。
android:windowSoftInputMode="stateVisible"
//activity中有edittext时也不显示输入法，也就是说只有当你点击了editText后输入法才会出现。
android:windowSoftInputMode="stateHidden"
//当输入法出现时，activity重新调整界面的布局，让原来的界面和输入法处于同一个平面中。
android:windowSoftInputMode="adjustResize"
//当输入框不会被遮挡时，该模式没有对布局进行调整，然而当输入框将要被遮挡时， 窗口就会进行平移。
android:windowSoftInputMode="adjustPan"
```
# 显示与隐藏
```
InputMethodManager inputMethodManager = (InputMethodManager)getSystemService(INPUT_METHOD_SERVICE);
inputMethodManager.showSoftInput(v, InputMethodManager.SHOW_FORCED); 显示
inputMethodManager.hideSoftInputFromWindow(v.getWindowToken(), 0); 隐藏
```
# 监听软键盘
```
 mBinding.getRoot().getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                // 应用可以显示的区域。此处包括应用占用的区域，
                // 以及ActionBar和状态栏，但不含设备底部的虚拟按键。
                Rect r = new Rect();
                mBinding.getRoot().getWindowVisibleDisplayFrame(r);
                // 屏幕高度，这个高度不含虚拟按键的高度
                int screenHeight = mBinding.getRoot().getRootView().getHeight();
                int heightDiff = screenHeight - (r.bottom - r.top);
                // 在不显示软键盘时，heightDiff等于状态栏的高度
                // 在显示软键盘时，heightDiff会变大，等于软键盘加状态栏的高度。
                // 所以heightDiff大于状态栏高度时表示软键盘出现了，
                // 这时可算出软键盘的高度，即heightDiff减去状态栏的高度
                if (keyboardHeight == 0 && heightDiff > ScreenUtil.sStatusBarHeight) {
                    keyboardHeight = heightDiff - ScreenUtil.sStatusBarHeight;
                }
                if (isShowKeyboard) {
                    // 如果软键盘是弹出的状态，并且heightDiff小于等于状态栏高度，
                    // 说明这时软键盘已经收起
                    if (heightDiff <= ScreenUtil.sStatusBarHeight) {
                        isShowKeyboard = false;
                        mBinding.chatswiperecyclerview.scrollBottomToShowNew();
                    }
                } else {
                    // 如果软键盘是收起的状态，并且heightDiff大于状态栏高度，
                    // 说明这时软键盘已经弹出
                    if (heightDiff > ScreenUtil.sStatusBarHeight) {
                        isShowKeyboard = true;
                        mBinding.chatswiperecyclerview.scrollBottomToShowNew();
                    }
                }
            }
        });
    }
```
上述代码参考于互联网，只做整理。  
值得注意的是，使用该方法处理仅适用于手机界面视图没有导航栏，如果有导航栏还需要把导航栏的高度考虑进去（典型的手机如华为荣耀6）。
