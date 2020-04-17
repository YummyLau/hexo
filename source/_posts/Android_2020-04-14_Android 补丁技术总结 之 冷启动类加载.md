---
title: Android 补丁技术总结 之 冷启动类加载
date: 2020-04-14 15:17:34
comments: true
categories: Android
tags: [热修复]
---

所谓 **冷启动类加载**，即应用进行重启之后，加载修复后的类文件达到修复类的已知问题。

一个类已经被虚拟机所加载，为了清除虚拟机中的类信息，只能通过重启手段解决。

这里以 **QZone** [插桩](https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=0) ，**手Q** [QFix](https://cloud.tencent.com/developer/article/1071333) 和  **微信** [tinker](https://github.com/Tencent/tinker)  方案为分析，并引用 **Sophix** 的做法做对比。

### QZone方案
一个 **ClassLoader** 可以加载多个 **dex** 文件，每一个 **dex** 文件实是一个 *Element*，多个 **dex** 文件排列成一个有序的数组 *dexElements*。 对于 [ClassLoader](http://yummylau.com/2019/03/01/java_2019-03-01_%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8/) 不熟悉的朋友，建议先看看链接里的储备知识。

如果一个类已经被  **ClassLoader**  加载，那么在查找 *class* 对象 *findClass(String name, List<Throwable> suppressed)* 过程中，如果已存在已查找的 *class* ，则直接返回该 *class*。所有 QZone 采用的方案是把经过修改过的类打包成新的 **dex** 文件，把该 **dex** 文件优先插到 *dexElements* 的待修复类 **dex** 文件的前面。

这个方案涉及到类校验问题，该问题是由于 dex 文件被优化导致的。当我们第一次安装 APK 的时候，会对原 dex 执行 dexopt。如果使用上述方案插入一个 **dex** 文件，则会先执行 *dexopt* 进行优化。对应 [DexPrepare.cpp#verifyAndOptimizeClass](https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/analysis/DexPrepare.cpp) 

```
/*
 * Verify and/or optimize a specific class.
 */
static void verifyAndOptimizeClass(DexFile* pDexFile, ClassObject* clazz,
    const DexClassDef* pClassDef, bool doVerify, bool doOpt)
{
	//...
	
    if (doVerify) {
        if (dvmVerifyClass(clazz)) { //Verify class
            ((DexClassDef*)pClassDef)->accessFlags |= CLASS_ISPREVERIFIED;	// 打上 CLASS_ISPREVERIFIED
            verified = true;  
        } else {
            // TODO: log when in verbose mode
            ALOGV("DexOpt: '%s' failed verification", classDescriptor);
        }
    }
    if (doOpt) {
        bool needVerify = (gDvm.dexOptMode == OPTIMIZE_MODE_VERIFIED ||
                           gDvm.dexOptMode == OPTIMIZE_MODE_FULL);
        if (!verified && needVerify) {
            ALOGV("DexOpt: not optimizing '%s': not verified",
                classDescriptor);
        } else {
            dvmOptimizeClass(clazz, false);  //Optimize class
            /* set the flag whether or not we actually changed anything */
            ((DexClassDef*)pClassDef)->accessFlags |= CLASS_ISOPTIMIZED; // 再打上 CLASS_ISOPTIMIZED
        }
    }
}
```

* dvmVerifyClass 的目的在于防止类被篡改。会对类的 *static方法*，*private方法*，*构造函数*，*虚函数(可被继承的函数)* 进行校验，如果类的所有方法中直接引用到的第一层类和当前类是在同一个 **dex** 文件，则会返回 true
* dvmOptimizeClass 的目的在于把部分指令优化成虚拟机内部指令，为了提升方法的执行速度。

假如原先有个 **dex** 文件中类 B 引用了类 A，则 B 会被打上 **CLASS_ISPREVERIFIED**，现在打了修复 **dex** 文件包含了类 A。当类 B 某个方法引用到类 A 的时候，就会尝试去解析类 A。 对应 [Resolve.cpp#dvmResolveClass](https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/oo/Resolve.cpp)

```
ClassObject* dvmResolveClass(const ClassObject* referrer, u4 classIdx,bool fromUnverifiedConstant){

	//...
	 resClass = dvmDexGetResolvedClass(pDvmDex, classIdx);
    if (resClass != NULL)
        return resClass;
	 
	 className = dexStringByTypeIdx(pDvmDex->pDexFile, classIdx);
    if (className[0] != '\0' && className[1] == '\0') {
        /* primitive type */
        resClass = dvmFindPrimitiveClass(className[0]);
    } else {
        resClass = dvmFindClassNoInit(className, referrer->classLoader);
    }
	
    if (resClass != NULL) {
        if (!fromUnverifiedConstant &&
            IS_CLASS_FLAG_SET(referrer, CLASS_ISPREVERIFIED)) //B类已经被打上 CLASS_ISPREVERIFIED，满足条件
        {
            ClassObject* resClassCheck = resClass;    //类 A
            if (dvmIsArrayClass(resClassCheck))
                resClassCheck = resClassCheck->elementClass;
            if (referrer->pDvmDex != resClassCheck->pDvmDex &&
                resClassCheck->classLoader != NULL)  //发现类 A 和 类 B 不在同一个 dex
            {
                dvmThrowIllegalAccessError(
                    "Class ref in pre-verified class resolved to unexpected "
                    "implementation");
                return NULL;
            }
        }
        dvmDexSetResolvedClass(pDvmDex, classIdx, resClass);
    } else {
        assert(dvmCheckException(dvmThreadSelf()));
    }
    return resClass;
}
```
为了解决类校验的问题， 只需要让 *dvmVerifyClass* 返回 false 就可以了。QZone 的做法是使用字节码修改技术，在所有 .class 文件的构造器中引用一个帮助类，该类单独存放在一个 **dex** 文件中，就可以实现所有类都不会被打上 **CLASS_ISPREVERIFIED** 标志，进而避免在 *dvmResolveClass* 解析中出现异常。

**存在的问题与限制**

由于类的加载涉及 *dvmResolveClass*，*dvmLinkClass* 和 *dvmInitClass* 三个阶段。*dvmInitClass* 会在类解析完并尝试初始化类的时候执行，如果类没有被打上  **CLASS_ISPREVERIFIED**  或  **CLASS_ISOPTIMIZED** ，校验和优化都会在该阶段进行。而正常情况下类的校验和优化应该在 APK 第一次安装的时候执行 *dexopt* 操作时执行的，但是我们干预了 **CLASS_ISPREVERIFIED** 的设置流程，导致在同一时间加载大量类，加载效率也收到比较大的影响。应用刚启动的时候如果出现这种情况，则容易导致白屏。

### 手Q方案

为了避免插桩带来的性能问题，手Q在 *dvmResolveClass* 直接避开了 **CLASS_ISPREVERIFIED** 的相关逻辑。参考上面 [Resolve.cpp#dvmResolveClass](https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/oo/Resolve.cpp)

```
ClassObject* dvmResolveClass(const ClassObject* referrer, u4 classIdx,bool fromUnverifiedConstant){

	DvmDex* pDvmDex = referrer->pDvmDex;

	//...
	
	//从dex缓存中查找类 class
	 resClass = dvmDexGetResolvedClass(pDvmDex, classIdx);
    if (resClass != NULL)
        return resClass;
    //...
    
    
    if (resClass != NULL) {
		
        if (!fromUnverifiedConstant &&IS_CLASS_FLAG_SET(referrer, CLASS_ISPREVERIFIED)){
            ClassObject* resClassCheck = resClass;
            if (dvmIsArrayClass(resClassCheck))
                resClassCheck = resClassCheck->elementClass;
            if (referrer->pDvmDex != resClassCheck->pDvmDex &&resClassCheck->classLoader != NULL){
                dvmThrowIllegalAccessError(
                    "Class ref in pre-verified class resolved to unexpected "
                    "implementation");
                return NULL;
            }
        }
			
		 //已经解析的类放入 dex 缓存
        dvmDexSetResolvedClass(pDvmDex, classIdx, resClass);
    }
    //...
}
```
* *dvmDexGetResolvedClass* 方法是尝试从 dex 缓存中查找引用的类，找到了就直接返回
* *dvmDexSetResolvedClass* 方法是将已经解析的类存入 dex 缓存中

所以，只需要将补丁类 A 提前解析并设置 **fromUnverifiedConstant** 为 true 绕过类校验，然后存储 dex 缓存中。这一步可以通过 jni 调用 *dalvik#dvmRsolveClass* 方法实现。后续引用到该补丁类 A 的时候就可以直接从 dex 缓存中找到。当类 B 在校验的时候，`referrer->pDvmDex != resClassCheck->pDvmDex` 如果不打破这个条件，依然会出现异常。所以对补丁类 A 进行 dex 缓存的时候，拿到的 *pDvmDex* 应该是原来类 A 所在的 dex 。

那么在 *dalvik#dvmRsolveClass* 的过程中，*referrer* 和 *classIdx* 要怎么确定 ？

* *referrer* 为和原类 A 同个 dex 下的一个任意类即可。但是需要调用 *dvmFindLoadedClass* 来实现，在补丁注入之后，在每个 dex 中找一个已经成功加载的引用类的描述符作为参数来实现。比如主 dex 就用 *Application* 类描述符，其他 dex，手Q 确保了每一个分 dex 有一个空类完成初始化，使用的是空类的描述符。
* *classIdx* 为原类 A 在所有 dex 下的类索引 ID，通过 `dexdump -h ` 指令获取。

这套方案可以完美避开插桩所带来的类校验影响。但是假如在某个待修复多态类中新增方法，可能会导致修复前类的 *vtable* 的索引与修复后类的 *vtable* 索引对不上。因此修复后的类不能新增 *public* 函数，同样 QZone 也存在这样的问题。所以只能寻找全量合成新 **dex** 文件的方案。


### Tinker方案
tinker 的方案是 **“全量替换 dex”**。使用自研的比较算法，把重新生成新的 **dex** 文件与需要修复的 **dex** 文件进行计算并得到差异 **dex** 文件，该差异 **dex** 文件会被下发到客户端与需要修复的 **dex** 文件进行合并生成全量 **dex** 文件，并把该 **dex** 文件插到 *dexElements* 数组的最前面。与 QZone 方法不一样的时，由于被修复的类与原类是在同一个 **dex** 文件内，所以不存在类校验的问题。

虚拟机加载 **dex** 文件时

* dalvik 虚拟机调用 [Dalvik_dalvik_system_DexFile_openDexFileNative](https://android.googlesource.com/platform/dalvik/+/android-4.4.4_r2/vm/native/dalvik_system_DexFile.cpp), 如果是一个压缩包则只会加载第一个 dex

	```
	if (dvmJarFileOpen(sourceName, outputName, &pJarFile, false) == 0) {
        ALOGV("Opening DEX file '%s' (Jar)", sourceName);
        pDexOrJar = (DexOrJar*) malloc(sizeof(DexOrJar));
        pDexOrJar->isDex = false;
        pDexOrJar->pJarFile = pJarFile;
        pDexOrJar->pDexMemory = NULL;
    }
	```
* art 虚拟机调用 [LoadDexFiles](https://android.googlesource.com/platform/art/+/master/runtime/oat_file_assistant.cc), 加载 oat 中多个 dex 文件

	```
	  // Load the main dex file.
  std::string error_msg;
  const OatDexFile* oat_dex_file = oat_file.GetOatDexFile(dex_location.c_str(), nullptr, &error_msg);
  if (oat_dex_file == nullptr) {
    LOG(WARNING) << error_msg;
    return false;
  }
  
	 // Load the rest of the multidex entries
  for (size_t i = 1;; i++) {
    std::string multidex_dex_location = DexFileLoader::GetMultiDexLocation(i, dex_location.c_str());
    oat_dex_file = oat_file.GetOatDexFile(multidex_dex_location.c_str(), nullptr);
    if (oat_dex_file == nullptr) {
      // There are no more multidex entries to load.
      break;
    }
    dex_file = oat_dex_file->OpenDexFile(&error_msg);
    if (dex_file.get() == nullptr) {
      LOG(WARNING) << "Failed to open dex file from oat dex file: " << error_msg;
      return false;
    }
    out_dex_files->push_back(std::move(dex_file));
  }
	```
在 art 虚拟机加载的压缩包下，可能存在多个 **dex** 文件，main dex 为 **classes.dex**，其他的 **dex** 文件依次命名为 **classes(2,3,4...)dex**。 假如某个 **classesNdex** 出现了问题，tinker 会重新合成 **classesNdex** 。

	1. 保留原来修复前 **classesNdex** dex 文件
	2. 获取修复后的 **classedNdexFix** dex 文件
	3. 使用算法计算得到 **classesNdexPatch** 补丁文件
	4. 下发 **classesNdexPatch** 补丁文件在客户端与 **classesNdex** dex 文件进行合并，得到 **classedNdexFix** dex 文件
	5. 重启应用，提前加载 **classedNdexFix** dex 文件修复问题。

这种全量合成修复 **dex** 的做法，确保了修复前后的类与原 **dex** 其他类在同一个 **dex** 中，遵循了原来虚拟机所有校验方式，避开了QZone方案面临的类校验问题。

### Sophix方案

阿里 **Sophix** 方案认为 **“既然 art 能加载压缩文件中的多个 dex 且优先加载 classes.dex，如果把补丁 dex 作为 classes.dex,然后 apk 中原来的 dex 改成 classes(2,3,4...)dex，然后重新打包压缩文件，让 DexFile.loadDex 得到 DexFile 对象，并最终替换掉旧的 dexElements 数组就可以了。”**

但是这种方案下，Art 需要重新加载整个压缩文件，针对每一个 dex 执行 dexoat 来得到 odex 的过程是很耗时的。需要把整个过程事务化，在接收到服务端补丁之后再启动一个子线程在后台进行异步处理。如果下次重启之后发现存在处理完的完整 odex 文件集，才进行处理。

同时认为 **“针对 dalvik 下，全量合成dex可参照 multi-dex 方案，在原来 dex 文件中剔除需要修复的类，然后再合并进修复的类。并不需要想 tinker 方案中针对 dex 的所有内容进行比较，粒度非常细也非常复杂，以类作为粒度作为替换是较佳选择”**。

但是如果 *Application* 加载了新 dex 的类 *Application* 刚好被打上 **CLASS_ISPREVERIFIED** ，那么就会面临前面 QZone 方案的类校验问题，实际上所有全量合成的方案都会面临这个问题。 **tinker** 使用的是 **TinkerApplication** 接管应用 **Application** 并在生命周期回调的时候反射调用原 **Application** 的对应方案。而 **Sophix** 也是使用 *SohpixStubApplication* 做了类似的事情。


**小结**

冷启动方案几乎可以修复任何代码场景，但是补丁注入前已经被加载的类，如 *Application* 等是无法被修复的。综合上面的多种方案可以得到针对不同虚拟机的优先冷启动方案：

* Dalvik 虚拟机下使用类 **multi-dex** 全量方案避免插桩的方案
* Art 虚拟机下使用补丁类作为 **classes.dex** 重新打包压缩文件进行加载的方案


**参考资料**

* [聊一聊 "类加载器"](http://yummylau.com/2019/03/01/java_2019-03-01_%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8/)
* [Android Dex分包最全总结：含Facebook解决方案](https://juejin.im/post/5ccee8776fb9a031ee3c2475)
* [基于cydia Hook在线热修复补丁方案](https://blog.csdn.net/xwl198937/article/details/49801975)
* [QFix探索之路——手Q热补丁轻量级方案](https://cloud.tencent.com/developer/article/1071333)
* [我理解的热修复中的ART地址错乱问题](https://www.jianshu.com/p/fa593cf27b5d)
* [Android热修复升级探索——代码修复冷启动方案](https://www.jianshu.com/p/68d2a92eb7ab)