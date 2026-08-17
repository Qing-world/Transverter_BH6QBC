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
### 项目技术说明文档 (Project Specification / 架构设计说明)

```markdown
# 多中频全频段微波变频器系统设计与技术规格说明书

## 1. 系统概述
本系统是一套专为业余无线电微波通联设计的高线性度、低相噪变频控制单元。传统微波变频器受限于固定晶体振荡器（Xtal）与硬件跳线，频率调整极不灵活。本设计采用 ESP32 作为控制中枢，驱动 ADF4351 宽带高频锁相环（35MHz ~ 4.4GHz），通过软件定义参数实现任意中频（HF/VHF/UHF）到微波频段的双向精密混频。

---

## 2. 射频拓扑设计与电平规划

### 2.1 接收链路 (RX Link)
* **信号流向**：RX 天线 $\rightarrow$ U15 (LNA 低噪声放大器) $\rightarrow$ U4 (RF 电子开关 RF1 通路) $\rightarrow$ U8 (SAW 带通滤波器) $\rightarrow$ U12 (ADE-1 双平衡混频器 RF 端) $\rightarrow$ U16 (RF 开关 RF1 通路) $\rightarrow$ U14 (RF 开关 RF2 通路) $\rightarrow$ 电台中频口 (RF_IN)。
* **设计考量**：
  * 天线微弱信号第一时间进入 LNA 放大，建立极低噪声系数（NF）。
  * 接收链路全程绕过 15dB 功率衰减器，实现中频无损回传至后级电台接收机。

### 2.2 发射链路 (TX Link)
* **信号流向**：电台中频口 (RF_IN) $\rightarrow$ U14 (RF 开关 RF1 通路) $\rightarrow$ U7 (15dB 功率衰减器) $\rightarrow$ U16 (RF 开关 RF2 通路) $\rightarrow$ U12 (混频器 IF 端) $\rightarrow$ U8 (SAW 滤波器) $\rightarrow$ U4 (RF 开关 RF2 通路) $\rightarrow$ U3 (2.5W 功率放大器) $\rightarrow$ TX 天线。
* **设计考量**：
  * 电台 TX 输出信号先经过 15dB 衰减器，将电平压制在混频器线性动态范围内（$-10\text{dBm} \sim +3\text{dBm}$），消除三阶互调失真（IMD3）并防止混频二极管烧毁。

---

## 3. 控制系统与电气隔离设计

### 3.1 双 PC817 物理光电隔离
* **PTT 信号采集 (K1)**：电台 ACC 口对地触发，输入回路走 13.8V 驱动 PC817 发光管；输出侧独立连接 ESP32 3.3V 逻辑电平（GPIO 32），实现电平硬隔离。
* **继电器驱动隔离 (K2)**：独立 PC817 隔离驱动 5V/12V 继电器线圈，彻底阻断继电器线圈断开时产生的反峰电压（Back-EMF）倒灌入单片机。

### 3.2 低侧 NMOS + 继电器协同控电
* 采用双 MOS 模块驱动 PA 负极回流通断，具备无机械磨损、毫秒级响应特性。
* 与继电器触点配合，确保收发转换时“先切射频开关、后通 PA 电源”，关断时“先断 PA 电源、后切射频开关”，避免热切换（Hot Switching）打火损坏射频元器件。

---

## 4. 软件系统与算法实现

1. **VCO 与分频器自适应搜索算法**：
   $$f_{VCO} = f_{LO} \times 2^N \quad (2200\text{MHz} \le f_{VCO} \le 4400\text{MHz})$$
   自动迭代计算 RF 分频比 $2^N$、参考时钟分频器 $R$、整数分频 $INT$ 与分数分频 $FRAC/MOD$，实现 1kHz 步进的精确锁相。
2. **微秒级抖动消除与 MUX 状态滤波**：
   在 `handleStatus()` 中引入多点循环采样机制，配合 ESP32 内部上拉，滤除高频环境下的感应毛刺，确保锁相环状态指示无跳变。
3. **NVS 非易失性参数存储**：
   所有工作模式、差频设定、频偏校准及功率挡位均实时持久化至 Flash，断电重启自动恢复。
