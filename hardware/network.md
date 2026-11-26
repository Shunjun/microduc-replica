# 主机名、Wi-Fi 与网络

## SSH 与地址

| 项目 | 历史记录 |
| --- | --- |
| 主机名 | `orangepizero3w` |
| 登录用户 | `orangepi` |
| 最近聊天中成功使用的局域网 IP | `192.168.11.170` |
| Studio 地址 | `http://192.168.11.170:8080`，以设备当前 IP 为准 |

没有证据确认 `192.168.11.170` 已配置为静态地址或 DHCP 保留，不能当作永久 IP。

Mac 曾将 `orangepizero3w` 解析到 `198.18.2.16`，SSH 报连接关闭。这个地址不能当作板子的真实 LAN 地址；实际联调使用 `192.168.11.170`。后来 GitHub SSH 的 `198.18.0.26` 也属于另一条网络排查记录，不是设备地址。

历史问题还包括在 Mac 的 `/etc` 查找板子配置文件。先确认 `hostname`，或直接使用 `ssh orangepi@设备IP '命令'`，避免操作错机器。

## NetworkManager 与热点要求

用户贴出的实机结果：

```text
nmcli version: 1.52.1
wlan0: connected
连接名: S.9000_5G
WIFI-PROPERTIES.AP: yes
2.4 GHz / 5 GHz: yes
```

用户确定的行为：

- 优先连接已保存、当前可用的 Wi-Fi。
- 扫描后没有认识的无线网络时，释放兜底热点。
- 热点 SSID 基础名为 `Microduck-Setup`。
- 后缀根据 **CPU/SoC 序列号**生成，避免只靠镜像中的 machine-id；换板后应跟随硬件。
- 读取失败时按讨论约定不加后缀，使用基础名。
- 热点网络中，**鸭子自身地址固定为 `192.168.50.100`**。这不是家中 Wi-Fi 的静态地址，也不是客户端分配地址范围的完整定义。

相关设备脚本/服务名：

```text
/usr/local/sbin/microduck-hotspot-name
/usr/local/sbin/microduck-wifi-fallback
microduck-wifi-fallback.timer
NetworkManager profile: microduck-hotspot
```

用户后续回复“好了”，记录为用户报告配置完成。最终脚本内容、hash 算法与后缀长度、定时器间隔、热点密码和实际断网切换结果未在当前仓库留档，不能按名字重新猜一版当作现有实现。

曾执行 `microduck-hotspot-name` 报 `no such connection profile`，当时用户说明还没执行前面的热点创建命令；不是 CPU 序列号错误。

## CPU 序列号来源

`/sys/firmware/devicetree/base/serial-number` 在本机未找到。实机可读取：

```bash
cat /sys/class/sunxi_info/sys_info | grep sunxi_serial
```

用户提供的返回值：

```text
sunxi_serial : 3c8a1c5558b2821c00007c0000000000
```

不把这个硬件序列号写死为其他板子的热点后缀；应在每块板启动时读取本机值。

## 下载与代理排查

- 直连 `raw.githubusercontent.com`：DNS 返回 `185.199.108.133` 至 `185.199.111.133`，但 TLS ClientHello 后连接被重置。
- 使用 HTTP 代理后，日志显示 CONNECT 成功、TLS 证书验证成功；不能继续把所有失败都归因于 DNS。
- 微软 ONNX Runtime GitHub 仓库/API/Release 的 404 在双方请求中都复现，已改用 PyPI 的已发布包。该事件记录于 2026-11-25/26，不代表未来一直不可访问。
- 当前 shell 的 curl/Python urllib 使用 `http_proxy`、`https_proxy`；经 sudo 执行安装器时需要显式传入。
- 安装前置 hook 清空环境，因此下载主程序走代理成功，并不意味着 hook 内部 curl 也继承代理。

此目录不记录代理认证、Wi-Fi 密码或 SSH 密码。自动更新守护进程也不会自动继承 SSH shell 的代理变量，需要单独配置时再记录。
