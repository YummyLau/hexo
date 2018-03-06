---
title: Android View体系之基础常识及技巧
layout: post
date: 2018-03-05
comments: true
categories: Android
tags: [Android View体系] 
---

### 可视界面结构 
通过 `Android Device Monitor` - `Hierarchy View` 查看任意界面，都可看到以下布局层次。
也可以通过 `Tool` - `Android` - `LayoutInspector` 查看。

```
- DecorView
	- LinearLayout
		- ViewStub [id = action_mode_bar_stub]
		- FrameLayout
			- ActionBarOverlayLayout [id = decor_content_parent]
				- ContentFrameLayout [id = content]
					- <setContentView加载的布局>
				- ActionBarContainer [id = action_bar_containe]
					- Toolbar[id = action_bar]
						- AppCompatTextView
						- ActionMenuView
	- View [id = statusBarBackground]
```






    

                                