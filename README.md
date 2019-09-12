# 安装 Archlinux

> 以下操作需要在arm64系统下面进行, 比如 archlinux, armbian, raspbian 等等. 
> x86下面也是有可能的, 需要用到 systemd-nspawn 来启动一个 archlinux arm64 的系统.

## 安装 base 和 kernel

- **宿主是 archlinux 系统**, 如果是别的系统, 直接跳转到 **宿主是其他 arm64 系统**, 再回来.
    
    > 准备一下系统环境

    ```bash
    # 确保网络通畅, DNS 解析没问题. 
    ping www.163.com
    # 确保时间正确
    timedatectl set-timezone Asia/Shanghai # 设置时区
    timedatectl set-ntp 1 # 开始同步时间, 可能需要1分钟左右.
    timedatectl status # 确认 Local time 是否为当前的时间.

    # 准备 pacman 环境
    pacman-key --init
    pacman-key --populate archlinuxarm
    pacman -Sy arch-install-scripts uboot-tools dosfstools
    ```
    > 分区, 格式化, 挂载

    - 寻找设备路径
        ````
        lsblk
        ````
    - 对于安装到U盘来说, 分区和格式化以及挂载

        ```bash
        fdisk /dev/sda 

        # 不一定叫做sda, 需要您仔细确认
        # o 创建空白的dos分区表
        # n 创建新分区, 选primary,容量512M
        # t 输入 c, 设置 type 为 W95 FAT32 (LBA)
        # n 创建新分区, 选primary,容量为剩余所有
        # w 保存

        mkfs.vfat /dev/sda1 # mkfs.vfat 需要 dosfstools 这个包
        mkfs.ext4 /dev/sda2
        mount /dev/sda2 /mnt
        mkdir -p /mnt/boot
        mount /dev/sda1 /mnt/boot
        ```

    - 对于安装到MMC来说, 分区和格式化以及挂载

        > MMC的设备名是 **/dev/mmcblk1**

        > 分区必须严格按照下面的格式

        > 注意 **Start**, **End**

        > 这样分区的目的是为了避免写入数据到uboot所在的block导致系统变砖.

        ```bash
        fdisk /dev/mmcblk1

        # o 创建空白dos分区表
        # n 创建第一个分区, 选择Primary, 起始 221184 , 终止 1269760.
        # t 修改类型, 输入 c 
        # n 创建第二个分区, 选择Primary, 起始 1400832, 终止 15269887.
        # w 保存分区表.
        ```
        > 分区正确的情况下 `fdisk -l /dev/mmcblk1` 结果如下

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
        > 格式化并挂载

        ```bash
        mkfs.vfat /dev/mmcblk1p1 # mkfs.vfat 需要 dosfstools 这个包
        mkfs.ext4 /dev/mmcblk1p2
        mount /dev/mmcblk1p2 /mnt
        mkdir -p /mnt/boot
        mount /dev/mmcblk1p1 /mnt/boot
        ```
    > 开始安装

    ```bash
    # 安装 base
    pacstrap /mnt base
    genfstab -U /mnt >> /mnt/etc/fstab
    # 检查下 /mnt/etc/fstab, 除了 "#" 开头的之外, 确保里面只有两行内容, 一行是挂载 "/" 另一行是挂载 "/boot", 其他的(比如/dev/zram*)删掉
    arch-chroot /mnt

    # 安装 kernel
    for item in $(curl -sL https://archlinux.jerryxiao.cc/any | grep keyring | grep -v sig | cut -d  \" -f2 |xargs); do curl -OL https://archlinux.jerryxiao.cc/any/$item ; done
    pacman -U jerryxiao-keyring-*.pkg.tar.xz
    echo '[jerryxiao]
    Server = https://archlinux.jerryxiao.cc/$arch' >> /etc/pacman.conf
    pacman -Sy linux-phicomm-n1 linux-phicomm-n1-headers firmware-phicomm-n1 
    ```

- **宿主是其他 arm64 系统**, 比如 armbian,raspbian 等
    > 受到 jerry 的启发, 先启动进入 armbian, 然后将 archlinuxarm 的 base 解压到随便一个文件夹中, 
    > 然后 chroot 到archlinuxarm 的 base, 就得到一个 archlinux 的操作环境, 
    > 就可以跳转到 **宿主是 archlinux 系统** 继续安装.
    
    > bsdtar 某些版本可能会提示以下error, 可以忽略.
    
    ````
    bsdtar: Ignoring malformed pax extended attribute
    bsdtar: Ignoring malformed pax extended attribute
    bsdtar: Ignoring malformed pax extended attribute
    bsdtar: Ignoring malformed pax extended attribute
    bsdtar: Error exit delayed from previous errors.
    ````

    ```bash
    # 确保网络通畅, DNS 解析没问题. 
    ping www.163.com
    # 确保时间正确
    timedatectl set-timezone Asia/Shanghai # 设置时区
    timedatectl set-ntp 1 # 开始同步时间, 可能需要1分钟左右.
    timedatectl status # 确认 Local time 是否为当前的时间.

    # 准备 archlinuxarm环境
    cd ~ 
    curl -OL http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
    mkdir alarm
    bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C alarm
    rm ./alarm/etc/resolv.conf # 删除这个软连接, 它默认是指向一个一个动态文件
    cat /etc/resolv.conf > ./alarm/etc/resolv.conf # 重新生成此文件, 同步宿主的DNS设置
    mount --bind alarm alarm # 因为 alarm 不是一个挂载点, 所以需要自己 mount 自己, 否则后面会出错.
    cd alarm
    mount -t proc /proc proc
    mount --make-rslave --rbind /sys sys
    mount --make-rslave --rbind /dev dev
    mount --make-rslave --rbind /run run
    chroot ~/alarm /bin/bash
    source /etc/profile
    source ~/.bashrc # 如果提示无此文件, 并没有关系.
    export PS1="(chroot) $PS1"
    # 此时 archlinuxarm 的环境已经准备好, 可以跳转到 -- "宿主是 archlinux 系统".
    ```

## 设置 uboot


- 确保可以打印和写入uboot env (uboot 环境变量)

    - 生成配置文件

        ```bash
        echo '/dev/mmcblk1 0x27400000 0x10000' > /etc/fw_env.config
        ```
    - 打印 uboot env

        ```bash
        fw_printenv # 执行一下看是否可以成功输出 uboot env
        ```
    - 写入 uboot env

        ```bash
        fw_setenv # 后面会用到, 现在不需要执行.
        ```

- 创建 uEnv.ini

    ```bash
    # `lsblk -f` 找到 `sda2 或者 mmcblk1p2` 的 `UUID`
    # 生成 eth0 的 MAC 地址
    echo $(uuidgen) | md5sum | sed 's/^\(..\)\(..\)\(..\)\(..\)\(..\).*$/02:\1:\2:\3:\4:\5/'
    # 将生成的信息填入下面, 创建 uEnv.ini
    echo 'dtb_name=/dtb.img
    bootargs=root=UUID=找到的root分区的UUID rootflags=data=writeback rw console=ttyAML0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0
    ethaddr=生成的MAC地址' > /boot/uEnv.ini
    ```

- 设置uboot env脚本

    > **前方有巨坑, 注意** : 以下代码里面有变量, 所以 cat EOF 必须用单引号, 
    > 而且所有的内容都是单引号, 否则会被 shell 和 uboot 展开, 造成各种问题. 

    > Credit: aml_autoscript.cmd aml_autoscript.zip s905_autoscript.cmd  emmc_autoscript.cmd 均来自大神 150balbes 的 [armbian](https://disk.yandex.ru/d/srrtn6kpnsKz2).

    - aml_autoscript.cmd 和 aml_autoscript.zip

        ```bash
        cd /boot && curl -OL https://raw.githubusercontent.com/cattyhouse/N1-install/master/aml_autoscript.zip

        cat <<'EOF' > /boot/aml_autoscript.cmd
        defenv
        setenv bootcmd 'run start_autoscript; run storeboot'
        setenv start_autoscript 'if mmcinfo; then run start_mmc_autoscript; fi; if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript'
        setenv start_emmc_autoscript 'if fatload mmc 1 1020000 emmc_autoscript; then autoscr 1020000; fi;'
        setenv start_mmc_autoscript 'if fatload mmc 0 1020000 s905_autoscript; then autoscr 1020000; fi;'
        setenv start_usb_autoscript 'for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 s905_autoscript; then autoscr 1020000; fi; done'
        setenv upgrade_step 2
        saveenv
        sleep 1
        reboot
        EOF
        ```

    - s905_autoscript.cmd

        ```bash
        cat <<'EOF' > /boot/s905_autoscript.cmd
        if fatload mmc 0 0x11000000 boot_android; then if test ${ab} = 0; then setenv ab 1; saveenv; exit; else setenv ab 0; saveenv; fi; fi;
        if fatload usb 0 0x11000000 boot_android; then if test ${ab} = 0; then setenv ab 1; saveenv; exit; else setenv ab 0; saveenv; fi; fi;
        setenv env_addr 0x10400000
        setenv kernel_addr 0x11000000
        setenv initrd_addr 0x13000000
        setenv boot_start booti ${kernel_addr} ${initrd_addr} ${dtb_mem_addr}
        setenv addmac 'if printenv mac; then setenv bootargs ${bootargs} mac=${mac}; elif printenv eth_mac; then setenv bootargs ${bootargs} mac=${eth_mac}; fi'
        setenv try_boot_start 'if fatload ${devtype} ${devnum} ${kernel_addr} zImage; then if fatload ${devtype} ${devnum} ${initrd_addr} uInitrd; then fatload ${devtype} ${devnum} ${env_addr} uEnv.ini && env import -t ${env_addr} ${filesize} && run addmac; fatload ${devtype} ${devnum} ${dtb_mem_addr} ${dtb_name} && run boot_start; fi; fi;'
        setenv devtype mmc
        setenv devnum 0
        run try_boot_start
        setenv devtype usb
        for devnum in 0 1 2 3 ; do run try_boot_start ; done
        EOF
        ```

    - emmc_autoscript.cmd

        ```bash
        cat <<'EOF' > /boot/emmc_autoscript.cmd
        setenv env_addr 0x10400000
        setenv kernel_addr 0x11000000
        setenv initrd_addr 0x13000000
        setenv dtb_mem_addr 0x1000000
        setenv boot_start booti ${kernel_addr} ${initrd_addr} ${dtb_mem_addr}
        setenv addmac 'if printenv mac; then setenv bootargs ${bootargs} mac=${mac}; elif printenv eth_mac; then setenv bootargs ${bootargs} mac=${eth_mac}; fi'
        if fatload mmc 1 ${kernel_addr} zImage; then if fatload mmc 1 ${initrd_addr} uInitrd; then if fatload mmc 1 ${env_addr} uEnv.ini; then env import -t ${env_addr} ${filesize}; run addmac; fi; if fatload mmc 1 ${dtb_mem_addr} ${dtb_name}; then run boot_start;fi;fi;fi;
        EOF
        ```

    - 生成二进制文件

        ```bash
        cd /boot
        mkimage -C none -A arm -T script -d aml_autoscript.cmd aml_autoscript
        mkimage -C none -A arm -T script -d s905_autoscript.cmd s905_autoscript
        mkimage -C none -A arm -T script -d emmc_autoscript.cmd emmc_autoscript
        # 说明: 
        # uboot env 默认的参数为 
        # start_autoscript 'if mmcinfo; then run start_mmc_autoscript; fi; if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript'
        # 而 start_mmc_autoscript=if fatload mmc 0 1020000 s905_autoscript; then autoscr 1020000; fi; 是用来启动安卓的, 因为N1的 mmc为mmc 1, 所以会继续运行 start_emmc_autoscript
        # 而 start_emmc_autoscript=if fatload mmc 1 1020000 emmc_autoscript; then autoscr 1020000; fi; 它需要 emmc_autoscript 这个文件.
        # 所以在N1上, s905_autoscript用于启动U盘或者安卓系统, emmc_autoscript 用于启动 mmc. aml_autoscript 在uboot执行update的时候运行 (adb shell reboot update 之后)
        ```

    - 同步 aml_autoscript 的内容

        > 上面提到过, aml_autoscript 的执行需要特殊环境, 此目的是确保当前的 uboot 环境就像是运行过 aml_autoscript 一样

        ```bash
        fw_setenv bootcmd 'run start_autoscript; run storeboot'
        fw_setenv start_autoscript 'if mmcinfo; then run start_mmc_autoscript; fi; if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript'
        fw_setenv start_emmc_autoscript 'if fatload mmc 1 1020000 emmc_autoscript; then autoscr 1020000; fi;'
        fw_setenv start_mmc_autoscript 'if fatload mmc 0 1020000 s905_autoscript; then autoscr 1020000; fi;'
        fw_setenv start_usb_autoscript 'for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 s905_autoscript; then autoscr 1020000; fi; done'
        ```
## 收尾工作

- 安装必要的软件并开机启动

    ```bash
    pacman -S haveged openssh
    systemctl enable haveged
    systemctl enable sshd
    ```

- 临时允许ssh密码登陆root账户

    ```bash
    # 出于安全考虑, 建议启动N1后,删除这条,采用 `id_rsa` `ssh key` 登陆
    echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
    ```

- 设置ip

    > 为了简单起见, 这里用了 DHCP 自动获取 ip, 
    > 无显示器的话, 可以去路由器上查看获取的 ip 地址

    ```bash
    # 有线
    cp /etc/netctl/examples/ethernet-dhcp /etc/netctl/eth0-dhcp
    netctl enable eth0-dhcp

    # 无线
    pacman -S wpa_supplicant crda
    cp /etc/netctl/examples/wireless-wpa  /etc/netctl/wlan0-dhcp
    ## 编辑 wlan0-dhcp ESSID 填入 Wi-Fi 的名字 Key 填入 Wi-Fi 的密码
    netctl enable wlan0-dhcp
    ```

- 设置密码

    ````
    passwd root
    ````

- 退出

    ````        
    exit
    umount -vR /mnt
    ````    

# 题外话: MMC安装之克隆安装

- 用前面做好的archlinux的U盘启动N1, 给MMC [分区,格式化,挂载](https://github.com/cattyhouse/N1-install#安装-base-和-kernel)

- 用rsync克隆U盘的内容到 MMC 分区, 完成后, mmc的内容与U盘一模一样

    ```bash
    rsync -avPhHAX --numeric-ids  --delete --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/lost+found"} / /mnt
    ```

- 修改相关文件的UUID

    > 注意, UUID修改后, 如果再运行上面的rsync命令, mmc的这两个文件会被rsync恢复为U盘的对应文件, 这也是rsync的魅力所在, 100%克隆.

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

# 初次启动的一些设置

```bash
# 主机名, 语言, 时间同步
hostnamectl set-hostname xxxx

cat <<EOF >> /etc/hosts
127.0.0.1        localhost
::1              localhost
127.0.1.1        $(cat /etc/hostname).localdomain        $(cat /etc/hostname)
EOF

echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
locale-gen # 可能需要蛮久
localectl set-locale en_US.utf8
timedatectl set-timezone Asia/Shanghai
# 使用阿里云的时间服务器
echo 'NTP=ntp1.aliyun.com ntp2.aliyun.com ntp3.aliyun.com ntp4.aliyun.com' >> /etc/systemd/timesyncd.conf
# 启动 systemd-timesyncd 服务, 因为 N1 没有 RTC 时钟
timedatectl set-ntp 1

# cpu 变频
pacman -S cpupower
vim /etc/default/cpupower
# governor='ondemand', min_freq="500MHz" , max_freq="2GHz"
systemctl start cpupower.service
systemctl enable cpupower.service
# 查看
while true ; do sleep 2 ; cpupower -c all frequency-info | grep -E "current CPU" | cut -d " " -f 5-7 ; echo "" ; done

# 提高网络性能
# 注意 systemd 已经不再开机加载 /etc/sysctl.conf
# 必须放到 /etc/sysctl.d/
cat <<'EOF' > /etc/sysctl.d/sysctl.conf
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.ip_forward = 1
net.core.somaxconn = 2048
net.core.rmem_default = 1048576
net.core.rmem_max = 16777216
net.core.wmem_default = 1048576
net.core.wmem_max = 16777216
net.core.optmem_max = 65536
net.ipv4.tcp_rmem = 4096 1048576 2097152
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.ip_local_port_range = 20000 60999
net.ipv4.tcp_max_syn_backlog = 20000
net.core.netdev_max_backlog = 20480
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.eth0.send_redirects = 0
net.ipv4.conf.eth0.accept_redirects = 0
net.ipv4.conf.lo.send_redirects = 0
net.ipv4.conf.lo.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.udp_rmem_min = 8192
net.ipv4.udp_wmem_min = 8192
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_syncookies = 1
net.core.netdev_budget = 50000
net.core.netdev_budget_usecs = 5000
EOF

sysctl -p /etc/sysctl.d/sysctl.conf

# 提高文件性能
cat <<'EOF' >> /etc/systemd/system.conf
DefaultLimitNOFILE=2097152:2097152
DefaultLimitNPROC=2097152:2097152
EOF

cat <<'EOF' >> /etc/security/limits.conf
*         hard    nofile      500000
*         soft    nofile      500000
root      hard    nofile      500000
root      soft    nofile      500000
*         hard    nproc      500000
*         soft    nproc      500000
root      hard    nproc      500000
root      soft    nproc      500000
EOF

# 更新一下系统
pacman -Syyuu
# 重启一遍
reboot
```

# 在 N1上用 archlinux 编译主线 kernel

> 感谢
> [@isjerryxiao](https://github.com/archlinux-jerry/pkgbuilds) for 二进制编译和下载, 
> [@RedL0tus闲鱼大佬](https://github.com/RedL0tus/Linux-Phicomm-N1) for PKGBUILD 修改, 
> [@MarvelousBlack惠狐](https://github.com/MarvelousBlack/firmware-phicomm-n1_PKGBULD) for dts 修改.

> 编译需要在N1上进行, 配合distcc会快很多, distcc需要在N1和x86主机上安装. [参考](https://archlinuxarm.org/wiki/Distcc_Cross-Compiling)

## 手动 (不建议, 此处只是为了记录原理)

- 下载 archlinuxarm 的 linux-aarch64 源代码

    ``` bash
    mkdir -p ~/n1
    git clone https://github.com/archlinuxarm/PKGBUILDs
    cp -r PKGBUILDs/core/linux-aarch64 ~/n1/
    ```

- 修改 *`~/n1/linux-aarch64/PKGBUILD`*

    - 在 **`cat "${srcdir}/config" > ./.config`** 后面插入
    
        ```bash
        # Amlogic meson SoC TEXT_OFFSET
        sed -i "s/TEXT_OFFSET := 0x00080000/TEXT_OFFSET := 0x01080000/g" arch/arm64/Makefile
        sed -i "s/#error TEXT_OFFSET must be less than 2MB//g" arch/arm64/kernel/head.S
        # dts fix
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
        ```

    - 在 **`make INSTALL_DTBS_PATH="${pkgdir}/boot/dtbs" dtbs_install`** 后面增加

        ```bash
        # cp meson-gxl-s905d-phicomm-n1.dtb as dtb.img, for s905_autoscript to load
        cp "${pkgdir}/boot/dtbs/amlogic/meson-gxl-s905d-phicomm-n1.dtb" "${pkgdir}/boot/dtb.img" 
        ```

    - 修改 *`pkgname=("${pkgbase}" "${pkgbase}-headers" "${pkgbase}-chromebook")`* 为 **`pkgname=("${pkgbase}" "${pkgbase}-headers")`** 避免编译chromebook的kernel

- 编译 kernel 

    > 使用distcc的情况下大约3小时~4小时

    ```bash
    cd ~/n1/linux-aarch64
    makepkg -s
    ```

- 安装 kernel

    ````
    pacman -U *.pkg.tar.xz
    ````

## 自动 (建议使用的方法)

> 直接下载修改好的PKGBUILD编译

```bash
git clone https://github.com/archlinux-jerry/pkgbuilds
cd pkgbuilds/linux-phicomm-n1
makepkg -s
pacman -U *.pkg.tar.xz
```

## 二进制 (懒人版)

```bash
for item in $(curl -sL https://archlinux.jerryxiao.cc/aarch64/ | grep -E "linux-phicomm-n1-.*-aarch64.pkg.tar.xz" | grep -Ev "sig|git"  | cut -d \" -f2 | xargs); do curl -OL https://archlinux.jerryxiao.cc/aarch64/$item ; done
pacman -U *.pkg.tar.xz
```



