---
title: 微信Tinker热修复的预研实践
date: 2017-02-06 09:35:20
comments: true
categories: Android
tags: [热修复]
---
<!--more-->

>  基于对热补丁的迫切认识及预研热补丁对当前项目的正向帮助，在主流热补丁技术中选择微信Tinker方案作为学习记录。本次记录基于自身认知及技术水平，如有错误或可改进，望指出。

# 基础认识

* 热修复补丁（hotfix），又称为**patch**，指能够修复软件漏洞的一些代码，是一种快速、低成本修复产品软件版本缺陷的方式。在Android端实现应用在无需重新安装的情况实现更新，帮助应用快速建立动态修复能力。
* 热修复意义，能节省Android大量应用市场发布的时间。同时用户也无需重新安装，只要上线就能无感知的更新。当然热补丁技术也存在它的局限性，热补丁主要用来进行代码修复和轻量级的升级。
* 应用场景
	* 代码修复和轻量级的升级。
	* 远端调试。可以向特定用户特定设备发送补丁，能帮助查找具体的验证问题。利用补丁技术，我们避免了骚扰用户而默默的为用户解决问题。
	* 数据统计。可将热补丁技术与数据统计结合起来。事实上，热补丁技术无论在数据统计还是ABTest都有着非常大的优势。例如若想对同一批用户做两种test, 传统方式无法让这批用户去安装两个版本,使用补丁技术，可以对同一批用户更换补丁达到效果。
	
# 修复方案的比较

|	团队	|	Tinker	|	QZone	|	AndFix	|	Robust	|
|	:----:	|	:----:	|	:----:	|	:----:	|	:----:	|
|	类替换	|	yes		|	yes		|	no		|	no		|
|	So替换	|	yes		|	no		|	no		|	no		|
|  资源替换	|	yes		|	yes		|	no		|	no		|
|全平台支持	|	yes		|	yes		|	yes		|	yes		|
|即时生效	|	no		|	no		|	yes		|	yes		|
|性能损耗	|	较小	|	较大	|	较小	|	较小	|
|补丁包大小	|	较小	|	较大	|	一般	|	一般	|
|开发透明	|	yes		|	yes		|	no		|	no		|
|	复杂度	|	较低	|	较低	|	复杂	|	复杂	|
|gradle支持	|	yes		|	no		|	no		|	no		|
|Rom体积	|	较大	|	较小	|	较小	|	较小	|	
|	成功率	|	较高	|	较高	|	一般	|	最高	|	

# Demo集成开发

通过集成Tinker SDK来生成补丁包，而官方推荐的补丁包分发则通过[Tinker Platform](http://www.tinkerpatch.com/)来实现。**Tinker Platform**是官方提供的付费分发平台，支持补丁分发监控等功能。如果是小应用则可考虑接入，否则可自行开发CMS补丁管理后台来实现其支持的功能。在预研过程中接入过**Tinker Platform**，实现起来比较简单，可参考[官网Demo](https://github.com/TinkerPatch/tinkerpatch-sample)来实现。
<iframe height=640 width=360 src="http://ojyx20l6a.bkt.clouddn.com/tinker_test_demo.mp4" frameborder=0 allowfullscreen></iframe>
这里我们才用比较传统的思路来模拟一次分发：客户端主动轮训补丁下载后台或者被动接收推送信令后主动请求下载补丁，然后在客户端中通过Tinker SDK完成补丁的载入与修复，只要理解大致的思路，完全可以开发出满足自身业务的功能。Demo中由于锁屏会默认终端录制，所以选择在修复完之后故意crash一次达到重启，实际项目中可带到锁屏或者应用进入后台之后进行重启即可。


## 	Tinker接入

### 依赖配置

* 项目gradle.properties定义tinker版本
```
#配置tinker版本为1.7.7
TINKER_VERSION=1.7.7
```

* 项目build.gradle添加tinker-patch-gradle-plugin的依赖
```
//添加tinker依赖
classpath "com.tencent.tinker:tinker-patch-gradle-plugin:${TINKER_VERSION}"
```

* App build.gradle添加tinker的库依赖以及apply tinker的gradle插件
```
dependencies {
    //tinker的核心库
    compile("com.tencent.tinker:tinker-android-lib:${TINKER_VERSION}") { changing = true }
    //可选，用于生成application类
    provided("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
	......
}
```

* App build.gradle配置参考
```
apply plugin: 'com.android.application'

def javaVersion = JavaVersion.VERSION_1_7

android {
    compileSdkVersion 24
    buildToolsVersion "24.0.3"

    compileOptions {
        sourceCompatibility javaVersion
        targetCompatibility javaVersion
    }

    //recommend
    dexOptions {
        jumboMode = true
    }

    //签名信息
    signingConfigs {
        release {
            try {
                storeFile file("./keystore/release.keystore")
                storePassword "testres"
                keyAlias "testres"
                keyPassword "testres"
            } catch (ex) {
                throw new InvalidUserDataException(ex.toString())
            }
        }

        debug {
            storeFile file("./keystore/debug.keystore")
        }
    }

    defaultConfig {
        applicationId "com.example.yummylau.tinkertest"
        minSdkVersion 15
        targetSdkVersion 22
        versionCode 1
        versionName "1.0.0"
        /**
         * you can use multiDex and install it in your ApplicationLifeCycle implement
         */
        multiDexEnabled true
        /**
         * buildConfig can change during patch!
         * we can use the newly value when patch
         */
        buildConfigField "String", "MESSAGE", "\"I am the base apk\""
        /**
         * client version would update with patch
         * so we can get the newly git version easily!
         */
        buildConfigField "String", "TINKER_ID", "\"${getTinkerIdValue()}\""
        buildConfigField "String", "PLATFORM",  "\"all\""
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"

        ndk{
            //极光推送 添加对应cpu类型的.so库
            abiFilters 'armeabi','armeabi-v7a','armeabi-v8a','x86'
        }

        //配置极光推送
        manifestPlaceholders = [
            JPUSH_PKGNAME : applicationId,
            JPUSH_APPKEY : "your app Jpush_key",
            JPUSH_CHANNEL : "developer-default",
        ]
    }

    //配置混淆信息
    buildTypes {
        release {
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
        }
    }

    //声明libs目录
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:24.2.1'
    testCompile 'junit:junit:4.12'

    compile "com.android.support:multidex:1.0.1"

    //tinker的核心库
    compile("com.tencent.tinker:tinker-android-lib:${TINKER_VERSION}") { changing = true }
    //可选，用于生成application类
    provided("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }

    //接入极光推送
    compile 'cn.jiguang.sdk:jpush:3.0.0'
    compile 'cn.jiguang.sdk:jcore:1.0.0'
}


//配置旧包路径
def bakPath = file("${buildDir}/bakApk/")
//基本配置信息
ext {
    //是否启用tinker热修复
    tinkerEnabled = true

    //Old Apk路径
    tinkerOldApkPath = "${bakPath}/app-debug-0119-19-32-57.apk"
    //proguard mapping路径
    tinkerApplyMappingPath = "${bakPath}/app-debug-0119-19-32-57-mapping.txt"
    //resource R.txt路径
    tinkerApplyResourcePath = "${bakPath}/app-debug-0119-19-32-57-R.txt"

    //only use for build all flavor, if not, just ignore this field
    tinkerBuildFlavorDirectory = "${bakPath}/app-debug-0119-19-32-57-R.txt"
}
def gitSha() {
    try {
        //把app的versionCode定义为Tinker id
        String gitRev = android.defaultConfig.versionCode
        if (gitRev == null) {
            throw new GradleException("can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId'")
        }
        return gitRev
    } catch (Exception e) {
        throw new GradleException("can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId'")
    }
}
//获取OldApk路径
def getOldApkPath() {
    return hasProperty("OLD_APK") ? OLD_APK : ext.tinkerOldApkPath
}
//获取ApplyMapping路径
def getApplyMappingPath() {
    return hasProperty("APPLY_MAPPING") ? APPLY_MAPPING : ext.tinkerApplyMappingPath
}
//获取ApplyResource路径
def getApplyResourceMappingPath() {
    return hasProperty("APPLY_RESOURCE") ? APPLY_RESOURCE : ext.tinkerApplyResourcePath
}
//获取TINKER_ID
def getTinkerIdValue() {
    return hasProperty("TINKER_ID") ? TINKER_ID : gitSha()
}
//是否启用tinker
def buildWithTinker() {
    return hasProperty("TINKER_ENABLE") ? TINKER_ENABLE : ext.tinkerEnabled
}
//获取多渠道路径
def getTinkerBuildFlavorDirectory() {
    return ext.tinkerBuildFlavorDirectory
}

//tinker的所有操作
if (buildWithTinker()) {

    apply plugin: 'com.tencent.tinker.patch'

    //全局信息相关的配置项
    tinkerPatch {

        //默认null，	基准apk包的路径，必须输入，否则会报错。
        oldApk = getOldApkPath()

        /**
         *  如果出现以下的情况，并且ignoreWarning为false，将中断编译。因为这些情况可能会导致编译出来的patch包带来风险：
         *  1. minSdkVersion小于14，但是dexMode的值为"raw";
         *  2. 新编译的安装包出现新增的四大组件(Activity, BroadcastReceiver...)；
         *  3. 定义在dex.loader用于加载补丁的类不在main dex中;
         *  4. 定义在dex.loader用于加载补丁的类出现修改；
         *  5. resources.arsc改变，但没有使用applyResourceMapping编译。
         */
        ignoreWarning = false

        /**
         *  在运行过程中，需要验证基准apk包与补丁包的签名是否一致，我们是否需要为你签名。
         */
        useSign = true

        /**
         *  是否打开tinker的功能。
         */
        tinkerEnable = buildWithTinker()

        //编译相关的配置项
        buildConfig {

            /**
             *  可选参数；在编译新的apk时候，我们希望通过保持旧apk的proguard混淆方式，从而减少补丁包的大小。
             *  这个只是推荐的，但设置applyMapping会影响任何的assemble编译。
             */
            applyMapping = getApplyMappingPath()

            /**
             * 可选参数；在编译新的apk时候，我们希望通过旧apk的R.txt文件保持ResId的分配，这样不仅可以减少
             * 补丁包的大小，同时也避免由于ResId改变导致remote view异常。
             */
            applyResourceMapping = getApplyResourceMappingPath()

            /**
             * 	在运行过程中，我们需要验证基准apk包的tinkerId是否等于补丁包的tinkerId。
             * 	这个是决定补丁包能运行在哪些基准包上面，一般来说我们可以使用git版本号、versionName等等。
             */
            tinkerId = getTinkerIdValue()

            /**
             * 	如果我们有多个dex,编译补丁时可能会由于类的移动导致变更增多。
             * 	若打开keepDexApply模式，补丁包将根据基准包的类分布来编译。
             */
            keepDexApply = false
        }

        //	dex相关的配置项
        dex {

            /**
             * 	只能是'raw'或者'jar'。
             *  对于'raw'模式，我们将会保持输入dex的格式。
             *  对于'jar'模式，我们将会把输入dex重新压缩封装到jar。
             *  如果你的minSdkVersion小于14，你必须选择‘jar’模式，而且它更省存储空间，但是验证md5时比'raw'模式耗时。
             *  默认我们并不会去校验md5,一般情况下选择jar模式即可。
             */
            dexMode = "jar"

            /**
             * 	需要处理dex路径，支持*、?通配符，必须使用'/'分割。
             * 	路径是相对安装包的，例如assets/...
             */
            pattern = ["classes*.dex",
                       "assets/secondary-dex-?.jar"]
            /**
             * 	这一项非常重要，它定义了哪些类在加载补丁包的时候会用到。这些类是通过Tinker无法修改的类，也是一定要放在main dex的类。
             *  这里需要定义的类有：
             *  1. 你自己定义的Application类；
             *  2. Tinker库中用于加载补丁包的部分类，即com.tencent.tinker.loader.*；
             *  3. 如果你自定义了TinkerLoader，需要将它以及它引用的所有类也加入loader中；
             *  4. 其他一些你不希望被更改的类，例如Sample中的BaseBuildInfo类。这里需要注意的是，这些类的直接引用类也需要加入到loader中。
             *  或者你需要将这个类变成非preverify。
             *  5. 使用1.7.6版本之后版本，参数1、2会自动填写。
             */
            loader = [
                    "com.tencent.tinker.loader.*",
                    "com.tencent.tinker.*",
                    "com.example.yummylau.tinkertest.report.*",
                    "com.example.yummylau.tinkertest.crash.*",
                    "com.example.yummylau.tinkertest.receiver.*",
                    "com.example.yummylau.tinkertest.service.*",
                    "com.example.yummylau.tinkertest.TestApplication",
                    "com.example.yummylau.tinkertest.Constants"
            ]

        }

        //  lib相关的配置项
        lib {
            /**
             * 	需要处理lib路径，支持*、?通配符，必须使用'/'分割。
             * 	与dex.pattern一致, 路径是相对安装包的，例如assets/...
             */
            pattern = ["lib/armeabi/*.so"]
        }

        //	res相关的配置项
        res {

            /**
             * 	需要处理res路径，支持*、?通配符，必须使用'/'分割。
             * 	与dex.pattern一致, 路径是相对安装包的，例如assets/...，
             * 	务必注意的是，只有满足pattern的资源才会放到合成后的资源包。
             */
            pattern = ["res/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]

            /**
             * 支持*、?通配符，必须使用'/'分割。若满足ignoreChange的pattern，在编译时会忽略该文件的新增、删除与修改。
             * 最极端的情况，ignoreChange与上面的pattern一致，即会完全忽略所有资源的修改。
             */
            ignoreChange = ["assets/sample_meta.txt"]

            /**
             * 	对于修改的资源，如果大于largeModSize，我们将使用bsdiff算法。
             * 	这可以降低补丁包的大小，但是会增加合成时的复杂度。默认大小为100kb
             */
            largeModSize = 100
        }

        //  用于生成补丁包中的'package_meta.txt'文件
        packageConfig {

            /**
             * NEW_TINKER_ID
             * configField("key", "value"), 默认我们自动从基准安装包与新安装包的Manifest中读取tinkerId,并自动写入configField。
             * 在这里，你可以定义其他的信息，在运行时可以通过TinkerLoadResult.getPackageConfigByName得到相应的数值。
             * 但是建议直接通过修改代码来实现，例如BuildConfig。
             */
            configField("patchMessage", "tinker is sample to use")
            /**
             * just a sample case, you can use such as sdkVersion, brand, channel...
             * you can parse it in the SamplePatchListener.
             * Then you can use patch conditional!
             */
            configField("platform", "all")
            /**
             * patch version via packageConfig
             */
            configField("patchVersion", "1")
        }
        //or you can add config filed outside, or get meta value from old apk
        //project.tinkerPatch.packageConfig.configField("test1", project.tinkerPatch.packageConfig.getMetaDataFromOldApk("Test"))
        //project.tinkerPatch.packageConfig.configField("test2", "sample")

        //7zip路径配置项，执行前提是useSign为true
        sevenZip {

            /**
             *  例如"com.tencent.mm:SevenZip:1.1.10"，将自动根据机器属性获得对应的7za运行文件，推荐使用。
             */
            zipArtifact = "com.tencent.mm:SevenZip:1.1.10"

            /**
             * 	系统中的7za路径，例如"/usr/local/bin/7za"。path设置会覆盖zipArtifact，若都不设置，将直接使用7za去尝试。
             *
             */
//        path = "/usr/local/bin/7za"
        }
    }

    List<String> flavors = new ArrayList<>();
    project.android.productFlavors.each {flavor ->
        flavors.add(flavor.name)
    }
    boolean hasFlavors = flavors.size() > 0
    /**
     * bak apk and mapping
     */
    android.applicationVariants.all { variant ->
        /**
         * task type, you want to bak
         */
        def taskName = variant.name
        def date = new Date().format("MMdd-HH-mm-ss")

        tasks.all {
            if ("assemble${taskName.capitalize()}".equalsIgnoreCase(it.name)) {

                it.doLast {
                    copy {
                        def fileNamePrefix = "${project.name}-${variant.baseName}"
                        def newFileNamePrefix = hasFlavors ? "${fileNamePrefix}" : "${fileNamePrefix}-${date}"

                        def destPath = hasFlavors ? file("${bakPath}/${project.name}-${date}/${variant.flavorName}") : bakPath
                        from variant.outputs.outputFile
                        into destPath
                        rename { String fileName ->
                            fileName.replace("${fileNamePrefix}.apk", "${newFileNamePrefix}.apk")
                        }

                        from "${buildDir}/outputs/mapping/${variant.dirName}/mapping.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("mapping.txt", "${newFileNamePrefix}-mapping.txt")
                        }

                        from "${buildDir}/intermediates/symbols/${variant.dirName}/R.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("R.txt", "${newFileNamePrefix}-R.txt")
                        }
                    }
                }
            }
        }
    }
    project.afterEvaluate {
        //sample use for build all flavor for one time
        if (hasFlavors) {
            task(tinkerPatchAllFlavorRelease) {
                group = 'tinker'
                def originOldPath = getTinkerBuildFlavorDirectory()
                for (String flavor : flavors) {
                    def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Release")
                    dependsOn tinkerTask
                    def preAssembleTask = tasks.getByName("process${flavor.capitalize()}ReleaseManifest")
                    preAssembleTask.doFirst {
                        String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 15)
                        project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release.apk"
                        project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-mapping.txt"
                        project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-R.txt"

                    }

                }
            }

            task(tinkerPatchAllFlavorDebug) {
                group = 'tinker'
                def originOldPath = getTinkerBuildFlavorDirectory()
                for (String flavor : flavors) {
                    def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Debug")
                    dependsOn tinkerTask
                    def preAssembleTask = tasks.getByName("process${flavor.capitalize()}DebugManifest")
                    preAssembleTask.doFirst {
                        String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 13)
                        project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug.apk"
                        project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-mapping.txt"
                        project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-R.txt"
                    }

                }
            }
        }
    }
}

```
核心的逻辑是：带修复的apk包存放在*bakPath*中，然后修复旧包代码后，运行patch脚本生成**Diff 补丁包**，把该补丁包放到分发平台即可。

### Application配置

```
@DefaultLifeCycle(application = "com.example.yummylau.tinkertest.TestApplication",
        flags = ShareConstants.TINKER_ENABLE_ALL,
        loadVerifyFlag = false)
public class MyApplication extends DefaultApplicationLike{

    public MyApplication(Application application, int tinkerFlags, boolean tinkerLoadVerifyFlag, long applicationStartElapsedTime, long applicationStartMillisTime, Intent tinkerResultIntent) {
        super(application, tinkerFlags, tinkerLoadVerifyFlag, applicationStartElapsedTime, applicationStartMillisTime, tinkerResultIntent);
    }

    @Override
    public void onCreate() {
        super.onCreate();
        JPushInterface.init(getApplication());
        JPushInterface.setDebugMode(true);
    }

    @Override
    public void onBaseContextAttached(Context base) {
        super.onBaseContextAttached(base);
        MultiDex.install(base);

        TinkerManager.setTinkerApplicationLike(this);

        TinkerManager.initFastCrashProtect();
        //should set before tinker is installed
        TinkerManager.setUpgradeRetryEnable(true);

        //installTinker after load multiDex
        //or you can put com.tencent.tinker.** to main dex
        TinkerManager.installTinker(this);
        Tinker tinker = Tinker.with(getApplication());
    }
}

```
Application的重写选择官方推荐的“注解方案”，Tinker会自动帮我们生成一个真正的Application，这里需要注意的是**DefaultLifeCycle**中声明的**com.example.yummylau.tinkertest.TestApplication**才是我们真正需要配置的命名，其需要配置主要是在AndroidManifest中
```	
	//appcalition的声明
    <application
        android:name=".TestApplication"
        android:allowBackup="true"
        android:icon="@mipmap/icon"
        android:label="@string/app_name"
		.....
```
及其依赖配置中 App build.gradle
```
dex{
	loader = [
			"com.example.yummylau.tinkertest.TestApplication",
			......
	]
}
```

### 模拟错误场景

由于Demo演示需要监控Tinker从补丁载入到合并完成的整个过程，开发者可自行实现[官方监控Api](https://github.com/Tencent/tinker/wiki/Tinker-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%89%A9%E5%B1%95) ，这里不做多余展示。
在Demo中底部跳转页面过程留了一个空指针
```
public class CrashActivity extends AppCompatActivity{

    private TextView mFixText;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_crash);
        initView();
    }

    private void initView(){
//        mFixText = (TextView)findViewById(R.id.fix_info);
        //ignore findViewById() result to bug
        mFixText.setText(getString(R.string.to_crash_activity_tip));
    }
}
```
然后运行项目Gradle 中 Tasks-build-assembleDebug（也可以选择Release版本），会在之前定义的**def bakPath = file("${buildDir}/bakApk/")**目录下生成一个以app-debug-date.apk的包，我们这里记录为**旧包**，简记*old.apk*。
接着修复空指针代码
```
public class CrashActivity extends AppCompatActivity{

    private TextView mFixText;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_crash);
        initView();
    }

    private void initView(){
        mFixText = (TextView)findViewById(R.id.fix_info);
        //ignore findViewById() result to bug
        mFixText.setText(getString(R.string.fix_info));
    }
}
```

并在App build.gradle中配置**旧包**信息
```
//基本配置信息
ext {
    //是否启用tinker热修复
    tinkerEnabled = true

    //Old Apk路径
    tinkerOldApkPath = "${bakPath}/old.apk"
    //proguard mapping路径
    tinkerApplyMappingPath = "${bakPath}/old-mapping.txt"
    //resource R.txt路径
    tinkerApplyResourcePath = "${bakPath}/old7-R.txt"

    //only use for build all flavor, if not, just ignore this field
    tinkerBuildFlavorDirectory = "${bakPath}/old-R.txt"
}
```

debug版本下不需要用到混淆之后的mapping，只需要配置tinkerOldApkPath、tinkerApplyResourcePath及tinkerBuildFlavorDirectory即可。
运行项目Gradle 中 Tasks-tinker-tinkerPatchDebug,则会在项目app-build-outputs-tinkerPatch目录中生成**Diff补丁包**

## 上传补丁包
为了Demo效果，选择[七牛云](https://www.qiniu.com/)暂时托管补丁包，实际生产环境需要考虑补丁包的监控及安全性。把上述生成的**Diff补丁包**上传到私有云空间中，并拿到外链，简记为*http://dowmload/patch.zip*。

## Jpush推送集成
[Jpush文档](https://docs.jiguang.cn/jpush/client/Android/android_sdk/)对客户端的接入描述已足够清楚，这个不做多余记录。当我们运行上述旧包时，我们的修复策略是客户端主动轮训下载或者服务端主动推送下载信令让客户端实现下载。Demo演示选择后者方案。
写一个Jpush远端消息接受处理类
```
public class JpushReceiver extends BroadcastReceiver{

    private static final String TAG = "JpushReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        Bundle bundle = intent.getExtras();
        Log.d(TAG, "onReceive - " + intent.getAction() + ", extras: " + bundle.toString());

        if (JPushInterface.ACTION_REGISTRATION_ID.equals(intent.getAction())) {
            Log.d(TAG, "JPush用户注册成功");
            bundle.putInt(Constants.FLAG, Constants.MESSAGE_FLAG_PUSH_INIT);
            sendBroadcase(context,bundle);

        } else if (JPushInterface.ACTION_MESSAGE_RECEIVED.equals(intent.getAction())) {
            Log.d(TAG, "接受到推送下来的自定义消息");
            //服务端下发补丁信令
            bundle.putInt(Constants.FLAG, Constants.MESSAGE_FLAG_MESSAGE);
            sendBroadcase(context,bundle);


        } else if (JPushInterface.ACTION_NOTIFICATION_RECEIVED.equals(intent.getAction())) {
            Log.d(TAG, "接受到推送下来的通知");
            String title = bundle.getString(JPushInterface.EXTRA_NOTIFICATION_TITLE);
            Log.d(TAG, " notification-title : " + title);
            String message = bundle.getString(JPushInterface.EXTRA_ALERT);
            Log.d(TAG, "notification-message : " + message);
            String extras = bundle.getString(JPushInterface.EXTRA_EXTRA);
            Log.d(TAG, "notification-extras : " + extras);

        } else if (JPushInterface.ACTION_NOTIFICATION_OPENED.equals(intent.getAction())) {
            Log.d(TAG, "用户点击打开了通知");
        } else {
            Log.d(TAG, "Unhandled intent - " + intent.getAction());
        }
    }


    private void sendBroadcase(Context context, Bundle bundle){
        Intent intent = new Intent();
        intent.putExtras(bundle);
        intent.setAction(Constants.BIZ_RECEIVER_ACTION);
        context.sendBroadcast(intent);
    }
}
```
在接收到下载信令之后，发个广播通知App需要下载补丁并相应下载。
```
	case Constants.MESSAGE_FLAG_MESSAGE:{
		if(HandleMessage(bundle.getString(JPushInterface.EXTRA_MESSAGE))){
			addLog("start request patch, download....");
			TaskManager.execAsynTask(new Runnable() {
				@Override
				public void run() {
					//下载补丁文件，mPatchUrl实际为外链地址http://dowmload/patch.zip
					final String result = DowmloadHelper.download(MainActivity.this, mPatchUrl,mPatchSize);
					//下载完成之后合成补丁
					TaskManager.execTaskOnUIThread(new Runnable() {
						@Override
						public void run() {
							if(result.equals(DowmloadHelper.DOWNLOAD_SUCCESS)){
								addLog("开始下载补丁包...");
									TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(),
											SDCardHelper.getSDCardPrivateFilesDir(MainActivity.this, null) + "/patch/" + Constants.PATCH_NAME);
								addLog(result);
								addLog("patch进程服务运行中...");
								addLog("请求patch进程合成补丁...");
							}else{
								addLog(result);
							}
						}
					});
				}
			});
		}
		break;
	}
```
下载文件方法,考虑安全性需要存放在data-data目录下的私有空间，补丁合成后会自动删除补丁文件。
```
    public static String download(Context context, String docUrl, int size){
        if (SDCardHelper.isSDCardMounted()) {
            if (SDCardHelper.getSDCardAvailableSize() < size * 3) {
                return SD_SPACE_LIMIT;
            } else {
                String dirName = "";
                dirName = SDCardHelper.getSDCardPrivateFilesDir(context, null) + "/patch/";
                File f = new File(dirName);
                if (!f.exists()) {      //判断文件夹是否存在
                    f.mkdir();        //如果不存在、则创建一个新的文件夹
                }
                String fileName = Constants.PATCH_NAME;
                fileName = dirName + fileName;
                File file = new File(fileName);
                if (file.exists()) {    //如果目标文件已经存在
                    file.delete();    //则删除旧文件
                }
                //1K的数据缓冲
                byte[] bs = new byte[1024];
                //读取到的数据长度
                int len;
                try {
                    //通过文件地址构建url对象
                    URL url = new URL(docUrl);
                    //获取链接
                    //URLConnection conn = url.openConnection();
                    //创建输入流
                    InputStream is = url.openStream();
                    //获取文件的长度
                    //int contextLength = conn.getContentLength();
                    //输出的文件流
                    OutputStream os = new FileOutputStream(file);
                    //开始读取
                    while ((len = is.read(bs)) != -1) {
                        os.write(bs, 0, len);
                    }
                    //完毕关闭所有连接
                    os.close();
                    is.close();
                    return DOWNLOAD_SUCCESS;
                } catch (MalformedURLException e) {
                    fileName = null;
                    e.printStackTrace();
                    return URL_CREAT_FAIL;
                } catch (FileNotFoundException e) {
                    fileName = null;
                    e.printStackTrace();
                    return LOAD_FILE_FAIL;
                } catch (IOException e) {
                    fileName = null;
                    e.printStackTrace();
                    return CONNECT_FAIL;
                }
            }
        } else {
            return SD_NO_LOAD;
        }
    }
```

# Demo集成总结

## 一些记录

* 使用Tinker需要兼容原有Application，建议使用注解方式兼容；
* Tinker开放检测补丁载入、合成的监听，建议写一个模块单独处理整个过程并做好容错处理；
* Tinker gradle文件中TinkerId建议设置为VersionCoce，同时对于tinkerPatch.dex.loade的配置应该重点关注及配置；
* Jpush的混淆配置重点关注； 
* 七牛云对多次覆盖上传文件，每次取文件都会取到缓存文件，受平台缓存限制。

## 一些建议
* 建议使用Tinker SDK作为热修复技术基础；
* 设计一套补丁获取机制，可通过客户端主动异步轮训请求和后端主动推送来获取补丁；
* 建议搭建具有保密传输功能的补丁下载后台；
* 重点关注app gradle中tinkerPatch.dex.loader的配置，详情见文档说明。


# 问题与思考

## Tinker已知问题

* Tinker不支持修改AndroidManifest.xml，Tinker不支持新增四大组件
* 在Android N上，补丁对应用启动时间有轻微的影响
* 由于各个厂商的加固实现并不一致，在1.7.6以及之后的版本，tinker不再支持加固的动态更新	
* 由于Google Play的开发者条款限制，不建议在GP渠道动态更新代码
* 不支持部分三星android-21机型，加载补丁时会主动抛出"TinkerRuntimeException:checkDexInstall failed"
* 对于资源替换，不支持修改remoteView。例如transition动画，notification icon以及桌面图标

由于Tinker暂时不支持加固与项目App防修改加密加固冲突，故暂时没有引入在实际项目引入Tinker，待观望，如有更好意见或建议请一定要给出...先谢谢啦~

# 参考文章

> [Tinker官方文档](https://github.com/Tencent/tinker) 
> [美团Android DEX自动拆包及动态加载简介](http://tech.meituan.com/mt-android-auto-split-dex.html)
> [安卓App热补丁动态修复技术介绍](https://github.com/Tencent/tinker/wiki/Tinker-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%89%A9%E5%B1%95) 
















