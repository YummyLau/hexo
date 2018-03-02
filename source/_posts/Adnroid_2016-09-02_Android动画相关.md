---
title: Android动画相关
layout: post
date: 2016-09-02
comments: true
categories: Android
tags: [Android基础]
---
<!--more-->
# View Animation

>即Tween（补间）动画，该动画执行之后并不会真实改变View的布局属性，支持透明度（alpha）、伸缩（scale）、位移（translate）、平面旋转（rotate）4中动画处理。

* 推荐在res/anim文件下定义xml文件资源，支持定义一个或多个动画
```
<?xml version="1.0" encoding="utf-8"?>
<!-- res/anim/alpha.xml -->
<!-- fromAlpha 动画开始时的透明度，0为全透明，1.0为不透明，默认1.0 -->
<!-- toAlpha   动画结束时的透明度，0为全透明，1.0为不透明，默认1.0 -->
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
       android:duration="1000"
       android:fromAlpha="1.0"
       android:toAlpha="0.5"
    />
<?xml version="1.0" encoding="utf-8"?> 
<!-- res/anim/scale.xml -->
<!-- fromXScale 动画开始时x坐标的缩放倍数 -->
<!-- fromYScale 动画开始时y坐标的缩放倍数 -->
<!-- toXScale   动画结束时x坐标的缩放倍数 -->
<!-- toYScale   动画结束时y坐标的缩放倍数 -->
<!-- pivotX     开始做动画时固定组件x轴上的位置，可以使用百分比，也可以使用数值 -->
<!-- pivotY     开始做动画时固定组件y轴上的位置，可以使用百分比，也可以使用数值 -->
<!-- 如果  android:pivotX="0%"  android:pivotY="100%" 则表示固定控件左下角，从改点开始对x轴y轴做伸缩 -->
<!-- 如果  android:pivotX="0"  android:pivotY="100" 则表示固定在控件左上角正下方100px处，从该点开始对x轴y轴做伸缩-->
<scale xmlns:android="http://schemas.android.com/apk/res/android"
       android:duration="1000"
       android:fromXScale="1"
       android:fromYScale="1"
       android:pivotX="0%"
       android:pivotY="100%"
       android:toXScale="2"
       android:toYScale="2"
    />
<?xml version="1.0" encoding="utf-8"?>
<!-- res/anim/translate.xml -->
<!-- fromXDelta 动画开始时x坐标的偏移-->
<!-- fromYDelta 动画开始时y坐标的偏移 -->
<!-- toXDelta   动画结束时x坐标的偏移-->
<!-- toYDelta   动画结束时y坐标的偏移-->
<!-- 上述四个值的取值可以为 n（数值）、n%、n%p-->
<!-- 如果值为 n，则表示偏移为npx，即离原位置距离为npx，左负右正-->
<!-- 如果值为 n%，则表示偏移为width * n%，即离原位置距离为百分之n个自身宽度像素，左负右正 -->
<!-- 如果值为 n%p，则表示偏移为parent_width * n%，即离原位置距离为百分之n个父布局宽度像素，左负右正 -->
<translate xmlns:android="http://schemas.android.com/apk/res/android"
           android:duration="1000"
           android:fromXDelta="20%p"
           android:fromYDelta="0"
           android:toXDelta="100%"
           android:toYDelta="0"/>
<?xml version="1.0" encoding="utf-8"?>
<!-- res/anim/rotate.xml -->
<!-- fromDegrees 旋转开始时的角度，0为默认位置-->
<!-- toDegrees   旋转结束时的角度 -->
<!-- pivotX      旋转时旋转中心的x坐标-->
<!-- pivotY      旋转时旋转中心的y坐标-->
<!-- 上述pivotX、pivotY可以为 n（数值）、n%、n%p-->
<!-- 如果值为 n，则表示偏移为npx，即离原位置距离为npx，左负右正-->
<!-- 如果值为 n%，则表示偏移为width * n%，即离原位置距离为百分之n个自身宽度像素，左负右正 -->
<!-- 如果值为 n%p，则表示偏移为parent_width * n%，即离原位置距离为百分之n个父布局宽度像素，左负右正 -->
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="5000"
    android:fromDegrees="0"
    android:toDegrees="360"
    android:pivotX="50%"
    android:pivotY="50%"/>
<?xml version="1.0" encoding="utf-8"?> 
<set xmlns:android="http://schemas.android.com/apk/res/android" 
    <alpha 
        android:duration="2000"> 
        android:fromAlpha="1" 
        android:toAlpha="0"/> 
    <rotate 
        android:fromDegrees="0" 
        android:pivotX="50%" 
        android:pivotY="50%" 
        android:toDegrees="360"/> 
    <translate 
        android:fromXDelta="0" 
        android:fromYDelta="0" 
        android:toXDelta="100%" 
        android:toYDelta="0"/> 
    <scale 
        android:fromXScale="1" 
        android:fromYScale="1" 
        android:pivotX="100%" 
        android:toXScale="5" 
        android:toYScale="1"/> 
</set>
```
* 涉及的动画类：AlpaAnimation、ScaleAnimation、TranslateAnimation、RotateAnimation均继承于抽象类Animation，Animation拥有重置动画、开始/取消动画、判断当前动画是否开始/取消、监听动画等方法，可参考[Google Animation文档](https://developer.android.com/reference/android/animation/Animator.html)；
* 加载xml动画资源可直接用AnimationUtils的loadAnimation方法；
* 通过Interpolator的子类来实现动画过程的速率，官方提供了部分实现。

另外，除了在xml文件中定义动画，我们也可以在代码里面实现动画效果，但是在项目中尽量减少代码量，最好还是回归到xml文件中定义较好。
# Drawable Animation

>即Frame（帧）动画，该动画的执行时依赖一组drawable资源来“轮播”，每个资源之间可定义时间间隔。

* 推荐在res/drawable目录下定义xml文件资源
```
<?xml version="1.0" encoding="utf-8"?>
<!-- frame_animation.xml文件位于res/drawable/目录下 -->
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
                android:oneshot="true">
    <item
        android:drawable="@drawable/drawable1"
        android:duration="1000"/>
    <item
        android:drawable="@drawable/drawable2"
        android:duration="1000"/>
    <item
        android:drawable="@drawable/drawable3"
        android:duration="1000"/>
    <item
        android:drawable="@drawable/drawable4"
        android:duration="1000"/>
</animation-list>
```
* 设置drawable资源并启动动画
```
ImageView image = (ImageView) findViewById(R.id.image);
image.setBackgroundResource(R.drawable.frame_animation);
AnimationDrawable animation = (AnimationDrawable)image.getBackground();
animation.start();
```
**注意：动画依附的View需确保已经绘制完毕。**