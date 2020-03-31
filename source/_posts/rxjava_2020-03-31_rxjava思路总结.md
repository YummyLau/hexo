---
title: Rxjava 的归纳思考
layout: post
date: 2020-03-31
comments: true
categories: Java
tags: [RxJava] 
---

> 最近 Code Review 项目的时候，发现有一些场景使用 rxjava 的写法并不对。后来在团队一问才了解有些 api 大家虽然使用熟悉，但是内部的原理及 api 的使用场景理解得并不好。所以希望在技术认识一致性的前提下，把 rxjava 的体会同步给伙伴。

Rxjava从4年前走进 Androider 的视野后非常火，除了编程风格深受喜爱外，开发效率的提升也是至关重要。一直以来 Rxjava 都非常喜欢，非常顺手。从原来 MVP 模式架构中使用纯 Rxjava 来完成数据-视图的更新交互到 MVVM 模式架构中使用 Rxjava + LiveData ，虽然视图层我们去掉了使用 Observable/Flowable 来交互，但是总体上使我们的框架越来越严谨清晰。至于未来一段时间，Rxjava 的使用依旧乐观。

Rxjava 的掌握仅仅从两方面切入就足矣。熟悉这两方面的原理及多关注官方 api 的更新及使用，Rxjava 变会成为你的开发利器。

> 希望下面每一句都能理解清楚，能力有限，如有表达不当请多包涵

### 理解观察者、被观察者的关联及事件触发

* 理解 **Observable** 被观察者是如何构建的，有哪些 api 可以符合项目场景使用。*observable* 持有 *observableOnSubscribe* ，当事件被订阅的时候，*ObservableOnSubscribe#subscribe* 被调用，事件就被触发了。

* 理解 **Observer/Subscriber** 观察者是如何消费的，有哪些 api 可以符合项目场景使用.

* 观察者和被观察者的关联是通过 **Observable#subscribe** 绑定的,进而触发事件.

	```
	observable.subscribe(observer)
	//最终调用于
	observableOnSubscribe.subscribe(observer);
	```
也就是说，当在绑定的那一刻，被观察者才会开始触发事件，消费者才开始消费事件，而这一过程全部在 *observableOnSubscribe#subscribe* 被处理了。

### 掌握变换的原理机制才可以对操作符的使用游刃有余

**lift变换** 是操作符的基础，**lift变换** 的本质就是  *“针对事件序列的处理和再发送“* 。

基于操作符生成新的观察者，新的观察者重新订阅原被观察者。核心逻辑如下：

```
Observer newObserver = operator.apply(observer);
source.subscribe(newObserver)
//等价于 
observableOnSubscribe.subscribe(newObserver)
```

每次经过 *lift()* 调用返回的 *observable* 会成为下游 **Observable#subscribe** 的调用者，而新的 *observable*  在执行 **Observable#subscribe** 的时候，会完成上面代码端的逻辑。不要忘记,新的 Observable 中持有的 source 就是 上游的 *observable*，*observableOnSubscribe* 就是上游 *observable* 的 *observableOnSubscribe*。

那么 N 次的变换抽象为

```
Observable.lift1().lift2()...liftN-1().liftN().subscribe(observer)
```

第 N 次 *lift()* 返回的 *observable#subscribe* 逻辑为 

```
Observer observerN = operatorN.apply(observerN-1);
ObservableN-1.subscribe(observerN)
```
经过 N-1 次递归订阅之后，等价为

```
//经过 N-1 次递归订阅之后，等价为
Observer observer1 = operatorN1.apply(observer);
Observer observer2 = operatorN2.apply(observer1);
......
Observer observerN-2 = operatorN-2.apply(observerN-3);
Observer observerN-1 = operatorN-1.apply(observerN-2);
Observer observerN = operatorN.apply(observerN-1);
Observable.subscribe(observerN)   //等价于下一句
observableOnSubscribe.subscribe(observerN)  //等价于下一句
observableOnSubscribe.subscribe(operatorN.apply( ... ( operator3.apply( operator2.apply( operator1.apply( observer ) ) ) ) )  )
```

*每一次变换无非就是生成新的观察者，而新的观察者重新订阅原被观察者。当下游完成订阅之后，向上游搜索源被观察者。无论经过多少次变换 最终observableOnSubscribe 对象就是源被观察者 observable 创建时持有的 observableOnSubscribe 对象*

**基于 lift 变换的线程切换 subscribeOn 和 observeOn**

* *subscribeOn* 线程切换的逻辑为 **“通过 scheduler 把 source.subscribe(observer)  进行线程切换”**。*source* 为上游 **Observable** 对象，则 *subscribeOn* 最终作用在源 *observableOnSubscribe.subscribe*。只需要指定一次，且以第一次指定为有效切换。

* *observerOn* 线程切换的逻辑为 **“通过 scheduler 把 operator.apply(observer) 进行线程切换”**。则下游提供的 *observer* 会被上游的 *observable* 所订阅，变换。所以每一次 *observerOn* 指定的是下游的线程环境。

上述 N 次变换有

```
observableOnSubscribe.subscribe(operatorN.apply( ... ( operator3.apply( operator2.apply( operator1.apply( observer ) ) ) ) )  )
```

存在线程切换作用后为

**observableOnSubscribe.subscribe**   <== *subscribeOn*    
*observeOn* ==>    **(operatorN.apply( ... ( operator3.apply( operator2.apply( operator1.apply( observer ) ) ) ) )**

> 灵活使用变换才能让承载着复杂业务逻的代码代码变得干净清晰









