# N1 安装 Archlinux

## 制作 `archlinux USB` 启动盘

以下操作需要在ARM64系统下面进行, ARCHLINUX, ARMBIAN 或者 RASPBIAN 也行... 只要是 ARM64. 

### 准备USB盘

- 寻找u盘设备路径

```bash
lsblk -f
```

- 分区

```bash
fdisk /dev/sda
```
        按o创建空白的dos分区表
        分区1, 512M, type c
        分区2, 余下所有, type 83
        按 w 保存

- 格式化

```bash
mkfs.vfat /dev/sda1
mkfs.ext4 /dev/sda2
```

- 挂载

```bash
mount /dev/sda2 /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

### 安装 archlinux 到U盘

- 安装 base **(宿主是archlinux arm64系统), 宿主为其他系统的忽略此章节**

```bash
pacman -S arch-install-scripts
pacstrap /mnt base
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
pacman -Syy
pacman -S uboot-tools
```

- 安装 kernel **(宿主是archlinux arm64系统), 宿主为其他系统的忽略此章节**

```bash
cd /tmp
for item in $(curl -sL https://archlinux.jerryxiao.cc/aarch64/ | grep -E "linux-phicomm-n1-.*-aarch64.pkg.tar.xz" | grep -Ev "sig|git"  | cut -d \" -f2 | xargs); do curl -OL https://archlinux.jerryxiao.cc/aarch64/$item ; done
pacman -U linux-phicomm-n1-*
```

- 安装 base **(宿主是其他 arm64 系统, 比如armbian), 宿主为archlinux的忽略此章节**

采用archlinuxarm [官方做好的base](https://archlinuxarm.org/platforms/armv8/generic), 留意它的一些说明以及注意事项, 后面提到的很多操作这个base已经做好了, 一定要完整阅读它的说明:

````
默认安装了的软件包 openssh,haveged
默认安装了通用的linux内核, 并不适合N1, 后面我们在安装N1的内核的时候, 会自动取代这个内核.
root的默认密码是root, 后面可以自行设置密码
包含一个alarm的普通用户, 不需要可以运行 userdel alarm 删除掉
sshd默认启动
haveged 默认启动
systemd-networkd 默认启动, 我们需要disable掉这个服务
systemd-resolved 默认启动, 我们需要disable掉这个服务, 然后手动编辑/etc/resolv.conf的DNS
systemd-timesyncd 时间同步服务默认启动.这个不需要做什么修改
````
下载base并解压

```bash
curl -OL http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
```
bsdtar 某些版本可能会提示以下error, 这个不影响使用, 可以忽略.

````
bsdtar: Ignoring malformed pax extended attribute
bsdtar: Ignoring malformed pax extended attribute
bsdtar: Ignoring malformed pax extended attribute
bsdtar: Ignoring malformed pax extended attribute
bsdtar: Error exit delayed from previous errors.

````

- 安装 kernel **(宿主是其他 arm64 系统, 比如armbian), 宿主为archlinux的忽略此章节**

注意: fstab 的 sda1 和 sda2 的 UUID 需要根据你的情况写入, 查询UUID用 `lsblk -f`

```bash
cd /mnt
mount -t proc /proc proc/
mount --bind /sys sys/
mount --bind /dev dev/
mount --bind /run run/
rm -f etc/resolv.conf
cp /etc/resolv.conf etc/resolv.conf
chroot /mnt /bin/bash
source /etc/profile
source ~/.bashrc
export PS1="(chroot) $PS1"
pacman-key --init
pacman-key --populate archlinuxarm
pacman -Syy
# mkimage命令需要安装 uboot-tools
pacman -S uboot-tools
# 因为 archlinuxarm 官方的这个base默认启动了这两个服务配置网络, 我们后面用的是netctl的方式, 所以必须disable这两个服务
systemctl disable systemd-networkd
systemctl disable systemd-resolved

echo '
# <file system> <dir> <type> <options> <dump> <pass>
# /dev/sda2
UUID=注意!!!sda2的UUID / ext4 rw,relatime 0 1
# /dev/sda1
UUID=注意!!!sda1的UUID /boot vfat rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro 0 2
' > /etc/fstab

cd /tmp
for item in $(curl -sL https://archlinux.jerryxiao.cc/aarch64/ | grep -E "linux-phicomm-n1-.*-aarch64.pkg.tar.xz" | grep -Ev "sig|git"  | cut -d \" -f2 | xargs); do curl -OL https://archlinux.jerryxiao.cc/aarch64/$item ; done
pacman -U linux-phicomm-n1-*
```

- 黑科技, chroot到archlinuxarm直接安装到mmc **(宿主是其他 arm64 系统, 比如armbian), 宿主为archlinux的忽略此章节**

受到jerry的启发, 此为黑科技, 原理就是: 先用armbian的U盘启动N1, 然后将archlinuxarm的base解压到内存中, 然后chroot到archlinuxarm的base, 然后就得到一个archlinux的操作环境,就可以对mmc操作, 直接将archlinux安装到mmc, 步骤:

```bash
cd ~ 
curl -OL http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
# 改变/tmp的大小为1.5G, 能容纳解压后的archlinuxarm base, N1的可用内存为1.8G, 所以1.5G的/tmp不会造成问题
mount -o remount,size=1500M,noatime /tmp
mkdir -p /tmp/alarm
bsdtar -xpf ~/ArchLinuxARM-aarch64-latest.tar.gz -C /tmp/alarm
cd /tmp/alarm
mount -t proc /proc proc/
mount --bind /sys sys/
mount --bind /dev dev/
mount --bind /run run/
rm -f etc/resolv.conf
cp /etc/resolv.conf etc/resolv.conf
chroot /tmp/alarm /bin/bash
source /etc/profile
source ~/.bashrc
export PS1="(chroot) $PS1"
# 此时 archlinuxarm的环境已经准备好, 可以跳转到 “(宿主是archlinux arm64系统), 宿主为其他系统的忽略此章节” 直接对mmc进行操作.
```

 **以下章节所有宿主都需要**

- 创建 uEnv.ini
    
    - `lsblk -f` 找到 `/dev/sda2` 的 `UUID`

    - 生成MAC地址: `dd if=/dev/urandom bs=1024 count=1 2>/dev/null|md5sum|sed 's/^\(..\)\(..\)\(..\)\(..\)\(..\)\(..\).*$/\1:\2:\3:\4:\5:\6/'`

    - 将上面的信息填入下面, 然后运行

```bash
echo 'bootargs=root=UUID=找到的sda2的UUID rootflags=data=ordered rw console=ttyAML0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0
ethaddr=生成的MAC地址' > /boot/uEnv.ini 
```

- 设置uboot env脚本

aml_autoscript.cmd

```bash
echo 'setenv bootcmd "run start_autoscript; run storeboot;"
setenv start_autoscript "if usb start ; then run start_usb_autoscript; fi; if mmcinfo; then run start_mmc_autoscript; fi; run start_emmc_autoscript;"
setenv start_emmc_autoscript "if fatload mmc 1 1020000 emmc_autoscript; then autoscr 1020000; fi;"
setenv start_mmc_autoscript "if fatload mmc 0 1020000 s905_autoscript; then autoscr 1020000; fi;"
setenv start_usb_autoscript "if fatload usb 0 1020000 s905_autoscript; then autoscr 1020000; fi; if fatload usb 1 1020000 s905_autoscript; then autoscr 1020000; fi; if fatload usb 2 1020000 s905_autoscript; then autoscr 1020000; fi; if fatload usb 3 1020000 s905_autoscript; then autoscr 1020000; fi;"
setenv upgrade_step "2"
saveenv
sleep 1
reboot' > /boot/aml_autoscript.cmd
```

s905_autoscript.cmd

```bash
echo 'setenv env_addr "0x10400000"
setenv kernel_addr "0x11000000"
setenv initrd_addr "0x13000000"
setenv boot_start booti ${kernel_addr} ${initrd_addr} ${dtb_mem_addr}
if fatload usb 0 ${kernel_addr} zImage; then if fatload usb 0 ${initrd_addr} uInitrd; then if fatload usb 0 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload usb 0 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;
if fatload usb 1 ${kernel_addr} zImage; then if fatload usb 1 ${initrd_addr} uInitrd; then if fatload usb 1 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload usb 1 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;
if fatload mmc 0 ${kernel_addr} zImage; then if fatload mmc 0 ${initrd_addr} uInitrd; then if fatload mmc 0 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload mmc 0 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;' > /boot/s905_autoscript.cmd
```

emmc_autoscript.cmd

```bash
echo 'setenv env_addr "0x10400000"
setenv kernel_addr "0x11000000"
setenv initrd_addr "0x13000000"
setenv boot_start booti ${kernel_addr} ${initrd_addr} ${dtb_mem_addr}
if fatload usb 0 ${kernel_addr} zImage; then if fatload usb 0 ${initrd_addr} uInitrd; then if fatload usb 0 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload usb 0 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;
if fatload usb 1 ${kernel_addr} zImage; then if fatload usb 1 ${initrd_addr} uInitrd; then if fatload usb 1 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload usb 1 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;
if fatload mmc 1 ${kernel_addr} zImage; then if fatload mmc 1 ${initrd_addr} uInitrd; then if fatload mmc 1 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload mmc 1 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;' > /boot/emmc_autoscript.cmd
```

生成二进制文件

```bash
cd /boot
/usr/bin/mkimage -C none -A arm -T script -d aml_autoscript.cmd aml_autoscript
/usr/bin/mkimage -C none -A arm -T script -d s905_autoscript.cmd s905_autoscript
/usr/bin/mkimage -C none -A arm -T script -d emmc_autoscript.cmd emmc_autoscript
# 说明: 
# uboot env 默认的参数为 
# start_autoscript=if usb start ; then run start_usb_autoscript; fi; if mmcinfo; then run start_mmc_autoscript; fi; run start_emmc_autoscript;
# 而 start_mmc_autoscript=if fatload mmc 0 1020000 s905_autoscript; then autoscr 1020000; fi; 是不能启动mmc的, 因为N1的mmc为mmc 1, 所以会继续运行 start_emmc_autoscript
# 而 start_emmc_autoscript=if fatload mmc 1 1020000 emmc_autoscript; then autoscr 1020000; fi; 它需要 emmc_autoscript 这个文件.
# 所以在N1上, s905_autoscript用于启动U盘, emmc_autoscript 用于启动 mmc. aml_autoscript 用于首次启动N1的时候的初始化(一般情况下不需要, 因为里面的内容是uboot env 默认的配置, 制作这个是以防万一)

```

- 安装必要的软件并开机启动

```bash
pacman -S haveged
systemctl enable haveged
pacman -S openssh
systemctl enable sshd
```

- 临时允许ssh密码登陆root账户

**出于安全考虑, 建议启动N1后,删除这条,采用 `id_rsa` `ssh key` 登陆**

```bash
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
```

- 设置ip

为了简单起见, 这里用了dhcp, N1获取的ip到底是什么, 需要去路由器上看, 或者ping内网网段的ip找出来.

```bash
echo '
Description='A basic dhcp ethernet connection'
Interface=eth0
Connection=ethernet
IP=dhcp
' > /etc/netctl/eth0-dhcp

netctl enable eth0-dhcp
```

- 设置密码

```bash
passwd root
```

- 退出

```bash
exit
umount -vR /mnt
```

### 用 archlinux 的U盘启动 N1

- 启动 N1 的前提是已经刷过机
- 刷机教程不再赘述

## MMC安装

### 方法一: 全新安装

MMC的安装与U盘过程一模一样, 但有几点区别:

- MMC的设备名是 `/dev/mmcblk1`, 分区必须严格按照下面的格式, 注意 `Start` `End`,这样分区的目的是为了避免写入数据到uboot所在的block导致系统变砖

````
fdisk /dev/mmcblk1
o 创建空白dos分区表
n 创建第一个分区, 选择Primary, 起始 221184 , 终止 1269760.
t 修改类型, 输入 c 
n 创建第二个分区, 选择Primary, 起始 1400832, 终止 15269887.
w 保存分区表.
````
结果如下

````
Disk /dev/mmcblk1: 7.3 GiB, 7818182656 bytes, 15269888 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xa502f7df

Device         Boot   Start      End  Sectors  Size Id Type
/dev/mmcblk1p1       221184  1269760  1048577  512M  c W95 FAT32 (LBA)
/dev/mmcblk1p2      1400832 15269887 13869056  6.6G 83 Linux

````

- /boot/uEnv.ini 填入 `/dev/mmcblk1p2` 的 UUID

- /etc/fstab 填入mmc的UUID, 确认再确认.

- 安装到MMC后, 必须拔掉U盘才可以从MMC启动, 因为U盘的启动优先级更高

### 方法二: 克隆安装

- U盘启动N1, 先按照 方法一 里面的格式给mmc分区

- 格式化分区

```bash
mkfs.vfat /dev/mmcblk1p1
mkfs.ext4 /dev/mmcblk1p2
```

- 挂载 mmc 分区

```bash
mount /dev/mmcblk1p2 /mnt
mkdir -p /mnt/boot
mount /dev/mmcblk1p1 /mnt/boot
```
- 用rsync克隆U盘的内容到 mmc 分区, 完成后, mmc的内容与U盘一模一样

```bash
rsync -avPhHAX --numeric-ids --one-file-system --delete --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/lost+found"} / /mnt
```

- 修改相关文件的UUID

注意, UUID修改后, 如果再运行上面的rsync命令, mmc的这两个文件会被rsync恢复为U盘的对应文件, 这也是rsync的魅力所在, 100%克隆.

```bash
lsblk -f # 获取 mmcblk1p1 和 mmcblk1p2 的 UUID, 并记住
vim /mnt/boot/uEnv.ini #将里面的UUID更换成 /dev/mmcblk1p2 的UUID
vim /mnt/etc/fstab # 修改注释里面的sda1 为 mmcblk1p1, 并更换为对应的UUID, 修改注释里面的sda2 为 mmcblk1p2, 并更换为对应的UUID.
# umount mmc 并关机
cd ~
umount -vR /mnt
halt
```
- 拔掉U盘, 拔插电源重启

## Post Install

```bash
# 添加kernel源
curl -o jerryxiao-keyring-20190410-1-any.pkg.tar.xz https://archlinux.jerryxiao.cc/any/jerryxiao-keyring-20190410-1-any.pkg.tar.xz
pacman -U jerryxiao-keyring-20190410-1-any.pkg.tar.xz
echo '[jerryxiao]
Server = https://archlinux.jerryxiao.cc/$arch' >> /etc/pacman.conf
pacman -Syu

# Wifi 蓝牙 固件
pacman -S firmware-phicomm-n1

# 主机名, 语言, 时间同步
hostnamectl set-hostname xxxx
echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
locale-gen
localectl set-locale en_US.utf8
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp true

# 查询 当前cpu频率
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_cur_freq

# 设备信息
lscpu
lsusb
lsblk
```

### 如何获取 uboot env

- 配置文件

```bash
echo '/dev/mmcblk1 0x27400000 0x10000' > /etc/fw_env.config
```
- 打印 uboot env

```bash
fw_printenv 
```


# 在N1上用archlinux编译主线kernel

- 感谢

[@isjerryxiao](https://github.com/archlinux-jerry/pkgbuilds) for 二进制编译和下载

[@RedL0tus闲鱼大佬](https://github.com/RedL0tus/Linux-Phicomm-N1) for PKGBUILD修改

[@MarvelousBlack惠狐](https://github.com/MarvelousBlack/firmware-phicomm-n1_PKGBULD) for n1 dts 修改


- 编译需要在N1上进行, 配合distcc会快很多, distcc需要在N1和x86主机上安装. [参考](https://archlinuxarm.org/wiki/Distcc_Cross-Compiling)

## 手动 (不建议, 此处只是为了记录原理)

- 下载archlinuxarm的linux-aarch64源代码

``` bash
mkdir -p ~/n1
git clone https://github.com/archlinuxarm/PKGBUILDs
cp -r PKGBUILDs/core/linux-aarch64 ~/n1/
```

- 修改 `~/n1/linux-aarch64/PKGBUILD`

    - 在 `cat "${srcdir}/config" > ./.config ` 后面插入
    
    ````
    # Amlogic meson SoC TEXT_OFFSET
    sed -i "s/TEXT_OFFSET := 0x00080000/TEXT_OFFSET := 0x01080000/g" arch/arm64/Makefile
    sed -i "s/#error TEXT_OFFSET must be less than 2MB//g" arch/arm64/kernel/head.S

    echo '// SPDX-License-Identifier: (GPL-2.0+ OR MIT)
    /*
    * Copyright (c) 2018 He Yangxuan
    */

    /dts-v1/;

    #include "meson-gxl-s905d-p230.dts"

    / {
        compatible = "phicomm,n1", "amlogic,s905d", "amlogic,meson-gxl";
        model = "Phicomm N1";

        cvbs-connector {
            status = "disabled";
        };

        leds {
            compatible = "gpio-leds";

            status {
                label = "n1:white:status";
                gpios = <&gpio_ao GPIOAO_9 GPIO_ACTIVE_HIGH>;
                default-state = "on";
            };
        };
    };


    &ethmac {
        pinctrl-0 = <&eth_pins>;
        pinctrl-names = "default";

        /* Select external PHY by default */
        phy-handle = <&eth_phy0>;

        amlogic,tx-delay-ns = <2>;

        /* External PHY reset is shared with internal PHY Led signals */
        snps,reset-gpio = <&gpio GPIOZ_14 0>;
        snps,reset-delays-us = <0 10000 1000000>;
        snps,reset-active-low;

        /* External PHY is in RGMII */
        phy-mode = "rgmii";

            mdio {
            #address-cells = <0x1>;
            #size-cells = <0x0>;
            compatible = "snps,dwmac-mdio";
            phandle = <0x1a>;

            eth_phy0: ethernet-phy@0 {
                reg = <0x0>;
                phandle = <0x1d>;
            };
        };
    };

    /* This UART is connected to the Bluetooth module */
    &uart_A {
        status = "okay";
        pinctrl-0 = <&uart_a_pins>, <&uart_a_cts_rts_pins>;
        pinctrl-names = "default";
        uart-has-rtscts;

        bluetooth {
            compatible = "brcm,bcm43438-bt";
            shutdown-gpios = <&gpio GPIOX_17 GPIO_ACTIVE_HIGH>;
            max-speed = <2000000>;
            clocks = <&wifi32k>;
            clock-names = "lpo";
        };
    };

    &cvbs_vdac_port {
        status = "disabled";
    };' > ./arch/arm64/boot/dts/amlogic/meson-gxl-s905d-phicomm-n1.dts
    ````

    - 在 `make INSTALL_DTBS_PATH="${pkgdir}/boot/dtbs" dtbs_install` 后面增加

    ````
    # cp meson-gxl-s905d-phicomm-n1.dtb as dtb.img, for s905_autoscript to load
    cp "${pkgdir}/boot/dtbs/amlogic/meson-gxl-s905d-phicomm-n1.dtb" "${pkgdir}/boot/dtb.img" 
    ````

    - 修改 `pkgname=("${pkgbase}" "${pkgbase}-headers" "${pkgbase}-chromebook")` 为 `pkgname=("${pkgbase}" "${pkgbase}-headers")` 避免编译chromebook的kernel

- 编译 kernel (使用distcc的情况下大约3小时~4小时)

````
cd ~/n1/linux-aarch64
makepkg -s
````

- 安装 kernel

````
pacman -U *.pkg.tar.xz
````

## 自动 (建议使用的方法)

````
git clone https://github.com/archlinux-jerry/pkgbuilds
cd pkgbuilds/linux-phicomm-n1
makepkg -s
pacman -U *.pkg.tar.xz
````

## 二进制 (懒人版)

```bash
for item in $(curl -sL https://archlinux.jerryxiao.cc/aarch64/ | grep -E "linux-phicomm-n1-.*-aarch64.pkg.tar.xz" | grep -Ev "sig|git"  | cut -d \" -f2 | xargs); do curl -OL https://archlinux.jerryxiao.cc/aarch64/$item ; done

pacman -U *.pkg.tar.xz
```



