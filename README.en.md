<p align="center">
  <img src="buildroot/share/pixmaps/logo/qingwa-logo-512.png" alt="Qingwa Pick-and-Place" height="213">
</p>

<p align="center">
  <b>🐸 Qingwa PnP Firmware · qingwa-pnp-marlin</b><br/>
  A pick-and-place firmware based on Marlin 2.1.x + the Opulo LumenPnP REV5 board
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-GPL--3.0-blue.svg" alt="GPL-3.0"></a>
  <img src="https://img.shields.io/badge/Marlin-2.1.x-important.svg" alt="Marlin 2.1.x">
  <img src="https://img.shields.io/badge/Board-LumenPnP%20REV5-success.svg" alt="LumenPnP REV5">
  <img src="https://img.shields.io/badge/MCU-STM32F407VET6-informational.svg" alt="STM32F407VET6">
  <img src="https://img.shields.io/badge/Build-PlatformIO-orange.svg" alt="PlatformIO">
</p>

<p align="center"><a href="README.md">简体中文</a> | <strong>English</strong></p>

---

## Table of Contents

- [📖 Introduction](#-introduction)
- [✨ Features](#-features)
- [🧱 Hardware & Pinout](#-hardware--pinout)
- [🏗 Repository Layout](#-repository-layout)
- [⚙️ Building & Flashing](#️-building--flashing)
- [🔧 Configuration & Versioning](#-configuration--versioning)
- [🧰 Common G-code](#-common-g-code)
- [🛠 FAQ](#-faq)
- [🤝 Credits & License](#-credits--license)

---

## 📖 Introduction

This project is the firmware for the **Qingwa (Frog) Pick-and-Place machine**. It is
*not* a 3D-printer firmware — it adapts the open-source **Marlin** firmware into the
**motion controller** of an SMT pick-and-place (PnP) machine.

It targets the **Opulo LumenPnP Motherboard REV05**, based on a **STM32F407VET6**
(ARM Cortex-M4, 512 KB Flash / 192 KB SRAM). The REV5 board is a six-axis PnP
controller that provides TMC stepper interfaces, endstops, feeder signals, RS485 and
RGB LED strips out of the box.

To behave as a pure motion controller, the build removes 3D-printing logic at the
configuration level:

| Item | This firmware |
|------|---------------|
| `EXTRUDERS` | `0` (no extruder) |
| Temperature sensors | all disabled (no heating / heated bed / fan control) |
| Axis mapping | X / Y / Y2 / Z / I / J (six steppers) |
| Driver type | TMC2209 on all axes (single-wire UART) |
| Serial | USB CDC (`SERIAL_PORT -1`) @ 250000 |

The firmware talks G-code / status over USB serial, so it can be driven by host
software such as OpenPnP for automated assembly jobs.

> This repository is customized from upstream **Marlin 2.1.x** and the
> **Opulo LumenPnP** open hardware/software ecosystem, keeping Marlin's powerful
> motion planning and HAL architecture while adding the Opulo board definitions and
> per-revision configuration snapshots.

---

## ✨ Features

- **Six independent steppers**: X / Y / Y2 / Z / I / J, each with dedicated
  STEP/DIR/EN pins and single-wire TMC UART readback (soft-serial @ 19200 baud).
  All axes use **TMC2209**.
- **Stall/DIAG detection pins**: DIAG for all six axes are routed out for
  stall/blockage detection by the host.
- **Pure motion / I/O controller**: `EXTRUDERS 0`, all temperature sensors off —
  heating-related code paths are removed so timers/interrupts stay focused on motion.
- **Complete on-board peripheral definitions**:
  - X / Y / Z limit switches
  - 4 fan / driver outputs (`FAN0`–`FAN3`) and a buzzer
  - Dual **NeoPixel** RGB strips (`PC7` / `PC8`)
  - 8-pin **AUX** port: 3×PWM + 1×analog input + I²C (SCL/SDA)
  - **RS485** TX/RX enable pins (external feeders / comm boards)
  - 4 ID pins (`PD0`–`PD3`) for hardware-revision detection
- **Parameter storage**: SRAM EEPROM emulation (8 KB), ideal when parameters are
  rewritten by the host.
- **Full G-code compatibility**: the complete Marlin G/M command set with its fast
  parser is preserved.
- **Per-revision configuration snapshots**: REV3 / REV4 plus feeder revisions
  (v3.0 / v3.1 / PWM-AUX) are kept for reference, regression and upgrades.
- **Marlin 2.1 ecosystem**: HAL layering (STM32 HAL used here), one-command
  PlatformIO builds, upstream CI workflows and Auto Build scripts.

---

## 🧱 Hardware & Pinout

### Board information

| Item | Value |
|------|-------|
| Board name (M115) | `LumenPnP Motherboard REV05` |
| Machine name | `LumenPnP` |
| MCU | STM32F407VET6 (ARM Cortex-M4 @ 168 MHz) |
| Flash / SRAM | 512 KB / 192 KB |
| Driver interface | 6 × TMC2209 (single-wire UART) |

### Key pin quick reference (from `Marlin/src/pins/stm32f4/pins_OPULO_LUMEN_REV5.h`)

| Function | Pins |
|----------|------|
| X / Y / Z stops | `PC6` / `PD15` / `PD14` |
| X / Y / Y2 / Z / I / J DIAG | `PB10` / `PB11` / `PC12` / `PB5` / `PC10` / `PC11` |
| X / Y / Y2 / Z / I / J step | `PB15` / `PE15` / `PD6` / `PE7` / `PC4` / `PE11` |
| NeoPixel ×2 | `PC7` / `PC8` |
| Buzzer | `PA7` |
| Fan / driver 0–3 | `PB8` / `PB9` / `PE5` / `PE6` |
| I²C (SDA / SCL) | `PB7` / `PB6` |
| SPI (MISO/MOSI/SCK) | `PB4` / `PB5` / `PB3` |
| RS485 TX/RX enable | `PD11` / `PD12` |
| ID pins | `PD0`–`PD3` |

### REV05 AUX port (8-pin)

```
             _________
      3.3V  |  1   2  |  GND
    PWM1 PA5|  3   4  | PC0 A1 (analog in)
    PWM2 PA6|  5   6  | SCL
    PWM3 PA15| 7   8  | SDA
             ---------
```

---

## 🏗 Repository Layout

```text
qingwa-pnp-marlin/
├── .devcontainer/              # VS Code dev container (build env in one click)
├── .github/                    # GitHub CI / community config (test-builds etc.)
├── .trellis/                   # AI-assisted development workflow scaffold (optional)
├── Marlin/                     # ⭐ Firmware source & configuration (core)
│   ├── Configuration.h         #   Active configuration (REV5)
│   ├── Configuration_adv.h     #   Advanced configuration
│   ├── config.ini              #   Compile-time config via ini (disabled by default)
│   ├── Makefile                #   Make entry for pio/arduino
│   ├── Version.h               #   Version strings (overridable at build time)
│   ├── rev3-feeder-config/     #   Legacy: REV3 feeder config snapshot
│   ├── rev4-v3.0-feeder-config/#   Legacy: REV4 + feeder v3.0 snapshot
│   ├── rev4-v3.1-feeder-config/#   Legacy: REV4 + feeder v3.1 snapshot
│   ├── rev4-v3.1-feeder-pwm-aux-config/  # REV4 v3.1 + PWM/AUX snapshot
│   ├── rev5-config/            #   REV5 snapshot matching the active config
│   └── src/                    #   Marlin core source
│       ├── MarlinCore.cpp|.h   #     Firmware entry point
│       ├── HAL/                #     Hardware Abstraction Layer (STM32 etc.)
│       ├── core/               #     boards.h board IDs / macros
│       ├── pins/               #     Per-board pin definitions
│       │   └── stm32f4/pins_OPULO_LUMEN_REV5.h   # ← REV5 pins
│       ├── module/             #     planner / stepper / endstop …
│       ├── gcode/              #     G/M-code parsing & execution
│       ├── feature/ libs/ lcd/ sd/ inc/ tests/
├── buildroot/                  # PlatformIO board support / scripts / tests
│   ├── share/PlatformIO/
│   │   ├── boards/marlin_opulo_lumen_rev5.json   # REV5 board JSON
│   │   └── scripts/            #   configuration.py build scripts
│   ├── share/vscode/auto_build.py                #   Auto Build (PIO Build)
│   └── tests/                  #   make tests-all-local etc.
├── config/  docker/  docs/     # Docs / Docker / dev docs (Bresenham, Serial…)
├── ini/                        # PlatformIO env matrix (grouped by MCU)
│   └── stm32f4.ini             #   contains [env:Opulo_Lumen_REV5]
├── .editorconfig  .gitattributes
├── platformio.ini              # ⭐ PlatformIO entry (default_envs = Opulo_Lumen_REV5)
├── Makefile                    # make shortcuts
├── marlin.code-workspace       # VS Code multi-root workspace
├── process-palette.json        # VS Code "Process Palette" → PIO Build
├── LICENSE                     # GPL-3.0
└── README.md / README.en.md    # Multilingual docs (Chinese by default)
```

---

## ⚙️ Building & Flashing

### Prerequisites

- Install [PlatformIO Core](https://docs.platformio.org/en/latest/core/installation.html);
  the easiest path is **VS Code + PlatformIO IDE** or Marlin's **Auto Build**
  script (`buildroot/share/vscode/auto_build.py`, or the **PIO Build** command from
  `process-palette.json`).
- The default build environment is already set to the REV5 board — no extra setup.

### Command line

```bash
# Build (default env is Opulo_Lumen_REV5)
pio run

# Explicit environment
pio run -e Opulo_Lumen_REV5

# Flash
pio run -e Opulo_Lumen_REV5 -t upload

# Serial monitor (250000 baud)
pio device monitor -b 250000
```

> After editing `Configuration.h` etc., just run `pio run` again — PlatformIO
> rebuilds automatically.

### Run upstream tests (optional)

```bash
make tests-all-local                  # all tests locally
make tests-single-local TEST_TARGET=Opulo_Lumen_REV5
```

---

## 🔧 Configuration & Versioning

- **`Marlin/Configuration.h` / `Configuration_adv.h`**: the configuration actually
  used by this repo (for the REV5 board).
- **`Marlin/rev5-config/`**: a snapshot matching the active config, kept for
  reference/archival.
- **`Marlin/rev3-*` / `rev4-*` folders**: config snapshots for older boards and each
  **feeder** hardware revision — pull them out when upgrading/downgrading hardware.
- **`Marlin/config.ini`**: Marlin compile-time config via ini
  (`[config:minimal/basic/advanced]`). Disabled by default
  (`ini_use_config = none`); enable it only if you prefer ini-driven config.

### Firmware baseline

- Source baseline: **Marlin 2.1.x** (config format `CONFIGURATION_H_VERSION 02010300`),
  preserving the upstream `import-2.1.x` / `bugfix-2.1.x` layout.

---

## 🧰 Common G-code

The firmware keeps Marlin's full G/M command set. Typical PnP/debug examples:

```gcode
M906 X1000 Y1000        ; set X/Y driver current (mA)
M204 T4000              ; trapezoid acceleration
G0 F60000               ; fast move example (60000 mm/min)

; —— Drive the ID pins (PD0-PD3) as generic I/O ——
M42 P15 T2              ; PD0 -> input + pull-up
M43 P43                 ; read PD0 level
M42 P31 S255            ; AUX pin high (PD13 example)
M42 P31 S0              ; AUX pin low

; TMC driver diagnostics
M122                    ; TMC status for all axes
```

See the [Marlin G-code docs](https://marlinfw.org/docs/gcode/M000-M001.html) for the
full command reference.

---

## 🛠 FAQ

<details>
<summary><b>No serial port after flashing?</b></summary>

Use a data USB cable to the STM32's USB port; you may need the USB CDC driver on
first use (Windows). A new serial device should appear in Device Manager /
`ls /dev/tty*`. Baud rate is `250000`.
</details>

<details>
<summary><b>How do I check the running firmware version/config?</b></summary>

Send `M115` from your host — the response includes the version string and the
machine name (LumenPnP).
</details>

<details>
<summary><b>Why is there no heating/extrusion?</b></summary>

A PnP machine doesn't need extruder/heating. This firmware sets `EXTRUDERS 0` and
disables all temperature sensors in `Configuration.h` — that's by design, not a fault.
</details>

---

## 🤝 Credits & License

- Firmware built on **Marlin**: <https://github.com/MarlinFirmware/Marlin>
  (by Scott Lahteine and the Marlin community)
- Board support & board JSON from the **Opulo LumenPnP** open hardware/software
  ecosystem: <https://github.com/opulo-inc/>
- All code is released under the **GPL-3.0** license — see [LICENSE](LICENSE).
  Per the GPL, redistribution of builds based on this firmware must also be open
  source and provide the corresponding source.
- This firmware is maintained for the **Qingwa (Frog) PnP** project. Feedback is
  welcome via Issues in this repository.
