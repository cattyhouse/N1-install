# 用 N1 自动编译 Kernel 并推送消息到 telegram 的机器人

## 准备PKGBUILD

```bash
git clone https://github.com/archlinux-jerry/pkgbuilds
cp -r pkgbuilds/linux-phicomm-n1 ~/
mkdir -p ~/linux-phicomm-n1/binary
```

## telegram 机器人脚本, 命名为 botmsg

```bash
#!/bin/bash
# /usr/bin/botmsg

TOKEN='xxxxxxxxxxxxxxxx'
CHAT_ID='xxxxxxxx'
[ $# -ge 1 ] && MESSAGE="$*" || MESSAGE="$(cat -)" 
# 如果参数大于等于1, 那么发送的消息即为参数, 如果参数小于1, 也就是参数为空, 那么从管道读取
URL="https://api.telegram.org/bot$TOKEN/sendMessage"
curl -s -X POST $URL -d chat_id=$CHAT_ID -d text="$MESSAGE"
```

## 自动检查更新并编译的脚本, 命名为 update.sh, 并且放到 linux-phicomm-n1 文件夹

```bash
#!/bin/bash
working_dir="$HOME/linux-phicomm-n1"
cd ${working_dir}
# 先运行一个 if 判断, 如果获取不到 kernel 版本号. 就终止, 并发送消息到 telegram bot
if [[ -z "$(curl -4sL https://www.kernel.org/releases.json | grep -A1 latest_stable | tail -n1 | tr -d '"' | cut -d \: -f 2 | xargs)" ]];then
echo "$(date) : not able to get kernel version" | tee -a update.log | botmsg > /dev/null 
exit 0
fi

# 从 kernel.org 获取最新的内核版本
latest_pkgver=$(curl -4sL https://www.kernel.org/releases.json | grep -A1 latest_stable | tail -n1 | tr -d '"' | cut -d \: -f 2 | xargs)
# 从当前的 PKGBUILD 获取当前的 内核版本
current_pkgver=$(grep -E "^pkgver" PKGBUILD | cut -d "=" -f2)

if [[ ${latest_pkgver%.*} != "${current_pkgver%.*}" ]]; then
    # 两个内核版本取大版本的值做比较, 比如 5.3.1 取值就是 5.3
    # 如果官方大版本号变了, 一般意味着需要更新 config
    echo "$(date) : major branch updated, need new kernel config" | tee -a update.log | botmsg > /dev/null
    exit 0
else
    if [[ "${latest_pkgver}" = "${current_pkgver}" ]]; then
        # 如果从官网获取的版本号等于当前的 PKGBUILD 的版本号, 就不需要更新了, 直接退出
        echo "$(date) : kernel already up to date" | tee -a update.log
        exit 0
    else
    # 检查到了更新的版本号
    # 获取各自的 md5sum
	latest_pkgver_md5sum=$(curl -4sL https://cdn.kernel.org/pub/linux/kernel/v5.x/patch-${latest_pkgver}.xz | md5sum | cut -d "-" -f1 | xargs)
	current_pkgver_md5sum=$(curl -4sL https://cdn.kernel.org/pub/linux/kernel/v5.x/patch-${current_pkgver}.xz | md5sum | cut -d "-" -f1 | xargs)
        sed -i "s/pkgver=${current_pkgver}/pkgver=${latest_pkgver}/g" PKGBUILD # 更新 PKGBUILD 里面的版本号
        sed -i "s/${current_pkgver_md5sum}/${latest_pkgver_md5sum}/g" PKGBUILD # 更新 md5sum
        sed -i "s/^pkgrel=.\+/pkgrel=1/g" PKGBUILD # 重置自定义的小版本
    fi
fi

# 开始编译, 如果中途出错, 删除原始文件, 并且通知 telegram bot
echo "$(date) : start building kernel ${latest_pkgver}" | tee -a update.log | botmsg > /dev/null
makepkg >/dev/null 2>&1 || { cd ${working_dir} ; rm -rf pkg src linux-${latest_pkgver%.*}.tar.xz patch-${latest_pkgver}.xz ; echo " $(date) : kernel failed to build" | tee -a update.log | botmsg > /dev/null ; exit 1 ; }

cd ${working_dir}
# 编译完了也要删除原始文件, 因为我的 N1 只有 8GB 空间.
rm -rf pkg src linux-${latest_pkgver%.*}.tar.xz patch-${latest_pkgver}.xz
# 将编译好的内核放到 binary 文件夹
mv -f linux-phicomm-n1-${latest_pkgver}-1-aarch64.pkg.tar.xz linux-phicomm-n1-headers-${latest_pkgver}-1-aarch64.pkg.tar.xz binary
# 如果一切都 ok, 记录 log 并通知 telegram bot
echo "$(date) : finished building kernel : ${latest_pkgver}" | tee -a update.log  | botmsg > /dev/null

```

## cron 设置

```bash
pacman -S cronie
systemctl enable cronie
systemctl start cronie
crontab -e 填入下面内容, 每小时运行一次 
0 * * * * ~/linux-phicomm-n1/update.sh
```