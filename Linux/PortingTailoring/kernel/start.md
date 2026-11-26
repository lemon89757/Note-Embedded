# 内核启动
## init 进程 - kernel_init()
init 进程是第一个内核进程

### 挂载根文件系统
在 `kernel_init_freeable()` 中挂载根文件系统

1. 调用链
    ```c
    // version: 4.1.15
    kernel_init()
        kernel_init_freeable()
            ...
            if (!ramdisk_execute_command)
		        ramdisk_execute_command = "/init";

            if (sys_access((const char __user *) ramdisk_execute_command, 0) != 0) {
		        ramdisk_execute_command = NULL;
		        prepare_namespace();
            }
            ...
    ```
    - 判断 `char *ramdisk_execute_command` 的赋值情况
    - 使用 *initramfs* 时：如果内核检测到存在 `/init`（通常来自 *initramfs*），挂载最终根文件系统的任务通常会由这个用户空间的 `/init` 程序接管（它可能会加载驱动后再 *pivot_root*）。在这种情况下，内核会跳过 `prepare_namespace()` 的调用
    - 无 *initramfs* 时：如果所有的存储和文件系统驱动都编译进了内核，或者没有提供用户空间的 `/init`，内核就会在 `kernel_init_freeable()` 中调用 `prepare_namespace()` 来直接挂载由 `root=` 参数指定的设备

2. `prepare_namespace()`
    ```c
    // version: 4.1.15
    // init/do_mounts.c
    /*
     * Prepare the namespace - decide what/where to mount, load ramdisks, etc.
     */
    void __init prepare_namespace(void)
    {
        if (root_delay) {
		    printk(KERN_INFO "Waiting %d sec before mounting root device...\n"， 
                root_delay);
		    ssleep(root_delay);
	    }
        ...
        mount_root();
    out:
        devtmpfs_mount("dev");
        sys_mount(".", "/", NULL, MS_MOVE, NULL);
        sys_chroot(".");
    }
    ```
    - 在函数内部，内核会解析启动参数（如 `root=/dev/sda1`），并最终调用 `mount_root()` 或相关函数来挂载实际的根文件系统
    - `devtmpfs_mount()`: *If configured, or requested by the commandline, devtmpfs will be auto-mounted after the kernel mounted the root filesystem.*

## 运行根文件系统中的应用程序
*Linux* 内核启动后，执行根文件系统内容的起点是 *init* 进程。对应的内核入口在源码 `init/main.c` 中，核心转折点是内核线程 `kernel_init()` 调用 `run_init_process()` 函数：
```c
// init/main.c
kernel_init()
	...
	if (ramdisk_execute_command) {
		ret = run_init_process(ramdisk_execute_command);
		...
	}
	if (execute_command) {
		ret = run_init_process(execute_command);
		...	
	}
	if (!try_to_run_init_process("/sbin/init") ||
		!try_to_run_init_process("/etc/init") ||
		!try_to_run_init_process("/bin/init") ||
		!try_to_run_init_process("/bin/sh"))
		return 0;
	panic("No working init found. Try passing init=operation to kernel.  "
			"see Linux Documentation/init.txt for guidance.");
```

当根文件系统挂载完成后，内核会寻找并执行第一个用户空间程序。这个程序的具体路径取决于启动流程和内核参数：
1. 常规情况（有 *initrd/initramfs*）：内核首先执行 *initramfs* 中的 `/init`（或由 `rdinit=` 参数指定）来挂载真正的根文件系统。切换根目录后，内核会执行由 `init=` 参数指定的程序。如果未指定，则按顺序尝试 `/sbin/init`、`/etc/init`、`/bin/init`、`/bin/sh`
2. 常见发行版：在现代 *Linux* 发行版上，这个最终的 *init* 程序通常是 *systemd*，它接着负责启动后续的所有服务。

`run_init_process()` 通过 `do_execve()` 加载并执行上述的 *init* 程序，此时系统从内核态正式切换到用户态。
简单来说，内核代码入口是 `kernel_init`，它通过 `run_init_process()` 启动了根文件系统上的 *init* 程序。

## ramdisk_execute_command
`ramdisk_execute_command` 是一个内核内部的 `char *` 变量，它的值决定了内核在挂载最终根文件系统之前，尝试从早期用户空间（通常是 *initramfs/initrd*）中执行哪个程序。

1. 赋值时机与具体值
    它的赋值分为默认初始化和命令行覆盖两种情况：
   1. 默认值：`"/init"`
        在现代内核中，这个变量在编译时就被直接初始化为 `"/init"`。这意味着即使不传递任何相关参数，内核也会默认尝试在 *initramfs* 中寻找并执行 `/init` 文件。
   2. 命令行覆盖：来自 `rdinit=` 参数
        如果在内核启动命令行中指定了 `rdinit=/some/program`，内核在解析参数时会调用 `rdinit_setup()` 函数，将 `ramdisk_execute_command` 的值修改为 `=` 后面的字符串。例如，`rdinit=/bin/sh` 会让内核转而执行 *initramfs* 里的 *shell*

2. `ramdisk_execute_command` 的作用域是“早期用户空间”：
   - 它指向的程序（如 `/init`）运行在 *initramfs* 这个临时的、基于内存的文件系统中。这个程序的主要职责之一，往往就是：去探测硬件、加载驱动，然后找到并挂载真正的磁盘根文件系统，最后切换过去（*pivot_root*）
   - 如果它不存在或执行失败：内核会打印警告，随后尝试执行 `execute_command`（由 `init=` 参数指定）或默认的 `/sbin/init` 等。如果系统根本没有 *initramfs*（`ramdisk_execute_command` 指向的文件不存在），内核就会直接尝试挂载 `root=` 指定的设备作为根文件系统

3. `rdinit_setup()`
    ```c
    // init/main.c
    rdinit_setup(char *str)
    {
        ...
        ramdisk_execute_command = str;
        ...
    }
    __setup("rdinit=", rdinit_setup);
    ```
    - 关于 `"rdinit"` 的相关内容可查看：`Documentation/kernel-parameters.txt、Documentation/filesystems/ramfs-rootfs-initramfs.txt ...`
    - `rdinit_setup()` 链接到 `.init.setup` 段中。在 `arch/arm/kernel/vmlinux.lds` 有宏定义：`INIT_SETUP(16)`。该宏定义位于：`include/asm-generic/vmlinux.lds.h`，16 表示对齐边界。在这里定义了：`__setup_start` 和 `__setup_end`。在 `/init/main.c` 中使用了这两个变量：`obsolete_checksetup()`，在该函数中进一步调用到由 `__setup_start` 所涉及的设置函数
    - 调用链：`start_kernel() -> parse_args() -> unknown_bootoption() -> obsolete_checksetup()`


***
ds:
问：为了搞清楚根文件系统中第一个执行的程序，我是不是应该去查看 busybox 的相关信息（因为我的根文件系统是由 busybox 构建的）？另外，buildroot 是不是也是使用 busybox 来构建的根文件系统？
答：是的，你的思路完全正确。对于由 BusyBox 构建的根文件系统，要搞清楚第一个执行的程序，确实应该去查看 BusyBox 的 `init` 相关机制。同时，Buildroot 确实默认使用 BusyBox 来构建根文件系统的基础环境。

🔍 如何查看 BusyBox 的入口：

BusyBox 作为一个“瑞士军刀”式的二进制文件，其 `init` 功能是核心组件之一。内核启动早期用户空间时，无论是通过默认的 `/init` 还是 `rdinit=` 参数指定的路径，最终调用的很可能就是 BusyBox 的 `init` 程序（或其软链接 `linuxrc`）。

具体可以从以下两个层面入手：

1.  **确认内核实际执行了谁**：你需要结合上一轮讨论的 `ramdisk_execute_command` 逻辑。查看你的内核启动命令行（`/proc/cmdline`），看是否有 `rdinit=` 参数。如果没有，内核会默认尝试执行 initramfs 中的 `/init`。如果存在该文件且是一个指向 BusyBox 的链接，那么第一个用户空间程序就是 BusyBox 的 `init`。
2.  **查看 BusyBox 的初始化逻辑**：一旦确认是 BusyBox 在跑，它的行为由 `/etc/inittab` 文件驱动。BusyBox 的 `init_main()` 函数会解析这个文件，并执行其中 `sysinit` 等动作指定的脚本（通常是 `/etc/init.d/rcS`）。因此，你的“第一个程序”的实际工作内容，是由这个脚本定义的。

🧱 Buildroot 与 BusyBox 的关系：

是的，**Buildroot 默认使用 BusyBox**。

Buildroot 的定位是一个构建框架，它帮你自动下载、配置、编译并集成 BusyBox 及其他软件包，最终生成一个完整的根文件系统镜像。在 Buildroot 的配置中，BusyBox 是默认启用的核心包。

你可以这样理解两者的分工：

*   **BusyBox**：提供了嵌入式系统所需的大量基础命令（如 `ls`, `mount`, `sh`）和一个轻量级的 `init` 实现。
*   **Buildroot**：在集成 BusyBox 的基础上，还负责处理库文件、设备节点、以及集成你选中的其他第三方软件，省去了手动拼凑的麻烦。

因此，你基于 Buildroot 构建的系统，其用户空间的起点通常就是 Buildroot 配置并打包进去的那个 BusyBox `init`。
***