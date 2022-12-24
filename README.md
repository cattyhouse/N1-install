# 前言
>建了一个新的分支, 采用新版本的uboot来安装. 原来的 master 分支保留, 里面的内容依然可以参考.

# 步骤概览
> 以下操作都是直接在 N1 上进行
1. 所需资源
1. 分区
1. 安装 userspace
1. 安装 kernel 和 dtb
1. 挂载 u-boot
1. 基本设置
1. 克隆到 mmc

# 所需资源
1. 1个 N1, 自带 armbian 或者 任意其他的 aarch64 linux 系统, 以下均在 N1上操作.
1. 4GB 以上的 U盘
1. 网络

# 分区
>这里暂时只考虑安装到U盘, 后面再讨论如何克隆到内置的 mmc. 
1. 确认 U盘 设备名
    ```sh
    sudo lsblk -f /dev/sd[a-z]
    ```
1. 用 parted 分区
    > 假设上面找到的 U盘为 **/dev/sda(再三确认)**, 将U盘设置为 mbr 分区表, 分为2个区, 第一个为 FAT32 类型, 容量为 256MB, 第二个为 linux 类型, 容量为剩余所有.  
    ```sh
    sudo parted -s -a optimal /dev/sda mklabel msdos mkpart primary fat32 0% 256MiB mkpart primary ext4 256MiB 100%
    ```
    
1. 格式化分区
    > 将分区1格式化为 FAT32, 取名 ARCHBOOT ; 将分区2格式化为 EXT4 取名 ARCHROOT.
    ```sh
    sudo mkfs.vfat -F 32 -n ARCHBOOT /dev/sda1
    sudo mkfs.ext4 -L ARCHROOT /dev/sda2
    ```
1. 挂载分区
    > 将 sda2 挂载到 /mnt, 将 sda1 挂载到 /mnt/boot
    ```sh
    sudo mount /dev/sda2 /mnt
    sudo mkdir -p /mnt/boot
    sudo mount /dev/sda1 /mnt/boot
    ```

# 安装 userspace

> userspace 就是除了内核之外的其他的东西, archlinuxarm 有提供

```sh
cd ~ && curl -L -O http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
sudo bsdtar -xpvf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt # 解压, 比较耗时间, 可以打一局游戏去.
rm -f ArchLinuxARM-aarch64-latest.tar.gz # 阅后即焚
```
> chroot 到 userspace

```sh
# 处理下 DNS
sudo unlink /mnt/etc/resolv.conf
cat /etc/resolv.conf | sudo tee /mnt/etc/resolv.conf
# chroot
cd /mnt
sudo mount -t proc /proc proc
sudo mount --make-rslave --rbind /sys sys
sudo mount --make-rslave --rbind /dev dev
sudo mount --make-rslave --rbind /run run
cd /
sudo chroot /mnt /bin/bash
source /etc/profile
export PS1="(chroot) $PS1" # 做个标记, 提醒你在 chroot 下面 

# 此时就已经进入了 chroot 环境, 以下步骤均在 chroot下进行
```
> 清理一下 userspace

```sh
echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
echo 'en_GB.UTF-8 UTF-8' >> /etc/locale.gen
locale-gen
localectl list-locales # 列出 locale-gen 生成的 locales
localectl set-locale LANG=en_US.UTF-8 LC_TIME=en_GB.UTF-8 # 语言为美国格式, 时间为英国格式(默认显示24小时时间)

passwd root # 设置下 root 密码, 输入不会显示
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config # 允许ssh密码登陆root账户, 出于安全考虑, 建议启动 N1 后,删除这条,采用 ssh key 登陆
userdel -rf alarm # 删除这个用户, 只留下 root
```
> 设置一下网络
```sh
systemctl enable systemd-networkd.service # 如果上面提示没有启动, 就启动它
systemctl disable systemd-resolved.service # 关掉这个服务, 我们已经自行设置 /etc/resolv.conf
cat /etc/systemd/network/eth.network # 检查网络设置, 默认为 dhcp
```
> 
# 安装 kernel 和 dtb

```sh
ping -c 3 www.163.com # 确保网络 OK 
pacman-key --init # 获取 pacman 运行所需要的 key
pacman-key --populate archlinuxarm # 获取 pacman 运行所需要的 key
pacman -Syu # 更新一下系统

# 安装内核
pacman -S --needed linux-aarch64

# 安装 dtb
pacman -S --needed gcc curl dtc git # 安装所需工具
git clone https://github.com/cattyhouse/build_dtb && cd build_dtb
ver=$(pacman -Q linux-aarch64 | cut -d ' ' -f2 | cut -d- -f1)
# 生成 dtb, 一般只需要几秒时间.
./build.sh mainline $ver
sync
```

# 挂载新版 u-boot

>N1 系统自带了 u-boot, 但是只能启动打了 TEXT_OFFSET 补丁的内核, 所以为了启动一个原生的内核, 需要挂载新版本的 u-boot, 所有脚本已经写好, 只需复制到 boot 分区, N1 就可以用自身的 u-boot, 挂载这个新的 u-boot

> 启动原理 第一步 : N1 内置 u-boot -> 优先寻找 U盘 -> 寻找 boot 分区的 s905_autoscript -> 根据这个 s905_autoscript 的内容加载新的 u-boot, 也就是 u-boot.ext

> 启动原理 第二步 : 根据 u-boot.ext 内置的脚本依次执行 : bootcmd -> distro_bootcmd -> boot_targets -> bootcmd_usb0 -> usb_boot -> scan_dev_for_boot_part -> scan_dev_for_boot -> scan_dev_for_extlinux -> boot_extlinux, 然后找到了 extlinux.conf

> 启动原理 第三步 : 根据 extlinux.conf 的设置, 定位 root 分区 -> 寻找 zImage 和 uInitrd 也就是安装 kernel 后生成的 两个文件. 至此, 控制权交给了 linux 内核. 

> 确保已经在 chroot 环境

1. 下载文件
    ```sh
    ping -c 3 www.163.com # 确保网络 OK
    pacman -S --needed git
    cd /tmp && git clone --depth 1 https://github.com/cattyhouse/new-uboot-for-N1
    cd new-uboot-for-N1
    cp -fr * /boot/
    ```
1. 设置 extlinux.conf

    ```sh
    root_uuid=$(lsblk -n -o UUID /dev/sda2)
    sed -i "s/root_uuid/${root_uuid}/" /boot/extlinux/extlinux.conf
    # 确认一下修改是否成功
    grep ${root_uuid} /boot/extlinux/extlinux.conf
    ```
1. 设置 fstab
    ```sh
    root_uuid=$(lsblk -n -o UUID /dev/sda2)
    boot_uuid=$(lsblk -n -o UUID /dev/sda1)
    echo "UUID=${root_uuid} / ext4 rw,relatime 0 1" > /etc/fstab
    echo "UUID=${boot_uuid} /boot vfat rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro 0 2" >> /etc/fstab
    cat /etc/fstab # 检查一下
    ```

# 基本设置
> 重启
```sh
exit # 退出 chroot 环境
cd / # 确保所有的窗口不占用 /mnt
sudo umount -vfR /mnt # target is busy 的错误可以忽略
sudo reboot # 祝您好运.
```
> 登陆
```sh
# 从路由器找到 N1 的 ip, 通过 ssh 登陆
ssh root@n1_ip # 然后输入密码
# 如果接显示器和键盘的话, 直接登陆.

# TODO
```

# 克隆到 MMC
> rsync 可以完美做到 100% 克隆
```sh
# TODO
```

# ext4 on /boot

> just a note, if you don't know what it is, DO NOT run it

```sh
# ref: https://bugcheck.win/2019/03/06/use-ext4-for-boot-partition-on-phicomm-n1/

# we do everything as root so redirect simply works 
((EUID)) && sudo su

# format p1 of mmc1
cd / # make sure we do not block anything
mkdir -p /boot.bk
cp -a /boot/* /boot.bk/ # backup our files
umount -f /boot
fdisk /dev/mmcblk1 # change partition id, i like fdisk
t 1 83 w # manual input one by one
mkfs.ext4 /dev/mmcblk1p1 # format it as ext4
systemctl daemon-reload # fuck systemd
mount /dev/mmcblk1p1 /boot # remount
mv /boot.bk/* /boot/ # cp back our files
rm -rf /boot.bk # clean

# fstab
boot_uuid=$(blkid -o value -s UUID /dev/mmcblk1p1)
printf '%s\n' "UUID=$boot_uuid /boot ext4 rw,relatime,data=writeback 0 2" >> /etc/fstab # add a new fstab line
sed -i '/vfat/d' /etc/fstab # remove any vfat lines
cat /etc/fstab # review

# internal u-boot event
pacman -S uboot-tools # this is a must have
printf '%s\n' '/dev/mmcblk1 0x27400000 0x10000' > /etc/fw_env.config # enable the ability to modify u-boot environment

fw_setenv start_autoscript 'if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript' # start usb before mmc

fw_setenv start_emmc_autoscript 'if ext4load mmc 1 1020000 emmc_autoscript; then autoscr 1020000; fi; if fatload mmc 1 1020000 emmc_autoscript; then autoscr 1020000; fi;' # look for ext4 first, fallback to fat

fw_setenv start_usb_autoscript 'for usbdev in 0 1 2 3; do if ext4load usb ${usbdev} 1020000 s905_autoscript; then autoscr 1020000; fi; if fatload usb ${usbdev} 1020000 s905_autoscript; then autoscr 1020000; fi; done' # same as above, but for usb

# review
fw_printenv | grep -E 'start_autoscript=|start_usb_autoscript=|start_emmc_autoscript=' # always review what have been done

# external uboot config to load mainline kernel
# for mmc
printf '%s\n' 'for mdev in 1 0 2 ; do if ext4load mmc ${mdev} 0x1000000 uboot; then go 0x1000000; fi; if fatload mmc ${mdev} 0x1000000 uboot; then go 0x1000000; fi; done' > /boot/emmc_autoscript.cmd # again, ext4 first, fallback to fat
mkimage -C none -A arm64 -T script -d /boot/emmc_autoscript.cmd /boot/emmc_autoscript

# for usb
printf '%s\n' 'for udev in 0 1 2 3 ; do if ext4load usb ${udev} 0x1000000 uboot; then go 0x1000000; fi; if fatload usb ${udev} 0x1000000 uboot; then go 0x1000000; fi; done' > /boot/s905_autoscript.cmd # same as above
mkimage -C none -A arm64 -T script -d /boot/s905_autoscript.cmd /boot/s905_autoscript

# time to reboot
reboot
```

