
q	日常开发中我们可能需要对在textview或者edittext中添加表情(图片)，超链接等信息，则可借助`SpannableString`进行实现。  
基本的实现可参考官网Api或网上各种博文实现，这篇文章只针对一个“bug”场景进行深入分析，案例及分析如下：
>当我们在开发游戏助手动态模块的时候，需要对用户的每一条动态信息（类朋友圈）进行展示，展示的内容可能包括：文本、表情、 超链接、图片等信息且可点击，同时产品还需要实现文本可动态折叠（类全文/收起）。某些游戏助手还需要根据不同的页面（如动态列表页面，动态详情页面）区分动态显示的长度，而这种情况，一般我们会复用同一个view来实现。    
由于我们需要实现显示富文本，所以采用`TextView`实现；加之需要实现折叠或者动态控制不同页面显示不同的内容长度，故我们需要动态控制`TextView`的最大行数，即`maxlines`。  

上述的实现使用Recyclerview显示仅有一个TextView的holder作为演示，抽象出项目中我们实现的公用部分，就是holder中装着一个拥有可点击的富文本且文本内容移除显示区域的TextView。  

```
public class ExampleHolder extends RecyclerView.ViewHolder {

    private HolderExampleBinding mBinding;
    private Context mContext;

    public ExampleHolder(HolderExampleBinding binding) {
        super(binding.getRoot());
        this.mBinding = binding;
        this.mContext = binding.getRoot().getContext();
    }

    public void bindData(int position) {
        if (position % 2 == 0) {
            mBinding.textview.setMaxLines(3);
        } else {
            mBinding.textview.setMaxLines(Integer.MAX_VALUE);
        }

        String content = "北国风光，千里冰封，万里雪飘。\n" +
                "望长城内外，惟余莽莽；大河上下，顿失滔滔。\n" +
                "山舞银蛇，原驰蜡象，欲与天公试比高。\n" +
                "须晴日，看红装素裹，分外妖娆。\n" +
                "江山如此多娇，引无数英雄竞折腰。\n" +
                "惜秦皇汉武，略输文采；唐宗宋祖，稍逊风骚。\n" +
                "一代天骄，成吉思汗，只识弯弓射大雕。\n" +
                "俱往矣，数风流人物，还看今朝。";

        mBinding.textview.setEllipsize(TextUtils.TruncateAt.END);
        SpannableString spannableString = new SpannableString(content);					

        String strings1 = "北国风光";
        int start1 = content.indexOf(strings1);
        int end1 = start1 + strings1.length();
        spannableString.setSpan(new ExampleSpan(mContext),
                start1, end1, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);

        String strings2 = "惟余莽莽";
        int start2 = content.indexOf(strings2);
        int end2 = start2 + strings2.length();
        spannableString.setSpan(new ExampleSpan(mContext),
                start2, end2, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);

        String strings3 = "欲与天公试比高";
        int start3 = content.indexOf(strings3);
        int end3 = start3 + strings3.length();
        spannableString.setSpan(new ExampleSpan(mContext),							   // 1处
                start3, end3, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);

        mBinding.textview.setText(spannableString);
        mBinding.textview.setMovementMethod(LinkMovementMethod.getInstance());		   // 2处
    }
}
```
代码的逻辑非常简单，奇数项时`TextView`显示3行，偶数项时`TextView`显示全部内容。传统的纯文本实现与bug版本区别在于1/2处，那我们就从这两处入手，从源码的角度来解决问题。

`SpannableString`,谷歌官网的描述为：
>This is the class for text whose content is immutable but to which markup objects can be attached and detached.  
>即这是一个对文本进行附加或移除"标记对象"的类。  

对于`setSpan`的调用链，有：

```
/** Form SpannableString Class **/
    public void setSpan(Object what, int start, int end, int flags) {
        super.setSpan(what, start, end, flags);
    }

/** Form SpannableStringInternal Class **/
    void setSpan(Object what, int start, int end, int flags) {
        int nstart = start;
        int nend = end;

		//1. 检查是否越界
        checkRange("setSpan", start, end);

		//2. 检测Spannable.SPAN_PARAGRAPH的段落格式
        if ((flags & Spannable.SPAN_PARAGRAPH) == Spannable.SPAN_PARAGRAPH) {
            if (start != 0 && start != length()) {
                char c = charAt(start - 1);

                if (c != '\n')
                    throw new RuntimeException(
                            "PARAGRAPH span must start at paragraph boundary" +
                            " (" + start + " follows " + c + ")");
            }

            if (end != 0 && end != length()) {
                char c = charAt(end - 1);

                if (c != '\n')
                    throw new RuntimeException(
                            "PARAGRAPH span must end at paragraph boundary" +
                            " (" + end + " follows " + c + ")");
            }
        }

        int count = mSpanCount;
        Object[] spans = mSpans;
        int[] data = mSpanData;

		// 3. 如果当前spans数组存在传入的span对象，则覆盖其之前的设置并改变样式
        for (int i = 0; i < count; i++) {
            if (spans[i] == what) {
                int ostart = data[i * COLUMNS + START];
                int oend = data[i * COLUMNS + END];

                data[i * COLUMNS + START] = start;
                data[i * COLUMNS + END] = end;
                data[i * COLUMNS + FLAGS] = flags;

                sendSpanChanged(what, ostart, oend, nstart, nend);
                return;
            }
        }

		//4. 如果当前mSpans为初始化状态，元素个数为0。
		// 如果有新的Span添加且mSpans空间不足，则会申请存放空间，规则见ArrayUtils和GrouwingArrayUtils
		// 之后把原mSpans和mSpanData数据复制到新的数据结构中并把其引用指向mSpans和mSpanData
        if (mSpanCount + 1 >= mSpans.length) {
            Object[] newtags = ArrayUtils.newUnpaddedObjectArray(
                    GrowingArrayUtils.growSize(mSpanCount));
            int[] newdata = new int[newtags.length * 3];

            System.arraycopy(mSpans, 0, newtags, 0, mSpanCount);
            System.arraycopy(mSpanData, 0, newdata, 0, mSpanCount * 3);

            mSpans = newtags;
            mSpanData = newdata;
        }

		//5. 保存span对象及stat，end及flags
        mSpans[mSpanCount] = what;
        mSpanData[mSpanCount * COLUMNS + START] = start;
        mSpanData[mSpanCount * COLUMNS + END] = end;
        mSpanData[mSpanCount * COLUMNS + FLAGS] = flags;
        mSpanCount++;

		//6. 添加span样式
        if (this instanceof Spannable)
            sendSpanAdded(what, nstart, nend);
    }

/** From ArrayUtils Class **/
   public static Object[] newUnpaddedObjectArray(int minLen) {
        return (Object[])VMRuntime.getRuntime().newUnpaddedArray(Object.class, minLen);
   }

/** From VMRuntime Class **/
  /**
     * Returns an array of at least minLength, but potentially larger. The increased size comes from
     * avoiding any padding after the array. The amount of padding varies depending on the
     * componentType and the memory allocator implementation.
     */
	public native Object newUnpaddedArray(Class<?> componentType, int minLength);


/** From GrowingArrayUtils Class **/
    public static int growSize(int currentSize) {
        return currentSize <= 4 ? 8 : currentSize * 2;
    }

```
上述6处标记可知，如果当在设置的span对象存在，则覆盖原有的span对象；否则，则添加新的对象进`mSpans`数组。当`mSpans`数组不够用时，便会申请空间扩容存储。
上述有两个函数`sendSpanChanged`和`sendSpanAdded`,继续看一下。  

```
/** Form SpannableStringInternal Class **/
    private void sendSpanChanged(Object what, int s, int e, int st, int en) {
        SpanWatcher[] recip = getSpans(Math.min(s, st), Math.max(e, en),
                                       SpanWatcher.class);
        int n = recip.length;

        for (int i = 0; i < n; i++) {
            recip[i].onSpanChanged((Spannable) this, what, s, e, st, en);
        }
    }

    private void sendSpanAdded(Object what, int start, int end) {
        SpanWatcher[] recip = getSpans(start, end, SpanWatcher.class);
        int n = recip.length;

        for (int i = 0; i < n; i++) {
            recip[i].onSpanAdded((Spannable) this, what, start, end);
        }
    }

/** Form SpanWatcher interface**/
public interface SpanWatcher extends NoCopySpan {
    public void onSpanAdded(Spannable text, Object what, int start, int end);
    public void onSpanRemoved(Spannable text, Object what, int start, int end); 
    public void onSpanChanged(Spannable text, Object what, int ostart, int oend,
                              int nstart, int nend);
}
```
无论是`setSpanChanged`还是`setSpanAdded`,其内部都是遍历区间之后拿到SpanWatcher对象，而这些对象和TextView有什么关联呢？[TextView源码](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/widget/TextView.java?av=h#TextView.ChangeWatcher)可看到，`TextView`有一个实现了SpanWatcher接口的`ChangeWatcher`内部类，并申明了`mChangeWatcher`变量。至此，最开始的1处中我们基础都分析了`spannableString`#`setSpan`方法，大概逻辑清楚了：我们`spannableString`设置了span对象之后，`spannableString`内部通过`SpannableStringInternal`,并通过`SpanWatcher`响应变化，而如何变化，则通过`spannableString`#`setSpan`的第一个参数`Object`对象`what`。这个对象最终被传递`TextView`的`ChangeWatcher`实例中处理,看看`ChangeWatcher`是如何处理的。 

```
/** Form ChangeWatcher Class **/
        public void onSpanChanged(Spannable buf, Object what, int s, int e, int st, int en) {
            if (DEBUG_EXTRACT) Log.v(LOG_TAG, "onSpanChanged s=" + s + " e=" + e
                    + " st=" + st + " en=" + en + " what=" + what + ": " + buf);
            TextView.this.spanChange(buf, what, s, st, e, en);
        }

        public void onSpanAdded(Spannable buf, Object what, int s, int e) {
            if (DEBUG_EXTRACT) Log.v(LOG_TAG, "onSpanAdded s=" + s + " e=" + e
                    + " what=" + what + ": " + buf);
            TextView.this.spanChange(buf, what, -1, s, -1, e);
        }

        public void onSpanRemoved(Spannable buf, Object what, int s, int e) {
            if (DEBUG_EXTRACT) Log.v(LOG_TAG, "onSpanRemoved s=" + s + " e=" + e
                    + " what=" + what + ": " + buf);
            TextView.this.spanChange(buf, what, s, -1, e, -1);
        }

/** Form TextView  Class**/
		
	//1.对于添加span，其oldStart和oldEnd都传-1
	//  对于删除span，其newStart和newEnd都传-1
	//  这比较好理解，删除和添加也是一种变化。
    void spanChange(Spanned buf, Object what, int oldStart, int newStart, int oldEnd, int newEnd) {
        // XXX Make the start and end move together if this ends up
        // spending too much time invalidating.

        boolean selChanged = false;
        int newSelStart=-1, newSelEnd=-1;

        final Editor.InputMethodState ims = mEditor == null ? null : mEditor.mInputMethodState;

		// 2. Selection.SELECTION_END和Selection.SELECTION_START否是NoCopySpan实现类
		// 如果满足，则更新光标及大小
        if (what == Selection.SELECTION_END) {
            selChanged = true;
            newSelEnd = newStart;

            if (oldStart >= 0 || newStart >= 0) {
                invalidateCursor(Selection.getSelectionStart(buf), oldStart, newStart);
                checkForResize();
                registerForPreDraw();
                if (mEditor != null) mEditor.makeBlink();
            }
        }

        if (what == Selection.SELECTION_START) {
            selChanged = true;
            newSelStart = newStart;

            if (oldStart >= 0 || newStart >= 0) {
                int end = Selection.getSelectionEnd(buf);
                invalidateCursor(end, oldStart, newStart);
            }
        }

		//3. 如果满足2，则更新光标并更新视图标识，这里不太理解
        if (selChanged) {
            mHighlightPathBogus = true;
            if (mEditor != null && !isFocused()) mEditor.mSelectionMoved = true;

            if ((buf.getSpanFlags(what)&Spanned.SPAN_INTERMEDIATE) == 0) {
                if (newSelStart < 0) {
                    newSelStart = Selection.getSelectionStart(buf);
                }
                if (newSelEnd < 0) {
                    newSelEnd = Selection.getSelectionEnd(buf);
                }

                if (mEditor != null) {
                    mEditor.refreshTextActionMode();
                    if (!hasSelection() && mEditor.mTextActionMode == null && hasTransientState()) {
                        // User generated selection has been removed.
                        setHasTransientState(false);
                    }
                }
                onSelectionChanged(newSelStart, newSelEnd);
            }
        }
		
		//4. 如果span对象是UpdateAppearance /ParagraphStyle /CharacterStyle子类
		// 则根据mEditor的值得情况更新文本显示
        if (what instanceof UpdateAppearance || what instanceof ParagraphStyle ||
                what instanceof CharacterStyle) {
            if (ims == null || ims.mBatchEditNesting == 0) {
                invalidate();
                mHighlightPathBogus = true;
                checkForResize();
            } else {
                ims.mContentChanged = true;
            }
            if (mEditor != null) {
                if (oldStart >= 0) mEditor.invalidateTextDisplayList(mLayout, oldStart, oldEnd);
                if (newStart >= 0) mEditor.invalidateTextDisplayList(mLayout, newStart, newEnd);
                mEditor.invalidateHandlesAndActionMode();
            }
        }
	
		//5. 主要是来追踪Shift/Alt等key行为
        if (MetaKeyKeyListener.isMetaTracker(buf, what)) {
            mHighlightPathBogus = true;
            if (ims != null && MetaKeyKeyListener.isSelectingMetaTracker(buf, what)) {
                ims.mSelectionModeChanged = true;
            }

            if (Selection.getSelectionStart(buf) >= 0) {
                if (ims == null || ims.mBatchEditNesting == 0) {
                    invalidateCursor();
                } else {
                    ims.mCursorChanged = true;
                }
            }
        }

		//6. 是否是可跨域传递的span对象
        if (what instanceof ParcelableSpan) {
            // If this is a span that can be sent to a remote process,
            // the current extract editor would be interested in it.
            if (ims != null && ims.mExtractedTextRequest != null) {
                if (ims.mBatchEditNesting != 0) {
                    if (oldStart >= 0) {
                        if (ims.mChangedStart > oldStart) {
                            ims.mChangedStart = oldStart;
                        }
                        if (ims.mChangedStart > oldEnd) {
                            ims.mChangedStart = oldEnd;
                        }
                    }
                    if (newStart >= 0) {
                        if (ims.mChangedStart > newStart) {
                            ims.mChangedStart = newStart;
                        }
                        if (ims.mChangedStart > newEnd) {
                            ims.mChangedStart = newEnd;
                        }
                    }
                } else {
                    if (DEBUG_EXTRACT) Log.v(LOG_TAG, "Span change outside of batch: "
                            + oldStart + "-" + oldEnd + ","
                            + newStart + "-" + newEnd + " " + what);
                    ims.mContentChanged = true;
                }
            }
        }
        if (mEditor != null && mEditor.mSpellChecker != null && newStart < 0 &&
                what instanceof SpellCheckSpan) {
            mEditor.mSpellChecker.onSpellCheckSpanRemoved((SpellCheckSpan) what);
        }
    }
```
到这里，我们并没有发现任何问题。可好奇的，为什么我们点击第三行的span之后，内容会偏移呢？我们继续深入看我们传递的span对象`Object`实例what是否存在猫腻？  

```
/** From ExampleSpan Class **/
public class ExampleSpan extends ClickableSpan {

    private Context mContext;

    public ExampleSpan(Context context) {
        this.mContext = context;
    }

    @Override
    public void onClick(View view) {
        Toast.makeText(mContext, "点击了span", Toast.LENGTH_SHORT).show();
    }

    public void updateDrawState(TextPaint ds) {
        ds.setColor(ContextCompat.getColor(mContext, R.color.colorAccent));
        ds.setUnderlineText(false);
    }
}
```
上诉类是我项目中实现的ClickableSpan，除了去除超链接样式显示和更换文本颜色之后，并无其他问题，但是存在点击事件，难道
和我们bug场景下的点击难道有一定的关系？说到点击事件，最常用的便是Click事件，而针对`TextView`，官方提供了一个策略接口专门处理`TextView`的key Event，其实现是由框架层`framework`提供，开发者无法在实际项目中实现使用。看看官方提供给开发者的一个子类实现。 

```
/** From LinkMovementMethod Class **/
    @Override
    public boolean onTouchEvent(TextView widget, Spannable buffer,
                                MotionEvent event) {
        int action = event.getAction();

		//1. 处理UP/DOWM事件
        if (action == MotionEvent.ACTION_UP ||
            action == MotionEvent.ACTION_DOWN) {
            int x = (int) event.getX();
            int y = (int) event.getY();

            x -= widget.getTotalPaddingLeft();
            y -= widget.getTotalPaddingTop();

            x += widget.getScrollX();
            y += widget.getScrollY();

            Layout layout = widget.getLayout();
            int line = layout.getLineForVertical(y);
            int off = layout.getOffsetForHorizontal(line, x);
			
			//2. 获取点击处的ClickableSpan对象
            ClickableSpan[] link = buffer.getSpans(off, off, ClickableSpan.class);
			
			//3. 如果点击处存在ClickableSpan对象
            if (link.length != 0) {
                if (action == MotionEvent.ACTION_UP) {
					//4. 回调onClick事件
                    link[0].onClick(widget);
                } else if (action == MotionEvent.ACTION_DOWN) {
					//5. 重新设置光标
                    Selection.setSelection(buffer,
                                           buffer.getSpanStart(link[0]),
                                           buffer.getSpanEnd(link[0]));
                }

                return true;
            } else {
                Selection.removeSelection(buffer);
            }
        }
		//由父类处理
        return super.onTouchEvent(widget, buffer, event);
    }

/** From ScrollingMovementMethod Class (LinkMovementMethod父类)**/
    @Override
    public boolean onTouchEvent(TextView widget, Spannable buffer, MotionEvent event) {
        return Touch.onTouchEvent(widget, buffer, event);
	}
```
在我们点击`TextView`时会尝试去获取是否有`SpanableClick`对象，如果有，则先调用5处，在调用4处。  
值得注意的是，TextView的模式实现是内容不可滚动，一旦我们调用了`setMovementMethod`方法，其最终会回调到`Touch`类中实现，默认打开滑动。如何知道这个规律呢？看代码。  

```
/** From Touch Class **/
    public static boolean onTouchEvent(TextView widget, Spannable buffer,
                                       MotionEvent event) {
        DragState[] ds;

        switch (event.getActionMasked()) {
        case MotionEvent.ACTION_DOWN:
            ds = buffer.getSpans(0, buffer.length(), DragState.class);

			//1. 删除久的DragState span对象
            for (int i = 0; i < ds.length; i++) {
                buffer.removeSpan(ds[i]);
            }

			//2. 重新标记DragState span对象
            buffer.setSpan(new DragState(event.getX(), event.getY(),
                            widget.getScrollX(), widget.getScrollY()),
                    0, 0, Spannable.SPAN_MARK_MARK);
            return true;

        case MotionEvent.ACTION_UP:
            ds = buffer.getSpans(0, buffer.length(), DragState.class);

			//3. 删除久的DragState span对象
            for (int i = 0; i < ds.length; i++) {
                buffer.removeSpan(ds[i]);
            }

            if (ds.length > 0 && ds[0].mUsed) {
                return true;
            } else {
                return false;
            }

        case MotionEvent.ACTION_MOVE:
			//4. 获取DOWM时设置的span对象
            ds = buffer.getSpans(0, buffer.length(), DragState.class);

            if (ds.length > 0) {
                if (ds[0].mFarEnough == false) {
                    int slop = ViewConfiguration.get(widget.getContext()).getScaledTouchSlop();

                    if (Math.abs(event.getX() - ds[0].mX) >= slop ||
                        Math.abs(event.getY() - ds[0].mY) >= slop) {
                        ds[0].mFarEnough = true;
                    }
                }

                if (ds[0].mFarEnough) {
                    ds[0].mUsed = true;
                    boolean cap = (event.getMetaState() & KeyEvent.META_SHIFT_ON) != 0
                            || MetaKeyKeyListener.getMetaState(buffer,
                                    MetaKeyKeyListener.META_SHIFT_ON) == 1
                            || MetaKeyKeyListener.getMetaState(buffer,
                                    MetaKeyKeyListener.META_SELECTING) != 0;

                    float dx;
                    float dy;
                    if (cap) {
                        // if we're selecting, we want the scroll to go in
                        // the direction of the drag
                        dx = event.getX() - ds[0].mX;
                        dy = event.getY() - ds[0].mY;
                    } else {
                        dx = ds[0].mX - event.getX();
                        dy = ds[0].mY - event.getY();
                    }

					//5. 设置当前位置
                    ds[0].mX = event.getX();
                    ds[0].mY = event.getY();

					//6. 获取偏移后的位置
                    int nx = widget.getScrollX() + (int) dx;
                    int ny = widget.getScrollY() + (int) dy;

					//7. 获取上下内边距
                    int padding = widget.getTotalPaddingTop() + widget.getTotalPaddingBottom();
                    Layout layout = widget.getLayout();

                    ny = Math.min(ny, layout.getHeight() - (widget.getHeight() - padding));
                    ny = Math.max(ny, 0);

                    int oldX = widget.getScrollX();
                    int oldY = widget.getScrollY();
					
					//8.滑动到指定位置
                    scrollTo(widget, layout, nx, ny);

                    //9. 滑动中，需要取消长按事件
                    if (oldX != widget.getScrollX() || oldY != widget.getScrollY()) {
                        widget.cancelLongPress();
                    }

                    return true;
                }
            }
        }

        return false;
    }
```

从DOWM事件开始设置span，到MOVE事件中调用`scrollTo`,最后在UP事件中能够删除span对象。这一系列的span标识提供了足够的信息让系统去操作我们的`TextView`内容滑动。  
到这里，我们可以得出结论：如果不想`TextView`文本被移动，就在`Touch`上层消费掉`MotionEvent.ACTION_DOWN`事件。这里对于处理非滑动列表显示`TextView`的场景居多，比如你的`TextView`存在可点击的富文本但是又不需要显示全部，但又不想让它滑动，那么就可以直接在自己实现`LinkMovementMethod `子类并且在`onTouch`处理`MotionEvent.ACTION_DOWN`和`MotionEvent.ACTION_UP`事件并直接return`true`。  
说了那么多，还是没有解决最开始的bug啊，我们的bug是点击之后出来的，又不是滑动 =。=，别急，看源码也是一个学习的过程，学习到的不仅仅是解决问题，还有很多延伸的知识点。回到正题，根据上面我的分析，我把`LinkMovementMethod `#`onTouch`进行修改，代码为：   

```
/** From Example code **/
    @Override
    public boolean onTouchEvent(TextView widget, Spannable buffer, MotionEvent event) {
        int action = event.getAction();

        if (action == MotionEvent.ACTION_UP ||
                action == MotionEvent.ACTION_DOWN) {
            int x = (int) event.getX();
            int y = (int) event.getY();

            x -= widget.getTotalPaddingLeft();
            y -= widget.getTotalPaddingTop();

            x += widget.getScrollX();
            y += widget.getScrollY();

            Layout layout = widget.getLayout();
            int line = layout.getLineForVertical(y);
            int off = layout.getOffsetForHorizontal(line, x);

            ClickableSpan[] link = buffer.getSpans(off, off, ClickableSpan.class);

            if (link.length != 0) {
                if (action == MotionEvent.ACTION_UP) {
					//1. 回调点击
                    link[0].onClick(widget);
                } else if (action == MotionEvent.ACTION_DOWN) {
					//2. 设置光标
                    Selection.setSelection(buffer,
                            buffer.getSpanStart(link[0]),
                            buffer.getSpanEnd(link[0]));
                }
            } else {
                Selection.removeSelection(buffer);
            }
        //调整返回值    
            return true;
        }else{
            return super.onTouchEvent(widget, buffer, event);
        }
    }
```
运行之后内容确实不能滑动了。而我们点击之后出现的bug，无非是由12处导致引起的，而1只是回调listener处理一些业务，所以基本可以锁定导致bug的代码便是2处。来看看2处到底做了些什么？  

```
/** From Selection Class**/
    /**
     * Set the selection anchor to <code>start</code> and the selection edge
     * to <code>stop</code>.
     */
    public static void setSelection(Spannable text, int start, int stop) {
        // int len = text.length();
        // start = pin(start, 0, len);  XXX remove unless we really need it
        // stop = pin(stop, 0, len);

        int ostart = getSelectionStart(text);
        int oend = getSelectionEnd(text);
		
		//1. 设置前后光标锁定span区间
        if (ostart != start || oend != stop) {
            text.setSpan(SELECTION_START, start, start,
                         Spanned.SPAN_POINT_POINT|Spanned.SPAN_INTERMEDIATE);
            text.setSpan(SELECTION_END, stop, stop,
                         Spanned.SPAN_POINT_POINT);
        }
    }
```
由于`text`实例实际上就是我们项目中`SpanableString`对象，这里调用`setSpan`方法的过程和上面的过程是一模一样的。但是设置了区别在于我们设置span对象为`ClickableSpan`实例，而这里是`SELECTION_START`和`SELECTION_END `。而实际上这两个表示在`TextView `#`spanChange`中我们便看到过。

```
/** Form TextView  Class**/
    void spanChange(Spanned buf, Object what, int oldStart, int newStart, int oldEnd, int newEnd) {

		//省略部分代码

        if (what == Selection.SELECTION_END) {
            selChanged = true;
            newSelEnd = newStart;

            if (oldStart >= 0 || newStart >= 0) {
                invalidateCursor(Selection.getSelectionStart(buf), oldStart, newStart);
                checkForResize();
                registerForPreDraw();
                if (mEditor != null) mEditor.makeBlink();
            }
        }

        if (what == Selection.SELECTION_START) {
            selChanged = true;
            newSelStart = newStart;

            if (oldStart >= 0 || newStart >= 0) {
                int end = Selection.getSelectionEnd(buf);
                invalidateCursor(end, oldStart, newStart);
            }
        }


		//省略部分代码
}

```