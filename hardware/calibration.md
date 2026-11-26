# 首次装机与标定

## 已遇到的启动问题

用户安装 `daemon 0.23.2` 后执行 `systemctl enable --now robotd studio`，robotd 报：

```text
invalid Feetech calibration; no motor port was opened
read Feetech calibration /etc/robot/feetech1910.json: No such file or directory
```

这次失败发生在打开串口之前，不能据此判断 UART 或舵机通信故障。原因是首次装机尚无完整标定文件。

## 使用过的装机启动方式

为 robotd 添加专用 drop-in，而不是同时运行第二个串口进程：

```bash
sudo mkdir -p /etc/systemd/system/robotd.service.d
sudo tee /etc/systemd/system/robotd.service.d/commissioning.conf >/dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=/opt/robot/daemon/current/bin/robotd --socket /run/robotd.sock --servo-debug --no-policy --port /dev/ttyS0
EOF

sudo systemctl daemon-reload
sudo systemctl reset-failed robotd
sudo systemctl restart robotd studio
```

用户随后确认“启动了”。记录为调试启动成功，**未记录到 15 颗舵机全部回读、标定完成或进入普通策略模式**。

此模式允许标定文件不完整时装机调试，暂停普通动作策略；没有身体 IMU 也能先做舵机调试。关闭页面或退出调试不等于自动恢复行走。

## 用户确定的零位交互

1. 调零页的虚拟鸭子固定显示标准参照姿态。
2. 用户手动把实物摆成标准姿态。
3. “将当前姿态设为零位”读取稳定关节角，计算并保存本机零位。
4. “回归零位”让舵机回到当前已保存标定所对应的标准姿态。
5. 身体 IMU 使用单独的标定按钮；鸭子直立桌面、保持静止。

“标准零位姿态”不代表每个训练关节角都是数学上的 0 rad；保存算法按项目标准姿态计算偏移。

## 支撑、保持与方向

- 用户希望标定时可选择卸力或弱保持，避免全部关节自然下垂。
- 当前实现提供卸力、固定保持、柔顺跟随。P 只表示位置增益，不是限定扭矩百分比。
- 柔顺跟随依据位置偏差更新保持目标，无法可靠区分手推和重力，仍可能下垂。
- 启用保持/回零前，先核对实际关节方向和机械限位并扶稳实物；不以完整 0～4095 编码器量程代替装机安全范围。

标定保存 `/etc/robot/feetech1910.json`，包括 15 个关节的零位/方向/限位以及所需单位、身体 IMU 安装参数。方向/限位的会话应用和最终保存行为见 [Studio 说明](../microduck/studio/README.md)。

## 标定后恢复普通模式

仅在完整标定、必需参数和硬件反馈已经确认后，移除**这个装机用** drop-in：

```bash
sudo rm /etc/systemd/system/robotd.service.d/commissioning.conf
sudo systemctl daemon-reload
sudo systemctl restart robotd
```

不要使用 `systemctl revert robotd` 清掉未知的其他本机配置。恢复普通模式也不代表飞特 RL 步态已经验证，需要另行加载、验证模型。

标定 JSON 应备份为该设备的实际数据，不要用示例里的 `null`、另一只鸭子的零位或厂家 BAM 的 `q_offset` 填充。
