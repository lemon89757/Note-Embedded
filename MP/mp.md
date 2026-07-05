# 多核处理器笔记
## 概述
以一种有效的方式在一颗芯片中集成多个具有相同或者不同功能、不同结构的微处理器核，称之为多核异构。例如，一颗芯片中的 DSP 和 ARM 属于不同的架构，它们是异核。

采用大容量缓存、多核异构处理器已经成为一种更加有效降低成本和提高性能的嵌入式处理器系统设计方案。

问题：不同的核在运行不同的操作系统时，不同的核心是如何启动运行的？它们之间又是如何进行异核通信的？
在裸机或 RTOS 下（Cortex-M4 内核）会用到 OpenAMP 框架，在 Linux 下（Cortex-A7 内核）也会用到一些框架，**这些框架都是服务于共享内存的，即 Cortex-A7 内核核 Cortex-M4 内核之间进行核间通信，本质上是通过共享内存来完成的**。

多核处理器：
- STM32MP157：
  - 有两个 Cortex-A7 内核和一个 Cortex-M4 内核，并集成 3D GPU，它属于多核异构；
  - Cortex-A7 内核可以运行 Linux 操作系统；
  - Cortex-M4 内核可以运行裸机程序或 RTOS。

## OpenAMP
OpenAMP 是一个针对非对称多处理器（AMP）系统开发应用程序的软件框架。它提供了所需的软件组件，用户通过调用这些组件中的 API，就可以实现核间通信。

## IPCC
IPC (Inter-Process Communication, 进程间通信)是指两个进程的数据之间发生交互，它们是通过处理器间通信控制器 IPCC (Inter-Process Communication Controller, IPC 控制器) 来实现的。

IPCC 属于硬件部分，用于在两个 CPU 之间进行信号交换，邮箱依赖于 IPCC。

### 配置 STM32MP157
配置 Cortex-M4 工程，主要开启 IPCC 和 OpenAMP。

## 参考
1. 正点原子-STM32MP1 异核通信（基于 CubeIDE）