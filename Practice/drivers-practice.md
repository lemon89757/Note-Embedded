# 驱动实操
## 字符设备驱动驱动开发
1. 查看当前系统中存在的模块：
    ```bash
    lsmod
    ```
2. 查看设备号的使用情况：
    ```bash
    cat /proc/devices
    ```
3. 实现设备的具体操作函数：`file_operations`：
   - `printk`；
   - `copy_from_user`、`copy_to_user`；
   - `ioremap`、`iounmap`
4. `struct file` 中有个叫做 `private_data` 的成员变量，类型为 `void *`，一般在驱动中将 `private_data` 指向设备结构体，设备结构体会存放设备的一些属性。
5. 设置文件私有数据：对于一个设备的所有属性信息最好将其做成一个结构体。编写 open 函数的时候将设备结构体作为私有数据添加到设备文件中：
    ```c
    struct test_dev {
        dev_t devid;
        struct cdev cdev;
        struct class *class;
        struct device *device;
        int major;
        int minor;
    };

    struct test_dev testdev;

    static int test_open(struct inode *inode, struct file *filp) {
        filp->private_data = &testdev;
        return 0;
    }
    ```
6. 如果在 `open` 函数中设置了 `flip` 的 `private_data` 成员（申请了资源），那么在 `release` 函数（应用程序 `close` 时调用）中就要释放掉（释放掉相关资源）。
7. 加载时，如果不存在 `/lib/modules/<kernel-version>` 文件夹，则需要自行创建，否则影响 `modprobe` 命令使用。
8. modprobe 安装驱动时报错："'modules.dep': No such file or directory"：
    ```bash
    # 通过如下命令生成 modules.dep
    depmod
    ```
    - 自动生成 modules.alias、modules.symbols 和 modules.dep 这三个文件；
    - 如果根文件系统没有这个命令，则需重新配置 busybox 使能此命令，然后重新编译。
9. 如果驱动加载后，在 `/dev` 目录下没有相应文件，可能是驱动程序并未创建，可以通过如下命令来手动创建（创建设备节点文件，目的是得到设备在用户空间中的实现）：
    ```bash
    mknod /dev/xxx c MAJOR MINOR
    ```
    - c：表示创建的是字符设备
    - 前提：确认存在目标设备并得到设备号；
10. 测试 APP 文件编译：交叉编译工具链编译、`file` 命令查看目标文件信息；
11. **注意对各函数的返回值进行判断，防止执行出错时仍继续运行**。
12. 问题：disagrees about version of symbol device_create
    - 内核版本与模块版本不一致造成，更改版本一致即可解决

## 设备树

## pinctrl 和 gpio 子系统
- [ ] 如何确定引脚的电气属性呢？

## 驱动基础
### 同步与互斥
1. 原子操作、自旋锁、信号量、互斥体：实现原理、使用场景

### 阻塞与非阻塞 I/O

### 定时器
1. `_IO` - `ioctl`
2. 安装完驱动后，第一次通过 app 打开设备文件正常运行。通过 ctrl + c 退出 app 后，再次运行 app 打开设备文件时出错：perror 输出：Invalid argument.
   - 是因为 ctrl + c 导致 app 未关闭设备文件吗？影响了后续 app 的执行？
   - 重新安全驱动并不能解决问题；
   - 重启可以。
   - 原因：再次 open 时仍然通过 gpio_request 进行了 gpio 的请求，由于之前已使用相同的 gpio，因此再次 open 时会在这一步失败。
   - 新的问题：为什么打开失败之后，led 闪烁的间隔变回默认了呢？因为 open 调用时，先执行了给设备的 timeperiod 字段赋值默认值 1000 的操作。当原来的超时时间到达时，在超时处理函数中使用了新的时间间隔，即 1000ms。

### 异步通知
1. fcntl: `F_SETFL` 和 `F_SETFD` 是不同的，前者是修改 file status flags，后者是修改 file descriptor flags。
2. File status flags: each open file descriptor has certain associated status flags, initialized by open(2) and possibly modified by fcntl(). Duplicated file descriptors (made with dup(2), fcntl(F_DUPFD), fork(2), etc.) refer to the same open file descriptor, and thus share the same file status flags.

## 中断子系统
- [ ] 硬件中断注册时，如果为共享中断，在中断处理函数中如何通过 `dev` 来进行区分呢？`dev` 的类型是 `void *` 啊，中断处理函数怎么知道传递进来的 `void *` 是什么类型的数据呢？
- [ ] 如何从设备树获取中断号以及中断触发方式？

- 通过 `cat /proc/interrupts` 查看当前系统中注册的中断情况，信息包括：中断号、CPU、触发方式和中断名等。其中，中断名可以对应到 `request_irq` 时传入的名字。
- 应用程序不断轮询读取按键，导致该程序的 CPU 占用率很高，通过 `top` 命令查看可知。

## platform
1. 通过 `ARRAY_SIZE` 计算数组个数；

## Linux 自带 LED 灯驱动

## misc 驱动

## input 子系统
### linux 自带按键驱动程序源码阅读
- [ ] probe 函数及相关内容阅读 + 理解 + 笔记
  - input_dev 的回调函数 open, close...
  - input->name = pdata->name ? : pdev->name;
  - input 的 EV_REP 的重复事件上报是怎样实现的？上报的周期又是多少？
  - sysfs_create_group 等 API
  - device_init_wakeup?
  - work 和 delay_work 的比较
  - driver.pm?
  - ...

## 多点触摸屏
1. 使能内核驱动时，多点触摸屏无响应排查：
   - 问题：在系统启动时，无相关匹配日志输出："input: EP0820M09 as /devices/platform/soc/2100000.apis-bus/21a4000.i2c/i2c-1/1-0038/input/input1"
   - 排查发现：
     - 设备树下有相应的设备节点：`/proc/device-tree/...` -> `/sys/firmware/devicetree/base/soc/aips-bus@02100000/i2c@021a4000/ft5426@38`；
     - 在总线目录下有内核驱动目录：`/sys/bus/i2c/drivers/edt_ft5x06`，但是没有设备目录：`/sys/bus/i2c/drivers/` 下没有 `1-0038`（i2c 设备命名：1-控制器编号；0038-设备在该 i2c 控制器下的地址） 目录（补充：在总线-设备-驱动模型中，只有驱动注册后才会在总线目录（`/sys/bus/xxx/drivers`）下创建入口，但对于设备而言，只有其能够与总线上的驱动匹配时才会在总线目录（`/sys/bus/xxx/devices`）下创建入口）；
     - 进一步地，查看内核日志消息：
        ```bash
        dmesg | grep -i -E "touch|probe|defer|fail|error|i2c"
        # 输出
        i2c i2c-0: IMX I2C adapter registered
        i2c i2c-0: can't use DMA
        i2c i2c-1: of_i2c: modalias failure on /soc/aips-bus@02100000/i2c@021a4000/ft5426@38
        i2c i2c-1: IMX I2C adapter registered
        i2c i2c-1: can't use DMA
        mxsfb 21c8000.lcdif: failed to find mxc display driver
        fsl-quadspi 21e0000.qspi: Freescale QuadSPI probe failed
        libphy: fec_enet_mii_bus: probed
        g_mass_storage ci_hdrc.0: failed to start g_mass_storage: -22
        i2c /dev entries driver
        # 可以看到出错处的日志为：of_i2c: modalias failure on /soc/aips-bus@02100000/
        # 根据日志内容搜索到：
        grep "modalias failure" * -nr
        # 输出
        grep: built-in.o：匹配到二进制文件
        i2c/i2c-core.c:1269:            dev_err(&adap->dev, "of_i2c: modalias failure on %s\n",
        i2c/busses/i2c-powermac.c:320:  dev_err(&adap->dev, "i2c-powermac: modalias failure"
        grep: i2c/i2c-core.o：匹配到二进制文件
        grep: i2c/built-in.o：匹配到二进制文件
        # 位于 i2c/i2c-core.c:1269
        if (of_modalias_node(node, info.type, sizeof(info.type)) < 0) {
            dev_err(&adap->dev, "of_i2c: modalias failure on %s\n", node->full_name);
            return ERR_PTR(-EINVAL);
        }
        # 而函数 of_modalias_node() 是根据设备树节点的 `compatilbe` 属性进行解析。
        # 于是查到是因为设备编辑时 compatilbe 拼写错误导致。
        ```
2. 设备树检查是否有使用冲突的引脚，可以从以下几点来进行检查：
   - PIN 脚的名字；
   - pinctrl 的名称；
   - 对 gpio 的引用；
3. 设备与驱动匹配后，通过 tslib 的 ts_test 程序进行测试，触摸屏无响应。
   - 前提：使用的驱动程序是 NXP 发布的 Linux 源码中的 edt-ft5x06.c.
   - 测试：（因为触摸屏通过中断发起处理触摸数据，因此想看看触摸显示屏时中断的变化）
        ```bash
        cat /proc/interrupts
        # 输出内容：
                  CPU0
        16:        633       GPC  55 Level     i.MX Timer Tick
        18:        113       GPC  26 Level     2020000.serial
        38:          0  gpio-mxc   9 Edge      edt-ft5206
        48:          0  gpio-mxc  19 Edge      2190000.usdhc cd
        197:          0       GPC   4 Level     20cc000.snvs:snvs-powerkey
        198:         74       GPC 120 Level     20b4000.ethernet
        199:          0       GPC 121 Level     20b4000.ethernet
        200:          0       GPC  80 Level     20bc000.wdog
        203:          0       GPC  49 Level     imx_thermal
        208:          0       GPC  19 Level     rtc alarm
        214:          0       GPC   2 Level     sdma
        219:          0       GPC  43 Level     2184000.usb
        220:         33       GPC  42 Level     2184200.usb
        221:          0       GPC 118 Level     2188000.ethernet
        222:          0       GPC 119 Level     2188000.ethernet
        223:          0       GPC  22 Level     mmc0
        224:        247       GPC  23 Level     mmc1
        225:          0       GPC  36 Level     21a0000.i2c
        226:         58       GPC  37 Level     21a4000.i2c
        229:          2       GPC   5 Level     21c8000.lcdif
        232:          1       GPC 107 Level
        233:          0       GPC  27 Level     21e8000.serial
        IPI0:          0  CPU wakeup interrupts
        IPI1:          0  Timer broadcast interrupts
        IPI2:          0  Rescheduling interrupts
        IPI3:          0  Function call interrupts
        IPI4:          0  Single function call interrupts
        IPI5:          0  CPU stop interrupts
        IPI6:          0  IRQ work interrupts
        IPI7:          0  completion interrupts
        Err:          0
        ```
        - 经过测试，触摸并不会触发中断。即上述名为 edt-ft5206 的中断次数一直为 0。（名称应该是根据 compatible 的内容而来，源码中使用的 i2c_client->name 作为名字，猜测：它应该是与 of_modalias_node 函数有关）。
        - 猜测是源码问题，但是通过正点原子自带的系统进行验证却能够正常操作。
        - 通过 [这篇博客](https://blog.51cto.com/u_13640625/5478074) 找到答案，确实是源码问题，源码与正点原子触摸屏不适配。通过查看 edt-ft5x06.c 源代码确认了这一点。
   - 源码修改：
     1. 修改设备树解析函数：添加从设备树中获取中断引脚；
     2. 修改 probe 函数，固定设置屏幕参数；
     3. 修改 probe 函数，增加中断引脚设置（gpio_request 和 gpio_direction_input）；
     4. 修改中断处理的线程函数，从正点原子源码知：触摸屏和 FT5x06 是反过来的；
     5. 修改宏定义：#define EDT_RAW_DATA_RETRIES		100 -> 10.
   - 问题：触摸屏设备是 i2c_client，它应该会自动处理从设备树获取中断等相关操作，是不是不用进行上述的中断参数也可以呢？
     - 经验证，不可以。与最初的效果一样，无响应，中断次数一直为 0。
     - 应该是需要将中断引脚设置为 input 才行，而源码中这样做了，但是默认的设备树解析函数并没有从设备树中获取中断引脚，导致该操作未运行。

## RGB LCD 驱动
### 裸机代码编写流程
1. 初始化 I.MX6ULL 的 eLCDIF 控制器，重点是 LCD 屏幕宽、高、hspw、hbp、hfp、vspw、vbp 和 vfp 等信息；
2. 初始化 LCD 像素时钟；
3. 设置 RGB LCD 显存；
4. 应用程序直接通过操作显存来操作 LCD，实现 LCD 上显示字符、图片等信息。

### 驱动相关
1. framebuffer，简称 fb，即帧缓冲。fb 是一种机制，将系统中所有跟显示有关的硬件以及软件集合起来，虚拟出一个 fb 设备。当我们编写好 LCD 驱动以后会生成一个名为 `/dev/fbX(X=0~n)` 的设备，应用程序通过访问该设备就可以访问 LCD。
2. 字符设备，`file_operations` fb_ops 定义在 `drivers/video/fbdev/core/fbmem.c` 中；
3. 驱动源码：
   - 根据设备树中 `lcdif` 节点的 `compatible` 属性性为 "fsl,imx6ul-lcdif", "fsl,imx28-lcdif" 找到对应的驱动程序为：`drivers/video/fbdev/mxsfb.c`；
   - platform 驱动；
   - 内核将所有的 Framebuffer 抽象为一个叫做 `struct fb_info` (`include/linux/fb.h`)的数据结构，其中包含了 Framebuffer 设备的完整属性和操作集合。LCD 驱动需要分配、设置、注册 `fb_info` 结构体数据；
   - probe 函数的主要工作内容：
     - 申请 fb_info;
     - 初始化 fb_info 结构体中的各个成员变量；
     - 初始化 eLCDIF 控制器；
     - 注册 fb_info。
   - [ ] 添加 framebuffer 子系统的笔记
   - [ ] 上述源码内容的阅读

### 驱动适配
1. 在内核中使能相关驱动（厂商默认已使能）：
   根据驱动源文件目录下的 Makefile 内容找到对应的配置信息为：`CONFIG_FB_MXS`，在 menuconfig 的路径如下：
    ```bash
    Device Drviers
        Graphics support
            Frame buffer Devices
                MXS LCD framebuffer support [=y]
    ```
2. 根据使用的 LCD 设备，修改设备树中的硬件参数：
   - IO 设置：
     - pinctrl_lcdif_dat: RGB LCD 的 24 根数据线；
     - pinctrl_lcdif_ctrl: RGB LCD 的 4 根控制线，包括 CLK、ENABLE、VSYNC、HSYNC；
     - pinctrl_pwm1: LCD 背光 PWM 引脚配置。
     - 修改 dat 和 ctrl 的电气属性-降低 I/O 驱动能力，防止对模拟开关影响到网络。
   - LCD 参数设置：
     - LCD 设备节点相关属性值的修改；
     - 使用到的 pinctrl；
     - bpp 大小；
     - 时序信息：像素时钟、x 轴像素个数、...
   - pwm 背光控制设置：
     - pwm 驱动源码；
     - [ ] 设备树绑定：`Documentation/devicetree/bindings/video/backlight/pwm-backlight.txt`。
3. 测试
   - 使能 Linux logo 显示：Device Drivers -> Graphics supports -> Bootup logo[y] -> Standard black and white Linux logo ~ Standard 224-color Linux logo；
   - 设置 LCD 作为终端控制台：
     - bootargs: `console=tty1 console=ttymxc0,115200 ...`;
     - `etc/inittab`：
        ```config
        ...
        tty1::askfirst:-/bin/sh
        ```
     - 通过串口让 LCD 显示信息：
        ```bash
        echo hello linux > /dev/tty1
        ```
   - LCD 背光调节
        ```bash
        cd /sys/devices/platform/backlight/backlight/backlight/
        ...
        ```
   - 禁止 LCD 自动关闭
     - 熄屏后需按键唤醒（键盘或开发板上的模拟的回车键）；
     - 关闭 10 分钟熄屏功能：`/drivers/tty/vt/vt.c`，将 `static int blankinterval = 10 * 60` 设置为 0 即可。修改完成后需重新编译内核，并使用新的镜像文件启动。
     - 通过编写 APP 并开机自启来实现：
        ```c
        // lcd_always_on.c
        #include <fcntl.h>
        #include <stdio.h>
        #include <sys/ioctl.h>

        int main(int argc, char *argc[]) {
            int fd;
            fd = open("/dev/tty1", O_RDWR);
            write(fd, "\033[9;0]", 8);
            close(fd);
            return 0;
        }
        ```
        ```bash
        # 将上述程序拷贝到根文件系统中的 /usr/bin 目录下并给予可执行权限
        # /etc/init.d/rcS
        cd /usr/bin
        ./lcd_always_on
        cd /
        ```

## rtc 驱动
- I.MX6U RTC 驱动源码阅读
  - [ ] 设备树 regmap 节点
  - [ ] probe 函数各调用 api 解析，如 `syscon_regmap_lookup_by_phandle()` 等
  - [ ] 为什么注册中断处理函数时并没有指定它的触发方式？

## 系统移植问题
1. 在 Linux 开发板上，通过 nfs 挂载 rootfs 时可以正常启动系统，但是将 rootfs 打包并烧写到开发板上启动时，就报错：
```bash
devtmpfs: error mounting -2
Freeing unused kernel memory: 432K (80906000 - 80972000)
Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.
```
根文件系统内容不完整或结构错误导致。
- 在根文件系统的上一层目录进行打包：
    ```bash
    tar -cjf rootfs_melodic.tar.bz2 ./rootfs
    ```
    - 这会导致打包后的文件中多出一个目录：`rootfs`（归档时保留了这一层目录，解压时会创建该目录），根文件系统被保存在该目录下。使用该压缩包会导致上述问题的发生。
- 在根文件系统的目录下进行打包：
    ```bash
    tar -cjf ../rootfs_melodic.tar.bz2 .
    ```
    - 通过上述命令可生成正常可用的根文件系统。
    - `tar -cjf rootfs_melodic.tar.bz2 .` 会导致：“tar: .: 在我们读入文件时文件发生了变化”的发生，因为打包时目标目录文件已发生了变化。
    - `tar -cjf rootfs_melodic.tar.bz2 ./*`：不会匹配以点开头的隐藏文件，可能跳过空目录。
