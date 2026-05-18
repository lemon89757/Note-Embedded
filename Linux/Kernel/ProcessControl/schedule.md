# 进程调度
## preempt_disable
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
- preempt_count 的位域布局和相关状态判断：
    ```c
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
- preempt_count_inc: 将 current_thread_info 的 preempt_count 的值加 1（不是原子操作）。通过 thread_info 结构体定义处的注释：0 => preemptable, <0 => bug 可知，只有当 preempt_count 等于 0 时才是可抢占的。通过相关源码可知，当 preempt_count 大于 0 时，其对应的 current_thread_info 是无法被抢占的：
  - 在 `/include/linux/preempt_mask.h` 中：
    ```c
    # define preemptible()	(preempt_count() == 0 && !irqs_disabled())
    ```
  - 调度器会在以下几个位置检查：
    1. preempt_schedule() - 抢占式调度入口
        ```c
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
    3. 中断返回路径（最关键的检查点）
- barrier: