# 原版 HAT 音频与 Orange Pi Zero 3W 针脚

核对日期：2026-11-27。下表全部是 **40pin 排针物理编号**。

## 核对依据

- 本目录 [Orange Pi V1.2 原理图](<原理图/OPI ZERO 3W V1_2_原理图.pdf>)：PDF 第 18 页 `18_EX_GPIO` 给出物理排针与网络对应；第 9 页给出 A733 的 PB 引脚复用功能。
- 原版 [RPI Robot HAT](https://github.com/pollen-robotics/elec_RPI_Robot_HAT/tree/6de86283297e4c49a9da8312cd3395ab83c437e3) D1 原理图：第 2 页 `main` 给出 40pin 到 `IO_XX` 的连接；第 6 页 `audio` 给出 TLV320AIC3104 与麦克风/扬声器的连接。本机来源为 `MicroDuck/hardware-reference/RPI_Robot_HAT/D1/ASE01187-D1_elec_RPI_Robot_HAT_SCH.pdf`。
- 本目录 [用户手册](用户手册/OrangePi_Zero3W_A733_用户手册_v1.0.pdf)：正文第 98 页（PDF 第 105 页）确认 I²C0 的物理 3/5 脚；它不是音频初始化教程。

## 连接关系

| 用途 | 物理脚 | HAT 网络/音频芯片端 | 香橙派引脚 | 可复用功能 |
| --- | --- | --- | --- | --- |
| 音频芯片控制 SDA | 3 | IO_02 → SDA | PB3 | TWI0/I²C0 SDA |
| 音频芯片控制 SCL | 5 | IO_03 → SCL | PB2 | TWI0/I²C0 SCK |
| 音频位时钟 | 12 | IO_18 → BCLK | PB5 | I2S0-BCLK |
| 左右声道/帧时钟 | 35 | IO_19 → WCLK | PB6 | I2S0-LRCK |
| 录音数据，HAT → 主控 | 38 | IO_20 ← DOUT | PB8 | I2S0-DIN0 |
| 播放数据，主控 → HAT | 40 | IO_21 → DIN | PB7 | I2S0-DOUT0 |

四根 I²S 线和两根 I²C 线的物理位置、信号方向均对应。这里确认的是上述 D1 资料的音频接口；用户实物 HAT 修订尚未留档。

HAT 的麦克风先进入 TLV320AIC3104，编码后的录音数据通过物理 38 脚送回主控；麦克风并非独立 USB 设备。扬声器播放使用相反方向的数据线（40 脚），经音频芯片及功放输出。

HAT D1 音频页中，音频芯片 MCLK 来自板载 Y1 **12 MHz 振荡器**。香橙派物理 7 脚虽然支持 I2S0-MCLK，但这块 HAT 的 MCLK 不需要从 7 脚接入。设备树中的 codec 时钟应与实际 12 MHz 输入一致。

## 系统仍需要配置的部分

1. 将 PB5/PB6/PB7/PB8 复用为 I²S0，避免被其他外设占用。
2. 启用相应 I²S/AHUB 音频通路；具体设备树节点以所装 A733 内核为准。
3. 在已启用的 I²C0 下声明音频芯片，通常为 `0x18`，并确认 TLV320AIC3x 驱动支持该芯片。
4. 建立 CPU DAI 与 codec DAI 的声卡连接，配置 I²S 格式、时钟与录放路由。
5. 确认 ALSA 枚举后，再测试录音和播放。

仓库现有 `microduck/deploy/audio/aic3104-i2c3.dts` 明确针对 Radxa/RK3566，使用 `i2s3_2ch` 和 `i2c3`。不能仅因排针相同就把它直接加载到 A733；需要替换平台节点、pinctrl 和时钟配置。

此前实机 `/dev/i2c-0` 扫到 `0x18`，只能证明该地址有应答，不能证明 I²S 或声卡已经配置成功。

只读检查：

```sh
cat /proc/asound/cards
arecord -l
aplay -l
```

本次未连接设备修改设备树或驱动，也未进行实机录放测试。
