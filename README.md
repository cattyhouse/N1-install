# 前言
>建了一个新的分支, 采用新版本的uboot来安装. 原来的 master 分支保留, 里面的内容依然可以参考.

# 步骤概览
1. 所需资源
1. 分区
1. 挂载 u-boot
1. 安装 userspace
1. 安装 kernel
1. 基本设置
1. 克隆到 mmc

# 所需资源
1. 任意 aarch64 的 linux 系统
    > 如果采用 x86_64 系统的话, 后面用到的 chroot 方面需要用到 qemu-user-static 和 binfmt-qemu-static 这两个包. 具体方法 [参考这个](https://unix.stackexchange.com/questions/41889/how-can-i-chroot-into-a-filesystem-with-a-different-architechture). aarch64系统 请忽略这个.
2. 4GB 以上的 U盘
3. 网络

# 分区
>这里暂时只考虑安装到U盘, 后面再讨论如何克隆到内置的 mmc. 
1. 用 parted 分区
2. 格式化分区
3. 挂载分区

# 挂载 u-boot
>N1 系统自带了 u-boot, 但是只能启动打了 TEXT_OFFSET 补丁的内核, 所以为了启动一个原生的内核, 需要挂载新版本的 u-boot, 所有脚本已经写好, 只需复制到 /boot, N1 就可以用自身的 u-boot, 挂载这个新的 u-boot
1. 下载文件 
git clone
2. 复制
3. 设置UUID 

# 安装 userspace
> userspace 就是除了内核之外的其他的东西, archlinuxarm 有提供

# 安装 kernel
> kernel 是我编译的, 删减了 N1 上没有的东西, 比如 PCIE 等. 

# 基本设置
> 一些基本设置

# 克隆到 MMC
> rsync 可以完美做到 100% 克隆