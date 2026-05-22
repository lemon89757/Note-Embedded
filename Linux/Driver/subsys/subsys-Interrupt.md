# 中断子系统
CPU 每执行完一条指令都会检查有无中断/异常产生，当 CPU 发现有中断/异常产生时，开始处理。对于不同的异常，跳去不同的地址执行程序。

## Linux 中的中断
Linux 系统中有**硬件中断，也有软件中断**。对硬件中断的处理有两个原则：不能嵌套、越快越好。

软件中断即软中断，详情见中断下半部对应部分。

### 硬件中断
对于按键中断等硬件产生的中断，称之为硬件中断（`hard irq`）。
   - **硬件中断不能嵌套**；
   - 对硬件中断的处理要快进快出；
   - 每个硬件中断都有对应的处理函数；

1. 不能嵌套
   中断处理函数需要调用 C 函数，这就需要用到栈（函数调用需要保存现场）。中断嵌套的爆发导致栈空间消耗严重（函数调用也导致额外的时间消耗）。
   - [ ] (补充相关链接-韦东山参考资料)
   - [官方资料]()。
   - 在 Linux 系统上中断无法嵌套，即当前中断 A 处理完之前不会响应另一个中断 B，即使 B 的优先级更高；
   - 通常硬件中断处理函数中会先关闭中断，避免中断嵌套的突然爆发；
2. 快进快出
   - 越快越好；
   - 在单芯片系统中，假设中断处理很慢，那应用程序在这段时间内就无法执行，导致系统显得很迟钝；在 SMP 系统中，假设中断处理很慢，那么正在处理这个中断的 CPU 上的其他线程也无法执行；
   - **在中断处理过程中，对应的 CPU 是不能进行进行调度的**，所以中断的处理要越快越好，尽早让其他中断能被处理（**进程调度靠定时器中断实现**）；
3. 注册中断处理函数
   ```c
   // 为某个硬件中断 irq 注册中断处理函数 handler
   int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev);
   ```
   - `handler` 函数需要尽可能高效；

从 `request_irq` 函数原型可引出后续章节内容（中断号、中断拆分）。

### 中断号
> 来自：`linux/Documentation/zh_CN/IRQ.txt`
> 何为 IRQ？
> 一个 IRQ 是来自某个设备的一个中断请求。目前，它们可以来自一个硬件引脚，或来自一个数据包。多个设备可能连接到同个硬件引脚，从而共享一个 IRQ。
> 一个 IRQ 编号是用于告知硬件中断源的内核标识。**通常情况下，这是一个全局 `irq_desc` 数组的索引**，但是除了在 `Linux/interrupt.h` 中的实现，具体的细节是体系结构特定的。
> 一个 IRQ 编号是设备上某个可能的中断源的枚举。通常情况下，枚举的编号是该引脚在系统内中断控制器的所有输入引脚中的编号。对于 ISA 总线中的情况，枚举的是在两个 i8299 中断控制器中 16 个输入引脚。
> 架构可以对 IRQ 编号指定额外的含义，在硬件涉及任何手工配置的情况下，是被提倡的。ISA 的 IRQ 是一个分配这类额外含义的典型例子。

屏蔽底层硬件的差别，对中断进行统一管理，Linux 将硬件中断号（比如 GIC SPI 12）映射为内部的虚拟中断号，即中断号。

具体映射关系由中断子系统的实现者（通常为 SOC 厂商）决定，Linux 中断子系统只提供统一的 API（与其他子系统类似）。

类似地，Linux 中的 GPIO 号也是如此。

### 中断拆分
在通过 `request_irq()` 函数注册的（硬件）中断处理函数中，其需要处理的内容很难完全避免耗时较长的工作，例如在按键中断中需要等待几十毫秒来消除机械抖动。

当一个（硬件）中断要耗费很长时间来处理时，它的坏处是：在这段时间内，其他中断无法被处理。换句话说，**在这段时间内，系统是关中断的**。

为解决上述问题，实现硬件中断的快进快出，Linux 将中断拆分为上下半部两部分，上半部处理紧急的，处理完后重新开中断让系统得以正常运行（对应到硬件中断）、下半部处理不紧急的，此时是开中断的（对应到软中断等）。

<!-- ## 中断上下半部 -->
- 主要目的
   实现中断处理函数的快进快出，**那些对时间敏感、执行速度快的操作可以放到中断处理函数中，即上半部**；剩下的所有工作都可以放到下半部去执行。

- 参考设置
  - 如果要处理的内容不希望被其他中断打断，那么可以放到上半部；
  - 如果要处理的任务对时间敏感，可以放到上半部；
  - 如果要处理的任务与硬件有关，可以放到上半部；
  - 除了上述三点以外的其他任务，优先考虑放到下半部；

- 处理流程：
   ```c
   __irq_svc()    // at arch/arm/kernel/entry-armv.S:219, svc 模式（管理模式/特权模式）
   // 也有可能是 __irq_usr() 等
      gic_handle_irq // at drivers/irqchip/irq-gic.c:364
         handle_domain_irq // at .include/linux/irqdesc.h:168
            __handle_domain_irq  // at kernel/irq/irqdesc.c:627
               irq_enter();      // 进入硬件中断
                  __irq_enter()
                     preempt_count_add(HARDIRQ_OFFSET)   // preempt_count++，表示开始处理中断
                  generic_handle_irq(irq);   // 会调用 request_irq 注册的中断函数：上半部
                  irq_exit()     // 退出硬件中断
                     preempt_count_sub(HARDIRQ_OFFSET)   // preempt_count--，表示硬件中断处理完毕
                     if (!in_interrupt() && local_softirq_pending()) // in_interrupt: preemt_count > 0
                        invoke_softirq()
                           __do_softirq()
                              __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET) // preempt_count++，开始软中断
                              local_irq_enable()   // 开中断
                              // 执行 softirq_vec 数组上各类软中断：下半部
                              local_irq_disable()  // 关中断，马上就会再开中断
                              __local_bh_enable(SOFTIRQ_OFFSET)   // preempt_count--，完成软中断
   // 通过 preempt_count 避免软中断多次执行
   // 在处理软中断时开中断
   ```
   - 上述流程简化为：
      preempt_count++ -> **中断上半部** -> preempt_count-- -> preempt_count? -> preempt_count++/完成处理（当...）-> 开中断 -> **中断下半部** -> 关中断 -> preempt_count-- -> 完成处理
   - 通过 `preempt_count` 避免软中断多次执行：
     - 当在 `local_irq_enable()` 函数被执行后，此时允许中断产生（可以是同一中断也可以是其他中断）；
     - 中断产生后，CPU 去执行新的处理流程（同上）；
     - 由于在新的处理流程中 `preemt_count` 的初始值为前一次流程中 `local_irq_enable()` 前的值（非零），因此在新的处理流程中并不会进入到 `invoke_softirq()` 函数内；
     - **在新的流程中只会进行中断上半部的处理**；

- 总结：
  - 中断的处理可以分为上半部、下半部；
  - 中断**上半部**，用来处理紧急的事，它是在**关中断**的状态下执行的；
  - 中断**下半部**，用来处理耗时的、不那么紧急的事，它是在**开中断**的状态下执行的；
  - 中断下半部执行时，有可能会被多次打断，有可能会再次发生同一个中断；
  - 中断上半部执行完后，触发中断下半部的处理；
- [ ] 中断下半部中不能休眠吗？为什么？
  - 中断上半部、下半部的执行过程中，不能休眠；

### 上半部
   - 对应到（硬件）中断处理函数；
   - 关中断情况下进行处理；
   - 此时无法处理其他中断；

1. 注册中断处理函数
```c
// 为某个硬件中断 irq 注册中断处理函数 handler
int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev);
```
- `request_irq()` 会使能中断；可能会导致休眠，因此不能再中断上下文或者其他禁止休眠的代码段中使用 `request_irq` 函数；
- `handler` 函数需要尽可能高效；
- `flag`：
   ```c
   // include/linux/interrupt.h

   // 多个设备共享一个中断，所有共享的中断都必须指定此标志。如果使用共享中断的话，dev 参数就是唯一区分它们的标志
   IRQF_SHARED
   // 单次中断，中断执行一次就结束
   IRQF_ONESHOT
   // 无触发
   IRQF_TRIGGER_NONE
   // 上升沿触发
   IRQF_TRIGGER_RISING
   // 下降沿触发
   IRQF_TRIGGER_FALLING
   // 高电平触发
   IRQF_TRIGGER_HIGH
   // 低电平触发
   IRQF_TRIGGER_LOW
   ```
- `name`：中断名，设置以后可以在 `/proc/interrupts` 文件中看到对应的中断名字；
- `dev`：**如果将 `flags` 设置为 `IRQF_SHARED` 的话，`dev` 用来区分不同的中断**。一般情况下将其设置为设备结构体，它会以第二个参数传递给中断处理函数。
- 返回值：0-中断申请成功；其他负值-中断申请失败，-EBUSY 表示中断已经被申请了。

2. 释放中断
```c
// 使用完后释放中断
// 共享中断时，通过 dev 参数来进行区分。只有当最后一个中断处理函数被释放后共享中断才会被禁止掉。
void free_irq(unsigned int irq, void *dev);
```

3. 中断处理函数
```c
// 中断处理函数
// int: 中断号
irqreturn_t (*irq_handler_t)(int, void*);

// 返回值类型 irqreturn_t 为枚举类型：
enum irqreturn {
   IRQ_NONE          = (0 << 0),
   IRQ_HANDLED       = (1 << 0),
   IRQ_WAKE_THREAD   = (1 << 1),
};
typedef enum irqreturn irqreturn_t
// 一般中断服务函数返回值使用的形式：return IRQ_RETVAL(IRQ_HANDLED)
```

4. 中断使能与禁止
```c
void enable_irq(unsigned int irq);
void disable_irq(unsigned int irq);          // 需要等到当前正在执行的中断处理函数执行完才返回
void disable_irq_nosync(unsigned int irq);   // 立即返回

// 使能/失能当前处理器中断系统
local_irq_enable()
local_irq_disable()

// 带有中断标志的使能和失能
local_irq_save(flags)
local_irq_restore(flags)
```

### 下半部
#### 软中断
一开始 Linux 内核提供了 `Bottom half, BH` 机制来实现下半部，不过已被废弃（2.5 版本），目前主要使用软中断和 `tasklet`。

人为制造中断：软中断（`soft irq`）。是内核软件触发的异步处理机制，与中断编号无关，是完全独立的机制。

软中断不是“软件模拟的中断”，而**是内核的下半部（`Bottom Half`）机制之一**，与硬件中断编号体系完全解耦。

1. 内核使用 `softirq_action` 表示软中断：
```c
// include/linux/interrupt.h
struct softirq_action {
   void (*action)(struct softirq_action *);     // 软中断服务函数
}
```

2. 软中断类型（有哪些软中断）：软中断没有编号概念，只有类型。
```c
// include/linux/interrupt.h
enum {
   HI_SOFTIRT=0,        // 高优先级 tasklet
   TIMER_SOFTIRQ,       // 定时器
   NET_TX_SOFTIRQ,      // 网络发送
   NET_RX_SOFTIRQ,      // 网络接收
   BLOCK_SOFTIRQ,       // 块设备
   IRQ_POLL_SOFTIRQ,    // 轮询
   TASKLET_SOFTIRQ,     // 普通 tasklet
   SCHED_SOFTIRQ,       // 调度
   HRTIMER_SOFTIRQ,     // 高精度定时器；Unused, but kept as tools rely on the numbering, Sigh!
   RCU_SOFTIRQ,         // RCU；Preferable RCU should always be the last softirq
   NR_SOFTIRQS          // 总数（10 个）
}
```

3. 软中断表-10 个软中断
```c
// kernel/softirq.c
// 软中断表（全局，与硬件中断无关）
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned;
```
- 所有 CPU（对于 SMP 系统而言）都可以访问到，每个 CPU 都有自己的触发和控制机制，并且**只执行自己所触发的软中断**；但是各个 CPU 所执行的软中断服务函数却是相同的，都是 `softirq_vec` 中定义的 `action` 函数。

4. 注册软中断处理函数
```c
/* 
 * @nr: 要开启的软中断，从软中断枚举类型中选择
 * @action: 软中断对应的处理函数
 */
extern void open_softirq(int nr, void (*action)(struct softirq_action *));
```

5. 触发方式
   通过 `raise_softirq()` 来触发，参数是从软中断类型中选择一个需要触发的软中断。
- 表示方式：通过位图 `__softirq_pending` 标记 `pending` 状态；
   
- 触发时机：由软件决定，对于 X 号软中断，只需要把它的 `flag`（掩码对应 bit）设置为 1 就表示发生了该中断；
   ```c
   __raise_softirq_irqoff(nr);      // 设置 __softirq_pending 位图

   // 对上述函数的进一步包装？
   extern void raise_softirq(unsigned int nr);     // 简单理解就是设置 softirq_veq[nr] 的标记位
   ```

- 处理时机：在处理完硬件中断后，再处理软中断。（各种硬件中断频繁发生，至少系统时钟中断每 10ms 发生一次）；

6. **软中断必须在编译的时候静态注册**，内核使用 `softirq_init()` 函数来初始化软中断：
```c
void __init softirq_init(void) {
   int cpu;
   for_each_possible_cpu(cpu) {
      per_cpu(tasklet_vec, cpu).tail = &per_cpu(tasklet_vec, cpu).head;
      per_cpu(tasklet_hi_vec, cpu).tail = &per_cpu(tasklet_hi_cpu, cpu).head;
   }

   open_softirq(TASKLET_SOFTIRQ, tasklet_action);
   open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```
- `softirq_init` 默认开启 `TASKLET_SOFTIRQ` 和 `HI_SOFTIRQ`。

#### tasklet
在中断下半部的执行过程中，虽然是开中断的，期间可以处理各类中断。但是毕竟整个中断的处理还没有走完，这期间 APP 是无法执行的。

当下半部比较耗时但能忍受，并且它的处理比较简单时，可以用 `tasklet` 来处理下半部。

- 封装 `softirq`（软中断）；
   `tasklet` 是使用 `softirq`（软中断）来实现，对应到软中断的类型为 `TASKLET_SOFTIRQ`。

- 使用方式：
   1. 先定义 `tasklet`；
   2. 需要使用时调用 `tasklet_schedule(...)`；
   3. 驱动卸载前调用 `tasklet_kill(...)`；
   4. 注意：
      - `tasklet_schedule(...)` 只是把 `tasklet` 放入内核队列，它的 `func` 函数会在软中断的执行过程中被调用；
      - 在软中断（包括 `tasklet`）执行过程中不能休眠；

- 数据结构
   ```c
   // include/linux/interrupt.h
   struct tasklet_struct {
      struct tasklet_struct *next;
      unsigned long state;
      atomic_t count;
      void (*func)(unsigned long);
      unsigned long data;
   }
   ```
   - `state`：
     - `bit0`：表示 `TASKLET_STATE_SCHED`，等于 1 时表示已经执行了 `tasklet_schedule(...)` 把该 `tasklet` 放入队列了；`tasklet_schedule(...)` 会判断该位，如果已经等于 1 那么它就不会再次把 `tasklet` 放入队列；
     - `bit1`：表示 `TASKLET_STATE_RUN`，等于 1 时表示正在只运行 `tasklet` 中的 `func` 函数，函数执行完后内核会把该位清 0；
   - `count`：表示该 `tasklet` 是否使能。0-使能，非 0-被禁止了。对于 `count` 非 0 的 `tasklet`，里面的 `func` 函数不会被执行；

- 通过 `task_init` 或者 `DECLARE_TASKLET` 宏等进行结构体数据初始化（使用中断下半部之前，要先实现一个 `tasklet_struct` 结构体）：
   ```c
   #define DECLARE_TASKLET(name, func, data)    struct tasklet_struct name = {NULL, 0, ATOMIC_INIT(0), func, data}

   #define DECALRE_TASKLET_DISABLE(name, func, data) struct tasklet_struct name = {NULL, 0, ATOMIC_INIT(1), func, data}
   ```
   - 使用 `DECLARE_TASKLET` 定义的 `tasklet` 是使能的；
   - 使用 `DECLARE_TASKLET_DISABLE` 定义的 `tasklet` 是失能的，需要先调用 `tasklet_enable(...)` 使能它；
   ```c
   extern void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data);
   ```

- 使能/失能
   ```c
   static inline void tasklet_enable(struct tasklet_struct *t);   // 把 count 减 1
   static inline void tasklet_disable(struct tasklet_struct *t);  // 把 count 加 1
   ```

- 调度
  ```c
  // 在上半部中通过 task_schedule 注册需要在下半部中进行处理的 tasklet；
  // 将 tasklet 加入到队列中
  static inline void tasklet_schedule(struct tasklet_struct *t);
  ```

- kill tasklet
  ```c
  extern void tasklet_kill(struct tasklet_struct *t);
  ```
  - 如果一个 `tasklet` 未被调度，`tasklet_kill(...)` 会把它的 `TASKLET_STATE_SCHED` 状态清 0；
  - 如果一个 `tasklet` 已被调度，`tasklet_kill(...)` 会等待它执行完毕，再把它的 `TASKLET_STATE_SCHED` 状态清 0；

- 内部机制
  - `tasklet` 是类型为 `TASKLET_SOFTIRQ` 的软中断（软中断），入口函数为 `tasklet_action`：
      ```c
      // kernel/softirq.c
      void __init softirq_init(void) {
         ...
         open_softirq(TASKLET_SOFTIRQ, tasklet_aciton);
      }
      ```
  - `tasklet_action(...)`：
      ```c
      static __latent_entropy void tasklet_action(struct softirq_action *a) {
         struct tasklet_struct *list;

         local_irq_disable();
         list = __this_cpu_read(tasklet_vec.head);
         // 整个链表都会处理完，因此新的链表头被置空，等待新的 tasklet 加入在下一次 tasklet_action 中处理
         __this_cpu_write(tasklet_vec.head, NULL);
         // 结束位置的哨兵 NULL
         __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
         local_irq_enable();

         while (list) {
            struct tasklet_struct *t = list;
            list = list->next;
            if (tasklet_trylock(t)) {
               if (!atomic_read(&t->count)) {
                  if (!test_and_clear_bit(TASKLET_STATE_SCHED, &t->state))
                     BUG();
                  t->func(t->data);
                  tasklet_unlock(t);
                  continue;
               }
               tasklet_unlock(t);
            }

            // 未能成功获取当前 tasklet 的锁
            // 将该未处理的 tasklet 重新加入队列末尾等待处理
            local_irq_disable();
            t->next = NULL;
            *__this_cpu_read(tasklet_vec.tail) = t;
            __this_cpu_write(tasklet_vec.tail, &(t->next));
            __raise_softirq_irqoff(TASKLET_SOFTIRQ);
            local_irq_enable();
         }
      }
      ```
  - 当 `tasklet_schedule(tasklet)` 被调用时，会设置 `tasklet` 的 `state` 为 `TASKLET_STATE_SCHED`，并将 `tasklet` 放入某个链表：
      ```c
      static inline void tasklet_schedule(struct tasklet_struct *t) {
         if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
            __tasklet_schedule(t);
      }

      void __tasklet_schedule(struct tasklet_struct *t) {
         ...
         // 放入队列
         t->next = NULL;
         *__this_cpu_read(tasklet_vec.tail) = t;
         __this_cpu_write(tasklet_vec.tail, &(t->next));
         // 触发 TASKLET 软中断
         raise_softirq_irqoff(TASKLET_SOFTIRQ);
         ...
      }
      ```
  - 调用一次 `tasklet_schedule(tasklet)`，只会导致 `tasklet` 的函数被执行一次，而且不是调用 `tasklet_schedule(...)` 后就马上执行；
  - 如果 `tasklet` 的函数尚未执行，多次调用 `tasklet_schedule(...)` 也是无效的，因为它只会将未调度的 `tasklet` 放入队列一次；

#### 工作队列
如果中断要做的事情实在太耗甚至需要休眠时，那就不能用软中断来做，而应该用**内核线程**来做：在中断上半部唤醒内核线程。内核线程和 APP 都一样竞争执行，APP 有机会执行，系统不会卡顿。

内核线程是系统帮助创建的，一般是 `kworker` 线程，通过下述命令可查看相关的线程信息：
   ```bash
   ps -A | grep kworker
   ```
`kworker` 线程会去“工作队列”（`work queue`）上取出一个一个“工作”（`work`），来执行它里面的函数。

- 工作队列（`workqueue`）
   内核初始化工作队列时，就为它创建了内核线程。要使用工作队列时，只需要把 `work` 放入到 `workqueue` 中，对应的内核线程就会取出 `work` 并执行里面的函数。
   - 内核线程、工作队列（`workqueue`）均由内核创建，开发者只需使用即可；
   - 使用流程：
      1. 构造一个 `work_struct` 结构体，里面有需要执行的函数；
      2. 把这个结构体放入工作队列，内核线程就会运行其中的函数；
- 创建 `work`：
   ```c
   typedef void (*work_func_t)(struct work_struct *work);

   struct work_struct {
      atomic_long_t data;
      struct list_head entry;
      work_func_t func;
   #ifdef CONFIG_LOCKDEP
      struct lockdep_map lockdep_map;
   #endif
   };
   ```
   创建一个函数，然后用这个函数填充一个 `work` 结构体。
   ```c
   // include/linux/workqueue.h
   // 定义一个 work_struct 结构体
   #define DECLARE_WORK(n, f)    struct work_struct n = __WORK_INITIALIZER(n, f);
   // 定义一个 delayed_work 结构体
   // delayed: 意思是要让它在运行时可以指定，即某段时间之后再去执行
   #define DECLARE_DELAYED_WORK(n, f)  struct delayed_work n = __DELAYED_WORK_INITIALIZER(n, f, 0)

   // 初始化 work_struct 结构体
   #define INIT_WORK(_work, _func);

   // aer_recover_work: work 结构体
   // aer_recover_work_func: 需要在工作队列中执行的函数
   static DECLARE_WORK(aer_recover_work, aer_recover_work_func);
   ```
- 将 `work` 结构体提交到工作队列
   ```c
   schedule_work(&aer_recover_work);
   ```
   - `schedule_work` 将 `aer_recover_work` 提供给系统默认的 `work queue`：`system_wq`，它是一个队列；
   - `schedule_work` 同时会将 `kworker` 线程唤醒。此线程抢占到 CPU 资源时，它就会从队列中取出 `work`，执行里面的函数；
   - 可以在中断上半部调用 `schedule_work` 函数；
- 其他 API
   | 函数                                        | 说明                                                                        |
   | ------------------------------------------- | --------------------------------------------------------------------------- |
   | `create_workqueue` | 在 Linux 中已经有了现成的 `system_wq` 等工作队列，不过也可以自己调用该函数创建工作队列。<br />对于 SMP 系统，这个工作队列会有多个内核线程与它对应，创建工作队列时，内核会帮助这个工作队列创建多个内核线程。 |
   | `create_singlethread_workqueue` | 如果想只有一个内核线程与工作队列对应，可以通过该函数创建工作队列。<br />创建工作队列时，内核会帮这个工作队列创建一个内核线程。 |
   | `destroy_workqueue` | 销毁工作队列。 |
   | `schedule_work` | 调度执行一个具体的 `work`，执行的 `work` 将会被挂入 Linux 提供的工作队列。 |
   | `schedule_delayed_work` | 延迟一定时间去执行一个具体任务，功能与 `schedule_work` 类似，多了一个延时。 |
   | `queue_work` | 功能与 `schedule_work` 类似，`schedule_work` 是在系统默认的工作队列上执行一个 `work`，而 `queue_work` 需要自己指定工作队列。 |
   | `queue_delayed_work` | 依上类推。 |
   | `flush_work` | 等待一个 `work` 执行完毕。<br />如果这个 `work` 已经被放入队列，那么该函数等它执行完毕，并返回 `true`；<br />如果这个 `work` 已经执行执行完毕才调用该函数，那么直接返回 `false`。|
   | `flush_delayed_work` | 等待一个 `delayed_work` 执行完毕。<br />如果这个 `delayed_work` 已经被放入队列，那么该函数等它执行完毕，并返回 `true`；<br />如果这个 `delayed_work` 已经执行执行完毕才调用该函数，那么直接返回 `false`。|
- 内部机制
  - `work_struct` 中的函数运行于内核线程的上下文；
  - 在 `2.xx` 版本的 Linux 内核中，创建 `workqueue` 时就会创建内核线程；
      ```c
      // kernel/workqueue.c
      init_workqueues
      keventd_wq = create_workqueue("events")
         __create_workequeue((name), 0, 0)
            for_each_possible_cpu(cpu) {
               err = create_workqueue_thread(cwq, cpu);
                  p = kthread_create(worker_thread, cwq, fmt, wq->name, cpu);
            }
      ```
      - 对于每一个 CPU，都创建一个名为 "events/X" 的内核线程，X 从 0 开始；
      - 在创建 `workqueue` 的同时创建内核线程；
  - 在 `4.xx` 版本的 Linux 内核中，内核线程和 `workqueue` 是分开创建的；
      ```c
      // kernel/workqueue.c
      init_workqueues
      /* initialize CPU pools */
      for_each_possible_cpu(cpu) {
         for_each_cpu_worker_pool(pool, cpu) {
            // 对每一个 CPU 都创建 2 个 worker_pool 结构体，它是含有 ID 的。
            // 一个 worker_pool 对应普通优先级的 work，第二个对应高优先级的 work。
            // ...
         }
      }

      /* create the initial worker */
      for_each_online_cpu(cpu) {
         for_each_cpu_worker_pool(pool, cpu) {
            对每一个 CPU 的每一个 worker_pool，创建一个 worker，每一个 worker 对应一个内核线程
            BUG_ON(!create_worker(pool));
         }
      }

      // 
      static struct worker* create_worker(struct worker_pool *pool) {
         ...
         snprintf(id_buf, sizeof(id_buf), "%d:%d%s", pool->cpu, id, pool->attrs->nice < 0 ? "H" : "");
      }
      // "%d%d": 在哪个 CPU 上运行、poll 中的第几个线程；
      // "H": 高优先级
      // id_buf: 内核线程的名字

      // 创建好内核线程后，再创建 workqueue
      // kernel/workqueue.c
      init_workqueues
      system_wq = alloc_workqueue("events", 0, 0);
         __alloc_workqueue_key
            wq = kzalloc(sizeof(*wq) + tbl_size, GFP_KERNEL);
            alloc_and_link_pwqs(wq)    // 跟 worker_pool 建立联系
      ```
      - 内核线程和工作队列分开创建；
      - 先创建内核线程，再创建 `workqueue`；
      - 一开始每个 `worker_pool` 下只有一个线程，但是系统会根据任务情况动态创建、销毁内核线程；
- 总结
   - 很耗时的中断处理，应该放到线程里去，可以使用 `work`、`work queue`；
   - 在中断上半部调用 `schedule_work` 函数，触发 `work` 的处理；
   - 工作队列在进程上下文执行，将要推后的工作流交给一个**内核线程**去执行；
   - 工作队列**允许睡眠或重新调度**；
   - 缺点：多个 `work` 是在某个内核线程中依序执行的，一旦前面 `work` 执行慢就会影响后面的 `work`（在多 CPU 系统下，一个工作队列可以有多个内核线程，可以在一定程度上缓解这个问题）；

#### threaded irq
使用线程（如 `work`）来处理中断的另一种方式。

以前用 `work` （指工作队列中的任务）来线程化地处理中断，一个 `worker` 线程只能由一个 CPU 执行，多个中断的 `work` 都由同一个 `worker` 线程来处理。
  - 在 `SMP` 系统造成 CPU 资源的浪费；
  - 依序执行，后面 `work` 的执行；

新技术 `threaded irq` 为每一个中断都创建一个内核线程，多个中断的内核线程可以分配到多个 CPU 上执行，从而提高了效率。同时这个内核线程专用于这个中断，避免了依序执行带来的问题。

- 函数原型：
   ```c
   /* 
    * irq：中断号
    * handler: 上半部函数，可以为空
    * thread_fn: 在线程里运行的函数 
    */
   extern int __must_check request_threaded_irq(unsigned int irq, irq_handler_t handler, irq_handler_t thread_fn, unsigned long flags, const char *name, void *dev);
   ```
   - `thread_fn` 和 `handler` 同时提供时，先执行 `handler` 用来处理紧急的事情，`handler` 执行完毕后 Linux 内核会唤醒对应的内核线程，在内核线程里会调用 `thread_fn` 函数；
   - 可以只提供 `hander`，此时退化为一般的 `request_irq` 函数；
   - 可以只提供 `thread_fn`，系统会为这个函数创建一个**内核线程**。发生中断时，内核线程就会执行这个函数。但内核会提供默认的上半部处理函数：`irq_default_primary_handler`，它是直接返回 `IRQ_WAKE_THREAD`；
   - 发生中断时调用 `handler` 函数：
     - 若返回值是 `IRQ_HANDLED`，表示中断处理完成；
     - 若返回值是 `IRQ_WAKE_THREAD`，则唤醒线程；
   - 在 `thread_fn` 中，如果中断被正确处理，应该返回 `IRQ_HANDLED`；
   - **内核线程被唤醒后，执行 `thread_fn` 函数。可能有程序在等待 `thread_fn` 函数被执行，调用 `wake_threads_waitq(desc)` 来唤醒它（可以调用 `synchronize_irq` 来等待 `thread_fn` 被执行）**；
- 函数实现
   ```c
   // kernel/irq/manage.c
   int request_threaded_irq(unsigned int irq, irq_handler_t handler, irq_handler_t thread_fn, unsigned long irqflags, const char *devname, void *dev_id) {
      // 分配、设置一个 irqaction 结构体
      action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
      if (!action)
         return -ENOMEM;
      
      action->handler = handler;
      action->thread_fn = thread_fn;
      action->flags = irqflags;
      action->name = devname;
      action->dev_id = dev_id;

      retval = __setup_irq(irq, desc, action);
   }

   // __setup_irq 部分
   if (new->thread_fn && !nested) {
      ret = setup_irq_thread(new, irq, false);
   }

   // setup_irq_thread 部分
   if (!secondary) {
      t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
   } else {
      t = kthread_create(irq_thread, new, "irq/%d-s-%s", irq, new->name);
      param.sched_priority = -1;
   }
   new->thread = t;
   ```
- 中断执行过程
   以 GPIO 为例，通过调试获取函数调用堆栈（可以参考“中断拆分”章节开始处的调用流程）。最终调用关系： `__handle_irq_event_percpu(...) -> gpio_keys_gpio_isr`。
   ```c
   // kernel/irq/handle.c
   irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc, unsigned int *flags) {
      for_each_action_of_desc(desc, action) {
         irqreturn_t res;
         ...
         res = action->handler(irq, action->dev_id);  // 调用上半部处理函数
         ...
         switch (res) {
         case IRQ_WAKE_THREAD:
            ...
            __irq_wake_thread(desc, action); // 唤醒中断对应的内核线程
            ...
         case ...
         }
      }
   }

   // 线程的处理函数为 irq_thread
   // kernel/irq/handle.c
   while (!irq_wait_for_interrupt(action)) { // 1. 休眠等待中断
      irqreturn_t action_ret;

      irq_thread_check_affinity(desc, action);

      action_ret = handler_fn(desc, action); // 2. 执行 thread_fn
      if (action_ret == IRQ_HANDLED)
         atomic_inc(&desc->threads_handled);
      if (action_ret == IRQ_WAKE_THREAD)
         irq_wake_secondary(desc, action);
      
      wake_threads_waitq(desc);              // 3. 唤醒等到 thread_fn 的线程
   }
   ```

#### 总结
- 软中断（包括 `tasklet`、定时器等）的执行上下文是在（硬件）中断处理的上下文中，因此不允许休眠；
- 线程化的中断处理（如工作队列和 `threaded irq`）的处理环境是内核线程，它与其他线程一样，同样需要竞争 CPU 的使用权，因此在它们的处理函数中是允许休眠的，所在的线程也可以被重新调度；

- [ ] 增加中断下半部几种机制的对比

## 设备树中的中断
- [ ] 补充中断环节的硬件拓扑框图

在硬件上，“中断控制器”只有 `GIC` 这一个，但是在软件上可以把上图中的“GPIO”称为“中断控制器”。很多芯片有多个 GPIO 模块，比如 GPIO1、GPIO2 等等。所以软件上的“中断控制器”就有很多个：GIC、GPIO1、GPIO2 等等。

根据拓扑的连接关系，可以确定当前节点的父节点。比如 GPIO1 连接到 GIC，GIC 是 GPIO1 的父节点。

假设 GPIO1 有 32 个中断源，但是它把其中 16 个汇聚起来向 GIC 发出一个中断。把另外 16 个汇聚起来向 GIC 发出另一个中断。这就意味着 GPIO1 会用到 GIC 的两个中断，会涉及 GIC 里的 2 个 `hwirq`。

这些层级关系、硬件中断号（`hwirq`），都会在设备树中有所体现。

### 设备树中的中断描述
- `interrupt-controller`：表示当前节点为中断控制器（不是只有 `GIC` 才是中断控制器）；
- `#interrupt-cells`：中断控制器下的属性，用以表明引用这个中断控制器的话需要多少个 `cell`。经典取值：
  - `#interrupt-cells=<1>`：使用该中断控制器只需要一个 `cell` 来表明使用“哪一个中断”；
  - `#interrupt-cells=<2>`：使用该中断控制器需要两个 `cell`，一个用来表示使用“哪一个中断”，另一个表示触发类型（第二个 `cell` 的 `bits[3:0]` 用来表示中断触发类型 `trigger type and level flags`）：
    - 1 = `low-to-high edge triggered`，上升沿触发；
    - 2 = `high-to-low edge triggered`，下降沿触发；
    - 4 = `active high level-sensitive`，高电平触发；
    - 8 = `active low level-sensitive`，低电平触发；
  - 示例：
      ```dts
      vic: intc@10140000 {
         compatible = "arm, versatile-vic";
         interrupt-controller;
         #interrupt-cells = <1>;
         reg = <0x10140000 0x1000>;
      }
      ```
      如果中断控制器有级联关系，下级的中断控制器还需要表明它的“`interrupt-parent`” 是谁，用了 `interrupt-parent` 中的哪一个 “`interrupts`”。
- `interrupt-parents`：指定中断控制器（表示当前中断被该控制器控制）；
- `interrupts`：指定中断号（可从芯片手册上查找），触发方式等（根据 `#interrupt-cells` 指定信息），可以多个：
   ```dts
   i2c@7000c000 {
      gpioext: gpio-adnp@41 {
         compatible = "ad, gpio-adnp";

         interrupt-parent = <&gpio>;
         interrupts = <160 1>;

         gpio-controller;
         #gpio-cells = <1>;

         interrupt-controller;
         #interrupt-cells = <2>;
      }
   }
   ```
- `interrupts-extended`：新写法，可以同时指定 `interrupt-parent` 和 `interrupts`：
   ```dts
   interrupts-extended = <&intc1 5 1>, <&intc2 1 0>;
   ```

---
`IMX6ULL` 设备树文件部分源代码：
```dts
intc: interrupt-controller@00a01000 {
   compatible = "arm,cortex-a7-gic";  // drivers/irqchip/irq-gic.c
   #interrupt-cells = <3>;
   interrupt-controller;
   reg = <0x00a01000 0x1000>,
         <0x00a02000 0x100>;
};

soc {
   #address-cells = <1>;
   #size-cells = <1>;
   compatible = "simple-bus";
   interrupt-parent = <&gpc>;       // 在 soc 节点下的子节点其中断父节点均是 gpc

   gpc: gpc@020dc000 {
      compatible = "fsl,imx6ul-gpc", "fsl,imx6q-gpc"; // arch/arm/mach-imx/gpc.c
      reg = <0x020dc000 0x4000>;
      interrupt-controller;
      #interrupt-cells = <3>;
      interrupts = <GIC_SPI 89 IRQ_TYPE_LEVEL_HIGH>;
      interupt-parent = <&intc>;
      fsl,mf-mix-weakup-irq = <0xfc00000 0x7d00 0x0 0x1400640>;
   };

   gpio1: gpio@0209c000 {
      compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio"; // drivers/gpio/gpio-mxc.c
      reg = <0x0209c000 0x4000>;
      interrupts = <GIC_SPI 66 IRQ_TYPE_LEVEL_HIGH>,
                   <GIC_SPI 67 IRQ_TYPE_LEVEL_HIGH>;
      gpio-controller;
      #gpio-cells = <2>;
      interrupt-controller;
      #interrupt-cells = <2>;
   };
// 以上都是厂家 BSP 工程师实现的（imx6ull.dtsi）

   spidev: icm20608@0 {
      compatible = "invensense,icm20608"; // drivers/iio/imu/inv_mpu6050/inv_mpu_spi.c
      interrupt-parent = <&gpio1>;
      interrupts = <1 1>;
      spi-max-frequency = <8000000>;
      reg = <0>;
   };
}
```
- 根据设备树反推 `IMX6ULL` 的中断拓扑，比之前的框图多了一个 `GPC INTC`：
  - [ ] 补充 `IMX6ULL` 的中断拓扑
   - `GPC INTC`: `General Power Controller, Interrupt Controller`，它提供中断屏蔽、中断状态查询功能；
   - 实际上这些功能在 `GIC` 里也实现了；
   - 它还提供唤醒功能，这才是保留它的原因；

### 在代码中获取设备树中的中断信息
1. 对于 `platform_device`
   一个节点能被转换为 `platform_device`，如果它的设备树里指定了中断属性，那么可以从 `platform_device` 中获得中断资源：
   ```c
   // 通过下述函数获得 IORESOURCE_IRQ 资源，即中断号
   // get a resource for a device
   /* 
    * dev: platform device 
    * type: resource type, eg: IORESOURCE_MEM, IORESOURCE_REG, IORESOURCE_IRQ 等
    * num: resource index, 该类资源中的哪一个
    */
   struct resource *platform_get_resource(struct platform_device *dev, unsigned int type, unsigned int num);
   ```

2. 对于 `I2C`、`SPI` 设备
   对于 `I2C` 设备节点，`I2C` 总线驱动在处理设备树中的 `I2C` 子节点时，也会处理其中的中断信息。一个 `I2C` 设备会被转化为一个 `i2c_client` 结构体，中断号会被保存在 `i2c_client` 的 `irq` 成员里（`drivers/i2c/i2c-core.c`）。

   对于 `SPI` 设备节点，`SPI` 总线驱动在处理设备树中的 `SPI` 子节点时，也会处理其中的中断信息。一个 `SPI` 设备会被转化为一个 `spi_device` 结构体，中断号会被保存在 `spi_device` 的 `irq` 成员里（`drivers/spi/spi.c`）。

3. 调用 `of_irq_get` 获得中断号
   如果设备节点既不能转化为 `platform_device`，它也不是 `I2C` 设备，不是 `SPI` 设备，那么在驱动程序中可以调用 `of_irq_get` 函数去解析设备树，从而得到中断号。

4. 对于 GPIO
   参考：`drivers/input/keyboard/gpio_keys.c`
   可以使用 `gpio_to_irq` 或 `gpiod_to_irq` 获取中断号：
   ```dts
   gpio-keys {
      compatible = "gpio-keys";
      pinctrl-names = "default";

      user = {
         label = "User Button";
         gpios = <&gpio5 1 GPIO_ACTIVE_HIGH>;
         gpio-key,wakeup;
         linux,code = <KEY_1>;
      }
   }
   ```
   那么可以通过下面的方式获得中断号：
   ```c
   // 获得引脚和 flag
   button->gpio = of_get_gpio_flags(pp, 0, &flags);
   bdata->gpiod = gpio_to_desc(button->gpio);
   // 获得中断号
   irq = gpiod_to_irq(bdata->gpiod);
   ```

`irq_of_parse_and_map`
`gpio_to_irq`


## 中断子系统

### 子系统 API
- [ ] 补充中断子系统相关 API

### 中断的处理
- [ ] 添加中断 pipeline
- [ ] 添加各个数据结构的关系图

- 中断的输入：多个外部设备可以共享一个 GPIO 中断（例如中断号 B），多个 GPIO 中断汇聚到 GIC（通用中断控制器）（例如中断号 A），GIC 再去中断 CPU；
- 中断的处理：
  1. 先读取 GIC 获得中断号 A：
     GIC 中断 CPU 时，CPU 读取 GIC 状态得到中断号并执行 GIC 处理函数：假设 `irq_desc[A].handle_irq` 是 `xxx_gpio_irq_handler`（`xxx` 指厂家）；
  2. 再细分出 GPIO 中断 B：
     `xxx_gpio_irq_handler` 读取芯片的 GPIO 控制器获取发生的是哪一个 GPIO 中断（假设为 B）。`irq_desc[A].handle_irq` 调用对应的 `irq_desc[B].handle_irq`；
  3. 最后判断是哪一个外部设备发生了中断：
     `irq_desc[B].handle_irq`，BSP 开发人员会设置对应的处理函数，一般是 `handle_level_irq` 或 `handle_edge_irq`。导致 GPIO 中断 B 发生的原因很多，可能是外部设备 1，可能是外部设备 n，可能只是某一个设备，也有可能是多个设备。**所以 `irq_desc[B].handle_irq` 会调用某个链表里面的函数，这些函数由外部设备提供，它们能够自行判断该中断是否是自己产生，若是则处理**。
- 对于共享中断，比如 GPIO 中断 B，它的中断来源可能有多个，每个中断源对应一个中断处理函数。所以 `irq_desc[B]` 应该有一个链表，存放着多个中断源的处理函数。**一旦确定发生了 GPIO 中断 B，那么就会从链表里把这些函数取出来一一执行，这个链表就是 `action` 链表**。

### 数据结构
可以从函数 `request_irq` 一路分析得到。位置：`include/linux/interrupt.h`。

#### `struct irq_desc`
最核心的结构体是 `struct irq_desc`，在 Linux 内核中有一个中断数组，对于每一个硬件中断都有一个数组项，这个数组就是 `irq_desc` 数组。
- [ ] 具体 SOC 的汇编代码（例如中断向量表）如何与内核中的这个中断数组产生联系的呢？
   - 注意：如果内核配置了 `CONFIG_SPARSE_IRQ`，则内核会用 `radix tree`（基数树）来替代 `irq_desc` 数组（`sparse`-稀疏）。使用基数树替代数组来避免资源浪费。

- 数据结构代码
```c
// include/linux/irqdesc.h
struct irq_desc {
   struct irq_data irq_data;
   irq_flow_handler_t handle_irq;
   const char *name;
   struct irqaction *action;
   // ...
};
```

#### `struct irqaction`
- 数据结构代码
   ```c
   // include/linux/interrupt.h
   struct irqaction {
      const char *name;
      void *dev_id;
      unsigned int irq;
      unsigned int flags;
      irq_handler_t handler;
      irq_handler_t thread_fn;
      struct task_struct *thread;
      struct irqaction *secondary;
      struct irqaction *next;
      //...
   };
   ```
  - 在 `irqaction` 中保存了 `name`、`dev_id` 等内容，最重要的是 `handler`、`thread_fn`、`thread`；
  - `handler` 是中断处理的上半部函数，用来处理紧急的事情；
  - `thread_fn` 对应一个内核线程 `thread`。在内核线程里会调用 `thread_fn` 函数；
- 当调用 `request_irq`、`request_threaded_irq` 注册中断处理函数时，内核就会构造一个 `irqaction` 结构体；
- `dev_id` 的作用：
   在调用 `request_irq` 和 `request_threaded_irq` 函数时传入 `void *dev` 参数，`dev_id` 的作用有：
   - 中断处理函数执行时，可以使用 `dev_id`；
   - 写在中断时要传入 `dev_id`，这样才能在 `action` 链表中（位于 `struct irq_desc`）根据 `dev_id` 找到对应项；
   - 因此在共享中断中必须提供 `dev_id`，非共享中断可以不提供；

#### `struct irq_data`
作为中转站，里面有 `irq_chip` 指针和 `irq_domain` 指针，都指向别的结构体；

- 数据结构代码
   ```c
   // include/linux/irq.h
   struct irq_data {

   };
   ```
  - `irq` 表示中断号；
  - `hwirq` 表示硬件中断号；

#### `struct irq_domain`
建立硬件中断号 `hwirq` 和中断号 `irq` 之间的映射关系。`irq_domain` 会把本地的 `hwirq` 映射为全局的 `irq`。比如 GPIO 控制器里有第 1 号中断，UART 模块里也有第 1 号中断，这两个“第 1 号中断”是不一样的，它们属于不同的“域”——`irq_domain`。

- 数据结构代码
   ```c
   // include/linux/irqdomain.h
   struct 
   ```

- 与设备树的联系
   如何在设备树中指定中断？设备树的中断如何被转换为 `irq`？`irq_domain` 将会在这个过程中起到极大的作用。
   eg:
   1. 设备树源码
      ```dts
      interrupt-parent = <&gpio1>;
      interrupts = <5 IRQ_TYPE_EDGE_RISING>;
      ```
         - 表示要使用 `gpio1` 里的第 5 号中断，`hwirq` 就是 5；
    2. 在驱动中会使用 `request_irq(irq, handler)` 方式来注册中断。其中，`irq` 即中断号，需要从“`gpio1` 的第 5 号中断”转换得来；
    3. 由 `gpio1` 相关的数据结构将 `hwirq` 转换为 `irq`，就是 `gpio1` 对应的 `irq_domain` 结构体；

- `hwirq -> irq`
   `irq_domain` 结构体中有一个 `irq_domain_ops` 结构体，里面有各种操作函数：
   - `xlate`：用来解析设备树的中断属性，提取出 `hwirq`、`type` 等信息；
   - `map`：把 `hwirq` 转换 irq；

#### `struct irq_chip`
与芯片相关，用来保存具体芯片的中断操作函数，如 `void (*irq_enable)(struct irq_data *data)` 等。是中断子系统提供的统一 API。

- 数据结构代码
   ```c
   // include/linux/irq.h
   ```

- 在 `request_irq` 后，并不需要手动去使能中断，原因是系统调用对应的 `irq_chip` 里面的函数完成了此工作；同理，在提供的中断处理函数中，不需要执行主芯片相关的清中断操作，也是系统调用了 `irq_chip` 中的函数；

- 对于外部设备相关的清中断操作，还是需要手动完成；

## 使用

### 中断在驱动中的示例
- [ ] 补充相关设备树到驱动程序的代码

### 注意事项
1. 中断里面可以使用自旋锁，但是使用之前（在获取锁之前）一定要先禁止本地中断（本 cpu 中断），否则可能导致死锁的发生。

## 参考
1. 韦东山-《嵌入式 Linux 应用开发完全手册 V5.2_IMX6ULL_Pro 开发板》
2. 正点原子-《I.MX6U 嵌入式 Linux 驱动开发指南 V1.81》
3. 内核文档：`Documentation/devicetree/bindings/interrupt-controller/interrupts.txt`