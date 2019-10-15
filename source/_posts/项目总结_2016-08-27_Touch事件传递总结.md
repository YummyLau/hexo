---
title: Android Touch事件传递总结
layout: post
date: 2016-08-27
comments: true
categories: Android
tags: [Android进阶] 
---
# 场景  
>　　在实际业务开发中，当Google提供的原生控件满足不了业务需求（可能实现不了或者实现需要重写修改的地方过多导致性能下降/代码耦合度太高）时，我们往往需要自己写View。在重写View的过程中经常需要处理? 点击?、? 滑动?、? 长按?等事件，因此需要对Touch事件的传递有一定的认识了解。

# 认识

## Touch传递过程
 * 分派过程，假设以Activity为最底层，则从最底层开始往最上层分派，中途可拦截;
 * 消费过程，任何一层都可以消费事件，如果不消费事件，则会往下层方向传递;如果消费了则表示该事件传递结束。

## 事务事件
　　一个事务过程有大致有(事件):1个`ACTION_DOWM`，0-N个`ACTION_MOVE`，1个`ACTION_UP`。

## 涉及方法
　　主要的方法涉及到以下3个，其中ViewGroup可重写3个，而View和Activity只能重写后两个:
<pre><code>
@Override
 public boolean onInterceptTouchEvent(MotionEvent ev) {
 //TODO 主要用于拦截事件，
 }
 @Override
 public boolean onTouchEvent(MotionEvent ev) {
 //TODO 主要用于拦截事件
 }
 @Override
 public boolean dispatchTouchEvent(MotionEvent ev) {
 //TODO 主要用于拦截事件
 }
</code></pre>

## 方法返回值说明
 * dispatchTouchEvent默认返回`false`表示不消费事件，返回`true`则表示事件已消费;
 * onInterceptTouchEvent默认返回`false`表示不拦截事件，返回`true`则表示拦截事件;
 * onTouchEvent默认返回`false`表示不消费事件，返回`true`则表示消费事件;
 * 如果调用onTouchEvent，则onTouchEvent返回值会作为dispatchTouchEvent返回值。

## 方法调用说明
 * dispatchTouchEvent方法处理的事情很多，对子View做处理（是否可用、点击、长按等）。假设只涉及4点中的三个方法，如果是ViewGroup类型，会调用onInterceptTouchEvent方法;如果是View类型，会调用onTouchEvent方法;
 * onInterceptTouchEvent方法返回`true`，则拦截了事件，即后续调用onTouch处理;返回`flase`，则不拦截,继续调用上一层子View的dispatchTouchEvent方法，(之后处理方法同上一点);
 * onTouchEvent返回`true`，则消费了事件，返回值也作为dispatchTouchEvent返回值;返回`false`则不处理事件，往下层抛事件。

# 流程
 * 以传递`ACTION_DOWM`为例子，假设布局的嵌套为：

<pre><code>（底层）Activity–>ViewGroup`–>`ViewGroup2`–>`View`(最上层)
Activity → dispatchTouchEvent（开始）
false（默认）? ViewGroup1 → dispatchTouchEvent → onInterceptTouchEvent
			 |——true ? ViewGroup1 → onTouchEvent
					 |——true ? 结束
					 |——false? Activity → onTouchEvent
							 |——true ? 结束
							 |——false? 系统处理
			 |——false? ViewGroup2 → dispatchTouchEvent → onInterceptTouchEvent
					 |——true ? ViewGroup2 → onTouchEvent
							 |——true ? 结束
							 |——false? ViewGroup1 → onTouchEvent
									 |——true ? 结束
									 |——false? Activity → onTouchEvent
											 |——true ? 结束
											 |——false? 系统处理
					 |——false?View → dispatchTouchEvent → onTouchEvent
							 |——true ? 结束
							 |——false? ViewGroup2 → onTouchEvent
						 			 |——true ? 结束
									 |——false?ViewGroup1 → onTouchEvent
											 |——true ? 结束
											 |——false?Activity → onTouchEvent
													 |——true ? 结束
													 |——false? 系统处理						
</code></pre>  
 * 从上述流程中得到：
	* 只要事件一被拦截，则进入onTouchEvent;
	* 只要事件一被消费，则事件传递结束，否则递归返回;
	* 调用onTouchEvent方法的前置条件：
		1. 要么该层拦截;
		2. 要么该层为最上层且内层都不拦截;
		3. 要么该层非最上层，上层都不消费事件。

* 另外上述三个事件还存在以下关系：
	* 传递的时间顺序：`?ACTION_DOWM?`可能前驱于`?ACTION_MOVE?`前驱于`?ACTION_UP?`;
	* 传递域的大小关系：`?ACTION_DOWM?`>=`?ACTION_MOVE?`>=`?ACTION_UP?`;

>**值得说明的是**：在分派的过程中，系统可能会分派?ACTION_CANCEL?。Google文档中大致解释这种场景为*“如果本该接收到某个事件的布局层因为事件在分派的过程中被拦截，那么该层则会收到?ACTION_CANCEL?”*。

如何理解呢?
假设当布局嵌套为：***Activity***–>***ViewGroup1***–>***ViewGroup2***–>***View***
用户点击屏幕且产生滑动行为最后离开屏幕的过程中：
* 倘若ViewGroup2 消费了`?ACTION_DOWM?`，则 View 会收到`?ACTION_CANCEL?`且后续如果还有`?ACTION_MOVE?`，则最多只能被传递到ViewGroup2(考虑到可能被内层拦截);
* 倘若ViewGroup1拦截了`?ACTION_MOVE?`，则 Activity会收到`?ACTION_CANCEL?`且后续如果还有多个`?ACTION_MOVE?`，则这些事件最多只能被传递到ViewGroup1(考虑到可能被Activity拦截)。这里要注意，如果ViewGroup1在onTouchEvent中处理`?ACTION_MOVE?`后返回false，事件也不会回传给上层(Activity);
* 倘若ViewGroup1消费了所有`?ACTION_MOVE?`，则`?ACTION_UP?`理论上会传递到ViewGroup1。当Activity拦截`?ACTION_UP?`，则ViewGroup1会收到`?ACTION_CANCEL?`。同样，如果Activity不拦截`?ACTION_UP?`且ViewGroup1在onTouchEvent中处理`?ACTION_UP?`返回false，也不会把事件回传给上层(Activity)。

从上述场景中得到：
 * 在onTouchEvent方法中，处理?ACTION_DOWM?时返回false事件会回传给上层，如果事件为?ACTION_MOVE?和?ACTION_UP?则不会;
 * ?ACTION_CANCEL?产生的条件为“事件被拦截”。在三个事件的分派流程中，如果事件中途被拦截，则拦截层的上层会收到?ACTION_CANCEL?。
 
以上总结参考Android源码与实际开发经验，如有错误请指出，共勉！
    

                                