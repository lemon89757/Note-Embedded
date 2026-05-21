# 进程调度
## preempt 概述

## preempt_count
### `struct thread_info` 中的 preempt_count
```c
//version-4.1.15
// arch/arm/include/asm/thread_info.h
static inline struct thread_info *current_thread_info(void) __attribute_const__;

static inline struct thread_info *current_thread_info(void)
{
    return (struct thread_info *)
        (current_stack_pointer & ~(THREAD_SIZE - 1));
}

register unsigned long current_stack_pointer asm ("sp");

#define THREAD_SIZE_ORDER	1
#define THREAD_SIZE		(PAGE_SIZE << THREAD_SIZE_ORDER)

/*
* low level task data that entry.S needs immediate access to.
* __switch_to() assumes cpu_context follows immediately after cpu_domain.
*/
struct thread_info {
    unsigned long		flags;		/* low level flags */
    int			preempt_count;	/* 0 => preemptable, <0 => bug */
    mm_segment_t		addr_limit;	/* address limit */
    struct task_struct	*task;		/* main task structure */
    __u32			cpu;		/* cpu */
    __u32			cpu_domain;	/* cpu domain */
    struct cpu_context_save	cpu_context;	/* cpu context */
    __u32			syscall;	/* syscall number */
    __u8			used_cp[16];	/* thread used copro */
    unsigned long		tp_value[2];	/* TLS registers */
#ifdef CONFIG_CRUNCH
    struct crunch_state	crunchstate;
#endif
    union fp_state		fpstate __attribute__((aligned(8)));
    union vfp_state		vfpstate;
#ifdef CONFIG_ARM_THUMBEE
    unsigned long		thumbee_state;	/* ThumbEE Handler Base register */
#endif
};
```
- 通过 `thread_info` 结构体定义处的注释：“0 => preemptable, <0 => bug” 可知，只有当 preempt_count 等于 0 时才是可抢占的。
- 通过相关源码可知，当 preempt_count 大于 0 时，其对应的 current_thread_info 是无法被抢占的（见后续分析）。

### preempt_count 的位域布局和状态判断
```c
//version-4.1.15
//include/linux/preempt_mask.h
/*
* We put the hardirq and softirq counter into the preemption
* counter. The bitmask has the following meaning:
*
* - bits 0-7 are the preemption count (max preemption depth: 256)
* - bits 8-15 are the softirq count (max # of softirqs: 256)
*
* The hardirq count could in theory be the same as the number of
* interrupts in the system, but we run all interrupt handlers with
* interrupts disabled, so we cannot have nesting interrupts. Though
* there are a few palaeontologic drivers which reenable interrupts in
* the handler, so we need more than one bit here.
*
* PREEMPT_MASK:	0x000000ff
* SOFTIRQ_MASK:	0x0000ff00
* HARDIRQ_MASK:	0x000f0000
*     NMI_MASK:	0x00100000
* PREEMPT_ACTIVE:	0x00200000
*/
#define PREEMPT_BITS	8
#define SOFTIRQ_BITS	8
#define HARDIRQ_BITS	4
#define NMI_BITS	1

#define PREEMPT_SHIFT	0
#define SOFTIRQ_SHIFT	(PREEMPT_SHIFT + PREEMPT_BITS)
#define HARDIRQ_SHIFT	(SOFTIRQ_SHIFT + SOFTIRQ_BITS)
#define NMI_SHIFT	(HARDIRQ_SHIFT + HARDIRQ_BITS)

#define __IRQ_MASK(x)	((1UL << (x))-1)

#define PREEMPT_MASK	(__IRQ_MASK(PREEMPT_BITS) << PREEMPT_SHIFT)
#define SOFTIRQ_MASK	(__IRQ_MASK(SOFTIRQ_BITS) << SOFTIRQ_SHIFT)
#define HARDIRQ_MASK	(__IRQ_MASK(HARDIRQ_BITS) << HARDIRQ_SHIFT)
#define NMI_MASK	(__IRQ_MASK(NMI_BITS)     << NMI_SHIFT)

#define PREEMPT_OFFSET	(1UL << PREEMPT_SHIFT)
#define SOFTIRQ_OFFSET	(1UL << SOFTIRQ_SHIFT)
#define HARDIRQ_OFFSET	(1UL << HARDIRQ_SHIFT)
#define NMI_OFFSET	(1UL << NMI_SHIFT)

#define SOFTIRQ_DISABLE_OFFSET	(2 * SOFTIRQ_OFFSET)

#define PREEMPT_ACTIVE_BITS	1
#define PREEMPT_ACTIVE_SHIFT	(NMI_SHIFT + NMI_BITS)
#define PREEMPT_ACTIVE	(__IRQ_MASK(PREEMPT_ACTIVE_BITS) << PREEMPT_ACTIVE_SHIFT)

#define hardirq_count()	(preempt_count() & HARDIRQ_MASK)
#define softirq_count()	(preempt_count() & SOFTIRQ_MASK)
#define irq_count()	(preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK \
                | NMI_MASK))

/*
* Are we doing bottom half or hardware interrupt processing?
* Are we in a softirq context? Interrupt context?
* in_softirq - Are we currently processing softirq or have bh disabled?
* in_serving_softirq - Are we currently processing softirq?
*/
#define in_irq()		(hardirq_count())
#define in_softirq()		(softirq_count())
#define in_interrupt()		(irq_count())
#define in_serving_softirq()	(softirq_count() & SOFTIRQ_OFFSET)

/*
* Are we in NMI context?
*/
#define in_nmi()	(preempt_count() & NMI_MASK)

/*
* The preempt_count offset after preempt_disable();
*/
#if defined(CONFIG_PREEMPT_COUNT)
# define PREEMPT_DISABLE_OFFSET	PREEMPT_OFFSET
#else
# define PREEMPT_DISABLE_OFFSET	0
#endif

/*
* The preempt_count offset after spin_lock()
*/
#define PREEMPT_LOCK_OFFSET	PREEMPT_DISABLE_OFFSET

/*
* The preempt_count offset needed for things like:
*
*  spin_lock_bh()
*
* Which need to disable both preemption (CONFIG_PREEMPT_COUNT) and
* softirqs, such that unlock sequences of:
*
*  spin_unlock();
*  local_bh_enable();
*
* Work as expected.
*/
#define SOFTIRQ_LOCK_OFFSET (SOFTIRQ_DISABLE_OFFSET + PREEMPT_LOCK_OFFSET)

/*
* Are we running in atomic context?  WARNING: this macro cannot
* always detect atomic context; in particular, it cannot know about
* held spinlocks in non-preemptible kernels.  Thus it should not be
* used in the general case to determine whether sleeping is possible.
* Do not use in_atomic() in driver code.
*/
#define in_atomic()	((preempt_count() & ~PREEMPT_ACTIVE) != 0)

/*
* Check whether we were atomic before we did preempt_disable():
* (used by the scheduler, *after* releasing the kernel lock)
*/
#define in_atomic_preempt_off() \
        ((preempt_count() & ~PREEMPT_ACTIVE) != PREEMPT_DISABLE_OFFSET)

#ifdef CONFIG_PREEMPT_COUNT
# define preemptible()	(preempt_count() == 0 && !irqs_disabled())
#else
# define preemptible()	0
#endif
```
- 从注释中可知：“we run all interrupt handlers with interrupts disabled, so we cannot have nesting interrupts. Though there are a few palaeontologic drivers which reenable interrupts in the handler.” **即在中断处理函数中是处于关中断的状态，但并不排除有少部分驱动程序会在处理函数中重新使能中断**。
- 从宏定义 `#define preemptible() (preempt_count() == 0 && !irqs_disable())` 可知，当 thread_info->preempt_count（见后续 `preempt_count()` 的定义） 不等于 0 时就是不可抢占的。
- **检查抢占状态：`preemptible()` 和 `in_atomic_preempt_off()`**。

### preempt_disable
```c
// version-4.1.15
// include/linux/preempt.h
/*
* include/linux/preempt.h - macros for accessing and manipulating
* preempt_count (used for kernel preemption, interrupt count, etc.)
*/
...
/*
* We use the MSB mostly because its available; see <linux/preempt_mask.h> for
* the other bits -- can't include that header due to inclusion hell.
*/
#define PREEMPT_NEED_RESCHED	0x80000000

...

#ifdef CONFIG_PREEMPT_COUNT

#define preempt_disable() \
do { \
    preempt_count_inc(); \
    barrier(); \
} while (0)
...

#define preempt_count_inc() preempt_count_add(1)

// kernel/sched/core.c
/* Kernel scheduler and related syscalls */
#if defined(CONFIG_PREEMPT) && (defined(CONFIG_DEBUG_PREEMPT) || \
            defined(CONFIG_PREEMPT_TRACER))

void preempt_count_add(int val)
{
#ifdef CONFIG_DEBUG_PREEMPT
    /*
    * Underflow?
    */
    if (DEBUG_LOCKS_WARN_ON((preempt_count() < 0)))
        return;
#endif
    __preempt_count_add(val);
#ifdef CONFIG_DEBUG_PREEMPT
    /*
    * Spinlock count overflowing soon?
    */
    DEBUG_LOCKS_WARN_ON((preempt_count() & PREEMPT_MASK) >=
                PREEMPT_MASK - 10);
#endif
    if (preempt_count() == val) {
        unsigned long ip = get_parent_ip(CALLER_ADDR1);
#ifdef CONFIG_DEBUG_PREEMPT
        current->preempt_disable_ip = ip;
#endif
        trace_preempt_off(CALLER_ADDR0, ip);
    }
}
EXPORT_SYMBOL(preempt_count_add);
NOKPROBE_SYMBOL(preempt_count_add);

// include/asm-generic/preempt.h
#define PREEMPT_ENABLED	(0)

static __always_inline int preempt_count(void)
{
    return current_thread_info()->preempt_count;
}
...

static __always_inline void __preempt_count_add(int val)
{
    *preempt_count_ptr() += val;
}

static __always_inline int *preempt_count_ptr(void)
{
    return &current_thread_info()->preempt_count;
}
```
- preempt_count_inc: 将 current_thread_info 的 preempt_count 的值加 1（不是原子操作）。
  - 在 `/include/linux/preempt_mask.h` 中：
    ```c
    # define preemptible()	(preempt_count() == 0 && !irqs_disabled())
    ```
- barrier:
    ```c
    // version-4.1.15
    // include/linux/compiler-gcc.h
    #define barrier() __asm__ __volatile__("": : :"memory")
    ```
    - `__asm__`：GCC 内联汇编关键字；
    - `__volatile__`：禁止优化，告诉编译器不要删除、不要重排序这段汇编；
    - ""：空汇编指令，没有实际机器码生成；
    - 空的输入和输出；
    - `memory`：破坏描述，告诉编译器这段代码会修改任意内存。编译器会：
      - 放弃所有内存相关的优化假设；
      - 重新从内存加载变量，而不是使用寄存器缓存；
      - 将寄存器中的修改写回内存，而不是延迟到后面。
    - barrier() 与真正的硬件内存屏障
        | 特性        | `barrier()` (编译器屏障) | `smp_mb()` (硬件内存屏障)           |
        | --------- | ------------------- | ----------------------------- |
        | **作用对象**  | **编译器优化**           | **CPU 乱序执行**                  |
        | **生成指令**  | ❌ 无（空汇编）            | ✅ 有（如 ARM `dmb`，x86 `mfence`） |
        | **防止重排序** | 编译阶段                | 运行阶段                          |
        | **开销**    | 零                   | 有（流水线刷新）                      |
        | **适用场景**  | 单核、编译器优化控制          | 多核、缓存一致性                      |
    - 与 volatile 的区别
        | 特性   | `volatile`  | `barrier()` |
        | ---- | ----------- | ----------- |
        | 作用范围 | 单个变量        | **所有内存**    |
        | 语义   | 每次访问都走内存    | 此点刷新所有内存状态  |
        | 优化控制 | 对该变量禁用优化    | 全局内存优化边界    |
        | 使用场景 | 硬件寄存器、多线程标志 | 临界区边界、锁操作   |
    - 总结：
      - 是一个“空指令”编译器屏障，通过 `"memory"` 强制编译器在该点放弃所有内存优化假设—— **barrier() 前的代码，编译器保证所有内存读写操作已完成（刷到内存）；barrier() 后的代码，编译器重新从内存加载变量，不使用旧缓存**；
      - **barrier() 只防编译器优化，不防 CPU 乱序**；
      - 多核场景需要配合 smp_mb() 等硬件内存屏障。
- 为什么 preempt_disable() 需要 barrier()：
  - 确保 preempt_count 的修改对调度器立即可见，并防止临界区代码被编译器重新排序到锁外。

### preempt_count 在 schedule 中的体现
调度器会在以下几个位置检查 preempt_count：
1. `preempt_schedule()` - 抢占式调度入口：当 preempt_count 不为 0 或者中断 disabled 时，直接返回，不进行调度。
```c
// version-4.1.15
// kernel/sched/core.c
static void __sched notrace preempt_schedule_common(void)
{
    do {
        __preempt_count_add(PREEMPT_ACTIVE);
        __schedule();
        __preempt_count_sub(PREEMPT_ACTIVE);

        /*
        * Check again in case we missed a preemption opportunity
        * between schedule and now.
        */
        barrier();
    } while (need_resched());
}

#ifdef CONFIG_PREEMPT
/*
* this is the entry point to schedule() from in-kernel preemption
* off of preempt_enable. Kernel preemptions off return from interrupt
* occur there and call schedule directly.
*/
asmlinkage __visible void __sched notrace preempt_schedule(void)
{
    /*
    * If there is a non-zero preempt_count or interrupts are disabled,
    * we do not want to preempt the current task. Just return..
    */
    if (likely(!preemptible()))
        return;

    preempt_schedule_common();
}
NOKPROBE_SYMBOL(preempt_schedule);
EXPORT_SYMBOL(preempt_schedule);
```

2. __schedule() — 通用调度器核心
```c
// version-4.1.15
// kernel/sched/core.c

/*
 * Various schedule()-time debugging checks and statistics:
 */
static inline void schedule_debug(struct task_struct *prev)
{
#ifdef CONFIG_SCHED_STACK_END_CHECK
	BUG_ON(unlikely(task_stack_end_corrupted(prev)));
#endif
	/*
	 * Test if we are atomic. Since do_exit() needs to call into
	 * schedule() atomically, we ignore that path. Otherwise whine
	 * if we are scheduling when we should not.
	 */
	if (unlikely(in_atomic_preempt_off() && prev->state != TASK_DEAD))
		__schedule_bug(prev);
	rcu_sleep_check();

	profile_hit(SCHED_PROFILING, __builtin_return_address(0));

	schedstat_inc(this_rq(), sched_count);
}

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
- `__schedule()` is the main scheduler function；
- `__schedule()` 在执行调度前会先通过 `preempt_disable()` 禁止抢占；
- `__schedule()` 通过调用 `schedule_debug()` 进一步检查：`in_atomic_preemopt_off()`；

3. 中断返回路径（最关键的检查点）
```asm
// version-4.1.15
// arch/arm/kernel/entry-armv.S
__irq_usr:
    usr_entry
    kuser_cmpxchg_check
    irq_handler
    get_thread_info tsk
    mov why, #0
    b   ret_to_user_from_irq
 UNWIND(.fnend     )
ENDPROC(__irq_user)

// arch/arm/kernel/entry-common.S
work_pending:
    mov r0, sp
    mov r2, why
    bl  do_work_pending
    cmp r0, #0
    beq no_work_pending
    ...

ENTRY(ret_to_user_from_irq)
    ldr r1, [tsk, #TI_FLAGS]
    tst r1, #_TIF_WORK_MASK
    bne work_pending
no_work_pending:
    asm_trace_hardirqs_on

    /* perform architecture specific actions before user return */
    arch_ret_to_user r1, lr
    ct_user_enter save = 0

    restore_user_regs fast = 0, offset = 0
ENDPROC(ret_to_user_from_irq)
```
```c
// version-4.1.15
// arch/arm/kernel/signal.c
asmlinkage int do_work_pending(struct pt_regs *regs, unsigned int thread_flags, int syscall) {
    do {
        if (likely(thread_flags & _TIF_NEED_RESCHED)) {
            schedule();
        } else ...
    }
}

// kernel/sched/core.c
asmlinkage __visible void __sched schedule(void) {
    struct task_struct *tsk = current;

    sched_submit_work(tsk);
    do {
        __schedule();
    } while (need_resched());
}
EXPORT_SYMBOL(schedule);
```
- `_TIF_NEED_RESCHED`、`_TIF_WORK_MASK` 等可在 `arch/arm/include/asm/thread_info.h` 中找到；
- `TI_FLAGS` 位于 `arch/arm/kernel/asm-offset.c` 文件内；
- 从中断中返回用户程序时，可能会调用 `__schedule()` 来进行调度。