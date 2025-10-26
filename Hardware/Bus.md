# Bus

## AHB 和 APB
架构图可参考 `STM32 参考手册` 

AHB（Advanced High-performance Bus）和 APB（Advanced Peripheral Bus）是 ARM **AMBA（Advanced Microcontroller Bus Architecture）** 总线规范中的两条经典总线，分别服务于 **高性能** 和 **低功耗/低速外设** 场景。一句话区分：

- **AHB**：高速、高带宽、支持多主设备，用于 CPU、DMA、片上存储器等。
- **APB**：低速、低功耗、单主（桥），用于 GPIO、UART、Timer 等外设。

---

### 1. 主要特征对照表

| 特性               | AHB                            | APB                            |
|--------------------|--------------------------------|--------------------------------|
| **全称**           | Advanced High-performance Bus  | Advanced Peripheral Bus        |
| **定位**           | 系统骨干总线                   | 外设扩展总线                   |
| **带宽**           | 高（32/64/128-bit）            | 低（通常 32-bit）              |
| **时钟**           | 高频（与 CPU 同频或倍频）      | 低频（可单独分频）             |
| **主设备**         | 多主（CPU/DMA/其他主控）       | 单主（AHB-to-APB 桥）          |
| **仲裁**           | 支持多主仲裁                   | 无仲裁                         |
| **流水线/突发**    | 支持突发（burst）传输          | 不支持突发，单周期/非流水线   |
| **功耗**           | 较高                           | 低（适合长时间空闲外设）       |
| **典型连接**       | SRAM、Flash、DMA、LCD 控制器   | GPIO、UART、SPI、I²C、Timer    |

---

### 2. 典型 SoC 连接示意

```
CPU ── AHB ── Memory Controller ─ SRAM/Flash
        │
        ├─ DMA ── AHB Slaves
        │
        └─ AHB-APB Bridge ── APB ── GPIO
                                   UART
                                   Timer
                                   SPI/I²C
```

- **AHB** 作为主干，负责高速数据搬运；
- **APB** 通过桥接器挂接在 AHB 上，为低速外设提供简单、低功耗接口。

---

### 3. 何时选用？

| 场景示例                  | 推荐总线 |
|---------------------------|----------|
| CPU 访问片上 SRAM/Flash   | AHB      |
| DMA 搬运大量数据          | AHB      |
| 读取 GPIO 按键状态        | APB      |
| 配置 UART 波特率寄存器    | APB      |

---

### 一图速记
```
高速核心 ↔ AHB ↔ 存储器/DMA
                ↘
             AHB-APB Bridge ↔ APB ↔ 低速外设
```

总结：**AHB 跑得快、能仲裁、扛大旗；APB 跑得慢、低功耗、挂外设。**