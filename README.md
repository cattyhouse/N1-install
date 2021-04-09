# 前言
>建了一个新的分支, 采用新版本的uboot来安装. 原来的 master 分支保留, 里面的内容依然可以参考.

# 步骤概览
> 以下操作都是直接在 N1 上进行
1. 所需资源
1. 分区
1. 挂载 u-boot
1. 安装 userspace
1. 安装 kernel
1. 基本设置
1. 克隆到 mmc

# 所需资源
1. 1个 N1, 自带 armbian 或者 任意其他的 aarch64 linux 系统
2. 4GB 以上的 U盘
3. 网络

# 分区
>这里暂时只考虑安装到U盘, 后面再讨论如何克隆到内置的 mmc. 
1. 确认 U盘 设备名
    ```
    sudo parted --list 2>/dev/null | grep "/dev/sd.*"
    ```
1. 用 parted 分区
    > 一句话解释 : 假设上面找到的 U盘为 **/dev/sda(再三确认)**, 将U盘设置为 mbr 分区表, 分为2个区, 第一个为 FAT32 类型, 容量为 128MB, 第二个为 linux 类型, 容量为剩余所有.  
    ```
    sudo parted -s -a optimal /dev/sda mklabel msdos mkpart primary fat32 0% 128MiB mkpart primary ext4 128MiB 100%
    ```
    
2. 格式化分区
    > 一句话解释 : 将分区1格式化为 FAT32, 取名 ARCHBOOT ; 将分区2格式化为 EXT4 取名 ARCHROOT.
    ```
    sudo mkfs.vfat -F 32 -n ARCHBOOT /dev/sda1
    sudo mkfs.ext4 -L ARCHROOT /dev/sda2
    ```
3. 挂载分区
    > 一句话解释 : 将 sda2 挂载到 /mnt, 将 sda1 挂载到 /mnt/boot
    ```
    sudo mount /dev/sda2 /mnt
    sudo mkdir -p /mnt/boot
    sudo mount /dev/sda1 /mnt/boot
    ```

# 挂载 u-boot

>N1 系统自带了 u-boot, 但是只能启动打了 TEXT_OFFSET 补丁的内核, 所以为了启动一个原生的内核, 需要挂载新版本的 u-boot, 所有脚本已经写好, 只需复制到 /boot, N1 就可以用自身的 u-boot, 挂载这个新的 u-boot

> 启动原理 第一步 : N1 内置 u-boot -> 优先寻找 U盘 -> 寻找 boot 分区的 s905_autoscript -> 根据这个 s905_autoscript 的内容加载新的 u-boot, 也就是 u-boot.ext

> 启动原理 第二步 : 根据 u-boot.ext 内置的脚本依次执行 : bootcmd -> distro_bootcmd -> boot_targets -> bootcmd_usb0 -> usb_boot -> scan_dev_for_boot_part -> scan_dev_for_boot -> scan_dev_for_extlinux -> boot_extlinux, 然后找到了 extlinux.conf

> 启动原理 第三步 : 根据 extlinux.conf 的设置, 定位 root 分区 -> 寻找 zImage 和 uInitrd 也就是安装 kernel 后生成的 两个文件. 至此, 控制权交给了 linux 内核. 

1. 下载文件
    ```
    cd ~ && git clone https://github.com/cattyhouse/new-uboot-for-N1
    cd new-uboot-for-N1
    sudo cp -r * /mnt/boot/
    cd ~ && sudo rm -rf new-uboot-for-N1
    ```
3. 设置 extlinux.conf

    ```
    root_uuid=$(lsblk -f  | grep sda2 | xargs | cut -d ' ' -f5)
    sudo sed -i "s/root_uuid/${root_uuid}/" /mnt/boot/extlinux/extlinux.conf
    # 确认一下修改是否成功
    grep UUID /mnt/boot/extlinux/extlinux.conf
    # 清理一下临时变量
    unset root_uuid
    ```

# 安装 userspace
> userspace 就是除了内核之外的其他的东西, archlinuxarm 有提供
```sh
cd ~ && curl -OL http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
sudo bsdtar -xpvf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt/
```

# 安装 kernel
> kernel 是我编译的, 删减了 N1 上没有的东西, 比如 PCIE 等. 

# 基本设置
> 一些基本设置

# 克隆到 MMC
> rsync 可以完美做到 100% 克隆