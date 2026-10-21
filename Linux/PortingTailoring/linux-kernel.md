# linux kernel
## 目录结构

## 编译
1. 编译全部
2. 单独编译内核
3. 单独编译模块
4. 单独编译设备树

## Makefile 解析
### 配置-makef xxx_defconfig
### 编译-make
### Makefile.build 脚本

## 映像文件
*vmlinux, Image, zImage* 和 *uImage*

## 链接脚本
- 位于 `arch/arm/kernel/vmlinux.lds`

## 内核启动相关源码
在连接脚本 `arch/arm/kernel/vmlinux.lds` 中指定程序的入口：`ENTRY(stext)`；

### head.s
- 符号 `stext` 定义于 `arch/arm/kernel/head.s`：
  ```asm
  ENTRY(stext)
  ...
  ENDPROC(stext)
  ```
  - `ENTRY()` 和 `ENDPROC` 是宏定义，位于 `include/linux/linkage.h`：
    ```c
    #ifndef ENTRY
    #define ENTRY(name) \
      .globl name ASM_NL \
      ALIGN ASM_NL \
      name:
    #endif

    ...

    #ifndef END
    #define END(name) \
      .size name, .-name
    #endif

    /* If symbol 'name' is treated as a subroutine (gets called, and returns)
     * then please use ENDPROC to mark 'name' as STT_FUNC for the benefit of
     * static analysis tools such as stack depth analyzer.
     */
    #ifndef ENDPROC
    #define ENDPROC(name) \
    	.type name, @function ASM_NL \
    	END(name)
    #endif
    ```
- 在此汇编文件中指出："*kernel startup entry point. This is normally called from the decompressor code. The requirements are:*"
  - *MMU off*;
  - *D-Cache = off*;
  - *I-Cache = don't care*;
  - *r0 = 0*;
  - *r1 = machine nr*;
  - *r2 = atags or dtb pointer*.
- 在此文件中：
  1. 调用 `safe_svcmode_maskall(), arch/arm/include/asm/assembler.h`，确保 *cpu* 处于 *svc* 模式，并且关闭所有中断；
  2. 读处理器 *ID*；
  3. 调用 `__lookup_processor_type(), arch/arm/kernel/head-common.S`，检查当前系统是否支持此 *cpu*，如果支持就获取 *procinfo* 信息；
     - *procinfo* 数据类型为 `struct proc_info_list`，位于 `arch/arm/include/asm/procinfo.h`；
     - 在 *linux* 内核中，将每种处理器都抽象为一个 *proc_info_list* 结构体数据，每种处理器都对应一个 *procinfo*。因此可以通过处理器 *ID* 来找到对应的 *procinfo* 结构。
  4. 调用 `__vet_atags(), arch/arm/kernel/head-common.S` 验证 *atags* 或设备树（*.dtb*）的合法性；
  5. 调用 `__create_page_tables(), arch/arm/kernel/head.S`，创建页表；
  6. 保存 `__mmap_switched(), arch/arm/kernel/head-common.S` 函数地址，该函数会调用 `start_kernel()` 函数；
  7. 调用 `__enable_mmu(), arch/arm/kernel/head.S` 以使能 *MMU*。它会通过 `__turn_mmu_on()` 来打开 *MMU*，并继续执行 `__mmap_switched()` 函数。

### mmap_switched
- 位于 `arch/arm/kernel/head-common.S`；
- 最终调用 `start_kernel()` 来启动内核。

### start_kernel
通过调用众多的子函数来完成 *Linux* 启动之前的一些初始化工作
```c
// version: 4.1.15
// init/main.c
asmlinkage __visible void __init start_kernel(void)
{
  char *command_line;
  char *after_dashes;
  
  ...
  setup_arch(&command_line);
  ...
  pr_notice("%s", linux_banner);
  ...
  after_dashes = parse_args("Booting kernel",
				  static_command_line, __start___param,
				  __stop___param - __start___param,
				  -1, -1, &unknown_bootoption);
  ...
}
```
- 这些工作包括：SMP、页、VFS、RCU、内核异常列表等的初始化或设置；
- 在 `setup_arch(&command_line)` 中是架构相关的初始化；
    - 此函数会解析传递进来的 atags 或者设备树（.dtb）文件，会根据设备树里面的 model 和 compatible 属性值来判断 linux 是否支持这个板子；
    - 此函数也会获取设备树中 chosen 节点下的 bootargs 属性值（也就是 uboot 中环境变量 bootargs 的值），从而得到命令行参数；
    - 获取到的参数会保存到 `command_line` 里面。
- 最后会调用 `rest_init`。

1. `command_line` 及其他 *command line* 变量
   - [ ] command_line 是如何通过 uboot 传入到 Linux Kernel 中的？
  ```c
  // version: 4.1.15
  // init/main.c
  /* Untouched command line saved by arch-specific code. */
  char __initdata boot_command_line[COMMAND_LINE_SIZE];
  /* Untouched saved command line (eg. for /proc) */
  char *saved_command_line;
  /* Command line for parameter parsing */
  static char *static_command_line;
  /* Command line for per-initcall parameter parsing */
  static char *initcall_command_line;

  static char *execute_command;
  static char *ramdisk_execute_command;
  ```
  - `__start___param` 和 `__stop___param` 源自链接脚本 `arch/arm/kernel/vmlinux.lds`

2. `linux_banner`
  ```c
  // version: 4.1.15
  // init/version.c
  /* FIXED STRINGS! Don't touch */
  const char linux_banner[] = 
    "Linux version " UTS_RELEASE " (" LINUX_COMPILE_BY "@"
    LINUX_COMPILE_HOST ") (" LINUX_COMPILE ") " UTS_VERSION "\n";
  ```
  - 注：`UTS_RELEASE`、`LINUX_COMPILE_BY`... 自动生成于 `include/generated` 目录下的 `utsrelease.h` 和 `compile.h` 等文件。

#### setup_arch
```c
// version: 4.1.15
// arch/arm/kernel/setup.c
void __init setup_arch(char **cmdline_p) {
  const struct machine_desc *mdesc;

  setup_processor();
  ...
}
```

1. `setup_processor()`:
  ```c
  // version: 4.1.15
  // arch/arm/kernel/setup.c
  static void __init setup_processor(void) {
    struct proc_info_list *list;

    /* 
     * locate processor in the list of supported processor 
     * types. The linker builds this table for us from the entries
     * in arch/arm/mm/proc-*.S
     */
    list = lookup_processor_type(read_cpuid_id());
    if (!list) {
      pr_err("CPU configuration botched (ID %08x), unable to continue.\n",
        read_cpuid_id());
      while (1);
    }

    cpu_name = list->cpu_name;
    __cpu_architecture = __get_cpu_architecture();
  
  #ifdef MULTI_CPU
    processor = *list->proc;
  #endif
    ...
  }
  ```

   - 关于 `processor` 的解析：
     1. `processor` 是一个 `struct processor` 类型的全局变量：
      ```c
      // version: 4.1.15
      // arch/arm/kernel/setup.c
      /* 
       * 定义 
       */
      #ifdef MULTI_CPU
      struct processor processor __read_mostly;
      #endif

      // arch/arm/include/asm/proc-fns.h
      /* 
       * 声明 
       */
      /*
       * Don't change this structure - ASM code relies on it.
       */
      extern struct processor {
      /* MISC
       * get data abort address/flags
       */
        void (*_data_abort)(unsigned long pc);
      /*
      * Retrieve prefetch fault address
      */
      unsigned long (*_prefetch_abort)(unsigned long lr);
      /*
      * Set up any processor specifics
      */
      void (*_proc_init)(void);
      /*
      * Disable any processor specifics
      */
      void (*_proc_fin)(void);
      /*
      * Special stuff for a reset
      */
      void (*reset)(unsigned long addr) __attribute__((noreturn));
      /*
      * Idle the processor
      */
      int (*_do_idle)(void);
      /*
      * Processor architecture specific
      */
      /*
      * clean a virtual address range from the
      * D-cache without flushing the cache.
      */
      void (*dcache_clean_area)(void *addr, int size);
      /*
      * Set the page table
      */
      void (*switch_mm)(phys_addr_t pgd_phys, struct mm_struct *mm);
      /*
      * Set a possibly extended PTE.  Non-extended PTEs should
      * ignore 'ext'.
      */
      #ifdef CONFIG_ARM_LPAE
        void (*set_pte_ext)(pte_t *ptep, pte_t pte);
      #else
        void (*set_pte_ext)(pte_t *ptep, pte_t pte, unsigned int ext);
      #endif

        /* Suspend/resume */
        unsigned int suspend_size;
        void (*do_suspend)(void *);
        void (*do_resume)(void *);
      } processor;
      ```
     2. `processor` 在 `setup_processor()` 中被赋值，需要先根据 *CPU* 的 *ID* 从给定列表信息中检索得到 `struct proc_info_list` 类型的变量。这个列表是在链接内核映像文件时生成的（*The linker builds this table for us from the entries in `arch/arm/mm/proc-*.S`*），其中数组元素的来源是 `arch/arm/mm/proc-*.S` 中的内容。在链接脚本中定义了如下内容：
      ```script
      // arch/arm/kernel/vmlinux.lds
      ...
      __proc_info_begin = .; *(.proc.info.init) __proc_info_end = .;
      ...
      ```
      进一步在 `System.map` 中找到：
      ```map
      80826b50 T __proc_info_begin
      80826b50 t __v7_ca5mp_proc_info
      80826b84 t __v7_ca9mp_proc_info
      80826bb8 t __v7_ca8_proc_info
      80826bec t __v7_cr7mp_proc_info
      80826c20 t __v7_ca7mp_proc_info
      80826c54 t __v7_ca12mp_proc_info
      80826c88 t __v7_ca15mp_proc_info
      80826cbc t __v7_b15mp_proc_info
      80826cf0 t __v7_ca17mp_proc_info
      80826d24 t __krait_proc_info
      80826d58 t __v7_proc_info
      80826d8c T __proc_info_end
      ```
      这里得到了支持的处理器型号列表，考虑到 *i.mx6ull* 为 *Cortex-V7* 类型的处理器，因此进一步定位到 `__v7_proc_info`。
      结合源码中的注释内容和上述找到的信息，在 `arch/arm/mm/proc-v7.S` 中定义了：
      ```asm
       /*
        * Match any ARMv7 processor core.
        */
        .type	__v7_proc_info, #object
      __v7_proc_info:
        .long	0x000f0000		@ Required ID value
        .long	0x000f0000		@ Mask for ID
        __v7_proc __v7_proc_info, __v7_setup
        .size	__v7_proc_info, . - __v7_proc_info
      ```
      其中 `__v7_proc` 是一个宏：
      ```asm
      .macro __v7_proc name, initfunc, mm_mmuflags = 0, io_mmuflags = 0, hwcaps = 0, proc_fns = v7_processor_functions
        ALT_SMP(.long	PMD_TYPE_SECT | PMD_SECT_AP_WRITE | PMD_SECT_AP_READ | \
            PMD_SECT_AF | PMD_FLAGS_SMP | \mm_mmuflags)
        ALT_UP(.long	PMD_TYPE_SECT | PMD_SECT_AP_WRITE | PMD_SECT_AP_READ | \
            PMD_SECT_AF | PMD_FLAGS_UP | \mm_mmuflags)
        .long	PMD_TYPE_SECT | PMD_SECT_AP_WRITE | \
          PMD_SECT_AP_READ | PMD_SECT_AF | \io_mmuflags
        initfn	\initfunc, \name
        .long	cpu_arch_name
        .long	cpu_elf_name
        .long	HWCAP_SWP | HWCAP_HALF | HWCAP_THUMB | HWCAP_FAST_MULT | \
          HWCAP_EDSP | HWCAP_TLS | \hwcaps
        .long	cpu_v7_name
        .long	\proc_fns
        .long	v7wbi_tlb_fns
        .long	v6_user_fns
        .long	v7_cache_fns
      .endm
      ```
      上述宏定义的内容与 `struct proc_info_list` 的结构体定义一致：
      ```c
      // version: 4.1.15
      // 
      /*
       * Note!  struct processor is always defined if we're
       * using MULTI_CPU, otherwise this entry is unused,
       * but still exists.
       *
       * NOTE! The following structure is defined by assembly
       * language, NOT C code.  For more information, check:
       *  arch/arm/mm/proc-*.S and arch/arm/kernel/head.S
       */
      struct proc_info_list {
        unsigned int		cpu_val;
        unsigned int		cpu_mask;
        unsigned long		__cpu_mm_mmu_flags;	/* used by head.S */
        unsigned long		__cpu_io_mmu_flags;	/* used by head.S */
        unsigned long		__cpu_flush;		/* used by head.S */
        const char		*arch_name;
        const char		*elf_name;
        unsigned int		elf_hwcap;
        const char		*cpu_name;
        struct processor	*proc;
        struct cpu_tlb_fns	*tlb;
        struct cpu_user_fns	*user;
        struct cpu_cache_fns	*cache;
      };
      ```
      因此 `processor` 变量的值为 `__v7_proc` 宏中的 `v7_processor_functions`。它的定义如下（`arch/arm/mm/proc-v7.S`）
      ```asm
      @ define struct processor (see <asm/proc-fns.h> and proc-macros.S)
	    define_processor_functions v7, dabort=v7_early_abort, pabort=v7_pabort, suspend=1
      ```
      宏 `define_processor_functions` 定义在 `arch/arm/mm/proc-macros.S` 中：
      ```asm
      .macro define_processor_functions name:req, dabort:req, pabort:req, nommu=0, suspend=0
        .type	\name\()_processor_functions, #object
        .align 2
      ENTRY(\name\()_processor_functions)
        .word	\dabort
        .word	\pabort
        .word	cpu_\name\()_proc_init
        .word	cpu_\name\()_proc_fin
        .word	cpu_\name\()_reset
        .word	cpu_\name\()_do_idle
        .word	cpu_\name\()_dcache_clean_area
        .word	cpu_\name\()_switch_mm

        .if \nommu
        .word	0
        .else
        .word	cpu_\name\()_set_pte_ext
        .endif

        .if \suspend
        .word	cpu_\name\()_suspend_size
      #ifdef CONFIG_ARM_CPU_SUSPEND
        .word	cpu_\name\()_do_suspend
        .word	cpu_\name\()_do_resume
      #else
        .word	0
        .word	0
      #endif
        .else
        .word	0
        .word	0
        .word	0
        .endif

        .size	\name\()_processor_functions, . - \name\()_processor_functions
      .endm
      ```
      这里定义了一个 `struct processor` 的多个回调函数，其实现位于同样位于 `arch/arm/mm/proc-v7.S`，相关函数的解析见笔记 `Hardware\Arch\arm-v7.md`
      - [ ] 调试获取 i.mx6ull 的 CPU ID 及确认所使用的型号是否为 v7_proc_info (因为上述还存在其他关于 v7 的选项)

#### rest_init
```c
// version: 4.1.15
// init/main.c
static noinline void __init_refok rest_init(void)
{
  int pid;

  rcu_scheduler_starting();
  smpboot_thread_init();
  /* 
   * We need to spawn init first so that it obtains pid 1, however 
   * the init task will end up wanting to create kthreads, which, if
   * we schedule it before we create kthreadd, will OOPS.
   */
  kernel_thread(kernel_init, NULL, CLONE_FS);
  numa_default_policy();
  pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
  rcu_read_lock();
  kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
  complete(&kthreadd_done);

  /* 
   * The boot idle thread must execute schedule() 
   * at least once to get things moving:
   */
  init_idle_bootup_task(current);
  schedule_preempt_disabled();
  /* Call into cpu_idle with preempt disabled */
  cpu_startup_entry(CPUHP_ONLINE);
}
```

1. 调用 `rcu_scheduler_starting`，启动 RCU 锁调度器；
2. 调用 `kernel_thread` 创建 `kernel_init` 进程，即 `init` 内核进程，PID 为 1。
   - **init 进程一开始是内核进程，后面 init 进程会在根文件系统中查找名为 "init" 的程序，通过运行这个 "init" 程序，init 进程会实现从内核态到用户态的转变**。
3. 调用 `kernel_thread` 创建 `kthreadd` 内核进行，PID 为 2。
   - `kthreadd` 进程负责所有内核进程的调度和管理。
4. 调用 `cpu_startup_entry` 进入 `idle` 进程，即空闲进程，PID 为 0。
   - 调用 `cpu_idle_loop`，是一个 while 循环，也就是 `idle` 进程代码；
   - **`idle` 进程并没有使用 `kernel_thread` 或者 `fork` 函数来创建，因为它是由主进程演变而来的**。

通过在终端中输入 `ps -A` 就能查看到上述进程的信息（除 `idle` 以外，因为它是内核进程）。

### init 进程
```c
// version: 4.1.15
// init/main.c
static int __ref kernel_init(void *unused)
{
  int ret;

  kernel_init_freeable();
  ...
}
```

`kernel_init(), (init/main.c)` 就是 `init` 进程具体做的工作，在 `kernel_init()` 中：
1. 调用 `kernel_init_freeable()` 用于完成 `init` 进程的一些其他初始化工作：
   - 等待 `kthreadd` 进程准备就绪 -> SMP 初始化 -> 多核（SMP）调度初始化；
   - **调用 `do_basic_setup()` 完成 Linux 下设备驱动初始化工作，它会调用 `driver_init` 函数完成 Linux 下驱动模型子系统的初始化**；
   - 打开设备 `/dev/console`，文件描述符为 0，作为标准输入（0）；
   - 通过 `sys_dup` 将标准输入（0）复制两次，一个作为标准输出（1），一个作为标准错误（2）；
   - 调用 `prepare_namespace()` 来挂载根文件系统，根文件系统也由命令行参数指定，即 uboot 的 bootargs。比如 `root=/dev/mmcblk1p2 rootwait rw`。
2. 查找 `init` 程序（依次查找，找到即执行）：
   - `ramdisk_execute_command`，值为 "/init"，即根目录下的 init 程序。另外，值也可以通过 uboot 传递，在 bootargs 中使用 `rdinit=xxx` 即可。其中 "xxx" 表示具体的 init 程序名字；
   - `execute_command`，通过 uboot 传递，在 bootargs 中使用 `init=xxx` 即可。例如 `init=/linuxrc` 表示根文件系统中的 linuxrc 就是要执行的用户空间 init 程序；
   - 如果 `ramdisk_execute_command` 和 `execute_command` 都为空，则依次查找 `/sbin/init`、`/etc/init`、`/bin/init` 和 `/bin/sh`。这四个相当于是备用的 init 程序。

   反正就是不管如何一定要在根文件系统中找到一个可以运行的 init 程序。找到后通过 `run_init_process` 来运行 init 程序；如果没有找到，则 Linux 启动失败，提示错误发生。

#### 驱动初始化
1. `driver_init()`：
  ```c
  // version: 4.1.15
  // drivers/base/init.c
  /* driver_init - initialize driver model. 
    * Call the driver model init functions to initialize their
    * subsystems. Called early from init/main.c
    */
  void __init driver_init(void) {
    /* These are the core pieces */
    devtmpfs_init();
    devices_init();
    buses_init();
    classes_init();
    firmware_init();
    hypervisor_init();

    /* These are also core pieces */
    platform_bus_init();
    cpu_dev_init();
    memory_dev_init();
    container_dev_init();
    of_core_init();
  }
  ```
2. 调用链
  ```c
  start_kernel()
    rest_init()
      kernel_thread(kernel_init, NULL, CLONE_FS)
    
  kernel_init()
    kernel_init_freeable()
      do_basic_setup()
        driver_init();
  ```
3. 功能：

## 内核移植
将 NXP 官方提供的 Linux 内核移植到目标开发板上。

### NXP 官方 Linux 内核编译
- 修改顶层 Makefile：设置 `ARCH` 和 `CROSS_COMPILE` 的值（固定设置）；
- 配置并编译 linux 内核：
  - 配置文件位置：`arch/arm/configs`：imx_v7_defconfig 或 imx_v7_mfg_defconfig，其中 "mfg" 表示 zImage 可以通过 NXP 官方工具 MfgTool 来烧写。
  - make 命令：
     ```bash
     make clean # make distclean
     make imx_v7_mfg_defconfig
     make -j16
     ```
  - 得到 zImage 镜像文件和 dtb 文件：`arch/arm/boot/` 下、`arch/arm/dts/` 下。
- 启动测试
  - 将 zImage 和 dtb 文件置于网络文件夹之下；
  - uboot 部分：
    - bootargs 为 `console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw`；
    - 通过命令下载 zImage 和 dtb：`tftp 8080 zImage`、`tftp 83000000 imx6ull-14x14-evk.dtb`；
    - 启动：`bootz 80800000 - 83000000`。
- 根文件系统：
  - 正常加载："VFS: Mounted root (ext3 filesystem) on device 1/9:10 ......"；
  - 不正常："VFS: cannot open root device "(null)" or unknown-block(0, 0): error-6 ... kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0, 0)......"。内核启动了，但是根文件系统不存在。

### 在 linux 中添加目标开发板
- 添加开发板默认配置文件：
  - `arch/arm/configs/imx_v7_mfg_defconfig` ->(copy) `arch/arm/configs/imx_alientek_emmc_defconfig`，并屏蔽掉 `CONFIG_ARCH_MULTI_V6=y`，因为 imx6ull 为 ARMv7 架构，所以要屏蔽掉 v6 相关选项，否则可能会遇到驱动无法加载的情况。
- 添加开发板对应的设备树文件；
- 修改设备树 Makefile（`arch/arm/boot/dts/Makefile`），在其中添加上述设备树文件；
- 编译测试。

### CPU 主频和相关驱动修改
首先需要确保有可用的根文件系统，因为需要在 linux 系统中对 CPU 频率进行查询和其他相关操作。
1. cpu 部分
   - 查看 cpu 信息：
        ```bash
        cat /proc/cpuinfo
        ```
        - BogoMIPS：是 Linux 系统中衡量处理器运行速度的一个“尺子”。处理器性能越强，主频越高，BogoMIPS 值就越大。但 BogoMIPS 只是粗略地计算 CPU 性能。
   - 查看 cpu 频率：
        ```bash
        # 在 /sys/devices/system/cpu/cpux/cpufreq 下执行 ls 命令，可获取与频率相关的文件
        # 通过 cat 命令即可查看相关信息
        ```
        - cpuinfo_cur_freq: 当前 cpu 工作频率，从 cpu 寄存器读取到的工作频率；
        - cpuinfo_max_freq: 处理器所能运行的最高工作频率，单位 KHz；
        - cpuinfo_min_freq: 处理器所能运行的最低工作频率，单位 KHz；
        - cpuinfo_transition_latency: 处理器切换频率所需要的时间，单位 ns；
        - scaling_available_frequencies: 处理器支持的主频列表，单位 KHz；
        - scaling_available_governors: 当前内核中支持的所有 governor（调频）类型；
        - scaling_cur_freq: 保存着 cpufreq 模块缓存的当前 cpu 频率，不会对 cpu 硬件寄存器进行检查；
        - scaling_driver: 保存着当前 cpu 所使用的调频驱动；
        - scaling_governor: 当前 governor（调频）策略，Linux 内核中一共有 5 个调频策略：
          - Performance: 最高性能，直接用最高频率，不考虑耗电；
          - Interactive: 一开始直接用最高频率，然后根据 cpu 负载慢慢降低；
          - Powersave: 省电模式，通常以最低频率运行，系统性能会受影响，一般不会用这个；
          - Userspace: 可以在用户空间手动调节频率；
          - Ondemand: 定时检查负载，然后根据负载来调节频率。负载低时降低 cpu 频率，省电、负载高时提高 cpu 频率，增加性能。
        - scaling_max_freq: governor（调频）可以调节的最高频率；
        - scaling_min_freq: governor（调频）可以调节的最低频率；
    - 查看 cpu 在各个频率下的工作时间：
        ```bash
        cat /sys/devices/system/cpu/cpux/cpufreq/stats/time_in_state
        ```
    - 调频策略修改：
      - 配置内核调整调频策略：
        - make menuconfig
        - CPU Power Management -> CPU Frequency scaling -> Default CPUFreq governor -> 选择目标策略 -> 保存
        - 重新编译，编译之前无需重新配置；
      - 修改 xxx_defconfig 文件：其中有如下内容：
        ```
        CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND=y
        CONFIG_CPU_FREQ_GOV_POWERSAVE=y
        CONFIG_CPU_FREQ_GOV_USERSPACE=y
        CONFIG_CPU_FREQ_GOB_INTERACTIVE=y
        ```
        - 将 `CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND=y` 屏蔽掉，并在最后添加 `CONFIG_CPU_FREQ_GOV_ONDEMAND=y` 即可将策略调整到 Performance；
        - 修改完成后重新编译 Linux 内核，编译之前需要重新配置；
    - 在实际产品开发中，从省电的角度考虑，建议使用 ondemand 策略，兼顾省电、发热和性能。
    - 超频相关
      - 在实际产品中，禁止任何超频。务必严格按照手册上给出的标准工作频率来运行。如果想要更高的性能，请购买相应型号的处理器；
      - 超频设置：
        - 修改设备树（`arch/arm/dts/imx6ull.dtsi`）
            ```dts
            cpu0: cpu@0 {
                compatible = "arm, cortex-a7";
                device_type = "cpu";
                reg = <0>;
                clock-latency = <61036>; /* two CLK32 periods*/
                operating-points = <
                    /*KHz    uV*/
                    996000   127500
                    792000   122500
                    696000   117500     // 增加 696MHz
                    528000   117500
                    396000   102500
                    198000   950000
                >;
                fsl,soc-operating-points = <
                    /*KHz    uV*/
                    996000   127500
                    792000   122500
                    696000   117500     // 增加 696MHz
                    528000   117500
                    396000   102500
                    198000   950000
                >;
            }
            ```
            - 重新编译设备树 `make dtbs`；
            - 使用新的设备树文件重新启动 linux；
            - 检查频率支持：`cat /sys/devices/system/cpu/cpux/cpufreq/scaling_available_frequencies`；
            - 设置调频策略到 performance，即可超频到 xxx 频率（可以对比下前后 BogoMIPS 的值）；

2. 驱动部分


## 启动流程
*i.mx6ull* 启动时的输出信息-部分：
![](../../img/linux-kernel/260513-234429.png)
   
*linux* 内核最终是需要和根文件系统打交道的，需要挂在根文件系统，并执行根文件系统中的 *init* 程序，以此来进入用户态。

从上电到系统运行的启动流程大致是：
*Bootrom -> TPL -> SPL -> Trust -> uboot -> kernel -> rootfs* (参考 *RK3568* 的启动流程，见笔记 `Linux/PortingTailoring/SDK/Rockchip-3568-linux-sdk.md`)

### smp 的启动流程
