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

### 从 Activity 开始说起
从 `window` 层开始下发事件后， `Activity` 开始处理事件，会调用 `#dispatchTouchEvent`  

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

### 始于 dispatchTouchEvent
`DecorView` 调用 `dispatchTouchEvent` 分发 `Touch` 事件。代码很长，可是不难，逻辑比较清晰。

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {

		//返回值
        boolean handled = false;

		// 1. 检测是否分发Touch事件，主要是判断窗口是否被遮挡住了。
		// 如果被遮挡，则Touch事件无效不分发
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

			// 2. 如果当前是按下事件，则需要初始化一些状态
			// 2.1 清空所有接收触摸事件View的引用
			// 2.2 设置mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT，默认允许拦截事件
			// 2.3 设置mNestedScrollAxes = SCROLL_AXIS_NONE，默认视图不滚动
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

           	// 3. 检测拦截
			// 3.1 如果当前是点击事件且已经有视图接收该事件（可以理解为传递已经开始），则
			// 判断是否允许拦截，如果允许拦截，则调用onInterceptTouchEvent判断是否拦截，反之
			// 则不可拦截。
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
                intercepted = true;
            }

            // 4. 判断事件是否已经取消
			// 源码涉及到PFLAG_CANCEL_NEXT_UP_EVENT，该flag是View 脱离视图时被设置true
			// 也就是说，如果View detach时，则calceled返回true
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;

			// 5. 如果事件未取消且未拦截
            if (!canceled && !intercepted) {

                // 检测无障碍焦点
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

				// 5. 如果是单指某一点 或者 多指按下 或者 获取hover但是并没有点击（类似鼠标悬浮）
				// 获取actionIndex，任何点击都返回 0 
				// 获取点击id，单支id为0，第二手指id为1，依次递增
				// 删除已有的id对应的target
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    removePointersFromTouchTargets(idBitsToAssign);
	
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {

                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                 		
						// 6 扫描可传递的子 View 列表，按照Z轴坐标大小排序返回列表
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();

                        final View[] children = mChildren;
						
						// 7 遍历子 view ，匹配 6 中排序的 view
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // 8. 如果该 View 存在无障碍焦点，则跳过
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

							// 9. 如果View不可见且没有动画 或者 不在 View 的范围内，则跳过
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

							// 10. 遍历 target 链表，找到当前 child 对应的 target
							// 如果存在，则把手指的触摸 id 赋值 pointerIdBits 遍历
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

							// 11. 重置cancel标志并下发转化后的Touch事件给child
                            resetCancelNextUpFlag(child);

							// 12 
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();

								// 12. 找到最后点击的子 View 索引，并添加target到列表
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

					// ？？？
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

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
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

            // Update list of touch targets for pointer up or cancel, if needed.
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

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
```
如果看完上述注释还有点蒙，一定要多撸几次源码。有几个点想说明一下，可能大家会好理解要一点。

* `TouchTarget mFirstTouchTarget` 的作用  
`mFirstTouchTarget` 贯穿 `dispatchTouchEvent` 流程，实际上它是一个列表，用于记录所有接受 `Touch` 事件的 `view` 。在日常开发中有没有遇到过这样一个逻辑“　如果一个 `view` 没有接收过 `ACTION_DOWN` 的事件，那么后续 `ACTION_MOVE` 和 `ACTION_UP` 也一定不会分发到这个 `view`。这个逻辑是基于　`mFirstTouchTarget`　记录的 `view`　实现的。


```
    /** 
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

		// 1. 如果事件传递已经取消 或者当前事件是 ACTION_CANCEL
		// 如果有子 view，则让子 view 继续传递 该取消事件
		// 否则，调用自身的 dispatchTouchEvent
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

        // 2. 获取触摸手指ids
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        if (newPointerIdBits == 0) {
            return false;
        }

		// 3. 处理转化事件
		// 如果新旧手势的 ids 相同，则不需要重新计算 event，区分 view 和 viewgroup
		// 其中 view  直接调用 dispatchTouchEvent，而 viewGroup则调用 child#dispatchTouchEvent，
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

        // Perform any necessary transformations and dispatch.
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


