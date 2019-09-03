# N1 安装 Archlinux

## 制作 `archlinux USB` 启动盘

**以下均在 `archlinux arm64` 系统下面操作**

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

- 安装 base 

```bash
pacman -S arch-install-scripts
pacstrap /mnt base
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

- 安装 kernel

```bash
cd /tmp
for item in $(curl -s https://archlinux.jerryxiao.cc/aarch64/ | grep linux-phicomm-n1-lts-git | cut -d \" -f2  | grep -v sig | xargs); do curl -O https://archlinux.jerryxiao.cc/aarch64/$item ; done
pacman -U linux-phicomm-n1-lts-git*
```

- 创建 uEnv.ini
    
    - `lsblk -f` 找到 `/dev/sda2` 的 `UUID`

    - 生成MAC地址: `dd if=/dev/urandom bs=1024 count=1 2>/dev/null|md5sum|sed 's/^\(..\)\(..\)\(..\)\(..\)\(..\)\(..\).*$/\1:\2:\3:\4:\5:\6/'`

    - 将上面的信息填入下面, 然后运行

```bash
echo "bootargs=root=UUID=找到的sda2的UUID rootflags=data=ordered rw console=ttyAML0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0
ethaddr=生成的MAC地址" > /boot/uEnv.ini 
```

- 设置uboot env脚本

aml_autoscript.cmd

```bash
echo "
setenv bootcmd "run start_autoscript; run storeboot;"
setenv start_autoscript "if usb start ; then run start_usb_autoscript; fi; run start_mmc_autoscript;"
setenv start_mmc_autoscript "if fatload mmc 1 1020000 s905_autoscript; then autoscr 1020000; fi;"
setenv start_usb_autoscript "if fatload usb 0 1020000 s905_autoscript; then autoscr 1020000; fi; if fatload usb 1 1020000 s905_autoscript; then autoscr 1020000; fi; if fatload usb 2 1020000 s905_autoscript; then autoscr 1020000; fi; if fatload usb 3 1020000 s905_autoscript; then autoscr 1020000; fi;"
setenv upgrade_step "0"
saveenv
sleep 1
reboot
" > /boot/aml_autoscript.cmd
```

s905_autoscript.cmd

```bash
echo "
setenv env_addr "0x10400000"
setenv kernel_addr "0x11000000"
setenv initrd_addr "0x13000000"
setenv boot_start booti ${kernel_addr} ${initrd_addr} ${dtb_mem_addr}
if fatload usb 0 ${kernel_addr} zImage; then if fatload usb 0 ${initrd_addr} uInitrd; then if fatload usb 0 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload usb 0 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;
if fatload usb 1 ${kernel_addr} zImage; then if fatload usb 1 ${initrd_addr} uInitrd; then if fatload usb 1 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload usb 1 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;
if fatload mmc 1 ${kernel_addr} zImage; then if fatload mmc 1 ${initrd_addr} uInitrd; then if fatload mmc 1 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize};fi; if fatload mmc 1 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;
" > /boot/s905_autoscript.cmd
```

生成二进制文件

```bash
cd /boot
/usr/bin/mkimage -C none -A arm -T script -d aml_autoscript.cmd aml_autoscript
/usr/bin/mkimage -C none -A arm -T script -d s905_autoscript.cmd s905_autoscript
```

- 安装必要的软件

```bash
pacman -S haveged
systemctl enable haveged
```

- 开机启动sshd

```bash
systemctl enable sshd
```
- 设置ip

```bash
echo "
Description='A basic dhcp ethernet connection'
Interface=eth0
Connection=ethernet
IP=dhcp
" > /etc/netctl/eth0-dhcp

netctl enable eth0-dhcp
```

- 设置密码

```bash
passwd root
```

### 用 archlinux 的U盘启动 N1

- 启动 N1 的前提是已经刷过机
- 刷机教程不再赘述
- 启动成功后, `aml_autoscript` 以及 `aml_autoscript.cmd` 可以从 `/boot` 移除, 其余不要动

## MMC安装

MMC的安装与U盘过程一模一样, 但有几点区别:

- MMC的设备名是 `/dev/mmcblk1`, 分区必须严格按照下面的格式, 注意 `Start` `End`, 否则变砖:

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

- uEnv.ini 填入 `/dev/mmcblk1p2` 的 UUID

- `/boot/` 不需要 `aml_autoscript` 和 `aml_autoscript.cmd`

- 安装到MMC后, 必须拔掉U盘才可以从MMC启动, 因为U盘的启动优先级更高

## 其他

### 在 archlinux下面操作 `uboot env`

- 配置文件

```bash
echo "/dev/mmcblk1            0x27400000      0x10000" > /etc/fw_env.config
```
- 打印 uboot env

```bash
fw_printenv 
```

- 设置 uboot env ( !!! 慎用 )

```bash
fw_setenv
```

### 如果 MMC 启动失败

从U盘启动, 运行

```bash
fw_setenv start_autoscript "if usb start ; then run start_usb_autoscript; fi; run start_mmc_autoscript;"
fw_setenv start_mmc_autoscript "if fatload mmc 1 1020000 s905_autoscript; then autoscr 1020000; fi;"
````



