此WIKI来源自 **[HuaweiP10-GSI-And-Modify-Or-Support-KernelSU-Tutorial](https://github.com/Coconutat/HuaweiP10-GSI-And-Modify-Or-Support-KernelSU-Tutorial)**   
此文档遵循 **[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh)** 许可协议

# 如何为华为4.9版本内核集成KernelSU  
很多朋友希望能对华为老设备集成KernelSU，但又苦于不知如何下手，于是四处求人希望一些有能力编译的朋友去编译一个。  
正巧看到[KernelSU旧内核编译实践教程](https://www.bilibili.com/video/BV1cX4y127gQ)这个视频，  
里面只是简单提到华为的开源的内核在哪里下载，提示了一些对于小米的编译过程。  
本着授人与鱼不如授人以渔的基本方针，这里会告诉大家如何为华为4.9系列(EMUI 9以及EMUI 9.1.0)内核集成KernelSU。  
***   
### 第一节：EMUI 9的版本  
首先，你需要知道你的系统版本和内核版本。  
一般可以在你的设置里能找到。  
比如华为P10的设备的系统版本是EMUI 9.0.1.179，内核版本是4.9.111。
如果是系统是EMUI 9.1.0.210，内核版本是4.9.148。  
华为在EMUI 9的时期推出过9.0.1和9.1.0这两个大版本。  
简单的区别是9.0.1还是ext2的文件系统，9.1.0的是erofs文件系统。  
官方开源的这两个版本内核是有很大区别的。  
搞清楚系统的版本和内核的版本有助于接下来处理内核。  
***  
### 第二节：获取内核  
前置资源：  
1. [华为开源资源发布中心](https://consumer.huawei.com/en/opensource/)  
2. [AArch64交叉编译器_安卓9版本](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/+archive/refs/heads/pie-release.tar.gz)  
3. 一台搭载Linux开发版系统的实体电脑或者虚拟机。  

首先明确手机的开发代号，在手机设置里面和系统版本挨着。  
比如华为P10是VTR，P10 Plus是VKY。后面的-AL00之类的是华为为了区分在哪个地区售卖的地区编码。  
之后你可以去**华为开源资源发布中心**里搜索手机的开发代号，然后获得内核资源。  
这里注意你获取的内核的对应系统。华为是有标注的。  
这里我们讨论的是EMUI 9/9.1.0的。  
EMUI 5/8/10及其以上都不在讨论范围。  
当你下载好以后，解压你的内核文件，注意内核在kernel文件夹里。  
***  
### 第三节：处理内核
华为内核和其它手机内核有显著的区别。毕竟其他设备可能是高通或者联发科，而华为用的是海思麒麟。  
所以内核有更多的碎片化趋势，因为包含大量华为自定义的代码。  
所以接下来，我们需要处理内核的**defconfig**文件。  
路径是 arch/arm64/configs/XXXXXXX_defconfig。  
因为不同机型的defconfig文件不一样。所以，前面用XXXXXXX代替了。  
这个文件是用来确认哪些内核组建需要编译，哪些不需要编译。  
   
 
**这里我们需要处理以下内容**： 
> 有些可能没有，如果有就做改动，没有则无需。

`CONFIG_HISI_PMALLOC=y`  
`CONFIG_HIVIEW_SELINUX=y  `  
`CONFIG_HISI_SELINUX_EBITMAP_RO=y  `  
`CONFIG_HISI_SELINUX_PROT=y  `  
`CONFIG_HISI_RO_LSM_HOOKS=y  `  
`CONFIG_INTEGRITY=y`      
`CONFIG_INTEGRITY_AUDIT=y`    
`CONFIG_HUAWEI_CRYPTO_TEST_MDPP=y  `  
`CONFIG_HUAWEI_SELINUX_DSM=y  `  
`CONFIG_HUAWEI_HIDESYMS=y  `  
`CONFIG_HW_SLUB_SANITIZE=y  `  
`CONFIG_HUAWEI_PROC_CHECK_ROOT=y  `  
`CONFIG_HW_ROOT_SCAN=y  `  
`CONFIG_HUAWEI_EIMA=y  `  
`CONFIG_HUAWEI_EIMA_ACCESS_CONTROL=y  `  
`CONFIG_HW_DOUBLE_FREE_DYNAMIC_CHECK=y  `  
`CONFIG_HKIP_ATKINFO=y  `  
`CONFIG_HW_KERNEL_STP=y`  
`CONFIG_HISI_HHEE=y`    
`CONFIG_HISI_HHEE_TOKEN=y`    
`CONFIG_HISI_DIEID=y`    
`CONFIG_HISI_SUBPMU=y`   
`CONFIG_TEE_ANTIROOT_CLIENT=y`  
`CONFIG_HWAA=y`   


这些内容需要改成如下格式：  
`# CONFIG_XXXXXX is not set`  
例如：   
`# CONFIG_HW_ROOT_SCAN is not set`  
这个改动的含义是不编译这些模块。    

**可选部分**：
把  
`# CONFIG_SECURITY_SELINUX_DEVELOP is not set `  
改为  
`CONFIG_SECURITY_SELINUX_DEVELOP=y`  
改动此处是方便开机的时候手机SELinux默认是Permissive状态。  
> 注：在集成KernelSU的情况下，此处仅限刷入高于安卓9以上的GSI系统的情况下。原因会在下面解释。  
  
关闭AVB验证：  
`CONFIG_DM_VERITY=y`  
`CONFIG_DM_VERITY_AVB=y`  
改为  
`# CONFIG_DM_VERITY=y is not set`  
`# CONFIG_DM_VERITY_AVB=y is not set`  
  

Debug用：   
在不理解以下关于SELinux安全选项的部分时，请直接跳过，不要修改。    
关于SELinux安全选项：  
`CONFIG_SECURITY_SELINUX_BOOTPARAM`  
> 添加"selinux"内核引导参数.以允许在引导时使用'selinux=0'禁用SELinux或'selinux=1'启用SELinux.  
  

`CONFIG_SECURITY_SELINUX_BOOTPARAM_VALUE`   
> 此选项设置内核参数的默认值 'selinux'，允许SELinux在启动时禁用。 如果这个选项设置为0，SELinux内核参数将默认为0，设备在启动时禁用SELinux。 如果此选项为 设置为1，SELinux内核参数将默认为1，在启动时启用SELinux。  
  

`CONFIG_SECURITY_SELINUX_CHECKREQPROT_VALUE`  
> 内核引导参数"checkreqprot"的默认值.设为"0"表示默认检查内核要求执行的保护策略,设为"1"表示默认检查应用程序要求执行的保护策略.此值还可以在运行时通过/selinux/checkreqprot修改.不确定的选"1"。  
  
  
  
***  
### 第四节：集成KernelSU  
因为华为的kprobe工作并不正常。所以请按照官方指南去修改内核源码。  
官方指南：[如何为非 GKI 内核集成 KernelSU](https://kernelsu.org/zh_CN/guide/how-to-integrate-for-non-gki.html)  
请直接跳到**手动修改内核源码**部分。  
同步源码的命令请使用开发版本，即这个：  
`curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main`  
修改hooks.c：  
[commit](https://github.com/sticpaper/android_kernel_xiaomi_msm8998-ksu/commit/09a4672c0f521bf6b05daf24b207b125830a6fc5)  
可选：针对EMUI9/9.1.0 SELinux强制状态导致KernelSU不工作：    
[commit](https://github.com/Coconutat/android_kernel_huawei_ravel_KernelSU/commit/f67307c967280d9b863058e47bae7611c8bc3db9)  
参考第166行。  
***  
### 第五节：编译  
这个部分没啥好说的，可以参考网上很多教程。这里简单说一下就行。  
安装依赖：
`sudo apt install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev libgl1-mesa-dev libxml2-utils xsltproc unzip bc`    
设置环境变量：  
`export ARCH=arm64`  
`export PATH=$PATH:/media/coconutat/Files/Downloads/Github/android_kernel_huawei_ravel_KernelSU`  
`android_kernel_huawei_ravel_KernelSU/aarch64-linux-android-4.9/bin`  
`export CROSS_COMPILE=aarch64-linux-android-`  
这里第一行是声明你要编译arm64架构的内核。  
第二行，第三行是声明你的交叉编译器的路径，需要根据你自己的路径进行修改。  
   
编译命令：   
`make ARCH=arm64 O=out XXXXXX_defconfig`  
`make ARCH=arm64 O=out -j8`  
这里第一行是声明你编译的内核的defconfig，需要根据你自己的defconfig名称进行修改。   
第二行是声明编译的线程数，基于CPU核心数量x2即可，我是4核心，所以是8。  

之后静静等待编译完成即可。  
***  
### 第六节：打包内核  
如果内核编译成功，在out/arch/arm64/boot/路径下会有一个Image.gz文件，这就是内核了。  
我们需要把它复制到内核源码下的tools文件夹，你在里面能找到一个叫**pack_kernerimage_cmd.sh**的脚本。  
我们需要修改它，我以华为开源的荣耀Note10的内核里面的打包参数举例：  
`
#!/bin/bash
./mkbootimg --kernel kernel --base 0x0 --cmdline "loglevel=4 initcall_debug=n page_tracker=on unmovable_isolate1=2:192M,3:224M,4:256M printktimer=0xfff0a000,0x534,0x538 androidboot.selinux=enforcing buildvariant=user" --tags_offset 0x07A00000 --kernel_offset 0x00080000 --ramdisk_offset 0x07C00000 --header_version 1 --os_version 9 --os_patch_level 2020-01-01  --output kernel.img
`  
  
--kernel kernel 这部分指的是内核文件的位置。假设你把内核复制进tools文件夹了，那应该修改成：  
`--kernel Image.gz`    
--output kernel.img 这部分指的是内核打包后的名字，如果你想更改成你喜欢名字，可以改成：  
`--output KernelSU_kernel.img`    
  
可选：   
如果你在上面更改defconfig的时候开启了**CONFIG_SECURITY_SELINUX_DEVELOP**，  
那么你可以在  
`androidboot.selinux=enforcing`  
这部分改成：  
`androidboot.selinux=permissive`  
这样开机的时候手机SELinux默认是Permissive状态。  

打包内核：  
`bash pack_kernerimage_cmd.sh`  
这样你的文件夹里就会多出一个img文件了。  
这个就是你可以刷入的内核了。  
***  
### 结尾  
到这里基本就神功大成了。  
这里要解释下为什么SELinux工作不正常的问题。  
这是因为KernelSU的ksud.c文件无法对低于安卓10的系统正确处理init以及应用KernelSU修改的SELinux规则导致的。  
