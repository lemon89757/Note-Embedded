# Note for ARM
## 寄存器描述
```c
// version: 4.1.15
// arch/arm/include/asm/ptrace.h
struct pt_regs {
	unsigned long uregs[18];
};
```
- `uregs[0]` 到 `uregs[15]` 分别对应 `R0` 到 `R15` (`PC`)
- `uregs[16]` 是 `CPSR`（当前程序状态寄存器）
- `uregs[17]` 通常存放原始的 `R0`（`ORIG_R0`）

## thread info
### 数据结构描述
```c
// version: 4.1.15
// arch/arm/include/asm/thread_info.h
struct cpu_context_save {
	__u32	r4;
	__u32	r5;
	__u32	r6;
	__u32	r7;
	__u32	r8;
	__u32	r9;
	__u32	sl;
	__u32	fp;
	__u32	sp;
	__u32	pc;
	__u32	extra[2];		/* Xscale 'acc' register, etc */
};

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
- *TLS registers*: *TLS* 寄存器指的是 *TPIDRURO*（以及相关的 *TPIDRURW*）。它们是 *ARM* 协处理器（*CP15*）中的系统寄存器，专门用来存放线程本地存储（*Thread Local Storage*）的指针，供用户空间程序快速访问自己的线程私有数据。

### 获取 thread info
```c
// version: 4.1.15
// arch/arm/include/asm/thread_info.h
/* how to get the current stack pointer in C */
register unsigned long current_stack_pointer asm ("sp");

/* how to get the thread information struct from C */
static inline struct thread_info *current_thread_info(void) __attribute_const__;

static inline struct thread_info *current_thread_info(void) 
{
    return (struct thread_info *)(current_stack_pointer & ~(THREAD_SIZE - 1));
}

// arch/arm/include/asm
#define THREAD_SIZE_ORDER   1
#define THREAD_SIZE (PAGE_SIZE << THREAD_SIZE_ORDER)
#define THREAD_START_SP (THREAD_SIZE - 8)
```
- *thread info* 位于栈空间的顶部
- 栈空间大小通过 `THREAD_SIZE` 指定，大小为 2 个页

## MMU 相关
### LAZY TLB
> 《深入 Linux 内核架构》-第二章
> 惰性 TLB 处理（lazy TLB handling）：假如内核线程之后运行的进程与之前是同一个。在这种情况下，内核并不需要修改用户空间地址表，地址转换后备缓冲器（即 TLB）中的信息仍然有效。只有在内核线程之后执行的进程是与此前不同的用户层进程时，才需要切换（并对应清除 TLB 数据）

```c
// version: 4.1.15
// arch/arm/include/asm/mmu_context.h
/** 
 * This is called when "tsk" is about to enter lazy TLB mode. 
 *
 * mm: describes the currently active mm context
 * tsk: task with is entering lazy tlb
 * cpu: cpu number which is entering lazy tlb
 *
 * tsk->mm will be NULL
 */
static inline void enter_lazy_tlb(struct mm_struct *mm, struct task_struct *tsk) 
{
}
```

### switch mm
```c
// arch/arm/include/asm/mmu_context.h
/* 
 * This is the actual mm switch as far as the scheduler 
 * is concerned. No registers are touched. We avoid calling the
 * CPU specific function when the mm hasn't actually changed.
 */
static inline void switch_mm(struct mm_struct *prev, struct mm_struct *next,
    struct task_struct *tsk) {
#ifdef CONFIG_MMU
    unsigned int cpu = smp_processor_id();

    /* 
    * __sync_icache_dcache doesn't broadcast the I-cache invalidation,
    * so check for possible thread migration and invalidate the I-cache
    * if we're new to this CPU.
    */
    if (cache_ops_need_broadcast() &&
        !cpumask_empty(mm_cpumask(next)) &&
        !cpumask_test_cpu(cpu, mm_cpumask(next)))
        __flush_icache_all();
    
    if (!cpumask_test_and_set_cpu(cpu, mm_cpumask(next)) || prev != next) {
        check_and_switch_context(next, tsk);
        if (cache_is_vivt())
            cpumask_clear_cpu(cpu, mm_cpumask(prev));
    }
#endif
}

// arch/arm/include/asm/smp_plat.h
#if !defined(CONFIG_SMP) || __LINUX_ARM_ARCH__ >= 7
#define cache_ops_need_broadcast()	0
```
- `__flush_icache_all()`: 刷新指令缓存的数据，但是当 *ARM* 的架构大于 7 时，不需要此操作
- 如果下一个线程没有在当前 *CPU* 上运行过或者两个 *mm_struct* 不相同，则需要通过 `check_and_switch_context()` 进行实际的上下文切换。

1. `check_and_switch_context()`:
    ```c
    // version: 4.1.15
    // arch/arm/mm/context.c
    void check_and_switch_context(struct mm_struct *mm, struct task_struct *tsk)
    {
        unsigned long flags;
        unsigned int cpu = smp_processor_id();
        u64 asid;

        if (unlikely(mm->context.vmalloc_seq != init_mm.context.vmalloc_seq))
            __check_vmalloc_seq(mm);

        /*
        * We cannot update the pgd and the ASID atomicly with classic
        * MMU, so switch exclusively to global mappings to avoid
        * speculative page table walking with the wrong TTBR.
        */
        cpu_set_reserved_ttbr0();

        asid = atomic64_read(&mm->context.id);
        if (!((asid ^ atomic64_read(&asid_generation)) >> ASID_BITS)
            && atomic64_xchg(&per_cpu(active_asids, cpu), asid))
            goto switch_mm_fastpath;

        raw_spin_lock_irqsave(&cpu_asid_lock, flags);
        /* Check that our ASID belongs to the current generation. */
        asid = atomic64_read(&mm->context.id);
        if ((asid ^ atomic64_read(&asid_generation)) >> ASID_BITS) {
            asid = new_context(mm, cpu);
            atomic64_set(&mm->context.id, asid);
        }

        if (cpumask_test_and_clear_cpu(cpu, &tlb_flush_pending)) {
            local_flush_bp_all();
            local_flush_tlb_all();
        }

        atomic64_set(&per_cpu(active_asids, cpu), asid);
        cpumask_set_cpu(cpu, mm_cpumask(mm));
        raw_spin_unlock_irqrestore(&cpu_asid_lock, flags);

    switch_mm_fastpath:
        cpu_switch_mm(mm->pgd, mm);
    }
    ```
    - *ASID* 的位数由硬件决定（*Linux* 内核源码中的 `ASID_BITS`），因此可同时存在的 *ASID* 数量有限。内核用 `asid_generation` 来管理 *ASID* 的复用：当 *ASID* 空间耗尽时，`asid_generation` 递增，旧 `asid_generation` 的 *ASID* 全部作废，需要重新分配。内核中 `(asid ^ asid_generation) >> ASID_BITS` 就是在判断某个 `ASID` 是否属于当前 `asid_generation`
    - 无论走快速还是慢速路径，最终都调用 `cpu_switch_mm(mm->pgd, mm)`，把新进程的页表基址和 *ASID* 真正装入 *CPU*
    - `cpu_switch_mm(mm->pgd, mm)` 把信息装入 *CPU* 内部的 *MMU*（`cpu_switch_mm()` 是架构相关的底层函数，在 *ARM* 上通常最终会执行一条写协处理器寄存器的指令（如 `mcr p15, 0, ...`）或内存映射的系统寄存器写操作，把“*PGD* 物理地址 + *ASID*”这个组合值真正送进 *MMU* 的 *TTBR0* 寄存器。一旦写入完成，*MMU* 后续做虚拟地址翻译时，就会以这个新的页表为起点，并给产生的 *TLB* 表项打上新的 *ASID* 标签）：
      - *TTBR0*：页表基址寄存器，写入 `mm->pgd`（新进程页表的物理基地址）
      - *ASID* 字段：在 *ARMv7* 中，*ASID* 就存放在 *TTBR0* 的高位字段里，和页表基址一起被写入

## switch
1. `switch_to()`
    ```c
    // version: 4.1.15
    // arch/arm/include/asm/switch_to.h

    /*
    * switch_to(prev, next) should switch from task `prev' to `next'
    * `prev' will never be the same as `next'.  schedule() itself
    * contains the memory barrier to tell GCC not to cache `current'.
    */
    extern struct task_struct *__switch_to(struct task_struct *, struct thread_info *, struct thread_info *);

    #define switch_to(prev,next,last)					\
    do {									\
        last = __switch_to(prev,task_thread_info(prev), task_thread_info(next));	\
    } while (0)

    // arch/arm/kernel/entry-armv.S
    /*
    * Register switch for ARMv3 and ARMv4 processors
    * r0 = previous task_struct, r1 = previous thread_info, r2 = next thread_info
    * previous and next are guaranteed not to be the same.
    */
    ENTRY(__switch_to)
    UNWIND(.fnstart	)
    UNWIND(.cantunwind	)
        add	ip, r1, #TI_CPU_SAVE
    ARM(	stmia	ip!, {r4 - sl, fp, sp, lr} )	@ Store most regs on stack
    THUMB(	stmia	ip!, {r4 - sl, fp}	   )	@ Store most regs on stack
    THUMB(	str	sp, [ip], #4		   )
    THUMB(	str	lr, [ip], #4		   )
        ldr	r4, [r2, #TI_TP_VALUE]
        ldr	r5, [r2, #TI_TP_VALUE + 4]
    #ifdef CONFIG_CPU_USE_DOMAINS
        ldr	r6, [r2, #TI_CPU_DOMAIN]
    #endif
        switch_tls r1, r4, r5, r3, r7
    #if defined(CONFIG_CC_STACKPROTECTOR) && !defined(CONFIG_SMP)
        ldr	r7, [r2, #TI_TASK]
        ldr	r8, =__stack_chk_guard
        ldr	r7, [r7, #TSK_STACK_CANARY]
    #endif
    #ifdef CONFIG_CPU_USE_DOMAINS
        mcr	p15, 0, r6, c3, c0, 0		@ Set domain register
    #endif
        mov	r5, r0
        add	r4, r2, #TI_CPU_SAVE
        ldr	r0, =thread_notify_head
        mov	r1, #THREAD_NOTIFY_SWITCH
        bl	atomic_notifier_call_chain
    #if defined(CONFIG_CC_STACKPROTECTOR) && !defined(CONFIG_SMP)
        str	r7, [r8]
    #endif
    THUMB(	mov	ip, r4			   )
        mov	r0, r5
    ARM(	ldmia	r4, {r4 - sl, fp, sp, pc}  )	@ Load all regs saved previously
    THUMB(	ldmia	ip!, {r4 - sl, fp}	   )	@ Load all regs saved previously
    THUMB(	ldr	sp, [ip], #4		   )
    THUMB(	ldr	pc, [ip]		   )
    UNWIND(.fnend		)
    ENDPROC(__switch_to)
    ```
    - **在 `__switch_to()` 里面就切换到下一个进程开始运行了**
    - 根据 *AAPCS* 调用约定，进入 `__switch_to()` 时，`r0 = prev`、`r1 = thread info of prev`、`r2 = thread info of next`
    - `TI_CPU_SAVE` 是 *thread_info* 结构体中 `cpu_context` 字段的偏移，其定义位于 `arch/arm/kernel/asm-offsets.c` 中。其他偏移值也可以在其中找到
    - 这里的 `ARM(...)` 和 `THUMB(...)` 为条件汇编宏，它们的作用是一样的。但根据当前编译目标是 *ARM* 指令集还是 *Thumb* 指令，只选择其中一个编译进内核。
      - *ARM* 指令集固定 4 字节，功能完整，寄存器操作灵活；*THUMB* 指令集的指令长度更短（*THUMB-2* 是 2 或 4 字节混合的），代码密度更高，但部分指令的寄存器使用受限
      - 内核可以通过 `CONFIG_THUMB2_KERNEL` 配置选择用 *THUMB-2* 编译内核，`__switch_to()` 这段汇编必须在两种模式下都能工作，所以用 `ARM()` 和 `THUMB()` 宏分别提供两套实现
      - 举例：`ARM(	stmia	ip!, {r4 - sl, fp, sp, lr} )`，一条 `stmia` 就把 `r4~r10`（`sl`）、`r11`（`fp`）、`sp`、`lr` 全部保存，并让 `ip` 自增。*ARM* 指令集允许在一条多寄存器加载/存储指令里同时包含 `sp` 和 `lr`，寄存器列表很灵活。但是要表达同样功能，*THUMB* 使用了三条指令，因为它的 `stmia` 指令对寄存器列表有更严格的限制。
    - 切出进程的上下文保存：
        ```asm
        add	ip, r1, #TI_CPU_SAVE
        ARM(	stmia	ip!, {r4 - sl, fp, sp, lr} )	@ Store most regs on stack
        ```
        - 上述代码将寄存器 `r4 ~ sl` 和 `fp`（根据 *AAPCS*）、`sp`（当前栈地址）、`lr`（函数返回地址，即 `__switch_to()` 的返回地址，即未来的 `pc` 值）保存到 `prev` 对应的 `thread_info->cpu_context_save` 中，正好可以与 `struct cpu_context_save` 的定义对应上
    - 切入进程的上下文恢复：
        ```asm
        ldr	r4, [r2, #TI_TP_VALUE]
	    ldr	r5, [r2, #TI_TP_VALUE + 4]
        ```
        - 恢复 `next` 对应的 `thread_info->tp_value`，它们保存了线程本地存储的指针
        ```asm
        ARM(	ldmia	r4, {r4 - sl, fp, sp, pc}  )	@ Load all regs saved previously
        ```
        - 从 `next` 的 `thread_info->cpu_context` 中恢复 `r4 ~ sl` 和 `fp`、`sp` 和 `pc` 值
        - `r0` 寄存器作为 `__switch_to()` 的返回值，它保存了 `prev` 的值。**因此，`__switch_to()` 的返回值是当前线程运行前的上一个进程**

2. `finish_arch_switch()`
    ```c
    // version: 4.1.15
    // arch/arm/include/asm/switch_to.h
    /*
     * For v7 SMP cores running a preemptible kernel we may be pre-empted
     * during a TLB maintenance operation, so execute an inner-shareable dsb
     * to ensure that the maintenance completes in case we migrate to another
     * CPU.
     */
    #if defined(CONFIG_PREEMPT) && defined(CONFIG_SMP) && defined(CONFIG_CPU_V7)
    #define finish_arch_switch(prev)	dsb(ish)
    #endif
    ```
    - 执行数据同步的内存屏障指令