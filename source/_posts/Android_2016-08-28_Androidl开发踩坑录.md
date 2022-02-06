---
title: Android开发踩坑经验贴(持续更新...)
layout: post
date: 2020-03-30
comments: true
categories: Android
tags: [Android经验]
---

<!--more-->

### 总结目录

* [视图篇](#1)
	* [如何理解非主线程可以更新UI](#1-1)
	* [dialogFragment 全屏时左右留空的解决方案](#1-2)
	* [dialogFragment 全屏时状态栏出现黑色布局的解决方案](#1-3)
	* [多个fragment 切换重叠的解决方案](#1-21)
	* [多个fragment 保存状态时可能出现 TransactionTooLargeException 的解决方案](#1-22)
	* [recyclerview 调用 notifyItemRemoved 方法移除某个 Item 之后因为引用 position 引起 crash 的原因](#1-4)
	* [recyclerview 局部刷新Item时会因为默认动画导致闪烁的解决方案](#1-5)
	* [recyclerview 中的 item 出现莫名的偏移滚动](#1-6)
	* [recyclerview 内容超过一屏时，findFistCompletelyVisibleItemPosition 会返回 -1 的原因](#1-7)
	* [textview 中富文本点击事件拦截了长按事件的解决方案](#1-8)
	* [如何禁止 ViewPager 的滑动](#1-9)
	* [如何仿蘑菇街/马蜂窝 Viewpager 装载图片之后切换时动态变更高度](#1-10)
	* [如何控制 appbarLayout 随时定位到某个位置](#1-11)
	* [如何禁止 appbarLayout 滚动](#1-12)
	* [edittext 未能响应 onClickListener 事件的解决方案](#1-13)
	* [使用 listview 或gridview 的处理 item 的 state_selected 事件是无效的解决方案](#1-14)
	* [解决5.0以上Button自带阴影效果的方案](#1-15)
	* [针对 onSingleTapUp 和 onSIngleTapConfirmed 的使用区别](#1-16)
	* [如何使用layer-list画三角形](#1-17)
	* [关于属性动画中旋转 View 时部分机型出现 View 闪烁的解决方案](#1-18)
	* [关于 ConstraintLayout 的代码布局下的注意事项](#1-19)
	* [TextView 在 6.0 版本下设置单行尾部缩略的坑](#1-20)
* [服务篇](#2)
	* [后台手动清理应用之后，service中启动的notifications并没有消失的解决方案](#2-1)
	* [全局的Context使用更为优雅的获取方案](#2-2)
* [线程篇](#3)
 	* [建议新起线程不要随便调用网络请求，一般的newThread没有looper队列，参考handlerThread](#3-1)
* [网络篇](#4)
	* [Retorfit get 请求参数出现错误的解决方案](#4-1)
* [数据篇](#5)
 	* [如何优雅处理 sqlite 多线程读写问题](#5-1)
 	* [如何理解 Intent 传递数据出现 TransactionTooLargeException 的问题](#5-2)
* [机型系统适配篇](#6)
	* [如何解决 NatigationBar 被 PopupWindow 遮挡的问题](#6-1)
	* [如何解决MIUI系统后台无法 toast 的问题](#6-2)
	* [如何解决关闭通知栏权限无法弹出 toast 的问题](#6-3)
	* [如何适配 vivo 等双面屏幕](#6-4)
	* [如何解决华为设备产生太多 broadcast 导致crash的问题](#6-5)      
  	* [各种通知栏的适配方案](#6-6)
	* [解决针对魅族推送内容限制的问题](#6-7)
	* [解决从系统安装起安装应用后启动，Home 隐藏后 Launcher 重复启动的问题](#6-8)
 	* [针对有launcher做为Activity的应用，在完全没有启动下收到第三方推送（小米，华为，魅族）/分享拉起的注意事项](#6-9)
 	* [针对 App 多场景拉起场景下的场景判断分析](#6-10)
 	* [8.0 部分 ROM 出现 Only fullscreen opaque activities can request orientation 的解决方案](#6-12)
	* [9.0 android 支持明文连接（Http)](#6-11)
* [编译构建篇](#7)
 	* [travis-ci 高版本androidO编译遇到 license 没通过编译失败的解决方案](#7-1)
 	* [Dalvik 支持的 android 版本下进行分包执行会有一些限制](#7-2)
 	* [Dalvik 分包构建每一个 dex 文件时可能出现 java.lang.NoClassDefFoundError](#7-3)
 	* [Java 8 methods of java.lang.Long and java.lang.Character are not desugared by D8](#7-4)
 	* [databinding 中 findBinding vs getBinding 的场景区别](#7-5)
 	* [版本构建出现 Gradle sync failed: Cannot choose between the following configurations of project](#7-6)
 	* [gradle 配置本地离线包](#7-7)
 	* [解决kvm/jvm 编译时 -classpath 遇到的分割及空格的问题](#7-8)
 	* [databinding NoSuchMethodError with buildTool 3.4.0](#7-9)
 	* [AS连接真机调试出现 debug info can be unavailabe 的解决方法](#7-10)
* [版本控制篇](#8)
 	* [git 修改 commit 记录](#8-1)
 	* [解决git ignore 文件不生效的问题](#8-2)
* [其他](#9)
	* [ExoPlayer在接听电话之后会导致原来设置的 Source 中静音状态消失了导致可能返回 app 续播的时候视频突然有声音](#9-1)
	* [AndroidStudio 提示 Please select Android SDK](#9-2)
	* [关于 Java 中字符与字节的编码关系认识](#9-3)
	* [关于 emoji 编码的长度计算问题](#9-4)


<h3 id="1"><<视图篇>></h3>

* <h4 id="1-1"> 如何理解非主线程可以更新UI</h4>
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

* <h4 id="1-2"> dialogFragment 全屏时左右留空的解决方案</h4>

	在 fragment#onResume 中重新调整 window 布局
	
	```
	android.view.WindowManager.LayoutParams lp = window.getAttributes();
	lp.width = WindowManager.LayoutParams.MATCH_PARENT;
	lp.height = WindowManager.LayoutParams.WRAP_CONTENT;
	window.setAttributes(lp);
	```
	
* <h4 id="1-3"> dialogFragment 全屏时状态栏出现黑色布局的解决方案</h4>
	在主题中设置
	
	```
	<item name="android:windowIsFloating">true</item>
	```
	此时 window 为 wrap_content，如果出现左右空白，则考虑使用上个问题的方案。
	
* <h4 id="1-21"> 当应用退回后台一段时间重返后，Fragment 切换重叠的解决方案 </h4>

	在线上项目中我们遇到一个场景：当应用按下 Home 退回后台，然后过一段时间之后从后台拉起我们的项目。极少数机型在主页进行多个 *fragment* 的切换时出现了 *fragment* 的重叠。经过定位之后发现，这些机型的运存偏小，性能偏差，出现这种现象的原因是由于内存的压力的原因，系统并不知后台的程序哪一个才需要保持运行，就会尝试回收内存占用较大的页面，当我们的页面被系统销毁时，*fragmentActivity#onSaveInstanceState* 被执行并保存了一些瞬态信息，比如界面 *fragment* 的视图信息。当我们再次拉起应用的时候，会让原来的 *fragmentActivity* 重建并重新构建了一个新的 *fragment* ，此时会叠加到已经被恢复的 *fragment* 之上导致重叠。
	
	比较暴力的做法是不让 *activity* 保存状态，比如
	
	```
	 @Override
    public void onSaveInstanceState(Bundle outState) {
    	 //直接不调用 super.onSaveInstanceState(outState);
    	 //或者直接传递空数据 
        super.onSaveInstanceState(new Bundle());
    }
	
	```
	
	比较优雅的做法是，比如
	
	```
	 @Override
    public void onSaveInstanceState(Bundle outState) {
    	 getSupportFragmentManager().putFragment(outState, you_key, CusFragment);
	    super.onSaveInstanceState(outState);
    }
	
	//在onCreate的时候判断是否已经存在保存的信息
	 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if (savedInstanceState != null) {
            CusFragment fragment =  (CusFragment)getSupportFragmentManager().getFragment(savedInstanceState, you_key);
        } else {
        	  //init CusFragment
        }
    }
	```
	
* <h4 id="1-22"> 多个fragment 保存状态时可能出现 TransactionTooLargeException 的解决方案 </h4>

	出现 `TransactionTooLargeException` 异常时，因为线上我们使用了 *FragmentStatePagerAdapter* 作为 *fragment* 适配器为了尽可能过缓存下浏览过的 *fragment* 以获得更好的体验，承载多个 *FragmentStatePagerAdapter#saveState* 会被调用并对每一个 *fragment* 的 *bundle* 数据进行保存。由于我们的 *bundle* 较大，并且保存下来的 *bundle* 并不会因为 *fragment* 被销毁而销毁，所以需要保存的 *bundle* 数据会一直增长，直到出现` TransactionTooLargeException` 异常. 我们参考[stackoverflow相关问题](https://stackoverflow.com/questions/11451393/what-to-do-on-transactiontoolargeexception) 直接重载 *saveState* 丢弃 *states* 内容。
	
	```
    public Parcelable saveState() {
	    Bundle bundle = (Bundle) super.saveState();
	    bundle.putParcelableArray("states", null); // Never maintain any states from the base class, just null it out
	    return bundle;
    }
	```
	
	另外推荐 [toolargetool](https://github.com/guardian/toolargetool) 工具可以在开发中实时观测页面内存变化。
	
	

* <h4 id="1-4"> recyclerview 调用 notifyItemRemoved 方法移除某个 Item 之后因为引用 position 引起 crash 的原因</h4>

	`notifyItemRemoved`方法并不会移除列表的数据源的数据项导致数据源中的数据与列表Item数目不一致，需要同步刷新数据源。

* <h4 id="1-5"> recyclerview 局部刷新Item时会因为默认动画导致闪烁的解决方案</h4>

	因为`recyclerview`存在ItemAnimator，且在删除/更新/插入Item时会触发，可设置不支持该动画即可。
	
	```
	((SimpleItemAnimator)recyclerView.getItemAnimator()).setSupportsChangeAnimations(false);
	```
	
* <h4 id="1-6"> recyclerview 中的 item 出现莫名的偏移滚动</h4>

	这个问题经过定位存在于 viewholder 中的某个 view 可能提前获取到焦点。 同时在。alibaba-vlayout 库中也发现有人反馈改问题。 [issues-255](https://github.com/alibaba/vlayout/issues/225)  解决的方法是在 Recyclerview中外层父布局中添加 `android:descendantFocusability="blocksDescendants"` 用于父布局覆盖 Recyclerview 优先抢占焦点。

* <h4 id="1-7"> recyclerview 内容超过一屏时，findFistCompletelyVisibleItemPosition 会返回 -1 的原因</h4>

	原因是在 `findOneVisibleChild` 计算出来的 start 和 end 已经超过了 reclclerview 的 start 和 end.经过研究源码得到以下。
	
	```
	findFirstCompletelyVisibleItemPosition -> -1
	findLastCompletelyVisibleItemPosition -> -1
	findFirstVisibleItemPosition -> 正常
	findLast
	```
	
* <h4 id="1-8"> textview 中富文本点击事件拦截了长按事件的解决方案</h4>	
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
	
* <h4 id="1-9"> 如何禁止 ViewPager 的滑动</h4>

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
	
* <h4 id="1-10"> 如何仿蘑菇街/马蜂窝 Viewpager 装载图片之后切换时动态变更高度</h4>

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
* <h4 id="1-11"> 如何控制 appbarLayout 随时定位到某个位置</h4>

	```
	CoordinatorLayout.Behavior behavior =((CoordinatorLayout.LayoutParams)mAppBarLayout.getLayoutParams()).getBehavior();
	if (behavior instanceof AppBarLayout.Behavior) {
	      AppBarLayout.Behavior appBarLayoutBehavior = (AppBarLayout.Behavior) behavior;
	      int topAndBottomOffset = appBarLayoutBehavior.getTopAndBottomOffset();
	      if (topAndBottomOffset != 0) {
	             appBarLayoutBehavior.setTopAndBottomOffset(0);
	      }
	```

* <h4 id="1-12"> 如何禁止 appbarLayout 滚动</h4>

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
	
* <h4 id="1-13"> edittext 未能响应 onClickListener 事件的解决方案</h4>
	
	`Edittext`监听未获取焦点的Edittext的点击事件，第一次点击触发OnFocusChangeListener，在获取焦点的情况下才能响应onClickListener
	
* <h4 id="1-14"> 使用 listview 或gridview 的处理 item 的 state_selected 事件是无效的解决方案</h4>

	在xml布局中对listview或gridview设置**Android:choiceMode="singleChoice"**,并使用state_activated状态来代替state_selected状态。（2016.12.10）

* <h4 id="1-15"> 解决5.0以上Button自带阴影效果的方案</h4>

	在xml定义的Button中，添加以下样式定义
	```
	style="?android:attr/borderlessButtonStyle"
	```
* <h4 id="1-16"> 针对 onSingleTapUp 和 onSIngleTapConfirmed 的使用区别</h4>

	前者在按下并抬起时发生，后者有一个附加条件时Android会确保点击之后在短时间内没有再次点击才会触发。常用于如果需要监听单击和双击事件。
	
* <h4 id="1-17"> 如何使用layer-list画三角形</h4>

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

* <h4 id="1-18"> 关于属性动画中旋转 View 时部分机型出现 View 闪烁的解决方案 </h4>

	在大神 app 信息流快捷评论模块中，在交付快捷评论动画的时候发现，使用属性动画实现的抖动效果在部分机型上出现闪烁。而我们的实现抖动效果是通过 `View.ROTATION` 来实现的。经过研究，部分机型因为硬件加速的原因导致的。为动画 view 进行以下设置
	
	```
	view.setLayerType(View.LAYER_TYPE_HARDWARE,null);
	```

* <h4 id="1-19"> 关于 ConstraintLayout 的代码布局下的注意事项 </h4>

	不同于其他 ViewGroup 控制子 View 的排版，ConstraintLayout 需要构建 `ConstraintSet` 对象来粘合。 在手动添加子 View 的场景下，可以通过 `ConstraintSet#clone(ConstraintLayout constraintLayout)` 来克隆当前已有 ConstraintLayout 的排版信息，然后最后调用 `ConstraintSet#applyTo(ConstraintLayout constraintLayout)` 确认最终的排版信息。

* <h4 id="1-20"> TextView 在 6.0 版本下设置单行尾部缩略的坑 </h4>

	在大神信息流中，有一些卡片信息需要设置单行缩略。在 MTL 兼容测试过程中发现有一些机型显示异常，经过归纳及校验，这部分机型的版本都是 < 6.0。 通过在 stackoverflow 也找到了相同的问题场景 [text ellipsize behavior in android version < 6.0](https://stackoverflow.com/questions/42524277/text-ellipsize-behavior-in-android-version-6-0) . 针对这部分版本的手机，我们需要在设置单行的时候把 `android:maxLines="1"` 改成 `android:singleLine="true"`。即使 IDE 提示该 API 已经过期了！
	

<h3 id="2"><<服务篇>></h3>

* <h4 id="2-1"> 后台手动清理应用之后，service中启动的notifications并没有消失的解决方案</h4>

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
	
* <h4 id="2-2"> 全局的Context使用更为优雅的获取方案 </h4>
	由于我们在优化 Application 启动时间时，打算移除 *applciation* 所有有关静态申明的变量，其中就包含全局 *context* 这个变量。我们参考的是 *leakCanary* 库的做法，使用 *ContentProvider* 来承载全局 *context* 的获取，原因是在 [ActivityThread](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/app/ActivityThread.java) 的初始化流程中，*ContentProvider#onCreate()* 是在 *Application#attachBaseContext(Context)*  和 *Application#onCreate()* 之间的。所以获取的 *context* 是有效的
	
	```
	class ContextProvider : ContentProvider() {
	
	    companion object {
	        private lateinit var mContext: Context
	        private lateinit var mApplication: Application
	        fun getGlobalContext(): Context = mContext
	        fun getGlobalApplication(): Application = mApplication
	    }
	    override fun onCreate(): Boolean {
	        mContext = context!!
	        mContext = context!!.applicationContext as Application
	        return false
	    }
	
	    override fun insert(uri: Uri, values: ContentValues?): Uri? = null
	    override fun query(uri: Uri, projection: Array<out String>?, selection: String?, selectionArgs: Array<out String>?, sortOrder: String?): Cursor? = null
	    override fun update(uri: Uri, values: ContentValues?, selection: String?, selectionArgs: Array<out String>?): Int = -1
	    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<out String>?): Int = -1
	    override fun getType(uri: Uri): String? = null
	}
	

	//manifest申明
    <!-- Context提供者 -->
    <provider
            android:name=".ContextProvider"
            android:authorities="${your_application_id}.contextprovider"
            android:exported="false" />
	```

<h3 id="3"><<线程篇>></h3>

* <h4 id="3-1"> 建议新起线程不要随便调用网络请求，一般的newThread没有looper队列，参考handlerThread</h4>

<h3 id="4"><<网络篇>></h3>

* <h4 id="4-1"> Retorfit get 请求参数出现错误的解决方案</h4>

	```
	@GET( BASE_URL + "index/login" )
	Observable< LoginResult > requestLogin( @QueryMap(encoded = true) Map< String, String > params );
	 
	final Map< String, String > paramsMap = new ParamsProvider.Builder().
	        append( "username", account ).
	        append( "password",URLEncoded(password) ).
	
	```
	比如登陆，encode = true 表示未对特殊字符进行url编码，默认是false。
	
<h3 id="5"><<数据篇>></h3>

* <h4 id="5-1"> 如何优雅处理 sqlite 多线程读写问题</h4>

	* 一个helper实例对应一个 db connectton，这个连接能提供读连接和写连接，如果只是调用 read-only，则默认也会有写连接。
	* 一个helper实例可以在多个线程中使用，java层会使用锁机制保证线程同步，哪怕有100个线程，对数据度的调用也会被序列化
	* 如果尝试从不同 connection 同时对数据库进行写操作，则有一个会失败。并不会按照第一个写完再轮到第二个写，有一些sqlite版本甚至不会有错误提示。
	
	一般而言，如果要在多线程环境下使用数据库，则确保多个线程中使用的是同一个SQLiteDataBase对象，该对象对应一个db文件。
	
	特殊情况，如果多个 SQLiteDataBase 打开同一个 db 文件，同时使用不同线程同时写（insert，update，exexSQL）会导致在 `SQLiteStatement.native_execute` 方法时可能导致异常。这个异常来自本地方法里面，仅仅在Java对有对 SQLiteDataBase 进行同步锁保护。但是多线程读（query）返回的事 SQLiteCursor保存查询条件并没有立刻执行查询，仅仅在需要时加载部分数据，可以多线程不同 SQLiteDataBase 进行读。
	
	如果要处理上述问题，可以使用 “*一个线程写，多个线程同时读，每个线程都用各自SQLiteOpenHelper*。”
	
	在android 3.0版本以上 打开 enableWriteAheadLogging。当打开时，它允许一个写线程与多个读线程同时在一个SQLiteDatabase上起作用。实现原理是写操作其实是在一个单独的文件，不是原数据库文件。所以写在执行时，不会影响读操作，读操作读的是原数据文件，是写操作开始之前的内容。在写操作执行成功后，会把修改合并会原数据库文件。此时读操作才能读到修改后的内容。但是这样将花费更多的内存。
	
* <h4 id="5-1"> 如何理解 Intent 传递数据出现 `TransactionTooLargeException`的问题</h4>

  Intent 传输数据的机制中，用到了 Binder。Intent 中的数据，会作为 Parcel 被存储在 Binder 的事务缓冲区(Binder transaction buffer)中的对象进行传输.而这个 Binder 事务缓冲区具有一个有限的固定大小，当前为 1MB。你可别以为传递 1MB 以下的数据就安全了，这里的 1MB 空间并不是当前操作独享的，而是由当前进程所共享。也就是说 Intent 在 Activity 间传输数据，本身也不适合传递太大的数据.
  
  参考阿里 《Android 开发者手册》 对于Activity间数据通讯数据较大，避免使用Intent+Parcelable的方式，可以考虑使用EventBus等代替方案，避免 `TransactionTooLargeException`。EventBus使用黏性事件来解决，但是针对Activity重建又能拿到Intent而EventBus则不可以，所以需要根据业务来调整。
	
<h3 id="6"><<机型系统适配篇>></h3>

* <h4 id="6-1"> 如何解决 NatigationBar 被 PopupWindow 遮挡的问题</h4>

	```
	popupWindow.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);
	```

* <h4 id="6-2"> 如何解决MIUI系统后台无法 toast 的问题</h4>

	参考 https://github.com/zhitaocai/ToastCompat_Deprecated 项目，但是在小米3或者小米Note（4.4.4）手机上
	```
	mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
	```
	mContext 需要使用 ApplicationContext 才能生效。

* <h4 id="6-3"> 如何解决关闭通知栏权限无法弹出 toast 的问题</h4>

	由于谷歌把 Toast 设置为系统消息权限，可以参考 [Android关闭通知消息权限无法弹出Toast的解决方案](http://w4lle.com/2016/03/27/Android%E5%85%B3%E9%97%AD%E9%80%9A%E7%9F%A5%E6%B6%88%E6%81%AF%E6%9D%83%E9%99%90%E6%97%A0%E6%B3%95%E5%BC%B9%E5%87%BAToast%E7%9A%84%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/) 维护自己的消息队列.
	
* <h4 id="6-4"> 如何适配 vivo 等双面屏幕</h4>

	在 AndroidManifest.xml 中声明一下 meta-data
	```
	<meta-data
	android:name="andriod.max_aspect" android:value="ratio_float"/>
	```
	或者使用 `android:maxAspectRatio="ratio_float"(API LEVEL 26)`
	ratio_float 一般为屏幕分辨率高宽比。
	其他比如凹槽区域，圆角切割等问题可以参考市面上最新的vivo机型对应的 [vivo开放平台](https://www.jianshu.com/p/4ed3b38d888d) 文档。
	
* <h4 id="6-5"> 如何解决华为设备产生太多 broadcast 导致crash的问题</h4>

	由于部分华为中，如果app注册超过500个BroadcastReceiver就会抛出 “ *Register too many Broadcast Receivers* ” 异常。通过分析发现其内部有一个白名单，自己可以通过创建一个新的app，使用微信包名进行测试，发现并没有这个限制。通过反射 LoadedApk 类拿到 mReceiverResource 中的 mWhiteList 对象添加我们的包名就可以了。 可以参考 https://github.com/llew2011/HuaWeiVerifier 这个项目。
	
* <h4 id="6-6"> 各种通知栏的适配方案</h4>
	
	参考 [网易考拉实现的适配方法]( https://iluhcm.com/2017/03/12/experience-of-adapting-to-android-notifications/)
	
* <h4 id="6-7"> 解决针对魅族推送内容限制的问题</h4>

	今天收到魅族渠道的警报称“推送内容可能过长”。IM功能针对离线设备走设备商的推送，魅族推送限制了title标题1-32字符，content内容1-100字符。如果频繁推送超过限制的通知，魅族推送服务器可能不会下发推送到魅族设备。故服务端限制发送到魅族服务器的消息标题和内容长度解决。
	
* <h4 id="6-8"> 解决从系统安装起安装应用后启动，Home 隐藏后 Launcher 重复启动的问题</h4>

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

* <h4 id="6-9"> 针对有launcher做为Activity的应用，在完全没有启动下收到第三方推送（小米，华为，魅族）/分享拉起的注意事项</h4>
	
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

* <h4 id="6-10"> 针对 App 多场景拉起场景下的场景判断分析</h4>
	
	可参考我另一篇文章 [对线上项目拉起应用场景的思考总结](http://yummylau.com/2019/06/26/%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93_2019-06-06_%E7%BA%BF%E4%B8%8A%E9%A1%B9%E7%9B%AE%E6%8B%89%E8%B5%B7%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF%E7%9A%84%E6%80%9D%E8%80%83%E6%80%BB%E7%BB%93/)
	
* <h4 id="6-12"> 8.0 部分 ROM 出现 Only fullscreen opaque activities can request orientation 的解决方案</h4>
	
	由于我们项目需要处理沉浸式，所以针对 *android:windowIsTranslucent* 的属性默认打开的。但是线上发现部分 8.0设备出现诡异的 crash，原因是我们对于页面的 *orientation* 申明都统一为 *portrait* 。查阅 android 源码的更新发现在 8.0 源码的逻辑里面这两个逻辑竟然不兼容，随后在 8.0 版本后谷歌进行了修复。但是国内部分 ROM 看起来并没有修复这个问题。后面同事提供了一个比较取巧的方案，通过为页面指定 *android:screenOrientation="behind"* 来避免 8.0 版本的问题同时兼容所有 android 版本。


* <h4 id="6-11"> 9.0 android 支持明文连接（Http）</h4>

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
	
<h3 id="7"><<编译构建篇>></h3>

* <h4 id="7-1"> travis-ci 高版本androidO编译遇到 license 没通过编译失败的解决方案</h4>

	参考 [CI 讨论区](https://travis-ci.community/t/unable-to-accept-license-for-build-28-0-3-and-android-sdk-27/2883) 添加 `dist: precise` 及 `before_install 项中新增 sdkmanager指令`。具体可参考我的 [开源项目配置](https://github.com/YummyLau/PanelSwitchHelper/blob/master/.travis.yml)
	

* <h4 id="7-2"> Dalvik 支持的 android 版本下进行分包执行会有一些限制</h4>

	* 冷启动时需要安装dex文件，如果dex文件太大则可能导致处理时间太长导致 ANR
	* 即使使用 multiDex 方案在低于 4.0 系统上可能会出现 Dalvik linearAlloc 的bug，这是因为该方案需要申请一个很大的内存，运行时可能的导致程序崩溃。这个限制在 4.0 上虽然有所改善了，但是还是可能在低于 5.0 的机器上触发。

* <h4 id="7-3"> Dalvik 分包构建每一个 dex 文件时可能出现 java.lang.NoClassDefFoundError</h4>

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
	
* <h4 id="7-4"> Java 8 methods of java.lang.Long and java.lang.Character are not desugared by D8 </h4>

	这个问题出现在使用 Kotlin 编译时，从 Kotlin1.3.30 版本开始 ndroid.compileOptions中的Java版本推断出JVM目标，如果同时设置了sourceCompatibility和targetCompatibility，则选择“1.8”到那个或更高. 可以通过指定 JavaVersion 1.6 来解决这个问题。
	
	```
	sourceCompatibility JavaVersion.VERSION_1_6
    targetCompatibility JavaVersion.VERSION_1_6
	```
	 [Issue](https://issuetracker.google.com/issues/129730297) 中表示，AGP（Android Gradle Plugin）3.4 已解决脱糖问题，可尝试升级解决。

* <h4 id="7-5"> databinding 中 findBinding vs getBinding 的场景区别 </h4>

	不同之处在于，findBinding将遍历父节点，而如果使用getBinding时当view不是跟节点会返回null。
	
* <h4 id="7-6"> 版本构建出现 Gradle sync failed: Cannot choose between the following configurations of project </h4>

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
	
* <h4 id="7-7"> gradle 配置本地离线包</h4>

	1. 离线下载 gradle 离线包保存在 url 中
	2. 修改 *gradle/wrapper/gradle-wrapper.properties* 调整 *distributionUrl* 目录指向 url
	3. 修改 *build.gradle*  classpath 版本映射 gradle 版本

* <h4 id="7-8"> 解决kvm/jvm 编译时 -classpath 遇到的分割及空格的问题</h4>

	linux/mac OS 上使用 “：” 分割多个classpath路径，window使用 “；” 分割。
	
	如果linux/mac OS 路径存在空格，暂时避免，使用多种方式尝试未果=。=。
	
* <h4 id="7-9"> databinding NoSuchMethodError with buildTool 3.4.0</h4>
	项目从 gradle 3.1 升级到 3.4.0 并使用了 androidx 之后，发现编译失败了。来项目就是使用 databinding，编译出现了
	
	```
	java.lang.NoSuchMethodError: No direct method <init>
	(Landroidx/databinding/DataBindingComponent;Landroid/view/View;I)V in 
	class Landroidx/databinding/ViewDataBinding; or its super classes
	(declaration of 'androidx.databinding.ViewDataBinding'
	```
	
	原因我们使用的 aar 库中使用了旧版本 gradle 编译，新版本主端 gradle 升级了，导致旧的 ViewDataBinding 构造器签名匹配不上新版 androidx.databinding.ViewDataBinding 的签名。
	
	```
	//旧版本
	protected ViewDataBinding(DataBindingComponent bindingComponent, View root, int localFieldCount)
	//新版本
	protected ViewDataBinding(Object bindingComponent, View root, int localFieldCount)
	```
	幸运的是，3.4.1已经修复了。更改 3.4.0 -> 3.4.1 就可以了。
	
* <h4 id="7-10"> AS连接真机调试出现 debug info can be unavailabe 的解决方法</h4>

	在使用 AS 连接华为真机调试的时候，IDE 一直出现 *“Warning: debug info can be unavailable. Please close other application using ADB: Restart ADB integration and try again”* 的错误提示。 重启 ADB 无数遍和关闭除 IDE 意外可能连接 ADB 的软件都无效，最终重启真机解决。原因是ADB连接的问题，因为有时ADB会在真实/虚拟设备上缓存一个无效的连接，并且由于该连接繁忙导致也无法连接到该设备。
	

<h3 id="8"><<版本控制篇>></h3>
	
* <h4 id="8-1"> git 修改 commit 记录</h4>

	```
	git reset --soft HEAD^
	
	撤销当前的commit，如果只是修改提示，则使用
	git commit --amend
	```

* <h4 id="8-2"> 解决git ignore 文件不生效的问题</h4>

	```
	git rm -r --cached .
	git add .
	git commit -m 'update .gitignore'
	```
	
<h3 id="9"><<其他>></h3>

* <h4 id="9-1"> ExoPlayer在接听电话之后会导致原来设置的 Source 中静音状态消失了导致可能返回 app 续播的时候视频突然有声音</h4>

	原因是多媒体焦点被通话抢夺之后播放音量被充值，解决方法可参考 https://github.com/google/ExoPlayer/issues/5092 
	
* <h4 id="9-2"> AndroidStudio 提示 Please select Android SDK</h4>

	解决手段： File->Project Structure中修改Build tools version

* <h4 id="9-3"> 关于 Java 中字符与字节的编码关系认识 </h4>

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

* <h4 id="9-4"> 关于 emoji 编码的长度计算问题 </h4>

    重点熟悉下 Unicode 编码标识的 emoji 下针对多平面 emoji 的拆分逻辑。[可参考这篇文章](http://objcer.com/2017/07/20/explore-emoji-length/)
