---
title: 系统源码解析——Messenger
layout: post
date: 2018-09-26
comments: true
categories: Android
tags: [源码解析] 
---

### 主要流程
使用messenger传递message，通过handler进行处理，完成ipc通讯
* 客户端，收到服务端binder之后转成messenger对象，然后send发送message会触发服务端的Handler#handleMessagae
* 服务端创建自己的service，然后在onBind方法中返回messenger.getBinder

### 源码分析

#### 服务端
```
public class MessengerService extends Service {
    private static final String TAG = "MessengerService";

    // 用来处理接收 msg 的 handler
    private Handler mMessengerHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            Log.d(TAG, "Recv Data : " + msg.getData().getString("msg"));
        }
    };

    // 定义 Messenger 对象，将关联的 Handler 作为参数传递进去
    private final Messenger mMessenger = new Messenger(mMessengerHandler);

    @Override
    public IBinder onBind(Intent intent) {
    	// 返回 Messenger 中的 Binder 对象
        return mMessenger.getBinder();
    }
}
```

#### 客户端
```
public final class Messenger implements Parcelable {
    private final IMessenger mTarget;

    //从messenger中获取handler对象
    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }

    //从服务端binder转成客户端messenger对象，并获取handler
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }

    
    //使用messenger关联的handler发送message，会调用hanler的handleMessage
    public void send(Message message) throws RemoteException {
        mTarget.send(message);
    }
    
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
  
}

 private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
        	// 将 Binder 对象转换为 Messenger
            mMessenger = new Messenger(service);
            // 构建个 Message 并将 Bundle 数据设置进去
            Message msg = Message.obtain();
            Bundle bundle = new Bundle();
            bundle.putString("msg", "I am Messenger from Client");
            msg.setData(bundle);
            try {
            	// 使用 Messenger 发送消息给远程服务端
                mMessenger.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
    };

```
### 注意事项
* Messenger 一次只处理一个请求，是串行执行的，因此在服务端不用考虑线程同步的问题，这有点类似于 IntentService 中的串行问题，因为都是使用的 Handler、MessageQueue 机制。
* Messenger 只能用来传递消息，不能跨进程调用远程的方法。
* msg.obj 只能传输系统实现了 Parcelable 接口的对象，一般情况下也不要使用 obj 这个字段跨进程传输，可以使用 Bundle 对象来替代 obj，Bundle 可以支持大量的数据类型。

