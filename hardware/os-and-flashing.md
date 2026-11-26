# 系统与 TF 卡烧录

## 镜像选择记录

最初主控没有刷系统。聊天中讨论过 Debian Bookworm/XFCE 桌面镜像、Trixie Server 和 Bullseye；其中明确出现的 Server 文件名为：

```text
Orangepizero3w_1.0.2_debian_trixie_server_linux6.6.98.7z
```

主控上的资料检查曾发现 `/boot/dtb-6.6.98-sun60iw2/`。本记录保留镜像选择和观察，不替代实际系统版本检查：

```bash
cat /etc/os-release
uname -a
```

Bookworm、Trixie、Bullseye 分别是 Debian 12、13、11 的代号，镜像必须匹配 **Orange Pi Zero 3W A733**，不是 Radxa 的同名 Zero 3W。

## Mac 烧录过程

- 用户未使用 balenaEtcher，采用 `dd` 写镜像。
- 先解压 `.7z`，烧录其内部磁盘镜像；不能把 `.7z` 压缩包直接写入卡。
- 当时 `diskutil list external physical` 没输出，但完整 `diskutil list` 能看到 62.9 GB 的卡，被系统标为 internal、设备 `/dev/disk4`。
- **`disk4` 只是当时的枚举结果，不作为以后烧录命令的固定目标。** 每次插卡都必须重新确认目标盘。
- 用户贴出的最终 `dd` 输出为 `4424990720 bytes transferred` 后回到提示符。
- `sync` 用于刷新尚未落盘的写入；完成后再弹出卡。`dd` 进程结束本身不等于板子已经成功启动。

后续用户已通过直接连接显示器进入系统，因此本机确实有过成功启动；不根据 LED 闪烁推断启动阶段。

## Linux 虚拟机挂卡

用户曾把 TF 卡交给 Linux 虚拟机修改配置：

```text
挂载目录：/media/toby/opi_root
当时分区：/dev/sdb1
文件系统：ext4
```

在虚拟机里编辑卡上的配置应使用：

```text
/media/toby/opi_root/etc/...
```

不是虚拟机自己的 `/etc/...`。分区和挂载目录可能随插拔变化，可用 `lsblk -f`、`findmnt` 重新确认。

## USB / ADB 与显示器的联调历史

- 用户确认使用可传数据的线缆，并强调板上只有一个 Type-C 口；不能仅凭线缆判断 Linux 已枚举为 USB 设备。
- Mac 的 `ioreg -p IOUSB -w0`、`ioreg -r -c IOUSBHostDevice -l -w0` 没看到新增设备；`adb devices` 为空。
- 镜像中有 `/etc/adb_conf.sh` 和 `/usr/bin/adbd`；曾创建并启用 `orangepi-usb-adb.service`。
- 后续日志显示服务被执行，UDC 名为 `4100000.udc-controller`，并写入 gadget 的 `UDC`；旧 `otg manager/.../usb_device` 路径不存在。
- **这些日志不能证明 Mac 已枚举或 ADB 已连接。** 最终用户直接使用显示器进入系统，明确决定不再纠结 ADB。
- 曾经拔电关机，调试文件一度为 0 字节；后来出现完整服务日志。不能据此倒推那几次一定没有启动。

LED 颜色/闪烁含义、精确启动耗时、最终 ADB 改动是否仍留在系统中，未保存可核验结论，待需要时再查设备和对应手册。

不要直接执行原版 Radxa 的 `scripts/setup-board.sh` 去设置香橙派设备树。仅改 robotd 参数通常重启服务即可；修改设备树后再按实际需要重启系统。
