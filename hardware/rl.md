# RL 与硬件配置

## 统一舵机选择

[servo_config.py](../microduck_rl/src/mjlab_microduck/robot/servo_config.py) 中：

```python
SERVO_DRIVER = "feetech1910"  # "xl330" | "feetech1910"
```

`microduck_constants.py` 的普通/回差执行器与 `scripts/infer_policy.py` 的 BAM 模型选择、P 和电压默认值读取这个入口。运行端机器人则由 TOML 的 `[bus].driver` 决定，**RL 常量不会远程修改香橙派配置**。

| 内容 | 文件 / 目录 |
| --- | --- |
| 原版训练参数 | [xl330/profile.json](../microduck_rl/src/mjlab_microduck/robot/xl330/profile.json) |
| 飞特实机设定、厂家规格、待测项 | [feetech1910/profile.json](../microduck_rl/src/mjlab_microduck/robot/feetech1910/profile.json) |
| 飞特加载和校验 | [feetech1910/config.py](../microduck_rl/src/mjlab_microduck/robot/feetech1910/config.py) |
| 飞特厂家模型、控制律、许可证、来源 | [feetech1910/reference/](../microduck_rl/src/mjlab_microduck/robot/feetech1910/reference/) |
| XL330 单舵机台架几何 | [xl330_test_bench/](../microduck_rl/src/mjlab_microduck/robot/xl330_test_bench/) |

`xl330_test_bench/config.json` 是 CAD/MJCF 导出配置，不是另一份舵机训练 profile，也不参与普通行走任务。

## 当前飞特设定与临时假设

- 供电 7.4 V、位置 P=180、P 寄存器地址 21，均由用户确定。
- `operating.delay_steps = [3, 6]`：**按用户要求暂沿用 XL330 的仿真物理步延迟，非本机实测**。说明同时保存在 profile 的来源字段和 Python 注释里，不能称为厂家给出的 HD1910 延迟。
- 物理步不同于 50 Hz 策略步；毫秒数取决于具体仿真 timestep，不固定写成策略周期的倍数。
- 厂家模型固定在 7.4 V 台架参考；其默认 P=32 与用户实机 P=180 分开记录。
- 当前没有本机带载压降、P=180 响应或完整端到端延迟测量。
- 默认机械模型仍有原 XL330 的几何、质量/惯量和回差假设，切换执行器不等于全部机械适配完成。

## BAM 是什么

[Rhoban/bam](https://github.com/Rhoban/bam) 是舵机动力学辨识与仿真研究库，不是 Dynamixel 厂商 SDK。官网规格提供额定工作点，BAM 用轨迹拟合摩擦、电气等效量、惯量和延迟，补充仿真细节。

当前项目锁定的 Rhoban BAM 已内置 XL330；没有内置这份 HD1910，所以飞特目录保存厂家参考文件。采用飞特 [ftservo-bam](https://github.com/ftservo/ftservo-bam) 的 M6 参数和控制律，固定提交 `bab3967811f0bec8b79898cb03e7f5812fe581f9`、原文件 SHA-256 及 Apache-2.0 许可证。

这些是**厂家辨识先验**，不是我们的实机测量。不要把厂家参数目录命名或描述成“本机已标定”。参数分类及后续校核步骤由 [RL 飞特说明](../microduck_rl/src/mjlab_microduck/robot/feetech1910/README.md)维护。

## 训练与设备分工

- 当前训练使用 mjlab/MuJoCo Warp + PPO，需要 NVIDIA CUDA；Mac 的 Apple GPU 和香橙派不直接承担这套训练。
- Mac 可以做代码修改、CPU MuJoCo 回放、模型检查和部署。
- 训练产物 `.pt` 通过仓库导出路径生成 ONNX，包含观测归一化；配套 `manifest.json` 可由 `publish --dry-run` 在本地生成，不必上传 Hub。
- Studio 导入 `.onnx` 和 `manifest.json` 两个文件；导入不自动启用。持续行走策略在配置页绑定，单次动作显式注册后执行。
- 运行端用 15 颗舵机，当前 RL 观测 61 维、动作 14 维，嘴部不在动作输出中。
- 本机标定、限位和 IMU 坐标必须与训练定义一致。

## 摄像头、ToF 与 IMU 的区别

当前动作训练没有使用头部摄像头图像或 `sim/tof.py` 的测距作为策略输入。`sim/body_server.py` 会使用虚拟 ToF，摄像头按 `--cameras` 开启，主要用于联调完整虚拟鸭子的视频、测距、建图等系统功能。

训练中的脚下地形辅助测量不等同于实体头部 ToF；身体 IMU/关节状态则属于当前运动策略的观测。训练录像给人看，不代表策略把录像作为输入。

## 验证边界

相关配置/模型 CPU 测试和 CPU MuJoCo 回放步进已通过；没有启动完整 CUDA 训练，没有本机飞特策略上机验收。CPU 回放也没有复现训练中的全部随机指令延迟，不能称为所有时序逐项一致。
