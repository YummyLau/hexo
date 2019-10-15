---
title: 系统源码解析——AIDL
layout: post
date: 2018-09-26
comments: true
categories: Android
tags: [源码解析] 
---

### Linxu 及 Android IPC
#### linxu进程间通信手段
* 管道 Pipe，在创建时分配一个page大小的内存，缓存区大小比较有限（拷贝2次）
* 信号 Signal，不适用于信息交换，更适用于进程中断控制，比如非法内存访问，杀死某个进程等
* 信号量 semaphore，常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段
* 套接字 Socket，作为更通用的接口，传输效率低，主要用于不通机器或跨网络的通信（拷贝2次）
* 共享内存 ，无须复制，共享缓冲区直接付附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决
* 消息队列 Message，信息复制两次，额外的CPU消耗；不合适频繁或信息量大的通信（拷贝2次）

#### Android - Binder驱动
* Binder采用 C/S架构
*  选择理由：从性能、稳定性、安全性、语言层面4个角度综合考虑的结果
    * 性能：Binder数据拷贝只需要一次，而管道、消息队列、Socket都需要2次，但共享内存方式一次内存拷贝都不需要；从性能角度看，Binder性能仅次于共享内存
    * 稳定性：Binder是基于C/S架构的，简单解释下C/S架构，是指客户端(Client)和服务端(Server)组成的架构，Client端有什么需求，直接发送给Server端去完成，架构清晰明朗，Server端与Client端相对独立，稳定性较好；而共享内存实现方式复杂，没有客户与服务端之别， 需要充分考虑到访问临界资源的并发同步问题，否则可能会出现死锁等问题；从这稳定性角度看，Binder架构优越于共享内存
    * 安全性：传统Linux IPC的接收方无法获得对方进程可靠的UID/PID，从而无法鉴别对方身份；Android为每个安装好的应用程序分配了自己的UID，故进程的UID是鉴别进程身份的重要标志，前面提到C/S架构，Android系统中对外只暴露Client端，Client端将任务发送给Server端，Server端会根据权限控制策略，判断UID/PID是否满足访问权限，目前权限控制很多时候是通过弹出权限询问对话框，让用户选择是否运行。传统IPC只能由用户在数据包里填入UID/PID；另外，可靠的身份标记只有由IPC机制本身在内核中添加。其次传统IPC访问接入点是开放的，无法建立私有通道。从安全角度，Binder的安全性更高。
    * 语言层面：Linux是基于C语言(面向过程的语言)，而Android是基于Java语言(面向对象的语句)，而对于Binder恰恰也符合面向对象的思想，将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法，而其独特之处在于Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中。Binder模糊了进程边界，淡化了进程间通信过程，从语言层面，Binder更适合基于面向对象语言的Android系统。
* 所有ipc都是通过driver传送

#### Android - 多进程
* 不同进程不能通过内存来达到共享数据，同一个类会产生副本（常见就是通过静态变量来传递）
* 静态变量与单例模式失效
* 线程同步机制失效
* sharedPreferences可靠性下降，底层读写cml，并发会有问题
* Application会创建多次，因为运行在同一进程的组件应该属于同一个虚拟机和同一个Application



### 定义AIDL接口
* 导入的包名 
* 如果有使用Object对象，需要该对象 implement Parcelable 接口，并且需要导入该接口包名+类名；如果是primitive type就不需要，同时需要创建一个同名的aidl文件。所有的.adil文件已经需要传递对象接口，需要在Service和Client中各一份
* 创建一个aidl文件，定义暴露的方法
* 客户端需要实现：conn接口，拿到接口对象
* 服务端实现stub，实现抽象方法

### 获取aidl对象
获取手段，本地service
* 先启动服务
    * startService启动，则还没有得到服务端Binder
    * 绑定启动则能直接得到
* 取得ServiceConnection对象
```
public class MainActivity extends AppCompatActivity{
private IMyAidlInterface iMyAidlInterface; 
@Override 
  protected void onCreate(Bundle savedInstanceState){ 
      super.onCreate(savedInstanceState);
     setContentView(R.layout.activity_main); 
     bindService(new Intent("cc.abto.server"), new ServiceConnection() { 
    @Override
    public void onServiceConnected(ComponentName name, IBinder service){
         iMyAidlInterface = IMyAidlInterface.Stub.asInterface(service);
     } 

    @Overridepublic void onServiceDisconnected(ComponentName name){ 
     }
     }, BIND_AUTO_CREATE); 
 }

    public void onClick(View view){
        try {
         Toast.makeText(MainActivity.this, iMyAidlInterface.getName(), Toast.LENGTH_SHORT).show();
     } catch (RemoteException e) { 
         e.printStackTrace(); 
     }
     }
 }
```


### 通讯过程
* 服务端，Binder类对象
    * 重载onTransact方法，用于存放数据，有4个参数
        * int code，区分具体的功能
        * Parcel data，客户端发送的数据
        * Parcel reply，服务端回复的数据
        * int flags 约定双发是否进行双向通讯
    * 接收Binder驱动发送的消息，执行onTransact方法
* 驱动端
    * 服务端在创建Binder对象时驱动端会创建对应的mRemote对象，也是Binder对象
* 客户端
    * 重载transact方法，同样也有4个参数
    * 获取服务端在驱动端对应的mRemote对象，并调用transact方法

```
public interface IBinder { 
 ... ... // 查看 binder 对应的进程是否存活
public boolean pingBinder();

// 查看 binder 是否存活，需要注意的是，可能在返回的过程中，binder 不可用 
  public boolean isBinderAlive();

/**
 * 执行一个对象的方法， 
 * 
 * @param 需要执行的命令
 * @param 传输的命令数据，这里一定不能为空
 * @param 目标 Binder 返回的结果，可能为空
 * @param 操作方式，0 等待 RPC 返回结果，1 单向的命令，最常见的就是 Intent.
 */
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException;

// 注册对Binder死亡通知的观察者，在其死亡后，会收到相应的通知 
  // 这里先跳过，后续
public void linkToDeath(DeathRecipient recipient, int flags) throws RemoteException; 
 ... ... 
 }
```

#### aidl接口生成 Stub
```
interface IMyAidlInterface {
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```
生成的实现IInterface类
* stub内部类
    * asInterface静态方法
    * onTransact方法，读取参数，写入值，根据code值读取每个方法对应的解析规则
* proxy内部类
    * 每一个接口定义的方法，写入参数，读取值，调用，mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0)
```
public interface IMyAidlInterface extends android.os.IInterface{

//生成的stub内部类
public static abstract class Stub extends android.os.Binder implements com.example.code.IMyAidlInterface   {
private static final java.lang.String DESCRIPTOR = "com.example.code.IMyAidlInterface";
public Stub(){
    this.attachInterface(this, DESCRIPTOR);
}

//用户外部service获取
public static com.example.code.IMyAidlInterface asInterface(android.os.IBinder obj){
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin!=null)&&(iin instanceof com.example.code.IMyAidlInterface))) {
    return ((com.example.code.IMyAidlInterface)iin);
}
    return new com.example.code.IMyAidlInterface.Stub.Proxy(obj);
}

@Override public android.os.IBinder asBinder(){
    return this;
}

@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException{
    switch (code){
        case INTERFACE_TRANSACTION:{
reply.writeString(DESCRIPTOR);
return true;
}
case TRANSACTION_basicTypes:{
    data.enforceInterface(DESCRIPTOR);
    int _arg0;
    _arg0 = data.readInt();
    long _arg1;
    _arg1 = data.readLong();
    boolean _arg2;
    _arg2 = (0!=data.readInt());
    float _arg3;
    _arg3 = data.readFloat();
    double _arg4;
    _arg4 = data.readDouble();
    java.lang.String _arg5;
    _arg5 = data.readString();
    this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
    reply.writeNoException();
    return true;
}
    }
    return super.onTransact(code, data, reply, flags);
}

private static class Proxy implements com.example.code.IMyAidlInterface {
        private android.os.IBinder mRemote;
    Proxy(android.os.IBinder remote)    {
            mRemote = remote;
    }

@Override 
public android.os.IBinder asBinder(){
    return mRemote;
}

public java.lang.String getInterfaceDescriptor(){
    return DESCRIPTOR;
}

/**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
@Override 
public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException{
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        _data.writeInt(anInt);
        _data.writeLong(aLong);
_data.writeInt(((aBoolean)?(1):(0)));
_data.writeFloat(aFloat);
_data.writeDouble(aDouble);
_data.writeString(aString);
mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
_reply.readException();
    }finally {
_reply.recycle();
_data.recycle();
        }
}
}
    static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
}
/**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException;
}
```




