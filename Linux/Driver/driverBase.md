# Linux 驱动开发基石
## 阻塞与非阻塞 I/O

- `poll`：使用 `poll` 时，如果传入的超时时间不为 0，这种访问方式也是阻塞的。如果超时时间为 0，这样即使没有数据它也会立即返回，属于非阻塞方式。

- 可以让 `read` 函数既能工作于阻塞方式，也可以工作于非阻塞方式：
  - 默认是阻塞方式；
  - 改为非阻塞方式的方法：
    1. APP 调用 `open` 函数时，传入 `O_NOBLOCK`；
    2. 在 `open` 函数之后，通过 `fcntl` 修改为非阻塞方式（当然也可以修改为阻塞方式）；
   ```c
   // open 之前设置
   int fd = open("/dev/xxx", O_RDWR | O_NOBLOCK);  // 非阻塞方式
   int fd = open("/dev/xxx", O_RDWR);              // 阻塞方式
   // open 之后设置
   int flags = fcntl(fd, F_GETFL);
   fcntl(fd, F_SETFL, flags | O_NOBLOCK);          // 非阻塞方式
   fcntl(fd, F_SETFL, flags & ~O_NOBLOCK);         // 阻塞方式
   ```

- 注意：
  - 对于普通文件、块设备文件，`O_NOBLOCK` 不起作用；
  - 对于字符设备文件，`O_NOBLOCK` 起作用的前提是驱动程序针对 `O_NOBLOCK` 做了处理；

### 休眠与唤醒

**适用场景**：当应用程序必须等待某个事件发生时可以适用“休眠-唤醒”机制。

在休眠与唤醒机制中当 APP 调用 `read` 等函数试图获取数据时：
- 当驱动中有数据时立即返回结果；
- 当驱动中没有数据时，APP 休眠（被挂起，运行状态改为非 `RUNNING`，内核调度器不会让它运行）。APP 在内核态被阻塞。当有数据产生时唤醒被阻塞的 APP，继续执行剩下的代码。

可以在中断服务程序中将 APP 唤醒，它不属于 APP 的上下文，不能在处理过程中休眠（也不能调用会导致休眠的函数）。

1. 休眠函数
   参考 `include/linux/wait.h`
   | 函数                                        | 说明                                                                        |
   | ------------------------------------------- | --------------------------------------------------------------------------- |
   | `wait_event_interruptible(wq, condition)` | 休眠，直到 `condition` 为真；<br />休眠期间是可以被打断的，可以被信号打断 |
   | `wait_event(wq, condition)` | 休眠，直到 `condition` 为真；<br />退出的唯一条件是 `condition` 为真，信号也不好使 |
   | `wait_event_interruptible_timeout(wq, condition, timeout)` | 休眠，直到 `condition` 为真或超时；<br />休眠期间是可以被打断的，可以被信号打断 |
   | `wait_event_timeout(wq, condition, timeout)` | 休眠，直到 `condition` 为真或超时；<br />不能被打断，信号也不好使 |
   - `wq`：`waitqueue`，等待队列。休眠时除了把程序状态改为非 `RUNNING` 之外，还要把进程放入 `wq` 中，以后中断服务程序要从 `wq` 中把它取出来唤醒；
   - `condition`：可以是一个变量，也可以是任何表达式。表示“一直等待，直到 `condition` 为真”
2. 唤醒函数
   参考 `include/linux/wait.h`
   | 函数                                        | 说明                                                                        |
   | ------------------------------------------- | --------------------------------------------------------------------------- |
   | `wake_up_interruptible(x)` | 唤醒 `x` 队列中状态为 `TASK_INTERRUPTIBLE` 的线程，只唤醒其中的一个线程 |
   | `wake_up_interruptible_nr(x, nr)` | 唤醒 `x` 队列中状态为 `TASK_INTERRUPTIBLE` 的线程，只唤醒其中的 `nr` 个线程 |
   | `wake_up_interruptible_all(x, nr)` | 唤醒 `x` 队列中状态为 `TASK_INTERRUPTIBLE` 的线程，唤醒其中的所有线程 |
   | `wake_up(x)` | 唤醒 `x` 队列中状态为 `TASK_INTERRUPTIBLE` 或 `TASK_UNINTERRUPTIBLE` 的线程，只唤醒其中的一个线程 |
   | `wake_up_nr(x, nr)` | 唤醒 `x` 队列中状态为 `TASK_INTERRUPTIBLE` 或 `TASK_UNINTERRUPTIBLE` 的线程，只唤醒其中的 `nr` 线程 |
   | `wake_up_all(x)` | 唤醒 `x` 队列中状态为 `TASK_INTERRUPTIBLE` 或 `TASK_UNINTERRUPTIBLE` 的线程，唤醒其中的所有线程 |
3. 驱动示例
   - 初始化 `wq` 队列；
   - 在驱动的 `read` 函数中，调用 `wait_event_interruptible`（它本身会判断 `event` 是否为 `FALSE`。如果为 `FALSE` 表示无数据，则休眠。因此该函数的调用并不一定导致休眠。）；
   - 当从 `wait_event_interruptible` 返回后，把数据复制回用户空间；
   - 在中断服务程序中：设置 `event` 为 `TRUE`，并调用 `wake_up_interruptible` 唤醒线程；
   - [ ] 补充相关示例代码（环形缓冲区）
   ```c
   #define BUF_LEN 128
   static int g_keys[BUF_LEN];
   static int r, w;

   #define NEXT_POS(x) ((x + 1) % BUF_LEN)

   static int is_key_buf_empty(void) {
      return r == w;
   }

   static int is_key_buf_full(void) {
      return (r == NEXT_POS(w));
   }

   static void put_key(int key) {
      if (!is_key_buf_full()) {
         g_keys[w] = key;
         w = NEXT_POS(w);
      }
   }

   static int get_key(void) {
      int key = 0;
      if (!is_key_buf_empty()) {
         key = g_keys[r];
         r = NEXT_POS(r);
      }
      return key;
   }
   ```
   考虑 `O_NOBLOCK` 方式的实现：
   ```c
   static ssize_t drv_read(struct file *fp, char __user *buf, size_t count, loff_t *ppos) {
      if (queue_empty(&as->queue) && fp->f_flags & O_NOBLOCK) {
         return -EAGAIN;
      }

      wait_event_interruptible(apm_waitqueue, !queue_empty(&as->queue));
      ...
   }
   ```
4. 应用程序使用
   - [ ] 补充相关内容

### POLL 机制
**适用场景**：阻塞 + 超时；

APP 不知道驱动程序中是否有数据，可以先调用 `poll` 函数查询以下，`poll` 函数可以传入超时时间。

APP 调用 `poll` 函数后，进入到内核态中驱动的 `poll` 函数：
- 如果有数据的话立即返回；
- 如果没有数据就休眠直到超时时间（内核唤醒 APP）或者在这之前已收到数据（中断服务程序被调用，在其中接收数据并唤醒 APP）；

APP 根据 `poll` 函数的返回值就可以知道是否有数据，如果有则可以调用 `read` 函数获取数据。

1. 从应用到驱动的 poll 流程
- [ ] 补充示意图

2. 驱动示例

3. 应用程序使用

4. 内核 poll 机制详解

## 异步通知
**适用场景**：使用“休眠-唤醒”、“POLL 机制”时都需要休眠等待某个事件发生，两者的区别在于后者可以指定休眠的时长。如果 APP 不想休眠，也可以通过异步通知机制使得驱动程序在有数据时主动通知 APP，APP 收到信号后执行处理函数。

驱动利用内核中提供的函数发送信号给 APP，APP 需要把自己告诉驱动并注册信号处理函数。

1. 信号类型
   参考 `include/uapi/asm-generic/signal.h`
   ```c
   #define SIGHUP    1
   #define SIGINT    2
   #define SIGQUIT   3
   ...
   #define SIGIO     29    // 驱动常用信号，表示有 I/O 事件
   #define SIGPOLL   SIGIO
   ```
2. APP 任务
   - 注册信号处理函数
      ```c
      // #include <signal.h>
      typedef void (*sighandler_t)(int);     // APP 需要先实现的处理函数类类型

      /* 绑定信号 signum 和信号处理函数 hander 以完成注册 */
      sighandler_t signal(int signum, sighandler_t handler);
      ```
   - 打开驱动程序对应的设备节点，并使能驱动程序的异步通知功能；
   - 将自己的进程 ID 告诉驱动程序；
   - 意愿（什么时候接收信号、什么时候不收信号）；
3. 驱动程序任务
   - 当 APP 设置进程 ID 时记录；
   - 驱动有对应的异步通知函数 `fasync`（APP 打开驱动程序时，内核会创建对应的 `file` 结构体，`file` 中有 `f_flags`，其中有一个 `FASYNC` 位，当它被置位时表示使能异步通知功能。当 `f_flags` 中的 `FASYNC` 位发生变化时，驱动程序的 `fasync` 函数被调用）；
   - **发生中断**并检测到数据时，驱动程序调用内核辅助函数（`kill_fasync`）发送信号；
4. 整体流程
   - [ ] 补充异步通知时整体流程，包括 APP 端和驱动端
5. 驱动示例
   - 提供对应的 `drv_fasync` 函数（调用 `fasync_helper` 函数就可以）：
      ```c
      static struct fasync_struct *button_async;

      static int drv_fasync(int fd, struct file *filp, int on) {
         return fasync_helper(fd, filp, on, &button_async);
      }
      ```
     - `fasync_helper` 函数会分配、构造一个 `fasync_struct` 结构体；
     - 驱动文件的 `flag` 的 `FASYNC` 被置位时：`button_async->fa_file = filp`；
     - 驱动文件的 `flag` 的 `FASYNC` 被清空时 `button_async->fa_file = NULL`；
     - 想要发送信号时，使用 `button_async` 作为参数就可以，它里面**可能**含有 `PID`；
   - 在合适的时机发信号：
      ```c
      // 示例在 GPIO 的中断服务程序中发送信号
      kill_fasync(&button_async, SIGIO, POLL_IN);
      ```
      - `button_async->fa_file` 非空时，可以从中获取 `PID`，表示发送给哪一个 APP；
      - `SIGIO`：信号类型；
      - `POLL_IN`：信号值（APP 用不到这个参数）；
6. 应用程序使用
   - 编写信号处理函数：
      ```c
      static void sig_func(int sig) {
         int val;
         read(fd, &val, 4);
         printf("get button: 0x%x\n", val);
      }
      ```
   - 注册信号处理函数：
      ```c
      signal(SIGIO, sig_func);
      ```
   - 打开驱动对应的设备文件：
      ```c
      fd = open(argv[1], O_RDWR);
      ```
   - 把进程 ID 告诉驱动：
      ```c
      fcntl(fd, F_SETOWN, getpid());
      ```
   - 置位驱动的 `FASYNC` `flag`：
      ```c
      flags = fcntl(fd, F_GETFL);
      fcntl(fd, F_SETFL, flags | FASYNC);
      ```
7. 内核异步通知机制详解
   - APP 执行 `fcntl` 系统调用时，会导致内核中 `fs/fcntl.c` 文件下的如下函数被调用：
      ```c
      // 位于 fs/fcntl.c
      // SYSCALL_DEFINE3 宏定义展开后可生成 sys_fcntl 标识符
      SYSCALL_DEFINE3(fcntl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
      {	
         struct fd f = fdget_raw(fd);
         long err = -EBADF;

         if (!f.file)
            goto out;

         if (unlikely(f.file->f_mode & FMODE_PATH)) {
            if (!check_fcntl_cmd(cmd))
               goto out1;
         }

         err = security_file_fcntl(f.file, cmd, arg);
         if (!err)
            err = do_fcntl(fd, cmd, arg, f.file);

      out1:
         fdput(f);
      out:
         return err;
      }
      ```
   - 对于 `F_SETOWN` 的调用，从上述代码中的 `do_fcntl(...)` 可以追溯到 `f_modown(...)` 函数，在该函数中会在 `struct file *filp` 中记录 `PID`：
      ```c
      static void f_modown(struct file *filp, struct pid *pid, enum pid_type type, int force) {
         ...
         filp->f_owner.pid = get_pid(pid);
      }
      ```
   - 对于 `F_GETFL` 的调用，从上述代码中的 `do_fcntl(...)` 可以追溯到仅仅是返回 `filp->f_flags`；
   - 对于 `F_SETFL` 的调用，从上述代码中的 `do_fcntl(...)` 可以追溯到 `setfl(...)` 函数：
      ```c
      static long do_fcntl(int fd, unsigned int cmd, unsigned long arg, struct file *filp) {
         long err = -EINVAL;

         switch (cmd) {
         case F_DUPFD:
            ...
         case F_GETFL:
            err = filp->f_flags;
            break;
         case F_SETFL:
            err = setfl(fd, filp, arg);
            break;
         }
      }
      ```
      在 `setfl(...)` 函数中会比较 `filp->f_flags` 中的 `FASYNC` 位，若发现它**发生变化**了则调用驱动的 `fasync` 函数：
      ```c
      if (((arg ^ filp->f_flags) & FASYNC) && filp->f_op->fasync) {
         error = filp->f_op->fasync(fd, filp, (arg & FASYNC) != 0);
         ...
      }
      ```
      示例驱动程序：
      ```c
      static int gpio_key_drv_fasync(int fd, struct file *file, int on) {
         if (fasync_helper(fd, file, on, &button_fasync) >= 0) {
            return 0;
         } else {
            return -EIO;
         }
      }
      ```
      `fasync_helper` 程序经过精简后可表示为：
      ```c
      if (on) {
         struct fasync_struct *new;
         new = fasync_alloc();
         new->magic = FASYNC_MAGIC;
         new->fa_file = filp;
         new->fa_fd = fd;
         button_fasync = new;
      } else {
         kfree(button_fasync);
         button_fasync = NULL;
      }
      ```
      因此，`FASYNC` 位发生变化且变为置位时，驱动程序的 `button_fasync` 就被设置了，它指向的 `fasync_struct` 结构体中含有 `filp`，而 `filp` 中又含有之前设置的接收方 `PID`。
   - 在驱动程序中调用 `kill_fasync()` 函数后：
      ```c
      kill_fasync(&button_fasync, SIGIO, POLL_IN);
      ```
      它的核心就是从 `button_fasync` 指针中取出 `fasync_struct` 结构体，从而由该结构体得到接收方的 `PID`，最后使用 `send_sigio()` 函数发送信号。
      ```c
      void kill_fasync(struct fasync_struct **fp, int sig, int band) {
         ...
         kill_fasync_rcu(rcu_dereference(*fp), sig, band);
         ...
      }

      static void kill_fasync_rcu(struct fasync_struct *fa, int sig, int band) {
         ...
         struct fown_struct *fown;
         ...
         fown = &fa->fa_file->f_owner;    // app 调用 F_SETOWN 时设置
         ...
         send_sigio(fown, fa->fa_fd, band);  // 发信号
      }
      ```
      `send_sigio` 函数的实质是：根据 `PID` 找到进程在内核的 `task_struct` 结构体，修改里面的某些成员表示收到了信号。
   - APP 收到信号后，处理函数被调用相当于 APP 的中断，处理过程和中断处理过程类似：保存现场、处理信号、恢复现场。
     - APP 进入内核态时，内核在 APP 的栈里保存 APP 的运行环境；
     - APP 退出内核态时，内核从 APP 的栈里恢复 APP 的运行环境；
     - APP 收到信号瞬间处于内核态。因为 `send_sigio` 函数要么由驱动程序调用，要么由 APP 通过系统调用来间接调用，而 `send_sigio` 函数运行时处于内核态；
     - 由内核态返回到用户态时，内核发现 APP 有信号等待处理就会修改 APP 的栈，增加一个新的运行环境。在该环境中运行地址是信号处理函数的地址，于是返回到用户态后开始执行的是信号处理函数；
     - 信号处理函数执行完后，会再次返回到内核态，在内核态里再使用旧的运行环境，从而在再次返回用户态时 APP 得以恢复运行；

## 定时器
涉及到两个要素：
   - 超时时间；
   - 函数；

1. API
   参考 `include/linux/timer.h`
   - `setup_timer(time, fn, data)`：
     - 设置定时器，主要是初始化 `timer_list` 结构体，设置其中的函数、参数；
   - `void add_timer(struct timer_list *timer)`：
     - 向内核添加定时器。`timer->expires` 表示超时时间；
     - 当超时时间达到，内核就会调用这个函数：`timer->function(timer->data)`；
   - `int mod_timer(struct timer_list *timer, unsigned long expires)`：
     - 修改定时器的超时时间；
     - 它等同于 `del_timer(timer); timer->expires = expires; add_timer(timer);`，但是更高效；
   - `int del_timer(struct timer_list *timer)`：删除定时器；
2. 定时器时间单位
   - 编译内核时，可以通过 `CONFIG_HZ` 来修改系统滴答（`tick`），100 时表示 10ms 发生一次系统滴答中断。每发生一次 `tick` 中断，全局变量 `jiffies` 就会累加 1；
      ```
      CONFIG_HZ=100
      ```
   - 定时器的时间就是基于 `jiffies` 的；
   - 修改超时时间时，一般使用如下两种方式：
      ```c
      // 在 add_timer 之前，直接修改
      timer.expires = jiffies + xxx;         // xxx 表示多少滴答后超时，也就是 xxx * 10ms
      timer.expires = jiffies + 2 * HZ;      // HZ 等于 CONFIG_HZ，2 * HZ 就相当于 2 秒

      // 在 add_timer 之后，使用 mod_timer 修改
      mod_timer(&timer, jiffies + xxx);
      mod_timer(&timer, jiffies + 2 * HZ);
      ```
3. 定时器内部机制
   - 每发生一次硬件中断时，硬件中断处理完后就会看看有没有软件中断（中断下半部）要处理。
   - **定时器中断就是通过软件中断来实现的**，它属于 `TIMER_SOFTIRQ` 类型的软中断；
   - 软中断（包括下半部中断中的 `tasklet`，以及这里的定时器）处理函数的执行上下文均为中断所在的上下文，因此在该函数中不允许休眠；
   - 对于 `TIMER_SOFTIRQ` 软中断，初始化代码如下：
      ```c
      void __init init_timers(void) {
         init_timer_cpus();
         init_timer_stats();
         open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
      }
      ```
   - 在硬件中断后，内核调用软件中断的处理函数。对于 `TIMER_SOFTIRQ`，会调用 `run_timer_softirq`：
      ```
      run_timer_softirq
         __run_timer(base);
            while (timer_after_eq(jiffies, base->clk)) {
               ...
            }
      expire_timers(base, heads + levels);
         fn = timer->function;
         data = timer->data;
         call_timer_fn(timer, fn, data);
            fn(data);
      ```
      - `add_timer(...)` 函数会把 `timer` 放到内核中的某个链表（应该是按照超时时间排序的有序链表）；
      - 在 `TIMER_SOFTIRQ` 的处理函数中，会从链表中把这些**超时的** `tiemr` 取出来，执行其中的函数（`timer->expires` 大于等于 `jiffies` 时表示超时）；
4. 找到系统滴答
   ```bash
   cat /proc/interrupts
   ```
   - 输出的内容中有一个数值变化特别块，它就是系统滴答中断；
   
   以 `IM6ULL` 为例，滴答中断名字就是 `i.MX Timer Tick`。在 Linux 内核源码目录下执行以下命令：
   ```bash
   grep "i.MX Timer Tick" * -nr
   ```
   - 得到输出： `drivers/clocksource/timer-imx-gpt.c:319 act->name = "i.MX Timer Tick"`；

   打开 `timer-imx-gpt.c`：
   ```c
   act->name = "i.MX Timer Tick"；
   act->flags = IRQF_TIMER | IRQF_IRQPOLL;
   act->handler = mxc_timer_interrupt;
   act->dev_id = ced;

   return setup_irq(imxtm->irq, act);
   ```
   - `mxc_timer_interrupt` 应该就是滴答中断的处理函数：
      ```c
      static irqreturn_t mxc_timer_interrupt(int irq, void *dev_id) {
         struct clock_event_device *ced  = dev_id;
         struct imx_timer *imxtm = to_imx_timer(ced);
         uint32_t tstat;

         tstat = readl_relaxed(imxtm->base + imxtm->gpt->reg_tstat);

         imxtm->gpt->gpt_irq_acknowledge(imxtm);

         ced->event_handler(ced);

         return IRQ_HANDLED;
      }
      ```
   - 通过调试找到 `event_handler(...)` 对应 `tick_handle_periodic`：
      ```c
      // kernel/time/tick-common.c
      tick_handle_periodic
         tick_periodic(cpu)
            do_timer(1)
               jiffies_64 += ticks;    // jiffies 就是 jiffies_64
      
      // arch/arm/kernel/vmlinux.lds.S
      #ifndef __ARMEB__
      jiffies = jiffies_64;
      #else
      jiffies = jiffies_64 + 4;
      #endif
      // 对 jiffies_64 的累加操作，就是对 jiffies 的累加操作
      ```
      - [ ] 上述代码说明了，对于大字节序的 CPU，`jiffies` 指向 `jiffies_64` 的高 4 字节；对于小字节序的 CPU，`jiffies` 指向 `jiffies_64` 的低 4 字节。

## 中断处理
分为上、下半部：
   - 在上半部处理紧急的事情，在上半部的处理过程中，中断是被禁止的；
   - 在下半部处理耗时的事情，在下半部的处理过程中，中断是使能的。

上半部即硬件中断，即通过 `request_irq` 完成注册的中断服务程序。它与裸机的中断服务程序类似，要求快进快出，但不同的是在 Linux 中要求它不能嵌套。因此在实现时通常先关中断，处理完成后开中断。上半部的内容不多，这里主要涉及的是对中断下半部的处理。

### 中断下半部 tasklet
见 [subsystem-interrupt.md]() 相关章节。

### 工作队列
见 [subsystem-interrupt.md]() 相关章节。

### 中断的线程化处理
见 [subsystem-interrupt.md]() 相关章节。

## mmap
1. 作用：
   应用程序和驱动之间传递数据时，可以通过 `read/write`（`copy_from_user/copy_to_user`）函数进行，这涉及在用户态 `buffer` 和内核态 `buffer` 之间传送数据。这种方式在数据量较大或者操作频繁时存在效率低的问题：
   - 数据在 `buffer` 之间的拷贝耗时；
   - 用户态和内核态之间频繁的状态切换导致的耗时；

   改进方式就是让程序可以直接读写驱动程序中的 `buffer`，这可以通过 `mmap` 实现。`mmap` 能够把内核的 `buffer` 直接映射到用户态，从而让 APP 在用户态直接读写。

2. 虚拟地址与 MMU
   背景：通过运行同一程序两次，并在程序中打印全局变量的地址和值，打印结果是地址一样，值可以不一样。

   **CPU 发出的地址是虚拟地址**，它经过 `MMU, Memory Manage Unit`（内存管理单元）映射到物理地址上。对于不同进程的同一个虚拟地址，`MMU` 会把它们映射到不同的物理地址。（上述程序打印结果中的地址是虚拟地址）
   - [ ] 添加虚拟地址到物理地址的转换示意图

3. 虚拟地址与物理地址的转换
   查看某一进程的映射关系：
   ```bash
   # 通过 ps 命令查看进程 ID
   ps
   # 查看目标 ID 的映射关系，325 即目标进程的 PID
   cat /proc/325/maps
   ```
   在 Linux 中进程使用 `struct task_strcut` 表示，每一个 APP 在内核里都有一个 `task_struct`。这个结构体保存有内存信息：`mm_struct`，在 `mm_struct` 中通过 `pgd`(`Page Global Directory`) 字段保存页目录表信息（含有虚拟地址和物理地址的映射关系）以及 `mmap` 字段保存虚拟地址的使用情况。
   - [ ] 添加由 `task_struct` 出发得到的与地址转换相关的数据结构
     - 每个 APP 在内核中都有一个 `task_struct` 结构体，它用来描述一个进程；
     - `task_struct` 中用 `mm_struct` 来管理进程占用的内存；
     - `mm_struct` 中用 `mmap` 字段来记录虚拟地址，用 `pgd` 来描述虚拟地址和物理地址之间的映射关系；
  
   `mmap` 字段对应的数据类型为 `struct vm_area_struct`：
      ```c
      struct vm_area_struct {
         vm_start;
         vm_end;
         vm_flags;
         vm_next;
         ...
      }
      ```
      - 每个 APP 都有一系列的虚拟内存，比如 APP 含有代码段、数据段、BSS 段、栈以及共享库等等。这些单元会保存在内存里，它们的地址空间不同，权限不同（代码段是只读、可运行的；数据段是可读可写的），内核用一系列 `vm_area_struct` 来描述它们；
      - `vm_start` 和 `vm_end` 字段记录的是虚拟地址；
      - 虚拟地址的转换需要借助上述 `mm_struct` 中的 `pgd` 字段；

4. ARM 架构中的内存映射
   ARM 架构支持一级页表映射，还支持二级页表映射。一级页表里地址映射的最小单位是 1M，二级页表地址映射的最小单位有 4K、1K，Linux 使用 4K。

   将页表细分的目的是减少内存浪费（内内存碎片）。

   一级**页表项**里的内容，决定了它是指向一块物理内存，还是指向二级页表：
   - [ ] Level 1 translation table entry format

   - 一级页表映射过程
      - 一级页表中每个表项用来设置 1 M 的空间。对于 32 位的系统，虚拟地址空间有 4G，因此需要 4G/1M = 4096 个页表项。
      - 第 0 个页表项用来表示虚拟地址第 0 个 1M（虚拟地址为 0-0xFFFFF）对应哪一块物理内存，以及一些权限设置。同理，第 1 个页表项表示第 1 个 1M（虚拟地址为 0x100000-0x1FFFFF）对应哪一块物理内存以及一些权限设置。依此类推。
      - **使用一级页表时，先在内存里设置号各个页表项，然后把页表基地址告诉 `MMU`，就可以启动 `MMU` 了**；
      - 映射过程：
         1. CPU 发出虚拟地址 `vaddr`，假设为 0x12345678；
         2. MMU 根据 `vaddr[31:20]` 找到第一级页表项（第 0x123 项，根据此项内容得知它是一个段页表项）；
         3. 在页表项中取出**物理地址**：`Section Base Address`，假设为 0x81000000；
         4. 物理地址加上由 `vaddr` 得到的段内偏移地址 0x45678 得到物理地址 0x81045678； 
      - [ ] 添加示意图

   - 二级页表映射过程
     - **首先要设置好一级页表、二级页表，并把一级页表的基地址告诉 `MMU`**；
     - 二级页表格式：
       - [ ] Format of level 2 translation table entry
     - 映射过程：
       1. CPU 发出虚拟地址 `vaddr`，假设为 0x12345678；
       2. MMU 根据 `vaddr[31:20]` 找到第一级页表项（第 0x123 项，根据此项内容得知它是一个二级页表项）；
       3. 在页表项中取出**物理地址**，是二级页表的首地址；
       4. 根据二级页表的首地址和偏移量大小（`vaddr[19:12]`）0x45，找到二级页表中的第 0x45 项；
       5. 从二级页表项中得到物理地址 `page base addr`，假设是 0x81889000；
       6. 结合页内偏移地址 `vaddr[11:0]` （即 0x678）得到物理地址 0x81889678；
     - [ ] 添加示意图

5. 给 APP 新建内存映射
   综上可知，要给 APP 新开辟一块虚拟内存，并且让它指向某块内核 buffer，需要完成：
   - 得到一个 `vm_area_struct`，它表示 APP 的一块虚拟内存空间：APP 调用 `mmap()` 系统函数时，内核就帮助我们构造了一个 `vm_area_struct`，里面包含虚拟地址的范围、权限；
   - 确定物理地址：若要映射某个内核 `buffer`，则需要得到它的物理地址，这得由你提供；
   - 给 `vm_area_struct` 和物理地址之间建立映射关系：内核提供相关函数；
   ```c
   // mmap 调用过程
   // app: mmap(addr, length, prot, flags, fd, offset)
   // ------
   // 内核：
   SYSCALL_DEFINE6(mmap_pgoff ...)  // mm/mmap.c
      vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);   // mm/util.c
         do_mmap_pgoff(file, addr, len, prot, flag, pgoff, &populate);  // include/linux/mm.h
            do_mmap(file, addr, len, prot, flags, 0, pgoff, populate);  // mm/mmap.c
               addr = get_unmmaped_area(file, addr, len, pgoff, flags); // 得到未使用的虚拟地址
               addr = mmap_region(file, addr, len, vm_flags, pgoff); // mm/mmap.c
                  vma = kmem_cache_zalloc(vm_area_cachep, GFP_KERNEL);  // 分配新的 vm_area_struct

                  // 设置 vm_area_struct
                  vma->vm_mm = mm;
                  vma->vm_start = addr;
                  vma->vm_end = addr + len;
                  vma->vm_flags = vm_flags;
                  vma->vm_page_prot = vm_get_page_prot(vm_flags);
                  vma->vm_pgoff = pgoff;

                  // 调用驱动程序的 mmap
                  err = file->f_op->mmap(file, vma);
   ```
   - 在驱动的 `mmap` 函数中需要完成：
     1. 提供物理地址；
     2. 设置属性：是否使用 `cache, buffer`；
     3. 给 `vm_area_struct` 和物理地址之间建立映射；

6. `mmap` 驱动实现
   - `cache` 和 `buffer`
      使用 `mmap` 时需要有 `cache` 和 `buffer` 的知识。
      - [ ] 添加链接内容 
      参考：[ARM 的 cache 和写缓冲器（write buffer）]()；
      - [ ] 添加 cache、处理器内核及主存之间的关系

      `cache` 是一块高速内存；写缓冲器（`write buffer`）相当于一个 `FIFO`，可以把多个写操作集合起来一次写入内存。

      结合程序运行时的局部性原理。CPU 的速度非常快，内存的速度相对来说很慢。根据局部性原理，通过引入 `cache` 使得 CPU 能够以较快的速度对数据进行读写：
        - 读取内存 `addr` 处数据时：
         1. 先看 `cache` 中有没有 `addr` 的数据，如果有就直接从 `cache` 里返回数据，这被称为 `cache` 命中；
         2. 如果 `cache` 中没有 `addr` 的数据，则从内存里把数据读入。这里它会读入一行（`cache line`）数据，而非仅此一个数据；
         3. CPU 很可能再次用到这个 `addr` 的数据，或者它附近的数据，这时就可以快速地从 `cache` 中获得数据；
       - 写数据：
         CPU 要写数据时，可以直接写入内存（很慢）；也可以先写入 `cache` 再通过 `cache` 写入内存。这里有 2 种从 `cache` 写入内存的策略：
         1. 写通（`write through`）：**数据要同时写入 `cache` 和内存，所以 `cache` 和内存中的数据保持一致**，使用写缓冲器提高效率；有些写缓冲器有“写合并”功能，对于内存来说这没什么差别，但是对于硬件寄存器这可能导致问题。因此，**对于寄存器操作，不会启用 `buffer` 功能**；对于内存操作（例如 LCD），可以启用 `buffer` 功能；
         - [ ] 如何理解写通？
         2. 写回（`write back`）：**新数据只是写入 `cache`，不会立刻写入内存，`cache` 和内存中的数据并不一致**；新数据写入 `cache` 时，这一行 `cache` 被标记为 “脏”（`Dirty`），**当 `cache` 不够用时，才需要把脏数据写入内存**；

      **使用写回功能可以大幅提高效率，但是要注意 `cache` 和内存中的数据很可能不一致**。这在很多时候需要小心处理：比如 CPU 产生了新数据，DMA 把数据从内存搬到网卡，这时候就需要 CPU 先执行命令把新数据从 `cache` 刷到内存；反过来也是一样的，DMA 从网卡获得的新数据在内存里，CPU 读数据之前先把 `cache` 中的旧数据丢弃。
   
   - 是否使用 `cache`、`buffer`
      有 4 种组合方式（`arch/arm/include/asm/pgtable-2level.h`）：
      ```c
      #define L_PTE_MT_UNCACHED           ...   // 00
      #define L_PTE_MT_BUFFERABLE         ...   // 01
      #define L_PTE_MT_WRITETHROUGH       ...   // 10
      #define L_PTE_MT_WRITEBACK          ...   // 11
      ```
      对应下表中的各项（下表来自 s3c2410 芯片手册）：
      | 是否启用 `cache` | 是否启用 `buffer` | 说明 |
      | ----- | ----- | ----- |
      | 0 | 0 | Non-cached, Non-buffered(NCNB)；<br />读、写都直达外设硬件。 |
      | 0 | 1 | Non-cached, buffered(NCB)；<br />读、写都直达外设硬件；<br />写操作通过 `buffer` 实现，CPU 不等待写操作完成，马上执行下一条指令。 |
      | 1 | 0 | Cached, Write-Throuth mode(WT)；<br />读：cache hit 时从 `cache` 读数据；cache miss 时入一行数据到 `cache`；<br />写：通过 `buffer` 实现，CPU 不等待写操作完成，马上执行下一条指令。 |
      | 1 | 1 | Cached, Write-Back mode(WB)；<br />读：cache hit 时从 `cache` 读数据；cache miss 时入一行数据到 `cache`；<br />写：通过 `buffer` 实现，cache init 时新数据不会到达硬件，而是在 `cache` 中被标记为“脏”；cache miss 时，通过 `buffer` 写入硬件，CPU 不等待写操作完成，马上执行下一条指令。 |
      - 第一种适合寄存器的读写；
      - 第二种适合显存操作，因为对显存很少有读操作，基本都是写操作，而写操作即使被“合并”也没有关系；
      - 第三种适用于只读设备，在读数据时用 cache 加速，基本不需要写；
      - 第四种适合一般的内存读写；

   - 驱动实现示例
     - [ ] framebuffer 子系统？
      ```c
      // drivers/video/fbdev/mxsfb.c
      static int mxsfb_mmap(struct fb_info *info, struct vm_area_struct *vma) {
         u32 len;
         unsigned long offset = vma->vm_pgoff << PAGE_SHIFT;

         if (offset < info->fix.smem_len) {
            // mmapping framebuffer memory
            len = info->fix.smem_len - offset;
            // 1. 确定物理地址，单位：page, 4k
            vma->vm_pgoff = (info->fix.smem_start + offset) >> PAGE_SHIFT;
         } else {
            return -EINVAL;
         }

         len = PAGE_ALIGN(len);
         if (vma->vm_end - vma->vm_start > len)
            return -EINVAL;
         
         // make buffers bufferable
         // 2. 设置属性: cache? buffer?
         vma->vm_page_prot = pgprot_writecombine(vma->vm_page_prot);

         // 3. 映射
         if (remap_pfn_range(vma, vma->vm_start, vma->vm_pgoff, vma->vm_end - vma->vm_start, vma->vm_page_prot)) {
            dev_dbg(info->device, "mmap remap_pfn_range failed\n");
            return -ENOBUFS;
         }

         return 0;
      }

      // drivers/video/fbdev/controlfb.c
      static int controlfb_mmap(struct fb_info *info, struct vm_area_struct *vma) {
         unsigned long mmio_pgoff;
         unsigned long start;
         u32 len;

         // 1. 确定物理地址
         start = info->fix.smem_start;
         len = info->fix.smem_len;
         mmio_pgoff = PAGE_ALIGN((start & ~PAGE_MASK) + len) >> PAGE_SHIFT;
         if (vma->vm_pgoff >= mmio_pgoff) {
            if (info->var.accel_flags)
               return -EINVAL;
            vma->vm_pgoff -= mmio_pgoff;
            start = info->fix.mmio_start;
            len = info->fix.mmio_len;
            // 2. 确定属性: cache ? buffer ?
            vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
         } else {
            // framebuffer
            vma->vm_page_prot = pgprot_cached_wthru(vma->vm_page_prot);
         }

         // 3. 映射
         return vm_iomap_memory(vma, start, len);
      }
      ```
7. 应用 + 驱动示例
   在驱动程序中申请一个 8K 的 buffer，让 APP 通过 mmap 能直接访问。
   - app
      `mmap(...)` 函数原型：
      ```c
      // man 2 mmap:
      // mmap, munmap - map or unmap files or devices into memory
      /* mmap() creates a new mmaping in the virtual address space
       * of the calling process. The starting address for the new
       * mapping is spesified in addr. The length argument specifies
       * the length of the mapping(which must be greater than 0).
       * @addr：NULL 时由内核选择一个（页对齐的）地址进行 mapping（最常用方式）；
       *       如果不为 NULL，内核将会选择一个最为接近的页边界（大于等于由 /proc/sys/vm/mmap_min_addr 指定的值）以进行 mapping。
       *       如果当前地址（addr）已经被映射，内核可能会/不会按照 addr 选择一个新地址
       * @prot: describes the desired memory protection of the mapping and must not conflict with the open mode of the file.
       *       PROT_EXEC <| PROT_READ | PROT_WRITE | PROT_NONE>
       * @flags: determines whether updates to the mapping are visible to other process mapping the same region, and are carried through to the underlying file.
       *       MAP_SHARED, MAP_SHARED_VALIDATE, MAP_PRIVATE
       * @fd：after the mmap() call has returned, the fd can be closed immediately without invalidating the mmaping.
       * @offset: must be a multiple of the page size as returned by sysconf(_SC_PAGE_SIZE)
       * return: 完成映射的（用户空间虚拟）地址（mapping），出错时返回 MAP_FAILED(-1)
       */
      void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
      int munmap(void *addr, size_t length);
      ```
      app 程序（部分）：
      ```c
      // 1. 打开文件
      fd = open("/dev/hello", O_RDWR);
      if (fd == -1) {
         printf("can't open file /dev/hello\n");
         return -1;
      }

      // 2. mmap
      buf = mmap(NULL, 1024*8, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
      if (buf == MAP_FAILED) {
         printf("can't mmap file /dev/hello\n");
         return -1;
      }
      printf("mmap address = 0x%x\n", buf);
      printf("buf origin data = %s\n", buf);

      // 3. write
      strcpy(buf, "new");

      // 4. read & compare
      // 对于 MAP_SHARED 映射，str = "new"；MAP_PRIVATE，str = "old"
      read(fd, str, 1024);
      if (strcmp(buf, str) == 0) {
         // MAP_SHARED
         printf("compare ok!\n");
      } else {
         // MAP_RPIVATE
         printf("compare err!\n");
         printf("str = %s!\n", str);   // old
         printf("buf = %s!\n", buf);   // new
      }
      ```
      - `MAP_SHARED`：多个 app 都调用 mmap 映射同一块内存时，对内存的修改大家都可以看到。就是说多个 app、驱动程序实际上访问的都是同一块内存；
      - `MAP_PRIVATE`：创建一个 `copy to write` 的私有映射。在没有发生写操作时，APP、驱动访问的都是同一块内存。当 app 对该内存进行修改时（写操作），其他程序是看不到的，因为当 app 写内存时内核会先创建一个（这块内存的）拷贝给这个 app（触发 `copy to write`）。这个拷贝是这个 app 私有的，其他的 app、驱动无法访问。
        - Linux 中多个 APP 共享同一个动态库，在写操作之前大家都使用内存中唯一一份代码。当 APP1 发起写操作时，内核会为它复制一份代码，再执行写操作。于是 APP1 就有了专享的、私有的动态库，在里面做的修改只会影响到 APP1。其他程序仍然共享原先的、未修改的代码。
      - 查看该程序的内存使用情况：
         ```bash
         cat /proc/PIC/maps
         ```
   - 驱动
      驱动需要：
     1. 分配 8K 的内存；
         | 函数名 | 说明 |
         | ----- | ----- |
         | `kmalloc` | 分配到的内存在物理地址上是连续的 |
         | `kzalloc` | 分配到的内存在物理地址上是连续的，内容清 0 |
         | `vmalloc` | 分配到的内存在物理地址上不保证是连续的 |
         | `vzalloc` | 分配到的内存在物理地址上不保证是连续的，内容清 0 |
         - 应使用 `kmalloc` 或 `kzalloc`，这样得到的内存物理地址是连续的，在 mmap 后 APP 才可以使用同一个基地址去访问这块内存；
         - 如果物理地址不连续，就要执行多次 mmap；
     2. 提供 mmap 函数：
         ```c
         static int hello_drv_mmap(struct file *file, struct vm_area_struct *vma) {
            // 1. 获得物理地址
            // kernel_buf 是内核使用的虚拟地址
            unsigned long phy = virt_to_phys(kernel_buf);
            // 2. 设置属性
            // 不使用 cache，使用 buffer
            vma->vm_page_prot = pgprot_writecombine(vma->vm_page_prot);
            // 3. 映射
            // 按页映射 phy >> PAGE_SHIFT
            if (remap_pfn_range(vma, vma->vm_start, phy >> PAGE_SHIFT, vma->vm_end - vma->vm_start, vma->vm_page_prot)) {
               printk("mmap remap_pfn_range failed\n");
               return -ENOBUFS;
            }

            return 0;
         }
         ```
         - **`remap_pfn_range` 中 `pfn` 是指 `Page Frame Number`（页帧号）**，当给定物理地址 `phy`，它的 `pfn = phy / PAGE_SIZE = phy >> PAGE_SHIFT`；
         - APP 调用 mmap 后，会导致驱动程序的 mmap 函数被调用，最终 APP 的虚拟地址和驱动程序中的物理地址就建立了映射关系（驱动和 APP 中的虚拟地址指向同一块物理地址）。APP 可以直接访问驱动程序的 buffer。

## 同步与互斥
