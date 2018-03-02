---
title: Android系统栏开发实践
layout: post
date: 2016-10-10 
comments: true
categories: Android
tags: [Android基础]
---
<!--more-->
# 策略
安卓的系统栏包括了顶部状态栏和底部导航栏，少数部分机型如nexus、华为荣耀6等在界面视图中存在可操作的底部导航栏。综合考虑安卓碎片化适配问题，遵循市场app主流设计风格，对系统栏做一下设计处理
>1. Activity页面遵循状态栏着色，导航栏按需处理。状态栏可选择透明处理或着色处理，导航栏处理的页面常见于闪屏页、引导页或者视频观看页等;
2. 统一在styles样式中设置theme，只处理4.4版本以上机型。截止2016/09/05，安卓官网数据表明4.4及以上版本机型占所有安卓手机比例为81.4，不必承担不可控的低版本适配风险;
3. 统一在xml或者代码设置是否fitsSystemWindows。具体风格由个人而定;
4. 统一在应用Acitivity基类中决定透明栏着色。子类默认着色选择colorPrimary，可根据业务自定覆盖选择其他颜色。

# 实现
## styles样式定义
```
    <!--Base application theme.  4.4 以下版本 -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="android:windowActionBar">false</item>
        <item name="android:windowNoTitle">true</item>
    </style>

    <!-- Base application theme. v4.4以上版本状态栏透明 -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="android:windowActionBar">false</item>
        <item name="android:windowNoTitle">true</item>
        <item name="android:windowTranslucentStatus">true</item>
    </style>

    <!-- Base application theme. v5.0以上版本状态栏透明，对状态栏赋值 -->
    <style name="AppTheme" parent="Theme.AppCompat.Light">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="windowActionBar">false</item>
        <item name="windowNoTitle">true</item>
        <item name="android:windowTranslucentStatus">false</item>
        <item name="android:statusBarColor">@android:color/transparent</item>
    </style>
```
注意：上述设置不对导航栏进行处理（theme遵循普遍性），如需要特殊处理再针对特定Activity写theme即可。5.0以上版本设置和4.4版本不同，原因在于按照4.4版本的设置windowTranslucentStatus，实际机型看到的效果为半透明，需要多设置一个透明值。
## 设置fitSystemWindows
```
    /**
     * 是否适应系统布局，只处理4.4以上
     * @param activity
     */
    public static void setFitsSystemWindows(Activity activity) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            ((ViewGroup) activity.getWindow().getDecorView().findViewById(android.R.id.content)).getChildAt(0)
                    .setFitsSystemWindows(true);
        }
    }
```
##  状态栏着色
```
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
            mWindow.setFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS, WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
            SystemBarTintManager mTintManager = new SystemBarTintManager(activity);
            mTintManager.setStatusBarTintEnabled(true);
            mTintManager.setStatusBarTintColor(color);
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            //兼容5.0及以上支持全透明
            Window window = activity.getWindow();
            window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
            activity.getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                    | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
            window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
            window.setStatusBarColor(color);
        }
    }
```
# 实践
下面是项目中实际处理系统栏的通用做法，统一在Activity处理。
```
public class BaseActivity extends AppCompatActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        SystemUiUtil.initSystemBar(this, getSystemBarColor());
    }

    @Override
    protected void onStart() {
        super.onStart();
        if(isFitSystemWindows()){
            SystemUiUtil.setFitsSystemWindows(this);
        }
    }

    /** 子类可继承实现染色 **/
    public int getSystemBarColor(){
        return ContextCompat.getColor(this,R.color.colorPrimary);
    }

    /** 子类可继承决定是否适应 **/
    public boolean isFitSystemWindows(){
        return true;
    }
}
```
其中initSystemBar方法和setFitsSystemWindows方法封装于SystemUiUtil类中，具体实现见仁见智。
具体代码可参考[AppArchitecture](https://github.com/YummyLau/AppArchitecture)项目。