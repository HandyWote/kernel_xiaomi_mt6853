# 本 fork 说明（HandyWote/kernel_xiaomi_mt6853）

在原仓库基础上，本 fork **只做了一处改动**：在 `arch/arm64/configs/cannon_defconfig`
里打开容器（dockerd）所需的 9 个内核选项，其余代码与配置保持上游原样。

```
CONFIG_SYSVIPC=y
CONFIG_POSIX_MQUEUE=y
CONFIG_IPC_NS=y
CONFIG_CGROUP_DEVICE=y
CONFIG_CGROUP_PIDS=y
CONFIG_TMPFS_XATTR=y
CONFIG_TMPFS_POSIX_ACL=y
CONFIG_BINFMT_MISC=y
CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
```

## 已实机验证的适配环境

| 项 | 值 |
|---|---|
| 机型 | Redmi Note 9 5G（cannon / M2007J22C / MT6853T） |
| 系统 | MIUI 14 `V14.0.6.0.SJECNXM`（Android 12，原厂 vendor 分区未动） |
| 刷入方式 | 解包 boot 镜像替换内核段后 `fastboot flash boot`（保留原 ramdisk，可叠加 Magisk） |
| 内核版本串 | `4.14.336-perf-v0.1-docker`（编译时由 CI 写入 `.scmversion` 决定） |

启动后实测正常：WiFi（自动连上校园网并取到 IP，可访问外网）、移动网络接口、蓝牙、
音频（`mt6853-mt6359` 声卡 / AudioFlinger 输出线程）、传感器、GPS、相机。

## 两点需要知道的行为

1. **vendor 分区里的原厂 `.ko` 不会加载**：原厂模块 vermagic 是 `4.14.186-perf-00243-g9a0d96109bda`，
   与 4.14.336 对不上，`lsmod` 为空。这棵树的 WiFi 等驱动是内建（built-in）的，因此不影响使用；
   若要使用原厂模块，需要另外做版本串伪装 + 模块校验放行。
2. **开机后可能出现“您的设备内部出现了问题”弹窗**：改过 boot 分区、启动校验状态变化所致，
   点掉即可，不影响功能。

## 构建

直接用上游自带的 `.github/workflows/autobuild.yml`（Actions → Auto Kernel Builer → Run workflow），
三条支线都会各出一个 AnyKernel3 卡刷包；`main` 支线的产物即为本 fork 的容器支持版。

# 使用教程
## 1. 自行 git 本项目编译
### 1. 选择你需要的支线
main：主线，默认不编译KernelSU Next<br>
ksunext：默认编译KernelSU Next<br>
ksunext-susfs：默认编译KernelSU Next和SUSFS<br>
请选择好自己需要的支线
### 2. 安装编译所需工具链
推荐使用 debian 12 或 ubuntu 23.04 进行编译
```
sudo apt update && sudo apt install -y build-essential clang-13 lld-13 \
libssl-dev libelf-dev flex bison bc ccache curl git git-lfs gnupg \
gperf imagemagick liblz4-tool libncurses5 libncurses5-dev libsdl1.2-dev \
libxml2 libxml2-utils lzop pngcrush rsync schedtool \
squashfs-tools xsltproc zip zlib1g-dev gcc-13-aarch64-linux-gnu \
gcc-13-arm-linux-gnueabi llvm-13 llvm-13-tools
```
### 3. 开始编译
配置 ccache（二次编译时加速）
```
#开启 ccache
export USE_CCACHE=1

#配置 ccache 缓存大小（根据自己实际情况调整）
ccache -M 20G
```
若编译ksunext或ksunext-susfs支线，需要初始化一下子模块
```
git submodule update --init
```
创建 .config 配置文件
```
make -j$(nproc --all) O=out \
          CC="ccache clang-13" \
          LD=ld.lld-13 \
          CLANG_TRIPLE=aarch64-linux-gnu- \
          CROSS_COMPILE=aarch64-linux-gnu- \
          CROSS_COMPILE_COMPAT=aarch64-linux-gnueabi- \
          CROSS_COMPILE_ARM32=arm-linux-gnueabi- \
          LLVM_IAS=1 \
          KCFLAGS="--target=aarch64-linux-gnu \
                   -Wno-unused-but-set-variable \
                   -Wno-unused-variable -Wno-unused-function -Wno-unused-label" \
          cannon_defconfig
```
开始编译
```
make -j$(nproc --all) O=out \
          CC="ccache clang-13" \
          LD=ld.lld-13 \
          CLANG_TRIPLE=aarch64-linux-gnu- \
          CROSS_COMPILE=aarch64-linux-gnu- \
          CROSS_COMPILE_COMPAT=aarch64-linux-gnueabi- \
          CROSS_COMPILE_ARM32=arm-linux-gnueabi- \
          LLVM_IAS=1 \
          KCFLAGS="--target=aarch64-linux-gnu \
                   -Wno-unused-but-set-variable \
                   -Wno-unused-variable -Wno-unused-function -Wno-unused-label"
```
若一切顺利，编译产物会生成于out/arch/arm64/boot/Image.gz-dtb<br>
使用AnyKernel3打包后进行刷入即可
## 2.使用 Github Actions 编译
本项目已经支持 Github Actions 编译，fork 本项目后，<br>
在 Actions -> All workflows -> Auto Kernel Builder -> 点击 run workflow -> 选择 main 分支 -> 开始编译<br>
（那四个选项可以不用改，只跟显示的内核版本号和AnyKernel3有关）<br>
等待编译完成后会有三个产物，分别是<br>
AnyKernel3-ksunext-susfs.zip、AnyKernel3-ksunext.zip和AnyKernel3-main.zip，
分别是<br>
带KernelSU Next的版本、<br>
带KernelSU Next和SUSFS的版本<br>
以及不带KernelSU Next的版本，<br>
~~（通过Github Actions编译时，ksunext和ksunext-susfs支线将自动补入KPatch-Next）~~<br>
由于某些原因，不能补入KPatch-Next，请在编译完成后自行修补<br>
先解压一下，解压出来就是AnyKernel3卡刷包，然后才能刷入

Linux kernel
============

This file was moved to Documentation/admin-guide/README.rst

Please notice that there are several guides for kernel developers and users.
These guides can be rendered in a number of formats, like HTML and PDF.

In order to build the documentation, use ``make htmldocs`` or
``make pdfdocs``.

There are various text files in the Documentation/ subdirectory,
several of them using the Restructured Text markup notation.
See Documentation/00-INDEX for a list of what is contained in each file.

Please read the Documentation/process/changes.rst file, as it contains the
requirements for building and running the kernel, and information about
the problems which may result by upgrading your kernel.
