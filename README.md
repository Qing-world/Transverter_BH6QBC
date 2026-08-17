# ESP32 + ADF4351 智能全频段微波变频器控制系统
### Smart Multi-IF RF Transverter Controller

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Platform: ESP32](https://img.shields.io/badge/Platform-ESP32-green.svg)](https://www.espressif.com/)
[![PLL: ADF4351](https://img.shields.io/badge/PLL-ADF4351-orange.svg)](https://www.analog.com/)
[![Band: Multi--Band](https://img.shields.io/badge/Band-50M%20--%202.4GHz-red.svg)]()
[![Satellite: QO--100](https://img.shields.io/badge/Satellite-QO--100%20Ready-purple.svg)](https://amsat-dl.org/en/qo-100-es-hail-2/)

基于 ESP32 与 ADF4351 宽带高频锁相环（PLL）构建的软件定义微波变频器（Transverter）智能控制系统。专为业余无线电爱好者（HAM Radio）、微波通联（50MHz ~ 2.4GHz）及 QO-100 地球静止轨道卫星通联设计。系统集成了全物理光电隔离、低侧 NMOS 软启供电、双向射频链路矩阵调度、Web 响应式控制仪表盘与 Web OTA 无线固件升级功能。

---

## 📑 目录 (Table of Contents)

1. [系统核心特性](#-系统核心特性)
2. [硬件架构与电气原理](#-硬件架构与电气原理)
3. [完整接线与引脚映射表](#-完整接线与引脚映射表)
4. [射频拓扑与信号流向](#-射频拓扑与信号流向)
5. [软件与 Web 控制台说明](#-软件与-web-控制台说明)
6. [编译与固件烧录指南](#-编译与固件烧录指南)
7. [装机调试与安全操作规程](#-装机调试与安全操作规程)
8. [开源协议](#-开源协议)

---

## 🌟 系统核心特性

* **全响应式 Web 仪表盘**：免装 App，手机/电脑连接 ESP32 Wi-Fi 热点即可访问控制台，具备动态 IF/LO/RF 算力看板与实时锁相环状态侦测。
* **100% 纯物理双光电隔离**：采用双 PC817 光耦独立隔离电台 13.8V PTT 信号与单片机/继电器回路，彻底杜绝电台与单片机之间的感应高压击穿与反峰电涌。
* **分级硬件供电与时序保护**：
  * **低侧 NMOS 功率开关**：消除继电器带载切换拉弧，实现功放（PA）快速软启与毫秒级切断。
  * **LNA 接收保护**：TX 发射瞬间继电器常闭端断开，微波低噪放毫秒级硬断电。
  * **TOT 发射超时保护**：软件设定发射时限（默认 180 秒），超时自动回切 RX 并闭锁，防止 PA 干烧。
  * **PTT 软件安全锁**：调机测试期间可一键切断发射响应。
* **多中频与目标频段智能换算**：
  * 支持 28MHz / 144MHz / 430MHz 常见中频电台无缝接入。
  * 覆盖 50M / 144M / 430M / 1.2G / 2.4G 业余频段，支持同频/异频中继差频（Shift）自动换算。
  * 内置 QO-100 卫星 CW / SSB / FT8 / DATV 快捷信道预设。
* **高精度时钟与功率控制**：
  * 适配 100MHz TCXO 板载晶振及外部 GPSDO（10MHz）参考基准。
  * 支持底层 $\pm\text{Hz}$ 级晶振频偏微调校准（Calibration）。
  * 软件寄存器四挡控制 ADF4351 输出功率（-4dBm / -1dBm / +2dBm / +5dBm），精确匹配 ADE-1 混频器的最佳 LO 激励电平。
* **Web OTA 无线固件升级**：变频器封装入屏蔽铝箱后，无需拆机接 USB 线，直接在网页上传 `.bin` 固件即可完成系统升级。

---

## 🛠️ 硬件架构与电气原理

系统主要由 **主电源与逻辑控制核心**、**双光耦隔离与功率执行单元**、**锁相环与射频变频链路** 三大部分构成：

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
