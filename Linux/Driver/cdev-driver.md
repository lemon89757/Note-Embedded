# 字符设备驱动
## 概述
基于总线设备驱动模型以及 *platform* 架构，可以方便高效地通过驱动建立设备与内核间的关联。但是，通常存在用户空间操控设备的需求：
- 由于 *Linux* 万物皆文件的思想，因此在用户空间是以访问文件的方式操作底层设备的，所以还需建立起驱动与文件系统间的关联，这反映在对 `struct file_operations` 的实现上；
- 能够根据文件的相关内容查找到对应设备的一种机制，这反应在 `struct inode` 的相关字段上（对于该数据结构的相关信息，见虚拟文件系统的相关笔记），主要是设备类型（字符设备和块设备）以及设备号信息

> 《深入 LInux 内核架构》
> 根据外设与系统之间交换数据的方法，可以将设备分为几种类别。有些设备非常适合于面向字符的数据交换，因为数据传输量很低；其他的设备则更适合于处理包含固定数目字节的数据块。内核会区分字符设备和块设备。前一类包括串行接口和文本终端，而后一类则包括硬盘、光驱等设备。

字符设备具有的特点：
- 数据传输的单位为字符
- 面向流的传输，不支持随机访问

Linux 内核通过 `struct cdev` 表示字符设备
- 可以将其视为对设备和驱动的一种管理机制。对于驱动方面，它提供 *file_operations*，允许用户空间以文件的形式操作设备；对于设备方面，它允许将若干个设备视为一个整体，并提供注册机制、实现根据设备号查找 *cdev*，以实现对设备的操作。
- `struct cdev` 提供设备（`struct device`）以设备号（以在用户空间创建 `/dev` 下设备的入口目录），并按照 *VFS* 要求的 `struct file_operations` 操作集为若干个（设备号约束）设备提供回调函数实现，从而允许用户空间使用统一的文件 *API* 来操作设备。

字符设备文件在用户空间的入口：
- 在 Linux 中一切皆文件，驱动加载成功以后会在 `/dev` 目录下生成一个相应的文件；（前提：这个文件的生成需要在驱动程序中调用相应函数，详情见后续“关于 *file_operations* 的两种注册方式”）
- 应用程序通过对这个名为 `/dev/xxx` 的文件进行操作即可实现对硬件的相应操作；

## 使用
### 用户空间程序调用流程
1. 用户空间应用程序
2. 标准库 *I/O*（`fopen()/fread()/fwrite()/fclose()` 等）（非必须）
3. 系统调用 *I/O*（`open()/read()/write()/close()` 等）
4. *svc* 异常处理（陷入指令，从用户态转为内核态）
5. 内核转接（`open()` 时）
6. 驱动程序（*file_operations* 其它回调函数）
7. 硬件

- [ ] 添加函数调用关系图

### 文件句柄和 file 结构体的关系
> 《深入 Linux 内核架构》-第六章 设备驱动程序 + 第八章 虚拟文件系统
这里需要明确一个概念是：`struct file` 是进程空间方面的信息，而 `struct inode` 是对文件系统方面的信息。

涉及的数据结构：`struct task_struct`、`struct files_struct`、`struct inode` 和 `struct file`...

简单来说，文件句柄是进程打开文件列表的元素对应索引值

- [ ] 添加相应的关系图

### file_operations 结构体
```c
// version: 4.1.15
// include/linux/fs.h
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iterate) (struct file *, struct dir_context *);
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*mremap)(struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*aio_fsync) (struct kiocb *, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
};
```
- 各回调函数的功能：
  - 略
- 驱动程序无需对全部回调进行实现

#### file_operations 回调函数实现
记录 *file_operations* 回调函数实现的一般方式

### 设备号
- 类型定义
    ```c
    // version: 4.1.15
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

- 设备号是 *Linux* 设备的唯一标识吗？
  - 在同一类型的设备中，如在字符设备，设备号是唯一的
  - 在不同类型的设备间，如字符和块设备之间，允许存在相同的设备号

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

### 字符设备两种注册方式
#### 字符设备的注册流程
1. 准备设备号
2. 注册字符设备
3. 创建设备节点
4. 注销字符设备

#### 旧版本
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

#### 新版本
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
        // version: 4.1.15
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

#### 两版本的对比
- 通过 `register_chrdev` 并以静态方式分配设备号（主设备号）带来的问题：
    - 需先确定哪个主设备号没有被使用；
    - 所有该主设备号下的次设备号都被占用，从而造成资源浪费；
- 若没有自动创建设备节点文件，还需在驱动模块加载成功以后，手动使用 `mknod` 命令来创建；
- 旧版本使用的 `register_chrdev` 和 `unregister_chrdev` 这两个函数是老版本驱动程序使用的函数；Linux 内核推荐使用新字符设备驱动 API 来注册；

## cdev 相关源码阅读
### 数据结构描述
```c
// version: 4.1.15
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
- `count` 字段，*cdev* 所管理的设备数量，一般与此设备号的数量一致。
- `list` 字段，用于链接与该 *cdev* 相关的 *inode*。由于一个 *cdev* 可能被多个设备（节点文件）使用，一个驱动多个设备，因此当用户空间程序 `open()` 设备节点文件时，与之对应的 *inode* 通过该字段链接到 *cdev* 的链表上（详情见 `chrdev_open()` 函数解析）。

### 字符设备管理
示意图、哈希表、调用链

总体图示可见：`Linux/Driver/image.drawio-cdev overview`

两个变量，`cdev_map` 和 `chrdevs`。
1. 哈希表表示
   - 通过 `struct kobj_map` 表示，在 `fs/char_dev.c` 中定义了一个该类型指针的全局变量：`cdev_map`。由下述定义可知，一个 *kobj_map* 表示的哈希表拥有 255 个 *bucket*，具有数组 + 链表的“拉链”形式，元素的数据类型为 `struct probe`。
    ```c
    // version: 4.1.15
    // drivers/base/map.c
    struct kobj_map {
        struct probe {
            struct probe *next;
            dev_t dev;
            unsigned long range;
            struct module *owner;
            kobj_probe_t *get;
            int (*lock)(dev, void *);
            void *data;
        } *probe[255];
        struct mutex *lock;
    };

    // include/linux/kobj_map.h
    typedef struct kobject *kobj_probe_t(dev_t, int *, void *);
    ```
   - 该变量用于保存设备号、设备数量等信息，便于后续根据 *cdev* 的相应属性快速查找对应的 *cdev* 实例。
   - 该变量在 `chrdev_init()` 中被初始化：
    ```c
    // version: 4.1.15
    // fs/char_dev.c
    void __init chrdev_init(void) {
        cdev_map = kobj_map_init(base_probe, &chrdevs_lock);
    }

    // drviers/base/map.c
    struct kobj_map *kobj_map_init(kobj_probe_t *base_probe, struct mutex *lock) {
        struct kobj_map *p = kmalloc(sizeof(struct kobj_map), GFP_KERNEL);
        struct probe *base = kzalloc(sizeof(*base), GFP_KERNEL);
        int i;
        if ((p == NULL) || (base == NULL)) {
            kfree(p);
            kfree(base);
            return NULL;
        }
        base->dev = 1;
        base->range = ~0;
        base->get = base_probe;
        for (i = 0; i < 255; i++)
            p->probes[i] = base;
        p->lock = lock;
        return p;
    }
    ```
   - 调用链：
    ```c
    start_kernel()
        vfs_caches_init()
            chrdev_init()
    ```

2. 全局变量 `chrdevs`：
    ```c
    // version: 4.1.15
    // fs/char_dev.c
    static struct char_device_struct {
        struct char_device_struct *next;
        unsigned int major;
        unsigned int baseminor;
        int minorct;
        char name[64];
        struct cdev *cdev;  /* will die */
    } *chrdevs[CHRDEV_MAJOR_HASH_SIZE];

    // include/linux/fs.h
    #define CHRDEV_MAJOR_HASH_SIZE 255
    ```
    - 也可以将 `chrdevs` 视为一个哈希表，*bucket* 数量同样是 255，元素类型是 `struct char_device_struct *`。
    - `chrdevs` 用于字符设备注册之前的申请设备号阶段、注册时的设备号有效性验证。

### 驱动接口解析
1. `cdev_init()`：初始化 *cdev*
    ```c
    // version: 4.1.15
    // fs/char_dev.c
    /** 
     * cdev_init() - initialize a cdev structure 
     * @cdev: the structure to initialize
     * @fops: the file_operations for this device
     *
     * Initialize @cdev, remembering @fops, making it ready to
     * add to the system with cdev_add().
     */
    void cdev_init(struct cdev *cdev, const struct file_operations *fops) {
        memset(cdev, 0, sizeof *cdev);
        INIT_LIST_HEAD(&cdev->list);
        kobject_init(&cdev->kobj, &ktype_cdev_default);
        cdev->ops = fops;
    }

    static struct kobj_type ktype_cdev_default = {
        .release = cdev_default_release,
    };

    static void cdev_default_release(struct kobject *kobj) {
        struct cdev *p = container_of(kobj, struct cdev, kobj);
        struct kobject *parent = kobj->parent;
        cdev_purge(p);
        kobject_put(parent);
    }

    static void cdev_purge(struct cdev *cdev) {
        spin_lock(&cdev_lock);
        while (!list_empty(&cdev->list)) {
            struct inode *inode;
            inode = container_of(cdev->list.next, struct inode, i_devices);
            list_dev_init(&inode->i_devices);
            inode->i_cdev = NULL;
        }
        spin_unlock(&cdev_lock);
    }
    ```

2. `cdev_alloc()`：申请内存 + 初始化 *cdev*
    ```c
    // version: 4.1.15
    // fs/char_dev.c
    /**
     * cdev_alloc() - allocate a cdev structure 
     * Allocates and returns a cdev structure, or NULL on failure.
     */
    struct cdev *cdev_alloc(void) {
        struct cdev *p = kzalloc(sizeof(struct cdev), GFP_KERNEL);
        if (p) {
            INIT_LIST_HEAD(&p->list);
            kobject_init(&p->kobj, &ktype_cdev_dynamic);
        }
        return p;
    }

    static struct kobj_type ktype_cdev_dynamic = {
        .release = cdev_dynamic_release,
    }

    static void cdev_dynamic_release(struct kobject *kobj) {
        struct cdev *p = container_of(kobj, struct cdev, kobj);
        struct kobject *parent = kobj->parent;
        cdev_purge(p);
        kfree(p);
        kobject_put(parent);
    }
    ```

3. `cdev_add()`：注册 *cdev*
    ```c
    // version: 4.1.15
    // fs/char_dev.c
    /** 
     * cdev_add() - add a char device to the system 
     * @p: the cdev structure for the device
     * @dev: the first device number for which this device is responsible
     * @count: the number of consecutive minor numbers corresponding
     *   to this device
     *
     * cdev_add() adds the device represented by @p to the system,
     * making it live immediately. A negative error code is returned on failure.
     */
    int cdev_add(struct cdev *p, dev_t dev, unsigned count) {
        int error;
        p->dev = dev;
        p->count = count;
        error = kobj_map(cdev_map, dev, count, NULL, exact_match, exact_lock, p);
        if (error)
            return error;
        kobject_get(p->kobj.parent);
        return 0;
    }

    static struct kobject *exact_match(dev_t dev, int *part, void *data) {
        struct cdev *p = data;
        return &p->kobj;
    }

    static int exact_lock(dev_t dev, void *data) {
        struct cdev *p = data;
        retur cdev_get(p) ? 0 : -1;
    }

    // drivers/base/map.c
    int kobj_map(struct kobj_map *domain, dev_t dev, unsigned long range,
        struct module *module, kobj_probe_t *probe, 
        int (*lock)(dev_t, void *), void *data) {
        unsigned n = MAJOR(dev + range - 1) - MAJOR(dev) + 1;
        unsigned index = MAJOR(dev);
        unsigned i;
        struct probe *p;
        if (n > 255)
            n = 255;
        p = kmalloc_array(n, sizeof(struct probe), GFP_KERNEL);
        if (p == NULL)
            return -ENOMEM;
        for (i = 0; i < ; i++, p++) {
            p->owner = module;
            p->get = probe;
            p->lock = lock;
            p->dev = dev;
            p->range = range;
            p->data = data;
        }
        mutex_lock(domain->lock);
        for (i = 0, p -= n; i < n; i++, p++, index++) {
            struct probe **s = &domain->probes[index % 255];
            while (*s && (*s)->range < range)
                s = &(*s)->next;
            p->next = *s;
            *s = p;
        }
        mutex_unlock(domain->lock);
        return 0;
    }
    ```
    - `cdev_add()` 将 *cdev* 的管理信息（设备号、设备数量）插入到哈希表中；
    - `cdev_add()` 并没有将 `cdev->kobj` 注册到系统中（无法在用户空间 `/sys` 目录下找到它），因此 *cdev* 只使用到了 *kobject* 的生命周期管理机制。
    - 一个 *cdev* 管理的设备可能需要多个 `struct probe` 来表示（如果跨主设备号的话），`kobj_map()` 将这些 *probe* 放入哈希表中。哈希表中每个 *bucket* 中的元素，按照 `probe->range` 升序排列。

4. `cdev_del()`
    ```c
    // version: 4.1.5
    // fs/char_dev.c
    /**
     * cdev_del() - remove a cdev from the system 
     * @p: the cdev structure to be removed
     *
     * cdev_del() removes @p from the system, possibly freeing the
     * structure itself.
     */
    void cdev_dev(struct cdev *p) {
        cdev_unmap(&p->dev, p->count);
        kobject_put(&p->kobj);
    }

    static void cdev_unmap(dev_t dev, unsigned count) {
        kobj_unmap(cdev_map, cdev, count);
    }

    // drivers/base/map.c
    void kobj_unmap(struct kobj_map *domain, dev_t dev, unsigned long range) {
        unsigned n = MAJOR(dev + range - 1) - MAJOR(dev) + 1;
        unsigned index = MAJOR(dev);
        unsigned i;
        struct probe *found = NULL;
        if (n > 255)
            n = 255;
        mutex_lock(domain->lock);
        for (i = 0; i < n; i++, index++) {
            struct probe **s;
            for (s = &domain->probes[index % 255]; *s; s = &(*s)->next) {
                struct probe *p = *s;
                if (p->dev == dev && p->range = range) {
                    *s = p->next;
                    if (!found)
                        found = p;
                    break;
                }
            }
        }
        mutex_unlock(domain->lock);
        kfree(found);
    }
    ```
    - 通过 `probe->dev` 和 `probe->range` 两个字段来检索符合要求的 *probe*。
    - 通过 `kmalloc_array()` 申请的内存也是通过 `kfree()` 来释放。
    - `kobj_map()` 和 `kobj_unmap()` 均通过二级指针来遍历、管理链表元素。

5. `alloc_chrdev_region()`
    ```c
    // version: 4.1.15
    // fs/char_dev.c
    /**
     * alloc_chrdev_region() - register a range of char device numbers
     * @dev: output parameter for first assigned number
     * @baseminor: first of the requested range of minor numbers
     * @count: the number of minor numbers required
     * @name: the name of the associated device or driver
     * 
     * Allocates a range of char device numbers. The major number will
     * chosen dynamically, and returned (along with first minor number)
     * in @dev. Returns zero or a negative error code.
     */
    int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count,
        const char *name) {
        struct char_device_struct *cd;
        cd = __register_chrdev_region(0, baseminor, count, name);
        if (IS_ERR(cd))
            return PTR_ERR(cd);
        *dev = MKDEV(cd->major, cd->baseminor);
        return 0;
    }

    /* 
     * Register a single major with a specified minor range.
     * If major == 0 this functions will dynamically allocate a major
     *   and return its number.
     * If major > 0 this function will attempt to reserve the passed
     *   range of minors and will return zero on success.
     *
     * Returns a -ve errno on failure.
     */
    static struct char_device_struct *__register_chrdev_region(unsigned int major,
        unsigned int baseminor, int minorct, const char *name) {
        struct char_device_struct *cd, **cp;
        int ret;
        int i;
        cd = kzalloc(sizeof(struct char_device_struct), GFP_KERNEL);
        if (cd == NULL)
            return ERR_PTR(-ENOMEM);
        mutex_lock(&chrdevs_lock);
        /* temporary */
        if (major == 0) {
            for (i = ARRAY_SIZE(chrdevs) - 1; i > 0; i--) {
                if (chrdevs[i] == NULL)
                    break;
            }
            if (i == 0) {
                ret = -EBUSY;
                goto out;
            }
            major = i;
        }
        cd->major = major;
        cd->baseminor = baseminor;
        cd->minorct = minorct;
        strcpy(cd->name, name, sizeof(cd->name));
        i = major_to_index(major);
        for (cp = &chrdev[i]; *cp; cp = &(*cp)->next)
            if ((*cp)->major > major || ((*cp)->major == major &&
                (((*cp)->baseminor >= baseminor) || ((*cp)->baseminor + (*cp)->minorct > baseminor))))
                break;
        /* check for overlapping minor ranges */
        if (*cp && (*cp)->major == major) {
            int old_min = (*cp)->baseminor;
            int old_max = (*cp)->baseminor + (*cp)->minorct - 1;
            int new_min = baseminor;
            int new_max = baseminor + minorct - 1;
            /* New driver overlaps from the left */
            if (new_max >= old_min && new_max <= old_max) {
                ret = -EBUSY;
                goto out;
            }
            /* New driver overlaps from the right */
            if (new_min <= old_max && new_min >= old_min) {
                ret = -EBUSY;
                goto out;
            }
        }
        cd->next = *cp;
        *cp = cd;
        mutex_unlock(&chrdevs_lock);
        return cd;
    out:
        mutex_unlock(&chrdevs_lock);
        kfree(cd);
        return ERR_PTR(ret);
    }

    static inline int major_to_index(unsigned major) {
        return major % CHRDEV_MAJOR_HASH_SIZE;
    }
    ```
    - `__register_chrdev_region()` 给定的主设备号 `major` 参数若为 0，则会反向遍历 `chrdevs` 数组，得到第一个为空的位置，其对应的索引值即为动态分配的主设备号。因此，动态分配的主设备号大小小于 `chrdevs` 数组的长度。如果 `chrdevs` 数组全部不为空，则动态分配主设备号失败。
    - 根据主设备号对应的索引值（对数组长度取余），对 `chrdevs` 中相应的 *bucket* (`struct chr_device_struct` 数组)进行遍历，*bucket* 中的元素根据主设备号、`baseminor`、`baseminor + minorct` 升序排列，且要求新增的设备号范围不能覆盖到已存在的内容。

6. `register_chrdev_region()` 和 `unregister_chrdev_region()`
    ```c
    // version: 4.1.15
    // fs/char_dev.c

    /**
     * register_chrdev_region() -  register a range of device numbers
     * @from: the first in the desired range of device numbers; must
     *   include the major number
     * @count: the number of consecutive device numbers required
     * @name: the name of the device or driver.
     *
     * Return value is zero on success, a negative error code on failure.
     */
    int register_chrdev_region(dev_t from, unsigned count, const char *name) {
        struct char_device_struct *cd;
        dev_t to = from + count;
        dev_t n, next;
        for (n = from; n < to; n = next) {
            next = MKDEV(MAJOR(n) + 1, 0);
            if (next > to)
                next = to;
            cd = __register_chrdev_region(MAJOR(n), MINOR(n), next - n, name);
            if (IS_ERR(cd))
                goto fail;
        }
        return 0;
    fail:
        to = n;
        for (n = froml; n < to; n = next) {
            next = MKDEV(MAJOR(n) + 1, 0);
            kfree(__unregister_chrdev_region(MAJOR(n), MINOR(n), next - n));
        }
        return PTR_ERR(cd);
    }

    static struct char_device_struct *__unregister_chrdev_region(unsigned major,
        unsigned baseminor, int minorct) {
        struct char_device_struct *cd = NULL, **cp;
        int i = major_to_index(major);
        mutex_lock(&chrdevs_lock);
        for (cp = &chrdevs[i]; *cp; cp = &(*cp)->next)
            if ((*cp)->major = major &&
                (*cp)->baseminor = baseminor &&
                (*cp)->minorct = minorct)
                break;
        if (*cp) {
            cd = *cp;
            *cp = cd->next;
        }
        mutex_unlock(&chrdevs_lock);
        return cd;
    }

    /** 
     * unregister_chrdev_region() - return a range of device numbers 
     * @from: the first in the range of numbers to unregister
     * @count: the number of device numbers to unregister
     *
     * This function will unregister a range of @count device numbers,
     * starting with @from. The caller should normally be the one
     * who allocated those numbers in the first place...
     */
    void unregister_chrdev_region(dev_t from, unsigned count) {
        dev_t to = from + count;
        dev_t n, next;
        for (n = from; n < to; n = next) {
            next = MKDEV(MAJOR(n) + 1, 0);
            if (next > to)
                to = next;
            kfree(__unregister_chrdev_region(MAJOR(n), MINOR(n), next - n));
        }
    }
    ```

### 字符设备文件操作调用
用户空间操作文件：
```c
// version: 4.1.15
// fs/char_dev.c

/**
 * called every time a character special file is opened 
 */
static int chrdev_open(struct inode *inode, struct file *filp) {
    const struct file_operations *fops;
    struct cdev *p;
    struct cdev *new = NULL;
    int ret = 0;
    spin_lock(&cdev_lock);
    p = inode->i_cdev;
    if (!p) {
        struct kobject *kobj;
        int idx;
        spin_unlock(&cdev_lock);
        kobj = kobj_lookup(cdev_map, inode->i_rdev, &idx);
        if (!kobj)
            return -ENXIO;
        new = container_of(kobj, struct cdev, kobj);
        spin_lock(&cdev_lock);
        /* Check i_cdev again in case somebody beat us to it while 
         * we dropped the lock.
         */
        p = inode->i_cdev;
        if (!p) {
            inode->i_cdev = p = new;
            list_add(&inode->i_devices, &p->list);
            new = NULL;
        } else if (!cdev_get(p))
            ret = -ENXIO;
    } else if (!cdev_get(p))
        ret = -ENXIO;
    spin_unlock(&cdev_lock);
    cdev_put(new);
    if (ret)
        return ret;
    ret = -ENXIO;
    fops = fops_get(p->ops);
    if (!fops)
        goto out_cdev_put;
    replace_fops(filp, fops);
    if (filp->f_op->open) {
        ret = filp->f_op->open(inode, filp);
        if (ret)
            goto out_cdev_put;
    }
    return 0;
out_cdev_put:
    cdev_put(p);
    return ret;
}

// drivers/base/map.c
struct kobject *kobj_lookup(struct kobj_map *domain, dev_t dev, int *index);
```
- 从哈希表中查找给定的 *kobject*：`kobj_lookup(cdev_map, inode->i_rdev, &idex)`。
- 将 *inode* 与 *cdev* 联系起来：`list_add(&inode->i_devices, &p->list)`。 
- 基于 *inode* 对应的 *cdev*，找到 *file_operations*，并将该 *fops* 赋值给 `struct file` 的 `f_op` 字段：`replace_fops(filp, fops)`。于是后续就可以通过该 *file* 来操作该 *cdev*。
- *inode* 中的设备号（*inode* 的 *cdev* 字段为空）-> 遍历 `cdev_map`，给定设备号在某个 *probe* 的 *devt ~ devt + range - 1* 范围内（best） -> 通过该 *probe* 的 `get()` 回调获取 *kobject* -> *kobject* *back-casting* 得到 *cdev* -> 将 *inode* 链接到 *cdev* 上 + 将 *file* 的 *file_operations* 指向 *cdev* 的 *file_operations*。
- `chrdev_open()` 的调用链：
    ```c
    // version: 4.1.15
    // fs/char_dev.c
    /* 
     * Dummy default file-operations: the only thing this does 
     * is contain the open that then fills in the correct operations
     * depending on the special file...
     */
    const struct file_operations def_chr_fops = {
        .open = chrdev_open,
        .llseek = noop_llseek,
    };

    // fs/inode.c
    void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev) {
        inode->i_mode = mode;
        if (S_ISCHR(mode)) {
            inode->i_fop = &def_chr_fops;
            inode->i_rdev = rdev;
        } else if (S_ISBLK(mode)) {
            inode->i_fop = &def_blk_fops;
            inode->rdev = rdev;
        } else if (S_ISFIFO(mode))
            inode->i_fop = &pipefifo_fops;
        else if (S_ISSOCK(mode))
            ;   /* level it no_open_fops */
        else
            printk(KERN_DEBUG "init_special_inode: bogus i_mode (%o)"
            " for inode %s:%lu\n", mode, inode->i_sb->s_id, inode->i_ino);
    }
    ```
    - `init_special_inode()` 被多个文件系统（如 *ext2,3,4*, *devpfs* 等）调用。
