# Note for SMP-SCHED
## 负载均衡
### 调度时机
   - `scheduler_tick()`:
        ```c
        // version: 4.1.15
        // kernel/sched/core.c
        void scheduler_tick() {
            ...
        #ifdef CONFIG_SMP
            rq->idle_balance = idle_cpu(cpu);
            trigger_load_balance(rq);
        #endif
            ...
        }
        ```
        - `idle_cpu()`: 判断给定的 *CPU* 当前是否是 *idle* 状态
        - `trigger_load_balance()`: 
            ```c
            // version: 4.1.15
            // kernel/sched/fair.c
            /* 
             * Trigger the SCHED_SOFTIRQ if it is time to do periodic load balancing
             */
            void trigger_load_balance(struct rq *rq) {
                /* Don't need to rebalance while attached to NULL domain */
                if (unlikely(on_null_domain(rq)))
                    return;
                if (time_after_eq(jiffies, rq->next_balance))
                    raise_softirq(SCHED_SOFTIRQ);
            #ifdef CONFIG_NO_HZ_COMMON
                if (nohz_kick_needed(rq))
                    no_balancer_kick()
            #endif
            }
            ```
### 函数入口
负载均衡的任务由 `SCHED_SOFTIRQ` 的中断下半部函数 `run_rebalance_domains()` 执行。
1. 函数注册：
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    __init void init_sched_fair_class(void) {
    #ifdef CONFIG_SMP
        open_softirq(SCHED_SOFTIRQ, run_rebalance_domains);
    #ifdef CONFIG_NO_HZ_COMMON
        ...
    #endif
    #endif
    }
    ```

2. 函数实现
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /* 
     * run_rebalance_domains is triggered when needed from the scheduler tick.
     * Also triggered for nohz idle balancing (with nohz_balancing_kick set).
     */
    static void run_rebalance_domains(struct softirq_action *h) {
        struct rq *this_rq = this_rq();
        enum cpu_idle_tyep idle = this_rq->idle_balance ? CPU_IDLE : CPU_NOT_IDLE;

        /* 
         * If this cpu has a pending nohz_balance_kick, then do the 
         * balancing on behalf of the other idle cpus whose ticks are
         * stopped. Do nohz_idle_balance *before* rebalance_domains to
         * give the idle cpus a chance to load balance. Else we may
         * load balance only within the local sched_domain hierarchy
         * and abort nohz_idle_balance altogether if we pull some load.
         */
        nohz_idle_balance(this_rq, idle);
        rebalance_domains(this_rq, idle);
    }
    ```
    - `nohz_idle_balance()`: 见 “*NO_HZ* 机制”部分
    - `rebalance_domains()`:
        ```c
        ...
        ```

## NO_HZ 机制
---
如下内容来自对 *AI* 问答的整理，还需进一步确认和完善：

*NO_HZ*: 即 *no tick* 或 *tickless*，表示无时钟 *tick*。
*Linux* 内核中，*CPU* 的时钟中断是驱动调度的基本机制。每个 *CPU* 以固定的频率（如 *100HZ, 250HZ, 1000HZ*）产生时钟中断（*timer tick*），每次 *tick* 中执行触发调度决策、更新进程时间片、更新 *cgroup* 统计、*softirq* 触发等。*NO_HZ* 机制的核心思想是在不需要 *tick* 的时候，停掉该 *CPU* 的时钟中断，以节省功耗或减少抖动。

1. *tick* 运行的几种状态及转移转系：
   - 当 *CPU* 上有 *runable* 任务时，*tick* 正常运行；
   - *CPU* 进入 *idle* (无任务可运行) 时停掉 *tick* (`tick_nohz_idle_enter()`)；
   - 有任务唤醒该 *CPU*，重新启用 *tick*。

2. 停掉 *tick* 带来的问题：该 *CPU* 无法通过正常的 *tick* 路径触发 `SCHED_SOFTIRQ`，进而无法执行 `run_rebalance_domains()`，导致负载均衡被冻结。

3. 问题解决方式：其他 *CPU* 唤醒，该任务通过 `nohz_idle_balance()` 完成。该函数的作用是代替那些已经停掉 *tick* 的 *idle CPU* 执行跨调度域的负载均衡（从忙碌的 *CPU* 上拉取任务到 *idle CPU*）。

4. *NO_HZ* 的演进
   - 早期（`~2.6.21`）
     - 当 *CPU* 进入 *idle*（所有 *runable* 进程都等待 *I/O* 或者无任务可运行）时，停止该 *CPU* 的时钟 *tick*；
     - 目的：减少 *idle CPU* 的功耗（避免无意义的 *tick* 唤醒）
     - 当有任务唤醒时，重新启用 *tick*
     - 这是“浅”的 *tickless*
   - 后期（*NOHZ_FULL/TICKLESS*, `~4.8+`, 对应 `CONFIG_NO_HZ_FULL`）
     - 不仅 *idle* 时停止 *tick*，即使 *CPU* 上只有一个任务运行时也停止 *tick*
     - 目的：减少实时/延迟敏感任务的抖动（*jitter*）（关于 *jitter* 的信息，在内核文档 `Documentation/kernel-per-cpu-kthreads.txt` 中相关的内容）
     - 需要用户态显式将 *CPU* 标记为“*isolated*”（如 `nohz_full = 内核参数`）
     - 这里是“深”的 *tickless*

5. `nohz_idle_balance()` 在 `run_rebalance_domains()` 中的含义：
   - 处理的是 *NOHZ_IDLE* 场景
   - 当 *CPU* 进入 *idle* 后，它的 *tick* 停了，所以它无法通过正常的 *tick* 路径触发 `SCHED_SOFTIRQ`
   - 但其他 *CPU* 的 *tick* 仍然在运行，它们需要“代替”那些 *idle* (*tick stopped*) 的 *CPU* 做负载均衡
   - `nohz_balance_tick()`: 当一个 *CPU* 即将进入 *idle*（停止 *tick*）时，它会 *kick* 另一个非 *idle CPU*，让那个 *CPU* 在下次 *tick* 时帮忙做负载均衡
   - 这就是 `run_rebalance_domains()` 中注释说的 *“do the balancing on behave of the other idle cpus whose ticks are stopped”*。

6. 具体机制（执行流程）：
   1. *CPU A* 即将 *idle* -> 调用 `nohz_balance_enter_idle()`
   2. `nohz_balance_enter_idle()` 函数选择一个“受害者” *CPU B*（非 *idle* 的）
   3. 向 *CPU B* 发送 *IPI*，设置 `nohz_balance_tick` 标志
   4. *CPU B* 下次 *tick* 中，`run_rebalance_domains()` 被触发
   5. 在 `run_rebalance_domains()` 中，线调用 `nohz_idle_balance(this_rq, idle)`，代替那些 *idle CPU* 做负载均衡
   6. 然后再做 `rebalance_domains(this_rq, idle)` 做本地域的负载均衡

7. 为什么 `nohz_idle_balance()` 要在 `rebalance_domains()` 之前：注释中解释了：先让 *idle CPU* 有机会做负载均衡（从其他 *CPU* 拉取任务），如果本地 `rebalance_domains()` 先执行从而改变了负载分布，可能导致后续 `nohz_idle_balance()` 判断“本地已均衡”而直接返回，放弃了跨域拉取的机会。

8. 总结：*NO_HZ* 指 *CPU* 因 *idle* 而停掉了时钟 *tick* 的状态。由于 *tick* 停了，该 *CPU* 无法自行触发负载均衡，所以内核通过 `nohz_balance_kick` 让一个仍在跑的 *CPU* 代劳。在 `run_rebalance_domains()` 中先替 *idle CPU* 做跨域均衡（`nohz_idle_balance()`）再做本地均衡（`rebalance_domains()`）

---