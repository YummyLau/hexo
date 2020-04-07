---
title: 源码解析——LiveData & Lifecycles 完整体系详解
layout: post
date: 2020-04-06
comments: true
categories: Android进阶
tags: [Android进阶,源码分析]
---
<!--more-->

> LiveData 作为 Jetpack 的一部分，扛着 “告知界面视图发生数据变化” 的责任，常与 Lifecycles 联合使用用于数据层驱动视图层作出变化的手段。随着项目迭代，我们的项目 MVP 架构中 rxjava 驱动更新视图演化成 MVVM 架构中 rxjava + LiveData + Lifecycles 组合，rxjava 仅仅作为数据源产生-处理的手段，最后输出到 LiveData 由其根据 Lifecycles 的生命感知作出对 UI 调整。 当然，rxjava + LiveData + Lifecycles + binding 则可以进一步进行数据视图双绑定，也一直在尝试中。

LiveData + Lifecycles 未来依然作为数据层通知视图更改的手段，对于项目架构的重要角色，深入理解并熟悉运用是必要的，尤其是在 codeReview 上可以基于更多项目代码的改进，提升代码质量。

针对 LiveData + Lifecycles 体系，本文会围绕以下几点展开，顺着我的逻辑对整个体系进行掌握，相信你可以对整个体系由更完整的理解。

> 我希望你已经实践过 LiveData 并希望进一步了解更多细节。 

在此，先看一张整个体系的类图结构，下面任何篇章的内容都可以回归到这张结构图。整个体系可以分割为五大部分，前四部分是核心的逻辑，形成整个体系的闭环，最后一部分是扩展思考，留给读者自行思考。

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200406/livedata_1.png" align=center  />


* LiveData 如何接收并分发数据 ？
* Observer 如何关联具有生命周期特征的角色，怎么接收数据 ？
* 谁是拥有 “生命周期特征” 的角色？
* Activity/Fragment 是如何成为 LifecycleOwner？
* 扩展思考

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

为什么 LiveData 当前分发会无效？ 实际上是因为，**LiveData** 分发是一个针对特定 *observer* 或遍历所有 **Observer** 并逐个分发的过程。当分发到中间时，突然有新的 *observer* 需要分发或者 **LiveData**  接收到新的数值需要分发数据，此时第一次的分发还没有完成则被认为是无效的分发，需重新遍历分发。

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


### Observer 如何关联具有生命周期特征的角色，怎么接收数据

上一章节，**LiveData** 分发的对象实际上的 **ObserverWrapper** 对象. 从类的字面意识就是 观察者的包装类。
先举个例子

> 假如安保人员（观察者）在线下看到一个人（被观察者）在偷东西，则基于此行为（事件触发）判断他是一个小偷，立刻抓拿送到公安局，这种场景下可以理解为每一个施行偷窃（触发事件）的人（被观察者），都会被安保人员（观察者）抓拿送审。但是商场的安保人员无法实时观察每个来到商场的人员，只能在监控室中看各路的监控情况，每个监控一般对应商场的某一条通道，当该监控器正常运行（激活）且监控视频中出现偷窃行为，安保人员就会立刻采取措施抓拿小偷。但是部分商场的通道由于监控器坏了（未激活），导致拿到这条通道内出现偷窃行为，则安保人员也无法知情，自然无法采取抓拿行为。

上述例子中，监控器存在运行状态，只有当监控器正常运行的时候，安保人员才有可能对监控下的偷窃行为有所感知。如果监控器坏了，则丢失知情权。**所以所谓包装，实际上就是观察者通过关联某个角色，该角色拥有生命周期，不同生命周期都可以表现成 “激活状态” 和 “非激活状态”。**上述例子 “激活状态” 对应监控器运行状态，“非激活状态” 对应监控器损坏状态。在 Android 中 Activity 也可以看成拥有生命周期的角色，resumed 阶段则为 “激活状态”，destoryed 阶段则为 “非激活状态”。**只有当关联角色处于 “激活状态” 时观察者才能感知到被观察者的行为变化。** 

如何关联呢？ 看下 **ObserverWrapper** 的实现 （只列出相关核心方法）

```
private abstract class ObserverWrapper {
    final Observer<? super T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<? super T> observer) {
        mObserver = observer;
    }
    abstract boolean shouldBeActive();
    
    void activeStateChanged(boolean newActive) {
    	if (newActive == mActive) {
        	return;
    	}
    	mActive = newActive;
    	//...
    }
}
```
接收一个 *observer* 对象。但是没有看到有关生命周期特征的角色信息，其实是在子类 **LifecycleBoundObserver** 实现. *LiveData#observe(LifecycleOwner owner,Observer<? super T> observer)* 实现真正关联。

```
//具有生命周期的对象
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }

	//只有 started 才算 “激活状态”
    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }
    
    //实现 GenericLifecycleObserver 接口，生命周期变化也要动态调整 mActive 值
    @override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }
}
//
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    owner.getLifecycle().addObserver(wrapper);
}
```
*LiveData#observeForever(Observer<? super T> observer)* 也提供了绑定观察者的方法，不需要关联任何生命周期特征的角色，则观察者随时都能获知被观察者的数据变化.

```
private class AlwaysActiveObserver extends ObserverWrapper {

    AlwaysActiveObserver(Observer<? super T> observer) {
        super(observer);
    }

	//默认一直处于激活状态
    @Override
    boolean shouldBeActive() {
        return true;
    }
}

public void observeForever(@NonNull Observer<? super T> observer) {
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    //默认设置 mActive 为true
    wrapper.activeStateChanged(true);
}
```

上述两种场景可以看到 *mActive* 用于表示是否处于 “激活状态”，同时针对关联生命周期特征的 **LifecycleBoundObserver**，随着自己生命周期的变化，状态也会随着动态更改。

上一章可知，分发数据的逻辑最终落在 *LiveData#considerNotify* 方法

```
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

1. 如果处于非激活状态，返回；
2. 获取最新的状态，如果是从 “激活状态” -> “非激活状态”，则重新更新状态，返回；
3. 如果处于激活状态，但是已经接收过 LiveData 的最新数据，则返回；
4. 接收 LiveData 最新数据更新并更新 mLastVersion。

在 **LifecycleBoundObserver** 类中，其实现的 **GenericLifecycleObserver** 是一个用于监听 “拥有生命周期特征的角色” 的生命状态变化。 而 **“拥有生命周期特征的角色”** 到底是个什么东西，见下章节。

### 谁是拥有 “生命周期特征” 的角色
 
**LifecycleBoundObserver** 实现的 **GenericLifecycleObserver** 提供的 *onStateChanged(LifecycleOwner source, Lifecycle.Event event)* 方法中，参数1见名知义，**LifecycleOwner**, **LifecycleOwner** 是一个接口，返回 拥有 “生命周期特征” 的角色。

```
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```
**Lifecycle** 就是拥有 “生命周期特征” 的角色。

```
public abstract class Lifecycle {

    public abstract void addObserver(@NonNull LifecycleObserver observer);
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    public abstract State getCurrentState();

    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY
    }

    public enum State {

        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
```

* **Lifecycle** 是一个抽象类，*addObserver*/*removeObserver*  用于添加/移除对自身的监听，*getCurrentState* 用于返回当前自身的生命状态。
* **Event** 表示 *Lifecycle* 在某个时刻所发射的事件
* **State** 表示 *Lifecycle* 当前所处的状态

比如 *LifecycleOwner* 处于 **STARTED** 状态，*LifecycleObserver* 则会收到 **ON_CREATE** 和 **ON_START** 事件。*LifecycleObserver* 实际上是一个空方法的接口，其继承者是 **GenericLifecycleObserver**。

*LifecycleBoundObserver* 作为 **ObserverWrapper** 的实例，完成了观察者对生命周期对象的关联。

1. *LifecycleBoundObserver.mOwner* 间接持有 *Lifecycle* 对象
2. **LifecycleBoundObserver** 本身实现了 **GenericLifecycleObserver**，就是一个 *LifecycleObserver* 对象
3. 通过调用 *Lifecycle#addObserver* / *Lifecycle#removeObserver* 实现关联
4. 在关联期间，只要 *Lifecycle* 的状态有变化，就会通过 *GenericLifecycleObserver#onStateChanged* 回调，进而更新 *ObserverWrapper.mActive*.

至此，谁是拥有 “生命周期特征” 的角色的问题得到解答。同时，从 **Lifecycle** 的角度进一步阐述了 **“Observer 如何关联具有生命周期特征的角色”**。

谁是观察者清楚了。

谁是被观察者也清楚了。

谁是有 “生命周期特征的” 角色也露出水面，观察者如何 “勾搭” 她的也一清二楚。

那么整条逻辑谁来盯着 **有 “生命周期特征的” 角色** 的生命周期变化的？ 从代码的角度就是被 *Lifecycle#addObserver* 所添加的 *LifecycleObserver* 对象是何时调用 **GenericLifecycleObserver r#onStateChanged** 方法呢？ 最后一章补上这个逻辑就完整了！

### Activity/Fragment 是如何成为 LifecycleOwner？
标题实际上有些 “剧透”了。在 Android 体系中，**Activity/Fragment** 实际上都是  “生命周期特征” 的角色 的拥有者，就是他们持有 *Lifecycle* 对象。*android.arch.lifecycle* 包为 **Lifecycle** 提供了 **LifecycleRegistry** 的实现，旨在为了 **Activity/Fragment** 方便管理多个 *LifecycleObserver* 的监听。

**ComponentActivity** 作为 **Activity** 的子类，实现了 **LifecycleOwner** 接口.同时，也是 **FragmentActivity** 和 **AppCompatActivity** 的父类。

```
public class ComponentActivity extends Activity implements LifecycleOwner {

	private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

	@Override
	public Lifecycle getLifecycle() {
   	 	return mLifecycleRegistry;
	}
}
public class FragmentActivity extends ComponentActivity{}
public class AppCompatActivity extends FragmentActivity{}
```

也就是说 **Activity** 实际上是委托 *mLifecycleRegistry* 完成有关生命周期的事务处理。也就是说，如果你在 *activity* 调用 *LiveData#observe(this,observer)*

```    
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    //绑定 LifecycleObserver
    owner.getLifecycle().addObserver(wrapper);
}
```

实际上就是调用 *activity.mLifecycleRegistry#addObserver* 进行绑定。先认识下 **LifecycleRegistry** 的核心代码逻辑。

```
public class LifecycleRegistry extends Lifecycle {
	private State mState;
	private final WeakReference<LifecycleOwner> mLifecycleOwner;
	private ArrayList<State> mParentStates = new ArrayList<>();
	private int mAddingObserverCounter = 0;   //是否正在添加 observer
   private boolean mHandlingEvent = false;   //是否正在处理 event 事件
   private boolean mNewEventOccurred = false; //是否有新的事件来临
	
	//默认使用弱饮用绑定外层 activity，设置初始状态
	public LifecycleRegistry(@NonNull LifecycleOwner provider) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
    }
    
    
   //内部类，用于每个 LifecycleObserver 关联状态 Lifecycle 的状态
   static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
        	//用于保证获取到 LifecycleObserver 的实例
            mLifecycleObserver = Lifecycling.getCallback(observer);              
            mState = initialState;
        }

		  //接受 event，然后计算 state 返回，实际上 state 都是通过 event 驱动计算的
        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
    
    //方法一
    public void addObserver(@NonNull LifecycleObserver observer) {
        //保存 observer
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        
        //去重判断
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
        if (previous != null) {
            return;
        }
        
        //lifecycleOwner 是否已经被回收
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            return;
        }

		 //确保 statefulObserver 的状态是正确的
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            sync();
        }
        mAddingObserverCounter--;
    }
    
    //方法二，外部设置 event 的入口
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

	//方法三
	//把 event 转化为 state并
    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }
    

    //方法四 同步 state 给 所有observerWithState 并实现状态回调
   private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                    + "new events from it.");
            return;
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            //mState 越小，则生命周期越靠前（除了DESTROYED），可见 State 枚举类
            //如果第一个 observerWithState 的状态都小于 mState，则所有 observerWithState 存在比他大的都需要更新
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            
            //如果最后一个 observerWithState 的状态都大于 mState，则所有 observerWithState 存在比他小的都需要更新
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
    
    //方法五
    private boolean isSynced() {
        if (mObserverMap.size() == 0) {
            return true;
        }
        State eldestObserverState = mObserverMap.eldest().getValue().mState;
        State newestObserverState = mObserverMap.newest().getValue().mState;
        return eldestObserverState == newestObserverState && mState == newestObserverState;
    }
}
```

上述类信息方法比较多，总结一下：

* 方法一，用于添加监听者 LifecycleObserver ，比较好理解；
* 方法二，给外部调用，通过接受 Event 来计算当前 State；
* 方法三，真正设置 State，开始同步工作；
* 方法四和方法五，主要用于内部的状态同步流程同时把信息同步给外部监听者 LifecycleObserver。

对于 *handleLifecycleEvent*  的调用，实际上我们自然能想到在 **Activity/Fragment** 的各个生命周期阶段回调方法中调用。但是我检索了所有 **Activity/Fragment** 的源码缺没有发现有调用该方法的痕迹。对于 **Activity**，唯一可利用的就是注册*Application.ActivityLifecycleCallbacks* **Activity** 生命周期回调了。

> 如果你也看到 RxPermission 的源码，那么接下来的操作你应该都很熟悉，傀儡化 “Fragment” ！

*android.arch.lifecycle* 包有一个 **Fragment** 的实现类 **ReportFragment** ，用于帮助为 **Activity/Fragment** 分发 Event 事件的。如果 *activity* 持有 *ReportFragment* 对象，那么 *ReportFragment* 可以分发代表其 *activity* 生命周期的事件。那么我们只要重写 *Fragment* 的生命周期回调方法添加事件驱动也就可以解决 **Activity** 生命周期回调的问题。对于 **Activity**内持有的**Fragment**，我们可以通过 *FragmentController* 实例间接操作 *FragmentManager* 可获取生命周期变化时机。

> 实际上整套体系中非 AndroidX库中还存在 LifecycleRegistryOwner 这个类，旧版本 Activity/Fragment 都实现了这个类用于表示其内部都持有 LifecycleRegistry 对象。但是 AndroidX 已经完全废弃了，源码流程中我都删除了这部分信息，新版本的思路以我下面分析为主。

```
class LifecycleDispatcher {

    private static AtomicBoolean sInitialized = new AtomicBoolean(false);

	//静态方法，外部调用注册
    static void init(Context context) {
        if (sInitialized.getAndSet(true)) {
            return;
        }
        ((Application) context.getApplicationContext())
                .registerActivityLifecycleCallbacks(new DispatcherActivityCallback());
    }

    static class DispatcherActivityCallback extends EmptyActivityLifecycleCallbacks {

        @Override
        public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            ReportFragment.injectIfNeededIn(activity);
        }
    }
}    
```
上述是在 *onActivityCreated* 调用 *ReportFragment#injectIfNeededIn* 注入 *activity* 对象。

```
//以下类中省略一些源码，只保留核心逻辑
public class ReportFragment extends Fragment {

    public static void injectIfNeededIn(Activity activity) {
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
    }

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
}
```
*ReportFragment* 对象最终被添加到 *activity* 的 *fragmentManager* 中完成了处理 **Activity** 生命周期事件驱动的使命。但是对于 **FragmentActivity** 内持有的 Fragment，则需要 **FragmentController** 来处理 *Fragment* 的生命周期而存在的。

```
//类一
public class FragmentActivity extends ComponentActivity implements ...{

	//HostCallbacks 是 FragmentContainer 对象，实际上 fragment 可以被任何类所持有使用，只是我们常见的 host 就是 Activity 而已。FragmentContainer 中保存 host 的大量信息，比如 FragmentManager
	final FragmentController mFragments = FragmentController.createController(new HostCallbacks());
	
	 @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        mFragments.dispatchCreate();
    }
    
    @Override
    protected void onStart() {
        if (!mCreated) {
            mFragments.dispatchActivityCreated();
        }
        mFragments.dispatchStart();
    }
    
    @Override
    protected void onPostResume() {
        mFragments.dispatchResume();
    }
    
    @Override
    protected void onPause() {
        if (mHandler.hasMessages(MSG_RESUME_PENDING)) {
				mFragments.dispatchResume();
        }
        mFragments.dispatchPause();
    }
    
    @Override
    protected void onStop() {
        mFragments.dispatchStop();
    }
    
    @Override
    protected void onDestroy() {
        mFragments.dispatchDestroy();
    }

}

//类二
public class FragmentController {
	private final FragmentHostCallback<?> mHost;
	public void dispatchCreate() {
        mHost.mFragmentManager.dispatchCreate();
    }
    public void dispatchActivityCreated() {
        mHost.mFragmentManager.dispatchActivityCreated();
    }
    public void dispatchStart() {
        mHost.mFragmentManager.dispatchStart();
    }
    public void dispatchResume() {
        mHost.mFragmentManager.dispatchResume();
    }
    public void dispatchPause() {
        mHost.mFragmentManager.dispatchPause();
    }
    public void dispatchStop() {
        mHost.mFragmentManager.dispatchStop();
    }
    public void dispatchDestroyView() {   //目前并没有找到调用，不影响整体流程
        mHost.mFragmentManager.dispatchDestroyView();
    }
    public void dispatchDestroy() {
        mHost.mFragmentManager.dispatchDestroy();
    }
}

//类三 略部分代码，保留核心逻辑
public abstract class FragmentManager {

	//方法一
    public void dispatchCreate() {
        dispatchStateChange(Fragment.CREATED);
    }

	//方法二
    public void dispatchActivityCreated() {
        dispatchStateChange(Fragment.ACTIVITY_CREATED);
    }

	//方法三
    public void dispatchStart() {
        dispatchStateChange(Fragment.STARTED);
    }

	//方法四
    public void dispatchResume() {
        dispatchStateChange(Fragment.RESUMED);
    }
    
	//方法五
    public void dispatchPause() {
        dispatchStateChange(Fragment.STARTED);
    }

	//方法六
    public void dispatchStop() {
        dispatchStateChange(Fragment.ACTIVITY_CREATED);
    }

	//方法七
    public void dispatchDestroyView() {
        dispatchStateChange(Fragment.CREATED);
    }

	//方法八
    public void dispatchDestroy() {
        dispatchStateChange(Fragment.INITIALIZING);
    }
    
    //方法九
    private void dispatchStateChange(int nextState) {
	    try {
	        mExecutingActions = true;
	        moveToState(nextState, false); //间接调用
	    } finally {
	        mExecutingActions = false;
	    }
	    execPendingActions();
	}
	
	//方法十
	void moveToState(int newState, boolean always) {
    		//...
    		moveFragmentToExpectedState(f);
    		//...
	}
	
	//方法十一
    void moveFragmentToExpectedState(Fragment f) {
     		//...
     		moveToState(f, nextState, f.getNextTransition(), f.getNextTransitionStyle(), false);
     		//...
    }
	   
	//方法十二  
	void moveToState(Fragment f, int newState, int transit, int transitionStyle,boolean keepActive) {
	
        if (f.mState <= newState) {
            switch (f.mState) {
                case Fragment.INITIALIZING:
                    if (newState > Fragment.INITIALIZING) {

                        dispatchOnFragmentPreAttached(f, mHost.getContext(), false);
                        dispatchOnFragmentAttached(f, mHost.getContext(), false);

                        if (!f.mIsCreated) {
                            dispatchOnFragmentPreCreated(f, f.mSavedFragmentState, false);
                            f.performCreate(f.mSavedFragmentState);
                            dispatchOnFragmentCreated(f, f.mSavedFragmentState, false);
                        }
                    }
    
                case Fragment.CREATED:
                    if (newState > Fragment.CREATED) {
                        f.performActivityCreated(f.mSavedFragmentState);
                        dispatchOnFragmentActivityCreated(f, f.mSavedFragmentState, false);
                    }
                case Fragment.ACTIVITY_CREATED:
                    if (newState > Fragment.ACTIVITY_CREATED) {
                        f.performStart();
                        dispatchOnFragmentStarted(f, false);
                    }
                case Fragment.STARTED:
                    if (newState > Fragment.STARTED) {
                        f.performResume();
                        dispatchOnFragmentResumed(f, false);
                    }
            }
        } else if (f.mState > newState) {
            switch (f.mState) {
                case Fragment.RESUMED:
                    if (newState < Fragment.RESUMED) {
                        f.performPause();
                        dispatchOnFragmentPaused(f, false);
                    }
                case Fragment.STARTED:
                    if (newState < Fragment.STARTED) {
                        f.performStop();
                        dispatchOnFragmentStopped(f, false);
                    }
                case Fragment.ACTIVITY_CREATED:
                    if (newState < Fragment.ACTIVITY_CREATED) {
                        f.performDestroyView();
                        dispatchOnFragmentViewDestroyed(f, false);
                    }
                case Fragment.CREATED:
                    if (newState < Fragment.CREATED) {
                        if (f.getAnimatingAway() != null || f.getAnimator() != null) {
                            newState = Fragment.CREATED;
                        } else {
                            f.performDetach();
                            dispatchOnFragmentDetached(f, false);
                        }
                    }
            }
        }

        if (f.mState != newState) {
            f.mState = newState;
        }
   }

```

上述代码比较长，但是为了完整的逻辑链还是展示出来，最大程度去处了干扰的代码。代码从上到下，逻辑链也是从上到下串起来的。

1. 类一 **FragmentActivity** 持有 *FragmentController* 对象用于管理 **Fragment** 的生命周期，在其自身的生命周期回调中同步调用 *FragmentController* 的通知方法
2. 类二 **FragmentController** 通过 *mHost* 操作 *FragmentManager* 对象，让其代理执行生命周期调用
3. 类三 **FragmentManager** 处理了所有 **Fragment** 的生命周期回调
	* 方法一至八，对应 **Fragment** 的八个回调
	* 方法九至十二，处理 **Fragment** 的状态
	* 如果 *Fragment* 存在子 *Fragment*，则 *dispatchOnFragmentXXX* 其同步状态
	* * Fragment #performXXX* 方法用于

到这里，还没有涉及 *LifecycleRegistry#handleLifecycleEvent* 方法调用。实际上 *Fragment#performXXX* 完成了这最后一步。

```
public class Fragment implements LifecycleOwner...{

    void performCreate(Bundle savedInstanceState) {
        onCreate(savedInstanceState);
        mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
    }

    void performActivityCreated(Bundle savedInstanceState) {
        onActivityCreated(savedInstanceState);
    }

    void performStart() {
        onStart();
        mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
    }

    void performResume() {
        onResume();
        mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
    }

    void performPause() {
        mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE);
        onPause();
    }

    void performStop() {
        mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP);
        onStop();
    }

    void performDestroyView() {
        onDestroyView();
    }
    
    void performDestroy() {
        mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
        onDestroy();
 
    }
    
    void performDetach() {
        onDetach();
    }
}

```

也就是说，如果你在 *fragment* 调用 *LiveData#observe(this,observer)*

```    
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    //绑定 LifecycleObserver
    owner.getLifecycle().addObserver(wrapper);
}
```

实际上就是调用 *fragment.mLifecycleRegistry#addObserver* 进行绑定。最后在 *fragment* 各个生命周期回调用中处理 *event*，转化成 *state*，最终反馈到 *LifecycleObserver* 上！

### 扩展思考
1. 在 **Activity/Fragment 是如何成为 LifecycleOwner？** 篇章出现了 **LifecycleDispatcher** 的这个类，有个 *init* 方法，我们的应用代码似乎没有调用过 ？ 
	* 利用 **ContentProvider#onCreate** 是一个非常优秀的方法

2. 既然 **Activity/Fragment** 都能做生命周期状态感知，那么 **Application** 应该也是可以吧 ？ 要怎么做呢 ？
	* **ProcessLifecycleOwner** 就是做这个事儿！

3. 既然 **Fragment** 已经能做到感知生命周期了，那为何还需要 **RepostFragment** ？
	* *Fragment* 哪怕能感知，那什么时候能转成对应 *activity* 的生命周期呢？ 还是需要重在对应生命周期回调方法。

篇幅较长，可能花费较长时间阅读，但是相信会有很大的收获！
