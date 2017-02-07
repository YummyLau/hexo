---
title: 读《打造高质量Android应用》之后记
date: 2017-02-06 14:13:29
comments: true
categories: Android
tags: [笔记]
---
<!--more-->

> 经常在开发过程中学习到一些有用高效的Tips，这些Tips能帮助我们更好的构建我们的应用。《打造高质量Android应用》中记录了50个诀窍，在此记录下自己陌生或或易忘Tips，供以后开发查询警惕。

# 布局相关

* 对于使用Weight属性的View控件而言，与其ParentView及Width(height同理)满足一下公式:
```
View's real Width = View's Width + View's Weight * ParentVie's Width / WeightSum
```

* include标签的注意点
	* **include** 标签中覆盖声明 android:layout\_width 和 android:layout\_height 属性会覆盖被引入布局中任何 android:layout\_\*属性。
	* 由于被引入的布局无法预先知道其父布局的类型（线性？相对？），所以建议把所有 android:layout\_\* 属性写在标签中,而引入的布局中 android:layout\_ width 和 android:layout\_height 都指定为0dp。

	
* 不要试图在OnCreat方法中获取View的宽高，不过可通过post方法获取
```
	view.post(new Runnable() {
		@Override
		public void run() {
		  //获取view的宽高
		}
	});
```

# 工具相关

* 实用ProGuard来移除日志语句，具体可参考官网[压缩代码与资源](https://developer.android.com/studio/build/shrink-code.html?hl=zh-cn#shrink-code)。

# 可复用性

* 同时可发起多个Intent操作，例如：
```
	Intent pickIntent = new Intent(Intent.ACTION_GET_CONTENT);
	pickIntent.setType("image/*");
	Intent takePhotoIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);

	Intent chooserIntent = Intent.createChooser(pickIntent,"选图or拍照");
	chooserIntent.putExtra(Intent.EXTRA_INITIAL_INTENTS, new Intent[]{takePhotoIntent});
	startActivityForResult(chooserIntent,0);
```

* 对于兼容旧API的处理方法，可参考v4包中SharedPreferencesCompat或ContextCompat等兼容类来处理。

# 心得感悟
整体过了两遍，里面部分Tips随着Android Api及开源技术的更新已经有了新的方法实现。不过从开发者角度（我自己）而言，可以学习到如何处理项目细节，补上知识遗漏点及强化一些之前未留意的知识点，学习别人的代码思维及项目大局观。




	



