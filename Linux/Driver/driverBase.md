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

## 同步与互斥
