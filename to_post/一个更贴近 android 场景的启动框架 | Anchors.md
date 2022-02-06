---
title: 一个更贴近 android 场景的启动框架 | Anchors
layout: post
date: 2020-07-27
comments: true
categories: Android
tags: [Java]
---
<!--more-->


### 背景
随着公司项目需求迭代，项目依赖库越来越多，`Application#onCreate()` 承载的初始化逻辑变得越来越复杂。

以上一年线上项目的初始化逻辑例子。

```
@Override
public void onCreate() {
    super.onCreate();
    if (LeakCanary.isInAnalyzerProcess(this))
        return;
    if (YXFDebuggable.strictMode()) {
        StrictModeUtil.startStrictMode();
    }
    AppDirs.init(app, PACKAGE_NAME);        	// dirs
    YXFStorage.init(app);   // storage
    YXFLogs.init(app, YXFDebuggable.log(), YXFBuildInfo.getBuildStamp()); 	// logs
    YXFBuildInfo.init(app);
    YXFDeviceInfo.init(app);  	// device & build info
    YXFReceivers.init(app); 
    GLLogManager.init(app);
    CrashHandler.getInstance().init(app); 	//crash
    YXCrashManager.init(app, new Bugrpt163Capture());  // skin
    ServiceManager.register(app, SkinGuideImpl.class);
    ServiceManager.register(app, SkinServiceImpl.class);
    //rxjavaplugins
    GodlikeRxPlugins.init();
        //省略 200 行逻辑
    ...
}
```
项目的初始化代码真的又臭又长... 在第一次项目重构的时候，尝试拆分初始化逻辑, 把原来的所有初始化逻辑划分为同步初始化和异步初始化，相对聚合的逻辑合成一个 Task 任务，Task 任务之间可相互依赖。在代码逻辑上划分三个方法来承载 Task 初始化。

* onCreateBlock(), 确保所有的 Task 在主线程上执行且在 `Application#onCreate()` 前完成
* onCreateSync(),确保主线程上执行完成，使用 `handle#post()` 发送 Task 到主线程消息队列排队等待执行
* onCreateAsync(),在子线程上完成所有 Task 执行

代码逻辑大致如下。

```
@Override
public void onCreate() {
    super.onCreate();
    if (LeakCanary.isInAnalyzerProcess(this)) {
        return;
    }
    onCreateBlock(this);
    onCreateSync(this);
    onCreateAsync(this);
}
private void onCreateBlock(Application app) {
    //task running
}
private void onCreateSync(Application app) {
    //post task to looper
}
private void onCreateAsync(final Application app) {
      new Thread(new Runnable() {
        @Override
        public void run() {
        	//task running
        }).start();
}
```

`onCreateBlock()` 内的任务为串行初始化，很大程度会影响到应用首帧显示的时机。如果 `onCreateBlock()` 耗费的时间太长，那就要好好考虑下这些代码存在的必要性及合理性，重新设计了。
`onCreateSync()`  对首帧显示的时机无影响，`onCreateAsync()` 则不一定。

倘若不进行任务分类初始化，则首帧显示前需要完成所有任务的初始化

![](https://user-gold-cdn.xitu.io/2020/7/24/17380cf17ac0b9b1?w=1686&h=214&f=jpeg&s=59978)

分类之后，对原来 Task 进行依赖排列如下 

![](https://user-gold-cdn.xitu.io/2020/7/24/17380cefe3c80e89?w=1183&h=629&f=jpeg&s=69420)

降低初始化阻塞时间收益如下

![](https://user-gold-cdn.xitu.io/2020/7/24/17380cf2f146fecf?w=1039&h=433&f=jpeg&s=62975)


然而在实际场景中，拆分出来的三个方法中的任务可能存在依赖关系导致情况变得复杂：

1. onCreateBlock() 只能依赖 onCreateAsync() 内任务m依赖 onCreateSync() 会导致死锁；
2. onCreateSync()  可依赖其他两个方法内的任务；
3. onCreateAsync() 可依赖其他两个方法内的任务,同时可尽可能支持多条子线程任务来加快缩短所有异步任务的完成时间，这取决于当前设备的 CPU 状态


![](https://user-gold-cdn.xitu.io/2020/7/24/17380d5b608588e1?w=717&h=748&f=jpeg&s=57642)

如果按照上述的逻辑来重新梳理启动任务的初始化，则需要实现一下逻辑：
1. 封装 Task 支持上述三种场景的运行
2. 提供线程池以运行异步任务，Handler#Post 运行同步非阻塞任务
3. 以 Task 为图节点，构建一张应用依赖启动图并从头部开始初始化
4. Task 运行状态支持拦截提供外部业务逻辑获取状态，打断初始化等
5. 多条异步子链运行时尽可能保持同时并发
6. ...

希望完成上述功能，优先考虑现有轮子。于是在 github找到了 [alpha](https://github.com/alibaba/alpha) 。 alpha  是一个阿里巴巴开源的，基于PERT图构建的Android异步启动框架，协助应用启动时正确执行依赖任务。集成了之后发现满足不了项目的应用场景，当时并没有很好的解决方法，迫于项目需求当晚就 clone 下了源码研究了实现，略感失落，但也找到了优化的方向。

### alpha 的缺陷

1. 启动节点粒度不够细

    **alpha** 封装了 Task 类用于表示一个异步任务, 衍生出Project 及 AnchorTask 用于处理多个 task 对象构成的图结构。其外层业务只需要继承 Task 或者同个构建 Project 来编写业务依赖逻辑,理想的结构应该如下
    ![](https://user-gold-cdn.xitu.io/2020/7/21/173701f13f298121?w=466&h=808&f=png&s=33688)
    但是如果你添加 Task 启动的时候，会收到 `xxxTask cannot be cast to com.alibaba.android.alpha.Project`。 从源码层上看确实不能从一个 task 启动，缺乏灵活性。
    
2. 无法满足同异步初始化依赖并阻塞 Application#onCreate

    **alpha** 定位为异步启动框架。在执行启动任务时判断其是否是主线程执行，如果是则通过 `handler#post` 发送出去排队处理, 否则交给线程池处理。任务处理完成之后通知依赖该任务的任务进行依赖检查, 此时若依赖其的所有依赖都已完成，则启动该任务。
    
    定义一个任务 T，其启动任务时刻为 tTStart，结束任务时刻为 tTEnd。若存在以下依赖任务，
    
    ```
    D（异步）-> C（异步） ->B （同步）->A（同步）
    ```
    则 **alpha** 中恒满足 `tAStart < tAEnd < tBStart < tBEnd < tCStart < tCEnd < tDStart < tDEnd`。
    由于同步任务时通过队列排队处理，任务的执行并不是与代码块的上下文严格同步，当 Application#onCreate() 中要求严格的代码执行同步时，如
    
    ```
    public void onCreate(){
        startInitTask（） //启动上述链	 
        code	          //后置代码块
    }
    ```
    则后置代码块会优先被执行。当`tCode` 为代码块执行时刻时，恒满足 ` tCode < txStart (x = {A,B,C,D}) `
    
    尽管 **alpha** 中提供 AlphaManager#waitUntilFinish 用于阻塞执行线程，但是存在很大缺陷：
    
    *假如在UI线程等待，则会造成死锁。其原因在于当前执行代码处等待解锁，而只有等到所有在主线程执行的 task 执行完才可能解锁，而 task 被 post 到消息队列里面，只有当解锁之后才能执行到消息队列的 task。*
    
3. 缺乏任务执行同步支持，同异步混合链支持及调度优化

    很多应用都会在 `application#onCreate` 前保证某些初始化任务完成之后再进入 `activity` 生命周期。同时，如果在进入 `activity` 生命周期前这块宝贵的时候可结合设备的 cpu 资源来尽可能执行一些异步初始化任务。
    

遗憾的是，官方上一次更新时 2 年前了，并没有好好打算支持并维护好 *alpha* 库。
    
### anchors 更适合 Android 启动场景

[anchors](https://github.com/YummyLau/Anchors) 是一个基于图结构，支持同异步依赖任务初始化 Android 启动框架。其锚点特性提供 "勾住" 依赖的功能，能灵活解决初始化过程中复杂的同步问题。参考 alpha 并改进其部分细节, 更贴合 Android 启动的场景, 同时支持优化依赖初始化流程, 选择较优的路径进行初始化。

目前已经稳定服务于我们线上项目一年多了，经过改造之后，相比 `alpha` 的优势

1. 支持配置 `anchors` 等待任务链，常用于 `application#onCreate` 前保证某些初始化任务完成之后再进入 `activity` 生命周期回调。
2. 支持主动请求阻塞等待任务，常用于任务链上的某些初始化任务需要用户逻辑确认；
3. 支持同异步任务链，使你的依赖链不再局限于同步或者异步，框架灵活帮你切换调度。


如果一个任务要确保在 `application#onCreate` 前执行完毕，则该任务成为锚点任务，使用起来非常简单。

添加依赖 

```
implementation 'com.effective.android:anchors:1.1.0'
```

在 `Application` 中启动依赖图，提供 java/kotlin 两套 api 方便使用。

```
//java 代码
AnchorsManager.getInstance().debuggable(true) 
        .addAnchors(anchorYouNeed)          //传递任务id设置锚点任务
        .start(dependencyGraphHead);        //传入依赖图头部
        
//kotlin code
getInstance()
    .debuggable { true }
    .taskFactory { TestTaskFactory() }          //可选，构建依赖图可以使用工厂，
    .anchors { arrayOf("TASK_9", "TASK_10") }   //传递任务 id 设置锚点任务
    .block("TASK_13") {			                //可选，如果需要用户在某个任务结束之后进行交互，则使用 block 功能
        //According to business  it.smash() or it.unlock()
    }
    .graphics {							      // 可选，当使用工程时支持 dsl 构建图，sons 接受子节点
        UITHREAD_TASK_A.sons(
                TASK_10.sons(
                        TASK_11.sons(
                                TASK_12.sons(
                                        TASK_13))),
                TASK_20.sons(
                        TASK_21.sons(
                                TASK_22.sons(TASK_23))),

                UITHREAD_TASK_B.alsoParents(TASK_22),

                UITHREAD_TASK_C
        )
        arrayOf(UITHREAD_TASK_A)
    }
    .startUp()

```

如果你打开了 *debug* 模式，则可以看到整个初始化过程的详细信息，可过渡不同 `TAG` 来获取对应信息。

* **TASK_DETAIL** 任务执行信息
    ```
     D/TASK_DETAIL: TASK_DETAIL
     ======================= task (UITHREAD_TASK_A ) =======================
     | 依赖任务 :
     | 是否是锚点任务 : false
     | 线程信息 : main
     | 开始时刻 : 1552889985401 ms
     | 等待运行耗时 : 85 ms
     | 运行任务耗时 : 200 ms
     | 结束时刻 : 1552889985686
     ==============================================
    ```

* **ANCHOR_DETAIL** 锚点任务信息

    ```
     W/ANCHOR_DETAIL: anchor "TASK_100" no found !
     W/ANCHOR_DETAIL: anchor "TASK_E" no found !
     D/ANCHOR_DETAIL: has some anchors！( "TASK_93" )
     D/ANCHOR_DETAIL: TASK_DETAIL
     ======================= task (TASK_93 ) =======================
     | 依赖任务 : TASK_92
     | 是否是锚点任务 : true
     | 线程信息 : Anchors Thread #7
     | 开始时刻 : 1552891353984 ms
     | 等待运行耗时 : 4 ms
     | 运行任务耗时 : 200 ms
     | 结束时刻 : 1552891354188
     ==============================================
     D/ANCHOR_DETAIL: All anchors were released！
    ```

* **LOCK_DETAIL** 阻塞等待信息
    ```
     D/LOCK_DETAIL: Anchors Thread #9- lock( TASK_10 )
     D/LOCK_DETAIL: main- unlock( TASK_10 )
     D/LOCK_DETAIL: Continue the task chain...
    ```

* **DEPENDENCE_DETAIL** 依赖图信息

    ```
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_9_start(1552890473721) --> TASK_90 --> TASK_91 --> PROJECT_9_end(1552890473721)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_9_start(1552890473721) --> TASK_90 --> TASK_92 --> TASK_93 --> PROJECT_9_end(1552890473721)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_8_start(1552890473721) --> TASK_80 --> TASK_81 --> PROJECT_8_end(1552890473721)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_8_start(1552890473721) --> TASK_80 --> TASK_82 --> TASK_83 --> PROJECT_8_end(1552890473721)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_7_start(1552890473720) --> TASK_70 --> TASK_71 --> PROJECT_7_end(1552890473720)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_7_start(1552890473720) --> TASK_70 --> TASK_72 --> TASK_73 --> PROJECT_7_end(1552890473720)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_6_start(1552890473720) --> TASK_60 --> TASK_61 --> PROJECT_6_end(1552890473720)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_6_start(1552890473720) --> TASK_60 --> TASK_62 --> TASK_63 --> PROJECT_6_end(1552890473720)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_5_start(1552890473720) --> TASK_50 --> TASK_51 --> PROJECT_5_end(1552890473720)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_5_start(1552890473720) --> TASK_50 --> TASK_52 --> TASK_53 --> PROJECT_5_end(1552890473720)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_4_start(1552890473720) --> TASK_40 --> TASK_41 --> TASK_42 --> TASK_43 --> PROJECT_4_end(1552890473720)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_3_start(1552890473720) --> TASK_30 --> TASK_31 --> TASK_32 --> TASK_33 --> PROJECT_3_end(1552890473720)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_2_start(1552890473719) --> TASK_20 --> TASK_21 --> TASK_22 --> TASK_23 --> PROJECT_2_end(1552890473719)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> PROJECT_1_start(1552890473719) --> TASK_10 --> TASK_11 --> TASK_12 --> TASK_13 --> PROJECT_1_end(1552890473719)
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> UITHREAD_TASK_B
     D/DEPENDENCE_DETAIL: UITHREAD_TASK_A --> UITHREAD_TASK_C
    ```


有使用锚点和使用锚点场景下, 针对每个依赖任务做 Trace 追踪, 可以通过 python systrace.py 来输出 trace.html 进行时间分析。

![](https://user-gold-cdn.xitu.io/2020/7/21/173705ad69836fd6?w=1861&h=629&f=jpeg&s=798284)

依赖图中有着一条 `UITHREAD_TASK_A -> TASK_90 -> TASK_92 -> Task_93` 依赖。假设我们的这条依赖路径是后续业务的前置条件,则我们需要等待该业务完成后再进行自身的业务代码。如果不是则我们不关系他们的结束时机。在使用锚点功能时，我们勾住 TASK_93，则从始端到该锚点的优先级将被提升，当在优先的 cpu 资源可用前提下，高优先级的链会优先抢占 cpu 资源进行使用。从上图可以看到执行该依赖链的时间缩短了。

[anchors](https://github.com/YummyLau/Anchors/blob/master/README-zh.md) 项目同时提供了核心的 Sample 场景进行演示。

1. 多进程初始化
2. 某初始化链中间节点需要等待响应
3. 某初始化链完成之后可能会再启动另一条新链

如 `某初始化链中间节点需要等待响应` 例子中，业务逻辑可决定是否终止链继续初始化等,如下点击 **继续执行** 。


![](https://user-gold-cdn.xitu.io/2020/7/21/17370752f8e8b62b?w=590&h=1280&f=gif&s=545960)

Log 信息能看到阻塞解除了。

```
D/LOCK_DETAIL: main- unlock( TASK_10 )
D/LOCK_DETAIL: Continue the task chain...
D/Anchors: TASK_10_waiter -- onFinish -- 
D/TASK_DETAIL: TASK_DETAIL
    
    ======================= task (TASK_10_waiter ) =======================
    | 依赖任务 :  
    | 是否是锚点任务 : false 
    | 线程信息 : Anchors Thread #10 
    | 开始时刻 : 1595319503047 ms
    | 等待运行耗时 : 1 ms
    | 运行任务耗时 : 3653 ms
    | 结束时刻 : 1595319506701 
    ==============================================
D/Anchors: TASK_10_waiter -- onRelease -- 
```

**anchors** 框架自由度非常高，同异步混合链及 anchor 功能的结合使用可以灵活处理很多复杂初始化场景，但是要充分理解使用功能时的线程场景哦。

如果你还在梳理凌乱无序的启动任务而烦恼，
又或者为任务的同异步任务控制而心烦， anchors 绝对适合你哦 👏 👏 👏

如果对启动场景有任何疑惑或者框架设计的意见与建议，欢迎评论留言～ 
 


文章首发于 [掘金-一个更贴近 android 场景的启动框架 | Anchors](https://juejin.im/post/5f168dd9f265da22ce394a7a) 欢迎关注我 👏 
