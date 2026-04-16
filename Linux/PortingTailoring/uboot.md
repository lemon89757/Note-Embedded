# 系统移植之 uboot

**uboot、Linux-kernel 和 rootfs** 这三者一起构成了一个完整的 Linux 系统，一个可以正常使用、功能完善的 Linux 系统。

一个单板启动时，uboot 先运行，它的作用是启动内核。uboot 会把内核和设备树文件都读入内存，然后启动内核。在启动内核时会把设备树在内存中的地址告诉内核。

## 简介
- uboot, Universal Bootloader，遵循 GPL 协议的开源软件。[官网](http://www.denx.de/wiki/U-Boot/)。

- Linux 系统要启动就必须需要一个 bootloader 程序，**在芯片上电以后先运行该程序来初始化 DDR 等外设，然后将 Linux 内核从 flash（NAND、NOR flash、SD、MMC 等）拷贝到 DDR 中，最后启动 Linux 内核**（bootloader 的实际工作要复杂得多，但是它最主要的工作就是启动 Linux 内核）。

- 地位相当于在 PC 中 Windows 的启动过程中的 BIOS 的地位一样。 

- 一般不会直接使用 uboot 官方提供的 uboot 源码，而是 SOC 厂商准备的。后者会将 uboot 官方源码对自己的芯片进行适配（移植），也可理解为在官方的基础上对芯片进行定制（例如，NXP 的 uboot 支持各种启动方式，比如 EMMC、NAND、NORFLASH 等，这些都是 uboot 官方所不支持的）。

- SOC 厂商提供的 uboot 是针对自家评估板的，使用其他开发板时，需要在此基础上进行修改、移植适配。

- 不同类型的 uboot：
  - uboot 官方；
  - 半导体 SOC 厂商；
  - 开发厂商；

- 相关产商的 uboot：
  - [NXP](https://github.com/nxp-imx/uboot-imx)，下载稳定分支，如 `lf_v2022.04`；

## uboot 目录内容
编译完成后，uboot 目录下的内容如下：
![](../../img/uboot/260415-174050.png)

### uboot 自带
- api：与硬件无关的 api 函数
- arch：与架构体系有关的代码
  - 关注 `/arch/arm` 文件夹；
  - `/arch/arm/mach-xxx`：与具体设备有关；
  - `/arch/arm/cpu`：具体架构 cpu 代码，例如 `armv7`。**该目录下存在链接脚本文件 `u-boot.lds`**。
- board：不同开发板的定制代码
  - uboot 移植时参考的官方评估板所在位置；
- cmd：命令相关代码
- common：通用代码
- configs：配置文件
  - `make xxx_defconfig` 时所用到的配置文件所在位置；
  - xxx_defconfig 文件，半导体或开发板厂商配置文件；
  - 基于已有的配置文件做修改定制；
- disk：磁盘分区相关代码
- doc：文档
- drivers：驱动代码
- dts：设备树
- examples：示例代码
- fs：文件系统
- include：头文件
- lib：库文件
- Licenses：许可认证相关文件
- net：网络相关代码
- post：上电自检程序
- scripts：脚本文件
- test：测试代码
- tools：工具文件夹
- config.mk：某 Makefile 会调用此文件
  - 定义了 ARCH、CPU、BOARD、VENDOR、SOC、CPUDIR 等等一些用于配置的内容（取值依赖于 .config 文件）；
  - 在 config.mk 中读取的文件有：arch/arm/config.mk、arch/arm/cpu/armv7/config.mk 等等；
- .mailmap：邮件列表
- MAINTAINERS：维护者联系方式文件
- README
  - 描述 uboot 的详细信息，包括如何编译、各文件夹含义、相应的命名等；
- Kbuild：用于生成一些和汇编有关的文件
- Kconfig：图形配置界面描述文件
- Makefile：顶层 Makefile 文件

### 编译生成
- .config：配置文件
  - 文件中都是以 `CONFIG_` 开头的配置向，用于控制某项功能是否使能。y-使能：
      ```Makefile
      #bootm 命令使能，cmd/Makefile
      ifndef CONFIG_SPL_BUILD
      # core command
      obj-y += boot.o
      obj-$(CONFIG_CMD_BOOTM) += bootm.o
      obj-y += help.o
      obj-y += version.o
      ```
      - 通过让 `CONFIG_CMD_BOOTM=y` 使 make 目标 `obj-y` 中添加了 `bootm.o` 这一项，进而编译 `cmd/bootm.c` 文件，将 bootm 相关内容被编译到 uboot 中，可以在 uboot 的使用中调用 bootm 命令（相当于使能了 bootm）；
  - 通过 `make xxx_defconfig` 对 uboot 进行配置，配置完成以后就会在 uboot 根目录下生成 .config 文件；
  - 默认情况下 .config 文件和 xxx_defconfig 内容是一样的，因为 .config 就是从 xxx_defconfig 复制过来的；
  - 如果后续自行调整了 uboot 的一些配置参数，那么这些新的配置参数就添加到了 .config 中，而是 xxx_defconfig。**相当于 xxx_defconfig 只是一些初始设置，而 .config 里面才是实时有效的配置**。
- .u-boot.xxx.cmd：一系列的命令文件；
   1. `.u-boot.cmd`
      ```bash
      cmd_u-boot := arm-linux-gnueabihf-ld.bfd -pie --gc-sections -Bstatic 
      -Ttext 0x87800000 -o u-boot -T u-boot.lds 
      arch/arm/cpu/armv7/start.o 
      --start-group arch/arm/cpu/built-in.o arch/arm/cpu/armv7/built-in.o arch/arm/imx-common/built-in.o ... net/built-in.o test/built/built-in.o test/dm/built-in.o --end-group 
      arch/arm/lib/eabi_compat.o 
      -L /usr/local/arm/gcc-linaro-.../bin/../lib/gcc/arm-linux-gnueabihf/4.9.4 -lgcc 
      -Map u-boot.map
      ```
      - `arm-linux-gnueabihf-ld.bfd`：链接工具；
      - `-pie`：
      - `--gc-section`：
      - `Bstatic`：
      - `Ttext 0x87800000`：指定生成文件的链接地址；
      - `-o u-boot`：输出的文件为 u-boot；
      - `-T u-boot.lds`：指定链接脚本文件；
      - `arch/arm/cpu/armv7/start.o`：`start.o` 位于生成的 u-boot 文件的头部，其中包含的信息与中断向量表等有关；
      - `--start-group` ~ `--end-group`：生成 u-boot 需要链接的其他文件，它们都以 `built-in.o` 命名，遵循同一套命名规则。可以通过对应文件下的 Makefile 文件找到这些 `built-in.o` 文件都是各个模块自身内部链接完后生成的文件，它们都编译到 u-boot 内部。**可以通过配置文件来设置哪些模块被编译到 u-boot 内部**；
      - `arch/arm/lib/eabi_compat.o`：u-boot 还依赖架构下的库文件；
      - `-L ...`：指定库文件路径；
      - `-lgcc`：需要链接到 `gcc` 库文件；
      - `-Map u-boot.map`：输出映射文件；
   2. `.u-boot.nodtb.bin.cmd`
      ```bash
      cmd_u-boot-nodtb.bin := arm-linux-gnueabihf-objcopy --gap-fill=0xff -j .text -j .secure_text -j .rodata -j .hash -j .data -j .got -j .got.plt -j .u_boot_list -j .rel.dyn -O binary u-boot u-boot-nodtb.bin
      ```
      - u-boot.nodtb.bin: u-boot without dtb;
      - 使用 `objcopy` 将 ELF 格式的 u-boot 文件转换为二进制的 u-boot-nodtb.bin 文件；

   3. `.u-boot.bin.cmd`
      ```
      cmd_u-boot.bin := cp u-boot-nodtb.bin u-boot.bin
      ```
   4. `.u-boot.imx.cmd`
      ```bash
      cmd_u-boot.imx := ./tools/mkimage -n
      board/freescale/mx6ull_alientek_emmc/imximage.cfg.cfgtmp -T imximage -e 0x87800000 -d u-boot.bin u-boot.imx
      ```
      - 实现在 u-boot.bin 文件的头部添加 IVT、DCD 等信息；
      - mfgtools 工具烧写的 uboot 即为 u-boot.imx；
      - IVT、DCD 等数据保存在了文件 `board/freescale/mx6ullevk/imximage-ddr512.cfg.cfgtmp` 中，工具 `mkimage` 就读取文件里面的信息，然后将其添加到 u-boot.bin 头部，从而生成 u-boot.imx 文件；
   5. `.u-boot.lds.cmd`
      - 用于生成 u-boot.lds 链接脚本；
      - uboot 根目录下的链接脚本就是源于 `arch/arm/cpu/u-boot.lds` 文件；
   6. 其他 `.u-boot.lds.xxx.cmd`：略。
- System.map：系统映射文件
- u-boot：编译出来的 ELF 格式的 uboot 镜像文件；
- u-boot.xxx
  1. u-boot.bin：编译出来的 uboot 可执行二进制镜像文件；
     - 需要注意的是：uboot 是个裸机程序，**需要在其前面加上头部（IVT、DCD 等数据）才能在 I.MX6U 上运行**。
     - [ ] 补充不同的厂家在这部分都有哪些策略、头部的作用是什么，补充在烧写部分。
  2. u-boot.imx 就是添加头部以后的 `u-boot.bin`，`u-boot.imx` 就是最终要烧写到开发板中的 uboot 镜像文件；
  3. u-boot.cfg：uboot 的另外一种配置文件；
  4. u-boot.lds：链接脚本；
  5. u-boot.map：uboot 映射文件，**通过查看此文件可以直到某个函数被链接到了哪个地址上**；
  6. u-boot.srec：S-Record 格式的镜像文件；
  7. u-boot.sym：uboot 符号文件；
  8. u-boot-nodtb.bin：和 u-boot.bin 一样，u-boot.bin 就是 u-boot-nodtb.bin 的复制文件；

## 编译和烧写
### 编译
1. 初次编译时需在 ubuntu 中安装 `ncurses` 库：
   ```bash
   sudo apt-get install libncurses5-dev
   ```
2. 初始化工作，清除工作空间原有设置（非必须）：
   ```bash
   make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean
   ```
   - `make distclean`，其他为参数部分；
   - `distclean` 可以在 `Makefile` 文件中找到对应目标，相关内容见详解；
3. 根据配置文件进行配置设置
   ```bash
   make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_ddr512_emmc_defconfig
   ```
   - `mx6ull_14x14_ddr512_emmc_defconfig` 为配置文件，位于 `./config/` 目录下。
4. 编译 uboot
   ```bash
   make V=1 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j12
   ```
   - `V`：输出编译的详细信息，在 `Makefile` 文件中能够找到该参数信息；
   - `-j12`：并行编译，加快编译速度；
   - 编译完成，通过输出信息可知**入口点（Entry Point）、加载地址（Load Address）、数据大小（Data Size）等**信息；
5. shell 脚本文件：
   ```bash
   #!/bin/bash
   make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean
   make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_ddr512_emmc_defconfig
   make V=1 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j12
   ```

### 烧写
- [ ] 不同厂商的烧写操作可能不相同，在这里进行补充

1. nxp 芯片烧写
- imx6ull
   **需要在 uboot 源码编译生成的二进制文件 `u-boot.bin` 中增加头部信息**。
  - 通过正点原子的工具进行烧写：将 uboot 烧写到 SD 卡中，然后通过 SD 卡来启动并运行 uboot：
    1. 获取可在芯片上运行的 uboot 程序：
      ```bash
      chmod 777 imxdownload
      ./imxdownload u-boot.bin /dev/sdd # 烧写到 SD 卡，不能烧写到 /dev/sda 或 sda1 设备里面
      ```
    2. 设置芯片的启动方式：BOOT 设置从 SD 卡启动，同时连接串口到电脑上（设置好串口参数并打开）；
    3. 复位开发板。

  - [ ] 通过 nxp 提供的 uuu 工具进行烧写

## 使用
### uboot 输出信息
uboot 在启动后可能会输出一些比较有用的信息：
- uboot 版本号和编译时间；
- CPU 信息，包括生产商、频率、运行频率等；
- 复位原因；
- 开发板名字；
- 内存信息；
- 提示信息；
- MMC/SD 卡控制器；
- Display 信息；
- 标准输入、输出、错误所使用的终端；
- 所使用存储器信息；
- 网口信息；

### uboot 命令
1. `help`
   - 使用 `?` 也可以。
   - 用于查看当前 uboot 所支持的命令。
   - uboot 是可配置的，需要什么命令就使能什么命令，而且还可以在 uboot 中自定义命令。
   - 通过 `help/? 命令名` 可以查看命令的详细用法；

2. 信息查询相关
   - `bdinfo`：用于查看板子信息。
   - `printenv/print`：用于输出环境变量信息（uboot 中的环境变量都是字符串）。
   - `version`：查看 uboot 版本号。

3. 环境变量操作
**一般环境变量存放在外部 flash 中，uboot 启动时会将环境变量从 flash 读取到内存中**。
   - `setenv`：用于设置或修改环境变量的值（修改的是内存中的环境变量的值）；
   - `saveenv`：用于保存修改后的环境变量（将环境变量保存到 flash 中）；

   1. 修改环境变量
      ```bash
      # 修改 uboot 启动延迟时间为 5 秒，5 秒之后启动 Linux 系统
      setenv bootdelay 5
      saveenv

      # 修改具有空格的环境变量，使用单引号括起来
      setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblklp2 rootwait rw'
      saveenv # 四组值，通过空格隔开，使用单引号表示这些内容都属于环境变量 bootargs
      ```

   2. 新建环境变量
      ```bash
      # 新建环境变量
      setenv author xxx
      saveenv
      ```

   3. 删除环境变量
      ```bash
      # 删除环境变量
      setenv author
      saveenv  # 通过给环境变量赋空值来删除环境变量
      ```

4. 内存操作命令
   **内存操作命令就是直接对 DRAM 进行读写操作的**，常用的内存操作命令有 `md`, `nm`, `mm`, `mw`, `cp` 和 `cmp`。
   1. `md` 命令
      用于显示内存值。
      ```bash
      md[.b, .w, .l] address [# of objects]
      ```
      - `.b, .w, .l`：表示以多长的内存占用为单位来显示内容，分别为 1 个字节、2 个字节、4 个字节；
      - `address`：要查看的内存起始地址；
      - `[# of objects]`：表示要查看的数据长度，单位是上述所选一致；
      - 注意：**uboot 命令中的数字都是十六进制的，不是十进制**。
      ```bash
      # 以 byte 为单位，显示 20 个字节的内容
      md.b 80000000 14

      # 以 word 为单位，显示 16 * 2 个字节的内容
      md.w 80000000 10

      # 以 long 为单位，显示 16 * 4 个字节的内容
      md.l 80000000 10
      ```
   2. `nm` 命令
      用于修改指定地址的内存值。
      ```bash
      nm [.b, .w, .l] address
      ```
      示例：
      ```bash
      nm.l 80000000
      80000000: 0500e031 ? 输入预写入的内容 # 这里输出的是输入提示，0500e031 表示 80000000 地址处的当前数据值
      ```
      - 输入完成以后按下回车，然后再输入 `q` 即可退出。
      - 修改完成以后可以通过 `md` 命令来进行确认。
   3. `mm` 命令
      也是修改指定地址内存值的，使用 `mm` 修改内存值的时候地址会自增，而使用命令 `nm` 的话不会。
   4. `mw` 命令
      用于使用一个指定的数据填充一段内存。
      ```bash
      mw [.b, .w, .l] address value [count]
      ```
      - `address`：要填充的内存起始地址；
      - `value`：要填充的数据；
      - `count`：是填充的长度（同样是十六进制）。
      ```bash
      # 以数据 0A0A0A0A 填充从内存地址 80000000 开始的 16 * 4 个字节数据
      mw.l 80000000 0A0A0A0A 10
      ```
   5. `cp` 命令
      是数据拷贝命令，用于将 DRAM 中的数据从一段内存拷贝到另一段内存中，或者把 NOR Flash 中的数据拷贝到 DRAM 中。
      ```bash
      cp [.b, .w, .l] source target count
      ```
      - `source`：源地址；
      - `target`：目的地址；
      - `count`：拷贝的长度；
      ```bash
      # 从地址 80000000 开始拷贝 16 * 4 个字节数据到 80000100 开始的地址处
      cp.l 8000000 80000100 10
      ```
   6. `cmp`
      比较命令，用于比较两段内存的数据是否相等。
      ```bash
      cmp [.b, .w, .l] addr1 addr2 count
      ```
      示例：
      ```bash
      cmp.l 80000000 80000100 10
      # 输出：
      Total of 16 word(s) were the same   # 情况 1
      word at 0x80000000 (0xf0c06f00) != word at 0x80000100 (0x3fc0f414)
      Total of 0 word(s) were the same    # 情况 2
      ```

5. 网络操作命令
   **在移植 uboot 的时候一般都要调通网络功能**，因为在移植 Linux kernel 的时候需要使用到 uboot 的网络功能做调试。

   **前期准备**：开发板和主机 PC 都连接到同一个路由器上，最后设置如下几个环境变量：
   - `ipaddr`：开发板 ip 地址，可以不设置，使用 `dhcp` 命令来从路由器获取 ip 地址；
   - `ethaddr`：开发板的 MAC 地址，一定要设置；
   - `gatewayip`：网关地址；
   - `netmask`：子网掩码；
   - `serverip`：服务器 ip 地址，也就是 ubuntu 主机 ip 地址，用于调试代码。
      ```bash
      setenv ipaddr 192.168.1.50
      setenv ethaddr b8:ae:ld:01:00:00
      setenv gatewayip 192.168.1.1
      setenv netmask 255.255.255.0
      setenv serverip 192.168.1.253
      saveenv
      ```
      - 注意：**网络地址环境变量的设置要根据实际情况而定，确保 ubuntu 主机和开发板的 ip 地址在用一个网段内**；
      - MAC 地址是一个 48bit 的地址，如果在同一个网段内有多个开发板的话，一定要保证每个开发板的 `ethaddr` 是不同的，否则通信会有问题。
  
   1. `ping` 命令
      通过 `ping` 命令验证网络通信是否正常，直接 `ping` 服务器的 ip 地址即可。
      ```bash
      ping 192.168.1.253
      ```
      - 注意：只能在 uboot 中 ping 其他机器，其他机器不能 ping uboot，因为 uboot 没有对 ping 命令做处理。如果用其他机器 ping uboot 的话会失败。
   2. `dhcp` 命令
      **`dhcp` 可用于从路由器获取 ip 地址，直接输入 `dhcp` 命令即可通过路由器获取到 ip 地址**。前提是开发板连接到路由器上，如果开发板和电脑是直连的，那么 `dhcp` 命令就会失效。
      `dhcp` 不单单是获取 ip 地址，还会通过 TFTP 来启动 Linux 内核。输入 `? dhcp` 即可查看详细信息：
      ```bash
      ? dhcp
      dhcp - boot image via network using DHCP/TFTP protocol
      Usage:
      dhcp [loadAddress] [[hostIPaddr:]bootfilename]
      ```
      因此，当只执行 `dhcp` 命令时，可能会有如下提示信息：
      ```bash
      warning: no boot file name;
      TFTP from server 192.168.1.1
      ```
   3. `nfs` 命令
      nfs, network file system，即网络文件系统。可以通过 nfs 来实现在计算机之间通过网络来分享资源，比如将 Linux 镜像和设备树文件放到 ubuntu 主机中，然后通过 nfs 将这些内容下载到开发的 DRAM 中。

      这样做的目的是方便调试 Linux 镜像和设备树，即网络调试，通过网络调试是 Linux 开发中最常用的调试方法。

      虽然半导体厂商一般都会提供一个烧写固件的软件，但是这个软件使用起来比较复杂，一般用于量产的。

      使用步骤：
      1. 开启 ubuntu 主机的 nfs 服务，创建一个 nfs 使用的目录（后面开发板通过 nfs 访问的目录）
         ```bash
         # 安装 nfs 服务
         sudo apt-get install nfs-kernel-server rpcbind

         # 配置 nfs
         sudo vi /etc/exports
         # 在后面添加内容： /home/xxx/linux/nfs *(rw,sync,no_root_squash)，前面指定 nfs 使用的目录

         # 重启 nfs 服务
         sudo /etc/init.d/nfs-kernel-server restart
         ```
      2. 在开发板上通过 nfs 命令下载数据到 DRAM 中：
         ```bash
         # nfs 命令格式
         nfs [loadAddress] [[hostIPaddr:]bootfilename]
         ```
         - `loadAddress`：要保存的 DRAM 地址；
         - `[[hostIPaddr:]bootfilename]`：要下载的文件地址；
         示例：
         ```bash
         # 将主机中的内核镜像文件下载到开发板 DRAM 的 0x80800000 地址处
         nfs 80800000 192.168.1.253:/home/xxx/linux/zImage
         ```
   4. `tftp` 命令
      同样用于网络下载，将数据保存到 DRAM 中，只是 tftp 命令使用 TFTP 协议。
      
      使用步骤：
      1. ubuntu 主机作为 TFTP 服务器，需要安装 `tftp-hpa` 和 `tftpd-hpa`：
         ```bash
         sudo apt-get install tftp-hpa tftpd-hpa
         sudo apt-get install xinetd
         ```
      2. 创建用于 tftp 传输的目录**并添加权限**：
         ```bash
         mkdir /home/xxx/linux/tftp
         chmod 777 /home/xxx/linux/tftp
         ```
         - 注意，如果不给文件夹权限的话，uboot 不能从其中下载文件。
      3. 配置 tftp
         ```bash
         # /etc/xinetd.d/tftp，没有的话自行创建，并在其中添加如下内容：
         server tftp
         {
            socket_type = dgram
            protocol    = udp
            wait        = yes
            user        = root
            server      = /usr/sbin/in.tftpd
            server_args = -s /home/xxx/linux/tftp/
            disable     = no
            per_source  = 11
            cps         = 100 2
            flags       = IPv4
         }
         ```
      4. 启动 tftp 服务
         ```bash
         sudo service tftpd-hpa start
         ```
      5. 修改 `tftpd-hpa` 文件：
         ```bash
         # /etc/defualt/tftpd-hpa

         TFTP_USERNAME="tftp"
         TFTP_DIRECTORY="/home/xxx/linux/tftp"
         TFTP_ADDRESS=":69"
         TFTP_OPTIONS="-l -c -s"
         ```
      6. 重启 tftp 服务器
         ```bash
         sudo service tftpd-hpa restart
         ```
      7. 将文件拷贝到事先配置好的目录下，**并给予相应的权限**
         ```bash
         cp zImage /home/xxx/linux/tftp
         cd /home/xxx/linux/tftp
         chmod 777 zImage
         ```
      8. 在开发板中通过 tftp 命令下载文件
         ```bash
         # 命令格式
         tftpboot [loadAddress] [[hostIPaddr:]bootfilename]

         # 示例
         tftp 80800000 zImage # tftp 不需要输入文件的绝对路径，只需要输入文件名即可
         ```
6. EMMC 和 SD 卡操作命令
   一般认为 EMMC 和 SD 卡是同一个东西，uboot 中常用于操作 MMC 设备的命令为 `mmc`。

   通过 `? mmc` 命令查看详细用法。

   可以通过该命令可执行：输出 MMC 设备信息、读取其中的数据、向其中写入数据、扫描 MMC 设备、列出 MMC 分区信息、切换 MMC 设备、列出当前有效的所有 MMC 设备、设置 MMC 设备的分区、复位 MMC 设备、设置 DSR 寄存器的值等。设置指定 MMC 设备的 BOOT_BUS_WIDTH 域的值、设置指定 MMC 设备的 boot 和 RPMB 分区的大小、设置指定 MMC 设备的 PATITION_CONFIG 域的值。

   1. 切换当前 MMC 设备
      ```bash
      mmc dev [dev] [part] # dev-设备号，part-分区号，不写的话默认为 0
      mmc dev 0
      mmc dev 1 2
      ```
   2. 查看分区
      ```bash
      mmc part
      ```
      - EMMC 3 个分区，第 0 个分区存放 uboot，第 1 个分区存放 Linux 镜像文件和设备树，第 2 个分区存放根文件系统；
      - 如果通过 `mmc part` 只显示两个分区，是因为第 0 个分区没有格式化，所以识别不出来，但实际上是存在的；
      - 一个新的 SD 卡默认只有一个分区，那就是分区 0（前面 uboot 烧写到 SD 卡，其实就是将 `u-boot.bin` 烧写到 SD 卡的分区 0 里面）；
      - Linux 内核移植时，在 SD 卡中创建并格式化第二个分区，将 Linux 镜像文件和设备树文件存放到第二个分区中；
   3. 读取和写入
      ```bash
      # 读取 mmc 设备的数据
      mmc read addr blk# cnt

      # 写入数据
      mmc write addr blk# cnt
      ```
      - `addr`：数据读取到 DRAM 中的地址；
      - `blk`：要读取的块起始地址（十六进制），一个块 512 个字节，同扇区一个意思；
      - `cnt` 要读取的块数量（十六进制）；

      示例：
      ```bash
      # 从当前所在 mmc 设备的 0x600 个块开始，读取 16 个块的数据到 DRAM 的 0x80800000 地址处
      mmc read 80800000 600 10 # 0x600 * 512 是 EMMC 核心板 uboot 环境变量的存储起始地址

      # 通过写入命令来将更新的 uboot 保存到 SD 卡中 
      tftp 80800000 u-boot.imx
      mmc dev 0 0
      mmc write 80800000 2 2E6

      # 更新 emmc 中的 uboot
      mmc dev 1 0
      tftp 80800000 u-boot.imx
      mmc write 80800000 2 32E
      mmc partconf 1 1 0 0 # 分区配置，emmc 需要这一步
      ```
      - 注意：**千万不要擦除 SD 卡或者 EMMC 的前两个块（扇区），里面保存着分区表**；
   4. 擦除指令
      ```bash
      mmc erase blk# cnt
      ```
7. FAT 格式文件系统操作指令
   1. `fatinfo` 命令
      用于查询指定 MMC 设备分区的文件系统信息。
      ```bash
      fatinfo <interface> <dev[:part]>
      ```
      - `interface`：表示接口，如 mmc；
      - `dev`：设备号；
      - `part`：分区；
      示例：
      ```bash
      # 查询 emmc 分区 1 的文件系统信息
      fatinfo mmc 1:1
      ```
   2. `fatls` 命令
      用于查询 FAT 格式设备的目录和文件信息。
      ```bash
      fatls <interface> [<dev[:part]>] [directory]
      ```
      - `directory`：要查询的目录
      示例：
      ```bash
      # 查询 emmc 分区 1 中的所有目录和文件
      fatls mmc 1:1
      ```
   3. `fstype` 命令
      用于查看 MMC 设备某个分区的文件系统格式。
      ```bash
      fstype <interface> <dev>:<part>
      # 示例
      fstype mmc 1:0 # 输出示例：fat
      ```
   4. `fatload` 命令
      用于将指定的文件读取到 DRAM 中。
      ```bash
      fatload <interface> [<dev[:part]> [<addr> [<filename> [bytes [pos]]]]]
      ```
      - `addr`：保存到 DRAM 中的起始地址；
      - `filename`：要读取的文件名字；
      - `bytes`：读取多少个字节，0 或者省略表示读取整个文件；
      - `pos`：相对于文件首地址的偏移，0 或者省略表示从文件首地址开始读取；
      示例：
      ```bash
      # 将 emmc 分区 1 中的 zImage 读取到 DRAM 中的 0x80800000 地址处
      fatload mmc 1:1 80800000 zImage
      ```
   5. `fatwrite` 命令
      uboot 默认没有使能 `fatwrite` 命令，需要修改板子配置头文件，比如 `mx6ullevk.h`、`mx6ull_alientek_emmc.h` 等。找到开发板对应的配置头文件添加如下内容：
      ```c
      #define CONFIG_FAT_WRITE   // 使能 fatwrite
      ```
      `fatwrite` 命令用于将 DRAM 中的数据写入到 MMC 设备中：
      ```bash
      fatwrite <interface> <dev[:part]> <addr> <filename> <bytes>

      # 示例，通过 fatwrite 命令在 uboot 中更新 linux 镜像文件和设备树
      fatwrite mmc 1:1 80800000 zImage 6788f8 # 通过网络命令下载时可知文件大小
      ```
8. EXT 格式文件系统操作命令
   uboot 有 ext2 和 ext4 这两种格式的文件系统的操作命令。常用命令 `ext2load`、`ext2ls`、`ext4load`、`ext4ls` 和 `ext4write`，用法与 fat 一样。
9. NAND 操作命令
   1. `nand info`：查看 NAND Flash 信息；
   2. `nand device`：用于切换 NAND Flash。不过一般情况下 CPU 只有一个 NAND 接口（只有一个 NAND Flash）。
   3. `nand erase`：**NAND Flash 的特性决定了在向 NAND Flash 写数据之前一定要先对要写入的区域进行擦除**。
      ```bash
      nand erase[.spread] [clean] off size # 从指定地址（off）开始，擦除指定大小（size）的区域；
      nand erase.part [clean] partition # 擦除指定的分区
      nand erase.chip [clean] # 全篇擦除
      ```
   4. `nand write`
      ```bash
      nand write addr off size
      ```
      - `addr`：要写入的数据首地址；
      - `off`：NAND 中的目的地址；
      - `size`：要写入的数据大小；
      - 注意：IMX6ULL 要求 NAND 对应的 uboot 可执行文件还需要另外包含 BCB 和 DBBT，因此 u-boot.imx 不能直接烧写到 NAND 中。在使用 `mfgtool` 烧写系统到 NAND 里面时，`mfgtool` 会使用一个叫做 `kogs-ng` 的工具完成 BCB 和 DBBT 的添加。
      - 可以使用 `nand write` 烧写 kernel 和 dtb。烧写之前需要先对 NAND 进行分区，即规划好 uboot、linux-kernel、设备树和根文件系统的存储区域：
         ```
         0x000000000000~0x0000003fffff：boot，4MB
         0x000000400000~0x00000041ffff: env，环境变量，128KB
         0x000000420000~0x00000051ffff: logo，启动图标
         0x000000520000~0x00000061ffff: dtb，1MB
         0x000000620000~0x000000e1ffff: kernel，8MB
         0x000000e20000~0x000020000000: rootfs
         ```
      - kernel 和 dtb 的更新
         ```bash
         # kernel
         tftp 0x87800000 zImage
         nand erase 0x620000 0x800000
         nand write 0x87800000 0x620000 0x800000
         # dtb
         tftp 0x87800000 imx6ull-14x14-emmc-7-1024x600-c.dtb
         nand erase 0x520000 0x100000
         nand write 0x87800000 0x520000 0x100000
         ```
      - 根文件系统就不要在 uboot 中更新了，还是使用 NXP 提供的 `mfgtool` 工具来烧写，因为根文件系统太大，很可能超过开发板 DRAM 的大小，这样连下载都没办法。
   5. `nand read`: 命令格式：`nand read addr off size`；
10. 设备树指令
   `fdt` 命令
   ```bash
   # 首先将设备树文件读取到内存中，假设读取地址为 0x83000000

   # 设置 fdt 地址
   fdt addr 83000000

   # 查看设备树的头信息
   fdt headr

   # 查看设备树文件的内容
   fdt print
   ```
11. BOOT 操作命令
    - uboot 的本质工作是引导 Linux，常用的跟 boot 相关的命令有：`bootz`、`bootm` 和 `boot`。

    - 要启动 Linux，需要先将 Linux 镜像文件拷贝到 DRAM 中，如果使用设备树的话，也需要将设备树拷贝到 DRAM 中。
      - 从 EMMC 或者 NAND 等存储设备中拷贝；
      - 通过 nfs 或 tftp 从网络设备中拷贝；

    1.  `bootz` 命令
    ```bash
    # 命令格式
    bootz [addr [initrd[:size]] [fdt]]
    ```
    - `addr`：Linux 镜像文件在 DRAM 中的位置；
    - `initrd`：initrd 文件在 DRAM 中的地址，如果不使用 initrd 的话，使用 `-` 代替即可；
    - `fdt`：设备树文件在 DRAM 中的地址；
    示例：
    ```bash
    # 通过网络启动 Linux
    tftp 80800000 zImage
    tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb
    bootz 80800000 - 83000000

    # 通过 emmc 启动 Linux
    # 先使用 fatls、fatwrite 命令确保在 emmc 中存在 Linux 镜像文件和设备树文件
    fatload mmc 1:1 80800000 zImage
    fatload mmc 1:1 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb
    bootz 80800000 - 83000000
    ```
    2.  `bootm` 命令
      和 `bootz` 功能类似，但是 `bootm` 用于启动 uImage 镜像文件。
      ```bash
      # 不使用设备树
      bootm addr

      # 使用设备树
      bootm [addr [initrd[:size]] [fdt]]
      ```
      - `addr`：uImage 镜像在 DRAM 中的首地址；
      - 其他参数同 `bootz`；
    3.  `boot` 命令
    `boot` 命令也是用来启动 Linux 系统的，**只是 `boot` 会读取环境变量 `bootcmd` 来启动 Linux 系统**。示例：
    ```bash
    # 通过 boot 命令直接从网络启动 Linux
    setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; bootz 80800000 - 83000000'
    saveenv
    boot

    # 通过 boot 命令直接从 emmc 启动 Linux
    setenv bootcmd 'fatload mmc 1:1 80800000 zImage; fatload mmc 1:1 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; bootz 80800000 - 83000000'
    saveenv
    boot
    ```
    - **uboot 倒计时结束以后就会启动 Linux 系统，其实就是执行的 `bootcmd` 中的启动命令**；
    - 只要不修改 `bootcmd` 中的内容，以后每次开机 uboot 倒计时结束后都会使用相同的命令来启动 Linux；
    - 在启动 Linux 内核时可能出现没有找到根文件系统的错误，这是因为没有设置 uboot 的 `bootargs` 环境变量。
     ```bash
     "kernel panic - not Syncing: VFS: Unable to mount root fs on unknown-block(0, 0)"
     ```
12. 其他命令
    1.  `reset` 命令：复位重启；
    2.  `go` 命令：用于跳到指定的地址处执行应用：
      ```bash
      go addr [arg ...]
      ```
      - 此时使用的可执行文件不需要再添加头部信息，因为 uboot 已经初始化好了 DDR；
      示例程序：
      ```bash
      tftp 87800000 printf.bin
      go 87800000
      ```
    3.  `run` 命令：**用于运行环境变量中定义的命令**，例如，可以通过 `run bootcmd` 来运行 `bootcmd` 中的启动命令。示例：
      ```bash
      # 在环境变量中保存多种启动方式，后续只需通过 run 即可快速地以不同的方式启动
      setenv mybootemmc 'fatload mmc 1:1 80800000 zImage; fatload mmc 1:1 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; bootz 80800000 - 83000000'
      setenv mybootnand 'nand read 80800000 4000000 800000; nand read 83000000 6000000 100000; bootz 80800000 - 83000000'
      setenv mybootnet 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; bootz 80800000 - 83000000'
      saveenv

      # 启动 Linux
      run mybootemmc
      # 或
      run mybootnand
      # 或
      run mybootnet
      ```
    4.  `mtest`：内存读写测试命令。可以用来测试开发板上的 DDR，格式：
      ```bash
      mtest [start [end [pattern [iterations]]]]
      ```
      - `start`：要测试的 DRAM 开始地址；
      - `end`：结束地址；

## Makefile 详解

### Makefile 基础
- make 是支持递归调用的，即在 Makefile 中使用 make 命令来执行其他的 Makefile 文件，一般都是子目录中的 Makefile 文件。
   ```Makefile
   # 在主目录中通过下述方式编译子目录
   $(MAKE) -C subdir
   ```
   - `$(MAKE)`：调用 make 命令；
   - `-C`：指定目录；

- 变量传递
  向子 make 传递变量，通过 `export` 来导出要传递给子 make 的变量。如果不希望导出，则使用 `unexport` 来声明：
  ```Makefile
  export VARIABLE
  unexport VARIABLE
  ```
  - 注意：`SHELL` 和 `MAKEFLAGS` 这两个变量除非使用 `unexport` 声明，否则的话，在整个 make 的执行过程中，它们的值始终自动地传递给子 make。

- 设置编译结果输出目录
  ```Makefile
  make O=out
  ```
  - 设置目标文件输出到 out 目录中，以实现源文件和编译产生的文件分开；
  - 顶层 Makefile 有这部分的说明：`kbuild supports saving output files in a separate directory.`；

- 代码检查
  - `make C=1`，使能代码检查，检查需要重新编译的文件；
  - `make C=2`，用于检查所有的源码文件；
  - 顶层 Makefile 有这部分的说明：`Call a source code checker...`

- 获取主机架构信息
   ```Makefile
   HOSTARCH := $(shell uname -m | \
            sed -e s/i.86/x86/ \
            -e s/sun4u/sparc64/ \
            -e s/arm.*/arm/ \
            -e s/sa110/arm/ \
            -e s/ppc64/powerpc/ \
            -e s/ppc/powerpc/ \
            -e s/macppc/powerpc/ \
            -e s/sh.*/sh/)
   HOSTOS := $(shell uname -s | tr '[:upper:]' '[:lower:]' | \
            sed -e 's/\(cygwin\).*/cygwin/')
   
   export HOSTARCH HOSTOS
   ```

- 目标架构、交叉编译器和配置文件
   ```Makefile
   # set default to nothing for native builds
   ifeq ($(HOSTARCH), $(ARCH))
   CROSS_COMPILE ?= 
   endif

   KCONFIG_CONFIG ?= .config
   export KCONFIG_CONFIG
   ```
   - 可以在这里直接将 ARCH 和 CROSS_COMPILE 的内容固定（定义在顶层 Makefile）：
      ```Makefile
      ...
      ARCH ?= arm
      CROSS_COMPILE ?= arm-linux-gnueabihf-
      ...
      ```

- 交叉编译工具变量设置
   ```Makefile
   AS    = $(CROSS_COMPILE)as
   # Always use GNU ld
   ifneq ($(shell $(CROSS_COMPILE)ld.bfd -v 2> /dev/null),)
   LD    = $(CROSS_COMPILE)ld.bfd
   else
   LD    = $(CROSS_COMPILE)ld
   ...
   ```

- 导出其他变量
   ```Makefile
   export VERSION ...
   ...
   ```
   - 有些变量没有定义在顶层 Makefile 中；
   - 在 config.mk 中有定义。`ARCH := $(CONFIG_SYS_ARCH:"%"=%)`，即提取 `CONFIG_SYS_ARCH` 里面的双引号之间的内容，如果 `CONFIG_SYS_ARCH="arm"` 的话，`ARCH=arm`；
     - .config 中有定义 CONFIG_SYS_ARCH、CONFIG_SYS_CPU、CONFIG_SYS_BOARD、CONFIG_SYS_VENDOR 和 CONFIG_SYS_SOC 这些变量的值；

- `MAKEFLAGS += -rR --include-dir=$(CURDIR)`
  - 给 `MAKEFLAGS` 追加内容；
  - `-rR` 表示禁止使用内置的隐含规则和变量定义；
  - `--include-dir`：指定所搜路径；
  - `$(CURDIR)`：表示当前目录；

- `origin` 函数和 make 的 `V` 选项
   用于告诉变量的来源，语法为 `$(origin <variable>)`，示例：
   ```Makefile
   ifeq ("$(origin V)", "command line")
      KBUILD_VERBOSE = $(V)
   endif
   ifndef KBUILD_VERBOSE
      KBUILD_VERBOSE = 0
   endif

   ifeq ($(KBUILD_VERBOSE), 1)
      quiet = 
      Q = 
   else
      quiet=quiet_
      Q=@
   ```
   - `V`：verbose，冗长的、繁琐的。控制 make 是否输出编译的详细信息，即 `make V=1`；
   - 实际上，通过 `V` 控制的是，`quiet` 和 `Q` 的值：
      ```Makefile
      $(Q)$(MAKE) $(build)=tools
      ```
      - `make` 在执行的时候默认会在终端输出命令，如果在命令前加上 `@` 则不会输出（`@make` 不会在终端输出命令消息）；
      ```Makefile
      quiet_cmd_sym ?= SYM    $@
      cmd_sys ?= $(OBJDUMP) -t $<> $@
      ```
      - 有些命令会有两个版本。如 `sym` 命令分为 `quiet_cmd_sym` 和 `cmd_sym` 两个版本，功能一样，区别在于 make 执行时输出的命令不同。前者输出信息少，即短命令；后者输出信息多，即完整命令；
      - `quiet` 为空的话，整个命令都会输出；为 `quiet_` 的话，仅输出短版本；为 `silent_` 的话，整个命令都不会输出；
      - 静默输出 `make -s`；

- `filer` 函数
   ```Makefile
   $(filter <pattern...>, <text>)
   ```
   - 功能：以 `pattern` 模式过滤 `text` 字符串中的内容，仅保留符合 `pattern` 的内容；
   - 可以有多个模式；
   - 返回 `text` 中符合 `pattern` 的内容；

- `firstword` 函数
   ```Makefile
   $(firstword <text>)
   ```
   - 用于取出 `text` 字符串中的第一个单词并返回；

- `patsubst` 函数
   ```Makefile
   $(patsubst <pattern>,<replacement>,<text>)
   ```
   - 功能：用于在 `text` 中查找符合 `pattern` 的部分，如果匹配的话就用 `replacement` 替换掉。
   - `pattern` 是可以包含通配符 `%`，如果 `replacement` 中也包含通配符 `%`，那么 `replacement` 中的 `%` 将是 `pattern` 中的 `%` 所代表的字符串；
   - 函数返回值为替换后的字符串；

- `wildcard` 函数

- Makefile 调试
   ```Makefile
   mytest:
      @echo 'firstword=' $(firstword x$(MAKEFLAGS))
   ```

### 单独编译
1. 单独编译模块
   在 uboot 中允许单独编译某个模块，使用 `make M=dir` 即可，旧语法 `SUBDIRS=dir` 也是支持的。
   - 顶层 Makefile 有这部分的说明：`Use M=dir to specify directory of external module to build, old syntax make ... SUBDIRS=$PWD is still supported.`
   - Makefile：
      ```Makefile
      PHONY += all
      ifeq ($(KBUILD_EXTMOD),)
      _all: all
      else
      _all: modules
      endif
      ```
     - `KBUILD_EXTMOD` 被 make 的 M 和 SUBDIR 选项控制，当两者中的一个不为空时，`KBUILD_EXTMOD` 不为空。
   - Makefile: `U-Boot does not need modules`；

2. 单独编译设备树


### kbuild
- 顶层 Makefile 会调用文件 `scripts/Kbuild.include` 文件：
   ```Makefile
   # we neet some generic definitions (do not try to remake the file)
   scripts/Kbuild.include:;
   include scripts/Kbuild.include
   ```
   - 通过 include 包含 `scripts/Kbuild.include` 中的内容；
   - `scripts/Kbuild.include` 文件中包含的内容（部分）：
      ```Makefile
      ###
      # Shorthand for $(Q)$(MAKE) -f scripts/Makefile.build obj=
      # Usage:
      # $(Q)$(MAKE) $(build)=dir
      build := -f $(srctree)/scripts/Makefile.build obj
      ```
      - `make -f FILE`、`make --file=FILE`、`make --makefile=FILE`：read `FILE` as a makefile.
      - `build := -f $(srctree)/scripts/Makefile.build obj`，当 `srctree` 为 `.` 时，结合用法 `$(Q)$(MAKE) $(build)=dir`，实现的功能就是：
        - **读取 `./scripts/Makefile.build` 作为一个 Makefile 文件**；
        - **`obj` 作为变量传入编译过程（在 `Makefile.build` 中有使用），从而通过`Makefile.build` 包含与 `obj` 相关的 Kbuild/Makefile 文件，前者优先级更高**；
        - **另外，如果传入一个编译目标，可以从与 `obj` 相关的 Kbuild/Makefile 文件中寻找匹配信息**；

- `Makefile.build`（部分）：
  ```Makefile
  # Modified for U-Boot
  prefix := tpl
  src := $(patsubst $(prefix)/%,%,$(obj))
  ifeq ($(obj),$(src))
  prefix := spl
  src := $(patsubst $(prefix)/%,%,$(obj))
  ifeq ($(obj),$(src))
  prefix := .
  endif
  endif

  PHONY := __build
  __build:

  ...

  # The filename Kbuild has precedence over Makefile
  kbuild-dir := $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
  kbuild-file := $(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
  include $(kbuild-file)

  ...
  __build: $(if $(KBUILD_BUILTIN),$(builtin-target) $(lib-target) $(extra-y)) \
	 $(if $(KBUILD_MODULES),$(obj-m) $(modorder-target)) \
	 $(subdir-ym) $(always)
	@:
  ```
  - `obj` 作为参数在编译过程中传递，在这里是定义 `src` 的依赖（多数情况下两者相等）；
  - 编译过程中会包含 `kbuild-file` 文件，而该变量表示的文件与 `src` -> `obj` 有关； 
  - `__build`：`Makefile.build` 文件的默认目标，当 `$(Q)$(MAKE) $(build)=dir` 命令没有指定编译目标时，使用它作为默认。
    - 从顶层 Makefile 知 `KBUILD_BUILTIN` 和 `KBUILD_MODULES` 分别为 1 和 0，因此，`__build` 依赖 `$(builtin-target) $(lib-target) $(extra-y)) $(subdir-ym) $(always)`。
    - 可以在 `__build` 处通过打印，获取这些变量的值，只有 `always = scripts/basic/fixdep` 有效。因此默认目标最终是生成 fixdep 这个软件。

### make xxx_defconfig 过程
1. 前提：`include scripts/Kbuild.include`。

2. `xxx_defconfig` 的匹配目标：
   ```Makefile
   config: scripts_basic outputmakefile FORCE
	   $(Q)$(MAKE) $(build)=scripts/kconfig $@

   # 匹配到下面这项
   %config: scripts_basic outputmakefile FORCE
	   $(Q)$(MAKE) $(build)=scripts/kconfig $@
   ```
   - `%config` 依赖于 `scripts_basic`、`outputmakefile` 和 `FORCE`；
   - `$(Q)$(MAKE) $(build)=scripts/kconfig $@` -> `make -f $(srctree)/scripts/Makefile.build obj=scripts/kconfig xxx_defconfig`（`$@` 表示目标）。于是该命令的功能就是读取 `Makefiel.build` 作为一个 Makefile 文件，并包含 `scripts/kconfig/makefile`，同时指定编译目标为 `xxx_defconfig`。编译目标能够在 `scripts/kconfig/makefile` 中找到匹配的项：
      ```Makefile
      defconfig: $(obj)/conf
      ifeq ($(KBUILD_DEFCONFIG),)
         $< $(silent) --defconfig $(Kconfig)
      else
         @$(kecho) "*** Default configuration is based on '$(KBUILD_DEFCONFIG)'"
         $(Q)$< $(silent) --defconfig=arch/$(SRCARCH)/configs/$(KBUILD_DEFCONFIG) $(Kconfig)
      endif

      %_defconfig: $(obj)/conf
         $(Q)$< $(silent) --defconfig=arch/$(SRCARCH)/configs/$@ $(Kconfig)

      # Added for U-Boot (backward compatibility)
      %_config: %_defconfig
         @:

      configfiles=$(wildcard $(srctree)/kernel/configs/$@ $(srctree)/arch/$(SRCARCH)/configs/$@)

      %.config: $(obj)/conf
         $(if $(call configfiles),, $(error No configuration exists for this target on this architecture))
         $(Q)$(CONFIG_SHELL) $(srctree)/scripts/kconfig/merge_config.sh -m .config $(configfiles)
         +$(Q)yes "" | $(MAKE) -f $(srctree)/Makefile oldconfig
      ```
      - 这里会将 `xxx_defconfig` 中的配置输出到 `.config` 文件中，最终生成 uboot 根目录下的 `.config` 文件。

3. `FORCE`
   ```Makefile
   PHONY += FORCE
   FORCE:

   ```
   - 目标 `FORCE` 是没有规则和依赖的，所以每次都会重新生成 `FORCE`。**当 `FORCE` 作为其他目标的依赖时，由于 `FORCE` 总是被更新过的，因此依赖所在的规则总是会执行的**。

4. `scripts_basic`
   ```Makefile
   # Basic helpers built in scripts/
   PHONY += scripts_basic
   scripts_basic:
      $(Q)$(MAKE) $(build)=scripts/basic
      $(Q)rm -f .tmp_quiet_recordmcount

   # To avoid any implicit rule to kick in, define an empty command.
   scripts/basic/%: scripts_basic ;
   ```
   - `$(Q)$(MAKE) $(build)=scripts/basic` -> `make -f ./scripts/Makefile.build obj=scripts/basic`，其中 `srctree` 经过分析为 `.`。于是，这个命令的功能就是在读取 `./scripts/Makefile.build` 作为一个 Makefile 文件的前提下，包含 `scripts/basic/Makefile`，编译目标为默认。

5. `outputmakefile`
   ```Makefile
   PHONY += outputmakefile
   # outputmakefile generates a Makefile in the output directory, if using a
   # separate output directory. This allows convenient use of make in the
   # output directory.
   outputmakefile:
   ifneq ($(KBUILD_SRC),)
      $(Q)ln -fsn $(srctree) source
      $(Q)$(CONFIG_SHELL) $(srctree)/scripts/mkmakefile \
         $(srctree) $(objtree) $(VERSION) $(PATCHLEVEL)
   endif
   ```
   - 只有 `KBUILD_SRC` 不为空时，`outputmakefile` 才有意义。`KBUILD_SRC` 为空，`outputmakefile` 无意义；

6. 总结
   - 通过如下命令查看编译的详细信息：
      ```bash
      make mx6ull_14x14_ddr512_emmc_defconfig V=1
      ```
   - 编译过程：
     - `./scripts/basic`：生成 `scripts/basic/fixdep` 这个软件；
     - 基于 `./scripts/kconfig/Makefile`，编译 `xxx_defconfig` 这个目标；

### make 流程

## 启动流程