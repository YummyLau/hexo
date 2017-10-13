>由于之前的渣本试过一次裸ubuntu编译Android源码，一次则用虚拟机。过程中参考了官网/网络博客的教程，失败了好几次，也成功了两次。上一次买了个稍微好一点的笔记本，也裸ubuntu编译过一次7.0的源码，后面因为其他工作的需求，暂且换成window系统，再一次重新折腾编译源码。整理下之前的笔记及参考资料，写下记录供需要者参考，减少踩坑。   
题外话：把之前在ubuntu编译的源码拉到windowAS阅读，虽然能够根据方法栈不断深入查看方法调用的逻辑，但是无法调试且难以修改原生应用。所以强烈建议在ubuntu上折腾，把整个过程弄得够熟悉就行了，也对整个Android系统有个大致的了解。

# 准备环境
* 编译环境选择`Ubuntu`
	* 我的电脑 `Window` `64bit`，`8G`内存，`256`固态硬盘
	* [ubuntu-16.04.3-desktop-amd64](http://releases.ubuntu.com/16.04/ubuntu-16.04.3-desktop-amd64.iso.torrent?_ga=2.264336523.581782411.1501948873-450244967.1501948873) 运行于`VMware`，分配`4G`内存，`100G`硬盘空间
	* 若是其他硬件基础，可参考[源代码-下载和构建-概要-硬件要求](https://source.android.com/source/requirements)

* Git安装  

```
$ sudo apt-get install git  
$ git config --global user.email "yummyl.lau@gmail"
$ git config --global user.name "yummyLau"
```

#  下载源码
* [选择源码分支](https://source.android.com/source/build-numbers)，这里我选择`android-6.0.1_r66`分支作为下载版本目标。
* 安装 [Repo
](https://source.android.com/source/downloading)  

```
$ mkdir ~/bin
$ PATH=~/bin:$PATH
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
$ chmod a+x ~/bin/repo
```
* 弃用官方下载源码方法，改用[清华大学开源软件镜像](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)  
这里我选择传统的初始化方法下载源码，创建存放源码的`source`目录，取名自定。    

```
$ mkdir source
$ cd source
$ touch auto_asyn_source.sh
$ chmod +x auto_asyn_source.sh
```
新建`auto_asyn_source.sh`是用来保证执行`repo编译环境选择
 sync`命令同步代码如果失败，则可以自动重试，参考的是[自己动手编译最新Android源码及SDK](http://blog.csdn.net/dd864140130/article/details/51718187)一文的做法，感谢该博主。  
写入shell脚本自动同步内容：   

```
$ vim auto_asyn_source.sh
```
把下面的代码复制进去并保存 

```
PATH=~/bin:$PATH
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-6.0.1_r66
repo sync
while [$? = 1]: do
echo "=========download failed,again============"
sleep 5
repo sync
done
```
其中`android-6.0.1_r66`是我选择下载的源码分支，可替换成其他。然后添加修改Repo的url  

```
$ cd ~/
$ vim .bashrc
```
把下面的链接添加进入并保存  

```
REPO_URL = 'https://gerrit-google.tuna.tsinghua.edu.cn/git-repo'
```
最后，跑一下脚本下载，静心等待呗...  

```
$ cd source
$ ./auto_asyn_source.sh
```
**Tip**：过程中我的虚拟机比较卡，可能会出现卡死的现象=。=，只需要`Ctrl+C`结束之后重新执行脚本就行了。
由于上班折腾下很久才下载完，和下载环境的网络有关，下载完成后大概有`58G`大小。

```
yummylau@ubuntu:~/source$ du -h --max-depth=1
53M		./libcore
11M		./dalvik
124M	./device
128K	./abi
112M	./hardware
31M		./system
16M		./build
27G		./out					//编译后生成的
308M	./development
84M		./ndk
1.5G	./frameworks
30M		./docs
44G		./.repo
948K	./platform_testing
686M	./tools
2.6G	./external
278M	./developers
6.5G	./prebuilts
408M	./packages
900K	./pdk
31M		./sdk
498M	./cts
29M		./bionic
29M		./art
5.7M	./bootable
232K	./libnativehelper
83G	.

```

#  选择编译目标
使用`lunch`选择要编译的目标。根据官网教程。该命令表示针对模拟器进行完整编译，并且所有调试功能均处于启用状态。 如果没有提供任何参数就运行命令，`lunch`将提示您从菜单中选择一个目标，所有编译目标都采用`BUILD-BUILDTYPE`形式。
这里，我输入以下指令选择编译目标。

```
$ lunch aosp_arm-eng
```
解释下上述名下的参数，`aosp_arm-eng`即`BUILD-BUILDTYPE`形式，分为`BUILD`和`BUILDTYPE`两部分。  
  
* BUILD为特定功能的组合代号，如`aosp_arm`为支持`32位ARM`架构，也有`64位ARM`架构，`X86`也有。关于CPU`ARM`和`X86`架构，可以看下这篇[分不清ARM和X86架构，别跟我说你懂CPU！](https://zhuanlan.zhihu.com/p/21266987)科普文。
* BUILDTYPE为编译类型，主要用于指定系统权限，有以下三种：  
	* `user`，权限受限，适用于生产环境；
	* `userdebug`，与“user”类似，但具有 root 权限和可调试性；是进行调试时的首选编译类型；
	* `eng`，具有额外调试工具的开发配置。  

这里我选择`aosp_arm-eng`作为编译目标进行编译。

#  开始编译
官方使用`make`来编译任何代码。Make可以借助 -jN 参数处理并行任务，通常使用的任务数 N 介于编译时所用计算机上硬件线程数的 1-2 倍之间。由于我用的是虚拟机，主机是`司核四线程`,虽然我`Ubuntu`虚拟机分配了2个处理器，每个处理器核数为2。一开始我用`make -j4`编译，可卡死我了，不小心点击了清理内存和高CPU程序，把我虚拟机直接杀了，后面直接使用`make`编译，大家随自己设备决定即可。  
`window`平台下可在运行中输入`wmic`,在新窗口中输入`cpu get`即可查看物理CPU数、CPU核心数、线程数。  
  
* Name:表示物理CPU数
* NumberOfCores：表示CPU核心数
* NumberOfLogicalProcessors：表示CPU线程数    

运行`make`使用单线程编译。  

```
$ make
```
慢悠悠看了几集冰与火之歌，大概编译了4个多小时，就搞定了，编译后的产物有`out`目录中。

```
yummylau@ubuntu:~/source/out$ du -h --max-depth=1
8.5G	./host
18G		./target
27G	.
```
其中各个目录的内容为：  
`/out/host/`:该目录下包含了针对主机的 Android 开发工具的产物。即 SDK 中的各种工具，例如：emulator，adb，aapt 等。  
`/out/target/common/`：该目录下包含了针对设备的共通的编译产物，主要是 Java 应用代码和 Java 库。
`/out/target/product/<product_name>/`：包含了针对特定设备的编译结果以及平台相关的 C/C++ 库和二进制文件。其中，<product_name>是具体目标设备的名称，产物如下图。

```
yummylau@ubuntu:~/source/out/target/product$ cd generic/
yummylau@ubuntu:~/source/out/target/product/generic$ ls
android-info.txt  hardware-qemu.ini         symbols
cache             installed-files.txt       system
cache.img         obj                       system.img
clean_steps.mk    previous_build_config.mk  userdata.img
data              ramdisk.img               userdata-qemu.img
dex_bootjars      recovery
gen               root
yummylau@ubuntu:~/source/out/target/product/generic$ 
```
注意下，上述有三个核心的镜像文件：`system.img`，`ramdisk.img`和`userdata.img`。  
`system.img`：包含了 Android OS 的系统文件，库，可执行文件以及预置的应用程序，将被挂载为根分区。  
`ramdisk.img`：在启动时将被 Linux 内核挂载为只读分区，它包含了 /init 文件和一些配置文件。它用来挂载其他系统镜像并启动 init 进程。  
`userdata.img`：将被挂载为 /data，包含了应用程序相关的数据以及和用户相关的数据。  
关于如何理解Android的Build系统，推荐阅读[IBM-理解 Android Build 系统](https://www.ibm.com/developerworks/cn/opensource/os-cn-android-build/)。

# 运行编译系统
在源码个目录重新执行  

```
yummylau@ubuntu:~/source$ source build/envsetup.sh
yummylau@ubuntu:~/source$ lunch aosp_arm-eng 			//选择之前的编译目标,如果还在当前编译环境，则这前两句不必执行
yummylau@ubuntu:~/source$ emulator						
```
在之前的编译流程会自动将模拟器添加到您的路径中，`emulator`加载的内核所在的目录为 

```
yummylau@ubuntu:~/source/prebuilts/qemu-kernel/arm$ ls
2.6          kernel-qemu-armv7     README      vmlinux-qemu
kernel-qemu  LINUX_KERNEL_COPYING  rebuild.sh  vmlinux-qemu-armv7
```
其中，`kernel-qemu`为加载的内核。
同时在`:~/source/out/target/product/generic`中生成的`system.img`，`ramdisk.img`和`userdata.img`也被载入了。  
最后，我们就静静地等待系统启动吧~





