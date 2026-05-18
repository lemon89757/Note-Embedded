# 同步与互斥
## 同步与互斥的失败例子
- 失败例子：略
- 失败原因：
  1. 变量的修改需要经过如下三个步骤，它们不是以原子性的操作完成的。
     1. 读出：将变量从内存读到寄存器中；
     2. 修改：修改寄存器中的变量；
     3. 写入：将变量从寄存器写入到内存中。
  2. 针对单个 CPU 执行关中断操作：`raw_local_irq_save(flags) ~ raw_local_irq_restore(flags)`，对单 CPU 环境有效，但在多核 CPU 的情况下无法消除其他 CPU 导致的竞争。
- 目标：
  - 同一个 CPU 上的线程和进程不会产生竞争问题；
  - 不同 CPU 上的线程和进程也不会产生竞争问题；
- 实现方式
  - 原子操作
  - 锁

## 相关概念
### preempt-抢占

## 原子操作
所谓原子操作就是该操作不会被打断。Linux 中有两种原子操作：原子变量、原子位。

### 原子变量
1. 原子变量类型：
    ```c
    // include/linux/types.h
    typedef struct {
        int counter;
    } atomic_t;
    ```

2. 原子变量操作函数：
    ```c
    // arch/arm/include/asm/atomic.h
    ```
    - 实现原子性的重点在于操作函数，原子变量类型只是普通的 int 类型；

3. 原子操作的内核实现
- 在内核中有两套实现机制：
  1. 小于 ARMv6 指令集（架构）的 CPU；
  2. 大于等于 ARMv6 指令集（架构）的 CPU；

- 小于 ARMv6 指令集的 CPU：
  - ARMv6 不支持多个 CPU 的系统（SMP）；
  - 实现方式：关中断 ~ 恢复中断；
    ```c
    // version-4.1.15
    // arch/arm/include/asm/atomic.h
    #else /* ARM_ARCH_6 */

    #ifdef CONFIG_SMP
    #error SMP not supported on pre-ARMv6 CPUs
    #endif

    #define ATOMIC_OP(op, c_op, asm_op)					\
    static inline void atomic_##op(int i, atomic_t *v)			\
    {									\
        unsigned long flags;						\
                                        \
        raw_local_irq_save(flags);					\
        v->counter c_op i;						\
        raw_local_irq_restore(flags);					\
    }									\

    #define ATOMIC_OP_RETURN(op, c_op, asm_op)				\
    static inline int atomic_##op##_return(int i, atomic_t *v)		\
    {									\
        unsigned long flags;						\
        int val;							\
                                        \
        raw_local_irq_save(flags);					\
        v->counter c_op i;						\
        val = v->counter;						\
        raw_local_irq_restore(flags);					\
                                        \
        return val;							\
    }

    static inline int atomic_cmpxchg(atomic_t *v, int old, int new)
    {
        int ret;
        unsigned long flags;

        raw_local_irq_save(flags);
        ret = v->counter;
        if (likely(ret == old))
            v->counter = new;
        raw_local_irq_restore(flags);

        return ret;
    }

    static inline int __atomic_add_unless(atomic_t *v, int a, int u)
    {
        int c, old;

        c = atomic_read(v);
        while (c != u && (old = atomic_cmpxchg((v), c, c + a)) != c)
            c = old;
        return c;
    }

    #endif /* __LINUX_ARM_ARCH__ */
    ```
    - 通过关中断和恢复中断的方式实现：`raw_local_irq_save(flags) ~ raw_local_irq_restore(flags)`。

- 大于 ARMv6 指令集的 CPU：
    - 支持 SMP 系统；
    - 实现方式：通过 ARMv6 的排它/独占（exclusive）汇编指令实现的；
    - 通过 `ldrex` 和 `strex` 指令，既可以防止同一 CPU 也可以防止不同 CPU 的进程或线程的竞争；
    - 通过独占汇编指令的实现方式效率更高，因为它无需关闭 CPU 的中断（关中断对系统性能影响较大）；
    ```c
    // version-4.1.15
    // arch/arm/include/asm/atomic.h
    #if __LINUX_ARM_ARCH__ >= 6

    /*
    * ARMv6 UP and SMP safe atomic ops.  We use load exclusive and
    * store exclusive to ensure that these are atomic.  We may loop
    * to ensure that the update happens.
    */

    #define ATOMIC_OP(op, c_op, asm_op)					\
    static inline void atomic_##op(int i, atomic_t *v)			\
    {									\
        unsigned long tmp;						\
        int result;							\
                                        \
        prefetchw(&v->counter);						\
        __asm__ __volatile__("@ atomic_" #op "\n"			\
    "1:	ldrex	%0, [%3]\n"						\
    "	" #asm_op "	%0, %0, %4\n"					\
    "	strex	%1, %0, [%3]\n"						\
    "	teq	%1, #0\n"						\
    "	bne	1b"							\
        : "=&r" (result), "=&r" (tmp), "+Qo" (v->counter)		\
        : "r" (&v->counter), "Ir" (i)					\
        : "cc");							\
    }									\

    #define ATOMIC_OP_RETURN(op, c_op, asm_op)				\
    static inline int atomic_##op##_return(int i, atomic_t *v)		\
    {									\
        unsigned long tmp;						\
        int result;							\
                                        \
        smp_mb();							\
        prefetchw(&v->counter);						\
                                        \
        __asm__ __volatile__("@ atomic_" #op "_return\n"		\
    "1:	ldrex	%0, [%3]\n"						\
    "	" #asm_op "	%0, %0, %4\n"					\
    "	strex	%1, %0, [%3]\n"						\
    "	teq	%1, #0\n"						\
    "	bne	1b"							\
        : "=&r" (result), "=&r" (tmp), "+Qo" (v->counter)		\
        : "r" (&v->counter), "Ir" (i)					\
        : "cc");							\
                                        \
        smp_mb();							\
                                        \
        return result;							\
    }

    static inline int atomic_cmpxchg(atomic_t *ptr, int old, int new)
    {
        int oldval;
        unsigned long res;

        smp_mb();
        prefetchw(&ptr->counter);

        do {
            __asm__ __volatile__("@ atomic_cmpxchg\n"
            "ldrex	%1, [%3]\n"
            "mov	%0, #0\n"
            "teq	%1, %4\n"
            "strexeq %0, %5, [%3]\n"
                : "=&r" (res), "=&r" (oldval), "+Qo" (ptr->counter)
                : "r" (&ptr->counter), "Ir" (old), "r" (new)
                : "cc");
        } while (res);

        smp_mb();

        return oldval;
    }

    static inline int __atomic_add_unless(atomic_t *v, int a, int u)
    {
        int oldval, newval;
        unsigned long tmp;

        smp_mb();
        prefetchw(&v->counter);

        __asm__ __volatile__ ("@ atomic_add_unless\n"
    "1:	ldrex	%0, [%4]\n"
    "	teq	%0, %5\n"
    "	beq	2f\n"
    "	add	%1, %0, %6\n"
    "	strex	%2, %1, [%4]\n"
    "	teq	%2, #0\n"
    "	bne	1b\n"
    "2:"
        : "=&r" (oldval), "=&r" (newval), "=&r" (tmp), "+Qo" (v->counter)
        : "r" (&v->counter), "r" (u), "r" (a)
        : "cc");

        if (oldval != u)
            smp_mb();

        return oldval;
    }
    ```
    - `ldrex` 和 `strex`，ex 表示 exclusive；
    - `ldrex %0, [%3]`：从第 3 个操作数所表示的地址处读取值到第 0 个操作数，它在将数据读入的同时标记该地址处为独占访问；
    - 修改数据值，将新值保存在第 0 个操作数中；
    - `strex	%1, %0, [%3]`：尝试将修改后的数值值从第 0 个操作数写入第 3 个操作数所表示的地址处，写入的返回值保存在第 1 个操作数中：
      - 正常情况，即地址仍被标记为独占访问，没有被其他任务抢占：写入新值 -> 清除独占标记 -> 返回 0；
      - 被抢占，即被其他任务抢先完成了 `ldrex` 和 `strex` 操作：`strex` 会发现地址的独占访问标记没有了 -> 放弃写入新值 -> 返回 1；
    - 当 `strex` 的返回值为 0 时，程序结束；当其返回值为 1 时，通过 `bne lb` 跳到 `1` 处继续执行。相当于写入失败时，不断循环尝试，直到成功完成独占读取->写入操作为止。

- atomic_inc(v) -> atomic_add(1, v) -> ATOMIC_OPS(add, +=, add) -> ATOMIC_OP(op, c_op, asm_op) + ATOMIC_OP_RETURN(op, c_op, asm_op) -> ATOMIC_OP 在 ARMv6 之下的实现方式和 ARMv6 之上的实现方式。

- 由于读操作只有一条汇编指令，因此直接返回读取到的结果即可。

- ARMv6 以下的 CPU 通过关闭中断确实实现了无法被抢占的原子操作，ARMv6 及以上架构的 CPU 其实它们的内部汇编程序是可以被抢占的。但是 ARMv6 的汇编程序会保证只要程序执行完成，它的整个流程中是没有产生竞争的。即如果读入->修改->写入的这整个过程中，如果产生了其他程序的抢占，它会重新从读入操作开始（这个时候可以读到最新的值），直到当整个过程的执行没有被抢占才算结束。

### 原子位
1. 实现方式同原子变量
2. 操作函数：
    ```c
    // arch/arm/include/asm/bitops.h
    ```

## Linux 锁
### 锁的类型
Linux 内核提供了多种类型的锁，它们可以分为两类：
- 自旋锁（spinning lock）；
- 睡眠锁（sleeping lock）；

### 自旋锁
1. 源码分析
```c
// version-4.1.15
// include/linux/spinlock.h
/* 在 spinlock.h 中有如下注释： 
 * include/linux/spinlock.h - generic spinlock/rwlock declarations
 *
 * here's the role of the various spinlock/rwlock related include files:
 *
 * on SMP builds:
 *
 *  asm/spinlock_types.h: contains the arch_spinlock_t/arch_rwlock_t and the
 *                        initializers
 *
 *  linux/spinlock_types.h:
 *                        defines the generic type and initializers
 *
 *  asm/spinlock.h:       contains the arch_spin_*()/etc. lowlevel
 *                        implementations, mostly inline assembly code
 *
 *   (also included on UP-debug builds:)
 *
 *  linux/spinlock_api_smp.h:
 *                        contains the prototypes for the _spin_*() APIs.
 *
 *  linux/spinlock.h:     builds the final spin_*() APIs.
 *
 * on UP builds:
 *
 *  linux/spinlock_type_up.h:
 *                        contains the generic, simplified UP spinlock type.
 *                        (which is an empty structure on non-debug builds)
 *
 *  linux/spinlock_types.h:
 *                        defines the generic type and initializers
 *
 *  linux/spinlock_up.h:
 *                        contains the arch_spin_*()/etc. version of UP
 *                        builds. (which are NOPs on non-debug, non-preempt
 *                        builds)
 *
 *   (included on UP-non-debug builds:)
 *
 *  linux/spinlock_api_up.h:
 *                        builds the _spin_*() APIs.
 *
 *  linux/spinlock.h:     builds the final spin_*() APIs.
 */
static inline void spin_lock(spinlock_t *lock) {
    raw_spin_lock(&lock->rlock);
}

/*
 * Define the various spin_lock methods.  Note we define these
 * regardless of whether CONFIG_SMP or CONFIG_PREEMPT are set. The
 * various methods are defined as nops in the case they are not
 * required.
 */
#define raw_spin_lock(lock) _raw_spin_lock(lock)

// kernel/locking/spinlock.c
/* This file contains the spinlock/rwlock implementations for the
 * SMP and the DEBUG_SPINLOCK cases. (UP-nondebug inlines them)
 *
 * Note that some architectures have special knowledge about the
 * stack frames of these functions in their profile_pc. If you
 * change anything significant here that could change the stack
 * frame contact the architecture maintainers.
 */
#ifndef CONFIG_INLINE_SPIN_LOCK
void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
	__raw_spin_lock(lock);
}
EXPORT_SYMBOL(_raw_spin_lock);
#endif

// include/linux/spinlock_api_smp.h
/* spinlock API declarations on SMP (and debug)
 * (implemented in kernel/spinlock.c)
 */
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
	preempt_disable();
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```
- 调用链：
- preempt_disable: 见 `./schedule.md`；
- spin_acquire:
    ```c
    ```
- LOCK_CONTENDED:
    ```c
    ```
- 总结：

### 锁的内核函数
1. 自旋锁
无法获得锁时，不会休眠，会一直循环等待。
   - 通过原子变量实现，会禁止抢占、禁止进程调度（preempt_disable）；
   - 使用场景：
     - 多 CPU 和单 CPU 系统中都适用；
     - 要求临界区的代码执行时间短；
   - 自旋锁的实现

1. 信号量
2. 互斥量
3. 信号量和互斥量的区别

### 锁的使用场景

### 内核抢占等额外感念
- 补充：
  - 不可抢占（初期 Linux 内核）内核和可抢占内核：是否支持在内核态下的进程切换（不可抢占：只要运行进程不主动放弃 CPU 就无法切换、可抢占：高优先级进程可抢占内核态的其他进程）；


## 参考
1. 韦东山-驱动大全
2. [Linux 锁-1](https://www.kernel.org/doc/html/latest/locking/index.html)
3. [Linux 锁-2](https://mirrors.edge.kernel.org/pub/linux/kernel/people/rusty/kernel-locking/)