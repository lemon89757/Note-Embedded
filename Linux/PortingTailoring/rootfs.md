# 系统移植之根文件系统
## 概述
根文件系统并不是 fatfs 这样的文件系统代码。（它更像是一个运行环境，linux 的运行环境）Linux 中的根文件系统更像是一个文件夹/目录，在这个目录里会有很多的子目录，在这些目录中包含若干文件，这些文件是 Linux 运行所必须的，比如库、常用软件和命令、设备文件、配置文件等等。

内核代码镜像文件可保存在根文件系统中。

根文件系统是 linux 内核启动后挂载（mount）的第一个文件系统，系统引导启动程序会在根文件系统挂载后从中把一些基本的初始化脚本和服务等加载到内容中去运行，比如 rcS、inittab 等。

根文件系统和 linux 内核是分开的，单独的 linux 内核是没法正常工作的，必须要搭配根文件系统。

常用的 ls、mv、ifconfig 等命令（可执行文件）就保存在根文件系统中，通过构建根文件系统获取。

## rootfs 目录结构
**根文件系统的目录名字为 `/`**，在根目录下存在如下文件夹：
- `/bin`：
    bin 文件，即可执行文件，的保存目录。在此目录下存放着系统需要的可执行文件，一般都是一些命令。**此目录下的命令所有用户都可以使用**。
- `/dev`
    设备文件，此目录下的文件都是和设备相关的。
- `/etc`
    此目录下存放着各种配置文件。
- `/lib`
    库文件目录，此目录下存放着 linux 所必须的库文件，这些库文件是共享库，命令和用户编写的应用程序要使用这些库文件。
- `/mnt`
    临时挂载目录，一般是空目录。可以在此目录下创建空的子目录，例如 `/mnt/sd` 等，这样就可以将对应的设备挂载到这个子目录下。
- `/proc`
    内存文件系统 proc 所挂载的目录。proc 基于内存而存在，`/proc` 目录下的文件都是临时存在的，一般用来存储系统运行信息文件。
- `/usr`
    `usr, Unix Software Resource`，即 Unix 操作系统软件资源目录。在此目录下存放着多个软件，一般系统安装完成以后此目录占用的空间最多。
- `/var`
    此目录存放一些可以更改的数据。各种日志文件常保存在此目录下。
- `/sbin`
    此目录也存放一些可执行文件，但是此目录下的文件或命令只能由管理员使用。
- `/sys`
    内存文件系统 sys 所挂载的目录。此目录是系统设备管理的重要目录，它通过一定的组织结构向用户提供详细的内核数据结构信息。
- `/opt`
    可选的文件、软件存放区，由用户选择将哪些文件或软件存放到此目录中。

## BusyBox 构建 rootfs
### 简介
BusyBox 是一个集成了大量 linux 命令和工具的软件。

一般会下载 BusyBox 的源码，然后配置 BusyBox，选择所需的功能，最后编译即可。

[BusyBox 官网](https://busybox.net/)

### 编译 BusyBox 构建根文件系统
一般在 linux 驱动开发的时候都是通过 nfs 挂载根文件系统的，当产品最终上市时才会将根文件系统烧写到 EMMC 或 NAND 中。

1. 修改 Makefile，添加编译器
    ```Makefile
    ...
    CROSS_COMPILE ?= /path/to/compiler-pre # 例如 arm-linux-gnueabihf-
    ...
    ARCH ?= arm
    ...
    ```

2. 中文字符支持
    默认编译的 BusyBox，通过 SecureCRT 显示中文字符是不正常的，中文字符会显示为 "?"。

    修改 BusyBox 源码，取消其对中文显示的限制：
    ```c
    // path/to/BusyBox-Project/libbb/printable_string.c
    const char* FAST_FUNC printable_string(uni_stat_t *stats, const char *str) {
        char *dst;
        const char *s;

        s = str;
        while (1) {
            if (c < ' ')
                break;
            // 注释掉下面两行代码
            /* if (c >= 0x7f)
                break; */
            s++;
        }

        #if ENABLE_UNICODE_SUPPORT
        dst = unicode_conv_to_printable(stats, str);
        #else
        {
            char *d = dst = xstrdup(str);
            while (1) {
                unsigned char c = *d;
                if (c == '\0')
                    break;
                // 修改下面代码
                /* if (c < ' ' || c >= 0x7f) */
                if (c < ' ')
                    *d = '?';
                d++;
            }
        }
        ...
        #endif
        return auto_string(dst);
    }

    // path/to/BusyBox-Project/libbb/unicode.c
    static char* FAST_FUNC unicode_conv_to_printable2(uni_stat_t *stat, const char *src, unsigned width, int flags) {
        char *dst;
        unsigned dst_len;
        unsigned uni_count;
        unsigned uni_width;

        if (unicode_status != UNICODE_ON) {
            char *d;
            if (flags & UNI_FLAG_PAD) {
                d = dst = xmalloc(width + 1);
                ...
                {
                    ...
                    /* *d++ = (c >= ' ' && c < 0x7f) ? c : '?'; */
                    *d++ = (c >= ' ') ? c : '?';
                    src++;
                }
                *d = '\0';
            } else {
                d = dst = xstrndup(src, width);
                while (*d) {
                    unsigned char c = *d;
                    /* if (c < ' ' || c >= 0x7f) */
                    if (c < ' ')
                        *d = '?';
                    d++;
                }
            }
        ...
        return dst;
    }
    ```
    还需要配置 BusyBox 来使能 unicode 码。

3. 配置 BusyBox
   - 配置选项：
     1. defconfig：缺省配置，即默认配置选项；
     2. allyesconfig：全选配置，即选中 BusyBox 的所有功能；
     3. allnoconfig：最小配置。
   - 配置：
        ```bash
        make defconfig # 一般选用默认配置即可
        ```
   - 图形化配置
        ```bash
        make menuconfig
        ```
        - Location -> Setting -> Build static binary (no shared libs)，选中 Build static binary 的话就不需要库文件，但是编译出来的库会很大；不选中即表示动态编译，根文件系统中有库文件，编译出来的 busybox 会小很多。**这里不能采用静态编译，因为会导致 DNS 出问题，无法进行域名解析**。
        - 选中 vi-style line editing commands: Location -> Settings -> vi-style line editing commands.
        - 取消选中 Simplified modutils: Location -> Linux Module Utilities -> Simplified modutils.
        - 确保 mdev 下全选中：Location -> Linux System Utilities -> mdev(16kb).
        - 使能 unicode 编码以支持中文：Location -> Settings -> Support Unicdoe（选中） -> Check $LC_ALL, $LC_CTYPE and $LANG environment variables（选中）

4. 编译 BusyBox
    ```bash
    make
    make install CONFIG_PREFIX=path/to/nfs/rootfs
    ```
    - `CONFIG_PREFIX` 指定安装目录；
    - 安装完成后，在 rootfs 下会有 `bin`、`linuxrc`、`sbin`、`usr` 文件夹或文件。其中 `linuxrc` 作为 Linux 内核 init 进程最后会查找的用户空间 init 程序，设置 bootargs 中 `init=/linuxrc`。因此，用户态空间的 init 程序是 BusyBox 来生成的。

### 向根文件系统添加 lib 库
1. 向 rootfs 的 `/lib` 目录添加库文件
    Linux 中的应用程序一般都是需要动态库的。
   - 在 rootfs 中创建名为 "lib" 的文件夹；
   - `/lib` 目录下库文件的来源：交叉编译器的库文件：
     - 将交叉编译器所有的库文件都放入根文件系统中；
     - 针对需求进行**裁剪**，只将所需的库文件放入根文件系统中；
        ```bash
        # 进入交叉编译器目录
        path/to/cross_compile/arm-linux-gnueabihf/libc/lib # 例如 /usr/local/arm/gcc-linaro-xxx-arm-linux-gnueabihf/arm-linux-gnueabihf/libc/lib

        # 将所有库文件，包括静态库文件（*.a）和动态库文件（*.so）拷贝到 rootfs/lib 目录中
        cp *so* *.a path/to/rootfs/lib -d

        # 将符号链接转换为对应的文件
        rm path/to/rootfs/lib/ld-linux-armhf.so.3
        cp path/to/cross_compile/arm-linux-gnueabihf/libc/lib/ld-linux-armhf.so.3 path/to/rootfs/lib

        # 拷贝另一目录下的库文件
        cd path/to/cross_compile/arm-linux-gnueabihf/lib
        cp *so* *.a path/to/rootfs/lib -d
        ```
        - `cp -d` 表示拷贝符号链接。

2. 向 rootfs 的 `usr/lib` 目录添加库文件
   - 在 rootfs 的 `/usr` 目录下创建 "lib" 文件夹；
   - 拷贝库文件：
        ```bash
        cd path/to/cross_compile/arm-linux-gnueabihf/libc/usr/lib
        cp *so* *.a path/to/rootfs/usr/lib -d
        ```

3. 检查库目录大小
    ```bash
    du ./lib ./usr/lib -sh
    ```

### 创建其他文件夹
在根文件系统中创建其他文件夹，如 "dev"、"proc"、"mnt"、"sys"、"tmp" 和 "root" 等。

### 根文件系统测试
1. 通过 nfs 挂载根文件系统。在 Linux 内核源码文档 `Documentation/filesystems/nfs/nfsroot.txt` 中介绍了如何设置：
    ```bash
    # 格式为：
    root=/dev/nfs nfsroot=[<server-ip>:]<root-dir>[,<nfs-options>] ip=<client-ip>:<server-ip>:<gw-ip>:<netmask>:<hostname>:<device>:<autoconf>:<dns0-ip>:<dns1-ip>
    ```
    - `server-ip`：服务器 ip 地址；
    - `root-dir`：根文件系统在服务器中的目录位置，即上述提到的 `path/to/rootfs`；
    - `nfs-options`：nfs 的其他选项，一般不设置；
    - `client-ip`：客户端 ip 地址。
    - `gw-ip`：网关地址；
    - `netmask`：子网掩码，如 `255.255.255.0`；
    - `hostname`：客户机的名字，一般不设置，此值可以空着；
    - `device`：设备名，即网卡名。一般是 eth0、eth1 等等；
    - `autuconf`：自动配置，一般不使用，所以设置为 off；
    - `dns0-ip`：DNS0 服务器 ip 地址，不使用；
    - `dns1-ip`：DNS1 服务器 ip 地址，不使用；

    参考示例：
    ```bash
    root=/dev/nfs nfsroot=192.168.1.250:/home/path/to/rootfs,proto=tcp rw ip=192.168.1.251:192.168.1.250:192.168.1.1:255.255.255.0::eth0:off
    ```
    - `proto=tcp` 为选项内容，表示使用 tcp 协议；
    - `rw` 为选项内容，表示 nfs 挂载的根文件系统为可读可写的。
    
2. 设置 uboot 中 bootargs 的 root 值
   ```bash
   setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.250:/home/path/to/rootfs,proto=tcp rw ip=192.168.1.251:192.168.1.250:192.168.1.1:255.255.255.0::eth0:off'
   saveenv
   ```

3. 启动 linux 内核
    ```bash
    boot
    ```

### 根文件系统完善
- 进入根文件系统的时候会有如下错误：
    ```bash
    can't run '/etc/init.d/rcS': No such file or directory
    ```
    - 创建 `/etc/init.d/rcS` 文件：
        **rcS 是个 shell 脚本，Linux 内核启动以后需要启动一些服务，而 rcS 就是规定启动哪些文件的脚本文件**。在 rootfs 中创建 `/ect/init.d/rcS` 文件，并添加以下内容：
        ```shell
        #!/bin/sh

        PATH=/sbin:/bin:/usr/sbin:/usr/bin:$PATH
        LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/lib:/usr/lib
        export PATH LD_LIBRARY_PATH

        mount -a
        mkdir /dev/pts
        mount -t devpts devpts /dev/pts

        echo /sbin/mdev > /proc/sys/kernel/hotplug
        mdev -s
        ```
        - `mount -a`：挂载所有文件系统，这些文件系统由文件 `/etc/fstab` 来指定；
        - `mkdir /dev/pts && mount -t devpts devpts /dev/pts`：创建目录并挂载 devpts；
        - `echo /sbin/mdev > /proc/sys/kernel/hotplug && mdev -s`：使用 mdev 来管理热插拔设备。通过此部分命令，linux 内核就可以在 `/dev` 目录下自动创建设备节点。**关于 mdev 的详细内容可以参考 BusyBox 中的 `docs/mdev.txt` 文档**；
        - 复杂的 rcS 可以借助其他工具创建，比如 Buildroot 等。
    - 给予 `/etc/init.d/rcS` 可执行权限：
        ```bash
        chmod 777 rcS
        ```

- 创建 `/etc/fstab` 文件
    在 rootfs 中创建 `/etc/fstab` 文件，`fstab` 在 linux 开机以后自动配置哪些需要自动挂载的分区，格式为：
    ```bash
    <file system> <mount point> <type> <options> <dump> <pass>
    ```
    - file system: 要挂载的特殊设备，也可以是块设备，比如 `/dev/sda` 等；
    - mount point: 挂载点；
    - type: 文件系统类型，比如 ext2、ext3、proc、romfs、tmpfs 等等；
    - options: 挂载选项，可以通过查询 mount 命令用法获取，一般使用 defaults。defaults 包含了 rw、suid、dev、exec、auto、nouser 和 async。
    - dump: 为 1 的话表示允许备份，为 0 不备份。一般不备份。
    - pass: 磁盘检查设置，为 0 表示不检查。根目录设置为 1，其他的都不能设置为 1，其他的分区从 2 开始。一般不在 fstab 中挂载根目录，因此这里一般设置为 0。

    示例：fstab 文件中的内容：
    ```
    #<file system>  <mount point>   <type>  <options>   <dump>  <pass>
    proc            /proc           proc    defaults    0       0
    tmpfs           /tmp            tmpfs   defaults    0       0
    sysfs           /sys            sysfs   defaults    0       0
    ```

- 创建 `/etc/inittab` 文件
  - **initab 的详细内容可以参考 BusyBox 下的文件 `examples/inittab`**。
  - init 程序会读取 `/etc/inittab`，inittab 由若干条指令组成。每条指令的结构都是一样的，由以 ":" 分隔的 4 个段组成：
    ```bash
    <id>:<runlevels>:<action>:<process>
    ```
    - id: 每个指令的标识符，不能重复。但是对于 BusyBox 的 init 来说，id 有着特殊意义。对于 BusyBox 而言 id 用来指定启动进程的控制 tty，一般将串口或者 LCD 屏幕设置为控制 tty。
    - runlevels: 对 BusyBox 而言此项完全没用，所以空着。
    - action: 用于指定 process 可能用到的动作。BusyBox 支持的动作如下：
        - sysinit：在系统初始化的时候 process 才会执行一次；
        - respawn：当 process 终止以后马上启动一个新的；
        - askfirst：和 respawn 类似，在运行 process 之前在控制台上显示 "Please press Enter to active this console."。只有用户按下 “Enter” 键以后才会执行 process。
        - wait：告诉 init，要等待相应的进程执行完以后才能继续执行；
        - once：仅执行一次，且不会等待 process 执行完成；
        - restart：当 init 重启时才会执行 process；
        - ctrlaltdel：当按下 ctrl+alt+del 组合键才会执行 process；
        - shutdown：关机的时候执行 process；
    - process：具体的动作，比如程序、脚本或命令等。
  - 参考 BusyBox 的 `examples/inittab` 创建 `/etc/inittab`：
    ```
    #etc/inittab
    ::sysinit:/etc/init.d/rcS
    console::askfirst:-/bin/sh
    ::restart:/sbin/init
    ::ctrlaltdel:/sbin/reboot
    ::shutdown:/bin/umount -a -r
    ::shutdown:/sbin/swapoff -a
    ```
    - `::sysinit:/etc/init.d/rcS`：系统启动以后运行 `/etc/init.d/rcS` 这个脚本文件；
    - `console::askfirst:-/bin/sh`：将 console 作为控制终端，也是就 ttymxc0；
    - `::restart:/sbin/init`：重启的话运行 `/sbin/init`；
    - `::ctrlaltdel:/sbin/reboot`：按下 ctrl + alt + del 组合键的话就运行 `/sbin/reboot`；
    - `::shutdown:/bin/umount -a -r`：关机的时候卸载各个文件系统；
    - `::shutdown:/sbin/swapoff -a`：关机的时候关闭交换分区。

### 根文件系统其他功能测试
1. 软件运行测试
2. 中文字符测试
3. 开机启动测试
   ```shell
   #!/bin/sh

   # /etc/init.d/rcS
   PATH=/sbin:/bin:/usr/sbin:/usr/bin
   LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/lib:/usr/lib
   runlevel=S
   umask 022
   export PATH LD_LIBRARY_PATH runlevel

   mount -a
   mkdir /dev/pts
   mount -t devpts devpts /dev/pts

   echo /sbin/mdev > /proc/sys/kernel/hotplug
   mdev -s

   # 开机自启动
   cd /drivers
   ./hello &
   cd /
   ```
4. 外网链接测试
   ping 百度失败。

   因为是百度的网址解析失败了，并没有解析出对应的 ip 地址。需要配置域名解析服务器的 ip 地址。一般域名解析地址可以设置为所处网络的网关地址，如 192.168.1.1，也可以设置为 114.114.114.114（这时运营商的域名解析服务器地址）。

   在 rootfs 中新建文件 `/etc/resolv.conf`：
   ```
   nameserver 114.114.114.114
   nameserver 192.168.1.1
   ```
   - 如果使用 udhcpc 命令自动获取 ip 地址，udhcpc 命令会修改 nameserver 的值，一般是将其设置为对应的网关地址。

### 最后
将测试好的根文件系统进行打包保存。

## Buildroot 构建 rootfs
SDK 的类型有很多，在 Linux 中常见的 SDK 类型有 Buildroot，Yocto 或者厂家提供的一个项目开发工具包。有些 SDK 是二次或多次开发，如 Rockchip 提供的 Buildroot 版本添加了对某款芯片的软件支持（例如在 Buildroot 中添加 kernel，uboot 等 Linux 软件）。为了方便构建他们的 SDK，厂家会编写一些方便构建 SDK 的脚本，这个时候就与传统（Buildroot 原版）的构建指令有点不一样了，我们应当按照 SDK 构建手册操作，尽管我们还能使用 Buildroot 里面支持的指令。

busybox 构建的根文件系统不齐全，很多内容需要自行添加，如 lib 库文件等。在驱动开发中，很多第三方软件也需要自行移植，而它们有很多又依赖其他的库文件，导致移植过程非常的繁琐。

Buildroot 不仅集成了 busybox，而且还集成了各种常见的第三方库和软件，需要什么就选择什么。Buildroot 极大的方便了嵌入式 Linux 开发人员构建实用的根文件系统。

Buildroot 和 uboot、Linux kernel 一样也支持图形化配置。

Buildroot 不仅仅能构建根文件系统，也可以编译 Linux 内核和 uboot。当配置 Buildroot，使能 linux 内核和 uboot 以后 buildroot 就会自动下载最新的 linux 内核和 uboot 源码并编译。一般不会使用 Buildroot 下载的 Linux 内核和 uboot，因为 Buildroot 下载的官方源码，里面会缺少很多驱动问年间，而且最新的内核和 uboot 会对编译器版本号有要求，可能导致编译失败。

### 源码获取
1. 厂家提供
    根据构建手册操作。
2. 从 Buildroot 官网下载
   [Buildroot 官方下载](https://buildroot.org/download.html)

### 环境搭建
1. 配置文件选择
   - 列出 `./configs/` 目录下的配置文件：
        ```bash
        make list-defconfigs
        ```
        - 需要注意的是，在这种官方支持的配置文件是针对某个内核版本的，得根据所使用的内核版本对配置文件进行修改，否则文件系统会启动不成功；
        - 使用开发板厂家提供的配置文件；

2. 依赖软件安装
    ```bash
    sudo apt-get install gcc g++
    sudo apt-get install lsb-core lib32stdc++6 libncurses-dev
    ```

### 构建根文件系统
1. 配置
    ```bash
    # 打开图形化配置
    make menuconfig
    
    # 配置 Target options
    Target options
        -> Target Architecture          = ARM(little endian)
        -> Target Binary Format         = ELF
        -> Target Architecture Variant  = cortex-A7
        -> Target ABI                   = EABIhf
        -> Floating point strategy      = NEON/VFPv4
        -> ARM instruction set          = ARM
    
    # 配置 Toolchain (交叉编译工具链)
    # Buildroot 可以自动从国外服务器下载交叉编译器，考虑到网络环境，建议设置成本地交叉编译器
    Toolchain
        -> Toolchain type               = External toolchain
        -> Toolchain                    = Custom toolchain  // 使用本地的交叉编译器
        -> Toolchain origin             = Pre-install toolchain // 预装的编译器
        -> Toolchain path               = /usr/local/arm/gcc-linaro-xxx_arm-linux-gnueabihf
        -> Toolchain prefix             = $(ARCH)-linux-gnueabihf // 设置交叉编译器前缀
        -> External toolchain gcc version               = 4.9.x
        -> External toolchain kernel headers series     = 4.1.x
        -> External toolchain C library                 = glibc/eglibc
        -> [*] Toolchain has SSP support?(NEW)  // 选中
        -> [*] Toolchain has RPC support?(NEW)
        -> [*] Toolchain has C++ support?
        -> [*] Enable MMU support(NEW)
    
    # 配置 System configuration
    # 用于设置一些系统配置，如开发板名字、欢迎语、用户名、密码等
    System configuration
        -> System hostname              = alpha_imx6ull
        -> System banner                = Welcome to alpha i.mx6ull
        -> Init system                  = BusyBox   // 使用 busybox
        -> /dev management              = Dynamic using devtmpfs + mdev     // 使用 mdev
        -> [*] Enable root login with password(NEW) // 使能登录密码
            -> Root password            = 123456
    
    # 配置 Filesystem images
    # 配置最终制作的根文件系统的格式
    -> Filesystem images
        -> [*] ext2/3/4 root filesystem     // 如果是 EMMC 或 SD 卡的话就用 ext3/4
            -> ext2/3/4 variant         = ext4  //  选择 ext4 格式
        -> [*] ubi image containing an ubifs root filesystem    // 如果使用 NAND 的话就用 ubifs

    # 禁止编译 Linux 内核和 uboot
    -> Kernel
        -> [] Linux Kernel      // 不要选择编译 Linux kernel 选项

    -> Bootloaders
        -> [] U-Boot            // 不要选择编译 U-Boot 选项
    
    # 配置 Target packages
    # 用于配置要选择的第三方库或软件
    # 可以先什么都不选，先编译最基础的根文件系统，之后再重新配置并编译
    ```

2. 编译
    ```bash
    sudo make   # 一定要加 sudo！而且不能通过 -jx 来指定多核编译！
    ```
    - Buildroot 编译的时候会先从网上下载所需的软件源码（终端会给出下载网址），可以先提前准备好相关离线包并将其放置在 `dl` 或者 `dl/package-name` 目录下；
    - 编译完成后得到根文件系统的压缩包（位于 `buildroot-xxx/output/images`），包含 ext2、ext4、ubi 等格式，.tar 就是打包好的根文件系统，使用 .tar 进行测试（测试时解压）；

3. 测试
   - 通过 nfs 挂载，设置 bootargs 环境变量；
   - 根据提示 "can't change directory to '/lib/modules': No such file or directory" 创建此目录；

4. 第三方软件和库的配置
    ```bash
    # 使能 alsa-lib
    Target packages
        -> Libraries
            -> Audio/Sound
                -> alsa-lib      // 此配置项下的文件全部选中

    # 使能 alsa-utils
    Target packages
        -> Audio and video applications
            -> alsa-utils       // 此目录下的软件全部选中
    ```

5. Buildroot 下的 busybox 配置
   - Buildroot 在构建根文件系统的时候需要用到 busybox。Buildroot 会自动下载 busybox 压缩包并保存到 `dl` 目录下。在 `dl` 目录下就有 “busybox” 文件夹，此目录下保存着 busybox 压缩包；
   - Buildroot 将所有解压缩的软件都保存在 `output/build` 中，可以找到 `output/build/busybox-xxx` 文件夹，其中是解压后的 busybox 源码；
   - 在上述目录中修改 busybox 源码或者进行配置：
        ```bash
        # 在 output/build/busybox-xxx 目录下
        make menuconfig
        ```
   - 编译：配置好以后重新编译 Buildroot 下的 busybox
        ```bash
        # 在 Buildroot 项目顶层目录下
        sudo make show-targets      # 查看当前 Buildroot 配置了哪些目标软件包
        # 单独编译 busybox
        sudo make busybox
        # 编译完成后重新编译 Buildroot，主要是对其进行打包
        sudo make # 如果 .tar 不是最新创建的话先删掉它，然后在重新 sudo make
        ```

6. 命令行显示用户名、当前目录等信息
   - `PS1` 环境变量用于设置命令提示符格式，格式为：
        ```bash
        PS1 = '命令列表'
        ```
        命令列表可选参数：
        - `\!`：显示该命令的历史记录编号；
        - `\#`：显示当前命令的命令编号；
        - `\$`：显示 $ 符作为提示符，如果用户是 root 的话，则显示 #；
        - `\\`：显示反斜杠；
        - `\d`：显示当前日期；
        - `\h`：显示主机名；
        - `\n`：打印新行；
        - `\nnn`：显示 nnn 的八进制值；
        - `\s`：显示当前运行的 shell 的名字；
        - `\t`：显示当前时间；
        - `\u`：显示当前用户的用户名；
        - `\W`：显示当前工作目录的名字；
        - `\w`：显示当前工作目录的路径；
   - 编辑 `/etc/profile` 文件：
        ```shell
        # 屏蔽下述内容
        if [ "$PS1" ]; then
            if [ "`id -u`" -eq 0]; then
                export PS1='# '
            else
                export PS1='$ '
            fi
        fi

        # 重新设置 PS1 的值
        PS1='[\u@\h]:\w$:'      # 格式为 [user@hostname]:currentpath$:
        export PS1
        ```

### 常见问题及解决方法

## Ubuntu-base 根文件系统构建

## 参考
1. 正点原子-《I.MX6U 嵌入式 Linux 驱动开发指南》
2. 正点原子-《Buildroot 快速使用手册》
3. 正点原子-《Buildroot 用户手册（中文版）》
