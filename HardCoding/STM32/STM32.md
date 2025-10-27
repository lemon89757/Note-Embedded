# Note for STM32
## 简介
- STM32 系列
  ![](../../img/STM32/251026-163249.png)
- ARM 家族架构信息
  ![](../../img/STM32/251026-164128.png)
- `STM32F103C8T6`:
  - 系列：主流系列，`STM32F1`；
  - 内核：`ARM Cortex-M3`；
  - 主频：`72MHz`；
  - RAM: 20K(SRAM);
  - ROM: 64K(Flash);
  - 供电：2.0~3.6v（标准 3.3v）；
  - 封装：`LQFP48`；
  - 引脚定义
    ![](../../img/STM32/251026-174855.png)
    - 红色-电源相关引脚、蓝色-最小系统相关引脚、绿色- `IO` 口、功能口相关引脚（正常工作起码需要电源和最小系统部分连接好）；
    - `S`：电源相关；
    - `IO` 口电平：`IO` 口所能容忍的电压，`FT, Five Tolarance` 表示可以容忍 5V 电压，无则表示只能容忍 3.3V；
    - 主功能：上电后的默认功能；
    - 默认复用功能：配置 `IO` 口时可以选择是通用 `IO` 口还是复用功能；
    - 重定义功能：如果有两个功能同时复用在了一个 `IO` 口上，可以把其中一个复用功能重映射到其他端口上，前提是这个重定义功能的表里有对应端口；
- 引脚介绍：
  - `VBAT`：备用电池供电引脚。当系统电源断电时，备用电池可以给内部的 `RTC` 时钟和备份寄存器提供电源；
  - `PC13-TAMPER-RTC`：可用作 `IO` 口或者侵入检测或者 `RTC`。侵入检测可以用来实现安全保障功能；`RTC` 引脚可以用来输出 `RTC` 校准时钟、`RTC` 闹钟脉冲或秒脉冲；
  - `PC14-OSC32_IN, PC15-OSC32_OUT`：可用作`IO` 口或者 `32.768KHz` （2 的 15 次方）的 `RTC` 晶振的输入输出端口；内部 `RTC` 电路经过 2 的 15 次方分频，就可以生成 1 秒的时间信号；
  - `OSC_IN, OSC_OUT`：用作系统主晶振的输入输出端口，一般为 `8MHz`，可通过内部锁相环电路倍频到 `72MHz`，以作为系统的主时钟；
  - `NRST`：系统复位引脚，`N` 表示低电平复位；
  - `VSSA, VDDA`：内部模式部分的电源，比如 `ADC, RC 震荡器`等。`VSS` 为负极，`VDD` 为正极；
  - `PA0-WKUP`：用作 `IO` 口或唤醒功能。可唤醒处于待机模式的 `STM32`；
  - `PB2/BOOT1`：可用于设置启动模式（`BOOT1` 时）；优先使用加粗的 `IO` 口；
  - `JTMS/SWDIO, JTCK/SWCLK, JTDI, JTDO, NJTRST`：`IO` 口（需额外配置）或调试端口，可用来调试程序和下载程序。支持 `SWD` 和 `JTAG` 两种调试方式，`SWD` （`STLink` 使用的方式）需要两根线：`SWDIO` 和 `SWCLK`；`JTAG` 需要五根线：`JTMS, JTCK, JTDI, JTDO, NJTRST`；
  - `BOOT0`：用作启动配置；
- 命名规则
  ![](../../img/STM32/251026-171135.png)
- 系统结构
  ![](../../img/STM32/251026-171446.png)
  来自 《STM32F10xxx参考手册》。
  - 内核：
    - `ICode`（指令总线）和 `DCode`（数据总线），连接`Flash`（存储程序）；
    - `System`（系统总线）：连接到 `SRAM`（存储程序运行时的数据）、`FSMC`、`AHB 系统总线`等；
  - `AHB`（外设相关）:
    `Advance High Performance Bus`，用于挂载主要的外设（最基本、性能较高的外设）：
    - 复位和时钟控制（`RCC`）；
    - `SDIO, Secure Digital Input/Output`；
    - 桥接电路，连接到 `APB, Advance Peripheral Bus`，`AHB` 性能总体比 `APB` 高一些，其中 `APB2` 的性能又比 `APB1` 高一些，`APB2` 和 `AHB` 一般同频率（都是 `72MHz`，`APB1` 一般是 `36MHz`）；
      - `APB2`：连接外设中更加重要的部分，如 `GPIO 端口`、一些外设的一号选手（`UASRT1`、`SPI1`、`TIM1`、`TIM8`）、`ADC`、`EXTI`、`AFIO`等；
      - `APB2`：其他编号外设、`DAC`、`PWR`、`BKP` 等；
  - `DMA`：
    执行数据搬运等工作，特点是简单重复；
    - 通过 `DMA` 总线连接到总线矩阵上；
    - 可以拥有和 `CPU` 一样的总线控制权，用于访问外设；
    - 当需要 `DMA` 搬运数据时，外设通过请求线与发送 `DMA` 请求，然后在 `DMA` 获得总线控制权后，访问并转运数据；整个过程无需 `CPU` 的参与，提高了 `CPU` 的工作效率；

## 启动配置
指定开始运行时的程序位置。一般情况下，程序都是在 `Flash` 程序存储器开始执行的，通过设置启动配置可以改变程序开始执行的位置。
![](../../img/STM32/251026-181145.png)
- 系统存储器存储了 `STM32` 中的一段 `Bootloader` 程序，其作用是接收串口的数据，然后刷新到主闪存（`Flash`）中，可用于串口下载程序；
- “在系统复位后，`SYSCLK` 的第 4 个上升沿，`BOOT` 引脚的值将被所存。用户可以通过设置 `BOOT1` 和 `BOOT0` 引脚的状态，来选择在复位后的启动模式”。说明 `BOOT` 引脚的值是在上电复位后的一瞬间有效的，之后就随便了（第 4 个时钟过后，`BOOT1` 转为 `PB2` 的功能）。

## 片上资源/外设（`peripheral`）：
`STM32F1` 整个系列的所有外设，并不是所有芯片都具有如下所有外设：
![](../../img/STM32/251026-164908.png)
  - 深颜色为位于 `cortex` 内核里的外设；
  - 其他为内核外外设；

## `NVIC`
设置中断优先级等；

## `RCC`
`Reset and Clock Controller`，对系统的时钟进行控制以及使能各模块的时钟。在 `STM32` 中，其他的外设在上电的情况下默认是没有时钟的，此时操作外设无效，这样的目的是降低功耗。因此，在操作外设之前必须要先使能它的时钟。

## `EXTI`
外部中断/事件控制器。包含 19 个边沿检测器，用于产生中断/事件请求。

## `IWDG` 和 `WWDG`
看门狗就是处理器处理任务时，要定期完成喂狗操作，当程序死了，就相当于忘记去喂狗，狗饿死了，触发复位。

## 相关电路
- 最小系统电路
  ![](../../img/STM32/251026-222519.png)
  - 电源部分的电容，用于提供供电电源的稳定；

## 裸机编程实操
### 开发方式
- 基于寄存器：直接操作寄存器；
- 基于函数库：使用 `ST` 官方提供的封装好的函数库，间接配置寄存器；
- 基于 `HAL` 库：图形化界面快速配置 `STM32`；

### 基于函数库的 STM32 开发
- 固件库 - `STM32F10x_StdPeriph_lib`
  - `Libraries`：库
    - `CMSIS`
    - `STM32F10x_StdPeriph_Driver`
  - `Project`：官方示例和模板；
  - `Utilies`：官方评估板的相关例程；
  - `Release_Notes`：发布文档，含版本说明等；
  - `stm32f10x_stdperiph_lib_um`：使用手册，含库函数使用说明等；
- `Keil` 新建工程（空）：
  - `New Project -> New uVision Project -> 选择本工程对应的文件夹 + 命名工程名 -> 选择器件型号 -> 跳过新建工程助手`；
  - 启动相关文件：
    - 启动文件（`固件库/Libraries/CMSIS/DeviceSupport/ST/STM32F10x/startup/arm/*`）：
    - `STM32` 的外设寄存器描述文件（`固件库/Libraries/CMSIS/DeviceSupport/ST/STM32F10x/stm32f10x.h`）和其他文件（主要用来配置系统时钟：`system_stm32f10x.c` 和 `system_stm32f10x.h`）；
    - 内核寄存器描述文件（`固件库/Libraries/CMSIS/CM3/CoreSupport/core_cm3.c 和 core_cm3.h`）；
  - 复制上述所有文件到新建工程中：新建文件夹 `Start`；
  - 在工程中新建目录（组） `Start`，添加启动文件工程中。其中，启动文件选择后缀为 `_md.s` 的文件（需要根据芯片型号(`Flash` 容量)来选择，`md` 表示中等容量），其余的 `.c` 和 `.h` 均添加进来：
    ![](../../img/STM32/251027-222150.png)
  - 添加相关文件的包含路径：`魔术棒 -> C/C++ -> Include Paths`；
  - 在工程文件夹下新建 `User`，项目入口 `main` （`main.c`）放到该位置，并在 `Keil` 工程做相应操作；
  - `main`：
    ```c
    #include "stm32f10x.h"

    int main(void){
      RCC->APB2ENR = 0x00000010;  // 使能时钟
      GPIOC->CRH = 0x00300000;    // GPIO 设置，电气属性、方向
      GPIOC->ODR = 0x00002000;    // 写数据，输出寄存器
      while (1) {

      }
    }

    ```
  - 调试器选择：`魔术棒 -> Debug -> Use 选择 ST-Link Debugger`，并点击 `Settings`，在 `Flash Download` 中勾选 `Reset and Run`（下载程序后会立马复位并执行）；
  - 添加其他函数，新建 `Library` 文件夹（本地目录和工程目录），添加固件库中的 `.../Libraries/STM3210x_StdPeriph_Driver/src/*`，其中 `misc.c`（混杂）为内核库函数，其他为内核外的库函数；同理，将与 `src` 同级的 `inc/*` 复制到工程中；
  - 添加 `.../Project/STM32F10x_StdPeriph_Template` 下的 `stm32f10x_conf.h`（用来配置库函数头文件的包含关系，以及用来参数检查的函数定义，所有库函数都需要）、`stm32f10x_it.c` 和 `stm32f10x_it.h`（中断相关）到 `User` 目录下；
  - 添加宏定义以包含标准外设库，在 `魔术棒 -> C/C++ -> Define` 中添加 `USE_STDPERIPH_DRIVER`（与 `stm32f10x.h` 中的条件编译相关）；
  - 完善头文件包含路径；
  ```c
  #include "stm32f10x.h"

  int main(void){
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
    GPIO_InitTypeDef GPIO_InitStructure;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;    // 推挽输出
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);
    // GPIO_SetBits(GPIOC, GPIO_Pin_13);
    GPIO_ResetBis(GPIOC, GPIO_Pin_13);
    while (1) {

    }
  }

  ```
  - 其他设置：右上角扳手图标；
- 工程架构
  ![](../../img/STM32/251027-222716.png)
  - `startup_xxx.s`：
    ```asm
    Reset_Handler   PROC
                    EXPORT    Reset_Handler     [WEAK]
          IMPORT  _main
          IMPORT  SystemInit
                    LDR   R0, =SystemInit
                    BLX   R0
                    LDR   R0, =_main
                    BX    R0
                    ENDP
    ```
    - `SystemInit` -> `system_xx.c/.h`，Setup the microcontroller system, Initialize the Embedded Flash Interface, the PLL and update the SystemCoreClock variable；
  - 在 `stm32f10x.h` 在对 `stm32f10x_conf.h` 进行条件编译；

## 参考
- [江科大视频](https://www.bilibili.com/video/BV1th411z7sn?spm_id_from=333.788.videopod.episodes&vd_source=f87b1dc9ce95722a192942b341ae04d2&p=2)
