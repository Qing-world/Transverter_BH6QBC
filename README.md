# ESP32 + ADF4351 智能全频段微波变频器控制器 (Smart RF Transverter Controller)

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Platform: ESP32](https://img.shields.io/badge/Platform-ESP32-green.svg)](https://www.espressif.com/)
[![PLL: ADF4351](https://img.shields.io/badge/PLL-ADF4351-orange.svg)](https://www.analog.com/)
[![Support: QO--100](https://img.shields.io/badge/Satellite-QO--100-purple.svg)](https://amsat-dl.org/en/qo-100-es-hail-2/)

基于 ESP32 与 ADF4351 锁相环（PLL）构建的软件定义微波变频器（Transverter）控制系统。专为业余无线电爱好者（HAM Radio）、微波通联（1.2G/2.4G/5.7G/10G）及 QO-100 卫星通联设计，集成了全物理光电隔离、双向射频时钟调度、Web 响应式控制台与多重硬件保护机制。

---

## 📸 系统特性 (Key Features)

* **全响应式 Web 仪表盘**：免安装 App，手机/电脑连接 ESP32 热点直接访问 `192.168.4.1`，支持实时 IF/LO/RF 矩阵动态算力看板。
* **全物理双光电隔离**：采用双 PC817 光耦独立隔离电台 13.8V PTT 信号与单片机/继电器回路，杜绝感应高压击穿与反峰电涌。
* **分级安全保护机制**：
  * **低侧 NMOS 软启供电**：配合双路继电器实现功放（PA）无拉弧纯净断电与接收（LNA）毫秒级硬保护。
  * **TOT 发射超时保护**：软件设定发射时限（默认 180s），超时自动闭锁，防止功放干烧。
  * **PTT 软件安全锁**：调试测试期间一键切断发射响应。
* **多中频与目标波段智能计算**：
  * 支持 28MHz / 144MHz / 430MHz 中频电台无缝接入。
  * 覆盖 50M / 144M / 430M / 1.2G / 2.4G 业余频段，支持同频/异频中继差频（Shift）自动换算。
  * 内置 QO-100 卫星 CW/SSB/FT8/DATV 快捷信道预设。
* **高精度时钟与功率控制**：
  * 适配 100MHz TCXO 板载晶振及外部 GPSDO（10MHz）基准。
  * 支持底层 ±Hz 级晶振频偏微调校准（Calibration）。
  * 软件寄存器四挡控制 ADF4351 输出功率（-4dBm / -1dBm / +2dBm / +5dBm），完美激励 ADE-1 等双平衡混频器。
* **Web OTA 无线固件升级**：变频器封装入屏蔽铝箱后，可直接在网页端上传 `.bin` 固件进行无线更新。

---

## 🛠️ 硬件架构与拓扑 (System Topology)

```text
               ┌─────────────────────── +13.8V 主供电 ───────────────────────┐
               │                                                            │
         [电台 ACC PTT]                                                [DCDC 5V 降压]
               │                                                            │
    ┌──────────┴──────────┐                                      ┌──────────┴──────────┐
    ▼                     ▼                                      ▼                     ▼
[PC817 ①]             [PC817 ②]                              [ESP32 系统]          [继电器/MOS]
 (3.3V 逻辑)          (5V 功率驱动)                                │                     │
    │                     │                                      │ (SPI)               │ (RX/TX 切换)
    ▼                     ▼                                      ▼                     ▼
[GPIO 32]             [继电器 X1/X2] ──────────────────────► [ADF4351] ──────► [射频开关/PA/LNA]
