# **准备一个 armbian 启动盘**

> 对于首次安装的人来说可能没有 archlinux 的环境, 那么我们需要准备一个armbian启动盘

- 下载img https://disk.yandex.ru/d/srrtn6kpnsKz2/Linux/ARMBIAN/5.98
- dd if=xxxxx.img of=/dev/sdx (sdx 为u盘的设备名, 千万要仔细确认)
    - 如果你只想在U盘上操作
        - 因为 dd 只会根据 img 所占用的空间给U盘格式化分区, 如果用这个U盘启动N1, 你可以看到根目录剩余空间非常少而无法进行下一步的操作,而U盘会多出来一个未分区的空间
        - 那么需要 fdisk 这个U盘, 将剩余的未分区的空间分出来一个sda3的分区, 在U盘启动N1的状态下, 分区完毕, 重启才可以生效.
        - 然后格式化这个sda3的分区, 并mount到任意文件夹, 然后在这个文件夹下面操作. 
        - 这种方式可以直接把 archlinux 安装到 mmc 或者第二个U盘.
    - 如果你已经把U盘的Armbian用install.sh安装到了mmc, 就不存在这个问题, 可以继续进行, 但此时你只能先把 archlinux 安装到U盘, 然后再通过本文的 **MMC安装之克隆安装** 干掉mmc上的armbian. 
- 然后从 **宿主是其他 arm64 系统** 开始

# **安装 Archlinux**

> 以下均在 arm64 系统下面操作, 比如 archlinuxarm, armbian, raspbian 等等

## 安装 base 和 内核

1. **宿主是 archlinux arm64 系统**
    
    > 如果是别的系统, 直接跳转到 **宿主是其他 arm64 系统** 再回来.
    
    1. 准备一下系统环境

        ```bash
        ping -c 3 www.163.com  # 确保网络通畅, DNS 解析没问题. 
        date # 确保时间正确
        # 准备 pacman 环境
        pacman-key --init
        pacman-key --populate archlinuxarm
        pacman -Sy arch-install-scripts uboot-tools dosfstools vim
        ```
    1. 分区, 格式化, 挂载

        1. 寻找设备路径

            ```bash
            lsblk
            ```
        1. 对于安装到U盘来说, 分区和格式化以及挂载

            ```bash
            fdisk /dev/sda 

            # 不一定叫做sda, 需要您仔细确认
            # o 创建空白的dos分区表
            # n 创建新分区, 选primary, 起始:默认回车, 然后输入 +512M, 设置分区大小为 512M
            # t 输入 c, 设置 type 为 W95 FAT32 (LBA)
            # n 创建新分区, 选primary,容量为剩余所有
            # w 保存

            mkfs.vfat /dev/sda1 # mkfs.vfat 需要 dosfstools 这个包
            mkfs.ext4 /dev/sda2
            mount /dev/sda2 /mnt
            mkdir -p /mnt/boot
            mount /dev/sda1 /mnt/boot
            ```

        1. 对于安装到MMC来说, 分区和格式化以及挂载

            > MMC的设备名是: 
            
            运行 `lsblk -f | grep -Eo "(^| )mmcblk.( |$)"`, 比如得到的是 mmcblk1, 那么设备名就是 /dev/mmcblk1 

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
    1. 开始安装

        ```bash
        pacstrap /mnt base # 安装 base
        # 注意如果U盘的性能不够好, pacstrap 可能会要很久, 甚至自动退出, 如果这一步没走好, 后面会出现各种奇怪的问题.
        # pacstrap 这一步的时候, 建议开另外一个窗口运行 dmesg -HTPwku, 观察是否有错误产生. 
        # 如果 pacstrap 卡住 或者自动退出, 先运行 umount -vR /mnt, 然后重新挂载分区, 再继续 pacstrap, 如果还是不行, 建议更换一个好一点的U盘, :(
        genfstab -U /mnt >> /mnt/etc/fstab
        # 检查下 /mnt/etc/fstab, 除了 "#" 开头的之外, 确保里面只有两行内容, 一行是挂载 "/" 另一行是挂载 "/boot", 其他的(比如/dev/zram*)删掉
        arch-chroot /mnt

        # 安装 内核
        参考: https://github.com/cattyhouse/new-uboot-for-N1

        ```

1. **宿主是其他 arm64 系统**, 比如 armbian, raspbian 等

    > 受到 Jerry 的启发, 先启动进入 armbian, 然后将 archlinuxarm 的 base 解压到随便一个文件夹中, 
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
    ping -c 3 www.163.com # 确保网络通畅, DNS 解析没问题. 
    date  # 确保时间正确

    # 准备 archlinuxarm 环境
    cd ~ 
    curl -OL http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
    mkdir alarm
    bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C alarm
    rm alarm/etc/resolv.conf # 删除这个软连接, 它默认是指向一个一个动态文件
    cat /etc/resolv.conf > alarm/etc/resolv.conf # 重新生成此文件
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
    # 此时 archlinuxarm arm64 环境已经准备好, 接下来跳转到 **宿主是 archlinux arm64 系统**
    ```

## 设置 uboot

1. 确保可以打印和写入uboot env (uboot 环境变量)

    - 生成配置文件

        ```bash
        echo '/dev/mmcblk1 0x27400000 0x10000' > /etc/fw_env.config
        ```
        注意 mmcblk1 是根据前文提到的命令获取的, 此处只是举例, 以获取为准.

    - 打印 uboot env (**需要在N1本机上操作**)

        ```bash
        fw_printenv # 执行一下看是否可以成功输出 uboot env
        ```
    - 写入 uboot env (**需要在N1本机上操作**)

        ```bash
        fw_setenv # 后面会用到, 现在不需要执行.
        ```
2. 外挂 主线 uboot

    - 参考: https://github.com/cattyhouse/new-uboot-for-N1

## 收尾工作

1. 安装必要的软件并开机启动

    ```bash
    pacman -S haveged openssh
    systemctl enable haveged
    systemctl enable sshd
    ```

1. 临时允许ssh密码登陆root账户

    ```bash
    # 出于安全考虑, 建议启动N1后,删除这条,采用 ssh key 登陆
    echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
    ```

1. 设置ip

    > 以下配置包含了 DHCP 和 静态, 根据情况注释掉.

    ```bash
    # 有线
    # 设置 ip
    cat <<'EOF' > /etc/systemd/network/20-wired.network
    [Match]
    Name=eth0
    [Network]
    ## DHCP 根据需要注释
    DHCP=ipv4

    ## Static 根据需要注释
    Address=192.168.99.99/24
    Gateway=192.168.99.1

    [Link]
    ## set to yes to disable this network
    Unmanaged=no
    EOF

    # 启动 networkd 服务
    systemctl enable systemd-networkd

    # DNS 设置
    rm -f /etc/resolv.conf
    echo 'nameserver 1.1.1.1
    nameserver 8.8.8.8' > /etc/resolv.conf
    
    ## -------------------
    # 无线
    # 安装软件
    pacman -S wpa_supplicant crda firmware-phicomm-n1
    echo 'WIRELESS_REGDOM="CN"' >> /etc/conf.d/wireless-regdom
    # 设置 wpa_supplicant 服务
    wpa_passphrase Wi-Fi名称 Wi-Fi密码 | tee /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
    systemctl enable wpa_supplicant@wlan0
    # 设置ip

    cat <<'EOF' > /etc/systemd/network/25-wlan0.network
    [Match]
    Name=wlan0
    [Network]
    ## DHCP 根据需要注释
    DHCP=ipv4

    ## Static 根据需要注释
    Address=192.168.99.99/24
    Gateway=192.168.99.1

    [Link]
    ## set to yes to disable this network
    Unmanaged=no
    EOF

    # 启动 networkd 服务
    systemctl enable systemd-networkd

    # DNS 设置同上

    # wifi 相关操作 
    pacman -S iw # 安装 iw 
    iw dev wlan0 info # 连接信息, 频道等
    iw dev wlan0 link # 连接信息, 速度等
    iw dev wlan0 scan # 扫描
    iw dev wlan0 get power_save # 节能状态
    iw dev wlan0 set power_save off/on # 关闭开启节能
    iw phy phy0 info # Wi-Fi 卡的信息
    iw list # 列出 Wi-Fi 卡, 物理属性 
    iw dev # 列出 Wi-Fi 卡, 连接属性

    ```

1. 设置密码

    ```bash
    passwd root
    ```

1. 重启

    ```bash
    exit # 退出 chroot环境
    cd ~ && umount -vR /mnt # 卸载, 非常重要的一步, 否则直接关机, U盘数据会因为没有 sync 而部分丢失, 造成各种问题.
    exit # 如果是非 archlinux 宿主, 还要再运行一次 exit
    cd ~ && umount -vR ~/alarm  # 如果是非 archlinux 宿主, 需要退出 alarm chroot 环境
    poweroff # 关机, 然后拔插电源开机, 从MMC启动的话需要拔掉U盘, 因为U盘优先级更高.
    ``` 

## 首次启动

1. 网络

    ```bash
    ip a # 确认是否获取ip
    ping -c 3 114.114.114.114 # 确认网络是否接通
    ping -c 3 www.163.com # 确认 DNS 是否解析正常
    ```

1. 语言

    ```bash
    echo 'en_GB.UTF-8 UTF-8
    zh_CN.UTF-8 UTF-8' >> /etc/locale.gen

    locale-gen # 可能需要蛮久 
    localectl set-locale en_GB.UTF-8 # 设置系统级别的语言, GB 可以显示 24h, US 是 12h
    mkdir -p ~/.config && echo 'LANG=zh_CN.UTF-8' > ~/.config/locale.conf # 设置单个用户的语言(优先级更高)
    unset LANG && source /etc/profile.d/locale.sh  # 立刻生效
    ```

    > notes: 让终端显示 24 小时
        
    ```bash
    # 1. 本机设置 LC_TIME=en_GB.UTF-8
        localectl set-locale en_GB.UTF-8
        localectl set-locale LC_ALL=en_GB.UTF-8

    # 2. 通常 ssh 都有设置 SendEnv LANG 和 AcceptEnv LANG, 那么从比如 macOS 上 ssh 过去的话, 本机的 locale 会被临时覆盖, 所以这种情况, 我们在 macOS 上设置为英国的时间显示方式
        vim ~/.alias 
        export LANG=en_GB.UTF-8
    ``` 

1. 时间同步

    ```bash
    timedatectl set-timezone Asia/Shanghai  # 设置时区
    echo 'NTP=ntp1.aliyun.com ntp2.aliyun.com ntp3.aliyun.com ntp4.aliyun.com' >> /etc/systemd/timesyncd.conf # 使用阿里云的时间服务器
    timedatectl set-ntp true  # 同步时间, 因为 N1 没有 RTC 时钟
    date # 确认时间是否正确
    ```
1. 修改主机名

    ```bash
    hostnamectl set-hostname xxxx
    ```

# **系统优化**

1. cpu 变频

    ```bash
    pacman -S cpupower
    vim /etc/default/cpupower # 设置 governor='ondemand', min_freq="500MHz" , max_freq="2GHz"
    systemctl enable --now cpupower.service
    while true ; do cpupower -c all frequency-info --hwfreq --human | grep -E 'current CPU frequency' ; echo ; sleep 3 ; done # 观察变频是否工作, 需要用 root 运行才准确.
    ```

1. 提高网络性能

    > 注意 systemd 已经不再开机加载 /etc/sysctl.conf, 
    > 必须放到 /etc/sysctl.d/

    ```bash
    cat <<'EOF' > /etc/sysctl.d/99-custom-sysctl.conf
    # 本地不需要开 bbr, 容易让局域网设备之间网速变慢. 只需要在服务器上面开. 
	#net.core.default_qdisc = fq
    #net.ipv4.tcp_congestion_control = bbr
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

    # 减少输出到终端的 debug 信息(接显示器的时候)
    echo '# reduce debug message on console when connected to a monitor.
    kernel.printk = 3 4 1 3' >> /etc/sysctl.d/99-custom-sysctl.conf

    sysctl -p /etc/sysctl.d/99-custom-sysctl.conf # 立刻生效
    ```

1. 提高文件性能

    ```bash
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
    ```

1. 切换到 zsh

    > 我推荐 [zimfw](https://github.com/zimfw/zimfw)
    ```bash
    pacman -S zsh # 安装 zsh
    # 配置步骤作者写的很详细, 参见上面的 zimfw 的链接.
    ```

1. 更新一下系统

    ```bash
    pacman -Syyuu
    # 重启一遍
    reboot
    ```
1. 控制内核更新频率

    ```bash
    #  如果你不想更新 kernel 那么勤快, 可以设置下 IgnorePkg 
    echo 'IgnorePkg   = linux-phicomm-n1 linux-phicomm-n1-headers' >> /etc/pacman.conf
    # 这样下次运行 pacman -Syu 更新系统的时候, 如果有新的 kernel, 就不会更新, 只会给你警告.
    # 等你想更新的时候, 直接运行
    pacman -Sy linux-phicomm-n1 linux-phicomm-n1-headers
    ```

1. 优化日志存储

    ```bash
    cat <<'EOF' >> /etc/systemd/journald.conf
    # 日志写入内存, 减少磁盘读写
    Storage=volatile
    # 日志最多占用100M空间
    SystemMaxUse=100M
    RuntimeMaxUse=100M
    EOF
    ```
1. 用硬件 RNG 替代 haveged

    ```bash
    pacman -S rng-tools
    vim /etc/conf.d/rngd # 编辑,内容为:
    RNGD_OPTS="-o /dev/random -r /dev/hwrng -x jitter -x pkcs11 -x rtlsdr" 
    # 其中 -x 表示排除, man rngd 可以查询所有 source 的名称，这里我们只用 hwrng 这个 source
    # -r 表示从/dev/hwrng 生成 entropy
    # -o 表示喂给 /dev/random
    # 更新服务
    systemctl disable --now haveged
    systemctl enable --now rngd
    # 查看可用的 entropy, 一般在 4000 左右
    cat /proc/sys/kernel/random/entropy_avail
    ```
# **题外话**

## MMC安装之克隆安装

1. 用前面做好的archlinux的U盘启动N1, 给MMC [分区,格式化,挂载](https://github.com/cattyhouse/N1-install#安装-base-和-内核)

1. 用rsync克隆U盘的内容到 MMC 分区, 完成后, mmc的内容与U盘一模一样

    ```bash
    rsync -avPhHAX --numeric-ids  --delete --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/lost+found"} / /mnt
    ```

1. 修改相关文件的UUID

    > 注意, UUID修改后, 如果再运行上面的rsync命令, mmc的这两个文件会被rsync恢复为U盘的对应文件, 这也是rsync的魅力所在, 100%克隆.

    ```bash
    lsblk -f # 获取 mmcblk1p1 和 mmcblk1p2 的 UUID, 并记住
    vim /mnt/boot/uEnv.ini #将里面的UUID更换成 /dev/mmcblk1p2 的UUID
    vim /mnt/etc/fstab # 修改注释里面的sda1 为 mmcblk1p1, 并更换为对应的UUID, 修改注释里面的sda2 为 mmcblk1p2, 并更换为对应的UUID.
    # umount mmc 并关机
    cd ~
    umount -vR /mnt
    poweroff
    ```
1. 拔掉U盘, 拔插电源重启

## 在N1上用 archlinux 编译内核

> 感谢
> [@isjerryxiao](https://github.com/archlinux-jerry/pkgbuilds) for 二进制编译和下载, 
> [@RedL0tus闲鱼大佬](https://github.com/RedL0tus/Linux-Phicomm-N1) for PKGBUILD 修改, 
> [@MarvelousBlack惠狐](https://github.com/MarvelousBlack/firmware-phicomm-n1_PKGBULD) for dts 修改.

> 编译需要在N1上进行, 配合distcc会快很多, distcc需要在N1和x86主机上安装. [参考](https://archlinuxarm.org/wiki/Distcc_Cross-Compiling)

### 手动修改PKGBUILD - 不建议, 此处只是为了记录原理

1. 下载 archlinuxarm 的 linux-aarch64 源代码

    ``` bash
    mkdir -p ~/n1
    git clone https://github.com/archlinuxarm/PKGBUILDs
    cp -r PKGBUILDs/core/linux-aarch64 ~/n1/
    ```

1. 修改 *`~/n1/linux-aarch64/PKGBUILD`*

注意, 用了 新版本的 uboot 后, TEXT_OFFSET 相关的 patch 可以不用了.

    1. 在 **`cat "${srcdir}/config" > ./.config`** 后面插入
    
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

    1. 在 **`make INSTALL_DTBS_PATH="${pkgdir}/boot/dtbs" dtbs_install`** 后面增加

        ```bash
        # cp meson-gxl-s905d-phicomm-n1.dtb as dtb.img, for s905_autoscript to load
        cp "${pkgdir}/boot/dtbs/amlogic/meson-gxl-s905d-phicomm-n1.dtb" "${pkgdir}/boot/dtb.img" 
        ```

    1. 修改 *`pkgname=("${pkgbase}" "${pkgbase}-headers" "${pkgbase}-chromebook")`* 为 **`pkgname=("${pkgbase}" "${pkgbase}-headers")`** 避免编译chromebook的kernel

1. 编译 kernel 

    > 使用distcc的情况下大约3小时~4小时

    ```bash
    cd ~/n1/linux-aarch64
    makepkg -s
    ```

1. 安装 kernel

    ```bash
    pacman -U *.pkg.tar.xz
    ```

### 使用修改好的 PKGBUILD - 建议使用的方法

> 直接下载修改好的PKGBUILD编译

```bash
git clone https://github.com/archlinux-jerry/pkgbuilds
cd pkgbuilds/linux-phicomm-n1
makepkg -s
pacman -U *.pkg.tar.xz
```

### 直接安装编译好的二进制 - 懒人版

```bash
for item in $(curl -sL https://archlinux.jerryxiao.cc/aarch64/ | grep -E "linux-phicomm-n1-.*-aarch64.pkg.tar.xz" | grep -Ev "sig|git"  | cut -d \" -f2 | xargs); do curl -OL https://archlinux.jerryxiao.cc/aarch64/$item ; done
pacman -U *.pkg.tar.xz
```

## 用 N1 自动编译 Kernel 并推送消息到 telegram 的机器人

[自己做的一个简单的自动编译的脚本](kernel_compile/README.md)



