---
title: 系统源码解析——Handler
layout: post
date: 2018-09-26
comments: true
categories: Android
tags: [源码解析] 
---
<h3 id="0">解析背景</h3>

常见于异步线程持有主线程 *handler* 对象，借助于 *handler* 发送 *message* 回调其 dispatchMessage 方法完成异步通讯 。 再者，Android 源码中大量使用 *handler* 用于 UI 线程间/线程内通讯。比如 Messener，ViewRootImpl 等等。最近在整理以前源码越苏笔记， 就把整个源码过程都分享下 。

<h3 id="1">源码解析</h3>

<h4 id="10">涉及核心类</h4>
* Looper 类， 负责在 *messageQueue* 中读取 *message* 并调用对应 target (handler) 的处理方法; 一个线程对应一个 *looper*，同时初始化 *messageQueue*
* MessageQueue 类，单向链表
* Handler 类，持有 *looper* 对象，可存在多个 *handler* 对象对应一个 *looper* 和一个*messageQueue*
* Message 类，持有 target（handler）和 *callback*

<h4 id="11">解析思路</h4>
先 “剧透” 一些剧情， 对整个 Handler 体系有很大的帮助： “每一个线程有且仅有一个 *looper* 用于轮询该线程需要处理的消息， 所以也只有一个 *messageQueue* 用于存放消息。 消息可由不同的发送者进行发送， 所以允许有多个 *handler* 发送 *message* 对象到 *messageQueue* 中，但是取出的消息要怎么处理呢？就需要 *message* 得持有 *handler* ，已便取出 *message* 之后可以获取 *handler* 进行分发处理。
所以，按照下面源码阅读有助于理解。
Looper -> MessageQueue -> Handler -> Message

<h4 id="12">Looper</h4>
主要作用是：在线程中初始化 *looper* 对象和 *messageQueue* 对象， 并且保证一个线程有且仅有一个 *looper* 对象； 轮询读取 *messageQueue* 消息。

Android 默认提供了 *sMainLooper* - UI 线程 *looper* 对象方便任何地方获取。
>值得留意：在  Android 8.0 源码中 **prepareMainLooper** 方法会在 ActivityThread#main 和 SystemServer#run 方法中被调用， 这也说明了 Android UI 线程为何不需要我们主动初始化 looper 的原因 。

```
//私有静态变量
private static Looper sMainLooper;

//初始化主线程 looper
public static void prepareMainLooper() {
    //核心初始化
	prepare(false);
	synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        //获取当前线程 looper 
        sMainLooper = myLooper();
        }
}

//获取主线程 looper
public static Looper getMainLooper() {
    synchronized (Looper.class) {
    	return sMainLooper;
    }
}
```

上述代码中，**prepare** 方法为初始化 *looper* 对象的入口。
```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

//所有 prepare 最终调用该方法
private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread");
	}
	sThreadLocal.set(new Looper(quitAllowed));
}

//调用构造器
private Looper(boolean quitAllowed) {
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}
```
*sThreadLocal* 内部实现为为每个线程保留特定的信息， 这里即为每个线程保留 *looper* 对象。所以如果已经初始化过 *looper* 对象，则会异常。
了解的如果初始化之后，剩余的就是如果轮训消息。

```
//略去不影响主干的部分代码，包括一些判空行为
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
    for (;;) {
        Message msg = queue.next(); // might block
        msg.target.dispatchMessage(msg);
        msg.recycleUnchecked();
    }
}
```
上述的流程精简之后是否干净，就是在代码调用线程获取当前线程 *looper* 对象， 取出 *messageQueue* 读取 *message* 消息， 并调用 Handler#dispatchMessage 进行消息分发 。

> 值得留意： 既然在 UI 线程循环，那是否意味着会发生阻塞呢？ 是的 ！但是 UI 线程是 Android 应用的基础，该线程通过 Looper#loop 不断接受事件和分发处理事件， 包括每一个点击， Activity的每个生命周期下的行为等， 如果一旦 UI 线程停止了， 应用也就挂了。 所以只会存在 某一个消息的分发处理会阻塞该循环， 这就意味着不能有耗时操作了。

<h4 id="13">MessageQueue</h4>
既然知道该类是用于存放消息对象， 核心功能在于存取。 整个类有 900+ 行，有部分 native 方法。 这里只看核心的两个方法。
* next 读取消息
* enqueueMessage 存放消息

```
Message next() {
	//native 判断，如果 loop 被停止。
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    
    //这两个变量对理解整个方法有很大的帮助。 
    //pendingIdleHandlerCount 用于计算通知 “队列情况” 时所调用的对象个数，比如我们可以注册 IdleHandler 用于接收 “looper 处理完所有 message” 这个行为的通知。
    //nextPollTimeoutMillis 表明要多久之后再次查询队列，用于当查询到的异步 message 还到时间点处理时记录下改时间。
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;

    for (;;) {
        synchronized (this) {
            // 获取开机到现在的时间
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
          	//如果遇到消息屏障，则会跳过同步 message
            if (msg != null && msg.target == null) {
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            //如果消息之后
            if (msg != null) {
                if (now < msg.when) {
					//消息还没有准备好，意味着该 message 还没有到时间点进行处理
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                	//获取 message 
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {              
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
          		//没有更多消息，设置 -1 
                nextPollTimeoutMillis = -1;
            }

            //终止处理
            if (mQuitting) {
               	//dispose()，这里我把调用的方法放进来，等价于下面三句代码
                if (mPtr != 0) {
            		nativeDestroy(mPtr);
            		mPtr = 0;
        		}
                return null;
            }

            //如果是首次静止，则获取 IdleHanlders 进行回调
            //IdleHandlers 当且仅当队列为空或者队列头部消息被延迟处理时会被调用。
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            //如果没有IdleHandlers 需要回调，则阻塞
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

 		//调用 IdleHandler#queueIdle，其作用只是通知该队列的情况
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
            	//如果外部返回 flase ，意味着不再继续监听队列状态
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                	//移除回调集
                    mIdleHandlers.remove(idler);
                }
            }
        }
        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0;
    }
}

```
**next** 方法主要需要理解以下几点
值得注意的是： 在整个流程中有一个注释为 “遇到消息屏障”（即对应 *message.target* 是为null ），会循环找出第一个异步消息 ，所有的同步消息将在本次遍历中都会被忽略，直到下次没有消息屏障的时候才取出运行。 
* 如果没有异步消息，则直接阻塞
* 如果有异步消息需要处理，先判断时间是否到了，如果没有则需要设置阻塞时间
* 否则把消息返回被设置 mBlocked = false 表示没有阻塞。

那什么时候会出现消息屏障呢 ？

```
//这里会插入一个消息屏障，在这个屏障之后的所有同步消息都不会被执行，即使时间已经到了也不会执行。
private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
//用于移除对应的屏障
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```
那消息屏障的运用有哪些呢 ？ 在系统源码 **ViewRootImpl** 中可见。
```
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //插入消息屏障
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

void unscheduleTraversals() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
//移除消息屏障        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        mChoreographer.removeCallbacks(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}
```
就是为了在处理系统更新视图 UI 时跳过当前队列中所有的同步消息，保证更新任务时最高优先级。
接下来看如何插入消息。
```
//略去部分代码
boolean enqueueMessage(Message msg, long when) {
    
    synchronized (this) {
    	//如果已经暂停，则回收消息
        if (mQuitting) {
        	msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;	//以开机时间到此刻的时长尺子，小于该值则立刻执行，大于则延迟
        Message p = mMessages; 
        boolean needWake;
        //如果当前队列为空或者when小于列表头节点when或者when为0，则直接插入头部，唤醒队列
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            //按照 when 从小到大规则插入 message
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // 唤起队列，mPtr != 0，mQuitting 为 false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
**enqueueMessage** 插入代码指定了一个 *message* 按照 **when** 规则插入消息队列。
* 如果当前队列为空或者when为0（立刻执行）或者when比头部链表节点when小，则插在头部
* 其他场景下按照 when 从小到大规则查到相邻节点（when比他大），插在它前面。


<h4 id="14">Message</h4>
如果仔细看了前两节，相信对整个存取过程会有清晰的理解。
接下来看一下在 *messageQueue* 中存储的消息 *message* 是怎样的存在。
```
//支持跨进程层/intent传递
public final class Message implements Parcelable {
    public int what;				//message标示，可理解id
    long when;						//何时被取出处理
    Handler target;					//发送者
    Message next;					//下一节点
    int flags;				 		//标示 message 的使用状况
    
    public int arg1;				//数据传输携带的辅助数据
    public int arg2;
    public Object obj;
    
    public Messenger replyTo;						//跨进程通讯使用
    public int sendingUid = -1;
    
    private static final int MAX_POOL_SIZE = 50;	//缓存相关
    private static int sPoolSize = 0;
    private static final Object sPoolSync = new Object();
    private static Message sPool
    
    static final int FLAG_IN_USE = 1 << 0;		
	static final int FLAG_ASYNCHRONOUS = 1 << 1;
	static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;
	
	//空构造器，通过静态方法获取
	public Message() {
    }
}
```

上述是绝大部分成员变量，主要涉及定义 message 的状态及获取回收 message 时缓存的处理。

```
//所有获取方法最后的入口
public static Message obtain() {
	//线程安全
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
```
可见缓存池实际上是一个 *message* 链表，*sPool* 为链表头部。 

```
void recycleUnchecked() {
    flags = FLAG_IN_USE;		//回收的时候设置 FLAG_IN_USEs
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
    	//如果缓存池没有满，则缓存 *message* 对象
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```
获取和回收流程比较简单， 看这里细心的读者可能有个疑惑 “那 message 的 flags 何时才会设置 FLAG_ASYNCHRONOUS” 呢 ？ 
```
int flags;
static final int FLAG_ASYNCHRONOUS = 1 << 1;
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```
这里的同步异步可能和我们理解线程同异步有所偏差。
实际上无论是同步消息还是异步消息，都是放到 *messageQueue* 中按照 **when** 规则进行获取分发 。但是当有一些 “紧急事务” 需要在队列中优先被执行，则可以通过 “同步屏障” 来过滤掉原来本在排队执行的同步消息，直接运行最接近运行时间的异步任务 。上述 **ViewRootImpl** 绘制就是一个很好的例子 。 

<h4 id="15">Handler</h4>
```
final Looper mLooper;		
final MessageQueue mQueue;		//消息队列，mLooper一一对应
final Callback mCallback;		//callback 接口
final boolean mAsynchronous;	//是否是异步消息

//不指定 looper 的构造器
public Handler(Callback callback, boolean async) {
	//默认获取当前线程 looper 对象
    mLooper = Looper.myLooper();
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

//指定 looper 的构造器
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;		//如果没有设置，则默认为 null
    mAsynchronous = async;		//如果没有设置，则默认为 false
}
```
上述指明了核心的成员变量及两种构造器 。
在 [Looper](#12) 的 **loop** 方法可知， 通过 [MessageQueue](#12) 的 **next** 方法获取异步消息之后调用 Handler#dispatchMessage 分发消息。

```
public void dispatchMessage(Message msg) {
	//如果存在 message.callback 不为空则调用 handleCallback
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
    	//如果设置了 Callback 对象，则使用 Callback 处理
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //否则默认使用 handleMessage处理
        handleMessage(msg);
    }
}
//mCallback是一个内部定义的接口
//也就是说如果有已知 handler ，可以通过设置 Callback 对象拦截并实现自己的逻辑。
public interface Callback {
    public boolean handleMessage(Message msg);
}

//handleMessage 是一个空方法，默认子类覆盖处理自己的分发逻辑
public void handleMessage(Message msg) {
}

//handleCallback 直接调用 callback#run 方法
private static void handleCallback(Message message) {
    message.callback.run();
}
```
其内部允许设置 *mCallback* 拦截 *handler* 的处理逻辑，另外如果一个 *message* 携带有 *callback* ，则直接调用 Callback#run 完成分发处理 。
上述代码为处理消息，以下代码为发送消息。
```
//发送代码会调用
public final boolean sendMessageDelayed(Message msg, long delayMillis){
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    //这里延迟处理使用的是开机到现在的时间长度加上我们需要延迟的毫秒数
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
//再调用
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    return enqueueMessage(queue, msg, uptimeMillis);
}
//最终调用
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	//设置 target 为发送者
    msg.target = this;
    //设置发送的消息是否是异步消息
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
也就是说，*handler* 决定了发送的 *message* 是否为异步消息 。
至此，整个 Handler 体系解析完毕 。

如果错误，请不吝指出 ！

<h3 id="2">参考文献</h3>
[你知道android的MessageQueue.IdleHandler吗？](https://wetest.qq.com/lab/view/352.html)







