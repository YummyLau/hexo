---
title: 源码解析——LiveData & Lifecycles 完整体系详解
layout: post
date: 2020-04-04
comments: true
categories: Android进阶
tags: [Android进阶,源码分析]
---
<!--more-->

> LiveData 作为 Jetpack 的一部分，扛着 “告知界面视图发生数据变化” 的责任，常与 Lifecycles 联合使用用于数据层驱动视图层作出变化的手段。随着项目迭代，我们的项目 MVP 架构中 rxjava 驱动更新视图演化成 MVVM 架构中 rxjava + LiveData + Lifecycles 组合，rxjava 仅仅作为数据源产生-处理的手段，最后输出到 LiveData 由其根据 Lifecycles 的生命感知作出对 UI 调整。 当然，rxjava + LiveData + Lifecycles + binding 则可以进一步进行数据视图双绑定，也一直在尝试中。

LiveData + Lifecycles 未来依然作为数据层通知视图更改的手段，对于项目架构的重要角色，深入理解并熟悉运用是必要的，尤其是在 codeReview 上可以基于更多项目代码的改进，提升代码质量。

针对 LiveData + Lifecycles 体系，本文会围绕以下几点展开，顺着我的逻辑对整个体系进行掌握，相信你可以对整个体系由更完整的理解。

> 我希望你已经实践过 LiveData 并希望进一步了解更多细节。 

在此，先看一张整个体系的类图结构，整个体系可以分割为三大部分

* LiveData 如何接收并分发数据 ？
* Observer 如何关联生命周期，怎么接收数据 ？
* Activity/Fragment 是如何成为 LifecycleOwner？

### LiveData 如何接收并分发数据 

LiveData 既一个数据接收者，也是一个数据分发者。

数据接收的方法有两个 `setValue` 和 `postValue`。唯一差别在于，`posttValue` 会通过 `handler` 手段在主线程调用 `setValue`，并通过 **对象锁** 保证数据接收的同步性。

```
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {	//post之前先上锁
        postTask = mPendingData == NOT_SET;
        mPendingData = value;   //保存临时数据
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {	//主线程同步获取到临时保存的数据
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //noinspection unchecked
        setValue((T) newValue);
    }
};
```

在 `setValue` 保存了当前接收的 *value* 对象之后，*mVersion* 递增加一，接着进行数据分发。*mVersion* 实际上表示的是 **LiveData** 当前接收到的数据的 “新鲜程度”。*mVersion* 值越大数据就越新，这个 “新鲜程度” 对比的时后面 **Observer** 已经接收的数据。

在看分发逻辑之前，有两个比较关键的变量需要先理解好

* *mDispatchingValue* 表示当前 **LiveData** 是否正在分发数据
* *mDispatchInvalidated* 表示当前 **LiveData** 当前分发是否无效，如果当前分发无效则需要重新分发/

为什么 LiveData 当前分发会无效？ 实际上是因为，**LiveData** 分发是一个针对特定 *observer* 或遍历所有 **Observer** 并逐个分发的过程。当分发到中间时，突然有新的 *observer* 绑定进来并且其处于激活状态，或者原有的 *observer* 从非激活状态转变为激活状态，则会再次调用 *dispatchingValue* ，此时第一次的分发还没有完成则需要先丢弃已经分发的数据再次重新分发。

```
private void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {		//如果正在分发，则认为已经遍历分发的过程是无效的
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) { //break 之后重新循环遍历        
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}

private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    if (!observer.shouldBeActive()) {	//如果当前状态发生变化，则
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    observer.mObserver.onChanged((T) mData);
}
    
```

上面讲述的逻辑你可能有疑惑 *observer* 不是一个单纯的观察者吗？ 但是上面的逻辑却涉及是否激活 （mActive） 等信息。实际上，**Observer** 确实是一个单纯的观察者，只是 **LiveData** 在分发数据的时候当它与某个拥有生命周期特征的角色相关联了。

下面就是介绍 “Observer 作为一个数据接收的角色，是怎么关联某个拥有生命周期的角色，进而怎么影响它接收数据的”


### Observer 如何关联生命周期，怎么接收数据

上一章节，**LiveData** 分发的对象实际上的 **ObserverWrapper** 对象. 从类的字面意识就是 观察者的包装类。所谓包装，就是让观察者要观察数据前，先看看。。。。。。。。经过包装的观察者，不会一旦检测到有数据变动就会立刻有所行动（回调），而是要先看与自己关联的角色是否处于激活，如果未激活，那么数据变动都不会有所行动。但是一旦从未激活变成激活，则会尝试获取当前最新的数据。



* LiveData 是一个数据接收/分发的角色
* Observer 是一个数据接收的角色，关联着某个拥有生命周期的角色，只有该角色处于激活状态，Observer 才能接收到最新数据
* Android 中 Activity/Fragment 时如何扮演 “拥有生命周期” 的角色




@startuml

class Lifecycle{
+{abstract} addObserver(LifecycleObserver observer)
+{abstract} removeObserver(LifecycleObserver observer)
+{abstract} State getCurrentState()
}

enum Lifecycle.Event{
ON_CREATE
ON_START
ON_RESUME
ON_PAUSE
ON_STOP
ON_DESTROY
ON_ANY
}

enum Lifecycle.State{
DESTROYED
INITIALIZED
CREATED
STARTED
RESUMED
+isAtLeast(State state)
}

Lifecycle +-- Lifecycle.Event
Lifecycle +-- Lifecycle.State

interface LifecycleOwner{
~getLifecycle():Lifecycle
}

LifecycleOwner -- Lifecycle

class LifecycleRegistry{
-State mState
-WeakReference<LifecycleOwner> mLifecycleOwner
-FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap
-moveToState(State next)
+handleLifecycleEvent(Lifecycle.Event event)
+markState(State state)
+getCurrentState():State
+addObserver(LifecycleObserver observer) 
+removeObserver(LifecycleObserver observer)
}

class LifecycleRegistry.ObserverWithState{
~State mState
~GenericLifecycleObserver mLifecycleObserver
~void dispatchEvent(LifecycleOwner owner, Event event)
}
LifecycleRegistry +-- LifecycleRegistry.ObserverWithState
Lifecycle <|--  LifecycleRegistry

class ComponentActivity{
-LifecycleRegistry mLifecycleRegistry
+Lifecycle getLifecycle() : LifecycleRegistry
}
ComponentActivity -->  LifecycleRegistry
ComponentActivity <--  LifecycleRegistry
LifecycleOwner <|-- ComponentActivity

class FragmentActivity{
~FragmentController mFragments
-{stataic}  markState(FragmentManager manager, Lifecycle.State state):boolean
markFragmentsCreated()
}
ComponentActivity <|--  FragmentActivity

class FragmentController{
+dispatchCreate()
+dispatchActivityCreated()
+dispatchStart()
+dispatchResume()
+dispatchPause()
+dispatchStop()
+dispatchDestroyView()
+dispatchDestroy()
}

class FragmentManager{
+dispatchCreate()
+dispatchActivityCreated()
+dispatchStart()
+dispatchResume()
+dispatchPause()
+dispatchStop()
+dispatchDestroyView() 
+dispatchDestroy()
-dispatchStateChange(int nextState)
~moveToState(int newState, boolean always)
}

FragmentActivity --> FragmentController : 委托
FragmentController -- FragmentManager : 委托

interface LifecleObserver{
}

interface GenericLifecycleObserver{
~onStateChanged(LifecycleOwner source, Lifecycle.Event event)
}

GenericLifecycleObserver --> LifecycleOwner
LifecleObserver <|-- GenericLifecycleObserver

class ContentProvider{
}
class ProcessLifecycleOwnerInitializer{
+boolean onCreate()
}
ContentProvider <|-- ProcessLifecycleOwnerInitializer

class ProcessLifecycleOwner{
-LifecycleRegistry mRegistry
-{static} ProcessLifecycleOwner sInstance
~{static} init(Context context)
~void activityStarted()
~void activityResumed()
~void activityPaused()
~void activityStopped()
~attach(Context context) 
}
LifecycleOwner <|-- ProcessLifecycleOwner

class LifecycleDispatcher{
~{static}init(Context context)
}

ProcessLifecycleOwnerInitializer --> ProcessLifecycleOwner
ProcessLifecycleOwnerInitializer --> LifecycleDispatcher







class LiveData<T>{
~{static} int START_VERSION = -1
-int mVersion = START_VERSION
~{static} Object NOT_SE
-volatile Object mData = NOT_SET
~Object mPendingData = NOT_SET
-SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers
~int mActiveCount = 0
-boolean mDispatchingValue
-boolean mDispatchInvalidated
-void considerNotify(ObserverWrapper observer)
~void dispatchingValue(ObserverWrapper initiator)
+observe(LifecycleOwner owner, Observer<? super T> observer) 
+observeForever(Observer<? super T> observer) 
+removeObserver(final Observer<? super T> observer)
+removeObservers(LifecycleOwner owner) 
#postValue(T value) 
#setValue(T value)
#onActive() 
#onInactive()
}

class LiveData.ObserverWrapper{
~Observer<? super T> mObserver
~boolean mActive;
~int mLastVersion = START_VERSION
{abstract}shouldBeActive()
~detachObserver()
~activeStateChanged(boolean newActive)
}

LiveData +-- LiveData.ObserverWrapper



class LiveData.LifecleBoundObserver{
-LifecycleOwner mOwner
~shouldBeActive()
+onStateChanged(LifecycleOwner source, Lifecycle.Event event)
-detachObserver()
}
GenericLifecycleObserver <|.. LiveData.LifecleBoundObserver
LiveData +-- LiveData.LifecleBoundObserver
LiveData.ObserverWrapper <|.. LiveData.LifecleBoundObserver
note bottom of LiveData.LifecleBoundObserver : observer 关联生命周期

class LiveData.AlwaysActiveObserver{
~shouldBeActive():true
}
LiveData +-- LiveData.AlwaysActiveObserver
LiveData.ObserverWrapper <|.. LiveData.AlwaysActiveObserver

@enduml

