---
title: 我是如何设计及改造 PanelSwitchHelper 库｜实战篇章
layout: post
date: 2020-07-27
comments: true
categories: Android
tags: [Java]
---
<!--more-->

之前分享 [拖更了三年，带回了一个非常好用的库｜墙裂推荐](https://juejin.im/post/5eddf8456fb9a04804041738) 之后，**[PanelSwitchHelper ](https://github.com/YummyLau/PanelSwitchHelper)** 库的反馈渠道收到了许多热心开发者的意见和建议，在此感谢大家！

反馈群里的朋友也反馈了一些使用过程中的问题。其中有一部分问题是如何使用 API 或者 API 使用不当导致业务场景的表现与 Demo 有所出入，我也针对每个问题认真地地解答并基于建议，但大致的场景问题基本都相同。因此，想写一篇关于 **[PanelSwitchHelper ](https://github.com/YummyLau/PanelSwitchHelper)** 原理及设计过程的文章，对于使用方法不解或技术实现感兴趣的朋友可进行参考，其次是把库的设计及改造思路暴露，或者会有更多好的想法可碰撞。

> 很多时候在网上搜索处理切换场景，得到的技术实现都是千篇一律的旧方案，开源这个库是想让更多开发者能更便捷/更好地处理切换场景，但鉴于个人能力有限，库的内部设计如可进一步改进，也欢迎任何有意见/建议的朋友可以提 PR 或反馈。 


### 如何有效计算软键盘高度

`Window` 有个属性 `softInputMode` 用于指定软键盘出现时 Window 的调整行为。比如

* `SOFT_INPUT_ADJUST_RESIZE`，软键盘出现时 Window 会缩小范围显示的 Content 区域的高度以显示软键盘。
* `SOFT_INPUT_ADJUST_PAN`，软键盘出现时会 Window 会把 Content 区域向上移动一段距离直到软键盘完全显示。
* `SOFT_INPUT_ADJUST_NOTHING`，软键盘出现时 Window 不会做任何调整。
* `SOFT_INPUT_ADJUST_UNSPECIFIED`，软键盘出现时由系统来决定如何调整 Window 内容。
* ...

上述列出来的四个值是代表最常见的模式，都是互斥使用的。由于需要计算软键盘的高度，对于无法引起界面布局变动的 `SOFT_INPUT_ADJUST_NOTHING` 对计算高度无能为力，`SOFT_INPUT_ADJUST_RESIZE` 及 `SOFT_INPUT_ADJUST_PAN` 则可以有效计算。

`SOFT_INPUT_ADJUST_RESIZE` 模式可结合监听界面布局变动来间接计算软键盘高度，逻辑如下：

```
var isKeyboardShowing = false;
window.decorView.rootView.viewTreeObserver.addOnGlobalLayoutListener {
    val screenHeight = getScreenHeightWithSystemUI(window)
    val contentHeight = getScreenHeightWithoutSystemUI(window)
    val systemUIHeight = getSystemUIHeight(window)
    val keyboardHeight = screenHeight - contentHeight - systemUIHeight
    if (isKeyboardShowing) {
        if (keyboardHeight <= 0) {
            isKeyboardShowing = false
        } 
    } else {
        if (keyboardHeight > 0) {
            isKeyboardShowing = true
            //save keyboardHeight
        }
    }
}
```
实际上 `SOFT_INPUT_ADJUST_PAN` 模式也可使用上述逻辑完成软键盘高度的计算，在尝试平滑过渡 Window 内容区域变化初期曾用过这种做法，但是由于软键盘仍然可能挡住部分业务视图，所以选择 `SOFT_INPUT_ADJUST_RESIZE` 来处理会相对灵活。

两种模式在拉起软键盘之后的视觉差异如下，图一为 `SOFT_INPUT_ADJUST_PAN`，图二为 `SOFT_INPUT_ADJUST_RESIZE`。
![](https://user-gold-cdn.xitu.io/2020/7/5/1731f8ae5812789b?w=1080&h=2340&f=png&s=94951)
![](https://user-gold-cdn.xitu.io/2020/7/5/1731f8b4b7cbe240?w=1080&h=2340&f=png&s=95190)

### 已有旧方案的弊端分析

对于旧的方案，你可能采用以下方式来适配 Window 的调整。

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <!-- 内容区域 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:orientation="vertical">
        
        <!-- 列表内容 -->
        <ListView
            android:id="@+id/list_view"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1" />

        <!-- 输入/切换面板驱动 -->
        <View
            android:id="@+id/action_view"
            android:layout_width="match_parent"
            android:layout_height="50dp" />
    </LinearLayout>

    <!-- 面板区域 -->
    <View
        android:id="@+id/panel_view"
        android:layout_width="match_parent"
        android:layout_height="200dp" />
        
</LinearLayout>
```

内容区域使用 weight 权重来适配 ContentView 可能发生的高度调整。面板区域动态隐藏，输入法显示时隐藏，点击表情等触发面板时显示。这是一种不需要任何干预布局测量及绘制的做法，LinearLayout 已经帮我们处理了。虽然 Android Studio 可能会提示 `Nested weights are bad for performance`，但这不是放弃这种做法的根本原因。

实际上 `LinearLayout` 使用 `weight` 权重分配布局区域是常见的做法，`LieanrLayout` 会对其首层子布局进行两次测量，第一次测量是为了计算未使用 `weight` 的子布局宽高，第二次则是测量为了精确使用 `weight` 子布局的宽高。虽然嵌套可能会导致嵌套内的布局进行多次测量，但这并不意味着我们应该放弃使用 **“嵌套 - weight”** 的联合手段。

大部分 Androider 第一印象会觉得 `RelativeLayout` 的性能会优于 `LieanrLayout`,优先选择 `RelativeLayout` 来编写 xml 布局。但真不是。

谷歌官方在 [Google I/O 2013 - Writing Custom Views for Android](https://www.youtube.com/watch?v=NYtB6mlu7vA&t=1m41s) 已经澄清过 “嵌套 LinearLayout-weight 会引起性能问题并不是推荐 RelativeLayout 的原因”。使用一层 **“嵌套 - weight”** 会引起两次测量，只有在嵌套多层的常见下会引起性能问题，特别是当你嵌套列表内存在 `ListView/RecyclerView` 等布局。而 `RelativeLayout` 至少会测量两次来保证子布局尺寸的正确性。在我的测试中，在 2-3 层布局结构下 `RelativeLayout` 的性能并不会优于 `LinearLayout`。

跑远了... 

旧方案为了兼容面板的显示，大致会有以下流程：
1. 在软键盘和面板区域都没有显示的前提下，直接显示软键盘或面板；
2. 在软键盘或面板区域显示的前提下，切换到软键盘或面板前，锁住内容区域，完成切换后释放内容区域；

锁住/释放内容区域高度代码为：

```
private fun lockContentHeight(contentView: View) {
    val params = contentView.layoutParams as LinearLayout.LayoutParams
    params.height = contentView.height
    params.weight = 0.0F
}

private fun unlockContentHeight(contentView: View) {
    contentView.postDelayed({ (contentView.layoutParams as LinearLayout.LayoutParams).weight = 1.0F; }, 200L)
}
```

`unlockContentHeight` 的时机并不好把控，这取决与切换的频率及切换之后布局完成调整的时间，部分性能较低的机型常有 UI 显示错误的异常。同时，由于高度的固定锁死会导致软键盘与面板布局的切换在固定的区域内完成，如果没有动画过渡则切会变得非常生硬。

在改造 `PanelSwitchHelper` 初期，尝试对面板的显示隐藏做动画过渡。

1. 软键盘未显示时显示面板，面板的渐变完成时间难以与内容区域面板的调整时间高度重合；
2. 软键盘先显示后切换面板，如果软键盘还没有隐藏就显示面板，则会引起内容区域的调整导致 UI 错位，如果软键盘要隐藏之后才显示面板，则依然会出现 “空白区域” 先腾出后面板再开始做动画过渡；
3. 如果面板是慢慢显示而输入法是慢慢隐藏，则似乎能让整体的切换变得流畅 ？ 可事实上不同机型软键盘切换速度不一样，受 ROM 和页面性能的影响，实际显示软键盘的效果差异很大。难以适配。


这困惑了我很久。既然 “面板是慢慢显示而输入法是慢慢隐藏” 这个方向应该是对的，可能只是我实现的方式有误。

### 重新设计切换的细节实现

细致的看了下微信的实现，似乎并不是简单的锁住高度实现的，看起来是整体的平移。这给了一个很好的灵感。 

* 如果要实现整体的平移，得先知道一开始整体的高度
* 如果要实现平移的高度，还得知道软键盘的高度（软键盘高度等于面板高度）
* 如果要实现平移的逻辑，可以重写 onLayout 进行干预
* 如果要干预 onLayout 的逻辑，需要收集内容区域/面板区域的布局进行动态布局
* ...

于是得先定一下整体布局结构，大致如下：

![](https://user-gold-cdn.xitu.io/2020/7/6/173223641e6648e9?w=668&h=682&f=jpeg&s=41766)

对应布局为：

```
<com.effective.android.panel.view.PanelSwitchLayout
    android:id="@+id/panel_switch_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    app:animationSpeed="standard">

    <!-- 内容区域 -->
    <!-- edit_view 指定一个 EditText 用于输入 ，必须项-->
    <com.effective.android.panel.view.content.LinearContentContainer
        android:id="@+id/content_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        ...
        
    </com.effective.android.panel.view.content.LinearContentContainer>


    <!-- 面板区域，仅能包含PanelView-->
    <com.effective.android.panel.view.panel.PanelContainer
        android:id="@+id/panel_container"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

</com.effective.android.panel.view.PanelSwitchLayout>
```

* `PanelSwitchLayout` 总容器，用于在软键盘/面板显示时干预 onLayout 进行协调控制；
* `LinearContentContainer` 线性内容区域（更多内容容器可参考库提供及自定义扩展），软键盘/面板显示后动态调整的区域；
* `PanelContainer` 面板区域，用于容纳及显示业务的所有面板。

下面根据上面的三个自定义 ViewGroup 进行详细拆解分析。

#### 内容区域的设计与实现

内容区域用于显示需要动态调整的区域。开放的框架必须能够支持原生常见的 `ViewGroup`，如 `LinearLayout`,`RelativeLayout`,`FrameLayout` 等,而不同特性的 `ViewGroup` 内部处理的逻辑大同小异。为了方便支持原生容器及扩展支持开发者可能已有的自定义 `ViewGroup`，需要使用代理模式来完成对内容区域的共有逻辑处理。

内容区域类图结构如下：

![](https://user-gold-cdn.xitu.io/2020/7/6/17322b67b3cc84c7?w=829&h=595&f=png&s=36349)

其中：
* IContentContainer 为容器区域需要实现的接口，包括获取触发面板切换的 view 映射等。
    * IResetAction 为框架自动隐藏软键盘/面板功能所需要实现的接口；
    * IInputAction 为框架处理输入View 如 Edittext 的焦点/点击等接口。
* ContentContainerImpl 为默认处理了所有 IContentContainer 所需要实现的接口，为不同特性的 ViewGroup 实现提供了便捷的代理。
* LinearContentContainer 线性容器，实现 IContentContainer 接口并把对应实现委托给 ContentContainerImpl 处理即可。
* OtherContentContainer 其他容器，可自主实现。


`IContentContainer` 暴露出来的内容大部分是为了后续 `PanelSwitchLayout` 在 `onLayout` 流程中使用而已。 

值得留意的是，框架为了最大化方便开发者处理点击内容区域可实现隐藏软键盘/面板的逻辑，提供的 `IResetAction` 功能的自由度非常高，可根据不同的业务使用及扩展，但前提是得理解其实现的原理。

![](https://user-gold-cdn.xitu.io/2020/7/6/17322c3495c84cce?w=338&h=250&f=jpeg&s=27468)

在库提供的原生 ViewGroup 扩展中有两个属性 `auto_reset_enable` 及 `auto_reset_area` 。

* `auto_reset_area` 接收一个 View 的 id，则事件落在该 view 的区域内且允许框架自动处理隐藏时，框架会尝试处理；
* `auto_reset_enable` 接收一个 Boolean 值，表示是否允许框架自动处理隐藏。当值为 true 时，当用户点击事件落在可处理区域（如果没有设置 auto_reset_area 则默认处理区域为容器大小，如果设置过则按照 auto_reset_area 指定的视图区域进行边界判断）内且没有被消费，尝试处理。

当用户点击在内容区域内且没有被消费后，就会自动隐藏已显示的软键盘/面板。如果你的内容区域内的布局比较复杂且点击命中的 View 已经消费了事件，则框架就不放弃处理，如果这种场景你还想自动隐藏，可以选择盖一个空白的 View 然后手动监听点击事件，点击时隐藏也可以。

但是对于 IM 场景中的消息列表就比较复杂。比如 RecyclerView 来实现列表后，你希望显示软键盘/面板时点击列表内空白区域（可能是 ViewHolder 填充不到的区域）时隐藏软键盘/面板，点击部分 View 比如头像，消息时又需要消费事件，这个时候还得特殊处理以下。

```
/**
 * 具体规则查 {@link com.effective.android.panel.view.content.ContentContainerImpl}
 *
 * @return
 */
@Override
public boolean onTouchEvent(MotionEvent e) {
    float curX = e.getX();
    float curY = e.getY();
    if (e.getAction() == MotionEvent.ACTION_DOWN) {
        startScroll = false;
        curDownX = curX;
        curDownY = curY;
    }

    if (e.getAction() == MotionEvent.ACTION_MOVE) {
        startScroll = Math.abs(curX - curDownX) > scrollMax || Math.abs(curY - curDownY) > scrollMax;
    }

    if (e.getAction() == MotionEvent.ACTION_UP && !startScroll) {
        return false;
    }
    return super.onTouchEvent(e);
}
```

上述代码让内容区域有了支持 RecyclerView 特殊场景的可能。由于原生 RecyclerView 会拦截事件，如果 Holder 不消费事件，则会将事件用于本身滑动消费。当非滑动时，可以尝试把最后的希望 `ACTION_UP` 丢出去。然后 `ContentContainerImpl` 尝试联合处理。

```
override fun hookDispatchTouchEvent(ev: MotionEvent?, consume: Boolean): Boolean {
    ev?.let { event ->
        if (event.action == MotionEvent.ACTION_UP) {
            action?.let {
                if (autoReset && enableReset && !consume) {
                    if (mResetView == null || eventInViewArea(mResetView, event)) {
                        it.run() //尝试隐藏逻辑
                        LogTracker.log("$tag#hookDispatchTouchEvent", "hook ACTION_UP")
                        return true
                    }
                }
            }

        }
    }
    return false
}
```

#### 面板区域的设计与实现

面板区域主要就是用于放面板及隐藏显示面板，而每一个面板实际上也是一个 View。 为了支持扩展面板，把面板的行为封装成一个接口，让自定义 View 来实现该接口完成业务面板的封装。

面板区域类图结构如下：

![](https://user-gold-cdn.xitu.io/2020/7/6/17322dbb68bcfca1?w=559&h=397&f=png&s=17528)

其中：
* ViewAssertion 只是用于确保子类实现应该是一个 View 类型
* IPanelView 封装了面板的基础功能，比如谁触发了该面板显示，重复点击触发源是否还原软键盘，是否正在显示
* PanelView 为框架模式显示的面板，支持使用 `panel_layout` 属性类 include 属性加载一个 layout.xml 布局
* PanelContainer 面板容器，通过 `SparseArray` 持有多个面板。

如果 PanelView 满足不了开发者的场景，只需要实现 `IPanelView` 接口自行扩展即可。

`PanelContainer` 最终持有的面板也是是为了后续 `PanelSwitchLayout` 在 `onLayout` 流程中使用而已。 

#### 总控制器的设计与实现

PanelSwitchLayout 中持有各类监听器，用于回调开发者设置的监听。比如触发器点击监听/面板切换监听/输入法状态监听/输入源焦点监听等。

持有 `OnScrollOutsideBorder` 对象用于实时获取模式，有固定模式和滑动模式。固定模式类旧方案，内容区域会因为软键盘/面板的显示而动态调整高度；滑动模式则不会动态调整高度，直接对内容区域进行平移处理。

```
interface OnScrollOutsideBorder {
    fun canLayoutOutsideBorder(): Boolean
    fun getOutsideHeight(): Int
}
```

持有 `DeviceRuntime` 运行时设备信息对象，用于提供横竖屏设备信息，包括状态栏/导航栏/刘海信息，是否全屏，是否 Pad 等信息。

持有 `Window` 对象，并监听 Window 调整引起的布局变化，用于获取输入法高度及状态。

```
fun bindWindow(window: Window) {
    this.window = window
    window.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_STATE_ALWAYS_HIDDEN or WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE)
    deviceRuntime = DeviceRuntime(context, window)
    deviceRuntime?.let {
        globalLayoutListener = ViewTreeObserver.OnGlobalLayoutListener {
            val screenHeight = getScreenHeightWithSystemUI(window)
            val contentHeight = getScreenHeightWithoutSystemUI(window)
            val info = it.getDeviceInfoByOrientation(true)
            val systemUIHeight = if (it.isFullScreen) {
                0
            } else {
                info.statusBarH + (if (it.isNavigationBarShow) info.getCurrentNavigationBarHeightWhenVisible(it.isPortrait, it.isPad) else 0)
            }
            val keyboardHeight = screenHeight - contentHeight - systemUIHeight
            if (isKeyboardShowing) {
                if (keyboardHeight <= 0) {
                    isKeyboardShowing = false
                    if (panelId == Constants.PANEL_KEYBOARD) {
                        checkoutPanel(Constants.PANEL_NONE)
                    }
                    notifyKeyboardState(false)
                } else {
                    if (getKeyBoardHeight(context) != keyboardHeight) {
                        PanelUtil.setKeyBoardHeight(context, keyboardHeight)
                        requestLayout()
                    }
                }
            } else {
                if (keyboardHeight > 0) {
                    isKeyboardShowing = true
                    if (getKeyBoardHeight(context) != keyboardHeight) {
                        PanelUtil.setKeyBoardHeight(context, keyboardHeight)
                        requestLayout()
                    }
                    notifyKeyboardState(true)
                }
            }
        }
        window.decorView.rootView.viewTreeObserver.addOnGlobalLayoutListener(globalLayoutListener)
    }
}
```

核心逻辑在 `onLayout` 方法中实现。

```
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    val visibility = visibility
    if (visibility != View.VISIBLE) {
        return
    }
    deviceRuntime?.let {
        val deviceInfo = it.getDeviceInfoByOrientation()
        val scrollOutsideHeight = scrollOutsideBorder.getOutsideHeight()
        val paddingTop = paddingTop
        var allHeight = deviceInfo.screenH

        if (it.isNavigationBarShow) {
            allHeight -= deviceInfo.getCurrentNavigationBarHeightWhenVisible(it.isPortrait, it.isPad)
        }

        val localLocation = getLocationOnScreen(this)
        allHeight -= localLocation[1]
        var contentContainerTop = getContentContainerTop(scrollOutsideHeight)
        contentContainerTop += paddingTop
        val contentContainerHeight = getContentContainerHeight(allHeight, paddingTop, scrollOutsideHeight)
        val panelContainerTop = contentContainerTop + contentContainerHeight

        //计算实际bounds
        val changeBounds = isBoundChange(l, contentContainerTop, r, panelContainerTop + scrollOutsideHeight)
        LogTracker.log("$TAG#onLayout", " changeBounds : $changeBounds")
        if (changeBounds) {
            val reverseResetState = reverseResetState()
            LogTracker.log("$TAG#onLayout", " reverseResetState : $reverseResetState")
            if (reverseResetState) {
                setTransition(animationSpeed.toLong(), panelId)
            }
        }

        //处理第一个view contentContainer
        run {
            contentContainer.layoutContainer(l, contentContainerTop, r, contentContainerTop + contentContainerHeight)
            LogTracker.log("$TAG#onLayout", " layout参数 contentContainer : height - $contentContainerHeight")
            LogTracker.log("$TAG#onLayout", " layout参数 contentContainer : " + " l : " + l + " t : " + contentContainerTop + " r : " + r + " b : " + (contentContainerTop + contentContainerHeight))
            contentContainer.changeContainerHeight(contentContainerHeight)
        }

        //处理第二个view panelContainer
        run {
            panelContainer.layout(l, panelContainerTop, r, panelContainerTop + scrollOutsideHeight)
            LogTracker.log("$TAG#onLayout", " layout参数 panelContainerTop : height - $scrollOutsideHeight")
            LogTracker.log("$TAG#onLayout", " layout参数 panelContainer : " + " l : " + l + "  : " + panelContainerTop + " r : " + r + " b : " + (panelContainerTop + scrollOutsideHeight))
            panelContainer.changeContainerHeight(scrollOutsideHeight)
        }
        return
    }

    //预览的时候由于 helper 还没有初始化导致可能为 null
    super.onLayout(changed, l, t, r, b)
}
```

大致流程如下：

1. 如果视图不可见，则不需要处理；
2. 如果运行时设备信息不可用，则交给 super 处理，兼容 IDE 预览的功能
3. 计算内容区域的高度
    * 屏幕的高度减去可能显示导航栏高度
    * 再减去屏幕上 panelSwitchLayout Y 坐标绝对值的高度
    * 如果是固定模式且当前显示软键盘/面板，还需要减去软键盘高度
    * 最后减去 panelSwitchLayout.paddingTop 高度
4. 计算内容区域和面板区域的布局信息，包括高度及layout时的坐标，如果是滑动模式且当前显示软键盘/面板，内容区域的 Top 坐标为 -H(软键盘高度)
5. 判断 panelSwitchLayout 的 Bound 是否发生更改，如果发生改变则需要用 changeBounds 过渡；
6. 布局并调整内容区域及面板区域。

最后还需要暴露切换软键盘/面板的入口，定义了一个状态 id 来区分三种场景

* `Constants.PANEL_NONE` 默认状态，软键盘/面板都没有显示
* `Constants.PANEL_KEYBOARD` 仅软键盘显示
*  非上述两种值表示仅面板显示

逻辑如下：

```
/**
 * 是否成功切换面板，当 panelId 为表示面板时，使用触发器 view 的 id 来表示。
 */
fun checkoutPanel(panelId: Int): Boolean {

    //如果正在切换，则跳过
    if (doingCheckout) {
        LogTracker.log("$TAG#checkoutPanel", "is checkouting,just ignore!")
        return false
    }
    doingCheckout = true
    
    //如果已经切换，则跳过
    if (panelId == this.panelId) {
        LogTracker.log("$TAG#checkoutPanel", "current panelId is $panelId ,just ignore!")
        doingCheckout = false
        return false
    }

    when (panelId) {
        Constants.PANEL_NONE -> { //隐藏输入法
            hideKeyboard(context, contentContainer.getInputActionImpl().getInputText())
            contentContainer.getInputActionImpl().clearFocusByEditText()
            contentContainer.getResetActionImpl().enableReset(false)
        }
        Constants.PANEL_KEYBOARD -> { //显示输入法
            val showKbResult = showKeyboard(context, contentContainer.getInputActionImpl().getInputText())
            if (!showKbResult) {
                LogTracker.log("$TAG#checkoutPanel", "system show keyboard fail, just ignore!")
                doingCheckout = false
                return false
            }
            contentContainer.getResetActionImpl().enableReset(true)
        }
        else -> { //隐藏可能显示的输入法，把对应面板显示出来
            val size = Pair(measuredWidth - paddingLeft - paddingRight, getKeyBoardHeight(context))
            val oldSize = panelContainer.showPanel(panelId, size)
            if (size.first != oldSize.first || size.second != oldSize.second) {
                notifyPanelSizeChange(panelContainer.getPanelView(panelId), isPortrait(context), oldSize.first, oldSize.second, size.first, size.second)
            }
            hideKeyboard(context, contentContainer.getInputActionImpl().getInputText())
            contentContainer.getResetActionImpl().enableReset(true)
        }
    }
    //记录状态并同步监听，requestLayout 重新布局
    this.lastPanelId = this.panelId
    this.panelId = panelId
    LogTracker.log("$TAG#checkoutPanel", "checkout success ! lastPanel's id : $lastPanelId , panel's id :$panelId")
    notifyPanelChange(this.panelId)
    requestLayout()
    doingCheckout = false
    return true
}
```

切换过程中缓存状态都是为了效率考虑，剩余的就是一些状态互斥的处理了。

到此， PanelSwitchLayout 的核心逻辑大致都已经讲述完了。

### 库 API 封装设计与实现

考虑到需要配置各种状态监听器，模式配置及调试开关，使用 `Builder` 模式来构建比较合理。

```
mHelper: PanelSwitchHelper? = null
if (mHelper == null) {
    mHelper = PanelSwitchHelper.Builder(this) //可选
            .addKeyboardStateListener {
                onKeyboardChange { visible, height ->
                    Log.d(TAG, "系统键盘是否可见 : $visible ,高度为：$height")
                }
            }
            .addEditTextFocusChangeListener {
                onFocusChange { _, hasFocus ->
                    Log.d(TAG, "输入框是否获得焦点 : $hasFocus")
                }
            }
            .addViewClickListener {
                onClickBefore {
                    Log.d(TAG, "点击了View : $it")
                }
            }
            .addPanelChangeListener {
                onKeyboard {
                    Log.d(TAG, "唤起系统输入法")
                }
                onNone {
                    Log.d(TAG, "隐藏所有面板")
                }
                onPanel {
                    Log.d(TAG, "唤起面板 : $it")
                }
                onPanelSizeChange { panelView, _, _, _, width, height ->
                    Log.d(TAG, "输入法高度动态变化引起的面板高度调整")
                }
            }
            .contentCanScrollOutside(true)
            .logTrack(true)
            .build()
}
```
结合 kotlin DSL 特性，用户可自由显示所需要的状态监听。比如针对 `addPanelChangeListener` 可仅选择重载 `onKeyboard`就可以，而 Java 可能就没办法了。`addPanelChangeListener` dsl 扩展如下

```
interface OnPanelChangeListener {
    fun onKeyboard()
    fun onNone()
    fun onPanel(panel: IPanelView?)
    fun onPanelSizeChange(panel: IPanelView?, portrait: Boolean, oldWidth: Int, oldHeight: Int, width: Int, height: Int)
}

//自定义类型
private typealias OnKeyboard = () -> Unit
private typealias OnNone = () -> Unit
private typealias OnPanel = (view: IPanelView?) -> Unit
private typealias OnPanelSizeChange = (panelView: IPanelView?, portrait: Boolean, oldWidth: Int, oldHeight: Int, width: Int, height: Int) -> Unit

class OnPanelChangeListenerBuilder : OnPanelChangeListener {

    private var onKeyboard: OnKeyboard? = null
    private var onNone: OnNone? = null
    private var onPanel: OnPanel? = null
    private var onPanelSizeChange: OnPanelSizeChange? = null

    override fun onKeyboard() {
        onKeyboard?.invoke()
    }

    override fun onNone() {
        onNone?.invoke()
    }

    override fun onPanel(panel: IPanelView?) {
        onPanel?.invoke(panel)
    }

    override fun onPanelSizeChange(panel: IPanelView?, portrait: Boolean, oldWidth: Int, oldHeight: Int, width: Int, height: Int) {
        onPanelSizeChange?.invoke(panel, portrait, oldWidth, oldHeight, width, height)
    }

    fun onKeyboard(onKeyboard: OnKeyboard) {
        this.onKeyboard = onKeyboard
    }

    fun onNone(onNone: OnNone) {
        this.onNone = onNone
    }

    fun onPanel(onPanel: OnPanel) {
        this.onPanel = onPanel
    }

    fun onPanelSizeChange(onPanelSizeChange: OnPanelSizeChange) {
        this.onPanelSizeChange = onPanelSizeChange
    }
}
```

而在 Builder 添加监听的时候采用以下代码完成监听器绑定

```
fun addPanelChangeListener(function: OnPanelChangeListenerBuilder.() -> Unit): Builder {
    panelChangeListeners.add(OnPanelChangeListenerBuilder().also(function))
    return this
}
```
其他监听器的实现也如此。

在调用 `build(showKeyboard: Boolean = false)` 构建 `PanelSwitchHelper` 时完成布局校验及设置 `PanelSwitchLayout` 所需要的信息即可。

```
fun build(showKeyboard: Boolean = false): PanelSwitchHelper {
    findSwitchLayout(rootView)
    requireNotNull(panelSwitchLayout) { "PanelSwitchHelper\$Builder#build : not found PanelSwitchLayout!" }
    return PanelSwitchHelper(this, showKeyboard)
}
```
如果框架找不到 `PanelSwitchLayout` 则会抛出运行时错误。

```
// PanelSwitchHelper
init {
    //全局 log 标志
    Constants.DEBUG = builder.logTrack
    //模式设置
    canScrollOutside = builder.contentCanScrollOutside
    if (builder.logTrack) {
        builder.viewClickListeners.add(LogTracker)
        builder.panelChangeListeners.add(LogTracker)
        builder.keyboardStatusListeners.add(LogTracker)
        builder.editFocusChangeListeners.add(LogTracker)
    }
    mPanelSwitchLayout = builder.panelSwitchLayout!!
    //绑定模式信息，外部可动态调整模式
    mPanelSwitchLayout.setScrollOutsideBorder(object : OnScrollOutsideBorder {
        override fun canLayoutOutsideBorder(): Boolean {
            return canScrollOutside
        }

        override fun getOutsideHeight(): Int = getKeyBoardHeight(mPanelSwitchLayout.context)
    })
    //绑定状态监听器
    mPanelSwitchLayout.bindListener(builder.viewClickListeners, builder.panelChangeListeners, builder.keyboardStatusListeners, builder.editFocusChangeListeners)
    //绑定 window 信息
    mPanelSwitchLayout.bindWindow(builder.window)
    if(showKeyboard){
        mPanelSwitchLayout.toKeyboardState()
    }
}
```
至此，框架已经初始化完毕了。后续的所有软键盘/面板操作，界面都会自动响应处理。 

整体效果如下：

![](https://user-gold-cdn.xitu.io/2020/7/7/17326e6f224b5ea8?w=200&h=433&f=gif&s=785716)

![](https://user-gold-cdn.xitu.io/2020/7/7/17326e87e27a2a28?w=200&h=433&f=gif&s=955636)

对了，当你的页面需要拦截返回时，别忘记了，`PanelSwitchHelper` 可尝试隐藏面板哦

```
@Override
public void onBackPressed() {
    if (mHelper != null && mHelper.hookSystemBackByPanelSwitcher()) {
        return;
    }
    super.onBackPressed();
}
```

如果你有更好的想法或者意见，欢迎评论哦。

如果你在使用 [PanelSwitchHelper](https://github.com/YummyLau/PanelSwitchHelper) 遇到任何难题，可提 *issue*，任何问题我都会第一时间回复处理。


文章首发于 [掘金-我是如何设计及改造 PanelSwitchHelper 库｜实战篇章](https://juejin.im/post/5f0198ede51d4534b66d421a) 欢迎关注我 👏 
