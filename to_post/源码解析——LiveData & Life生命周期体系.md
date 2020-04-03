---
title: Kotlin开发踩坑录(持续更新...)
layout: post
date: 2019-09-14
comments: true
categories: Android
tags: [Android经验,kotlin]
---
<!--more-->




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

