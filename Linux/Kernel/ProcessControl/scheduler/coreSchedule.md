# 核心调度器
## 周期性调度器
周期性调度器在 `scheduler_tick()` 中实现。

如果系统正在活动中，内核会按照频率 `HZ` 自动调用该函数。如果没有进程在等待调度，那么从低功耗角度考虑，也可以关闭调度器以减少电能消耗。

`scheduler_tick()` 函数的两个主要任务：
1. 管理内核中与整个系统和各个进程的调度相关的统计量。其间执行的主要操作是对各种计数器加 1。
2. 激活负责当前进程的调度类的周期性调度方法。

`scheduler_tick()` 函数解析：
```c
// version: 4.1.15
// kernel/sched/core.c
/* 
 * This function gets called by the timer code, with HZ frequency. 
 * We call it with interrupts disabled.
 */
void scheduler_tick(void) {
    int cpu = smp_processor_id();
    struct rq *rq = cpu_rq(cpu);
    struct task_struct *curr = rq->curr;

    sched_clock_tick();

    raw_spin_lock(&rq->lock);
    update_rq_clock(rq);
    curr->sched_class->task_tick(rq, cuur, 0);
    update_cpu_load_active(rq);
    raw_spin_unlock(&rq->lock);

    perf_event_task_tick();

#ifdef CONFIG_SMP
    rq->idle_balance = idle_cpu(cpu);
    trigger_load_balance(rq);
#endif
    rq_last_tick_reset(rq);
}
```
- 获取当前进程：*CPU ID* -> *rq* -> *task_struct*
- `sched_clock_tick()`: 与配置选项： `CONFIG_HAVE_UNSTABLE_SCHED_CLOCK` 有关，当没有选择该选项时函数 `sched_clock_tick()` 为空
- 就绪队列的时钟的更新：`update_rq_clock()` (书中 `__update_rq_clock()`)，本质上就是增加 `struct rq` 当前实例的时钟时间戳。
- `update_cpu_load_active()` (书中 `update_cpu_load()`) 负责更新就绪队列的 `cpu_load[]` 数组。本质上相当于将数组中先前存储的负荷值向后移动一个位置，将当前就绪队列的负荷计入数组的第一个位置。另外，该函数还引入了一些取平均值的技巧，以确保负荷数组的内容不会呈现出太多的不连续跳变。同时它还会考虑 *nohz* 的情况，从而对这些负载信息进行调整。
- 主要工作委托给特定调度器类的方法：`curr->sched_class->task_tick(rq, curr, 0)`。
  - 完全公平调度器会在该方法中检测是否进程已经运行太长时间，以避免过长的延迟。这里的做法实际上不等价于到期的时间片，因为完全公平调度器中不再存在所谓时间片的概念。
  - 如果当前进程应该被重新调度，那么调度器类方法会在 *task_struct* 中设置 `TIF_NEED_RESCHED` 标志，以表示该请求，而内核会在接下来的适当时机完成该请求。
- 在 *SMP* 环境下，将触发负载均衡，该部分的笔记见 `Linux/Kernel/ProcessControl/reinforcement/smp.md`
- `rq_last_tick_reset()` 定义如下，需要选中配置选项 `CONFIG_NO_HZ_FULL`（关于时间管理的笔记见：`Linux/Kernel/Misc/timeManagement.md`）
    ```c
    // version: 4.1.15
    // kernel/sched/sched.h
    static inline void rq_last_tick_reset(struct rq *rq) {
    #ifdef CONFIG_NO_HZ_FULL
        rq->last_sched_tick = jiffies;
    #endif
    }
    ```
- 总结而言，`scheduler_tick()` 的功能就是：
  - 更新就绪队列的时钟信息
  - 调用当前运行进程对应调度类的 `task_tick()` 回调，看需不需对当前运行进程进行调度（设置 `TIF_NEED_RESCHED`）
  - 更新就绪队列的负载信息
  - 如果在 *SMP* 平台上，触发负载均衡
- `scheduler_tick()` 的调度链：
    ```c
    tick_handle_periodic(); // kernel/time/tick-common.c
                            // 它会被注册为某个时钟事件设备的处理函数，然后在其中断处理函数中被调用
        tick_periodic()
            update_process_times()
                scheduler_tick()
    
    // 另外
    // 在动态时钟时也会通过 tick_nohz_handler() 调用到该函数
    // 在高分辨率定时器实现周期时钟仿真时会通过 tick_sched_timer() 调用到该函数
    ```

1. `update_rq_clock()`:
    ```c
    // version: 4.1.15
    // kernel/sched/core.c
    void update_rq_clock(struct rq *rq) {
        s64 delta;
        // 断言：当前是在 rq->lock 已获取的环境下
        lockdep_assert_held(&rq->lock);
        // 检查是否要跳过 clock 的更新
        if (rq->clock_skip_update & RQCF_ACT_SKIP)
            return;
        // 获取上次 clock 更新以来的时间间隔，保存在 delta 中，单位为纳秒
        delta = sched_clock_cpu(cpu_of(rq)) - rq->clock;
        if (delta < 0)
            return;
        // 更新就绪队列上的 clock
        rq->clock += delta;
        update_rq_clock_task(rq, delta);
    }

    // kernel/sched/clock.c
    u64 sched_clock_cpu(int cpu) {
        if (unlikely(!sched_clock_running))
            return 0;
        return sched_clock();
    }

    // kernel/sched/core.c
    static void update_rq_clock_task(struct rq *rq, u64 delta) {
        ...
        // 非配置选项的代码只有：
        rq->clock_task += delta;
        ...
    }
    ```
    - 关于 `sched_clock_cpu()`:
      - 变量 `sched_clock_running`，根据变量名称推测，它表示 *sched clock* 是否在运行。它在函数 `sched_clock_init()` 中被赋值为 1，而该函数在 `start_kernel()` 中被调用
      - `sched_clock()`:
        - 它在 `kernel/sched/clock.c` 中被定义为一个弱符号（`__weak`），函数注释为：*Scheduler clock - returns current time in nanosec units. This is default implementation. Architectures and sub-architectures can override this.*
        - 而在 `kernel/time/sched_clock.c` 中重新进程了定义：
            ```c
            // version: 4.1.15
            // kernel/time/sched_clock.c
            unsigned long long notrace sched_clock(void) {
                ...
            }
            ```
    - 关于 `update_rq_clock_task()`，除了上述更新 `rq->clock_task`，它还根据如下配置选项更新 *rq* 的相关字段：
      - `CONFIG_IRQ_TIME_ACCOUNTING`: `init/Kconfig`, *Select this option to enable fine granularity task irq time accounting (精细粒度任务 IRQ 时间核算). This is done by reading a timestamp on each transitions between softirq and hardirq state, so there can be a small performace impact*，涉及的 *rq* 字段有 `rq->prev_irq_time`、`rq->rt_avg`、`rq->age_stamp`
      - `CONFIG_PARAVIRT_TIME_ACCOUNTING`: `arch/x86/Kconfig`, *Select this option to enable fine granularity task steal time accounting. Time spent executing other tasks in parallel with the current vCPU is discounted from the vCPU power. To account for that, there can be a small performace impact*，涉及的 *rq* 字段有 `rq->prev_steal_time_rq`、`rq->rt_avg`、`rq->age_stamp`

2. `update_cpu_load_active()`
    ```c
    // version: 4.1.15
    // kernel/sched/proc.c
    /* 
     * Called from scheduler_tick() 
     */
    void upate_cpu_load_active(struct rq *this_rq) {
        unsigned long load = get_rq_runnable_load(this_rq);
        /* 
         * See the mess around update_ilde_cpu_load() / update_cpu_load_nohz 
         */
        this_rq->last_load_update_tick = jiffies;
        __update_cpu_load(this_rq, load, 1);

        calc_load_account_active(this_rq);
    }
    ```
    - `kernel/sched/proc.c`: *Kernel load calculations, forked from sched/core.c*
      - *Global load-average calculations*
      - *Once every LOAD_FREQ*
      - ...
    - `get_rq_runnable_load()`:
        ```c
        // version: 4.1.15
        // kernel/sched/proc.c
        #ifdef CONFIG_SMP
        static inline unsigned long get_rq_runnable_load(struct rq *rq) {
            return rq->cfs.runnable_load_avg;
        }
        #else
        static inline unsigned long get_rq_runnable_load(struct rq *rq) {
            return rq->load.weight;
        }
        #endif
        ```
    - `__update_cpu_load()`:
        ```c
        // version: 4.1.15
        // kernel/sched/proc.c
        /* 
         * Update rq->cpu_load[] statistics. This function is usually called
         * every scheduler tick (TICK_NSEC). With tickless idle this will
         * not be called every tick. We fix it up based on jiffies.
         */
        static void __update_cpu_load(struct rq *this_rq, unsigned long this_load,
            unsigned long pending_updates) {
            int i, scale;

            this_rq->nr_load_updates++;
            /* update our load: */
            this_rq->cpu_load[0] = this_load;   /* Fasttrack for idx 0 */
            for (i = 1, scale = 2; i < CPU_LOAD_IDX_MAX; i++, scale += scale) {
                unsigned long old_load, new_load;
                /* scale is effectively 1 << i now, and >> i divides by scale */
                old_load = this_rq->cpu_load[i];
                old_load = decay_load_missed(old_load, pending_updates - 1, i);
                new_load = this_load;
                /* 
                    * Round up the averaging division if load is increasing. 
                    * This prevents us from getting stuck on 9 if the load
                    * is 10, for example.
                    */
                if (new_load > old_load)
                    new_lod += scale - 1;
                this_rq->cpu_load[i] = (old_load * (scale - 1) + new_load) >> i;
            }

            sched_avg_update(this_rq);
        }

        /* 
         * update cpu_load for any missed ticks, due to tickless idle.
         * The backlog would be when CPU is idle and so we just decay
         * the old load without adding any new load.
         */
        static unsigned long decay_load_missed(unsigned long load, unsigned long missed_updates, int idx) {
            int j = 0;
            if (!missed_updates)
                return load;
            if (missed_updates >= degrade_zero_ticks[idx])
                return 0;
            if (idx == 1)
                return load >> missed_updates;
            while (missed_updates) {
                if (missed_updates % 2)
                    load = (load * degrade_factor[idx][j]) >> DEGRADE_SHIFT;
                missed_updates >> 1;
                j++;
            }
            return load;
        }

        // kernel/sched/core.c
        void sched_avg_update(struct rq *rq) {
            s64  period = sched_avg_period();
            while ((u64)(rq_clock(rq) - rq->age_stamp) > period) {
                /* 
                 * Inline assembly required to prevent the compiler 
                 * optimising this loop into a divmod call.
                 * See __iter_div_u64_rem() for another example of this.
                 */
                asm("" : "+rm" (rg->age_stamp));
                rq->age_stamp += period;
                rq->rt_avg /= 2;
            }
        }

        // kernel/sched/sched.h
        static inline u64 sched_avg_period(void) {
            return (u64)sysctl_sched_time_avg * NSEC_PER_MSEC / 2;
        }
        ```
    - `calc_load_account_active()`
        ```c
        // version: 4.1.15
        // kernel/sched/proc.c
        /* 
         * Called from update_cpu_load() to periodically update this
         * CPU's active count.
         */
        static void calc_load_account_active(struct rq *this_rq) {
            long delta;
            if (time_before(jiffies, this_irq->calc_load_update))
                return;
            delat = calc_load_fold_active(this_rq);
            if (delta)
                atomic_long_add(delta, &calc_load_tasks);
            this_rq->calc_load_update += LOAD_FREQ;
        }
        
        long calc_load_fold_active(struct rq *this_rq) {
            long nr_active, delta = 0;

            nr_active = this_rq->nr_running;
            nr_active += (long) this_rq->nr_uninterruptible;

            if (nr_active != this_rq->calc_load_active) {
                delta = nr_active - this_rq->calc_load_active;
                this->calc_load_active = nr_active;
            }
            return delta;
        }

        // include/linux/sched.h
        #define LOAD_FREQ   (5 * HZ + 1)    // 5 sec intervals
        ```

## 主调度器
在内核中的许多地方，如果要将 *CPU* 分配给与当前活动进程不同的另一个进程，都会直接调用主调度器函数（`schedule()`）。

在从系统调用返回之后，内核也会检查当前进程是否设置了重调度器标志（`TIF_NEED_RESCHED`）。例如，`scheduler_tick()` 就会设置该标志。如果是这样，则内核会调用 `schedule()`。

`schedule()` 假定当前活动进程一定会被另一个进程取代。

`__sched` 前缀：
  - 用于可能调用 `schedule()` 的函数，包括 `schedule()` 本身；
  - 该前缀的目的在于，将相关函数的代码编译之后，放到目标文件的一个特定的段中，即 `.sched.text` 中。该信息使得内核在显示栈转储或类似信息时，忽略所有与调度有关的调用。由于调度器函数调用不是普通代码流程的一部分，因此在这种情况下是没有意义的。

`schedule()` 函数解析：
```c
// version: 4.1.15
// kernel/sched/core.c
asmlinkage __visible void __sched schedule(void) {
    struct task_struct *tsk = current;
    sched_submit_work();
    do {
        __schedule();
    } while (need_resched());
}
EXPORT_SYMBOL(schedule);

static inline void sched_submit_work(struct task_struct *tsk) {
    if (!tsk->state || tsk_is_pi_blocked(tsk))
        return;
    /* 
     * If we are going to sleep and we have plugged IO queued, 
     * make sure to submit it to avoid deadlocks.
     */
    if (blk_needs_flush_plug(tsk))
        blk_schedule_flush_plug(tsk);
}
```
- 首先，获取当前运行的进程
- 通过 `sched_submit_work()` 提交当前进程需要完成的向块设备输入数据的请求。在这之前需要进行检查确认，如果进程不在运行状态（`TASK_RUNNING`, `!tsk->state`）或者进程卡在一个使用了优先级继承机制的锁（通常是 *rt_mutex*，需要选中 `CONFIG_RT_MUTEXES` 配置项，未选择时函数 `tsk_is_pi_blocked()` 返回 `false`），则直接返回。
- 在 `__schedule()` 函数中会清除当前进程的 `TIF_NEED_RESCHED` 状态，然后再进行进程切换的工作（如果需要）。当当前进程再度恢复执行后，需要再次检查是否需要进行调度，这就是上述 `do { ... } while (need_resched())` 的由来。 
- 总的来说，`schedule()` 只是对 `__schedule()` 的简单包装。
- 注意：该函数默认认为当前进程需要调度，而不管当前进程是否设置了 `TIF_NEED_RESCHED`

函数 `__schedule()` 的分析：
```c
/*
* __schedule() is the main scheduler function.
*
* The main means of driving the scheduler and thus entering this function are:
*
*   1. Explicit blocking: mutex, semaphore, waitqueue, etc.
*
*   2. TIF_NEED_RESCHED flag is checked on interrupt and userspace return
*      paths. For example, see arch/x86/entry_64.S.
*
*      To drive preemption between tasks, the scheduler sets the flag in timer
*      interrupt handler scheduler_tick().
*
*   3. Wakeups don't really cause entry into schedule(). They add a
*      task to the run-queue and that's it.
*
*      Now, if the new task added to the run-queue preempts the current
*      task, then the wakeup sets TIF_NEED_RESCHED and schedule() gets
*      called on the nearest possible occasion:
*
*       - If the kernel is preemptible (CONFIG_PREEMPT=y):
*
*         - in syscall or exception context, at the next outmost
*           preempt_enable(). (this might be as soon as the wake_up()'s
*           spin_unlock()!)
*
*         - in IRQ context, return from interrupt-handler to
*           preemptible context
*
*       - If the kernel is not preemptible (CONFIG_PREEMPT is not set)
*         then at the next:
*
*          - cond_resched() call
*          - explicit schedule() call
*          - return from syscall or exception to user-space
*          - return from interrupt-handler to user-space
*
* WARNING: all callers must re-check need_resched() afterward and reschedule
* accordingly in case an event triggered the need for rescheduling (such as
* an interrupt waking up a task) while preemption was disabled in __schedule().
*/
static void __sched __schedule(void)
{
    struct task_struct *prev, *next;
    unsigned long *switch_count;
    struct rq *rq;
    int cpu;

    preempt_disable();
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
    rcu_note_context_switch();
    prev = rq->curr;

    schedule_debug(prev);

    if (sched_feat(HRTICK))
        hrtick_clear(rq);

    /*
    * Make sure that signal_pending_state()->signal_pending() below
    * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
    * done by the caller to avoid the race with signal_wake_up().
    */
    smp_mb__before_spinlock();
    raw_spin_lock_irq(&rq->lock);

    rq->clock_skip_update <<= 1; /* promote REQ to ACT */

    switch_count = &prev->nivcsw;
    if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {
        if (unlikely(signal_pending_state(prev->state, prev))) {
            prev->state = TASK_RUNNING;
        } else {
            deactivate_task(rq, prev, DEQUEUE_SLEEP);
            prev->on_rq = 0;

            /*
            * If a worker went to sleep, notify and ask workqueue
            * whether it wants to wake up a task to maintain
            * concurrency.
            */
            if (prev->flags & PF_WQ_WORKER) {
                struct task_struct *to_wakeup;

                to_wakeup = wq_worker_sleeping(prev, cpu);
                if (to_wakeup)
                    try_to_wake_up_local(to_wakeup);
            }
        }
        switch_count = &prev->nvcsw;
    }

    if (task_on_rq_queued(prev))
        update_rq_clock(rq);

    next = pick_next_task(rq, prev);
    clear_tsk_need_resched(prev);
    clear_preempt_need_resched();
    rq->clock_skip_update = 0;

    if (likely(prev != next)) {
        rq->nr_switches++;
        rq->curr = next;
        ++*switch_count;

        rq = context_switch(rq, prev, next); /* unlocks the rq */
        cpu = cpu_of(rq);
    } else
        raw_spin_unlock_irq(&rq->lock);

    post_schedule(rq);

    sched_preempt_enable_no_resched();
}
```
- 函数源码的注释中包含非常有用的信息：
  - `__schedule()` 是主调度器函数（理解：书中（《深入理解 Linux 内核架构》）提到的是 `schedule()` 是主调度器，推测是版本升级后将主调度器的功能迁移到了该函数，以方便其他调度情况下的复用）
  - 调度触发的主要方式有：
    - 显式地线程阻塞。如 *mutex, semaphore, waitqueue,* 等等
    - 当在中断和用户空间的返回路径上检测到 `TIF_NEED_RESCHED` 置位时。在调度器 `scheduler_tick()` 中可能会置位进程的 `TIF_NEED_RESCHED`
    - **唤醒进程不会触发 `schedule()`，它只是将进程添加到对应的就绪队列（*run-queue*）中**。如果新创建的进程被添加到就绪队列上时，可以抢占当前运行的进程，那么唤醒代码将会设置进程的 `TIF_NEED_RESCHED` 并且 `schedule()` 将会在最近的如下情况中被调用：
      - 如果内核可以抢占：
        - 在系统调用或者异常处理上下文中，在下一次最外层的 `preempt_enable()` 被调用时（这可能只需要唤醒代码调用了 `spin_unlock()`）
        - 在中断上下文中，从中断处理函数返回到可抢占的上下文中
      - 如果内核不可以抢占：
        - `cond_resched()` 调用时
        - 显式调用 `schedule()`
        - 从系统调用或者异常返回到用户空间
        - 从中断处理函数返回到用户空间
  - 注意：所有 `__schedule()` 函数的调用者必须在该函数调用结束后重新检查 `need_resched()` 并且相应地重新调度，防止由于抢占在 `__schedule()` 中被 *disable* 但某些事件触发了重调度的情况（例如某个中断唤醒了一个进程）。
- 该函数在抢占 *disable* 的环境中执行，因此首先调用了 `preempt_disable()`
- 随后获取当前的进程信息：*CPU* -> 就绪队列 -> 当前进程
- `sched_feat(HRTICK)` 用于判断内核是否启用了高分辨率调度时钟节拍（*high resolution tick*）特性，如果启用了该特性，则通过 `hrtick_clear()` 将就绪队列上对应的高精度定时器取消（如果它已经被激活）
- `smp_mb__before_spinlock()`，内存屏障指令，保证在该指令前的所有读写操作都已完成（理解：更新当前进程的上下文信息？）
- 如果当前进程没有处于可执行状态（`TASK_RUNNING`）并且当前上下文也不是抢占引起的上下文，则：
  - 如果当前进程还有待处理的信号，则将当前进程的状态设置为 `TASK_RUNNING`
  - 否则，将当前进程从其就绪队列上拿掉，`deactive_task()` 会进一步调用当前进程的调度类对应的 `dequeue_task()` 回调。如果当前进程还是一个工作队列进程，则尝试唤醒其他本地 *CPU* 的工作队列进程
- 如果当前线程还在就绪队列上，则通过 `update_rq_clock()` 更新对应就绪队列的时钟信息
- 选择下一进程，`pick_next_task()` 会进一步调用某个调度类的 `pick_next_task()` 的回调，前者会先考虑 *CFS*，但是如果有更高优先级的进程等待调度，则会通过 `for_each_class()` 遍历挂在调度类链表上的所有实例，顺序按照优先级从高到低
- 清除当前进程的 `TIF_NEED_RESCHED` 标志、设置 `rq->clock_skip_update` 为 0（不要跳过下一次 *rq* 上的时钟更新）
- 如果需要调度（当前进程和 `pick_next_task()` 选择的下一个进程不是同一个），则更新 *rq* 和当前进程的上下文切换次数、重新设置 *rq* 的 `curr` 指针、进行上下文切换（`context_switch()`）。
- 当前上下文切换完成后（上下文切换完成后执行流已不在当前进程），在当前进程恢复执行时，通过 `post_schedule()` 执行某些调度类的“后调度”处理工作，`rq->post_schedule` 字段记录了完成上下文切换后是否还需要进行此处理。如果需要，则会试图调用 *rq* 的当前进程对应调度类的 `post_schedule()` 回调。
- 最后通过 `sched_preempt_enable_no_resched()` 重新使能抢占，并通过 `barrier()` 防止编译器重排序优化，从软件角度保证在这之前的读写操作先完成。

`__schedule()` 的调用链：
1. `schedule()`: 最常见的触发调度的接口
    ```c
    // version: 4.1.15
    // kernel/sched/core.c
    schedule()
        __schedule()
    ```

2. `preempt_schedule_common()`: 标记当前为抢占触发的调度，然后进行调度，恢复后清除抢占调度标记
    ```c
    // version: 4.1.15
    // kernel/sched/core.c
    preempt_schedule_common()
        __schedule()
    ```

3. `preempt_schedule_irq()`: 与 `preempt_schedule_common()` 类似，只是该函数在执行调度前会先将本地中断使能，恢复后先将本地中断禁止。另外，它只能在禁止抢占或者中断禁止的情况下才能使用（意味着调用该函数的上下文环境为禁止抢占或中断）。
    ```c
    // version: 4.1.15
    // kernel/sched/core.c
    preempt_schedule_irq()
        __schedule()
    ```

## 与 fork 的交互
每当使用 `fork()` 系统调用或其变体之一建立新进程时，调度器有机会用 `sched_fork()` 函数挂钩到该进程：
```c
do_fork()
    copy_process()
        // Perform scheduler related setup. Assign this task to a CPU.
        sched_fork()
```

在单处理其系统上，该函数实质上执行 3 个操作：初始化新进程与调度相关的字段、建立数据结构、确定进程的动态优先级：
```c
// version: 4.1.15
// kernel/sched/core.c
/* 
 * fork()/clone()-time setup:
 */
int sched_fork(unsigned long clone_flags, struct task_struct *p) {
    unsigned long flags;
    int cpu = get_cpu();

    __sched_fork(clone_flags, p);
    /* 
     * We mark the process as running here. This guarantees that
     * nobody will actually run it, and a signal or other external
     * event cannot wake it up and insert it on the runqueue either.
     */
    p->state = TASK_RUNNING;

    /* Make sure we do not leak PI boosting priority to the child. */
    p->prio = current->normal_prio;

    /* Revert (还原、恢复) to deefault priority/policy on fork if requested. */
    if (unlikely(p->sched_reset_on_fork)) {
        if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
            p->policy = SCHED_NORMAL;
            p->static_prio = NICE_TO_PRIO(0);
            p->rt_priority = 0;
        } else if (PRIO_TO_NICE(p->static_prio) < 0)
            p->static_prio = NICE_TO_PRIO(0);
        
        p->prio = p->normal_prio = __normal_prio(p);
        set_load_weight(p);

        /* 
         * We don't need the reset flag anymore after the fork. 
         * It has fulfilled (完成、履行) its duty.
         */
        p->sched_reset_on_fork = 0;
    }

    if (dl_prio(p->prio)) {
        put_cpu();
        return -EAGAIN;
    } else if (rt_prio(p->prio)) {
        p->sched_class = &rt_sched_class;
    } else {
        p->sched_class = &fair_sched_class;
    }

    if (p->sched_class->task_fork)
        p->sched_class->task_fork(p);

    /* 
     * The child is not yet in the pid-hash so no cgroup attach 
     * races, and the cgroup is pinned to this child due to cgoup_fork()
     * is ran before sched_fork().
     *
     * Silence PROVE_RCU
     */
    raw_spin_lock_irqsave(&p->pi_lock, flags);
    set_task_cpu(p, cpu);
    raw_spin_lock_irqrestore(&p->pi_lock, flags);

#if defined(CONFIG_SCHEDSTATS) || defined(CONFIG_TASK_DELAY_ACCT)
    if (likely(sched_info_on()))
        memset(&p->sched_info, 0, sizeof(p->sched_info));
#endif
#if defined(CONFIG_SMP)
    p->on_cpu = 0;
#endif
    init_task_preempt_count(p);
#ifdef CONFIG_SMP
    plist_node_init(&p->pushable_tasks, MAX_PRIO);
    RB_CLEAR_NODE(&p->pushable_dl_tasks);
#endif
    put_cpu();
    return 0;
}
```
- `__schedule_fork()`: 执行一些初始化设置，主要包括 `p->se` 的相关字段和 `p->on_cpu` 等
- 设置进程的状态 `task_struct->state` 为 `TASK_RUNNING`，保证此进程此时不会运行、信号或者其他外部事件不能唤醒它、也不会把它加入到就绪队列上
- 通过使用父进程的 `normal_prio` 作为子进程的动态优先级（`prio`），内核确保父进程优先级的临时提高不会被子进程继承。
- 如果进程要求重新设置调度（`task_struct->sched_reset_on_fork` 被设置），则将其恢复到默认优先级、调度策略、权重信息：调度策略 `policy` 为 `SCHED_NORMAL`、静态优先级 `static_prio` 为 `NICE_TO_PRIO(0)`、实时优先级 `rt_priority` 为 0，其他两个优先级根据 `__normal_prio()` 更新，同时通过 `set_load_weight()` 设置进程的权重信息
- 由于 *Linux* 内核在设计上明确禁止了 `SCHED_DEADLINE (DL)` 任务通过 `fork()` 或 `pthread_create()` 创建子任务，因此在这里如果进程的动态优先级位于 *dl* 的优先级范围内，则返回错误
- 进程的调度类根据其动态优先级（要么继承自父进程的 `normal_prio`，要么为默认值）设置
- 在设置调度类之后，调用调度类的 `task_fork()` 回调，将进程加入到就绪队列中
- 设置进程的 *CPU*，即进程对应 *thread info* 的 `cpu` 字段。`set_task_cpu()` 会检查参数 `cpu` 和参数进程对应的 *cpu* 是不是同一个，如果不是，则会触发进程的迁移，将进程迁移到参数 `cpu` 上，这是通过进程对应调度类的 `migrate_task_rq()` 完成的
- 设置 `task_struct->on_cpu` 为 0。`on_cpu` 字段是一个快速判断标志，用于在不获取运行队列锁的情况下，了解任务是否正在 *CPU* 上执行。对于刚通过 `fork()` 产生的子进程设置该字段为 0，明确标记该进程当前不在任何 *CPU* 上运行。另外，进程投入运行时，在上下文切换准备阶段（`prepare_task_switch()`），内核会将下一个要运行的进程的 `on_cpu` 设置为 1；进程让出 *CPU* 后，在切换完成后（`finish_task_switch()`），前一个进程的 `on_cpu` 会被清零
- 书：“在使用 `wake_up_new_task()` 唤醒新进程时，则是调度器与进程创建逻辑交互的第二个时机：内核会调用调度类的 `task_new()`。这提供了一个时机，将新进程加入到相应类的就绪队列中”。在 *version=4.1.15* 的内核源码中，`wake_up_new_task()-kernel/sched/core.c` 在 `do_fork()-kernel/fork.c` 中被调用。在 *SMP* 环境下，`wake_up_new_task()` 会调用 `task_struct->sched_class->task_woken()` 回调。

1. `__sched_fork()`
    ```c
    // version: 4.1.15
    // kernel/sched/core.c
    /* Perform scheduler related setup for a newly forked process p. 
     * p is forked by current.
     * __sched_fork() is basic setup used by init_idle() too.
     */
    static void __sched_fork(unsigned long clone_flags, struct task_struct *p) {
        p->on_rq = 0;

        p->se.on_rq = 0;
        p->se.exec_start = 0;
        p->se.sum_exec_runtime = 0;
        p->se.prev_sum_exec_runtime = 0;
        p->se.nr_migrations = 0;
        p->se.vruntime = 0;
    #ifdef CONFIG_SMP
        p->se.avg.decay_count = 0;
    #endif
        INIT_LIST_HEAD(&p->se.group_node);
    
    #ifdef CONFIG_SCHEDSTATS
        memeset(&p->se.statistics, 0, sizeof(p->se.statistics));
    #endif
        RB_CLEAR_NODE(&p->dl.rb_node);
        init_dl_task_timer(&p->dl);
        __dl_clear_params(p);

        INIT_LIST_HEAD(&p->rt.run_list);
    #ifdef CONFIG_PREEMPT_NOTIFIES
        INIT_HLIST_HEAD(&p->preempt_notifies);
    #endif
    #ifdef CONFIG_NUMA_BALANCING
        ...
    #endif
    }
    ```

## 上下文切换
内核选择新进程之后，必须处理与多任务相关的技术细节。这些细节总称为上下文切换（*context switching*）。辅助函数 `context_switch()` 是个分配器，它会调用所需的特定于体系结构的方法。
```c
// version: 4.1.15
// kernel/sched/core.c
/* 
 * context_switch - switch to the new MM and the new thread's 
 * register state. 
 */
static inline struct rq *context_switch(struct rq *rq, struct task_struct *prev,
    struct task_struct *next) {
    struct mm_struct *mm, *oldmm;

    prepare_task_switch(rq, prev, next);

    mm = next->mm;
    oldmm = prev->active_mm;
    /* 
     * For paravirt (虚拟化), this is coupled with an exit in switch_to
     * to combine the page table reload and the switch backend into
     * one hypercall.
     */ 
    arch_start_context_switch(prev);

    if (!mm) {
        next->active_mm = oldmm;
        atomic_inc(&oldmm->mm_count);
        enter_lazy_tlb(oldmm, next);
    } else
        switch_mm(oldmm, mm, next);
    
    if (!prev->mm) {
        prev->active_mm = NULL;
        rq->prev_mm = oldmm;
    }
    /* 
     * Since the runqueue lock will be released by the next 
     * task (which is an invalid locking op but in the case
     * of the scheduler it's an obvious special-case), so we
     * do an early lockdep release here:
     */
    spin_release(&rq->lock.dep_map, 1, _THIS_IP_);

    context_tracking_task_switch(prev, next);
    /* Here we just switch the register state and the stack */
    switch_to(prev, next, prev);
    barrier();

    return finish_task_switch(prev);
}
```
- 在进程切换之前，`prepare_task_switch()` 被调用，为切换做事先准备。
- 上下文切换本身通过调用两个特定于处理器的函数完成：
  - `switch_mm()`：更换通过 `task_struct->mm` 描述的内存管理上下文。该工作的细节取决于处理器，主要包括加载页表、刷出地址转换后备缓冲器（部分或全部）、向内存管理单元（*MMU*）提供新的信息。
  - `switch_to()`：切换处理器寄存器内容和内核栈（虚拟地址空间的用户部分在第一步已经变更，其中也包括了用户状态下的栈，因此用户栈就不需要显式变更了）。

1. `prepare_task_switch()`
    ```c
    // version: 4.1.15
    // kernel/sched/core.c
    /**
     * prepare_task_switch - prepare to switch tasks 
     * @rq: the runqueue preparing to switch
     * @prev: the current task that is being switched out
     * @next: the task we are going to switch to
     *
     * This is called with the rq lock held and interrupts off.
     * It must be paired with a subsequent finish_task_switch after
     * the context switch.
     *
     * prepare_task_switch sets up locking and calls architecture
     * specific hooks.
     */
    static inline void prepare_tasks_switch(struct rq *rq, struct task_struct *prev, 
        struct task_struct *next) {
        trace_sched_switch(prev, next);
        sched_info_switch(rq, prev, next);
        perf_event_task_sched_out(prev, next);
        fire_sched_out_preempt_notifiers(prev, next);
        prepare_lock_switch(rq, next);
        prepare_arch_switch(next);
    }
    ```
    - 在 *SMP* 环境下，`prepare_lock_switch()` 会设置 *next* 进程的 `on_cpu` 字段为 1，表示它即将切换到 *CPU* 上运行
    - 在 *i.mx6ull* 的配置中，`prepare_arch_switch()` 定义为空

2. `arch_start_context_switch()`: 在 *i.mx6ull* 的配置中，该函数定义为空。

3. 判断下一个进程的 `mm`（进程地址空间）是否为空，根据《深入 *Linux* 内核架构》的介绍，通过该判断来检查下一个进程是否为内核进程，而且内核线程后续需要将自身的 `task_struct->active_mm` 指向用户空间的内存管理数据结构。如果下一个进程是内核进程，则触发 *MMU* 的 *lazy tlb* 模式（惰性 *tlb* 处理：如果内核线程之后运行的进程与之前是同一个，在这种情况下，内核并不需要修改用户空间的地址表，地址转换后备缓冲器（即 *TLB*）中的信息仍然有效。只有在内核线程之后执行的进程是与此前不同的用户层进程时，才需要切换（并清除对应 *TLB* 数据））。`switch_mm()` 在 *ARM* 架构下，更新 *MMU* 的页表基地址等信息。由于这里的 `entry_lazy_tlb()` 和 `switch_mm()` 是和架构相关的代码，具体信息见笔记：`Linux/Arch/arm.md`。

4. 如果当前进程也是内核线程，则将其 `task_struct->active_mm` 重新置为 `NULL`，同时通过就绪队列 `rq->prev_mm` 保存这部分信息

5. `switch_to()`: 该函数执行完成后切换到下一个进程运行，同时它将当前进程在 `context_switch()` 内的局部变量 `prev` 重新设置，让其指向将当前进程换下去的那个进程。具体信息见笔记：`Linux/Arch/arm.md`

6. `barrier()`：避免编译器重排序优化

7. `finish_task_switch()`: 当上下文切换完成后，当前进程便失去了对 *CPU* 的使用权。当其再次恢复运行时，需要完成切换工作后的收尾处理（注意，是新进程在替旧进程做清理。这里是指切换到当前进程时的收尾工作，而不是上一次当前进程切换出去后，因为切换出去后 *CPU* 的任务执行流程就不在当前进程上了，这里的 `prev` 指向上一次将当前进程换下去的那个进程）。
    ```c
    // version: 4.1.15
    // kernel/sched/core.c
    /** 
     * finish_task_switch - clean up after a task-switch 
     * @prev: the thread we just switched away from.
     * 
     * finished_task_switch must be called after the context switch, paired with a
     * prepare_task_switch call before the context switch.
     * finish_task_switch will reconcile (协调，配合) locking set up by prepare_task_switch,
     * and do any other architecuture-specific clean actions.
     *
     * Note that we may have delayed dropping an mm in context_switch(). If so, we
     * finish that here outside of the runqueue lock. (Doing it with the lock held
     * can cause deadlocks; see schedule() for details.)
     *
     * The context switch have flipped the stack from under us and restored the local 
     * variables which were saved when this task called schedule() in the past. prev == current
     * is still correct but we need to recalculate this_rq because prev may have moved
     * to another CPU.
     */
    static struct rq* finish_task_switch(struct task_struct *prev) 
        __release(rq->lock)
    {
        struct rq *rq = this_rq();
        struct mm_struct *mm = rq->prev_mm;
        long prev_state;

        rq->prev_mm = NULL;

        /* 
         * A task struct has one reference for the use as "current".
         * If a task dies, then it sets TASK_DEAD in tsk->state and calls
         * schedule one last time. The schedule call will never return, and the scheduled
         * task must drop the that reference.
         * 
         * We must observe prev->state before clearing prev->on_cpu (in finish_lock_switch),
         * otherwise a concurrent wakeup can get prev running on another CPU and
         * we could rave with its RUNNING -> DEAD transition, resulting in a double group.
         */
        prev_state = prev->state;
        vtime_task_switch(prev);
        finish_arch_switch(prev);
        perf_event_task_sched_in(prev, current);
        finish_lock_switch(rq, prev);
        finish_arch_post_lock_switch();

        fire_sched_in_preempt_notifiers(current);
        if (mm)
            mmdrop(mm);
        if (unlikely(prev_state == TASK_DEAD)) {
            if (prev->sched_class->task_dead)
                prev->sched_class->task_dead(prev);
            
            /* 
             * Remove function-return probe instances associated with this task 
             * and put them back on the free list.
             */
            kprobe_flush_task(prev);
            put_task_struct(prev);
        }

        tick_nohz_task_switch(current);
        return rq;
    }
    ```
    - 前一个进程的状态可能是 `task_struct->state = TASK_DEAD`，它在状态更改之后调用 `schedule()` 从而切换到当前进程，因此在这里首先获取了前一个进程的状态值
    - `vtime_task_switch()` 和虚拟 *CPU* 有关
    - `finish_arch_switch()` 与体系构架有关，笔记内容见：`Linux/Arch/arm.md`
    - `finish_lock_switch()` 将 `prev->on_cpu` 设置为 0
    - `finish_arch_post_lock_switch()`: 被定义为空
    - `fire_sched_out_preempt_notifiers()`: 需要选择配置项 `CONFIG_PREEMPT_NOTIFIERS`，未选择时该函数为空
    - `mmdrop()` 的作用是释放对 *mm_struct* 的一次引用。当引用计数归零时，它会负责销毁该地址空间及其相关的页表
    - 如果 `prev` 已经是被杀死的进程，则会尝试调用其对应调度类的 `task_dead()` 回调，并通过 `put_task_struct()` 减少其 *task_struct* 的引用计数，当计数归零时，释放对应资源（同上）
    - `task_nohz_task_switch()`: 需要选择配置项 `CONFIG_NO_HZ_FULL`，如果使能了相关内容，该函数试图重启 *tick*。