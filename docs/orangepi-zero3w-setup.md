# Orange Pi Zero 3W 系统初始化与 Microduck 安装

整理日期：2026-11-28。适用于本项目的 **Orange Pi Zero 3W / Allwinner A733 / 1 GB**，不是 Radxa Zero 3W。

本文按这次重新烧录后的实际操作整理。系统启动、SSH、引脚配置、改用户名和欢迎标题已经有实机输出；Wi-Fi 定时检查已运行，但断网后热点连接尚未实测。本轮 Microduck 安装曾停在 ONNX Runtime 下载失败，后文提供的 **1.23.2 是临时绕过方案**，不是已修复的正式安装流程。

不保存实际登录密码、Wi-Fi 密码、代理认证信息或私钥。文中的 IP、磁盘编号和文件路径需要按设备实际情况确认。

## 1. 准备并烧录系统（Mac）

本次使用的官方系统压缩包：

```text
Orangepizero3w_1.0.2_debian_trixie_server_linux6.6.98.7z
```

这是 Debian 13 Trixie Server（无图形桌面）镜像。先解压得到 `.img`，**不能直接把 `.7z` 写到卡里**。

查 SD 卡的磁盘编号：

```bash
diskutil list external physical
diskutil list
```

部分读卡器会被 macOS 标为 `internal, physical`，第一条可能没有输出。本次卡为 62.9 GB，某次编号是 `/dev/disk5`；这个编号不是固定值。

以下 `N` 必须替换为这次确认的卡编号，`if=` 必须指向实际解压出的镜像。**`dd` 会覆盖目标卡全部内容；先确认它不是 Mac 系统盘，并把读卡器从虚拟机断开。**

```bash
diskutil unmountDisk /dev/diskN

sudo dd \
  if="/实际路径/Orangepizero3w_1.0.2_debian_trixie_server_linux6.6.98.img" \
  of=/dev/rdiskN \
  bs=4m

sync
diskutil eject /dev/diskN
```

`dd` 运行时按 `Ctrl+T` 可显示进度；只有写入正常结束才继续 `sync` 和弹出。本次镜像写入量为 `4424990720 bytes`。写入成功不等于回读校验通过。

macOS 提示“此电脑不能读取你连接的磁盘”时点“忽略”，不要初始化；根分区是 macOS 默认不能读取的 ext4。

把卡插入主板，连接显示器、键盘和可靠的 5 V 电源启动。板端检查日志时需要已经启动的环境；没有其他启动介质时，空白卡不能提供 initramfs 或系统终端。

## 2. 首次联网与 SSH

在开发板本地终端登录官方镜像的账号。本次使用 `orangepi`；该镜像默认账号/密码见官方手册，随后按下文改成自己的账号。

用交互界面连接 Wi-Fi，密码不写入本文：

```bash
sudo nmtui
hostname -I
```

本次局域网 IP 是 `192.168.11.170`，不代表已经配置静态 IP 或 DHCP 保留。

安装、启用 SSH：

```bash
sudo systemctl enable --now ssh
```

如果提示服务不存在：

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

Mac 连接：

```bash
ssh orangepi@设备IP
```

如果 ping 能通但提示 `Connection refused`，先检查板端 SSH 服务。重刷同一设备后提示主机密钥变化时，确认 IP 确实对应刚重刷的板子，再清理该 IP 的旧记录：

```bash
ssh-keygen -R 设备IP
ssh orangepi@设备IP
```

这删除的是 Mac 上保存的远端身份记录，不会删除自己的 SSH 私钥。重刷会清掉板上原有的 `authorized_keys`，免密登录需要重新部署公钥。

## 3. 释放 UART0 并启用 I²C0

本机沿用原版 HAT，物理 8/10 脚使用 UART0 `/dev/ttyS0`；头部 IMU 使用物理 3/5 脚的 I²C0 `/dev/i2c-0`。详细针脚记录见 [hardware/wiring.md](../hardware/wiring.md)。

新系统默认把 UART0 用于内核日志和串口登录，不能直接交给舵机总线。先在 **SSH 终端**检查：

```bash
cat /boot/orangepiEnv.txt
cat /proc/cmdline
systemctl status serial-getty@ttyS0.service --no-pager
grep -n -E 'console|earlycon|earlyprintk' /boot/boot.cmd
find /boot -name '*i2c0*.dtbo'
```

本次镜像确认：`console=display` 只保留显示器控制台，`earlycon=off` 禁用串口 earlyprintk，且存在 `sun60i-a733-i2c0.dtbo`。其他内核/镜像需重新核对。

备份后编辑，保留原有 UUID、设备树等内容：

```bash
sudo cp -a /boot/orangepiEnv.txt /boot/orangepiEnv.txt.bak
sudo nano /boot/orangepiEnv.txt
```

添加或修改以下键，不要重复添加同名键；如果已有其他 `overlays`，将 `i2c0` 合并到同一行：

```ini
console=display
earlycon=off
overlays=i2c0
```

关闭串口登录并重启：

```bash
sudo systemctl mask --now serial-getty@ttyS0.service
sudo reboot
```

重新 SSH 登录检查：

```bash
cat /proc/cmdline
ls -l /dev/ttyS0 /dev/i2c-0
```

本次验证结果：启动参数没有 `console=ttyS0`、`earlyprintk`；`ttyS0` 为 `root:dialout`、权限 `660`；`i2c-0` 为 `root:i2c`、权限 `660`。这些只确认接口配置，不代表舵机/IMU 已完成回读。

## 4. 主机名、用户名和自动登录

### 主机名

如果还没改：

```bash
sudo hostnamectl set-hostname microduck
sudo nano /etc/hosts
```

检查 `/etc/hosts` 中 `127.0.1.1` 对应的旧主机名，将其改为 `microduck`，不要覆盖其他行。主机名不等于网络中的 DNS/mDNS 已经配置成功，仍可直接用 IP 登录。

### 将 orangepi 改名为 microduck

不要在 `orangepi` 登录会话里直接改自己的账号。`su -` 虽然会得到 root 提示符，但其父会话仍属于 `orangepi`。

先在当前账号设置 root 密码：

```bash
sudo passwd root
```

之后从另一个终端**直接以 root 登录**（本地控制台，或已经允许 root 登录的独立 SSH 会话）。不要为此盲目修改 SSH 安全策略。确认当前是独立 root 会话，再执行：

```bash
who
systemctl stop getty@tty1.service
loginctl terminate-user orangepi
pgrep -a -u orangepi
```

这些操作会断开旧 `orangepi` 会话。最后一条没有输出后再改名；仍有进程时先确认进程用途，不要直接杀死不明进程。

```bash
usermod -l microduck orangepi
groupmod -n microduck orangepi
usermod -d /home/microduck -m microduck
passwd microduck
```

密码由操作者自行设置。任一步失败就停止检查，不要继续执行余下命令。测试新账号登录：

```bash
ssh microduck@设备IP
```

### 修正 tty1 自动登录

本次系统在 `/usr/lib/systemd/system/getty@.service.d/override.conf` 里仍设置了 `--autologin orangepi`。在 `/etc` 为 tty1 单独覆盖，不修改系统包文件：

```bash
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d
sudo tee /etc/systemd/system/getty@tty1.service.d/zz-microduck.conf >/dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=-/sbin/agetty --noissue --autologin microduck %I $TERM
EOF

sudo systemctl daemon-reload
systemctl show getty@tty1.service -p ExecStart --no-pager
```

最终生效值应包含 `--autologin microduck`。不同名称的 drop-in 按文件名排序，`99-microduck.conf` 会排在 `override.conf` 前面，被后者覆盖；因此这里使用 `zz-`。如果曾建过旧的 `99-microduck.conf`，删除自己的旧文件或将其改名，避免重复配置。

这只影响显示器上的 tty1 自动登录，SSH 仍需认证；不必立即重启当前 SSH。

## 5. 自定义彩色 SSH 欢迎标题（可选）

系统标题由 `/etc/update-motd.d/10-orangepi-header` 生成，直接修改会被板级支持包更新覆盖。保留原文件，通过配置禁用它，再创建自己的标题。

以下配置追加一次即可；保留其他已禁用的 MOTD 项目：

```bash
sudo tee -a /etc/default/orangepi-motd >/dev/null <<'EOF'

# Use the custom Microduck header.
MOTD_DISABLE="${MOTD_DISABLE:+$MOTD_DISABLE }header"
EOF
```

橙黄渐变文字、黄色小鸭、灰色副标题，适合支持真彩色的 SSH 终端：

```bash
sudo tee /etc/update-motd.d/10-microduck-header >/dev/null <<'EOF'
#!/bin/sh
awk '
BEGIN {
    reset = "\033[0m"
    yellow = "\033[38;2;255;224;80m"
    muted = "\033[90m"
    bird[2] = "  __"
    bird[3] = "<(o )___"
    bird[4] = " ( ._> /"
    bird[5] = sprintf("  `---%c", 39)
    print ""
}
{
    for (i = 1; i <= 56; i++) {
        t = (i - 1) / 55
        green = int(140 + 84 * t)
        blue = int(40 + 40 * t)
        char = i <= length($0) ? substr($0, i, 1) : " "
        printf "\033[38;2;255;%d;%dm%s", green, blue, char
    }
    printf "%s%s%s\n", yellow, bird[NR], reset
}
END {
    printf "\n%s                      A LITTLE COMPANION%s\n\n", muted, reset
}
' <<'TITLE'
            _                    _            _
  _ __ ___ (_) ___ _ __ ___   __| |_   _  ___| | __
 | '_ ` _ \| |/ __| '__/ _ \ / _` | | | |/ __| |/ /
 | | | | | | | (__| | | (_) | (_| | |_| | (__|   <
 |_| |_| |_|_|\___|_|  \___/ \__,_|\__,_|\___|_|\_\
TITLE
EOF

sudo chmod 755 /etc/update-motd.d/10-microduck-header
/etc/update-motd.d/10-microduck-header
```

重新 SSH 登录生效，无须重启。系统温度、内存、IP 等仍由 `30-orangepi-sysinfo` 显示；`Last login` 是另一项登录信息。

## 6. 没有可连接 Wi-Fi 时自动开热点

### 行为与前提

- NetworkManager 管理 `wlan0`，本机查询支持 `WIFI-PROPERTIES.AP: yes`。
- 优先使用已保存且允许自动连接的 Wi-Fi；连接成功但无法访问互联网时，不切换热点。
- 掉线后扫描，留约一分钟给 NetworkManager 自动连接；仍未连接才启用热点。
- SSID 为 `Microduck-Setup-<CPU序列号SHA256前6位>`；读不到有效序列号时使用 `Microduck-Setup`。
- 热点中板子地址固定 `192.168.50.100/24`，不影响家中 Wi-Fi 的 DHCP 地址。
- 热点一旦启动就保持，不周期性断开它来扫描。回到已知 Wi-Fi 环境后，可重启板子重新尝试普通 Wi-Fi。

先确认：

```bash
nmcli device status
nmcli -f GENERAL.DEVICE,WIFI-PROPERTIES.AP device show
nmcli -f NAME,TYPE,AUTOCONNECT connection show
```

### 创建热点配置，不立即启动

以下适用于首次创建；已有 `microduck-hotspot` 时先查看现有配置，不要重复创建同名连接。密码使用 8～63 位 ASCII 字符，不保存到文档中。

```bash
read -r -s -p "设置热点密码（8～63位）: " AP_PASS
echo
if [ "${#AP_PASS}" -lt 8 ] || [ "${#AP_PASS}" -gt 63 ]; then
  echo "密码长度不符合要求，请重新执行"
else
  sudo nmcli connection add \
    type wifi ifname wlan0 \
    con-name microduck-hotspot ssid Microduck-Setup \
    connection.autoconnect no \
    connection.autoconnect-priority -1 \
    802-11-wireless.mode ap \
    802-11-wireless.band bg \
    wifi-sec.key-mgmt wpa-psk \
    wifi-sec.psk "$AP_PASS" \
    ipv4.method shared \
    ipv4.addresses 192.168.50.100/24 \
    ipv6.method disabled
fi
unset AP_PASS
```

`shared` 由 NetworkManager 为热点客户端提供 DHCP。热点 `autoconnect no`，实际开启由回退脚本控制；优先级 `-1` 仅作为低于普通默认优先级 `0` 的配置，不代替回退逻辑。以后新增普通 Wi-Fi 不需要单独设置为 100。

如果之前把家中 Wi-Fi 设为了优先级 100，可恢复为 0，并保留自动连接；连接名替换为实际名称：

```bash
sudo nmcli connection modify "S.9000_5G" \
  connection.autoconnect yes connection.autoconnect-priority 0
```

### 启动时生成本机后缀

```bash
sudo tee /usr/local/sbin/microduck-hotspot-name >/dev/null <<'EOF'
#!/bin/sh
set -eu
serial=""
if [ -r /sys/class/sunxi_info/sys_info ]; then
  serial=$(awk -F: '/sunxi_serial/ {
    gsub(/[[:space:]]/, "", $2)
    print $2
    exit
  }' /sys/class/sunxi_info/sys_info)
fi
name="Microduck-Setup"
case "$serial" in
  ""|*[!0-9a-fA-F]*) ;;
  *)
    suffix=$(printf '%s' "$serial" | sha256sum | cut -c1-6)
    name="$name-$suffix"
    ;;
esac
nmcli connection modify microduck-hotspot 802-11-wireless.ssid "$name"
echo "$name"
EOF

sudo chmod 755 /usr/local/sbin/microduck-hotspot-name
sudo /usr/local/sbin/microduck-hotspot-name
```

配置本身会留在克隆镜像里，但回退脚本在启用热点前会重新读取本板序列号、更新 SSID。不同板的后缀通常不同；6 位哈希不是绝对唯一标识，不能当作认证凭据。

### 自动回退脚本

```bash
sudo tee /usr/local/sbin/microduck-wifi-fallback >/dev/null <<'EOF'
#!/bin/bash
set -euo pipefail
export LC_ALL=C

connected() {
  [[ "$(nmcli -g GENERAL.STATE device show wlan0)" == 100* ]]
}

connected && exit 0
nmcli device wifi rescan ifname wlan0 || true

for ((i=0; i<30; i++)); do
  connected && exit 0
  sleep 2
done

/usr/local/sbin/microduck-hotspot-name
connected && exit 0
nmcli --wait 20 connection up microduck-hotspot ifname wlan0
EOF

sudo chmod 755 /usr/local/sbin/microduck-wifi-fallback
```

这里依赖 NetworkManager 本身连接已保存的网络，没有另外实现逐个密码尝试。`100` 表示接口处于已连接状态，包含热点自身，因此热点启用后后续检查会直接退出。

### 定时器

```bash
sudo tee /etc/systemd/system/microduck-wifi-fallback.service >/dev/null <<'EOF'
[Unit]
Description=Microduck Wi-Fi fallback hotspot
Wants=NetworkManager.service
After=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/microduck-wifi-fallback
TimeoutStartSec=120
EOF

sudo tee /etc/systemd/system/microduck-wifi-fallback.timer >/dev/null <<'EOF'
[Unit]
Description=Check Microduck Wi-Fi connection

[Timer]
OnBootSec=45s
OnUnitInactiveSec=30s

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now microduck-wifi-fallback.timer
sudo systemctl start microduck-wifi-fallback.service
systemctl status microduck-wifi-fallback.timer --no-pager
sudo journalctl -u microduck-wifi-fallback.service -n 20 --no-pager
```

不必重启。当前 Wi-Fi 已连接时，检查应成功退出；oneshot 显示 `Deactivated successfully` 是正常完成。本机已验证该路径，**尚未验证断网后的 AP 激活、DHCP 与客户端连接**。

完整测试：临时关闭路由器 Wi-Fi或移动到覆盖外，等待约 1～2 分钟，连接热点，再执行：

```bash
ssh microduck@192.168.50.100
```

不要以 `nmcli device disconnect wlan0` 模拟自然掉线：该命令会影响接口自动连接行为，测试条件不同。

## 7. 下载 Microduck 安装脚本

只需将运行程序装到板子，不必克隆整个 RL 训练仓库。发布源是 **Shunjun/microduck**，脚本默认安装其发布版本，而非原上游的发布包。

```bash
curl -fL --retry 3 \
  https://raw.githubusercontent.com/Shunjun/microduck/main/scripts/install.sh \
  -o /tmp/microduck-install.sh
```

如果出现 `curl: (35) Recv failure: Connection reset by peer`，可配置实际可用的局域网代理再下载。代理必须允许来自板子的连接；不能用板子上的 `127.0.0.1` 指代 Mac/Windows 上的代理。

```bash
export https_proxy="http://用户名:密码@代理电脑局域网IP:端口"
export http_proxy="$https_proxy"
```

如果代理无需认证，去掉 `用户名:密码@`。上述占位符必须替换，带特殊字符的认证信息需进行 URL 编码。代理只用于当前 shell，不会自动成为 systemd 服务的环境。

下载完成后安装；无代理时直接使用 `sudo sh /tmp/microduck-install.sh`，有代理时：

```bash
sudo env https_proxy="$https_proxy" http_proxy="$http_proxy" \
  DUCK_REPO=Shunjun/microduck \
  sh /tmp/microduck-install.sh
```

安装器引导 updaterd 下载、验签、解压和运行安装 hook。默认流程会安装并启动服务；这次命令没有设置 `DUCK_NO_START`。如果显式使用 `DUCK_NO_START=1`，安装器最终会禁用并停止相关服务，不能期待重启后自动运行。

## 8. ONNX Runtime 1.23.2：临时方案

> **临时绕过，不是正式修复。** 本次安装包的 preinstall 仍从下方 GitHub 地址下载 1.28.0，实际返回 404。仓库安装 hook 的下载源尚未在本流程中修复；以后修复正式依赖安装逻辑后应重新核对本节是否还适用。

```text
https://github.com/microsoft/onnxruntime/releases/download/v1.28.0/onnxruntime-linux-aarch64-1.28.0.tgz
```

失败特征是：发布包已经下载并通过校验，随后 `RunningPreHook` 报 `ONNX Runtime absent is below 1.23`。这不是动作模型缺失，也不能靠反复下载同一个无效地址解决。

本次运行端要求 ONNX Runtime 至少 1.23，临时从 PyPI `onnxruntime==1.23.2` 的 ARM64 wheel 提取 **C 动态库**，放到原 hook 检查的路径。不运行 Python 绑定，因此选中的 cp311 wheel 不要求板上 Python 恰好为 3.11；这里也不是用 pip 安装 Python 依赖。

保留前面的代理环境变量，不加 sudo 下载：

```bash
python3 -u - <<'PY'
import urllib.request, json, hashlib, zipfile, io

print("读取 PyPI 文件信息……", flush=True)
with urllib.request.urlopen(
    "https://pypi.org/pypi/onnxruntime/1.23.2/json", timeout=60
) as r:
    meta = json.load(r)

asset = next(a for a in meta["urls"]
             if "aarch64" in a["filename"] and "cp311-" in a["filename"])

print("下载 ARM64 运行库……", flush=True)
with urllib.request.urlopen(asset["url"], timeout=60) as r:
    data = r.read()

if hashlib.sha256(data).hexdigest() != asset["digests"]["sha256"]:
    raise RuntimeError("SHA256 校验失败")

with zipfile.ZipFile(io.BytesIO(data)) as z:
    lib = z.read("onnxruntime/capi/libonnxruntime.so.1.23.2")

with open("/tmp/libonnxruntime.so.1.23.2", "wb") as f:
    f.write(lib)

print("下载、校验、提取成功", flush=True)
PY
```

只有输出成功后再安装：

```bash
sudo install -m 644 /tmp/libonnxruntime.so.1.23.2 /usr/local/lib/
sudo ln -sfn libonnxruntime.so.1.23.2 /usr/local/lib/libonnxruntime.so
sudo /usr/sbin/ldconfig
ls -l /usr/local/lib/libonnxruntime.so*
```

`install`、`ln`、`ldconfig` 成功时可能没有输出；至少检查链接指向和文件是否存在。需要进一步验证动态库可加载时执行：

```bash
python3 -c 'import ctypes; ctypes.CDLL("/usr/local/lib/libonnxruntime.so"); print("ONNX Runtime 动态库可加载")'
```

然后重新运行第 7 节的安装命令。当前 shell 的代理虽然显式传给安装器，**不能据此保证发布 hook 或以后 systemd 服务的下载也会继承它**；预装此动态库是为了绕过这次失效的依赖下载步骤。

PyPI 来源：[版本文件](https://pypi.org/project/onnxruntime/1.23.2/#files)、[文件元数据与 SHA256](https://pypi.org/pypi/onnxruntime/1.23.2/json)。以后运行端如果提高最低版本，需要重新选择兼容版本，不能长期假定 1.23.2 够用。

## 9. 安装后的核对与待办

以下是安装结束后的检查步骤，不代表本轮已经全部通过：

```bash
systemctl status robotd studio --no-pager -l
sudo journalctl -u robotd -u studio -b -n 100 --no-pager
cat /etc/robot/robotd.toml
```

确认基础配置的 `[bus]` 为本机硬件，不要整份覆盖其他参数：

```toml
[bus]
driver = "feetech1910"
port = "/dev/ttyS0"
```

本地当前模板端口仍是原板的 `/dev/ttyS2`，不能认为安装器已经替你识别香橙派接线。网页设置可能另写到 `/var/lib/robot/robotd.toml`，启动参数也可能覆盖端口；通过服务日志和 `systemctl cat robotd` 检查实际配置，不再添加旧的 commissioning.conf/`--servo-debug` 作为首次安装的常规步骤。当前代码在缺少有效标定时有装机引导路径，实际行为以安装的发布版本为准。

调整配置后按需重启：

```bash
sudo systemctl restart robotd studio
```

默认 Studio 地址为 `http://设备IP:8080`；热点下为 `http://192.168.50.100:8080`。舵机零位和 IMU 标定仍需要实际操作，安装脚本不会生成真实标定结果。此前的 0.23.2 安装和标定故障是历史记录，不代表本轮已恢复标定。

安装器若提示将当前用户加入 `robot` 组，重新 SSH 登录或执行 `newgrp robot` 后再检查 IPC：

```bash
robotctl version
robotctl health
```

尚待验证/处理：全舵机及 IMU 回读、首次标定、动作模型导入与实机步态、热点断网切换、HAT 音频和摄像头配置。服务启动不等于机器人已能行走。

## 10. 本次启动故障的排查记录

旧系统出现 `gave up waiting for root file system device` 并进入 `(initramfs)`。可用以下只读命令定位：

```sh
cat /proc/cmdline
cat /proc/partitions
blkid
ls /dev/mmc*
dmesg | grep -i -E 'mmc|sdhci|sdc|card'
```

本次 `/dev/mmc*` 不存在，照片有 `mmc1: error -95 whilst initialising SD card`，说明内核初始化卡失败，尚未进入正常挂载根分区的步骤。不要把其他控制器或音频驱动的所有报错都解释为 SD 卡故障。

电脑端能看到分区、能挂载，不能证明全卡健康。Linux 虚拟机读配置曾报 I/O 错误、USB 读卡器反复 reset；重新连接读卡器后恢复读取，且启动配置 rootdev UUID 与分区 UUID 一致。

最后在**同一张卡**上重刷上述官方镜像后成功启动。证明板载卡槽并非完全不能工作，但不能据此判定原故障的唯一原因，也不能保证卡长期健康。若再次出现读写错误，优先使用可靠读卡器和另一张卡对照；有 I/O 错误且需要保留数据时先做恢复副本，不直接运行 `fsck -y`。

官方资料归档：[Orange Pi Zero 3W 手册与原理图](../hardware/orangepi-zero3w/README.md)。手册 PDF 第 9 页（正文第 2 页）注明 TF 卡最高支持 128 GB；PDF 第 42 页（正文第 35 页）给出插入已烧录镜像的卡后启动的流程，并未直接声称“无卡无画面就能证明硬件正常”。
