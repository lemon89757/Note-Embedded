# sysfs
## 参考
1. 内核文档
   - `Documentation/zh_CN/filesystems/sysfs.txt`；
   - `Documentation/filesystems/sysfs.txt`；

> - sysfs 是一个最初**基于 ramfs 且位于内存的文件系统**。它提供导出内核数据结构及其属性，以及它们之间的关联到用户空间的方法。
> - sysfs 始终与 kobject 的底层结构紧密相关。任何 kobject 在系统中注册，就会有一个目录在 sysfs 中被创建。
> - kobject 的属性可在文件系统中以普通文件的形式导出。sysfs 为属性定义了面向文件 I/O 操作的方法，以提供对内核属性的读写。
   ```c
   // 一个简单的属性结构
   struct attribute {
      char *name;
      struct module *owner;
      umode_t mode;
   };

   int sysfs_create_file(struct kobject *kobj, const struct attribute *attr);
   void sysfs_remove_file(struct kobject *kobj, const struct attribute *attr);

   // 一个单独的属性结构并不包含读写其属性值的方法。子系统最好为增删特定对象类型的属性定义自己的属性结构体和封装函数。
   
   // 设备（include/linux/device.h）
   struct device_attribute {
      struct attribute attr;
      ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
      ssize_t (*store)(struct device *dev, struct device_attribute *attr, const char *buf, size_t count);
   }
   // 增删属性
   int device_create_file(struct device *, const struct device_attribute *);
   void device_remove_file(struct device *, const struct device_attribute *);
   // 为了定义设备属性，同时定义了辅助宏
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
   
   // 总线驱动程序（include/linux/device.h）
   struct bus_attribute {
      struct attribute attr;
      ssize_t (*show)(struct bus_type *, const *buf);
      ssize_t (*store)(struct bus_byte *, const char *buf, size_t count);
   }
   // 声明
   BUA_ATTR(_name, _mode, _show, _store)
   // 增删属性
   int bus_create_fie(struct bus_type *, struct bus_attribute *);
   void bus_remove_file(struct bus_type *, struct bus_attribute *);

   // 设备驱动程序（include/linux/device.h）
   struct driver_attribute {
      struct attribute attr;
      ssize_t (*show)(struct device_driver*, char *buf);
      ssize_t (*store)(struct device_driver*, const char *buf, size_t count);
   };
   // 声明
   DRIVER_ATTR(_name, _mode, _show, _store)
   // 增删属性
   int driver_create_file(struct device_driver *, const struct driver_attribute *);
   void driver_remove_file(struct device_driver *, const struct driver_attribute *);

   // 当一个子系统定义一个新的属性类型时，必须实现一系列的 sysfs 操作，以帮助读写调用实现属性所有者的显示和存储方法。
   struct sysfs_ops {
      ssize_t (*show)(struct kobject *, struct attribute *, char *);
      ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
   }
   // sysfs 会为这个类型调用适当的方法。当一个文件被读写时，这个方法会将一般的 kobject 和 attribute 结构体指针转换为适当的指针类型后调用相关联的函数。例如：
   #define to_dev(obj) container_of(obj, struct device, kobj)
   #define to_dev_attr(_attr) container_of(_attr, struct device_attribute, attr)

   static ssize_t dev_attr_show(struct kobject *kobj, struct attribute *attr, char *buf) {
      struct device_attribute *dev_attr = to_dev_attr(attr);
      struct device *dev = to_dev(kobj);
      ssize_t ret = -EIO;

      if (dev_attr->show) {
         ret = dev_attr->show(dev, dev_attr, buf);
      }
      if (ret >= (ssize_t)PAGE_SIZE) {
         print_symbol("dev_attr_show: %s returned bad count\n", (unsigned long)dev_attr->show);
      }
      return ret;
   }
   ```
> - 目录结构，顶层 sysfs 目录如下：
   ```
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
   - `bus/`：包含内核中各种总线类型的平面目录布局。每个总线目录包含两个子目录：`devices/` 和 `drivers/`。`devices/` 包含了系统中出现的每个设备的符号链接，它们指向 `root/` 下的设备目录；`drivers` 包含了每个已为特定总线上的设备而挂载的驱动程序的目录（这里假设驱动没有跨越多个总线类型）；
   - `fs/`：包含了一个为文件系统设立的目录。现在每个想要导出属性的文件系统必须在 `fs/` 下创建自己的层次结构（参见`Documentation/filesystems/fuse.txt`）；
   - `dev/`：包含两个子目录：`char/` 和 `block/`。在这两个子目录中，有以 `<major>:<minor>` 格式命名的符号链接。这些符号链接指向 sysfs 目录中相应的设备。`/sys/dev` 提供一个通过 `stat(2)` 操作结果，查找设备 sysfs 接口快捷的方法。
> - sysfs 目录结构以及其中包含的属性定义了一个内核与用户空间之间的 ABI。对于任何 ABI，其自身的稳定和适当的文档是非常重要的。所有新的 sysfs 属性必须在 `Documentation/ABI` 中有文档。
> - 只要内核配置中定义了 CONFIG_SYSFS，sysfs 总是被编译进内核。你可以通过以下命令挂载它：`mount -t sysfs sysfs /sys`。

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