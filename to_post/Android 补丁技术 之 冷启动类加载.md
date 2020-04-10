---
title: Android 补丁技术 之 冷启动类加载
date: 2020-04-09 05:45:34
comments: true
categories: Android
tags: [热修复]
---

所谓 **冷启动类加载**，即应用进行重启之后，加载修复后的类文件达到修复类的已知问题。

如果一个类已经被虚拟机所加载，要修正该类的某些方法，只能通过实现 **热替换** 来实现："在 **navite** 层替换到对应被虚拟机加载过的类的方法"。在业界还有其他冷启动的方案，比如 *tinker*，让 *Classloader* 去加载新的类，而原来的类还在虚拟机中，不重启应用是无法加载新的类的。

以阿里 **Andfix** [开源项目](https://github.com/alibaba/AndFix) 及 **Sophix** 方案为分析。

1. **[AndFix#replaceMethod(Method src,Method dest)](https://github.com/alibaba/AndFix/blob/master/src/com/alipay/euler/andfix/AndFix.java)** 为 Java 层替换错误方法的入口，通过 JNI 调用 Navite 层代码
2. **[andifx#replaceMethod](https://github.com/alibaba/AndFix/blob/master/jni/andfix.cpp)** 为 Navite 层被上层所调用的代码,对虚拟机内的方法进行 ”替换“

	```
	static void replaceMethod(JNIEnv* env, jclass clazz, jobject src,jobject dest) {
		if (isArt) {
			art_replaceMethod(env, src, dest);
		} else {
			dalvik_replaceMethod(env, src, dest);
		}
}
	```
	代码区分了 Dalvik 虚拟机和 Art 虚拟机的不同实现
	
	**[art_method_replace#art_replaceMethod](https://github.com/alibaba/AndFix/blob/master/jni/art/art_method_replace.cpp)** 实现 Art 虚拟机上的替换
	
	```
extern void __attribute__ ((visibility ("hidden"))) art_replaceMethod(JNIEnv* env, jobject src, jobject dest) {
	    if (apilevel > 23) {
	        replace_7_0(env, src, dest);
	    } else if (apilevel > 22) {
			replace_6_0(env, src, dest);
		} else if (apilevel > 21) {
			replace_5_1(env, src, dest);
		} else if (apilevel > 19) {
			replace_5_0(env, src, dest);
	    }else{
	        replace_4_4(env, src, dest);
	    }
}
	```
	
	不同的虚拟机版本，由于虚拟机底层数据结构并不相同，所以还进一步针对不同 Android 版本再做区分
	
	* **[art_method_replace_4_4#replace_4_4](https://github.com/alibaba/AndFix/blob/master/jni/art/art_method_replace_4_4.cpp)**
	* **[art_method_replace_5_0#replace_5_0](https://github.com/alibaba/AndFix/blob/master/jni/art/art_method_replace_5_0.cpp)**
	* **[art_method_replace_5_1#replace_5_1](https://github.com/alibaba/AndFix/blob/master/jni/art/art_method_replace_5_1.cpp)**
	* **[art_method_replace_6_0#replace_6_0](https://github.com/alibaba/AndFix/blob/master/jni/art/art_method_replace_6_0.cpp)**
	* **[art_method_replace_7_0#replace_7_0](https://github.com/alibaba/AndFix/blob/master/jni/art/art_method_replace_7_0.cpp)**
	
3. 以 6.0 版本的 Art虚拟机替换为例子

	```
	void replace_6_0(JNIEnv* env, jobject src, jobject dest) {
	
		//获取 被替换 Method 对象对应 ArtMethod 的地址
		art::mirror::ArtMethod* smeth =
				(art::mirror::ArtMethod*) env->FromReflectedMethod(src);
				
		//获取 替换 Method 对象对应 ArtMethod 的地址
		art::mirror::ArtMethod* dmeth =
				(art::mirror::ArtMethod*) env->FromReflectedMethod(dest);
	
		//确保 Classloader 一致
	    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->class_loader_ =
	    reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->class_loader_; //for plugin classloader
	    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->clinit_thread_id_ =
	    reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->clinit_thread_id_;
	    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->status_ = reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->status_-1;
	    //for reflection invoke
	    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->super_class_ = 0;
	
		 // 旧的函数所有成员变量都需要替换成新的函数的成员变量
	    smeth->declaring_class_ = dmeth->declaring_class_;
	    smeth->dex_cache_resolved_methods_ = dmeth->dex_cache_resolved_methods_;
	    smeth->dex_cache_resolved_types_ = dmeth->dex_cache_resolved_types_;
	    smeth->access_flags_ = dmeth->access_flags_ | 0x0001;
	    smeth->dex_code_item_offset_ = dmeth->dex_code_item_offset_;
	    smeth->dex_method_index_ = dmeth->dex_method_index_;
	    smeth->method_index_ = dmeth->method_index_;
	    
	    smeth->ptr_sized_fields_.entry_point_from_interpreter_ =
	    dmeth->ptr_sized_fields_.entry_point_from_interpreter_;
	    
	    smeth->ptr_sized_fields_.entry_point_from_jni_ =
	    dmeth->ptr_sized_fields_.entry_point_from_jni_;
	    smeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_ =
	    dmeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_;
	    
	    LOGD("replace_6_0: %d , %d",
	         smeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_,
	         dmeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_);
}
	```
	每一个 Java 方法在 Art 虚拟机内都对应一个 **[art_method](https://android.googlesource.com/platform/art/+/refs/heads/nougat-release/runtime/art_method.h)**, 用于记录 Java 方法的所有信息，包括归属类，访问权限，代码执行地址等等。替换完之后，再次调用替换方法的时候就会直接运行到新方法的实现。
	
	Java Code 会被编译成 Dex Code ，然后被 Art 虚拟机加载，可通过解释模式或者 AOT模式执行.但都先需要获取方法的执行入口:
	
	* 解释模式，获取 [art_method.entry_point_from_jni_](https://android.googlesource.com/platform/art/+/refs/heads/nougat-release/runtime/art_method.h)
	* AOT模式模式，获取 [art_method.entry_point_from_jni_](https://android.googlesource.com/platform/art/+/refs/heads/nougat-release/runtime/art_method.h)
	
	要实现方法替换,除了替换这几个指针入口地址. 
	
	上述的例子需要保证 **[art_method_replace_6_0#replace_6_0](https://github.com/alibaba/AndFix/blob/master/jni/art/art_method_replace_6_0.cpp)** 所用的数据结构与 **[art_method](https://android.googlesource.com/platform/art/+/refs/heads/nougat-release/runtime/art_method.h)**	对应的数据结构完全一致才可以.但是由于各种厂商存在各式各样经过改造的 ROM,难以保证能够修复成功.

4. **Sophix** 探索出了一种突破底层结构差异的方法。这种方法把一个 **[art_method](https://android.googlesource.com/platform/art/+/refs/heads/nougat-release/runtime/art_method.h)** 看成了一个整体进行替换而不必针对每个版本的 method 严格控制内容。**换句话说，只要知道当前设备 *art_method* 的长度，就可以把整个结构体完全替换掉。**
	
	由于 ArtMethod 是紧密排列的，所以相邻两个 ArtMethod 的起始地址差值就是 ArtMethod 的大小。通过定义一个简单的类来计算。
	
	```
	public class NativeMethodCal{
		final public static void f1(){}
		final public static void f2(){}
	}
	```
	两个方法属于 static 方法 且该类只有这两个方法，所以必定相邻,计算如下
	
	```
	size_t firMid = (size_t) env->GetStaticMethodID(nativeMethodCalClazz,"f1","()V");
	size_t secMid = (size_t) env->GetStaticMethodID(nativeMethodCalClazz,"f2","()V");
	size_t methodSize = secMid - firMid
	```
	navice 层的替换可为
	
	```
	void replacee(JNIEnv* env, jobject src, jobject dest) {
	
		art::mirror::ArtMethod* smeth =
				(art::mirror::ArtMethod*) env->FromReflectedMethod(src);
				
		art::mirror::ArtMethod* dmeth =
				(art::mirror::ArtMethod*) env->FromReflectedMethod(dest);
				
		//确保 Classloader 一致
	    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->class_loader_ =
	    reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->class_loader_; //for plugin classloader
	    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->clinit_thread_id_ =
	    reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->clinit_thread_id_;
	    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->status_ = reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->status_-1;
	    //for reflection invoke
	    reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->super_class_ = 0;
				
		size_t firMid = (size_t) env->GetStaticMethodID(nativeMethodCalClazz,"f1","()V");
		size_t secMid = (size_t) env->GetStaticMethodID(nativeMethodCalClazz,"f2","()V");
		size_t methodSize = secMid - firMid
		memcpy(smeth,dmeth, methodSize);
	}
	```

**存在的问题与限制**

1. 针对反射调用非静态方法产生的问题。这类问题只能通过冷启动修复，原因是反射调用的 `invoke` 底层回调用到 `InvokeMethod`,该方法会校验反射的对象和是不是 ArtMethod 的一个实例。由于替换了 ArtMethod 导致匹配不上。
2. 不适合类发生结构变化的修改。比如增删方法可能引起类及 Dex 方法数变化，进而改变方法索引。同样地，增删字段也会更改方法索引。

**参考资料**

* [AndFix](https://github.com/alibaba/AndFix)
* [Sophix](https://www.aliyun.com/product/hotfix?spm=5176.56143.765261.332.2NlVMD)