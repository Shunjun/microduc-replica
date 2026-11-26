# 设备部署记录

## 仓库与程序

- 当前发布源为 `Shunjun/microduck`，不是原上游仓库。
- `microduck` 包含设备运行程序和 Studio；`microduck_rl` 在训练机使用，不必整体下载到香橙派。
- Studio 是设备上的 Rust 网页服务，默认端口 8080；浏览器打开即可，串口仍由 robotd 操作。
- 发布包是 `.tar.zst`，首次安装脚本会引导 updaterd 下载、验签和安装，不需要用户随意手动解压到系统目录。
- 签名信任依赖设备公钥与发布签名匹配。私钥只用于签名，设备不需要私钥。

机制参考：[安装脚本](../microduck/scripts/install.sh)、[更新器设计](../microduck/docs/design/updater-design.md)。

## 最近一次成功安装

用户贴出的日志：`installed daemon 0.23.2`。

安装时设置了 `DUCK_NO_START=1`。安装器结束时服务未启用/未运行；它同时提示发布包自己的 postinstall 曾短暂启动过服务后又被停止，所以不能把这一轮理解成“整次安装从未运行任何程序”。随后用户显式启用了 robotd 和 Studio，并最终进入装机调试模式，见[标定记录](calibration.md)。

可重复使用的命令，需先确保脚本下载完整、代理按实际情况设置：

```bash
curl -fL \
  https://raw.githubusercontent.com/Shunjun/microduck/main/scripts/install.sh \
  -o /tmp/microduck-install.sh

sudo env http_proxy="$http_proxy" https_proxy="$https_proxy" \
  DUCK_REPO=Shunjun/microduck DUCK_NO_START=1 \
  sh /tmp/microduck-install.sh
```

源码通过 SSH 推送的偏好与安装器使用 HTTPS 下载发布资产是两件事。

## 配置与持久数据

| 路径 | 作用 |
| --- | --- |
| `/etc/robot/robotd.toml` | 安装器首次创建的基础运行配置；已有则保留 |
| `/var/lib/robot/robotd.toml` | robotd 用户配置层，按键覆盖基础配置；网页设置/CLI 修改可能写在这里 |
| `/etc/robot/feetech1910.json` | 本机飞特标定，不是安装器自动生成的实测值 |
| `/etc/robot/updater.toml` | 更新器来源、信任和应用策略 |
| `/etc/robot/trusted_keys/` | 发布验签公钥 |
| `/opt/robot/daemon/current/` | 当前激活的运行版本 |
| `/var/lib/microduck-studio/actions/` | Studio 导入的模型和 manifest，除非显式改 library-dir |

基础 TOML 与飞特标定 JSON 是不同文件。后者缺失时正常 robotd 会在打开串口前退出，不能靠空 JSON 通过校验。

“没有 token”提示不影响公开仓库这次已完成的下载；私有仓库的后续自动更新需要相应访问凭据。`DUCK_NO_START` 期间 updaterd 未运行，不能据安装器通用提示认定本机此刻正在自动更新。

## ONNX Runtime 的安装故障与解决记录

原上游安装逻辑使用 GitHub：

```text
https://github.com/microsoft/onnxruntime/releases/download/v1.28.0/onnxruntime-linux-aarch64-1.28.0.tgz
```

先遭遇 hook 下载 90 秒超时；随后 GitHub 仓库/API/下载资源均出现 404。这个问题不是 RL 模型未上传。此处下载的是 C/C++ 动态运行库，而非 `.onnx` 动作模型。

临时改从 PyPI 的 `onnxruntime==1.23.2` ARM64 wheel 提取动态库。已经实际下载并校验过的 wheel：

```text
onnxruntime-1.23.2-cp311-cp311-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl
SHA-256: bbfd2fca76c855317568c1b36a885ddea2272c13cb0e395002c402f2360429a6
```

来源是 `https://pypi.org/pypi/onnxruntime/1.23.2/json` 返回的 `files.pythonhosted.org` 资源；从 ZIP 内取 `onnxruntime/capi/libonnxruntime.so.1.23.2`，不是把 Python 包导入运行。

用户确认安装后的文件：

```text
/usr/local/lib/libonnxruntime.so       -> libonnxruntime.so.1.23.2
/usr/local/lib/libonnxruntime.so.1     -> libonnxruntime.so.1.23.2
/usr/local/lib/libonnxruntime.so.1.23.2  18693384 bytes
```

执行过 `sudo ldconfig` 后重试安装成功。运行端这次最低要求为 1.23；RL 训练环境锁定的 Python ONNX Runtime 版本是另一套依赖，不用强行与板端动态库文件号相同。

截至本记录，根仓库没有修复原安装 hook 的下载源；新装设备仍可能遇到相同问题。只执行 `pip install onnxruntime` 并不能保证原 hook 在 `/usr/local/lib` 找到库。

## 日常检查

在香橙派上执行：

```bash
systemctl status robotd studio --no-pager -l
sudo journalctl -u robotd -u studio -b -n 100 --no-pager
```

本地新提交不代表已发布或已更新到设备；设备版本必须通过设备查询确认。

安装器曾把用户加入 `robot` 组。新 SSH 登录会取得新组权限；原会话可使用 `newgrp robot`。服务尚未运行时，仅取得组权限也不会使 IPC 自动出现。
