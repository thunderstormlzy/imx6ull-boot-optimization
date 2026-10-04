# i.MX6ULL Linux 启动性能优化

## 项目概述

本项目针对 i.MX6ULL 开发板的网络启动流程进行性能分析和裁剪，启动路径为：

```text
上电 -> U-Boot -> TFTP 下载 zImage 和设备树
     -> Linux 内核初始化 -> PHY 建链和静态 IP 配置
     -> NFS 挂载根文件系统 -> BusyBox 用户空间
     -> Linux 控制台就绪
```

优化目标是在不影响 LCD、USB Host、以太网和 NFS 根文件系统的前提下，减少 Linux 内核阶段的启动时间，并通过冷启动数据验证结果。

## 硬件和软件环境

| 项目 | 配置 |
| --- | --- |
| 开发板 | i.MX6ULL EMMC LZY / 正点原子 ALPHA 底板 |
| CPU | i.MX6ULL，396 MHz |
| Linux 内核 | 4.1.15 |
| 初始化系统 | BusyBox init |
| 根文件系统 | NFS，NFSv3，TCP |
| 网络配置 | 静态 IP |
| 启动加载 | U-Boot TFTP |
| 串口 | 115200 baud |
| 测试方式 | 完全断电后冷启动 |

## 测试方法

每个版本均记录30次完整冷启动。完整启动必须同时包含以下标记：

```text
BOOTMARK:KERNEL_START
VFS: Mounted root (nfs filesystem)
BOOTMARK:CONSOLE_READY
```

时间戳来自串口日志。`BOOTMARK:KERNEL_START` 作为 Linux 内核阶段的统一起点，`BOOTMARK:CONSOLE_READY` 作为用户空间可用的结束点。Linux 阶段的结果以这两个标记之间的串口时间差为准；TFTP 阶段单独记录，不把网络传输波动误判为内核优化收益。

测试数据目录：

```text
logs/baseline/001.log ... 030.log
logs/optimize/001.log ... 030.log
```

两组数据均为30/30次成功启动。

## 优化前基线

以下数据由 `logs/baseline/001.log` 至 `030.log` 逐份提取，单位为秒。

| 指标 | 平均值 | 中位数 | 最小值 | 最大值 |
| --- | ---: | ---: | ---: | ---: |
| U-Boot 出现 -> TFTP 开始 | 3.574 | 3.518 | 3.508 | 5.176 |
| 内核标记 -> NFS 挂载 | 8.620 | 8.850 | 7.855 | 12.573 |
| NFS 挂载 -> 控制台就绪 | 0.574 | 0.567 | 0.540 | 0.783 |
| 内核标记 -> 控制台就绪 | 9.194 | 9.409 | 8.401 | 13.140 |

## 已完成的优化

### 1. U-Boot 自动启动等待

将正式优化测试中的 `bootdelay` 从3秒调整为1秒，减少人工打断等待，同时保留进入U-Boot命令行的机会。

优化后的正常 TFTP 启动流程不执行 `mfgtool_args`。其中保留的 `g_mass_storage` 参数仅用于制造工具启动路径。

### 2. RTL8201F 专用 PHY 驱动和硬件中断

板卡上的 PHY ID 为 `0x001cc816`，原内核将其识别为 Generic PHY（`irq=-1`）。为 Linux 4.1.15 回移 RTL8201F 驱动支持，并在设备树中配置 `GPIO5_IO06` 的低电平 PHY 中断。30 次优化日志均识别为专用驱动并分配了硬件 IRQ，例如：

```text
fec 20b4000.ethernet eth0: Freescale FEC PHY driver [RTL8201F Fast Ethernet] ... irq=169
```

### 3. 关闭未使用的第二个 FEC

NFS启动只使用 `20b4000.ethernet` 对应的 eth0，关闭未使用的 FEC1/eth1，减少无关MAC和PHY初始化。

### 4. 关闭无效设备探测

禁用未连接的 QSPI 节点，消除无效 JEDEC 探测；清理错误的 gpio-keys 和空 pinctrl 配置，保持设备树与实际硬件一致。

### 5. 裁剪音频子系统

关闭未使用的 WM8960、ES8388、SAI2、ASRC、ALSA 和音频电源节点，并移除用户空间的 `alsactl restore`。优化后不再出现音频探测失败和混音恢复日志。

### 6. 关闭 USB Mass Storage Gadget

关闭启动时未使用的 USB Mass Storage Gadget 实例化路径，消除无后端存储设备导致的 `g_mass_storage ... -22` 错误，同时保留 USB Host、`CONFIG_USB_STORAGE=y`、USB Hub 和 HID 支持。具体配置以仓库中提交的优化版 `.config` 为准。

## 尝试但失败的优化
### 1. 取消FEC的复位功能
尝试取消活动网口的复位配置，使FEC设备更早的进行注册，但使得链路状态错过轮询，导致链路检测时间延长了1秒。

## 优化后 30 次结果

以下数据由 `logs/optimize/001.log` 至 `030.log` 逐份提取，单位为秒。

| 指标 | 平均值 | 中位数 | 最小值 | 最大值 |
| --- | ---: | ---: | ---: | ---: |
| U-Boot 出现 -> TFTP 开始 | 1.652 | 1.517 | 1.503 | 2.894 |
| 内核标记 -> NFS 挂载 | 7.917 | 7.553 | 6.784 | 11.254 |
| NFS 挂载 -> 控制台就绪 | 0.563 | 0.565 | 0.509 | 0.618 |
| 内核标记 -> 控制台就绪 | 8.480 | 8.111 | 7.399 | 11.839 |

## 效果对比

| 指标 | 基线平均 | 优化后平均 | 变化 |
| --- | ---: | ---: | ---: |
| U-Boot 出现 -> TFTP 开始 | 3.574 s | 1.652 s | 减少 1.922 s |
| 内核标记 -> NFS 挂载 | 8.620 s | 7.917 s | 减少 0.703 s |
| NFS 挂载 -> 控制台就绪 | 0.574 s | 0.563 s | 减少 0.011 s |
| 内核标记 -> 控制台就绪 | 9.194 s | 8.480 s | 减少 0.714 s，约7.77% |

内核镜像大小从 `5624992` 字节减少到 `5386080` 字节：

```text
减少 238912 字节，约 233.3 KiB，缩小 4.25%
```

PHY 绑定到链路建立的平均时间从 4.264 秒降至 3.583 秒；优化日志中的 PHY 驱动均为 RTL8201F 专用驱动并使用硬件 IRQ，剩余时间主要由 PHY 复位和 100 Mbps 自动协商决定。

## 结果解读和限制

TFTP 下载速度在不同启动之间存在明显波动，基线和优化数据的内核镜像传输时间不能直接用于证明内核裁剪收益。最终结论使用 `BOOTMARK:KERNEL_START` 之后的 Linux 内核阶段指标，TFTP 只作为独立观测项。

U-Boot 等待阶段从3.574秒降至1.652秒，减少1.922秒；Linux 内核标记到控制台就绪从9.194秒降至8.480秒，减少0.714秒，约7.77%。两项指标分别统计，不将 U-Boot 等待时间计入 Linux 内核优化收益。优化数据中的第11、14、27次出现较长的 PHY 建链延迟，统计时予以保留。

项目最终保留的功能包括：

- RTL8201F 以太网和PHY硬件中断；
- 静态IP和NFS根文件系统；
- LCD显示链路；
- USB Host、USB Hub、U盘存储和HID；
- BusyBox用户空间和串口控制台。

## 日志证据

基线日志中每次均能观察到以下无效或未使用路径：

- `fsl-quadspi ... unrecognized JEDEC id bytes: ff, ff, ff`；
- `g_mass_storage ... failed to start ... -22`；
- `eth1: registered`；
- `Freescale FEC PHY driver [Generic PHY] ... irq=-1`；
- WM8960/ASRC 音频初始化和用户空间 `ALSA: Restoring mixer setting......`。

优化日志中 30/30 次均满足：

- 无 QSPI、`g_mass_storage`、eth1、音频和 gpio-keys 探测错误；
- 使用 `RTL8201F Fast Ethernet` 专用 PHY 驱动；
- 出现 `VFS: Mounted root (nfs filesystem)`；
- 出现 `BOOTMARK:CONSOLE_READY`。

逐次统计文件：

```text
results/baseline-summary.csv
results/optimize-summary.csv
```

启动测试条件：

- Linux 4.1.15；
- GCC 4.9.4 Linaro 工具链；
- U-Boot TFTP 下载 zImage 和设备树；
- NFSv3/TCP 挂载根文件系统；
- 115200 baud 串口；
- 完全断电后冷启动；
- 每个版本测试30次；
- 每次必须同时出现 `BOOTMARK:KERNEL_START`、`VFS: Mounted root (nfs filesystem)` 和 `BOOTMARK:CONSOLE_READY`。
