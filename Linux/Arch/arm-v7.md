# Note for ARM-v7

## 处理器状态处理函数
在 `arch/arm/mm/proc-v7.S` 中定义了 *Cortex-v7* 的处理器信息，其中包含了 *CPU* 状态转换时的处理函数：

1. `cpu_v7_do_idle()`: 当 *CPU* 无任务需要处理，进入空闲/低功耗状态时调用该函数
    ```c
    /*
     *	cpu_v7_do_idle()
     *	Idle the processor (eg, wait for interrupt).
     *	IRQs are already disabled.
     */
    ENTRY(cpu_v7_do_idle)
        dsb					@ WFI may enter a low-power mode
        wfi
        ret	lr
    ENDPROC(cpu_v7_do_idle)
    ```
    - `dsb` 指令：*Data Synchronization Barrier* (数据同步屏障)，确保当前核上所有已发起的内存访问（读/写）全部完成，并且清空缓冲区（*Write Buffer*），之后才会执行后面的指令
    - `wfi` 指令：*Wait for Interrupt*（等待中断），让 *CPU* 立即停止执行指令，进入低功耗，直到有中断（*IRQ*）、快速中断（*FIR*）或调试事件将其唤醒。具体行为
      - *CPU* 停止取指核执行
      - 时钟可能被门控（*gated*），大幅降低功耗
      - 保持寄存器和 *Cache* 状态（除非进入更深的睡眠层级）
      - 任何中断到来时立即恢复执行