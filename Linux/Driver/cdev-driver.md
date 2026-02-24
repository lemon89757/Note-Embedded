# 字符设备驱动
- 在 Linux 中一切皆文件，驱动加载成功以后会在 `/dev` 目录下生成一个相应的文件；（前提：这个文件的生成需要在驱动程序中调用相应函数，详情见后续“关于 file_operations 的两种注册方式”）

- 应用程序通过对这个名为 `/dev/xxx` 的文件进行操作即可实现对硬件的相应操作；

## 从应用程序开始的调用链
- [ ] 添加函数调用关系图

## 文件句柄和 file 结构体的关系
### file 结构体
- [ ] 添加相应的关系图

## 驱动运行的两种方式
- 编译到内核
  - 内核启动时自动运行驱动程序；
- 以模块方式加载
  - 文件扩展名为 `.ko`，在 Linux 运行时可通过 `insmod` 命令动态加载驱动模块；

驱动开发时一般以模块方式进行，完成后可将其编译到内核中。

## Linux 设备号
- [ ] 设备号是 Linux 设备的唯一标识吗？

- 设备号由主设备号和次设备号组成；
- 类型为 `dev_t`（`include/linux/types.h`、`include/uapi/asm-generic/int_ll64.h`），实际类型为 `unsigned int`，为一个 32 位数据类型；
- 高 12 位为主设备号（因此主设备号范围为`0~4095`）、低 20 位为次设备号；
- 相关宏定义（`include/linux/kdev_t.h`）：
    ```c
    #define MINORBITS   20
    #define MINORMASK   ((1u << MINORBITS) - 1)
    #define MAJOR(dev)  ((unsigned int)((dev) >> MINORBITS))
    #define MINOR(dev)  ((unsigned int)((dev) & MINORMASK))
    #define MKDEV(ma, mi)   (((ma) << MINORBITS) | (mi))
    ```
- （主）设备号的分配
  - 静态分配：易造成冲突，驱动开发静态指定，参考 `/Documentation/device.txt`，实际使用情况还是通过 `cat /proc/devices` 命令查看；
  - 动态分配：在注册设备之前先申请一个设备号，系统会自动给出一个未被使用的设备号；设备号申请与注销函数：
    ```c
    /* 
     * baseminor-次设备号起始地址，一般为 0；
     * count-申请的设备号数量
     */
    int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);

    // 注销字符设备后要释放掉设备号
    void unregister_chrdev_region(dev_t from, unsigned count);
    ```

## 字符设备两种注册方式
### file_operations 结构体

## 驱动开发 Coding 框架
1. 加载和卸载
   ```c
   module_init(xxx_init);   // 向内核注册一个模块加载函数，insmod 命令时调用
   module_exit(xxx_exit);   // 向内核注册一个模块卸载函数，rmmod 命令时调用
   ```
   其中，前者使用 `__init` 来修饰，后者使用 `__exit` 来修饰；
   补充：
      - `lsmod` 命令查看当前系统中存在的模块；
      - `insmod` 命令不能解决模块间的依赖关系；
      - `modprobe` 命令会分析模块的依赖关系，然后会将所有的依赖模块都加载到内核中。默认去 `/lib/modules/<kernel-version>` 目录下查找模块；

2. 字符设备注册和注销
   模块加载成功注册设备（`xxx_init`），模块卸载时注销设备（`xxx_exit`）：
   ```c
   static inline int register_chrdev(unsigned int major, const char *name, const struct file_operations *fops);
   static inline void unregister_chrdev(unsigned int major, const char *name);
   ```
   补充：
       - 关于主设备号的查看（也可用于设备情况）：
        ```bash
        cat /proc/devices
        ```

3. 实现设备的具体操作函数：`file_operations`、`copy_from_user`、`copy_to_user`、`printk`
4. 添加 LICENSE 和作者信息：
   ```c
   MODULE_LICENSE("GPL");   // linux 内核驱动一般使用 GPL 许可
   MODULE_AUTHOR("xxx");
   ```

## 新的字符设备驱动 API
- 旧方式（见“驱动开发 Coding 框架”）
  - 字符设备驱动开发的重点是使用 `register_chrdev` 函数注册字符设备，当不再使用时就使用 `unregister_chrdev` 来注销；
  - 驱动模块加载成功以后还需要手动使用 `mknod` 命令创建设备节点文件；
  - `register_chrdev` 和 `unregister_chrdev` 这两个函数是老版本驱动程序使用的函数；
  - 通过 `register_chrdev` 并以静态方式分配设备号（主设备号）带来的问题：
    - 需先确定哪个主设备号没有被使用；
    - 所有该主设备号下的次设备号都被占用，从而造成资源浪费；
- 新字符设备驱动（Linux 内核推荐）
  1. 创建设备号：
    都是向内核注册有一个设备号区间需要被当前驱动程序使用。
    - 已有主设备号（设备号）：`MKDEV(主设备号, 次设备号，一般为 0)`，从而得到设备号 `dev_t`，然后再通过 `register_chrdev_region(dev_t from, unsigned count, const char *name);` 函数完成注册；此时设备号作为传入参数；
    - 无主设备号（设备号）：`alloc_chrdev_region(...);` 进行设备号申请，设备号作为传出参数；
  2. 注销设备号
    无论是使用上述哪种方式得到的设备号，都需要在注销字符设备之后释放掉设备号：`void unregister_chrdev_region(dev_t from, unsigned count);`。相当于向内核说明这个设备号区间当前驱动已不再使用，从而释放被占用的设备号资源；
  3. 字符设备结构体（`cdev, /inlcude/linux/cdev.h`）：
    ```c
    struct cdev {
        struct kobject kobj;
        struct module *owner;
        const struct file_operations *ops;
        struct list_head list;
        dev_t dev;
        unsigned int count;
    };
    ```
    - 编写字符设备驱动之前需要定义一个 `cdev` 结构体变量，以表示一个字符设备：
        ```c
        struct cdev test_cdev;
        ```
    - 初始化 cdev：
        ```c
        void cdev_init(struct cdev *dev, const struct file_operations *fops);
        ```
        相当于建议字符设备结构体变量 `test_cdev` 与 `file_operations` 结构体变量的联系；
    - 向内核添加字符设备：
        ```c
        int cdev_add(struct cdev *p, dev_t dev, unsigned count);
        ```
        - [ ] 这里是建立 cdev 和 dev_t 之间的联系？
    - 卸载时从内核中删除相应的字符设备：
        ```c
        void cdev_del(struct cdev *p);
        ```
  4. 自动创建设备节点
    - `udev` 是一个**用户程序**，在 Linux 下通过 `udev` 实现设备文件的创建和删除。`udev` 可以检测系统中硬件设备状态，可以根据系统中硬件设备状态来创建和删除设备文件。比如使用 `modprobe` 命令卸载驱动模块以后就删除掉 `/dev` 目录下的设备节点文件。使用 `busybox` 构建根文件系统的时候，`busybox` 会创建一个 `udev` 的简化版本：`mdev`。所以在嵌入式 Linux 中使用 `mdev` 来实现设备节点文件的自动创建和删除，Linux 系统中的热插拔时间也由 `mdev` 管理，在 `/etc/init.d/rcs` 文件中：`echo /sbin/mdev > /proc/sys/kernel/hotplug`。
    - **自动创建设备节点的工作是在驱动程序入口函数中完成的**，一般在 `cdev_add` 函数后面添加自动创建设备节点相关代码；
    - 创建 `class` 结构体（`linux/include/device.h`）
        ```c
        class_create(owner, name)
        // 宏，对应的函数如下
        struct class* __class_create(...);
        // owner 一般为 THIS_MODULE，返回值为一个指向 class 结构体的指针
        ```
    - 创建设备
        ```c
        struct device* device_create(struct class *class, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...);
        ```
    - 卸载
        ```c
        void device_destory(struct class *class, dev_t devt);
        void class_destory(struct class *cls);
        ```

- [ ] proc 和 sys 文件系统。（通过 proc 可以查看设备的一些细心，通过 sys 可以对设备进行操作）；

## 基于字符设备新 API 的驱动框架
```c
#define NEWCHRLED_CNT 2
#define NEWCHRLED_NAME "led"

// 每个硬件设备都有一些属性，将其进行抽象，形成一个该设备的结构体类型
// 编写驱动 open 函数的时候将设备结构体作为 private_data 添加到设备文件中
struct newchrled_dev {
    dev_t devid;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    int major;
    int minor;
};

struct newchrled_dev newchrled;

void led_switch(u8 data) {
    ...
}

static int led_open(struct inode *inode, struct file *flip) {
    flip->private_data = &newchrled;
    return 0;
}

static ssize_t led_read(struct file *flip, char __user *buf, size_t cnt, loff_t *offt) {
    return 0;
}

static ssize_t led_write(struct file *flip, const char __user *buf, size_t cnt, loff_t *offt) {
    ...;
}

static int led_release(struct inode *inode, struct file *flip) {
    return 0;
}

static struct file_operations newchrled_fops = {
    .owner = THIS_MODULE,
    .open  = led_open,
    .read  = led_read,
    .write = led_write,
    .release = led_release
};

static int __init led_init(void) {
    u32 val = 0;
    // 初始化 LED
    // 1. 寄存器地址映射，略
    // 2. 使能 GPIO 时钟，略
    // 3. 设置 GPIO 复用功能，略
    // 4. 设置 GPIO 作为输出，并默认关闭 LED，略

    // 注册字符设备驱动
    // 1. 创建设备号
    if (newchrled.major) {
        newchrled.devid = MKDEV(newchrled.major, 0);
        register_chrdev_region(newchrled.devid, NEWCHRLED_CNT, NEWCHRLED_NAME);
    } else {
        alloc_chrdev_region(&newchrled.devid, 0, NEWCHRLED_CNT, NEWCHRLED_NAME);
        newchrled.major = MAJOR(newchrled.devid);
        newchrled.minor = MINOR(newchrled.devid);
    }
    // 2. 初始化 cdev
    newchrled.cdev.owner = THIS_MODULE;
    cdev_init(&newchrled.cdev, &newchrled_fops);
    // 3. 向内核注册 cdev
    cdev_add(&newcharled.cdev, newchrled.devid, NEWCHRLED_CNT);
    // 4. 创建类
    newchrled.class = class_create(THIS_MODULE, ...);
    if (IS_ERR(newchrled.class)) {
        return PTR_ERR(newchrled.class);
    }
    // 5. 创建设备
    newchrled.device = device_create(newchrled.class, NULL, newchrled.devid, NULL, NEWCHRLED_NAME);
    if (IS_ERR(newchrled.device)) {
        return PTR_ERR(newchrled.device);
    }
    
    return 0;
}

static void __exit led_exit(void) {
    // 取消内存映射，略
    // 注销字符设备
    cdev_del(&newchrled.cdev);
    unregister_chrdev_region(newchrled.devid, NEWCHRLED_CNT);
    device_destory(newchrled.class, newchrled.devid);
    class_destory(newchrled.class);
}

module_init(led_init);
module_exit(led_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("xxx");
```

## 注意事项
1. 通过 `copy_from_user` 和 `copy_to_user` 实现内核和用户空间数据的搬运；
2. 关于 `printk`:
    - 运行在内核态；
    - 8 个消息级别（`/include/linux/kern_levels.h`）：`KERN_EMERG-0（最高）, KERN_ALERT-1, KERN_CRIT-2, KERN_ERR-3, KERN_WARNING-4, KERN_NOTICE-5, KERN_INFO-6, KERN_DEBUG-7`，例如 `printk(KERN_EMERG "xxx");`。如果不显式设置消息级别，则采用默认级别：`MESSAGE_LOGLEVEL_DEFAULT-4`；
    - 在 `include/linux/printk.h` 中宏 `CONSOLE_LOGLEVEL_DEFAULT` 控制着哪些级别的消息可以显示在控制台上，它默认为 7；
3. `open, read, write` 等函数有一个参数 `struct file *flip`，它的 `private_data(void*)` 成员，可以在驱动中设置为指向设备结构体（自定义的，用于存放一些设备属性）；如果在 `open` 函数中设置了 `flip` 的 `private_data` 成员，那么在 `release` 函数（应用程序 `close` 时调用）中就要释放掉；
4. 驱动程序编译：
    ```makefile
    # linux 内核源码目录绝对路径
    KERNELDIR := ...
    CURRENT_PATH := $(shell pwd)
    obj-m := chrdevbase.o
    build : kernel_modules
    kernel_modules:
        $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
    clean:
        $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
    ```
    可以在 Linux 内核顶层 Makefile 文件中分析得到 `make modules` 和 `make clean` 两个目标。上述命令中，`-C` 表示将当前的工作目录切换到指定目录中；`-M` 用于指定模块源码目录，最终生成 `.ko` 文件；
5. 测试应用 APP 文件编译：交叉编译工具链编译、`file` 命令查看目标文件信息；
6. 加载时，如果不存在 `/lib/modules/<kernel-version>` 文件夹，则需要自行创建，否则影响 `modprobe` 命令使用。如果 `modprobe` 时出现 `modules.help: No Such file or directory` 的错误，则需先执行 `depmod` 命令。如果根文件系统中无 `depmod` 命令，则需重新构建根文件系统（`busybox, buildroot...`）；
7. 创建设备节点文件：
   目的是得到设备在用户空间中的实现。
   - 查看系统设备号信息，确认存在目标设备并得到设备号：`cat /proc/devices`；
   - 创建设备文件：`mknod /dev/设备名 c 主设备号 次设备号`，其中 `c` 表示字符设备；
8. 关于 MMU、ioremap、iounmap
   - Linux 下的任何外设驱动，最终都是要配置相应的硬件寄存器；
   - 在老版本的 Linux 中要求处理器必须要有 MMU，但是现在 Linux 内核已经支持无 MMU 的处理器了，MMU 主要完成的功能如下：
     - 完成虚拟空间到物理空间的映射；
     - 内存保护，设置存储器的访问权限，设置虚拟存储空间的缓冲特性；
   - Linux 内核启动的时候会初始化 MMU，设置好内存映射，设置好以后 CPU 访问的都是虚拟地址。需要通过 `ioremap` 和 `iounremap` 实现物理内存到虚拟内存之间的转换。`ioremap` 为宏定义，实现实现为 `__arm_ioremap(...)`，位于 `arch/arm/include/asm/io.h` 中。通过 `ioremap` 实现物理地址到虚拟地址的转换，`iounremap` 释放这层映射关系；
9. 内存访问函数
    Linux 建议使用这组函数来操作通过 `ioremap` 映射得到的虚拟地址：
    ```c
    // 读
    u8 readb(const volatile void __iomem *addr);
    u16 readw(...);
    u32 readl(...);
    // 写
    void writeb(u8 value, volatile void __iomem *addr);
    void writew(u16 value, ...);
    void writel(u32 value, ...);
    ``` 