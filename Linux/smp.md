# Note for SMP
这是一个关于 *SMP* 综合性文件，部分内容详细记录在别的笔记文件中，在这里会提供向导。

## CPU Mask
```c
// include/linux/cpumask.h
/* Cpumasks provide a bitmap suitable for representing the set of
 * CPU's in a system, one bit position per CPU number. In general,
 * only nr_cpu_ids (<= NR_CPUS) bits are valid.
 */

/* Don't assign or return these: may not be this big! */
typedef struct cpumask { DEClARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```
- `DECLARE_BITMAP(bits, NR_CPUS)` 展开为：`unsigned long bits[xxx]`，其中 `xxx` 为按照 `NR_CPUS` 对 `sizeof(long) * 8` 的向上取整。即表示 `NR_CPUS` 所需的位数，且按照 `sizeof(long) * 8` 对齐；
- `cpumask->bits` 表示的位图中只有 `nr_cpu_ids` 位是有效的，且 `nr_cpu_ids` 满足小于等于 `NR_CPUS`。
  - `NR_CPUS`: 编译时通过内核配置（`CONFIG_NR_CPUS`）设定的最大支持 CPU 数量；
  - `nr_cpu_ids`: 系统启动时实际探测到的有效 CPU 数量。
- `cpumask->bits` 位图中每一个比特位严格对应一个 CPU 编号（CPU ID）。若某位为 1，表示该 CPU 属于当前集合；反之，不属于；
- 典型应用场景：
  - 进程/线程的 CPU 亲和性控制（`task_struct::cpus_allowed`）；
  - 调度器可运行 CPU 集合管理（`cpus_ready`, `cpus_possible`）；
  - 中断请求（IRQ）与 CPU 绑定（`irq_affinity`）；
  - 全局 CPU 状态掩码（如 `cpu_online_mask`, `cpu_present_mask`）。

## 相关 API
1. `DECLARE_PER_CPU()`

## SMP 的通信
- [ ] GIC 框图

> [1]
- Linux 系统下，SMP 架构 CPU 核间通过预定义的 `IPI, Inter-Processor Interrupts` 中断来进行通信：
    ```c
    enum ipi_msg_type {
        IPI_WAKEUP,
        // 定时器广播
        IPI_TIMER,
        // 远程 CPU 执行任务调度。
        // 当 被唤醒进程 不是运行于本地 CPU 上时，就需要给 被唤醒进程 所在的 远程 CPU 发送 IPI 中断，让远程 CPU 发起调度
        // 在接收到 IPI 时，无论远程 CPU 是否处于空闲态，被唤醒进程都会在 IPI 中断进入到内核态时得到执行
        IPI_RESCHEDULE,
        // 在指定 CPU 集合上运行函数
        IPI_CALL_FUNC,
        // 远程 CPU 停止运行。
        // 先将指定的远程 CPU 从在线 CPU 位图中删除，然后将 D，A，I，F 中断屏蔽，最后处理器进入低功耗循环。
        IPI_CPU_STOP,
        IPI_IRQ_WORK,
        IPI_COMPLETION,
        IPI_CPU_BACKTRACE,
        /* SGI8-15 can be reserved by secure firmware, and thus
         * may not be usable by the kernel. Please keep the above
         * limited to at most 8 entries.
         */
    };
    ```
- 上述消息类型是 Linux 为 SMP 架构下 CPU 核间通信定义的、架构无关的消息类型。Linux 负责定义这些消息的语义，而这些消息语义的具体实现，依赖于具体的硬件架构类型。
- ARM 架构下通过 GIC 芯片生成 SGI (Software-generated interrupt) 中断来实现 `IPI_*` 消息的语义。其过程为：通过 GIC 生成一个 SGI 中断，发送给目标 CPU 集合，然后目标 CPU 处理 SGI 中断。
- `IPI_TIMER`
  - 如 BOOT CPU 发送定时器广播事件，唤醒系统中的其他非 BOOT CPU；
  - 先生成一个编码了 IPI_TIMER 信息的 SGI 中断，发送给目标 CPU。实现：
    ```c
    // version: 4.1.15
    // arch/arm/kernel/smp.c
    #ifdef CONFIG_GENERIC_CLOCKEVENTS_BROADCAST
    void tick_broadcast(const struct cpumask *mask) {
        smp_cross_call(mask, IPI_TIMER);
    }
    #endif

    static void smp_cross_call(const struct cpumask *target, unsigned int ipinr) {
        trace_ipi_raise_rcuidle(target, ipi_types[ipinr]);
        __smp_cross_call(target, ipinr);    /* gic_raise_softirq(), ... */
    }
    ```
    ```c
    #ifdef CONFIG_SMP
    static void gic_raise_softirq(const struct cpumask *mask, unsigned int irq) {
        ...
        /* 
         * GIC 生成一个 SGI 中断： 
         * map << 16：接收 SGI 中断的目标 CPU 掩码
         * irq: 生成的 SGI 中断编号（如 IPI_TIMER）
         */
        writel_relaxed(map << 16 | irq, gic_data_dist_base(&gic_data[0]) + GIC_DIST_SOFTINT);
    }
    ```
  - 目标 CPU 处理编码了 IPI_TIMER 信息的 SGI 中断
    ```c
    gic_handle_irq()
        handle_IPI(irqnr, regs)
    ```
    ```c
    // arch/arm/kernel/smp.c
    void handle_IPI(int ipinr, struct pt_regs *regs) {
        unsigned int cpu = smp_processor_id();
        struct pt_regs *old_regs = set_irq_regs(regs);

        if ((unsigned)ipinr < NR_IPI) {
            trace_ipi_entry_rcuidle(ipi_types[ipinr]);
            // 统计 IPI 中断次数，可从 /proc/interrupts 观察到
            __inc_irq_stat(cpu, ipi_irqs[ipinr]);
        }

        switch (ipinr) {
        case IPI_WAKEUP:
            break;
    #ifdef CONFIG_GENERIC_CLOCKEVENTS_BROADCAST
        case IPI_TIMER:
            irq_enter();
            tick_receive_broadcast();
            irq_exit();
            break;
    #endif
        ...
        }
        ...
    }
    ```
- `IPI_RESCHEDULE`
  - `IPI_RESCHEDULE` 用于将进程调度到非当前 CPU 运行：
    ```c
    void smp_send_reschedule(int cpu) {
        smp_cross_call(cpumask_of(cpu), IPI_RESCHEDULE);
    }
    ```
    ```c
    gic_handle_irq()
        handle_IPI(irqnr, regs)
            switch (irqnr) {
                ...
            case IPI_RESCHEDULE:
                scheduler_ipi();
                break;
                ...
            }
    
    void scheduler_ipi(void) {
        ...
        irq_enter();
        sched_ttwu_pending();
        ...
        irq_exit();
    }
    ```
- `IPI_CALL_FUNC`
  - `IPI_CALL_FUNC` 用来在指定 CPU 上运行函数，如 `smp_call_function_single()` 在指定的 CPU 核上运行回调函数：
    ```c
    // include/linux/smp.h
    // kernel/smp.c
    int smp_call_function_single(int cpuid, smp_call_func_t func, void *info, int wait) {
        call_single_data_t *csd;
        call_single_data_t csd_stack = {
            // CSD_FLAG_SYNCHRONOUS 标志，指示等待 smp call 回调执行完成再返回
            .flags = CSD_FLAG_LOCK | CSD_FLAG_SYNCHRONOUS,
        };
        int this_cpu;
        int err;

        /* prevent preemption and reschedule on another processor
         * as well as CPU removal
         */
        this_cpu = get_cpu();
        ...
        csd = &csd_stack;
        if (!wait) {
            csd = this_cpu_ptr(&csd_data);
            csd_lock(csd);
        }

        err = generic_exec_single(cpu, csd, func, info);
        if (wait)
            csd_lock_wait(csd);     // 等待回调执行完成
        put_cpu();
        return err;
    }

    static int generic_exec_single(int cpu, call_single_data_t *csd, smp_call_func_t func, void *info) {
        if (cpu == smp_processor_id()) {    // 在当前 CPU 执行，不用发起 IPI
            /* we can unlock early even for the synchronous 
             * on-stack case, since we're doing this from the same
             * CPU ...
             */
            csd_unlock(csd);
            lock_irq_save(flags);
            func(info);
            lock_irq_restore(flags);
            return 0;
        }

        // 在非当前 CPU 执行，发起 IPI
        ...
        csd->func = func;
        csd->info = info;

        // 通过 IPI 中断，发起远程 CPU 函数调用
        // 回调 @csd 添加到 @cpu 的回调列表 @call_single_queue
        if (llist_add(&csd->llist, &per_cpu(call_single_queue, cpu)))
            arch_send_call_function_single_ipi(cpu);
        return 0;
    }

    void arch_send_call_function_single_ipi(int cpu) {
        smp_cross_call(cpumask_of(cpu), IPI_CALL_FUNC);
    }

    // 处理 IPI_CALL_FUNC 中断，在目标 CPU 上调用回调
    gic_handle_irq()
        handle_IPI(irqnr, regs)
            switch(ipinr) {
                ...
            case IPI_CALL_FUNC:
                irq_enter();
                generic_smp_call_function_interrupt();
                irq_exit();
                break;
            ...
            }
    
    #define generic_smp_call_function_interrupt generic_smp_call_function_single_interrupt

    void generic_smp_call_function_single_interrupt(void) {
        flush_smp_call_function_queue(true);
    }

    static void flush_smp_call_function_queue(bool warn_cpu_offline) {
        struct llist_head *head;
        struct llist_node *entry;
        call_single_data_t *csd, *csd_next;
        ...
        head = this_cpu_ptr(&call_single_queue);
        entry = llist_del_all(head);
        entry = llist_reverse_order(entry);
        ...
        // 调用当前 CPU 回调列表 @call_single_queue 中的所有回调函数
        llist_for_each_entry_safe(csd, csd_next, entry, llist) {
            smp_call_func_t func = csd->func;
            void *info = csd->info;

            /* Do we wait until *after* callback ? */
            if (csd->flags & CSD_FLAG_SYNCHRONOUS) {    // smp call 等到 func 运行完成后返回，所以先不释放锁
                func(info);
                csd_unlock(csd);
            } else {    // smp call 不等 func 运行完成就返回，所以先释放锁
                csd_unlock(csd);
                func(info);
            }
        }
        ...
    }
    ```
- 其他...
- 通过 `cat /proc/interrupts` 观察 IPI 中断发生次数

> [3]
- AMP 与 SMP 架构对比
- 核间通信（IPC）机制详解（...），核间通信的核心组件：
  - 共享内存（shared Memory）：作为数据传输的物理载体
  - 软件中断（SGI/IPI）：用于通知对方核心有新数据到达
  - 内存屏障（DMB/DSB）：确保缓存一致性
- SMP 架构负载均衡机制
- 缓存一致性与内存屏障
  - 在多核系统中，每个核心拥有独立的 L1 Cache。当多个核心访问同一内存地址时，可能出现缓存不一致的问题；
  - 通过缓存一致性协议（MESI, Modified->Exclusive->Shared->Invalid）和内存屏障指令确保数据一致性；
  - ARM 内存屏障指令：
    ```c
    /* ARM Cortex-M/R/A 内存屏障指令详解 */

    /* 
     * DMB - Data Memory Barrier 
     * 确保屏障前的所有内存访问完成后再执行屏障后的内存访问
     * 使用场景：核间数据同步、DMA 操作
     */
    #define DMB() __asm volatile ("dmb" ::: "memory")

    /* 
     * DSB - Data Synchronization Barrier 
     * 确保所有缓存、TLB、分支预测操作完成
     * 使用场景：上下文切换、系统状态变更
     */
    #define DSB() __asm volatile ("dsb" ::: "memory")

    /* 
     * ISB - Instruction Synchronization Barrier
     * 刷新流水线：确保后续指令重新从内存获取
     * 使用场景：代码动态修改、MPU 配置变更
     */
    #define ISB() __asm volatile ("isb" ::: "memory")

    // 核间通信中的内存屏障使用示例
    /* 共享内存结构体 */
    typedef struct {
        volatile uint32_t flag;     // 数据就绪标志
        volatile uint32_t data;     // 共享数据
        volatile uint32_t seq;      // 序列号
    } SharedMem_t;

    // Core 0 写入数据
    void vCore0WriteData(SharedMem_t *shm, uint32_t value) {
        shm->data = value;
        shm->seq++;

        // 数据内存屏障，确保数据写入完成
        DMB();

        // 设置标志，通知 Core 1 数据就绪
        shm->flag = 1;

        // 触发软件中断通知 Core 1
        vTriggerSGI(1, 1);   // 向 Core 1 发送 SGI 中断
    }

    // Core 1 读取中断数据
    uint32_t vCore1ReadData(SharedMem_t *shm) {
        uint32_t value;

        // 等待标志被设置（在中断处理中检查）
        while (shm->flag == 0) {
            // 等待或休眠
            __WFE();        // 等待事件
        }

        // 数据同步屏障 - 确保看到最新的数据
        DSB();

        // 读取数据
        value = shm->data;

        shm->flag = 0;

        DMB();      // 确保标志清除被其他核心看到

        return value;
    }
    ```
- 典型应用场景
  - AMP：工业智能网关
  - SMP：高吞吐网络处理器
- 架构选择决策树
- 常见问题与最佳实践
  - AMP 常见问题：核间通信延迟高、共享内存数据损坏、RPMsg 通道建立失败、实时核被 Linux 干扰...
  - SMP 常见问题：任务在核间频繁迁移（缺少 CPU 亲和性设置）、自旋锁导致死锁（中断上下文获取锁）、缓存伪共享（多个核心访问同一缓存行，按缓存行对齐（64 字节）分离数据）、负载不均衡（任务优先级设计不当）
  - 示例：
    ```c
    // SMP 性能优化：避免缓存伪共享
    // 错误示例-多个核心频繁修改同一缓存行
    struct BadLayout {
        uint32_t core0_counter; // core 0 修改
        uint32_t core1_counter; // core 1 修改 - 同一缓存行
        uint32_t core2_counter; // core 2 修改 - 同一缓存行
    };

    // 正确示例 - 按缓存行对齐分离
    struct __attribute__((aligned(64))) GoodLayout {
        uint32_t counter;
        uint8_t padding[60];    // 填充至 64 字节缓存行
    };

    struct GoodLayout gCoreCounters[3]; // 每个核心独立缓存行

    // AMP 性能优化：批量传输减少中断
    // 批量发送传感器数据，减少 RPMsg 中断频率
    #define BATCH_SIZE 16

    void vBatchSendSensorData(void) {
        static SensorData_t batchBuffer[BATCH_SIZE];
        static uint8_t batchIndex = 0;

        // 采集数据到批量缓冲区
        batchBuffer[batchIndex].temperature = fReadTemp();
        batchBuffer[batchIndex].humidity = fReadHumidity();
        batchIndex++;

        if (batchIndex >= BATCH_SIZE) {
            rpmsg_send(&lept, batchBuffer, sizeof(SensorData_t) * BATCH_SIZE);
            batchIndex = 0;
        }
    }
    ```
- 总结与展望
    AMP 与 SMP 代表了嵌入式多核系统的两种截然不同的设计哲学
  - AMP 强调隔离与异构，通过让不同核心运行最适合其角色的操作系统，实现功能与实时性的最佳平衡。OpenAMP 框架的标准化使得 Linux + RTOS 的异构组合成为工业界的成熟方案。
  - SMP 强调统一与对称，通过单一操作系统映像和全局调度器，实现开发简化与自动负载均衡。FreeRTOS SMP 的成熟让嵌入式开发者也能够享受到多核并行处理的便利。
  - 在实际项目中，选择 AMP 还是 SMP 取决于以下核心问题：系统是否需要同时运行不同操作系统、是否需要强实时隔离。

> [4]
- 本地 CPU，即 `smp_processor_id()` 返回的 CPU
- 通过调用 `smp_call_function` 函数来指定其他一个或若干个 CPU 来执行某个回调函数：
  - `smp_call_function` 函数的基本参数：
    - `int cpu` 被指定的一个 CPU；
    - `struct cmpmask *mask` 被指定的一个 CPU 集合，这里可能有两种模式：1.在指定的 CPU 集合内的任意一个 CPU，在指定 CPU 集合中选择一个 CPU 的优先级为：本地 CPU；当前 NUMA 节点上的 CPU；其他在线 CPU。2. 在指定 CPU 集合上的每一个 CPU；
    - `smp_call_func_t func` 回调函数指针；
    - `int wait` 表示 `smp_call_function` 是否等待回调函数完成；
  - 指定 CPU：`int smp_call_function_single(int cpu, smp_call_func_t func, void *info, int wait)`；
  - 指定 CPU 集合中的一个 CPU：`int smp_call_function_any(const struct cpumask *mask, smp_call_func_t func, void *info, int wait)`；
  - 指定 CPU 集合中的每个 CPU：`void smp_call_function_many(const struct cpumask *mask, smp_call_func_t func, void *info, bool wait)`；
  - 系统内其他所有 CPU：`int smp_call_function(smp_call_func_t func, void *info, int wait)`；
- IPI 类型、目标及使用
- `smp_cross_call` 的调用链、最终实现

> [5]
- IPI, Inter-Processor Interrupt，需要可编程中断控制器 PIC 或 APIC 的支持，PIC 允许一个硬件线程中断其他的硬件线程，这种方式被称为核间中断，即 IPI；
- 使用 IPI 进行核间通信的关键在于要利用中断服务例程 ISR 去读取一个事先约定好的共享内存区域。发起方首先将消息写到一块共享内存中，然后发起核间中断。被中断的硬件线程在中断服务例程中读取该内存，以获得发起方通知的消息。
- 为防止多核间的竞争导致消息被改写，使用这种方式必须利用锁机制来确保消息的完整性。
- IPI 中断作用：多核处理器间通信，提升并行运算，多核处理能力，多核消息同步，多核负载均衡，发挥多核处理器高性能的优势。
- Linux 中断亲和性，IRQ 中断可绑定到哪些 CPU；
- 通过 `cat /proc/irq/{number}/smp_affinity` 查看中断号为 `number` 的中断亲和性；通过 `cat /proc/interrupts` 查看总体的中断信息；
- 其实在很大程度上，让专门的 CPU 处理专门的中断是效率最高的；
- `smp_affinity` 文件默认是全部 fffffff，一般一台机器只有几个 CPU，`echo 3 > /proc/irq/24/smp_affinity` 就是分配第一和第二个 CPU 给该 irq；
- `IRQ Affinity`：Binding IRQ's to a group of CPU's is a new feature of the 2.4 kernel. While it was originally developed as part of Red Hat Content Accelerator, it is now a generic and independent kernel feature. Every IRQ source in Linux has an entry in `/proc/irq` directory. For example, the settings for IRQ 40 is stored in `/proc/irq/40`. IRQ affinity, or IRQ bindings, is configured through the smp_affinity setting in that directory. For example, the smp_affinity for IRQ 40 is in `/proc/irq/40/smp_affinity`. The value of the smp_affinity setting is a bitmask of all CPU's that are permitted as a resource for the given IRQ. The default valur for smp_affinity is the HEX value 0xffffffff. This means the processors for the IRQ are sent to all CPU's. You are not allowed to turn off all CPU's for an IRQ. If the IRQ controller does not support IRQ affinity, the value can not be changed from the default. If multiple CPU's are defined, then the IRQ sources uses the least busy CPU. This called 'lowest priority APIC routing'. IRQ affinity is achieved by binding an IRQ to a specific CPU or group of CPU's by echoing a HEX value to smp_affinity for the IRQ.
- [Linux 内核文档-SMP IRQ affinity](https://www.kernel.org/doc/html/latest/core-api/irq/irq-affinity.html)

> [6]
- 在中断控制器驱动初始化时将调用 `set_smp_cross_call()` 函数去设定触发 IPI 的回调函数。
- IPI 不仅需要能够被一个核或多个核捕获，而且还需要能够被中断处理函数辨识出。在使用 GIC 中断控制器的系统中，小于等于 15 的硬中断号（`hwirq`）都属于 SGI，Software Generated Interrupt。在 Linux Kernel 中，默认定义了 8 个 IPI 中断。
    ```c
    enum ipi_msg_type {
        IPI_WAKEUP,         // 唤醒其他 CPU
        IPI_TIMER,          // 调用 tick 时钟设备 [tick-clock device] 的事件处理程序
        IPI_RESCHEDULE,     // 唤醒挂起的任务
        IPI_CALL_FUNC,      // 让其他 CPU 执行特定的函数
        IPI_CPU_STOP,       // 暂停其他 CPU
        IPI_IRQ_WORK,       // 执行当前 CPU 的 irq_work
        IPI_COMPLETION,
        NR_IPI,
        IPI_CPU_BACKTRACE = NR_IPI,
        MAX_IPI
    };
    ```
- 当 ARM core 受到 IRQ 后，会触发 cpu 的 irq 异常，会跳转到 Linux Kernel 的 irq 异常向量表，在该向量表中，会调用 gicv3 的 `gic_handle_irq` 函数。在 Linux Kernel 5.0 之前，在 `gic_handle_irq()` 的 gic 处理函数中，会判断硬件中断号，中断号小于 16 的属于 SGI 中断，单独走 `handle_IPI()` 函数，用于处理核间中断。5.0 之后不再区分，统一在 `handle_domain_irq()` 中处理。
- 在 Linux Kernel 5.0 中为硬件中断号 0-7 的每个中断号都注册了一个中断处理函数，其指向 `ipi_handler()` 函数：
    ```c
    void __init set_smp_ipi_range(int ipi_base, int n) {
        int i;

        WARN_ON(n < MAX_IPI);
        nr_ipi = min(n, MAX_IPI);

        for (i = 0; i < nr_ipi; i++) {
            int err;
            err = request_percpu_irq(ipi_base + i, ipi_handler, "IPI", &irq_stat);
            WARN_ON(err);
            ipi_desc[i] = irq_to_desc(ipi_base + i);
            irq_set_status_flags(ipi_base + i, IRQ_HIDDEN);
        }

        ipi_irq_base = ipi_base;

        /* Setup the boot CPU immediately */
        ipi_setup(smp_processor_id());
    }
    ```
    - 在触发中断异常后，它核一般的 `request_irq` 注册的中断没有什么不同，都是从异常向量表调过来，然后调用到这个 `ipi_handler()` 程序，该程序再调用 `do_handle_IPI()` 真正处理核间通信的任务：
        ```c
        static void do_handle_IPI(int ipinr) {
            unsigned int cpu = smp_processor_id();

            if ((unsigned)ipinr < NR_IPI)
                trace_ipi_entry_rcuidle(ipi_types[ipinr]);
            switch (ipinr) {
            case IPI_RESCHEDULE:
                scheduler_ipi();
                break;
            case IPI_CALL_FUNC:
                generic_smp_call_function_interrupt();l
                break;
            case IPI_CPU_STOP:
                local_cpu_stop();
                break;
            case IPI_CPU_CRASH_STOP:
                if (IS_ENABLED(CONFIG_KEXEC_CORE)) {
                    ipi_cpu_crash_stop(cpu, get_irq_regs());
                    unreachable();
                }
                break;
        #ifdef CONFIG_GENERIC_CLOCKEVENTS_BROADCAST:
            case IPI_TIMER:
                tick_receive_broadcast();
                break;
        #endif
        #ifdef CONFIG_IRQ_WORK
            case IPI_IRQ_WORK:
                irq_work_run();
                break;
        #endif
        #ifdef CONFIG_ARM64_ACPI_PARKING_PROTOCOL
            case IPI_WAKEUP:
                WARN_ONCE(!acpi_parking_protocol_valid(cpu),
                        "CPU%u: Wake-up IPI outside the ACPI parking protocol\n",
                        cpu);
                break;
        #endif
            default:
                pr_crit("CPU%u: Unknown IPI message 0x%x\n", cpu, ipinr);
                break;
            }

            if ((unsigned)ipinr < NR_IPI)
                trace_ipi_exit_rcuidle(ipi_types[ipinr]);
        }
        ```
- 触发核间中断：
  - `smp_call_function(func, info, wait)`，在所有其他处理器执行一个函数；
  - `smp_call_function_single(int cpuid, smp_call_func_t func, void *info, int wait)`，在指定处理器执行一个函数；
  - `smp_send_reschedule(int cpu)`，指定处理器重新调度进程
- SMP (IPI) 的这类函数，最终都是调用到 `gic_send_sgi()`，给 core 发送 SGI 中断：
    ```c
    static void gic_send_sgi(u64 cluster_id, u16 tlist, unsigned int irq) {
        u64 val;

        val = (MPIDR_TO_SGI_AFFINITY(cluster_id, 3) |
            MPIDR_TO_SGI_AFFINITY(cluster_id, 2) |
            irq << ICC_SGI1R_SGI_ID_SHIFT |
            MPIDR_TO_SGI_AFFINITY(cluster_id, 1) |
            MPIDR_TO_SGI_RS(cluster_id) |
            tlist << ICC_SGI1R_TARGET_LIST_SHIFT);
        
        pr_devel("CPU%d: ICC_SGI1R_EL1 %llx\n", smp_processor_id(), val);
        gic_write_sgi1r(val);
    }

    static inline void gic_write_sgi1r(u64 val) {
        write_sysreg_s(val, SYS_ICC_SGI1R_EL1);
    }
    ```

> [7]
- 概述、应用场景、性能影响

> [8]
- IPI 概况
- 多种 IPI 类型的作用、使用场景、调用路径和示例

---
1. `smp_cross_call()`
    ```c
    // version: 4.1.15
    // arch/arm/kernel/smp.c
    static void smp_cross_call(const struct cpumask *target, unsigned int ipinr) {
        trace_ipi_raise(target, ipi_types[ipinr]);
        __smp_cross_call(target, ipinr);
    }

    static void (*__smp_cross_call)(const struct cpumask *, unsigned int);

    void __init set_smp_cross_call(void (*fn)(const struct cpumask *,
        unsigned int)) {
        if (!__smp_cross_call)
            __smp_cross_call = fn;
    }

    // drivers/irqchip/irq-gic.c
    gic_of_init()       // gic 驱动入口函数
        gic_init_bases()
            #ifdef CONFIG_SMP
            set_smp_cross_call(gic_raise_softirq)
            register_cpu_notifier(&gic_cpu_notifier)
            #endif
            set_handle_irq(gic_handle_irq)
    ```

### 通信类型：
```c
// version: 4.1.15
// arch/arm/kernel/smp.c
enum ipi_msg_type {
    IPI_WAKEUP,
    IPI_TIMER,
    IPI_RESCHEDULE,
    IPI_CALL_FUNC,
    IPI_CALL_FUNC_SINGLE,
    IPI_CPU_STOP,
    IPI_IRQ_WORK,
    IPI_COMPLETION,
};
```

## SMP 的启动
### 启动 CPU 选择及参数定义
```c
// version: 4.1.15
// arch/arm/kernel/smp.c

/* as from 2.5, kernels no longer have an init_tasks structure 
 * so we need some other way of telling a new secondary core where
 * to place its SVC stack.
 */
struct secondary_data secondary_data;

/* control for which core is the next to come out of the 
 * secondary boot "holding pen"
 */
volatile int pen_release = -1;
```

## 负载均衡
[Linux 内核 SMP 负载均衡浅析](https://zhuanlan.zhihu.com/p/163482863)

## API
1. `smp_processor_id()`
   - 返回当前活动处理器的 id

## 参考
1. [Linux: SMP 架构下 CPU 核间通信](https://www.cnblogs.com/JiMoKuangXiangQu/articles/18812375)
2. 《Linux 设备驱动开发详解》
3. [多核 RTOS：AMP 与 SMP 架构在嵌入式中的选择——核间通信与负载均衡](https://blog.csdn.net/u014727709/article/details/162508769)
4. [CPU 多核通信在 Linux 上的实现](https://blog.csdn.net/suifengershi2000/article/details/129881704)
5. [Linux IPI 核间中断及亲和性](https://blog.csdn.net/wads23456/article/details/106948001)
6. [Linux 中断子系统之核间中断](https://blog.csdn.net/m0_51231887/article/details/130886291)
7. [处理器间中断（Inter-Processor Interrupt）](https://zhuanlan.zhihu.com/p/1985496563057967361)
8. [Linux 内核一文读懂 IPI 核间中断（arm64 架构）](https://www.cnblogs.com/linhaostudy/p/18805059)