# 字符设备驱动
- 在 Linux 中一切皆文件，驱动加载成功以后会在 `/dev` 目录下生成一个相应的文件；（前提：这个文件的生成需要在驱动程序中调用相应函数，详情见后续“关于 file_operations 的两种注册方式”）

- 应用程序通过对这个名为 `/dev/xxx` 的文件进行操作即可实现对硬件的相应操作；

## 驱动开发前瞻
### 用户空间程序调用流程
流程：app -> 标准 IO（fopen/fread/fwrite/fclose 等）-> 系统调用 IO（open/read/write/close 等） -> svc 异常处理 -> 驱动程序 -> 硬件
- [ ] 添加函数调用关系图

### 文件句柄和 file 结构体的关系
涉及的数据结构：file、indoe...
- [ ] 添加相应的关系图

### file_operations 结构体
```c
// include/linux/fs.h
struct file_operations {
    ...
}
```

### 设备号
- [ ] 设备号是 Linux 设备的唯一标识吗？

- 类型定义
    ```c
    // include/linux/types.h
    typedef __u32 __kernel_dev_t;
    ...
    typedef __kernel_dev_t dev_t;

    // include/uapi/asm-generic/int-ll64.h
    typedef unsigned int __u32;
    ```
    - `dev_t` 是一个 32 位的数据类型；
- 组成
  - 主设备号：高 12 位，范围为 0~4095。**可用于表示某一个具体的驱动**。
  - 次设备号：低 20 位。**可用于表示主设备号对应驱动的各个设备**。
- 相关宏定义：
    ```c
    // include/linux/kdev_t.h
    #define MINORBITS   20
    #define MINORMASK   ((1u << MINORBITS) - 1)
    #define MAJOR(dev)  ((unsigned int)((dev) >> MINORBITS))
    #define MINOR(dev)  ((unsigned int)((dev) & MINORMASK))
    #define MKDEV(ma, mi)   (((ma) << MINORBITS) | (mi))
    ```
- （主）设备号的分配
  - 静态分配：易造成冲突。驱动开发静态指定，参考 `/Documentation/device.txt` 查看已被分配的常用设备号，实际使用情况还是通过 `cat /proc/devices` 命令查看；
  - 动态分配：在注册设备之前先申请一个设备号，系统会自动给出一个未被使用的设备号；设备号申请与注销函数：
    ```c
    /* 
     * 申请 count 个设备号，它们的主设备号相同，次设备号从 baseminor 开始。
     * baseminor-次设备号起始地址，一般为 0；
     * count-申请的设备号数量
     */
    int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);

    // 注销字符设备后要释放掉设备号
    /* 注销 count 个设备号，从 from 开始 */
    void unregister_chrdev_region(dev_t from, unsigned count);
    ```
- 查看当前已经被使用掉的设备号
    ```bash
    cat /proc/devices
    ```

### 简单字符设备驱动开发内容
1. 驱动模块的加载和卸载
   - 驱动编程：
     - 模块注册：`module_init(xxx_init)` -> `static int __init xxx_init(void) {...}`，向内核注册一个模块加载函数，`insmod` 命令时调用；
     - 模块卸载：`module_exit(xxx_exit)` -> `static void __exit xxx_exit(void) {...}`，向内核注册一个模块卸载函数，`rmmod` 命令时调用。
   - 驱动模块：
     - 安装命令：`insmod` 或 `modprobe`，调用 `xxx_init()`；
     - 卸载命令：`rmmod` 或 `modprobe -r`，调用 `xxx_exit()`；
     - 查看当前系统中存在的模块：`lsmod`；
     - 注：`modprobe` 命令会自动处理依赖问题，默认依赖查找路径：`/lib/modules/<kernel-version>`。
2. 字符设备注册与注销
    ```c
    static inline register_chrdev(unsigned int major, const char *name, const struct file_operations *fops);
    static inline void unregister_chrdev(unsigned int major, const char *name);
    ```
    - 通常，在 `xxx_init` 函数中完成字符设备的注册，在 `xxx_exit` 中注销。
3. 实现设备的具体操作函数：打开、关闭、读、写 等等：
   - 用户空间和内核空间的数据交换：
        ```c
        // arch/arm/include/asm/uaccess.h
        /* 复制成功返回 0，失败则返回负数 */
        
        static inline unsigned long __must_check copy_to_user(void __user *to, const void *from, unsigned long n) {
            if (access_ok(VERIFY_WRITE, to, n))
		        n = __copy_to_user(to, from, n);
	        return n;
        }

        static inline unsigned long __must_check copy_from_user(void *to, const void __user *from, unsigned long n) {
            if (access_ok(VERIFY_READ, from, n))
                n = __copy_from_user(to, from, n);
            else /* security hole - plug it */
                memset(to, 0, n);
            return n;
        }
        ```
   - 物理地址到虚拟地址的映射：
        ```c
        // /arch/arm/include/asm/io.h
        #define ioremap(cookie,size)		__arm_ioremap((cookie), (size), MT_DEVICE)
        void __iomem *
        __arm_ioremap(phys_addr_t phys_addr, size_t size, unsigned int mtype)
        {
            return arch_ioremap_caller(phys_addr, size, mtype,
                __builtin_return_address(0));
        }

        #define iounmap				__arm_iounmap
        void __arm_iounmap(volatile void __iomem *io_addr)
        {
            arch_iounmap(io_addr);
        }
        ```
4. 添加 License 和作者信息
    ```c
    MODULE_LICENSE("GPL");  // linux 内核驱动一般使用 GPL 许可
    MODULE_AUTHOR("xxx");
    ```

### 驱动运行的两种方式
- 编译到内核
  - 内核启动时自动运行驱动程序；
- 以模块方式加载
  - 文件扩展名为 `.ko`，在 Linux 运行时可通过 `insmod` 命令动态加载驱动模块；

驱动开发时一般以模块方式进行，完成后可将其编译到内核中。

### 简单字符设备驱动实现框架
```c
#include ...

#define CHRDEVBASE_MAJOR	200
#define CHRDEVBASE_NANE		"xxx"

// 驱动数据
static char readbuf[100];
// ...

static int chrdevbase_open(struct inode *inode, struct file *file) { }

static ssize_t chrdevbase_read(struct file *flip, char __user *buf, size_t cnt, loff_t *offt) { }

static ssize_t chrdevbase_write(struct file *flip, const char __user *buf, size_t cnt, loff_t *offt) { }

static int chrdevbase_release(struct inode *inode, struct file *flip) { }

static struct file_operations chrdevbase_fops = {
	.owner = THIS_MODULE,
	.open  = chrdevbase_open,
	.read  = chrdevbase_read,
	.write = chrdevbase_write,
	.release = chrdevbase_release,
};

static int __init chrdevbase_init(void) {
	int retval = 0;
	
	retval = register_chrdev(CHRDEVBASE_MAJOR, CHRDEVBASE_NANE, &chrdevbase_fops);
	
    // ...
}

static void __exit chrdevbase_exit(void) {
	unregister_chrdev(CHRDEVBASE_MAJOR, CHRDEVBASE_NANE);
	// ...
}

module_init(chrdevbase_init);
module_exit(chrdevbase_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("xxx");
```
- 需手动创建设备节点文件：
    ```bash
    mknod /dev/xxx c MAJOR MINOR
    ```

## 字符设备两种注册方式
### 字符设备的注册流程
1. 准备设备号
2. 注册字符设备
3. 创建设备节点
4. 注销字符设备

### 旧版本
1. 自行准备设备号：
   - 事先确定好没有被使用的主设备号;
2. 向内核注册字符设备：
   ```c
   static inline register_chrdev(unsigned int major, const char *name, const struct file_operations *fops);
   ```
3. 创建设备节点：
   - 通过 `mknod` 自行创建设备节点：
        ```bash
        mknod /dev/xxx c MAJOR MINOR
        ```
   - 自动创建设备节点：同下
4. 从内核注销字符设备：
   ```c
   static inline void unregister_chrdev(unsigned int major, const char *name);
   ```

示例程序见“简单字符设备驱动实现框架”。

### 新版本
1. 向内核申请（注册）设备号
    ```c
    // fs/char_dev.c
    // 没有设备号，这里 dev_t 为传出参数
    // return: Return zero or a negative error code.
    int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
    
    // fs/char_dev.c
    // 已有设备号，这里 dev_t 为传入参数
    // 通过宏 `MKDEV(ma, mi)` 来得到设备号；
    // return: Return value is zero on success, a negative error code on failure.
    int register_chrdev_region(dev_t from, unsigned count, const char *name);
    ```
2. 向内核注册字符设备：
   1. 定义 `cdev` 结构体：
        ```c
        // include/linux/cdev.h
        struct cdev {
            struct kobject kobj;
            struct module *owner;
            const struct file_operations *ops;
            struct list_head list;
            dev_t dev;
            unsigned int count;
        }
        ```
   2. 初始化 `cdev`：建立字符设备和操作函数集的关联
        ```c
        void cdev_init(struct cdev *cdec, const struct file_operations *fops);
        ```
   3. 向内核添加 `cdev`：建立字符设备和设备号的关联等
        ```c
        // fs/char_dev.c
        // return: A negative error code is returned on failure. 0 on success.
        int cdev_add(struct cdev *p, dev_t dev, unsigned count);
        ```
3. 创建设备节点：
   1. 创建类：
        ```c
        // include/linux/device.h
        #define class_create(owner, name) \
        ({                                \
            static struct lock_class_key __key; \
            __class_create(owner, name &__key); \
        })

        struct class *__class_create(struct module *owner, const char *name, struct lock_class_key *key);
        ```
   2. 创建设备：
        ```c
        struct device *device_create(struct class *class, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...);
        ```
        - `parent`：父设备，一般为 `NULL`；
        - `drvdata`：设备可能使用的数据，一般为 `NULL`；
4. 从内核注销字符设备：
   1. 从内核删除 `cdev`：
        ```c
        void cdev_del(struct cdev *p);
        ```
   2. 注销字符设备所使用设备号：
        ```c
        void unregister_chrdev_region(dev_t from, unsigned count);
        ```
   3. 删除设备和类：
        ```c
        void device_destroy(struct class *class, dev_t devt);
        void class_destroy(struct class *cls);
        ```

示例程序：
```c
#define DEVICE_COUNT 1

struct xxx_dev {

}

static xxx_dev xxxdev;

static int xxx_open(struct inode *inode, struct file *filp) {
    filp->private_data = &xxxdev;
}

static ssize_t xxx_read(struct file *filp, char __user *buf, size_t cnt, loff_t *offt) {...}
static ssize_t xxx_write(struct file *filp, const char __user *buf, size_t cnt, loff_t *offt) {...}
static int xxx_release(struct inode *inode, struct file *filp) {...}

static struct file_operations xxx_fops = {.onwer = THIS_MODULE, ...};

static int __init xxx_init(void) {
    int ret;
    ... // 设备相关准备工作

    // 注册字符设备驱动
    // 1. 设备号
    if (xxxdev.major) {
        xxxdev.devid = MKDEV(xxxdev.major, 0);
        ret = register_chrdev_region(xxxdev.devid, DEVICE_COUNT, "xxx");
    } else {
        ret = alloc_chrdev_region(&xxxdev.devid, 0, DEVICE_COUNT, "xxx");
        xxxdev.major = MAJOR(devid);
        xxxdev.minor = MINOR(devid);
    }
    if (ret < 0) {
        // ... 错误处理
        return ret;
    }
    // 2. 注册
    xxxdev.cdev.onwer = THIS_MODULE;
    cdev_init(&xxxdev.cdev, &xxx_fops);
    ret = cdev_add(&xxxdev.cdev, xxxdev.devid, DEVICE_COUNT);
    if (ret < 0) {
        // ... 错误处理
    }
    // 3. 创建设备节点
    xxxdev.class = class_create(THIS_MODULE, "xxx");
    if (IS_ERR(xxxdev.class)) {
        // ... 错误处理
    }
    xxxdev.device = device_create(xxxdev.class, NULL, xxxdev.devid, NULL, "xxx");
    if (IS_ERR(xxxdev.device)) {
        // ... 错误处理
    }

    return 0;
}

static int __exit xxx_exit(void) {
    ... // 设备相关清理工作

    // 注销字符设备
    // 1. 删除 cdev
    cdev_dev(&xxxdev.cdev);
    // 2. 注销设备号
    unregister_chrdev_region(xxxdev.devid, DEVICE_COUNT);
    // 3. 删除设备和类
    device_destroy(xxxdev.class, xxxdev.devid);
    class_destroy(xxxdev.class);
}

module_init(xxx_init);
module_exit(xxx_exit);
MODULE_LICENSE("GPL");
```

### 两版本的对比
- 通过 `register_chrdev` 并以静态方式分配设备号（主设备号）带来的问题：
    - 需先确定哪个主设备号没有被使用；
    - 所有该主设备号下的次设备号都被占用，从而造成资源浪费；
- 若没有自动创建设备节点文件，还需在驱动模块加载成功以后，手动使用 `mknod` 命令来创建；
- 旧版本使用的 `register_chrdev` 和 `unregister_chrdev` 这两个函数是老版本驱动程序使用的函数；Linux 内核推荐使用新字符设备驱动 API 来注册；

## 补充内容：
### mdev
更详细地内容可见笔记：`sysfs.md`。

`udev` 是一个**用户程序**，在 Linux 下通过 `udev` 来实现设备文件地创建和删除，`udev` 可以检测系统中硬件设备状态，可以根据系统中硬件设备状态来创建或者删除设备文件。

例如，使用 `modprobe` 命令成功加载驱动模块以后就自动在 `/dev` 目录下创建对应地设备节点文件；使用 `rmmod` 命令卸载驱动模块以后就删除掉 `/dev` 目录下的设备节点文件。

使用 busybox 构建根文件系统时，它会创建一个 `udev` 的简化版本：`mdev`，在嵌入式 Linux 中使用 `mdev` 来实现设备节点文件的自动创建和删除。

Linux 系统中的热插拔事件也由 `mdev` 管理，在 `/etc/init.d/rcS` 文件中通过如下语句来设置热插拔事件由 `mdev` 来管理。
```shell
echo /sbin/mdev > /proc/sys/kernel/hotplug
```