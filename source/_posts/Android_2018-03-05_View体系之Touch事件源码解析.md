---
title: Android View体系之Touch事件传递源码解析(8.0)
layout: post
date: 2018-03-05
comments: true
categories: Android
tags: [Android View体系,源码解析] 
---

### 技术背景
>从 View 体系中认识 Touch 事件传递，暂时留一条线索：  
>＂　View 最原始的事件从哪里来？　”  
>从 WindowCallbacKWrapper开始的。  
>那么，我们开始吧！

*tip*：阅读源码前，建议读懂 [Android View体系之基础常识及技巧](xxxxx)。

### 始于　Activity # dispatchTouchEvent
从 `window` 层开始下发事件后， `Activity` 开始处理事件，会调用 `ViewGroup#dispatchTouchEvent`  

```
Activity.java
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }

PhoneWindow.java
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }

DecorView.java
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }

ViewGroup.java
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {

	...

	}
```

`Activity#dispatchTouchEvent` 方法最后会通过 `DecorView` 触发 `ViewGroup#dispatchTouchEvent` 开始分发事件。

*总结*：`Activity` 下发 `Touch` 事件到 `DecorView` 并由 `DecorView` 开始向下传递。

### 深入 ViewGroup # dispatchTouchEvent
`DecorView` 调用 `dispatchTouchEvent` 分发 `Touch` 事件。代码很长，可是不难，逻辑比较清晰。

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {

		//默认部分下发 Touch 事件
        boolean handled = false;

		// 1. 检测是否分发Touch事件（判断窗口是否被遮挡住）
		// 如果该 Touch 事件没有被窗口遮挡，则继续下面逻辑
        if (onFilterTouchEventForSecurity(ev)) {

			// 获取 Touch Action
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

			// 2. 判断事件是否是点击事件
			// 清空所有接收触摸事件View的引用
			// 设置mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT，默认允许拦截事件
			// 设置mNestedScrollAxes = SCROLL_AXIS_NONE，默认视图不滚动
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

           	// 3. 判断事件是否需要拦截 - intercepted 
			// 判断是否运行不允许拦截
			// 如果允许拦截，则通过 onInterceptTouchEvent 方法返回
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
				//说明传递的事件不是ACTION_DOWN，意味着ACTION_DOWN已经被拦截过了。
                intercepted = true;
            }

            // 4. 判断事件是否取消事件 - canceled 
			// 如果 view.mPrivateFlags 被设置 FLAG_CANCEL_NEXT_UP_EVENT，则该 view 已经脱离视图
			// 置 view.mPrivateFlags 标志
			// 如果 当前标志为 FLAG_CANCEL_NEXT_UP_EVENT 或者 接收 MotionEvent.ACTION_CANCEL 事件，返回 true
			// 也就是说，如果View detach时，则calceled返回true
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;

			// 5. 如果事件没有被取消且没有被拦截，走下面逻辑判断是否需要传递
            if (!canceled && !intercepted) {

                // 忽略：检测无障碍焦点
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

				// 6. 如果是点击事件，区分单指，多指，或者hover（类似鼠标悬浮）
				// actionIndex 返回 0 ，id则：第一指id为0，第二指id为1，依次递增
				// idBitsToassign，第一指为1，第二指为2，第三指4，依次指数递增
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

					//清除当前手指触摸的target引用
                    removePointersFromTouchTargets(idBitsToAssign);
	
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {

                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                 		
						// 7. 扫描可传递的子 View 列表，按照Z轴坐标大小排序返回列表 preorderedList 
						// 遍历子 view 列表并结合 preorderedList  找出符合绘制条件的 view
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();

                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            //忽略：如果该 View 存在无障碍焦点，则跳过
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

							// 8. 判断子 view 是否不可见或者不在子 view 的范围内
							// 如果满足，则跳过
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

							// 9. 遍历 target 链表，找到当前 child 对应的 target
							// 如果存在，则把手指的触摸 id 赋值 pointerIdBits 遍历
							// 这个比较难理解，举个栗子。
							// 假如我食指按在一个 view(A) 上，然后中指在食指未起来之前又按在 view(A)
							// 上，则会把后者的 idBitsToAssign 更新到指向view(A) 的 newTouchTarget 对象上
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

							// 清除子 view.mPrivateFlags PFLAG_CANCEL_NEXT_UP_EVENT 标志
                            resetCancelNextUpFlag(child);

							// 10. 返回下发 Touch 事件结果。
							// 如果没有子 view，则返回 view#dispatchTouchEvent 结果，实际上这里会发生递归。
							// 如果子 view 消费了 Touch 事件，则会添加到 target 链接
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
    
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

					// 11. 如果没有子 view 没有消费 Touch 事件且 mFirstTouchTarget 却不为 null
					// 这就郁闷了，比较难理解，举个栗子。
					// 前一个 if 条件是 newTouchTarget == null && childrenCount != 0
					// 如果我第一根手指按在 viewGroup（A）中 textview（a）
					// 同时，另一根手指按在除 textview（a）外空白区域（viewGroup（A）内）
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

	
			// 12. 如果 mFirstTouchTarget == null 意味着 没有任何 view 消费该 Touch 事件
			// 则 viewGroup 会调用dispatchTransformedTouchEvent处理，但是child==null，会调用
			// view#dispatchTouchEvent处理。
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
				// 13. 当前有子 view 消费了 Touch 事件
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;

					//如果当前 target是新加入
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {

						//是否需要向子 view 传递 ACTION_CANCEL 事件
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;

						// 14. 递归返回事件分发结果
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // 14. 检测是否是取消标志
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

		// 忽略：测试代码
        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
```
如果看完上述注释还有点蒙，一定要多撸几次源码。有几个点想说明一下，可能大家会好理解要一点。  

* `TouchTarget mFirstTouchTarget` 的作用  
`mFirstTouchTarget` 贯穿 `dispatchTouchEvent` 流程，实际上它是一个链表，用于记录所有接受 `Touch` 事件的 子`view` 。在日常开发中有没有遇到过这样一个逻辑“　如果一个 `view` 没有接收过 `ACTION_DOWN` 的事件，那么后续 `ACTION_MOVE` 和 `ACTION_UP` 也一定不会分发到这个 `view`。 ”。这个逻辑是基于　`mFirstTouchTarget`　记录的 `view`　实现的。

*总结*： 根据上述注释逻辑链。  
1. 过滤'不合法'的 `Touch` 事件；  
2. 如果是 MotionEvent.ACTION_DOWN ，则初始化一些状态；  
3. 判断事件是否需要拦截，是否需要取消；  
4. 如果不需要拦截＆不是取消事件，则会向子 `view` 下发 `Touch` 事件；  
5. 如果没有任何子 `view` 消费事件，则会自己处理，如果已有子 `view` 消费事件，判断当前新处理的 `target` 对象是否是 `mFirstTouchTarget` 链表最新一个，如果是则默认为当前传递已经传递事件，否则返回子 `view` 递归结果。

上述有两个递归，在 `注释10` 和 `注释14`，这里你可能会有疑惑，这两个的关系是什么。
`注释10` 实际上是返回以 viewGroup 为根节点的 view 下是否有节点消费点击事件，如果有则记录下当前子 view。
`注释14` 实际上是返回以 viewGroup 为根节点的 view 下是否有节点消费事件。　　


### 递归于 ViewGroup # dispatchTransformedTouchEvent
在上一章节，`dispatchTouchEvent`　多次调用　`dispatchTransformedTouchEvent`，这里做下简单分析。

```
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
		
		// 1. 是否下发 Touch 事件
        final boolean handled;

		// 2. 如果事件传递已经取消 或者当前事件是 ACTION_CANCEL
		// 如果没有子 view，则返回 view#dispatchTouchEvent 
		// 如果有子 view，则 返回 child#dispatchTouchEvent
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

   		// 3. 计算当前新手指id
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;
        if (newPointerIdBits == 0) {
            return false;
        }

		// 4. 处理事件转化
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // 5. 返回是否传递 Touch 事件
		// 如果没有子 view，则返回 view#dispatchTouchEvent 
		// 如果有子 view，则 返回 child#dispatchTouchEvent
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }
            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```
*总结*：`dispatchTransformedTouchEvent`　会对非 `MotionEvent.ACTION_CANCEL` 事件做转化并递归返回所有事件的下发结果。

### view # dispatchTouchEvent做了些什么？
既然不是 `view`，那么 `dispatchTouchEvent` 应该不是属于下发范畴的，那会是什么呢？

```
    public boolean dispatchTouchEvent(MotionEvent event) {
		
		//...

        boolean result = false

        //停止滑动
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }

		// 1.过滤不合法的 Touch 事件
        if (onFilterTouchEventForSecurity(event)) {

			// 2. 如果 view 可用且当前事件被当做拖拽事件处理，返回true
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
           
			// 3. 如果 view 可用监听且 OnTouchListener 处理了 touch 事件
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

			// 4. 如果 onTouchEvent 消费了事件
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

		//...

        return result;
    }
```
上述逻辑表明有三种场景下会返回 `true` 结果。  
两种场景为：第一种是拖拽场景，比如listview等控件存在这种逻辑；另一种是开发者设置了 OnTouchListener 对象并在 `onTouch` 函数中处理并返回 `true` 结果。  
最后一种场景为普遍场景，及如果没有上述两种场景且是当前是最外层 `view` 时（事件已经无法再传递），则会调用自身的 `onTouch` 方法处理。 
*总结*：`view#dispatchTouchEvent` 会在事件下发链末端调用，并把当前 `view` 的 `onTouch` 返回值作为 `dispatchTouchEvent` 返回值。
