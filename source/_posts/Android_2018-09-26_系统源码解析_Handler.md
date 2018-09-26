---
title: 系统源码解析——Handler
layout: post
date: 2018-09-26
comments: true
categories: Android
tags: [源码解析] 
---

### 核心类
* Looper， 负责在 Queue 中读取 Message 并调用对应Handler的处理方法；一个线程对应一个looper对象，同时初始化 messageQueue
* MessageQueue，FIFO（先入先出队列）的消息队列
* Handler，持有 mLooper 对象，可存在多个 handler 对象对应一个 looper 和一个messageQueue
* message，持有 target（handler）和 callback

### Handler构建
* handler 有多个构造器，接受looper或者获取当前线程 looper 对象
* 实现 handleMessage 方法用于处理自定义 message 逻辑

```
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
    
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    //调用的是sThreadLocal.get()，一个线程对应一个looper对象
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

//重写handleMessage处理message
public void handleMessage(Message msg) {
}
```

### Looper对象

* prepare 初始化 looper 对象， 在 looper 对象构造器中初始化 messagequeue
* loop 开始循环遍历消息
* 主线程 loop 为何不会发生阻塞呢 ？ 

```
//prepare为当前线程初始化一个looper对象
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
//looper对象构造器
private Looper(boolean quitAllowed) {
    //构建messagequeue
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
//loop
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
    //...略部分代码
    for (;;) {
        Message msg = queue.next(); // might block
        msg.target.dispatchMessage(msg);
    }
}


```
既然消息循环的必要性，那为什么这个死循环不会造成ANR异常呢？ 因为Android 的是由事件驱动的，looper.loop()  不断地接收事件、处理事件，每一个点击触摸或者说 Activity 的生命周期都是运行在 Looper.loop() 的控制之下，如果它停止了，应用也就停止了。只能是某一个消息或者说对消息的处理阻塞了 Looper.loop()，而不是 Looper.loop() 阻塞它。

### Message
* 默认可缓存 MAX_POOL_SIZE = 50 个 message 对象
* 缓存复用 message 

```
//常用于获取一个 message
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}

//回收 message
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}

```







