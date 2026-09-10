<p align="center">
  <img src="buildroot/share/pixmaps/logo/qingwa-logo-256.png" alt="青蛙贴片机 Qingwa Pick-and-Place" width="320">
</p>

<p align="center">
  <b>🐸 青蛙贴片机 固件 · qingwa-pnp-marlin</b><br/>
  基于 Marlin 2.1.x + Opulo LumenPnP REV5 主板的贴片机专用固件
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-GPL--3.0-blue.svg" alt="GPL-3.0"></a>
  <img src="https://img.shields.io/badge/Marlin-2.1.x-important.svg" alt="Marlin 2.1.x">
  <img src="https://img.shields.io/badge/Board-LumenPnP%20REV5-success.svg" alt="LumenPnP REV5">
  <img src="https://img.shields.io/badge/MCU-STM32F407VET6-informational.svg" alt="STM32F407VET6">
  <img src="https://img.shields.io/badge/Build-PlatformIO-orange.svg" alt="PlatformIO">
</p>

<p align="center"><strong>简体中文</strong> | <a href="README.en.md">English</a></p>

---

## 目录

- [📖 项目简介](#-项目简介)
- [✨ 功能特性](#-功能特性)
- [🧱 硬件与引脚](#-硬件与引脚)
- [🏗 目录结构](#-目录结构)
- [⚙️ 构建与烧录](#️-构建与烧录)
- [🔧 配置与版本管理](#-配置与版本管理)
- [🧰 常用 G 代码](#-常用-g-代码)
- [🛠 常见问题](#-常见问题)
- [🤝 致谢与许可证](#-致谢与许可证)

---

## 📖 项目简介

本项目是 **「青蛙贴片机」专用固件**。它并非 3D 打印机固件，而是把一个通用开源固件
**Marlin** 改造为 **SMT 贴片机（Pick-and-Place）的运动控制器固件**。

固件面向 **Opulo LumenPnP Motherboard REV05**（MCU：STMicroelectronics **STM32F407VET6**，
ARM Cortex-M4，512 KB Flash / 192 KB SRAM）编写。REV5 主板是一块六轴贴片机控制板，
自带 TMC 步进驱动接口、限位、送料器信号、RS485 与 RGB 灯带等贴片机需要的板载资源。

为达到"纯运动控制"目标，本固件在配置层面移除了 3D 打印相关逻辑：

| 项目 | 本固件 |
|------|--------|
| `EXTRUDERS` | `0`（无挤出机） |
| 温度传感器 | 全部关闭（无加热/热床/风扇温控） |
| 轴定义 | X / Y / Y2 / Z / I / J 六路步进 |
| 驱动类型 | 全部 TMC2209（单线 UART） |
| 串口 | USB CDC（`SERIAL_PORT -1`）@ 250000 |

固件通过 USB 串口接收 G 代码 / 状态查询，可与 OpenPnP 等上位机软件配合完成
自动贴装流程。

> 该仓库从上游 **Marlin 2.1.x** 与 **Opulo LumenPnP** 开源硬件/软件生态定制而来，
> 保留了 Marlin 强大的运动规划与 HAL 分层结构，并加入了 Opulo 主板定义与多版本配置。

---

## ✨ 功能特性

- **六轴独立步进**：X / Y / Y2 / Z / I / J 各轴独立 STEP/DIR/EN 引脚与单线 TMC UART
  回读（软串口 19200 baud），全轴 **TMC2209**。
- **失速 / 堵转检测信号（DIAG）**：六轴 DIAG 引脚直接引出，可接入上位机做堵转判断。
- **纯运动 / IO 控制器**：`EXTRUDERS 0`、全部温度传感器关闭，3D 打印热管理路径被裁掉，
  中断与定时器资源优先保证运动控制。
- **完善的板载外设定义**：
  - X / Y / Z 三路限位开关
  - 4 路风扇 / 驱动输出（`FAN0`–`FAN3`）、蜂鸣器
  - 双路 **NeoPixel** RGB 灯带（`PC7` / `PC8`）
  - 8 脚 **AUX** 扩展口：3×PWM + 1×模拟输入 + I²C（SCL/SDA）
  - **RS485** 收发使能脚（用于外部送料器 / 通讯板）
  - 4 个 ID 引脚（`PD0`–`PD3`），支持硬件版本识别
- **参数存储**：SRAM EEPROM 仿真（8 KB），配合上位机使用不掉电参数重写。
- **G 代码兼容**：完整保留 Marlin 的 G/M 命令体系与快速解析器。
- **多硬件版本配置快照**：REV3 / REV4 + 各代送料器（v3.0 / v3.1 / PWM-AUX）配置保留，
  便于对照、回归与硬件升级。
- **继承 Marlin 2.1 生态**：HAL 分层（本仓库重点使用 STM32 HAL）、PlatformIO 一键构建、
  官方 CI 工作流、Auto Build 脚本。

---

## 🧱 硬件与引脚

### 主板信息

| 项 | 值 |
|----|----|
| 固件上报主板名 | `LumenPnP Motherboard REV05` |
| 固件上报机器名 | `LumenPnP` |
| MCU | STM32F407VET6（ARM Cortex-M4 @ 168 MHz） |
| Flash / SRAM | 512 KB / 192 KB |
| 驱动接口 | 6 × TMC2209（单线 UART 回读） |

### 关键引脚速查（来自 `Marlin/src/pins/stm32f4/pins_OPULO_LUMEN_REV5.h`）

| 功能 | 引脚 |
|------|------|
| X / Y / Z 限位 | `PC6` / `PD15` / `PD14` |
| X / Y / Y2 / Z / I / J DIAG | `PB10` / `PB11` / `PC12` / `PB5` / `PC10` / `PC11` |
| X / Y / Y2 / Z / I / J 步进 | `PB15` / `PE15` / `PD6` / `PE7` / `PC4` / `PE11` |
| NeoPixel ×2 | `PC7` / `PC8` |
| 蜂鸣器 | `PA7` |
| 风扇 / 驱动 0–3 | `PB8` / `PB9` / `PE5` / `PE6` |
| I²C（SDA / SCL） | `PB7` / `PB6` |
| SPI（MISO/MOSI/SCK） | `PB4` / `PB5` / `PB3` |
| RS485 收发使能 | `PD11` / `PD12` |
| ID 引脚 | `PD0`–`PD3` |

### REV05 AUX 扩展口（8 脚）

```
             _________
      3.3V  |  1   2  |  GND
    PWM1 PA5|  3   4  | PC0 A1 (模拟输入)
    PWM2 PA6|  5   6  | SCL
    PWM3 PA15| 7   8  | SDA
             ---------
```

---

## 🏗 目录结构

```text
qingwa-pnp-marlin/
├── .devcontainer/              # VS Code 开发容器（构建环境一键起）
├── .github/                    # GitHub CI / 社区配置（test-builds 等工作流）
├── .trellis/                   # AI 辅助开发工作流脚手架（可选）
├── Marlin/                     # ⭐ 固件源码与配置（核心）
│   ├── Configuration.h         #   当前激活配置（REV5）
│   ├── Configuration_adv.h     #   进阶配置
│   ├── config.ini              #   Marlin 编译期配置（ini 语法，默认未启用）
│   ├── Makefile                #   供 pio/arduino 的 make 入口
│   ├── Version.h               #   版本字符串（构建期覆盖用）
│   ├── rev3-feeder-config/     #   历史：REV3 送料器配置快照
│   ├── rev4-v3.0-feeder-config/#   历史：REV4 + 送料器 v3.0 配置快照
│   ├── rev4-v3.1-feeder-config/#   历史：REV4 + 送料器 v3.1 配置快照
│   ├── rev4-v3.1-feeder-pwm-aux-config/  # REV4 v3.1 + PWM/AUX 配置快照
│   ├── rev5-config/            #   与当前 Configuration.* 一致的 REV5 快照
│   └── src/                    #   Marlin 核心源码
│       ├── MarlinCore.cpp|.h   #     固件入口
│       ├── HAL/                #     硬件抽象层（STM32 等）
│       ├── core/               #     boards.h 主板编号 / 宏
│       ├── pins/               #     各板引脚定义
│       │   └── stm32f4/pins_OPULO_LUMEN_REV5.h   # ← REV5 引脚
│       ├── module/             #     planner / stepper / endstop …
│       ├── gcode/              #     G/M 代码解析与执行
│       ├── feature/ libs/ lcd/ sd/ inc/ tests/
├── buildroot/                  # PlatformIO 板级支持 / 脚本 / 测试
│   ├── share/PlatformIO/
│   │   ├── boards/marlin_opulo_lumen_rev5.json   # REV5 板级 JSON
│   │   └── scripts/            #   configuration.py 等构建脚本
│   ├── share/vscode/auto_build.py                #   Auto Build（PIO Build）
│   └── tests/                  #   make tests-all-local 等自动化测试
├── config/  docker/  docs/     # 文档 / Docker / dev 文档（Bresenham、Serial…）
├── ini/                        # PlatformIO 环境矩阵（按芯片分类）
│   └── stm32f4.ini             #   内含 [env:Opulo_Lumen_REV5]
├── .editorconfig  .gitattributes
├── platformio.ini              # ⭐ PlatformIO 工程入口（default_envs = Opulo_Lumen_REV5）
├── Makefile                    # make 快捷入口
├── marlin.code-workspace       # VS Code 多根工作区
├── process-palette.json        # VS Code「Process Palette」→ PIO Build 命令定义
├── LICENSE                     # GPL-3.0
└── README.md / README.en.md    # 多语言说明（中文默认）
```

---

## ⚙️ 构建与烧录

### 环境准备

- 安装 [PlatformIO Core](https://docs.platformio.org/en/latest/core/installation.html)，
  推荐用 **VS Code + PlatformIO IDE** 或 Marlin 的 **Auto Build** 插件/脚本
  （`buildroot/share/vscode/auto_build.py`，或 `process-palette.json` 定义的
  「PIO Build」命令）。
- 默认构建目标已经设为 REV5 主板，无需额外选择。

### 命令行

```bash
# 编译（默认 env 即 Opulo_Lumen_REV5）
pio run

# 明确指定目标
pio run -e Opulo_Lumen_REV5

# 烧录
pio run -e Opulo_Lumen_REV5 -t upload

# 串口监视（250000 baud）
pio device monitor -b 250000
```

> 若修改了 `Configuration.h` 等配置，直接重新 `pio run` 即可；PlatformIO 会自动重建。

### 跑官方测试（可选）

```bash
make tests-all-local            # 本机跑全部测试
make tests-single-local TEST_TARGET=Opulo_Lumen_REV5
```

---

## 🔧 配置与版本管理

- **`Marlin/Configuration.h` / `Configuration_adv.h`**：当前仓库实际使用的激活配置
  （对应 REV5 主板）。
- **`Marlin/rev5-config/`**：与该激活配置一致的 REV5 快照，用于对照/归档。
- **`Marlin/rev3-*` / `rev4-*` 目录**：历史主板与各代**送料器（Feeder）**硬件版本
  的配置快照，升级/回退硬件时可按需取出使用。
- **`Marlin/config.ini`**：Marlin 的 ini 编译期配置入口
  （`[config:minimal/basic/advanced]`）。默认未启用（`ini_use_config = none`）；
  如需用 ini 方式维护配置可开启并按需组合。

### 关于本仓库固件版本基线

- 源码基线：**Marlin 2.1.x**（配置文件版本 `CONFIGURATION_H_VERSION 02010300`），
  保留上游 `import-2.1.x`/`bugfix-2.1.x` 演进结构。

---

## 🧰 常用 G 代码

固件保留 Marlin 的完整 G/M 命令。贴片机调试常用示例：

```gcode
M906 X1000 Y1000        ; 设置 X/Y 电流（mA）
M204 T4000              ; 梯形加速度
G0 F60000               ; 快速移动示例（速度 60000 mm/min）

; —— ID 引脚（PD0-PD3）以“通用 IO”方式读写示例 ——
M42 P15 T2              ; 设置 PD0 为输入+上拉
M43 P43                 ; 读取 PD0 电平
M42 P31 S255            ; AUX 引脚输出高（PD13 示例）
M42 P31 S0              ; AUX 引脚输出低

; TMC 驱动状态诊断
M122                    ; 查看各轴 TMC 状态
```

完整命令表见 [Marlin G-code 文档](https://marlinfw.org/docs/gcode/M000-M001.html)。

---

## 🛠 常见问题

<details>
<summary><b>烧录后没有串口？</b></summary>

确认使用 USB 数据线连接 STM32 的 USB 口；首次可能需要安装驱动
（Windows 下 STM32 USB CDC 驱动）。设备管理器/`ls /dev/tty*` 应能出现新串口，
波特率使用 `250000`。
</details>

<details>
<summary><b>如何确认刷入的固件版本/配置？</b></summary>

上位机发送 `M115`，会回传固件信息（包含版本字符串与机器名 LumenPnP）。
</details>

<details>
<summary><b>为什么没有温控/挤出？</b></summary>

贴片机不需要 3D 打印的挤出与加热功能。本固件在 `Configuration.h` 里
`EXTRUDERS 0` 且所有温度传感器关闭，属预期行为，而非故障。
</details>

---

## 🤝 致谢与许可证

- 固件本体基于 **Marlin** 开发：<https://github.com/MarlinFirmware/Marlin>
  （作者：Scott Lahteine 及 Marlin 社区）
- 主板支持与板级 JSON 来自 **Opulo LumenPnP** 开源硬件/软件生态：
  <https://github.com/opulo-inc/>
- 全部代码以 **GPL-3.0** 许可发布，见 [LICENSE](LICENSE)。按 GPL 要求，
  基于本固件的再分发需同样开放源码并提供兼容源码。
- 本固件供 **青蛙贴片机** 项目使用与学习交流；如有问题欢迎在
  **Bilibili @青蛙贴片机** 评论区交流，或提交 Issue。
