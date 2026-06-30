# sysfs
## sysfs.txt
将内核文档中 `sysfs.txt` 的中文翻译[1] 的内容摘抄如下。

### sysfs 简介
sysfs 是一个最初**基于 ramfs 且位于内存的文件系统**。**它提供导出内核数据结构及其属性，以及它们之间的关联到用户空间的方法**。

sysfs 始终与 kobject 的底层结构紧密相关。请阅读 `Documentation/kobject.txt` 文档以获得更多关于 kobject 接口的信息。

### 使用 sysfs
只要内容配置中定义了 `CONFIG_SYSFS`，sysfs 总是被编译进内核。你可以通过以下命令挂载它：
```bash
mount -t sysfs sysfs /sys
```

### 创建目录
**任何 kobject 在系统中注册，就会有一个目录在 sysfs 中被创建**。这个目录是作为该 kobject 的父对象所在目录的子目录创建的，以准确地传递内核的对象层次到用户空间。sysfs 中的顶层目录代表着内核对象层次的共同祖先；例如：某些对象属于某个子系统。

Sysfs 在与其目录关联的 sysfs_dirent 对象中内部保存一个指向实现目录的 kobject 的指针。以前，这个 kobject 指针被 sysfs 直接用于 kobject 文件打开和关闭的引用计数。而现在的 sysfs 实现中，kobject 引用计数只能通过 `sysfs_schedule_callback()` 函数直接修改。

### 属性
**kobject 的属性可在文件系统中以普通文件的形式导出**。sysfs 为属性定义了面向文件 I/O 操作的方法，以提供对内核属性的读写。

属性应为 ASCII 码文本文件。以一个文件只存储一个属性值为宜。但一个文件只包含一个属性值可能影响效率，所以一个包含相同数据类型的属性值数组也被广泛地接受。

混合类型、表达多行数据以及一些怪异的数据格式会遭到强烈反对。这样做是很丢脸的，而且其代码会在未通知作者的情况下被重写。

一个简单的属性结构定义如下：
```c
// 一个简单的属性结构
struct attribute {
   char *name;
   struct module *owner;
   umode_t mode;
};

int sysfs_create_file(struct kobject *kobj, const struct attribute *attr);
void sysfs_remove_file(struct kobject *kobj, const struct attribute *attr);
```

**一个单独的属性结构并不包含读写其属性值的方法。子系统最好为增删特定对象类型的属性定义自己的属性结构体和封装函数**。

例如：驱动程序模型定义的 device_attribute 结构体如下：
```c
// include/linux/device.h
struct device_attribute {
   struct attribute attr;
   ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
   ssize_t (*store)(struct device *dev, struct device_attribute *attr, const char *buf, size_t count);
}
   
// 增删属性
int device_create_file(struct device *, const struct device_attribute *);
void device_remove_file(struct device *, const struct device_attribute *);
```
为了定义设备属性，同时定义了辅助宏
```c
#define DEVICE_ATTR(_name, _mode, _show, _store) \
   struct device_attribute dev_attr_##_name = __ATTR(_name, _mode, _show, _store)

// 示例
static DEVICE_ATTR(foo, S_IWUSR | S_IRUGO, show_foo, store_foo);
// 等同于
static struct device_attribute dev_attr_foo = {
   .attr = {
      .name = "foo",
      .mode = S_IWUSR | S_IRUGO,
   },
   .show = show_foo,
   .store = store_foo,
};
```
- [X] attribute 的 umode_t 类型的 mode 字段是指属性文件的访问权限吗？
  - umode_t 是内核中表示文件类型和访问权限的标准类型，等效于用户空间的 mode_t。
  - 其他内容见笔记 `kernel.md`。
- [X] 属性文件的创建是根据 attribue 来的，那 device_attribute 的两个回调函数 show 和 store 又是在哪里调用的呢？例如 `samples/kobject/kobject-example.c` 中的 `foo_attribute` 等。
  - 答案是与子系统有关，在 “子系统特有的回调函数” 部分有介绍。
  - `kobject-example.c` 的解析见笔记 `deviceModel.md`。

### 子系统特有的回调函数
当一个子系统定义一个新的属性类型时，必须实现一系列的 sysfs 操作，以帮助读写调用实现属性所有者的显示和存储方法。
```c
// include/linux/sysfs.h
struct sysfs_ops {
   ssize_t (*show)(struct kobject *, struct attribute *, char *);
   ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```
[子系统应已定义了一个 struct kobj_type 结构体作为这个类型的描述符，并在此保存 sysfs_ops 的指针。更多信息参见 kobject 的文档]

sysfs 会为这个类型调用适当的方法。当一个文件被读写时，这个方法会将一般的 kobject 和 attribute 结构体指针转换为适当的指针类型后调用相关联的函数。

示例：
```c
#define to_dev(obj) container_of(obj, struct device, kobj)
#define to_dev_attr(_attr) container_of(_attr, struct device_attribute, attr)

static ssize_t dev_attr_show(struct kobject *kobj, struct attribute *attr, char *buf) {
   struct device_attribute *dev_attr = to_dev_attr(attr);
   struct device *dev = to_dev(kobj);
   ssize_t ret = -EIO;

   if (dev_attr->show)
      ret = dev_attr->show(dev, dev_attr, buf);
   if (ret >= (ssize_t)PAGE_SIZE) {
      print_symbol("dev_attr_show: %s returned bad count\n", (unsigned long)dev_attr->show);
   }
   return ret;
}
```

### 读写属性数据
在声明属性时，必须指定 show() 和 store() 方法，以实现属性的读或写。这些方法的类型应该和以下的设备属性定义一样简单。
```c
ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
ssize_t (*store)(struct device *dev, struct device_attribute *attr, const char *buf, size_t count);
```
也就是说，他们应只以一个处理对象、一个属性和一个缓冲指针作为参数。

**sysfs 会分配一个大小为 PAGE_SIZE 的缓冲区并传递给这个方法**。sysfs 将会为每次读写操作调用一次这个方法。这使得这些方法在执行时会出现以下的行为：
- 在读方面（read(2)），show() 方法应该填充整个缓冲区。回想属性应只导出了一个属性值或是一个同类型属性值的数组，所以这个代价将不会不太高。
  - 这使得用户空间可以局部地读和任意的向前搜索整个文件。如果用户空间向后搜索到零或使用 '0' 偏移执行一个 pread(2) 操作，show() 方法将再次被调用，以重新填充缓冲。
- 在写方面（write(2)），sysfs 希望在一次写操作时得到整个缓冲区。之后 sysfs 传递整个缓冲区给 store() 方法。
  - **当要写 sysfs 文件时，用户空间进程应首先读取整个文件，修改想要改变的值，然后回写整个缓冲区**。
  - 在读写属性值时，属性方法的执行应操作相同的缓冲区。
- 注记：
  - 写操作导致 show() 方法重载，会忽略当前文件为止。
  - 缓冲区应总是 PAGE_SIZE 大小。对于 i386，这个值为 4096。
  - show() 方法应该返回写入缓冲区的字节数，也就是 snprintf() 的返回值。
  - **show() 应时钟使用 snprintf()**。
  - store() 应返回缓冲区的已用字节数。如果整个缓存都已填满，只需返回 count 参数。
  - show() 和 store() 可以返回错误值。当得到一个非法值，必须返回一个错误值。
  - 一个传递给方法的对象将会通过 sysfs 调用对象内嵌的引用计数固定在内存中。尽管如此，对象代表的物理实体（如设备）可能已不存在。如有必要，应该实现一个检测机制。
- 一个简单的（未经实验证实的）设备属性实现如下：
   ```c
   static ssize_t show_name(struct device *dev, struct device_attribute *attr, char *buf) {
      return scnprintf(buf, PAGE_SIZE, "%s\n", dev->name);
   }

   static ssize_t store_name(struct device *dev, struct device_attribute *attr, const char *buf, size_t count) {
      snprintf(dev->name, sizeof(dev->name), "%.*s", (int)min(count, sizeof(dev->name) - 1), buf);
      return count;
   }

   static DEVICE_ATTR(name, S_IRUGO, show_name, store_name);
   ```
   - 注意：**真正的实现不允许用户空间设置设备名**。
   - [X] S_IRUGO...
     - 见笔记 `kernel.md`
   - [X] snprintf, scnprintf...
     - 见笔记 `kernel.md`

### 顶层目录布局
sysfs 目录的安排显示了内核数据结构之间的关系。

顶层 sysfs 目录如下：
```txt
block/
bus/
class/
dev/
devices/
firmware/
net/
fs/
```
- `devices/`：包含了一个设备树的文件系统表示。直接映射了内部的内核设备树，反映了设备的层次结构；
- `bus/`：包含内核中各种总线类型的平面目录布局。每个总线目录包含两个子目录：
  - `devices/`：`drivers` 包含了每个已为特定总线上的设备而挂载的驱动程序的目录（这里假设驱动没有跨越多个总线类型）；
  - `drivers/`：`devices` 包含了系统中出现的每个设备的符号链接，它们指向 `root/` 下的设备目录；
- `fs/`：包含了一个为文件系统设立的目录。现在每个想要导出属性的文件系统必须在 `fs/` 下创建自己的层次结构（参见`Documentation/filesystems/fuse.txt`）；
- `dev/`：包含两个子目录：`char/` 和 `block/`。在这两个子目录中，有以 `<major>:<minor>` 格式命名的符号链接。这些符号链接指向 sysfs 目录中相应的设备。`/sys/dev` 提供一个通过 `stat(2)` 操作结果，查找设备 sysfs 接口快捷的方法。

更多有关 driver-model 的特性信息可以在 `Documentation/driver-model/` 中找到。

T ODO: 完成这一节。

### 当前接口
以下的接口层普遍存在于当前的 sysfs 中：
```c
// 设备（include/linux/device.h）
// 结构体
struct device_attribute {
   struct attribute attr;
   ssize_t (*show)(struct device *dev, struct device_attr *attr, char *buf);
   ssize_t (*store)(struct device *dev, struct device_attr *attr, const char *buf, size_t count);
};
// 声明
DEVICE_ATTR(_name, _mode, _show, _store);
// 增/删属性
int device_create_file(struct device *dev, const struct device_attribute *attr);
void device_remove_file(struct device *dev, const struct device_attribute *attr);

// 总线（include/linux/device.h）
// 结构体
struct bus_attribute {
   struct attribute attr;
   ssize_t (*show)(struct bus_type *, const *buf);
   ssize_t (*store)(struct bus_byte *, const char *buf, size_t count);
};
// 声明
#define BUS_ATTR(_name, _mode, _show, _store) \
   struct bus_attribute bus_attr_##_name = __ATTR(_name, _mode, _show, _store)
#define BUS_ATTR_RW(_name) \
   struct bus_attribute bus_attr_##_name = __ATTR_RW(_name)
#define BUS_ATTR_RO(_name) \
   struct bus_attribute bus_attr_##_name = __ATTR_RO(_name)
// 增删属性
extern int __must_check bus_create_fie(struct bus_type *, struct bus_attribute *);
extern void bus_remove_file(struct bus_type *, struct bus_attribute *);

// 驱动（include/linux/device.h）
// 结构体
struct driver_attribute {
   struct attribute attr;
   ssize_t (*show)(struct device_driver*, char *buf);
   ssize_t (*store)(struct device_driver*, const char *buf, size_t count);
};
// 声明
#define DRIVER_ATTR(_name, _mode, _show, _store) \
   struct driver_attribute driver_attr_##_name = __ATTR(_name, _mode, _show, _store)
#define DRIVER_ATTR_RW(_name) \
   struct driver_attribute driver_attr_##_name = __ATTR_RW(_name);
#define DRIVER_ATTR_RO(_name) \
   struct driver_attribute driver_attr_##_name = __ATTR_RO(_name);
#define DRIVER_ATTR_WO(_name) \
   struct driver_attribute driver_attr_##_name = __ATTR_WO(_name);
// 增/删属性
extern int __must_check driver_create_file(struct device_driver *, const struct driver_attribute *);
extern void driver_remove_file(struct device_driver *, const struct driver_attribute *);

// include/linux/sysfs.h
/**
 * Use these macros to make defining attributes easier. See include/
 * linux/device.h for examples.. 
 */
#define SYSFS_PREALLOC 010000

#define __ATTR(_name, _mode, _show, _store) { \
   .attr = {.name = __stringify(_name), \
         .mode = VERIFY_OCTAL_PERMISSIONS(_mode) }, \
   .show = _show, \
   .store = _store, \
}

// __ATTR_RW... 的定义，略。
```

### 文档
sysfs 目录结构以及其中包含的属性定义了一个内核与用户空间之间的 ABI。对于任何 ABI，其自身的稳定和适当的文档是非常重要的。所有新的 sysfs 属性必须在 `Documentation/ABI` 中有文档。详见 `Documentatin/ABI/README`。

## 各目录间的关系
> [嵌入式 Linux -- uevent 机制：uevent 原理分析](https://blog.csdn.net/W1107101310/article/details/80211885)

- `/sys/block`：历史遗留问题，存放块设备，提供以设备名（如 sda）到 `/sys/devices` 的符号连接；
- `/sys/bus`：按总线类型分类，在每个总线目录下存在 `devices` 和 `drivers` 两个子目录。前者包含总线上设备的符号链接，指向 `/sys/devices`，后者包含了该总线所需的所有驱动的符号链接。对应 kernel 中的 `struct bus_type`；
- `/sys/class`：按设备功能分类，如输入设备在 `/sys/class/input` 之下，图形设备在 `/sys/class/graphics` 之下，是指向 `/sys/devices` 目录下对应设备的符号链接。对应 kernel 中的 `struct class`；
- `/sys/dev`：按设备驱动程序分层（字符设备、块设备），提供以 `major:minor` 为名到 `/sys/devices` 的符号链接。对应 kernel 中的 `struct device_driver`；
- `/sys/devices`：包含所有被发现的注册在各种总线上的各种物理设备。所有的物理设备都按其在总线上的拓扑结构来显示，除了 `platform device` 和 `system device`。`platform device` 一般是挂在芯片内部高速或者低速总线上的各种控制器和外设，能被 CPU 直接寻址。`system device` 不是外设，是芯片内部的核心结构，比如 CPU、timer 等，他们一般没有相关的 driver，但是会有一些体系结构相关的代码来配置它们对应 kernel 中的 `struct device`；

## sys文件系统、设备模型与挂载
- 设备（kobject）在系统中注册，就会有一个目录在 sysfs 中被创建；
- 挂载的本质是将不同存储设备（硬盘分区、U盘、网络共享、内存文件系统等）**嫁接**到统一的目录树中，使它们成为全局命名空间的一部分；
- 设备的发现和存储设备的挂载是两种不同的机制。
  - 设备的发现是由内核（与驱动相关）自动完成的，通过 uevent 机制在 sysfs 中创建设备节点文件，比如 `/dev/sda1`，它是裸的硬件接口；
  - 存储设备的挂载并不是自动完成的，将设备上的文件系统接入 vfs 树，创建对应入口文件，比如 `/mnt/data`，它经过了 vfs 的解析；
  - 它们是两种互相独立的机制，挂载是后续可选的步骤。没有挂载时，可以通过 sysfs 中的文件访问到设备，可以读取原始字节数据，但无法看到设备中的文件。
- sysfs 的挂载
  - sysfs 是伪文件系统（pseudo-filesystem），完全由内核动态生成，不依赖任何底层存储设备，完全在内存中；
  - sysfs 文件系统挂载到 `/sys` 目录下；
  - 由 `initramfs` 或内核直接挂载，挂载命令：`mount -t sysfs sysfs /sys`；
  - `initramfs` 中的挂载脚本
    - 内核解压启动；
    - `setup_arch()` 初始化硬件架构；
    - `vfs_caches_init()` 初始化 VFS 层；
    - `mnt_init()` 初始化根文件系统相关（此时还没有用户空间）；
    - `kernel_init()` 第一个线程 -> 如果配置了 initramfs，解压并挂载为临时根；如果没有 initramfs，尝试挂载真实根文件系统 -> 在用户空间 init 进程启动前/后，进行相关文件系统挂载：
   ```sh
   #/init (initramfs 中的第一个脚本，由内核直接执行)
   #!/bin/sh

   # 1. 挂载核心伪文件系统（无依赖！）
   mount -t proc proc /proc   # 进程信息
   mount -t sysfs sysfs /sys  # 设备模型
   mount -t devtempfs devtenpfs /dev # 设备节点

   # 2. 现在才能通过 /sys 访问设备信息
   echo /sys/block/sda/dev    # 输出 8:0

   # 3. 加载驱动（可能产生新的 /dev 节点）
   modprobe ext4

   # 4. 挂载真实根文件系统
   mount /dev/sda1 /mnt/root

   # 5. 切换根
   exec switch_root /mnt/root /sbin/init
   ```

## 使用
`/sys` 目录下有很多内核、驱动的信息。

1. 查看设备树信息
   ```bash
   ls /sys/firmware
   # 输出 devicetree fdt
   ```
   - `/sys/firmware/device` 目录下是以目录结构呈现的 `dtb` 文件，根节点对应 `base` 目录，每一个节点对应一个目录，每一个属性对应一个文件；
   - 设备正常加载在 `/sys/firmware/devicetree/base` 下有对应节点；
   - 属性查看：
     - 字符串类型：通过 `cat` 命令打印；
     - 数值类型：通过 `hexdump` 打印；
2. platform_device
   在 `/sys/devices/platform` 目录下存在目标 `platform_device`。
   - 一个设备对应一个目录，进入某个目录后，如果它有“driver”子目录，就表示这个 `platform_device` 跟某个 `platform_driver` 配对了；
   - 一个 `platform_device` 只能配对一个 `platform_driver`；
   - 查看驱动信息，可以找到驱动所在的位置：
    ```bash
    ls driver -ld
    ```
3. platform_driver
   在 `/sys/bus/platform/drivers` 目录下含有注册进内核的所有 `platform_driver`。
   - 一个 `driver` 对应一个目录，进入某个目录后，如果它有配对的设备，可以直接看到。
   - 一个 `platform_driver` 可以配对多个 `platform_device`；

## 参考
1. 内核文档
   - `Documentation/zh_CN/filesystems/sysfs.txt`；
   - `Documentation/filesystems/sysfs.txt`；