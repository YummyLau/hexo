---
title: Android开发踩坑录(持续更新...)
layout: post
date: 2019-06-13
comments: true
categories: Android
tags: [Android经验]
---

<!--more-->

### 问题目录

* View
* Service
* 系统适配
* 线程
* 网络
* 数据/数据库
* 编译/构建/版本
* Git
* 第三方库问题
* IDE
* 编码


## View

#### Activity

* **非主线也能更新ui的问题,如果在onCreate直接跑异步线程可以更新ui**
	谷歌在 viewRootImpl 中检查更新ui的线程
	
	```
	void checkThread() {  
        if (mThread != Thread.currentThread()) {  
            throw new CalledFromWrongThreadException(  
                    "Only the original thread that created a view hierarchy can touch its views.");  
        }  
    } 
	```
	在执行onCreate的时候这个判断并没有执行到

#### Fragment

* **dialogFragment 设置全屏时左右留空的问题**

	在 fragment#onResume 中重新调整 window 布局
	
	```
	android.view.WindowManager.LayoutParams lp = window.getAttributes();
	lp.width = WindowManager.LayoutParams.MATCH_PARENT;
	lp.height = WindowManager.LayoutParams.WRAP_CONTENT;
	window.setAttributes(lp);
	```

* **dialogFragment 设置全屏时状态栏出现黑色布局**
	
	在主题中设置
	
	```
	<item name="android:windowIsFloating">true</item>
	```
	此时 window 为 wrap_content，如果出现左右空白，则考虑使用上个问题的方案。



#### RecyclerView

*  **`Recyclerview`调用notifyItemRemoved方法移除某个Item之后可能因为引用position引起crash**

	`notifyItemRemoved`方法并不会移除列表的数据源的数据项导致数据源中的数据与列表Item数目不一致，需要同步刷新数据源。

* **`Recyclerview`局部刷新Item时会因为默认动画导致闪烁**
  
	因为`recyclerview`存在ItemAnimator，且在删除/更新/插入Item时会触发，可设置不支持该动画即可。
```
((SimpleItemAnimator)recyclerView.getItemAnimator()).setSupportsChangeAnimations(false);
```

* **在大神动态列表中曾出现 Recyclerview 中的 item 出现莫名的偏移滚动**

	这个问题经过定位存在于 viewholder 中的某个 view 可能提前获取到焦点。 同时在。alibaba-vlayout 库中也发现有人反馈改问题。 https://github.com/alibaba/vlayout/issues/225  解决的方法是在 Recyclerview中外层父布局中添加 `android:descendantFocusability="blocksDescendants"` 用于父布局覆盖 Recyclerview 优先抢占焦点。
	
* **item 内容超过一屏时，findFistCompletelyVisibleItemPosition会返回 -1 的问题**
	原因是在 `findOneVisibleChild` 计算出来的 start 和 end 已经超过了 reclclerview 的 start 和 end.经过研究源码得到以下。
	
	```
	findFirstCompletelyVisibleItemPosition -> -1
	findLastCompletelyVisibleItemPosition -> -1
	findFirstVisibleItemPosition -> 正常
	findLastVisibleItemPosition -> 正常
	```
#### TextView 

* **textview 中富文本点击事件拦截了长按事件**

	这个问题常见于消息列表中，某条消息使用了ClickableSpan用于处理富媒体的点击事件，同时这个消息又需要支持长按复制。由于LinkMovementMethod方法在onTouchEvent一直返回true，可以通过自定义View.onTouchListener来替换setMovenmentMethod达到效果。
	
	```
	public class ClickMovementMethod implements View.OnTouchListener {
	private LongClickCallback longClickCallback;
	
	public static ClickMovementMethod newInstance() {
	    return new ClickMovementMethod();
	}
	
	@Override
	public boolean onTouch(final View v, MotionEvent event) {
	    if (longClickCallback == null) {
	        longClickCallback = new LongClickCallback(v);
	    }
	
	    TextView widget = (TextView) v;
	    // MovementMethod设为空，防止消费长按事件
	    widget.setMovementMethod(null);
	    CharSequence text = widget.getText();
	    Spannable spannable = Spannable.Factory.getInstance().newSpannable(text);
	    int action = event.getAction();
	    if (action == MotionEvent.ACTION_DOWN || action == MotionEvent.ACTION_UP) {
	        int x = (int) event.getX();
	        int y = (int) event.getY();
	        x -= widget.getTotalPaddingLeft();
	        y -= widget.getTotalPaddingTop();
	        x += widget.getScrollX();
	        y += widget.getScrollY();
	        Layout layout = widget.getLayout();
	        int line = layout.getLineForVertical(y);
	        int off = layout.getOffsetForHorizontal(line, x);
	        ClickableSpan[] link = spannable.getSpans(off, off, ClickableSpan.class);
	        if (link.length != 0) {
	            if (action == MotionEvent.ACTION_DOWN) {
	                v.postDelayed(longClickCallback, ViewConfiguration.getLongPressTimeout());
	            } else {
	                v.removeCallbacks(longClickCallback);
	                link[0].onClick(widget);
	            }
	            return true;
	        }
	    } else if (action == MotionEvent.ACTION_CANCEL) {
	        v.removeCallbacks(longClickCallback);
	    }
	
	    return false;
	}
	
	private static class LongClickCallback implements Runnable {
	    private View view;
	
	    LongClickCallback(View view) {
	        this.view = view;
	    }
	
	    @Override
	    public void run() {
	        // 找到能够消费长按事件的View
	        View v = view;
	        boolean consumed = v.performLongClick();
	        while (!consumed) {
	            v = (View) v.getParent();
	            if (v == null) {
	                break;
	            }
	            consumed = v.performLongClick();
	        }
	    }
	}
	}
	
	textView.setOnTouchListener(ClickMovementMethod.newInstance());
	```

	
#### ViewPager
* **如何禁止ViewPager 滑动**

	重写ViewPager onTouchEvent 和 onInterceptTouchEvent 并返回false,不处理任何滑动事件

	```
	@Override
	public boolean onTouchEvent(MotionEvent arg0) {
	    return false;
	}
	
	@Override
	public boolean onInterceptTouchEvent(MotionEvent arg0) {
	    return false;
	}
	```
	
* **如何仿蘑菇街/马蜂窝Viewpager装载图片之后切换时动态变更高度**

	imageViewPager 为普通的 Viewpager 对象
	
	imageListInfo为存放图片信息的list，imageShowHeight为业务需要显示高度，通过切换时动态计算调整
	
	```
	        imageViewPager.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
	            @Override
	            public void onGlobalLayout() {
	                imageViewPager.getViewTreeObserver().removeOnGlobalLayoutListener(this);
	                //根据viewpager的高度，拉伸显示图片的宽度调整高度。
	                ViewGroup.LayoutParams layoutParams = imageViewPager.getLayoutParams();
	                layoutParams.height = imageListInfo.imageShowHeight[0];
	                imageViewPager.setLayoutParams(layoutParams);
	            }
	        });
	        imageViewPager.setAdapter(imagePagerAdapter);
	        imageViewPager.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
	            @Override
	            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
	                if (position == imageListInfo.getImageListSize() - 1) {
	                    return;
	                }
	                int height = (int) (imageListInfo.imageShowHeight[position] * (1 - positionOffset) + imageListInfo.imageShowHeight[position + 1] * positionOffset);
	                ViewGroup.LayoutParams params = imageViewPager.getLayoutParams();
	                params.height = height;
	                imageViewPager.setLayoutParams(params);
	            }
	
	            @Override
	            public void onPageSelected(int position) {
	                if (!clickListBySelf) {
	                    toSelectIndex(imageListInfo.selected, position);
	                }
	            }
	
	            @Override
	            public void onPageScrollStateChanged(int state) {
	
	            }
	        });
	```


#### AppBarLayout

* **如何控制 appbarLayout 随时定位到某个位置**

	```
	CoordinatorLayout.Behavior behavior =((CoordinatorLayout.LayoutParams)mAppBarLayout.getLayoutParams()).getBehavior();
	if (behavior instanceof AppBarLayout.Behavior) {
	      AppBarLayout.Behavior appBarLayoutBehavior = (AppBarLayout.Behavior) behavior;
	      int topAndBottomOffset = appBarLayoutBehavior.getTopAndBottomOffset();
	      if (topAndBottomOffset != 0) {
	             appBarLayoutBehavior.setTopAndBottomOffset(0);
	      }
	```

* **如何禁止 appbarLayout 滚动**

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


#### Edittext

* **edittext未能响应onClickListener事件**
	
	`Edittext`监听未获取焦点的Edittext的点击事件，第一次点击触发OnFocusChangeListener，在获取焦点的情况下才能响应onClickListener
	
#### 其他view

* **使用listview或gridview的处理item的state_selected事件是无效的**

	在xml布局中对listview或gridview设置**Android:choiceMode="singleChoice"**,并使用state_activated状态来代替state_selected状态。（2016.12.10）

* **解决5.0以上Button自带阴影效果**

	在xml定义的Button中，添加以下样式定义
	```
	style="?android:attr/borderlessButtonStyle"
	```

* **针对 onSingleTapUp 和 onSIngleTapConfirmed 的使用区别**

	前者在按下并抬起时发生，后者有一个附加条件时Android会确保点击之后在短时间内没有再次点击才会触发。常用于如果需要监听单击和双击事件。

* **如何使用layer-list画三角形**

	```
	<layer-list xmlns:android="http://schemas.android.com/apk/res/android" >
	//左
    <item>
        <rotate
            android:fromDegrees="45"
            android:pivotX="85%"
            android:pivotY="135%">
            <shape android:shape="rectangle">
                <size
                    android:width="16dp"
                    android:height="16dp" />
                <solid android:color="#7d72ff" />
            </shape>
        </rotate>

    </item>
    
    //右
    <item>
        <rotate
            android:fromDegrees="45"
            android:pivotX="15%"
            android:pivotY="-35%">
            <shape android:shape="rectangle">
                <size
                    android:width="16dp"
                    android:height="16dp" />
                <solid android:color="#7d72ff" />
            </shape>
        </rotate>

    </item>
    
    //上/正
    <item>
        <rotate
            android:fromDegrees="45"
            android:pivotX="-40%"
            android:pivotY="80%">
            <shape android:shape="rectangle">
                <size
                    android:width="16dp"
                    android:height="16dp"/>
                <solid android:color="#7d72ff"/>
            </shape>
        </rotate>
    </item>
    
    //下
    <item>
        <rotate
            android:fromDegrees="45"
            android:pivotX="135%"
            android:pivotY="15%">
            <shape android:shape="rectangle">
                <size
                    android:width="16dp"
                    android:height="16dp"/>
                <solid android:color="#7d72ff"/>
            </shape>
        </rotate>
    </item>
</layer-list>
	
	```
	

## Service

* **后台手动清理应用之后，service中启动的notifications并没有消失**

	从 [How to remove all notifications when an android app (activity or service) is killed?](https://stackoverflow.com/questions/22210241/how-to-remove-all-notifications-when-an-android-app-activity-or-service-is-kil) 的诸多讨论中学习到， `Service#onTaskRemoved` 是我们的App被清理之后Service的回调。尝试过一下方法并不能达到清除的效果。
	
	```
    @Override
    public void onTaskRemoved(Intent rootIntent) {
        super.onTaskRemoved(rootIntent);
        NotificationManager nManager = ((NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE));
        nManager.cancelAll();
    }
	```
	在线上应用中，由于我们的通知类似于将军令这种有定时更新的功能，需要彻底干掉所有serivce承载的功能，下面方法可行
	
	```
	 @Override
    public void onTaskRemoved(Intent rootIntent) {
        super.onTaskRemoved(rootIntent);
        stopSelf();
        stopForeground(true);
    }
	```
	
	
# 系统适配
* **如何解决 NatigationBar 被 PopupWindow 遮挡的问题**

	```
	popupWindow.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);
	```
	
* **如何解决MIUI系统后台无法 Toast 的问题**

	参考 https://github.com/zhitaocai/ToastCompat_Deprecated 项目，但是在小米3或者小米Note（4.4.4）手机上
	```
	mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
	```
	mContext 需要使用 ApplicationContext 才能生效。
	
* **如何解决关闭通知栏权限无法弹出toast的问题**

	由于谷歌把 Toast 设置为系统消息权限，可以参考 [Android关闭通知消息权限无法弹出Toast的解决方案](http://w4lle.com/2016/03/27/Android%E5%85%B3%E9%97%AD%E9%80%9A%E7%9F%A5%E6%B6%88%E6%81%AF%E6%9D%83%E9%99%90%E6%97%A0%E6%B3%95%E5%BC%B9%E5%87%BAToast%E7%9A%84%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/) 维护自己的消息队列.
	
* **如何适配 vivo 等双面屏幕**
	在 AndroidManifest.xml 中声明一下 meta-data
	
	```
	<meta-data
	android:name="andriod.max_aspect" android:value="ratio_float"/>
	```
	或者使用 `android:maxAspectRatio="ratio_float"(API LEVEL 26)`
	ratio_float 一般为屏幕分辨率高宽比。
	其他比如凹槽区域，圆角切割等问题可以参考市面上最新的vivo机型对应的 [vivo开放平台](https://www.jianshu.com/p/4ed3b38d888d) 文档。
	
* **华为设备产生太多 broadcast 导致crash的问题**

	由于部分华为中，如果app注册超过500个BroadcastReceiver就会抛出 “ *Register too many Broadcast Receivers* ” 异常。通过分析发现其内部有一个白名单，自己可以通过创建一个新的app，使用微信包名进行测试，发现并没有这个限制。通过反射 LoadedApk 类拿到 mReceiverResource 中的 mWhiteList 对象添加我们的包名就可以了。 可以参考 https://github.com/llew2011/HuaWeiVerifier 这个项目。
	
* **各种通知栏的适配问题**
	
	参考 [网易考拉实现的适配方法]( https://iluhcm.com/2017/03/12/experience-of-adapting-to-android-notifications/)
	
* **针对魅族推送内容限制的问题**

	今天收到魅族渠道的警报称“推送内容可能过长”。IM功能针对离线设备走设备商的推送，魅族推送限制了title标题1-32字符，content内容1-100字符。如果频繁推送超过限制的通知，魅族推送服务器可能不会下发推送到魅族设备。故服务端限制发送到魅族服务器的消息标题和内容长度解决。
	
* **从系统安装起安装应用后启动，Home隐藏后Launcher重复启动的问题**

	判断启动页面是否是根节点(推荐)
		
	```
	if(!isTaskRoot()){
		finish();
		return 
	}
	```
	或者判断Activity是否多了 **FLAG_ACTIVITY_BROUGHT_TO_FRONT** ，这个tag是该场景导致的
		
	```
	if ((getIntent().getFlags() & Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT) != 0) {
	        finish();
	        return;
	}
	```

	
* **针对有launcher做为Activity的应用，在完全没有启动下收到第三方推送（小米，华为，魅族）/分享拉起的注意事项**

	由于我们的应用LauncherActivity用于分发不同场景的入口，A逻辑进入特殊场景页面A，B逻辑进入主页面B。

	* onCreate中优先拦截 intent 判断拉起参数，如果有拉起参数则直接进入主页面B，intent交付给主页面B处理
	* 部分场景下，比如第三方消息推送，华为和小米拉起闪屏或 launcher intent无法区分，针对该做法是，限定进入闪屏/launcher的逻辑,剩余场景统一进入主页面B
	
		```
   if (后端控制是否需要进入特殊场景页面) {
            boolean goToA = false;
            if (getIntent() != null) {
                String action = getIntent().getAction();
                Set<String> category = getIntent().getCategories();
                if (TextUtils.equals(action, "android.intent.action.MAIN")) {
                    if (category != null && category.contains("android.intent.category.LAUNCHER")) {
                        Intent intent = new Intent(this, 页面A.class);
                        intent.setData(getIntent().getData());
                        startActivity(intent);
                        goToA = true;
                    }
                }
            }
            if (! goToA) {
                goToMainActivity();
            }
        } else {
            goToMainActivity();
        }
		```

* **针对 App 多场景拉起场景下的场景判断分析.**
	
	可参考我另一篇文章 [对线上项目拉起应用场景的思考总结](http://yummylau.com/2019/06/26/Adnroid_2019-06-06_%E5%AF%B9%E7%BA%BF%E4%B8%8A%E9%A1%B9%E7%9B%AE%E6%8B%89%E8%B5%B7%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF%E7%9A%84%E6%80%9D%E8%80%83%E6%80%BB%E7%BB%93/)
	
* **9.0 android 支持明文连接（Http）**

   Android 9（API级别28）开始，默认情况下禁用明文支持
   
	```
	<?xml version="1.0" encoding="utf-8"?>
	<manifest ...>
	    <uses-permission android:name="android.permission.INTERNET" />
	    <application
	        ...
	        android:usesCleartextTraffic="true"
	        ...>
	        ...
	    </application>
	</manifest>
	```

# 线程

* **新起线程不要随便调用网络请求，一般的newThread没有looper队列，参考handlerThread**

# 网络

* **Retorfit get 请求参数出现错误**

	```
	@GET( BASE_URL + "index/login" )
	Observable< LoginResult > requestLogin( @QueryMap(encoded = true) Map< String, String > params );
	 
	final Map< String, String > paramsMap = new ParamsProvider.Builder().
	        append( "username", account ).
	        append( "password",URLEncoded(password) ).
	
	```
	比如登陆，encode = true 表示未对特殊字符进行url编码，默认是false。

# 数据/数据库
* **如何处理 sqlite 多线程读写问题**
	* 一个helper实例对应一个 db connectton，这个连接能提供读连接和写连接，如果只是调用 read-only，则默认也会有写连接。
	* 一个helper实例可以在多个线程中使用，java层会使用锁机制保证线程同步，哪怕有100个线程，对数据度的调用也会被序列化
	* 如果尝试从不同 connection 同时对数据库进行写操作，则有一个会失败。并不会按照第一个写完再轮到第二个写，有一些sqlite版本甚至不会有错误提示。
	
	一般而言，如果要在多线程环境下使用数据库，则确保多个线程中使用的是同一个SQLiteDataBase对象，该对象对应一个db文件。
	
	特殊情况，如果多个 SQLiteDataBase 打开同一个 db 文件，同时使用不同线程同时写（insert，update，exexSQL）会导致在 `SQLiteStatement.native_execute` 方法时可能导致异常。这个异常来自本地方法里面，仅仅在Java对有对 SQLiteDataBase 进行同步锁保护。但是多线程读（query）返回的事 SQLiteCursor保存查询条件并没有立刻执行查询，仅仅在需要时加载部分数据，可以多线程不同 SQLiteDataBase 进行读。
	
	如果要处理上述问题，可以使用 “*一个线程写，多个线程同时读，每个线程都用各自SQLiteOpenHelper*。”
	
	在android 3.0版本以上 打开 enableWriteAheadLogging。当打开时，它允许一个写线程与多个读线程同时在一个SQLiteDatabase上起作用。实现原理是写操作其实是在一个单独的文件，不是原数据库文件。所以写在执行时，不会影响读操作，读操作读的是原数据文件，是写操作开始之前的内容。在写操作执行成功后，会把修改合并会原数据库文件。此时读操作才能读到修改后的内容。但是这样将花费更多的内存。
	
* **如何理解 Intent 传递数据出现 `TransactionTooLargeException`**
  
  Intent 传输数据的机制中，用到了 Binder。Intent 中的数据，会作为 Parcel 被存储在 Binder 的事务缓冲区(Binder transaction buffer)中的对象进行传输.而这个 Binder 事务缓冲区具有一个有限的固定大小，当前为 1MB。你可别以为传递 1MB 以下的数据就安全了，这里的 1MB 空间并不是当前操作独享的，而是由当前进程所共享。也就是说 Intent 在 Activity 间传输数据，本身也不适合传递太大的数据.
  
  参考阿里 《Android 开发者手册》 对于Activity间数据通讯数据较大，避免使用Intent+Parcelable的方式，可以考虑使用EventBus等代替方案，避免 `TransactionTooLargeException`。EventBus使用黏性事件来解决，但是针对Activity重建又能拿到Intent而EventBus则不可以，所以需要根据业务来调整。
	

# 编译/构建/版本

* **travis-ci 高版本androidO编译遇到 license 没通过编译失败。**

	参考 [CI 讨论区](https://travis-ci.community/t/unable-to-accept-license-for-build-28-0-3-and-android-sdk-27/2883) 添加 `dist: precise` 及 `before_install 项中新增 sdkmanager指令`。具体可参考我的 [开源项目配置](https://github.com/YummyLau/PanelSwitchHelper/blob/master/.travis.yml)
	

* **Dalvik支持的android版本下进行分包执行会有一些限制**
	* 冷启动时需要安装dex文件，如果dex文件太大则可能导致处理时间太长导致 ANR
	* 即使使用 multiDex 方案在低于 4.0 系统上可能会出现 Dalvik linearAlloc 的bug，这是因为该方案需要申请一个很大的内存，运行时可能的导致程序崩溃。这个限制在 4.0 上虽然有所改善了，但是还是可能在低于 5.0 的机器上触发。

* **Dalvik 分包构建每一个 dex 文件时可能出现 java.lang.NoClassDefFoundError**

	这个问题的原因是构建工具绘制行比较复杂决策来确定主 dex 文件中需要的类以便应用能够正常的启动。如果启动期间需要的任何类在主 dex 中未能找到，则会抛出上述异常。所有必须要 multiDexKeepFile 或 multiDexKeepProguard 属性中声明他们，手动将这些类指定为主 dex 文件中的必需项。
	
	创建 multidex-new.txt文件，写入以下新增的类
	
	```
	com/example/Main2.class
	com/example/Main3.class
	```
	创建 meltidex-new.pro,写入以下 keep 住的类
		
	```
	-keep class com.example.Main2
	-keep class com.example.Main3
	```
	然后在gradle multiDexKeepFile属性 和 multiDexKeepProguard属性声明上述文件
	
	```
	android {
    buildTypes {
        release {
            multiDexKeepFile file 'multidex-new.txt'
            multiDexKeepProguard 'multidex-new.pro'
            ...
        }
    }
	}
	```
* **Java 8 methods of java.lang.Long and java.lang.Character are not desugared by D8**

	这个问题出现在使用 Kotlin 编译时，从 Kotlin1.3.30 版本开始 ndroid.compileOptions中的Java版本推断出JVM目标，如果同时设置了sourceCompatibility和targetCompatibility，则选择“1.8”到那个或更高. 可以通过指定 JavaVersion 1.6 来解决这个问题。
	
	```
	sourceCompatibility JavaVersion.VERSION_1_6
    targetCompatibility JavaVersion.VERSION_1_6
	```
	 [Issue](https://issuetracker.google.com/issues/129730297) 中表示，AGP（Android Gradle Plugin）3.4 已解决脱糖问题，可尝试升级解决。

* **databinding 中 findBinding vs getBinding 的场景区别**

	不同之处在于，findBinding将遍历父节点，而如果使用getBinding时当view不是跟节点会返回null。
	
* **版本构建出现 `Gradle sync failed: Cannot choose between the following configurations of project`。**

	参考 [issues](https://github.com/dialogflow/dialogflow-android-client/issues/57) 的回答
	
	If you're using Android plugin for Gradle 3.0.0 or higher, the plugin automatically matches each variant of your app with corresponding variants of its local library module dependencies for you. That is, you should no longer target specific variants of local module dependencies, show as below
	
	```
	 dependencies {
  // Adds the 'debug' varaint of the library to the debug varaint of the app
  debugCompile project(path: ':my-library-module', configuration: 'debug')

  // Adds the 'release' varaint of the library to the release varaint of the app
  releaseCompile project(path: ':my-library-module', configuration: 'release')
	}	
	
	```
	
* **gradle 配置本地离线 zip**

	1. 离线下载 gradle 离线包保存在 url 中
	2. 修改 *gradle/wrapper/gradle-wrapper.properties* 调整 *distributionUrl* 目录指向 url
	3. 修改 *build.gradle*  classpath 版本映射 gradle 版本
	

## Git
	
* **kvm/jvm 编译时 -classpath 遇到的分割及空格问题。**

	linux/mac OS 上使用 “：” 分割多个classpath路径，window使用 “；” 分割。
	
	如果linux/mac OS 路径存在空格，暂时避免，使用多种方式尝试未果=。=。	
* **git 修改 commit 记录**

	```
	git reset --soft HEAD^
	```
	
	撤销当前的commit，如果只是修改提示，则使用
	
	```
	git commit --amend
	```

* **git ignore 文件不生效**

	```
	git rm -r --cached .
	git add .
	git commit -m 'update .gitignore'
	```
	
## 第三方库问题

* **ExoPlayer在接听电话之后会导致原来设置的 Source 中静音状态消失了导致可能返回 app 续播的时候视频突然有声音**

	原因是多媒体焦点被通话抢夺之后播放音量被充值，解决方法可参考 https://github.com/google/ExoPlayer/issues/5092 
	
## IDE
* **AndroidStudio 提示 Please select Android SDK**

	解决手段： File->Project Structure中修改Build tools version


## 编码

* **关于Java中字符与字节的编码关系 （2017.02.12）**

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

