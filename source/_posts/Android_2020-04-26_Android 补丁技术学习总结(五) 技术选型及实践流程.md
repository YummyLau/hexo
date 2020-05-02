---
title: Android 补丁技术学习总结(五) 技术选型及实践流程
date: 2020-05-02 18:00:05
comments: true
categories: Android进阶
tags: [热修复]
---

### 方案选型
两年前在旧的团队预研热修复的时候，我们选择了 tinker。现在所在的团队的还是 tinker。对于中小团队而言，我们选择方案一般需要：**“高兼容性，高修复性，免费，社区活跃”**。

* 高兼容性，需要兼容 Android 的所有版本，我们也尝试过 AndFix，QZone 等方案，基本 AndroidN 之后就放弃了；
* 高修复性，除了能修复类场景，资源，so 也需要考虑；
* 免费，一开始 AndFix 其实简单易用，后面转 sophix 后需要收费，就放弃了。如果有金主爸爸可以忽略，sophix 非常简单易用，而且之前的学习总结基本参考的是 sophix 的技术方案，非常优秀；
* 社区活跃，目前 tinker 的 github 升级及维护算还不错。

故我们最终选择以 tinker 作为热修复方案技术框架来实现热修功能。

### 集成与流程

#### Tinker 集成
在我们项目中，tinker 相关的代码是作为 **Service** 层中的一个模块。大概包含以下信息：

* **代码**，包含 tinker 提供的所有库及项目封装的代码，涉及下载，加载，调试，日志上报等场景
* **gradle 脚本**，tinker 配置信息
* **基线资源**，专门的目录，用于存放未加固包，mapping文件，r文件
* **shell 脚本**，打包补丁的脚本，提供给 jenkins 调用，用于读取基线资源联合 tinker 插件进行补丁生成

主端项目由于我们使用 **ApplicationLike** 进行代理，所以是否开启热修复，都需要 tinker 来代理我们的 **Application**。主端根据是否打开热修复功能来 **apply** gradle 脚本及对 **DefaultLifeCycle.flag** 进行开关切换。 

#### 项目实践流程

*项目环境* ： “ 在生产环境中，我们打包产物是通过 **Jenkins** 平台输出，由 **Jenkins** 先生成之后输出到内部测试平台。如需要对外发布，则会在此上传到具有 **CDN** 能力的文件服务器。另外，**CMS平台** 可对补丁信息进行分发，客户端通过读取 **CMS配置信息** 来补丁信息，进而驱动客户端修复行为 ”。

##### (1) release 分支保留 “基线资源”

一般的git开发流程可参考 [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/) 一文，核心的分支概念主要由以下五类分支：

* 主分支 master，发布线上应用及版本 tag
* 开发分支 develop，开发总分支
* 功能分支 feature，版本功能开发测试分支
* 补丁分支 hotfix，紧急 bug 修复分支
* 预发分支release，功能测试回归预发版分支

一般而言，一个版本可能需要开发多个功能，可从 develop 拉取一个该版本的总 feature 分支，然后该总 feature 分支再拉取各个子分支给团队内部人员开发。这样子可尽可能隔离与 develop 交互，尽可能避免或减少分支的合并冲突。
下面以我们团队日常开发分支实践展开，同时区分常规发版及补丁发版来修复紧急 Bug 来梳理整个版本的开发流程，见下图。

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200502/patch_info_4.png" align=center  />

*tip*：如果同一个版本存在多个补丁，比如 release 1.0.0 出现 bug 需要修复，则可衍生出 hotfix 1.0.0.1 作为第一个补丁的分支，hotfix 1.0.0.2 作为第二个补丁分支一次类推。

**在 release 回归结束之后，需要输出发版分支时，Jenkins 打开输出基线资源的配置，基线资源会跟随输输出发布版本 apk 一起发布到内部测试平台。这些资源会通过一个相同的序列化进行关联，也会在命名上体现，我们使用的是 git 提交记录来作为该序列。** 

##### (2) 修复紧急缺陷

从原发布版本对应的 **release** 分支中拉出 **hotfix** 分支，针对紧急缺陷进行修复。同时从内部测试平台下载 **“基线资源”** 存放到规定的目录后，把该分支推送到 **remote**。这里使用的是 *tinkerPatchRelease* 进行补丁合成，所有合成工作逻辑都写在了 **shell 脚本** 中连同项目一起推上远端。


##### (3) 生成补丁上传

**Jenkins** 为生成补丁建立一个 **Job**，每次构建补丁前，把 (2) 对应的分支名写到 **Job** 配置信息中。该 **Job** 执行的时候会先从 **remote** 拉取 **hotfix** 分支，然后执行  **shell 脚本** 对 **基线资源** 进行读取并完成 gradle 脚本的配置，调用 *tinkerPatchRelease* 进行补丁合成，最后对补丁产物进行重命名之后上传到内部测试平台。

##### (4) CMS 配置补丁信息

一个应用与版本，补丁间的关系：

* 一个应用存在多个版本
* 一个应用版本可存在多个补丁，同个版本的补丁可以互相覆盖

根据这个关系，我们需要设计对应的数据结构来承载补丁信息。

定义补丁信息，版本补丁信息，应用补丁信息

```
public class PatchInfo {
    public String appPackageName;
    public String appVersionName;
    public int percent = Constants.VERSION_INVALID;                //灰度或者全量，在(0-10000]之间
    public long version = Constants.VERSION_INVALID;                //补丁版本，有效的版本应该是(1-正无穷)，0为回滚，如果找到patchData下的补丁version匹配，则修复，否则跳过
    public long size;                                               //补丁包大小
    public String desc;                                             //补丁描述
    public long createTime;                                         //补丁创建时间
    public String downloadUrl;                                      //补丁下载链接
    public String md5;			                                    //补丁文件md5												
  }   
  
public class VersionPatchInfo {
    public String packageName;
    public String versionName;
    public long targetPatchVersion;
    public List<PatchInfo> patchList;
}              

public class PatchScriptInfo {
    public String packageName;                              //修复app的包名
    public Map<String, VersionPatchInfo> versionPatchList;  //当前app下所有修复列表，按版本区分
}                             
```
则三者的关系为

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200502/patch_info_1.png"  align=center />

定义一份配置信息文件，用于声明全平台所有版本的补丁信息。

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200502/patch_info_2.png" align=center />

**CMS** 根据上述规则通过配置项来构建配置信息文件，客户端通过 **CMS** 提供的 **api** 来请求这份配置信息文件。

##### (5) 客户端加载补丁

除了主动拉取 **CMS** 配置信息文件外，一般还需要支持被动接收推送信息。

* 被动接收推送，客户端通过接收推动信息来构建配置信息
* 主动拉取配置，通过 CMS 提供的 api 来实时拉取配置信息，进而构建配置信息

无论通过哪种方式来构建配置信息，后续都需要完成以下流程

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200502/patch_info_3.png" align=center />

##### (6) 调试/日志支持

**调试** 除了 IDE 的 deug 之后，还可支持线上应用某些入口基于补丁加载支持，比如说在某些业务无相关的页面如 **“关于页面”** 的某个 *view* 快速点击之后弹出对话框，在对话框输入内部测试码之后就可以进入 “调试界面”。

<img src="https://raw.githubusercontent.com/YummyLau/hexo/master/source/pics/20200502/patch_info_5.jpg" height="790" width="365" align=center />

另外，(4) 中涉及的配置信息下载或补丁下载 downloadUrl，可扩展协议进行支持。

* **cms协议，通过内部的 cms 文件协议来获取文件或者 api 接口来请求**
	`如果url是以cms:开头的协议，固定走驱动走cms后台配置`
* **http/https协议**   
	`如果url是常规http:/https:开头的协议，则默认需要下载。`
* **sdcard协议，以设备的 sdcard 根目录为起点进行检索**
	`如果url是以sdcard: 开头的协议，则默认读取sdcard本地文件，该协议用于测试使用，比如 /sdcard/patch/config.txt`

所以 “调试界面” 在扫描补丁脚本配置的时候，只需要输入满足上述 3 种协议中一种的 url 来获取补丁信息。除此之外，整个加载流程都会定义流程码进行标示,可定义枚举类来支持。

```
public enum ReportStep {

    /**
     * 获取脚本，1开头
     */
    STEP_FETCH_SCRIPT(1, "获取热修复配置脚本"),
    STEP_FETCH_SCRIPT_REMOTE(10, "获取远端配置脚本"),
    STEP_FETCH_SCRIPT_LOCAL(11, "获取本地配置脚本"),
    STEP_FETCH_SCRIPT_CMS(12, "获取CMS配置脚本"),
    STEP_FETCH_SCRIPT_SUCCESS(100, "获取配置成功", Level.DEBUG),
    STEP_FETCH_SCRIPT_FAIL(101, "获取配置失败", Level.ERROR),

    /**
     * 解析脚本，2开头
     */
    STEP_RESOLVING_SCRIPT(2, "解析热修复配置脚本"),
    STEP_RESOLVING_SCRIPT_REMOTE(20, "解析远端配置脚本"),
    STEP_RESOLVING_SCRIPT_LOCAL(21, "解析本地配置脚本"),
    STEP_RESOLVING_SCRIPT_CMS(22, "解析CMS配置脚本"),
    STEP_RESOLVING_SCRIPT_LOCAL_SUCCESS(200, "解析成功", Level.DEBUG),
    STEP_RESOLVING_SCRIPT_LOCAL_FAIL(201, "解析失败", Level.ERROR),
    STEP_RESOLVING_SCRIPT_MISS_CUR_PATCH_VERSION(2000, "当前客户端版本找不到目标补丁", Level.ERROR),
    STEP_RESOLVING_SCRIPT_CUR_PATCH_INVALID(2001, "补丁为无效补丁，补丁配置信息配置错误", Level.ERROR),
    STEP_RESOLVING_SCRIPT_CUR_PATCH_CANT_HIT(2002, "客户端版本目标补丁未命中灰度", Level.ERROR),
    STEP_RESOLVING_SCRIPT_CUR_PATCH_IS_REDUCTION(2003, "目标补丁为回滚补丁", Level.DEBUG),
    STEP_RESOLVING_SCRIPT_CUR_PATCH_HAS_PATCHED(2004, "目标补丁已经被加载过，跳过", Level.DEBUG),
    STEP_RESOLVING_SCRIPT_HAS_SAME_NAME_FILE_BUT_MD5(2005, "本地补丁目录查询到与目标补丁同名的文件，但md5校验失败", Level.ERROR),
    STEP_RESOLVING_SCRIPT_HAS_SAME_NAME_FILE_AND_MATCH_MD5(2006, "本地补丁目录查询到与目标补丁同名的文件，md5校验成功", Level.DEBUG),

    /**
     * 获取补丁，3开头
     */
    STEP_FETCH_PATCH_FILE(3, "获取补丁"),
    STEP_FETCH_PATCH_FILE_REMOTE(30, "从远端获取下载补丁文件"),
    STEP_FETCH_PATCH_FILE_LOCAL(31, "从本地目录获取补丁文件"),
    STEP_FETCH_PATCH_SUCCESS(300, "获取补丁文件成功", Level.DEBUG),
    STEP_FETCH_PATCH_FAIL(301, "获取补丁文件失败", Level.ERROR),
    STEP_FETCH_PATCH_MATCH_MD5(3000, "校验补丁文件 md5 成功", Level.DEBUG),
    STEP_FETCH_PATCH_MISS_MD5(3001, "校验补丁文件 md5 失败", Level.ERROR),
    STEP_FETCH_PATCH_WRITE_DISK_SUCCESS(3002, "补丁文件写入补丁目录成功", Level.DEBUG),
    STEP_FETCH_PATCH_WRITE_DISK_FAIL(3003, "补丁文件写入补丁目录失败", Level.ERROR),


    /**
     * 修复补丁，4开头
     */
    STEP_PATCH(4, "补丁修复"),
    STEP_PATCH_LOAD_SUCCESS(40, "读取补丁文件成功", Level.DEBUG),
    STEP_PATCH_LOAD_FAIL(41, "读取补丁文件失败", Level.ERROR),
    STEP_PATCH_RESULT_SUCCESS(400, "补丁修复成功", Level.DEBUG),
    STEP_PATCH_RESULT_FAIL(4001, "补丁修复失败", Level.ERROR),


    /**
     * 补丁回滚，4开头
     */
    STEP_ROLLBACK(5, "补丁回滚"),
    STEP_ROLLBACK_RESULT_SUCCESS(50, "补丁回滚成功", Level.DEBUG),
    STEP_ROLLBACK_RESULT_FAIL(51, "补丁回滚失败", Level.ERROR);


    public int step;
    public String desc;
    @Level
    public int logLevel;

    ReportStep(int step, String desc) {
        this(step, desc, Level.INFO);
    }

    ReportStep(int step, String desc, int logLevel) {
        this.step = step;
        this.desc = desc;
        this.logLevel = logLevel;
    }
}
```

在补丁流程的每一个节点都进行 Log 日志输出，除了输出到 IDE，“调试界面”外，还需要上传到每个项目的日志服务器中，以便分析线上补丁流程的具体情况。

**关联文章**

* [Android 补丁技术学习总结(一) 冷启动类加载](http://yummylau.com/2020/05/02/Android_2020-04-14_Android%20%E8%A1%A5%E4%B8%81%E6%8A%80%E6%9C%AF%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93(%E4%B8%80)%20%E5%86%B7%E5%90%AF%E5%8A%A8%E7%B1%BB%E5%8A%A0%E8%BD%BD/)
* [Android 补丁技术学习总结(二) 类热替换](http://yummylau.com/2020/05/02/Android_2020-04-09_Android%20%E8%A1%A5%E4%B8%81%E6%8A%80%E6%9C%AF%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93(%E4%BA%8C)%20%E7%B1%BB%E7%83%AD%E6%9B%BF%E6%8D%A2%20/)
* [Android 补丁技术学习总结(三) 资源修复](http://yummylau.com/2020/05/02/Android_2020-04-20_Android%20%E8%A1%A5%E4%B8%81%E6%8A%80%E6%9C%AF%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93(%E4%B8%89)%20%E8%B5%84%E6%BA%90%E4%BF%AE%E5%A4%8D/)
* [Android 补丁技术学习总结(四) so修复](http://yummylau.com/2020/05/02/Android_2020-04-26_Android%20%E8%A1%A5%E4%B8%81%E6%8A%80%E6%9C%AF%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93(%E5%9B%9B)%20so%E4%BF%AE%E5%A4%8D/)
* [Android 补丁技术学习总结(五) 技术选型及实践流程](http://yummylau.com/2020/05/02/Android_2020-04-26_Android%20%E8%A1%A5%E4%B8%81%E6%8A%80%E6%9C%AF%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93(%E4%BA%94)%20%E6%8A%80%E6%9C%AF%E9%80%89%E5%9E%8B%E5%8F%8A%E5%AE%9E%E8%B7%B5%E6%B5%81%E7%A8%8B/)