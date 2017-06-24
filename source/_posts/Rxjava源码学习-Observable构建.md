---
title: Rxjava源码分析－一次完成的事件消费过程
date: 2017-06-24 14:23:58
categories: Rxjava
tags: [rxjava,Observable]
---
<!--more-->
> 公司在项目上用Rxjava大概也有几个月了，从之前学习Rxjava到应用，虽然能自如使用Api，但细节原理性掌握并不好。因此写下学习源码的过程，好记性不如烂笔头，慢慢学习和积累。
>文章不对基本概念做阐述，一切尽在google，侧重掌握整个流程及细节逻辑。

*Tip 源码版本：１.2.1*
 compile 'io.reactivex:rxjava:1.3.0'
 compile 'io.reactivex:rxandroid:1.2.1' 
 Ｒxjava的简介及api适用范围可参考　[Rxjava api使用范围](https://gist.github.com/longqiany/bb4bf87a2edb6d969e9d )  

 
# observable
在as中直接对`Observable`使用`Ctrl+F12`搜索方法。常用的构建`Observable`对象的方法大概有：`just`/`from`/`startWIth`/`interval`/`repeat`/`defer`/`range`,还有一个核心的方法`create`。
上述所有方法最终都会调用`create`方法实现。在rxjava1.2版本，可直接调用`create`方法构建.在之后的版本里，官方并不推荐这么做。
**create实现**
```
//1
@Deprecated
public static <T> Observable<T> create(OnSubscribe<T> f) {
	return new Observable<T>(RxJavaHooks.onCreate(f));
}

//2
public static <T> Observable<T> create(Action1<Emitter<T>> emitter, Emitter.BackpressureMode backpressure) {
	return unsafeCreate(new OnSubscribeCreate<T>(emitter, backpressure));
}

//3
public static <T> Observable<T> unsafeCreate(OnSubscribe<T> f) {
	return new Observable<T>(RxJavaHooks.onCreate(f));
}

//4
public static <S, T> Observable<T> create(SyncOnSubscribe<S, T> syncOnSubscribe) {
	return unsafeCreate(syncOnSubscribe);
}

//5
@Beta
public static <S, T> Observable<T> create(AsyncOnSubscribe<S, T> asyncOnSubscribe) {
	return unsafeCreate(asyncOnSubscribe);
}    
```
方法１中，从官方的角度看不推荐直接传递`OnSubscribe`对象直接构建`Observable`，至于原因后续文章会讲，这里只看当前`create`的使用。
方法2/4/5最终调用的是３方法，从方法名上可知是一种`unsafe`的构建做法，与后续实现的`backpressure`背压有关系。
方法５从官方的解释中看到
>@Beta
>APIs marked with the @Beta annotation at the class or method level are subject to change. They can be modified in any way, or even removed in any major or minor release but not in a patch release. If your code is a library itself (i.e. it is used on the CLASSPATH of users outside your own control), you should not use beta APIs, unless you repackage them (e.g. using ProGuard, shading, etc).

大概的意思就是该api随时会改变，app正式版本谨用，最终所有构建方法都会调用
```
public static <T> Observable<T> unsafeCreate(OnSubscribe<T> f) {
	return new Observable<T>(RxJavaHooks.onCreate(f));
}
```
而`RxjavaHooks` 是啥呢？根据RxjavaHooks类注解
>Utility class that holds hooks for various Observable, Single and Completable lifecycle-related
>points as well as Scheduler hooks.
>The class features a lockdown state, see {@link #lockdown()} and {@link #isLockdown()}, to
>prevent further changes to the hooks.
>@since 1.3

soga，是一个工具类，用于保证Observable构建过程的生命周期控制及调度控制，进一步看下去
```
public static <T> Observable.OnSubscribe<T> onCreate(Observable.OnSubscribe<T> onSubscribe) {
	Func1<Observable.OnSubscribe, Observable.OnSubscribe> f = onObservableCreate;
	if (f != null) {
	    return f.call(onSubscribe);
	}
	return onSubscribe;
}
```
  这里不深究hook的任何操作，如有机会后续深入去了解hook在控制rxjava事件流中的作用。实际上方法返回参数`onSubscribe`对象。至此，我们只需要记住，`Observable`是按照这种方式构建出来。
  
# OnSubscribe
`ＯnSubscribe`到底是什么呢？`OnSubscribe`实际上定义于**Observable**类中

```
//1
public interface Function {}

//2
public interface Action extends Function {}

//3
public interface Action1<T> extends Action {
	void call(T t);
}

//4
public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
	// cover for generics insanity
}
```
直观看出`OnSubscribe`是一个连接`Subscriber`的接口，在`call`方法中把`Subscriber`作为参数传递。而`Subscriber`是一个观察者，
`Observable`是一个被观察者，于是有一个大胆的想法，若基于观看者模式而言，以下伪代码可模拟事件响应流程
```
//步骤１，假设事件对象T，产生Ｔ
T t = new T();

//步骤２，假设处理观察者响应事件,
Observable.onSubscribe#call(Subscribe subscribe){
	subscribe.handle(t);
}
```
到这里，我们可以认为：`OnSubscribe`是一个连接观察者和被观察者的枢纽，也是事件源产生之后，观察者被回调的重要调度者。后半句话怎么理解呢？慢慢往下看。

# subscriber
`Subscriber`，rxjava中常用的观察者，看源码
```
//1
public interface Subscription {
	void unsubscribe();
	boolean isUnsubscribed();
}
//2
public interface Observer<T> {
	void onCompleted();
	void onError(Throwable e);
	void onNext(T t);
}
//3 
public abstract class Subscriber<T> implements Observer<T>, Subscription {
	//......
	public void onStart() { }
}
```
接口１提供是否取消订阅的行为。
接口２为标准观察者相应事件后的行为。显然，subscriber为rxjava中处理事件流的积累，可能自由取下订阅与处理事件，*onStart*方法用于事件处理前的行为。先忽略`Subscriber`是如何处理事件，先回到上文。拥有了`Observable`对象和`OnSubscribe`枢纽，`Observable`对象如何拿到`Subscriber`的对象引用呢？
**Observable**类源码
```  
   //1
public final Subscription subscribe(Subscriber<? super T> subscriber) {
	return Observable.subscribe(subscriber, this);
}
    
    //2
static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {

	//２.1
	if (subscriber == null) {
		throw new IllegalArgumentException("subscriber can not be null");
	}
	if (observable.onSubscribe == null) {
		throw new IllegalStateException("onSubscribe function can not be null.");
	}

	//２.２
	subscriber.onStart();

	//2.３
	if (!(subscriber instanceof SafeSubscriber)) {
		subscriber = new SafeSubscriber<T>(subscriber);
	}

	try {
		//2.４ RxJavaHooks＃onObservableStart返回onSubscribe对象
		RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);
		return RxJavaHooks.onObservableReturn(subscriber);
	} catch (Throwable e) {
		//...省略部分代码
	}
	return Subscriptions.unsubscribed();
}
```
方法１中传递`Subscriber`对象调用方法２，源码中还有该类重载了很多subscribe方法，这里只看最后调用的核心方法。
在方法２中
1. 对`Subscriber`对象做非空判断保护
2. 事件处理前的操作
3.  对`Subscriber`对象做安全保护，遵循[响应策略](http://reactivex.io/documentation/contract.html ) 
4. `OnSubscribe`调用call方法。
从上述过程可知，当`Observable`对象调用*subscribe* 方式时，其持有的 `OnSubscribe`对象会调用*call*方法。而 *call*方法里面的具体实现则得看 `OnSubscribe`子类覆盖方法逻辑。至此，观察者/被观察者/枢纽都有了，整个事件回调链算完整了，下面看一下整个完整的过程。

# example
以`just`构建`Observable`为例子：
```
Observable.just("a","b")
        .subscribe(new Subscriber<String>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(String s) {
                System.out.print(s);
            }
        });
```
运行结果
```
a b
```
按照前三节的逻辑
1. Observable.just("a","b")创建被观察者`Observable`对象
2. observable#subscriber(匿名构建Subscriber对象) 创建观察者`Subscriber`对象并通过subscribe连接
3. `OnSubscribe`调用call方法
从打印ab猜想，call方法中应该把"a","b"参数传递给`Subscriber`对象并回调*onNext*方法，所以一切逻辑应该在`OnSubscribe`子类中定义，看源码。


```
//1
public static <T> Observable<T> just(T t1, T t2) {
	return from((T[])new Object[] { t1, t2 });
}
    
//2
public static <T> Observable<T> from(T[] array) {
	int n = array.length;
	if (n == 0) {
	    return empty();
	} else
	if (n == 1) {
	    return just(array[0]);
	}
	return unsafeCreate(new OnSubscribeFromArray<T>(array));
}
```
可以看到，传递给`Observable`构造器的是`OnSubscribe`子类`OnSubscribeFromArray`对象。
```
public final class OnSubscribeFromArray<T> implements OnSubscribe<T> {
    final T[] array;
    public OnSubscribeFromArray(T[] array) {
        this.array = array;
    }

    @Override
    public void call(Subscriber<? super T> child) {
        child.setProducer(new FromArrayProducer<T>(child, array));
    }
}

//Subscriber类
public abstract class Subscriber<T> implements Observer<T>, Subscription {
	　//省略其他代码
	    public void setProducer(Producer p) {
		long toRequest;
		boolean passToSubscriber = false;
		//1
		synchronized (this) {
		    toRequest = requested;
		    producer = p;
		    if (subscriber != null) {
		        if (toRequest == NOT_SET) {
		            passToSubscriber = true;
		        }
		    }
		}
		//2
		if (passToSubscriber) {
		    subscriber.setProducer(producer);
		} else {
			//3
		    if (toRequest == NOT_SET) {
		        producer.request(Long.MAX_VALUE);
		    } else {
		        producer.request(toRequest);
		    }
		}
	    }
｝
```
首先我们传递的**a**  **b**被构建成Array对象传递给`OnSubscribeFromArray`。在*call*方法调用的时候为`Subscriber`设置了`Producer`对象，何为Producer呢？
> Producer对象，主要用于生产并控制事件有序的发射。这里大概简单理解如此即可。

在`Subscriber`设置了`FromArrayProducer`数据生产对象中，*setProducer*方法逻辑为
1. 代码１中requested表示已经请求的数据,默认是NOT_SET。所以第一次会进入锁住的代码块，第二次则进入else
2. 代码２中如果没有设置请求数目，则默认请求Long.MAX_VALUE数据。

看看`FromArrayProducer`如何实现。
```
//1
public interface Producer {
	 void request(long n);
｝

//2
static final class FromArrayProducer<T> extends AtomicLong implements Producer {
        final Subscriber<? super T> child;
        final T[] array;
        int index;

        public FromArrayProducer(Subscriber<? super T> child, T[] array) {
            this.child = child;
            this.array = array;
        }

        @Override
        public void request(long n) {
            if (n < 0) {
                throw new IllegalArgumentException("n >= 0 required but it was " + n);
            }
            //2.1
            if (n == Long.MAX_VALUE) {
                if (BackpressureUtils.getAndAddRequest(this, n) == 0) {
                    fastPath();
                }
            } else
            if (n != 0) {
                if (BackpressureUtils.getAndAddRequest(this, n) == 0) {
                    slowPath(n);
                }
            }
        }

        void fastPath() {
            final Subscriber<? super T> child = this.child;
			//2.2
            for (T t : array) {
                if (child.isUnsubscribed()) {
                    return;
                }

                child.onNext(t);
            }

		//2.3
            if (child.isUnsubscribed()) {
                return;
            }
            child.onCompleted();
        }

        void slowPath(long r) {
		//省略
        }
    }
```
上述代码可知
1. 代码１定义了一个producer接口，提供请求数据requset方法
2. 代码２实现producer接口，关联一个观察者` subscriber`对象，但是为什么数据的生产却关联观察者而不被观察者呢？实际上这里理解为数据生产个人认为是偏差的。正确的理解应该是按照观察者意愿索要的数据提供者。`array`实际上包含了所有存在的事件对象，而把所有事件对象传递给`producer`对象，由`subscriber`对象决定要request多少事件对象
3. 代码2.1中发现请求数n为Long.MAX_VALUE，默认发射所有数据
4. 代码2.2中默认传递所有事件对象给`subscriber`对象回调
5. 代码2.3中判断是否取消订阅，如果取消订阅，则直接返;否则则完成事件回调后调用`onCompleted`，遵循[响应策略](http://reactivex.io/documentation/contract.html )
至此，我们回顾下整个过程

```
Observable.just("a","b")
        .subscribe(new Subscriber<String>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(String s) {
                System.out.print(s);
            }
        });
```
调用链如下
 `observable`#*subscribe*
－－>`Onsubscriber`#*call*
－－－－>`Subscribe`#*setProducer*
－－－－－－>`FromArrayProducer`#*request*　－> #*fastPath*
－－－－－－－－>`Subscribe`#*onNext* －> #*onCompleted*

至此，一次完成的事件消费过程已完成。

