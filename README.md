# 前言
>建了一个新的分支, 采用新版本的uboot来安装. 原来的 master 分支保留, 里面的内容依然可以参考.

# 步骤概览
> 以下操作都是直接在 N1 上进行
1. 所需资源
1. 分区
1. 安装 userspace
1. 安装 kernel
1. 挂载 u-boot
1. 基本设置
1. 克隆到 mmc

# 所需资源
1. 1个 N1, 自带 armbian 或者 任意其他的 aarch64 linux 系统, 以下均在 N1上操作.
2. 4GB 以上的 U盘
3. 网络

# 分区
>这里暂时只考虑安装到U盘, 后面再讨论如何克隆到内置的 mmc. 
1. 确认 U盘 设备名
    ```sh
    sudo parted --list 2>/dev/null | grep "/dev/sd.*"
    ```
1. 用 parted 分区
    > 假设上面找到的 U盘为 **/dev/sda(再三确认)**, 将U盘设置为 mbr 分区表, 分为2个区, 第一个为 FAT32 类型, 容量为 128MB, 第二个为 linux 类型, 容量为剩余所有.  
    ```sh
    sudo parted -s -a optimal /dev/sda mklabel msdos mkpart primary fat32 0% 128MiB mkpart primary ext4 128MiB 100%
    ```
    
2. 格式化分区
    > 将分区1格式化为 FAT32, 取名 ARCHBOOT ; 将分区2格式化为 EXT4 取名 ARCHROOT.
    ```sh
    sudo mkfs.vfat -F 32 -n ARCHBOOT /dev/sda1
    sudo mkfs.ext4 -L ARCHROOT /dev/sda2
    ```
3. 挂载分区
    > 将 sda2 挂载到 /mnt, 将 sda1 挂载到 /mnt/boot
    ```sh
    sudo mount /dev/sda2 /mnt
    sudo mkdir -p /mnt/boot
    sudo mount /dev/sda1 /mnt/boot
    ```

# 安装 userspace

> userspace 就是除了内核之外的其他的东西, archlinuxarm 有提供

```sh
cd ~ && curl -OL http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
sudo bsdtar -xpvf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt/ # 解压, 比较耗时间, 可以打一局游戏去.
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
source ~/.bashrc # 如果提示无此文件, 并没有关系.
export PS1="(chroot) $PS1" # 做个标记, 提醒你在 chroot 下面 

# 此时就已经进入了 chroot 环境, 以下步骤均在 chroot下进行
```
> 清理一下 userspace

```sh
echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
echo 'en_GB.UTF-8 UTF-8' >> /etc/locale.gen
locale-gen
localectl list-locales # 列出 locale-gen 生成的 locales
localectl set-locale en_US.UTF-8 # 设置为 英文美国

passwd root # 设置下 root 密码, 输入不会显示
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config # 允许ssh密码登陆root账户, 出于安全考虑, 建议启动 N1 后,删除这条,采用 ssh key 登陆
userdel -rf alarm # 删除这个用户, 只留下 root
```
> 设置一下网络
```sh
systemctl is-enabled systemd-networkd.service # 确保这个服务已经设置为开机启动
systemctl enable systemd-networkd.service # 如果上面提示没有启动, 就启动它
systemctl disable --now systemd-resolved.service # 关掉这个服务, 我们已经自行设置 /etc/resolv.conf
cat /etc/systemd/network/eth.network # 检查网络设置, 默认为 dhcp

# 设置网卡地址
text=whatever # whatever 可以替换成任意字符
macaddr=$(echo $text | md5sum | sed 's/^\(..\)\(..\)\(..\)\(..\)\(..\).*$/02:\1:\2:\3:\4:\5/')
echo "[Link]" >> /etc/systemd/network/eth.network
echo "MACAddress=${macaddr}" >> /etc/systemd/network/eth.network
unset text macaddr
```
> 
# 安装 kernel
> kernel 是我编译的, 删减了 N1 上没有的东西, 比如 PCIE 等. 
```sh
ping -c 3 www.163.com # 确保网络 OK 

pacman-key --init # 获取 pacman 运行所需要的 key
pacman-key --populate archlinuxarm # 获取 pacman 运行所需要的 key
pacman -Syu # 更新一下系统

pacman -Q | grep linux-aarch64 # 看看是否有安装官方的 kernel
pacman -Rcsun linux-aarch64 # 删除官方的 kernel
pacman -Rcsun linux-aarch64-headers # 删除官方的 headers, 如果存在的话. 

cd /tmp
ver=$(curl -s https://kr1.us.to/kernel/ | grep "linux-phicomm-n1-headers.*pkg.tar.xz" | cut -d \" -f2 | sort -rV | head -n1 | cut -d \- -f5) # 找到最新的 kernel 版本
curl -OL https://kr1.us.to/kernel/linux-phicomm-n1-${ver}-1-aarch64.pkg.tar.xz # 下载 kernel
curl -OL https://kr1.us.to/kernel/linux-phicomm-n1-headers-${ver}-1-aarch64.pkg.tar.xz # 下载 headers
pacman -U *.pkg.tar.xz # 安装
sync # 确保文件写入
unset ver # 清理
```
# 挂载 u-boot

>N1 系统自带了 u-boot, 但是只能启动打了 TEXT_OFFSET 补丁的内核, 所以为了启动一个原生的内核, 需要挂载新版本的 u-boot, 所有脚本已经写好, 只需复制到 boot 分区, N1 就可以用自身的 u-boot, 挂载这个新的 u-boot

> 启动原理 第一步 : N1 内置 u-boot -> 优先寻找 U盘 -> 寻找 boot 分区的 s905_autoscript -> 根据这个 s905_autoscript 的内容加载新的 u-boot, 也就是 u-boot.ext

> 启动原理 第二步 : 根据 u-boot.ext 内置的脚本依次执行 : bootcmd -> distro_bootcmd -> boot_targets -> bootcmd_usb0 -> usb_boot -> scan_dev_for_boot_part -> scan_dev_for_boot -> scan_dev_for_extlinux -> boot_extlinux, 然后找到了 extlinux.conf

> 启动原理 第三步 : 根据 extlinux.conf 的设置, 定位 root 分区 -> 寻找 zImage 和 uInitrd 也就是安装 kernel 后生成的 两个文件. 至此, 控制权交给了 linux 内核. 

> 确保已经在 chroot 环境

1. 下载文件
    ```sh
    ping -c 3 www.163.com # 确保网络 OK
    pacman -S git
    cd /tmp && git clone https://github.com/cattyhouse/new-uboot-for-N1
    cd new-uboot-for-N1
    cp -fr * /boot/
    ```
1. 设置 extlinux.conf

    ```sh
    root_uuid=$(lsblk -f  | grep sda2 | xargs | cut -d ' ' -f5)
    sed -i "s/root_uuid/${root_uuid}/" /boot/extlinux/extlinux.conf
    # 确认一下修改是否成功
    grep UUID /boot/extlinux/extlinux.conf
    # 清理一下
    rm /boot/README.md
    ```
1. 设置 fstab
    ```sh
    root_uuid=$(lsblk -f | grep sda2 | xargs | cut -d ' ' -f5)
    boot_uuid=$(lsblk -f | grep sda1 | xargs | cut -d ' ' -f5)

    echo "# /dev/sda2" >> /etc/fstab
    echo "UUID=${root_uuid} / ext4 rw,relatime 0 1" >> /etc/fstab

    echo "# /dev/sda1" >> /etc/fstab
    echo "UUID=${boot_uuid} /boot vfat rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro 0 2" >> /etc/fstab

    cat /etc/fstab # 检查一下
    ```


# 基本设置
> 重启
```sh
exit # 退出 chroot 环境
cd / # 确保所有的窗口不占用 /mnt
sudo umount -fR /mnt # target is busy 的错误可以忽略
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