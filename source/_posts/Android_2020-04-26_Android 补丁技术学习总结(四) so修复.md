---
title: Android 补丁技术学习总结(四) so修复
date: 2020-05-02 18:00:04
comments: true
categories: Android进阶
tags: [热修复]
---

#### 加载 so
* `System.loadLibrary(String soName)`， **libs** 目录下的 so 文件会被复制到应用安装目录并完成加载
* `System.load(String soPath)`,用于加载一个完整路径的 so 文件

#### 注册 so 
* 静态注册，使用 `Java_{类完整路径}_{方法名}` 作为 native 的方法名。当 so 已经被加载之后，native 方法在第一次被执行时候就会完成注册。

	```
	public class Test{
		public static native String test();
	}
	extern "C" jstring Java_com_effective_android_test(JNIEnv *env,jclass clazz)
	```
* 动态注册，借助 `JNI_OnLoad` 方法完成绑定。当 so 被加载时会调用 *JNI_OnLoad* 方法进行注册。

	```
	public class Test{
		public static native void testJni();
	}
	void test(JNIEnv *env,jclass clazz){
		//native impl
	}
	JNINativeMethod nativeMethods[] = {
		{"test","()V",(void *) test}
	}
	JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm,void *reserved){
		//...
		jclass clz = env->FindClass("com/effective/android/Test");
		if(env->RegisterNatives(clz, nativeMethods,sizeOf(nativeMethods)/sizeOf(nativeMethods[0])) != JNI_OK){
			return JNI_ERR;
		}
		//...
	}
	```

#### so 热替换的限制性

1. 针对动态注册场景
	* 对于 art 虚拟机下，可再次加载补丁 so 来完成方法映射的更新；
	* 对于 dalvik 虚拟机下，需要对补丁 so 重命名来避免来完成 art 下的方法映射的更新。
2. 针对静态注册场景
	* 解除已经完成静态注册的方法工作难度大
	* so 中哪些静态注册的方法需要更新也很难得知

上述两个场景涉及补丁 so 的二次加载，内存损耗大，可能导致 JNI OOM出现。同时如果动态注册 so 中新增了一些方法但是对应的 dex 中没有对应的代码，则会出现 `NoSuchMethodError`。 

#### so 冷启动方案

假如在在应用加载 so 之前，能够先尝试加载补丁 so，再加载应用 so，就可以实现修复。自定义一个方法，替换掉 `System.loadLibrary()` 来完成这个逻辑。 但是存在一个缺点就是很难修复已经混淆编译的第三方库。

这里最终采取的是类似 “类修复” 的注入方案。so 库被加载之后，最终会在 `DexPathList.nativeLibararyDirectories/nativeLiraryPathElements` 变量所表示的目录下遍历搜索。

**SDK < 23**

```
private final File[] nativeLibraryDirectories;
public String findLibrary(String libraryName){
	String fileName = System.mapLibraryName(libraryName);
	for(File directory : nativeLibraryDirectories){
		String path = new File(directory,fileName).getPath();
		//如果path文件存在且可读
		if(IoUtils.canOpenReadOnly(path)){
			return path;
		}
	}
}
```
只需要把补丁 so 库的路径插到 `nativeLibraryDirectories` 最前面。

**SDK >= 23**

```
private final File[] nativeLiraryPathElements;
public String findLibrary(String libraryName){
	String fileName = System.mapLibraryName(libraryName);
	for(Element element : nativeLibraryElements){
		String path = element.findNativeLibrary(fileName);
		if(path != null){
			return path;
		}
	}
}
```
只需要为补丁 so 构建一个 *element* 对象并插到 `nativeLiraryPathElements` 最前面。

但是 so 库文件存在多种 CPU 架构，补丁和 apk 一样都存在需要选择哪个 abi 的 so 来执行的问题。

**Sophix** 提供了一种思路, 通过从多个 abis 目录中选择一个合适的 `primaryCpuAbi` 目录插到 `nativeLibararyDirectories/nativeLiraryPathElements` 数组中。

* SDK >= 21,直接反射拿到 ApplicationInfo 对象的 `primaryCpuAbi`
* SDK < 21,由于不支持 64 位所以直接把 `Build.CPU_ABI, Build.CPU_ABI2` 作为 `primaryCpuAbi`

```
static{
	try{
		Package pm = mApp.getPackageManager();
		if(pm != null){
			ApplicationInfo mAppInfo = pm.getApplicationInfo(mApp.getPackageName(),0);
			if(mAppInfo != null){
				if(Build.VERSION>SDK_INT >= Build.VERSION_CODES>LOLLIPOP){
					File thirdFiled = ApplicationInfo.class.getDeclaredFiled("primaryCpuAbi");
					thirdFiled.setAccessable(true);
					String cupAbi = (String) thirdFiled.get(mAppInfo);
					primaryCpuAbis = new String[](cpuAbi);
				}else{
					primaryCpuAbis = new String[](Build.CPU_ABI，Build.CPU_ABI2);
				}
			}
		}
	}catch(Throwable t){
		//...
	}
}
```

**参考资料** 

* [从JNI_OnLoad看so的加载](https://www.jianshu.com/p/4c0f72233f65)
* [Android JNI 函数注册的两种方式(静态注册/动态注册)](https://www.jianshu.com/p/1d6ec5068d05)
* 《深入探索 Android 热修复技术原理》









  
