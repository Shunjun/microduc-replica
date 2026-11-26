# 接线与引脚

## HAT 与香橙派排针

以下编号是**排针物理脚号**，不是 Linux GPIO 号，也不是舵机寄存器地址。

| 功能 | HAT 使用的物理脚 | 本机配置 |
| --- | --- | --- |
| 串口 TX | 8 | UART0 |
| 串口 RX | 10 | UART0 |
| 头部 I²C SDA | 3 | I²C0，已有资料对应 PB3/TWI0-SDA |
| 头部 I²C SCL | 5 | I²C0，已有资料对应 PB2/TWI0-SCK |

UART 使用 `/dev/ttyS0`；头部 I²C 使用 `/dev/i2c-0`。不要把原 Radxa 的 `/dev/ttyS2`、I²C3 直接照搬到 A733。

I²C 映射的既有核对依据：原 HAT main/sensors 原理图和用户提供的《OrangePi_Zero3W_A733_用户手册_v1.0.pdf》，此前记录为印刷 98 页 / PDF 105 页。PDF 不在当前根仓库，本页不冒充重新核验过原图。

## 设备配置

香橙派上的 `/etc/robot/robotd.toml`：

```toml
[bus]
driver = "feetech1910"
port = "/dev/ttyS0"
```

目前随包模板仍可能是原版端口 `/dev/ttyS2`，首次安装要核对设备上的有效配置。运行端用户配置层也可能覆盖基础文件，见[部署记录](deployment.md)。

`/dev/ttyS0` 是串口设备节点，不是可编辑的文本配置文件。UART 是否启用由设备树和系统引脚复用决定，项目驱动不会自动配置。

## 调试控制台与总线占用

如果 UART0 被系统串口登录控制台使用，会与 HAT 的舵机总线冲突。需要确认串口 getty 和内核 console 配置；历史讨论涉及 `console=display`、`earlycon=off`，**最终启动配置内容尚未完整留档**，不提供盲目覆盖启动文件的命令。

修改设备树/UART 启用状态通常需要重启板子；仅修改 `robotd.toml` 的驱动/端口，重启 robotd 即可。

运行时串口由 **robotd 独占**。Studio 通过 robotd 调试接口访问，不另开串口；不要同时运行串口调试工具和正常控制服务。

头部 I²C 与身体 IMU 的舵机总线是两条不同链路，见[IMU 记录](imu.md)。实现依据见[飞特硬件设计](../microduck/docs/design/feetech-hardware.md)。
