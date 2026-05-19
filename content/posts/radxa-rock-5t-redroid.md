---
date: '2026-05-19T19:33:55+08:00'
draft: false
title: 'ROCK 5T编译瑞莎官方内核支持Redroid'
---

# 引言
CNflysky大佬在github上提供了编译好的redroid系统镜像和自定义内核，为了能在瑞莎官方内核上运行redroid，需要修改内核配置，基于CNflysky大佬的自定义内核给dma-buf相关功能打补丁，并编译内核。

# 准备工作
一台多核心Linux服务器，根据瑞莎官方文档，安装编译工具和依赖库克隆BSP仓库  
```bash
sudo apt update
sudo apt install -y git qemu-user-static binfmt-support
# Podman (推荐)
sudo apt install -y podman podman-docker
sudo touch /etc/containers/nodocker
# Docker
#sudo apt install docker.io
# 次要功能的可选依赖
sudo apt install -y systemd-container
git clone --recurse-submodules https://github.com/radxa-repo/bsp.git
```
克隆CNflysky大佬的内核仓库:
```bash
git clone https://github.com/CNflysky/linux-rockchip.git --depth=1
```

# 修改内核配置
根据CNflysky仓库提供的envcheck.sh脚本，修改内核配置,修改内核配置文件在`linux/rk2410/0001-common/kconfig.conf`，添加修改以下配置项：
```bash
# 默认是这样的就不用动，如果不是，修改为以下内容
CONFIG_ANDROID_BINDERFS=y
CONFIG_PSI=y
CONFIG_ARM64_VA_BITS=39
CONFIG_MALI_BIFROST=m
```
# 生成补丁
进入克隆好的bsp仓库`bsp`，拉取与配置代码
```bash
cd bsp
./bsp linux rk2410 --no-build
# `--no-build` 仅配置代码不编译
```
创建dma-patch目录，生成补丁文件
```bash
mkdir dma-patch
cd dma-patch
# 软链接瑞莎官方内核
ln -s bsp/.src/linux kernel-radxa
# 软链接CNflysky自定义内核
ln -s linux-rockchip kernel-armbian
#需要打补丁的文件： 
# drivers/dma-buf
# include/linux/dma-buf.h
# include/linux/dma-heap.h
# include/linux/android_kabi.h
# include/linux/dma-fence.h
# include/linux/dma-resv.h
# 生成补丁文件
touch kernel-radxa/include/linux/android_kabi.h
git diff --no-index \
kernel-radxa/drivers/dma-buf \
kernel-armbian/drivers/dma-buf \
> dma_buf.patch
git diff --no-index \
kernel-radxa/include/linux/dma-heap.h \
kernel-armbian/include/linux/dma-heap.h \
>> dma_buf.patch
git diff --no-index \
kernel-radxa/include/linux/android_kabi.h \
kernel-armbian/include/linux/android_kabi.h \
>> dma_buf.patch
git diff --no-index \
kernel-radxa/include/linux/dma-fence.h \
kernel-armbian/include/linux/dma-fence.h \
>> dma_buf.patch
git diff --no-index \
kernel-radxa/include/linux/dma-resv.h \
kernel-armbian/include/linux/dma-resv.h \
>> dma_buf.patch
git diff --no-index \
kernel-radxa/include/linux/dma-buf.h \
kernel-armbian/include/linux/dma-buf.h \
>> dma_buf.patch
```
# 应用补丁
```bash
cd kernel-radxa
git apply -p2 --check ~/dma-patch/dma_buf.patc
git apply -p2 ~/dma-patch/dma_buf.patch
```

# 编译内核
```bash
cd bsp
./bsp --no-prepare-source linux rk2410 -r 999
# 参数说明：
# --no-prepare-source   # 首次编译不需要加该参数，加该参数是为了使用本地修改进行编译，如果不加这个参数将会从 Radxa kernel 仓库同步最新代码并覆盖本地修改
# -r 999                # 指定内核的版本号为 999，以优先使用
```
编译完成后会在当前目录生成许多deb包,只需要安装下面两个deb即可  
(软件包文件名会根据 profile 和 -r 参数变化)
```bash
linux-headers-6.1.84-999-rk2410_6.1.84-999_arm64.deb  
linux-image-6.1.84-999-rk2410_6.1.84-999_arm64.deb
```
将上面两个 deb 包复制到板子上使用 dpkg 指令安装即可完成内核安装
```bash
sudo dpkg -i linux-headers-6.1.84-999-rk2410_6.1.84-999_arm64.deb
sudo dpkg -i linux-image-6.1.84-999-rk2410_6.1.84-999_arm64.deb
sudo reboot
```
# 系统调整
根据瑞莎官方文档切换 GPU 驱动章节安装Mali用户空间驱动
参考链接：[切换 GPU 驱动](https://docs.radxa.com/rock5/rock5t/radxa-os/mali-gpu)  
添加Linux启动参数 psi=1
```bash
sudo nano /etc/kernel/cmdline   # 添加或修改你所需要的选项

root=UUID=x console=ttyFIQ0,1500000n8 quiet splash loglevel=4 rw earlycon consoleblank=0 console=tty1 coherent_pool=2M irqchip.gicv3_pseudo_nmi=0 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory swapaccount=1 kasan=off psi=1

sudo u-boot-update  # 执行命令以更新启动参数
```
# 运行Redroid
参考链接：[CNflysky/redroid-rk3588](https://github.com/CNflysky/redroid-rk3588)
# 存在的问题
- 目前Android 12(12.0.0-latest)镜像正常使用，Android 13(13.0.0-latest),LineageOS 20 (lineage-20)镜像使用scrcpy连接存在编码问题，无法正常显示屏幕内容，会花屏断连，系统运行正常，估计和驱动有关。