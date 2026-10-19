# platform 设备驱动
## 概述
考虑到驱动的可重用性，提出驱动的分离与分层。
- 分离：资源与驱动分离（设备树描述 + 总线匹配设备和驱动）、主机驱动与设备驱动分离；
- 分层：通过中间层（核心层）提供统一的 *API* 接口来实现分层，中间层为硬件驱动层提供统一的实现（回调函数）接口，同时为上层提供统一的服务接口。

*platform* 架构是对总线-设备-驱动模型的实现，在该模型的基础上，它引入了 *platform_device* 和 *platform_driver*，实现 **将“资源”与“驱动”分离**，使得代码易于扩展。

*platform* 不同于如 *SPI* 和 *I2C* 等实际的物理总线，它是一个抽象的虚拟总线。*SoC* 上外设控制器的描述及其驱动的实现通常基于 *platform* 架构。另外，当某个设备没有对应挂接的物理总线（如 *GPIO* 上的设备），一般都可以借助 *platform* 架构。这使得对总线-设备-驱动模型的实现更加便捷、高效了。

在解析设备树资源时，`struct device_node` 实例在某些条件下可以进一步转换为 `struct platform_device` 的实例，从而驱动程序员可以基于 *platform* 架构中对 *platform_device* 提供的相关 *API* 完成对设备树硬件资源的解析。这进一步提高了驱动程序开发的效率。

资源表示方式的比较：
- *.c* 文件：冗余代码多，修改硬件配置时需要重新编译；
- *.dts* 文件：无冗余代码，修改硬件配置时只需修改设备树文件并重新编译后传给内核即可（无需重新编译内核/驱动）；

## 使用
### platform 设备
1. 通过 *C* 程序源码的 *platform_device* 来描述设备信息；
   - 向内核注册：
      ```c
      // 负数-失败；0-成功
      int platform_device_register(struct platform_device *pdev);
      ```
   - 不再使用时注销掉：
      ```c
      void platfrom_device_unregister(struct platform_device *pdev);
      ```
   - 代码框架：
      ```c
      #define PERIPH1_REGISTER_BASE       (0x20000000)
      #define PERIPH2_REGISTER_BASE       (0x020e0068)
      #define REGISTER_LENGTH             4

      static struct resource xxx_resources[] = {
         [0] = {
            .start = PERIPH1_REGISTER_BASE,
            .end   = (PERIPH1_REGISTER_BASE + REGISTER_LENGTH - 1),
            .flags = IORESOURCE_MEM,
         },
         [1] = {
            .start = PERIPH2_REGISTER_BASE,
            .end   = (PERIPH2_REGISTER_BASE + REGISTER_LENGTH - 1),
            .flags = IORESOURCE_MEM,
         },
      };

      static struct platform_device xxx_device = {
         .name = "xxx",
         .id = -1,
         .num_resources = ARRAY_SIZE(xxx_resources),
         .resource = xxx_resources,
         .dev {
            .release = xxx,
         },
      };

      static int __init xxxdevice_init(void) {
         return platform_device_register(&xxx_device);
      }

      static void __exit xxxdevice_exit(void) {
         platform_device_unregister(&xxx_device);
      }

      module_init(xxxdevice_init);
      module_exit(xxxdevice_exit);
      MODULE_LICENSE("GPL");
      ```

2. 通过设备树描述：
   ```dts
   ```

### platform 驱动
1. `platform_driver->probe()`：设备与驱动匹配成功后被调用，一般在该函数中完成：
   - 分配、设置、注册 *file_operations* 结构体；
   - 根据设备树解析硬件；
   - 使用 `ioremap()` 映射寄存器，操作寄存器；
   - ...；

2. `platform_driver->driver`：会在判断设备与驱动是否匹配的过程中被调用（比较名字）；

3. 在编写 *platform* 驱动的时候：
   1. 首先定义一个 *platfrom_driver* 结构体变量；
   2. 实现结构体中的各个成员，重点是实现匹配方法及 *probe* 函数；
   3. 在定义并初始化 *platform_driver* 结构体变量后，在驱动入口函数中向内核注册 *platform* 驱动：
      ```c
      // 负数-失败；0-成功
      int platform_driver_register(struct platform_driver *driver);
      ```
   4. 在驱动卸载的时候卸载 *platform* 驱动：
   ```c
   void platform_driver_unregister(struct platfrom_driver *drv);
   ```

4. 驱动框架
   ```c
   struct xxx_dev {
      struct cdev cdev;
      ...
   };

   struct xxx_dev xxxdev;

   static int xxx_open(struct inode *inode, struct file *filp) { ... }

   static ssize_t xxx_write(struct file *filp, const char __user *buf, size_t size, loff_t *offt) { ... }

   static struct file_operations xxx_fops = {
      .owner = THIS_MODULE,
      .open  = xxx_open,
      .write = xxx_write,
   };

   static int xxx_probe(struct platfrom_device *dev) {
      ...
      cdev_init(&xxxdev.cdev, &xxx_fops);
      ...
      return 0;
   }

   static int xxx_remove(struct platfrom_device *dev) {
      ...
      cdev_del(&xxxdev.cdev);
      ...
      return 0;
   }

   static const struct of_device_id xxx_of_match[] = {
      {.compatible = "xxx"},
      { /* sentinel */ }
   };

   static struct platform_driver xxx_driver = {
      .driver = {
         .name = "xxxuuu";
         .of_match_table = xxx_of_match,
      },
      .probe = xxx_probe,
      .remove = xxx_remove,
   };

   static int __init xxxdriver_init(void) {
      return platform_driver_register(&xxx_driver);
   }

   static void __exit xxxdriver_exit(void) {
      platform_driver_unregister(&xxx_driver);
   }

   module_init(xxxdriver_init);
   module_exit(xxxdriver_exit);
   MODULE_LICENSE("GPL");
   ```

### 设备树相关
#### 设备树中转化为内核中 platform_device 的结点
1. 会转化为 *platform_device*：
   - 根节点下含有 *compatible* 属性的子节点；
   - 含有特定 *compatible* 属性的**节点的子节点**：
     - 如果一个节点的 *compatible* 属性值为这 4 者之一：`"simple-bus", "simple-mfd", "isa", "arm,amba-bus"`；
     - 含有上述 *compatible* 属性值的节点的子节点、且子节点含有 *compatible* 属性可以转化。
2. 不会转化为 *platform_device*：
   - 总线 *I2C*、*SPI* 控制器节点下的子节点不转化为 *plaform_device*（某个总线控制器下的子节点应该交给对应的总线驱动程序来处理）

#### 设备树中的资源信息
- *platform_device* 中含有 *resource* 数组，它来自设备树节点的 *reg*、*interrupts* 属性；
- `platform_device->dev.of_node` 指向 *device_node* 实例，可以通过它获得其他设备树结点属性；

#### 设备树与 platform_device 相关函数
1. `of_find_device_by_node()`：
   ```c
   extern struct platform_device *of_find_device_by_node(struct device_node *np);
   ```
   - 设备树中的每一个节点，在内核中都有一个 *device_node* 实例，可以通过它去找到对应的 *platform_device* 实例

### 设备与驱动的匹配判断
判断设备与驱动是否匹配的比较顺序：
1. `platform_device->driver_override` 和 `platform_driver->driver.name`；
2. *OF style*，设备树对应的匹配方式：`platform_driver->driver.of_match_table`，`struct device_driver` 有个名为 `of_match_table` 的字段，对应着 `struct of_device_id` 类型的指针，可以用于表示支持的设备列表。`{.compatible = xxx}` 会和设备树中节点的 *compatible* 属性相比较。如果相同的话则实现设备和驱动的匹配；
   ```c
   // 比较的是 pdev 中的 of_node 字段，它的类型是：
   struct device_node {
      const char *name; // 来自节点的 name 属性
      const char *type; // 来自节点的 device_type 属性，已弃用
      phandle phandle;
      const char *full_name;
      struct fwnode_handle fwnode;
      struct property *properties;     // 含有 compatible 属性
      // ...
   };
   ```
3. *ACPI* 方式，略；
4. `platform_device->name` 和 `platform_driver->id_table[i].name`：`platform_driver.id_table` 是 `struct platform_device_id` 类型的指针，表示该驱动支持若干个设备。`{.name = xxx, .driver_data=xxx, }` 中 `name` 字段表示该驱动支持的设备的名字，`driver_data` 字段是提供给该设备的私有数据；
5. `platform_device->name` 和 `platform_driver->driver.name`：`platform_driver.id_table` 可能为空，这时可以根据 `platform_driver->driver.name` 来寻找同名的 *platform_device*；

### API
1. `platform_set_drvdata()`
   ```c
   // version: 4.1.15
   // include/linux/platform_device.h
   static inline void platform_set_drvdata(struct platform_device *pdev, void *data) {
      dev_set_drvdata(&pdev->dev, data);
   }

   // include/linux/device.h
   static inline void dev_set_drvdata(struct device *dev, void *data) {
      dev->driver_data = data;
   }
   ```

2. `dev_get_platdata()`
   ```c
   // version: 4.1.15
   // include/linux/device.h
   static inline void *dev_get_platdata(const struct device *dev) {
      return dev->platform_data;
   }
   ```

3. `EXPORT_SYMBOL(xxx)`：`a.c` 编译为 `a.ko`，里面定义了 `func_a`。如果它想让 `b.ko` 使用该函数，那么 `a.c` 里需要导出此函数（如果 `a.c`，`b.c` 都编进内核，则无需导出）；

4. `platform_get_device_id()`
   ```c
   // version: 4.1.15
   // include/linux/platform_device.h
   #define platform_get_device_id(pdev)	((pdev)->id_entry)
   ```

5. 获取资源
   1. 获取平台设备中指定类型资源中的第 `num` 个：
      ```c
      struct resource *platform_get_resource(struct platform_device *dev, unsigned int type, unsigned int num);
      ```
   2. 获取平台设备中的第 `num` 个中断：
      ```c
      int platform_get_irq(struct platform_device *dev, unsigned int num);
      ```
   3. 获取平台设备中指定类型和资源名称的资源：
      ```c
      struct resource *platform_get_resource_byname(struct platform_device *dev, unsigned int type, const char *name);
      ```
   4. 获取平台设备的指定中断资源名称的中断号：
      ```c
      int platform_get_irq_byname(struct platform_device *dev, const char *name);
      ```


### 示例程序
1. 通过 *.c* 文件描述设备硬件信息（实现 *platform_device*）：
   - 设备描述
      ```c
      // platform_device
      static void led_dev_release(struct device *dev) {

      }

      static struct resource resources[] = {
         {
            .start = GROUP_PIN(3, 1),
            .flags = IORESOURCE_IRQ,
            .name = "100ask_len_pin",
         },
         {
            .start = GROUP_PIN(5, 8),
            .flags = IORESOURCE_IRQ,
            .name = "100ask_led_pin",
         }
      };

      static struct platform_device board_A_led_dev = {
         .name = "100ask_led",
         .num_resources = ARRAY_SIZE(resources),
         .resource = resources,
         .dev = {
            .release = led_dev_release,
         },
      };

      static int __init led_dev_init(void) {
         int err;
         err = platform_device_register(&board_A_led_dev);
         return 0;
      }
      ```
      - 在入口函数中向内核注册 *platform_device*；
      - 注意：如果在 *platform_device* 结构体初始化时未提供 `dev.release()`，则在调用 `platform_device_unregister()` 时会出现警告。可以提供一个 `release` 函数，如果实在无事可做，把这个函数写为空；
      - 要根据依赖关系顺序加载各个模块；
   
   - 驱动实现（部分）
      ```c
      // platform_driver
      static int chip_demo_gpio_probe(struct platform_device *pdev) {
         struct resource *res;
         int i = 0;
         while (1) {
            res = platform_get_resource(pdev, IORESOURCE_IRQ, i++);
            if (!res)
               break;
            
            g_ledpins[g_ledcnt] = res->start;      // 记录 GPIO 引脚信息
            led_class_create_device(g_ledcnt);
            g_ledcnt++;
         }
         return 0;
      }

      static struct platform_driver chip_demo_gpio_driver = {
         .probe = chip_demo_gpio_probe,
         .remove = chip_demo_gpio_remove,
         .driver = {
            .name = "100ask_led",
         },
      };

      static int __init chip_demo_gpio_drv_init(void) {
         int err;

         err = platform_driver_register(&chip_demo_gpio_driver);
         register_led_operations(&board_demo_led_opr);

         return 0;
      }
      ```

2. 通过设备树 *.dts* 文件记录设备硬件信息：
   - 驱动要求设备树节点提供什么，就得按照要求去编写设备树。驱动程序中要根据 *pin* 属性来确定引脚，因此需要在设备树节点中添加 *pin* 属性：
      ```dts
      #define GROUP_PIN(g, p)    ((g << 16) | (p))

      100ask_led@0 {
         compatible = "100ask,leddrv";
         pin = <GROUP_PIN(3, 1)>;
      };
      100ask_led@1 {
         compatible = "100ask,leddrv";
         pin = <GROUP_PIN(5, 8)>;
      };
      ```
   
   - 在驱动程序中，可以从 *platform_device* 中得到 *device_node*，再用 `of_property_read_u32()` 得到属性的值。
      ```c
      // part
      struct device_node *np = pdev->dev.of_node;
      int led_pin;
      int err = of_property_read_u32(np, "pin", &led_pin);

      // 注册 platform_driver
      static const struct of_device_id ask100_leds[] = {
         {.compatible = "100ask,leddrv"},
         {},
      };

      static struct platform_driver chip_demo_gpio_driver = {
         .probe = chip_demo_gpio_probe,
         .remove = chip_demo_gpio_remove,
         .driver = {
            .name = "100ask_led",
            .of_match_table = ask100_leds,
         },
      };

      static int __init chip_demo_gpio_drv_init(void) {
         int err;

         err = platform_driver_register(&chip_demo_gpio_driver);
         register_led_operations(&board_demo_led_opr);

         return 0;
      }
      ```

### 调试
1. *sysfs*
   - 在 `sys/bus/platform/devices` 下对应的设备文件（目录）
   - 在 `sys/bus/platform/drivers` 下对应的驱动文件（目录）
   - 在 `sys/class/` 对应的子系统（设备分类）目录；

## platform 架构
*platform* 架构是基于总线-设备-驱动模型的实现，因此从总线、设备、驱动三个方面描述

### platform 总线
#### 数据结构描述
*platform* 架构对总线 `struct bus_type` 的实现如下：
```c
// version: 4.1.15
// drivers/base/platform.c
struct bus_type platform_bus_type = {
   .name = "platform",
   .dev_groups = platform_dev_groups,
   .match = platform_match,
   .uevent = platform_uevent,
   .pm = &platform_dev_pm_ops,
};
```

#### 总线上设备的属性集
```c
// version: 4.1.15
// drivers/base/platform.c
static struct attribute *platform_dev_attrs[] = {
   &dev_attr_modalias.attr,         // 属性名为 modalias
   &dev_attr_driver_override.attr,  // 属性名为 driver_override
   NULL,
};
ATTRIBUTE_GROUPS(platform_dev);

// include/linux/sysfs.h
#define ATTRIBUTE_GROUPS(_name)  \
static const struct attribute_group _name##_group = { \
   .attrs = _name##attrs,  \
}; \
__ATTRIBUTE_GROUPS(_name)

#define _ATTRIBUTE_GROUPS(_name) \
static const struct attribute_group *_name##_groups[] = { \
   &_name##_group, \
   NULL, \
}
```
- 文件 *modalias*，只读，返回内容：`"platform:%s\n", pdev->name` 或者 `of_device_uevent_modalias()` 返回的内容（见下文解析）。
- 文件 *driver_override*，可读写，读-返回内容：`"%s\n", pdev->driver_override`；写-赋值给 `pdev->driver_override` 字段。

#### match 回调
见 `deviceModel.md` 中“总线的 *match* 机制” 部分。用来判断总线上的设备与驱动是否能够匹配。
```c
// version: 4.1.15
// drivers/base/platform.c
/**
   * platform_match - bind platform device to platform driver 
   * @dev: device
   * @drv: driver
   *
   * Platform device IDs are assumed to be encoded like this:
   * "<name><instance>", where <name> is a short description of the
   * type of device, like "pci" or "floppy", and <instance> is the
   * enumerated instance of the device, like '0' or '42'. Driver
   * IDs are simply "<name>". So, extract the <name> from the
   * platform_device structure, and compare it against the name
   * of the driver. Return whether they match or not.
   */
static int platform_match(struct device *dev, struct device_driver *drv) {
   struct platform_device *pdev = to_platform_device(dev);
   struct platform_driver *pdrv = to_platform_driver(dev);
   /* When driver_override is set, only bind to the matching driver */
   if (pdev->driver_override)
      return !strcmp(pdev->driver_override, drv->name);
   /* Attempt an OF style match first */
   if (of_driver_match_device(dev, drv))
      return 1;
   /* Then try ACPI style match */
   if (acpi_driver_match_device(dev, drv))
      return 1;
   /* Then try to match against the id table */
   if (pdrv->id_table)
      return platform_match_id(pdrv->id_table, pdev) != NULL;
   /* fall-back to driver name match */
   return (strcmp(pdev->name, drv->name) == 0);
}

static const struct platform_device_id *platform_match_id(
   const struct platform_device_id *id, struct platform_device *pdev) {
   while (id->name[0]) {
      if (strcmp(pdev->name, id->name) == 0) {
         pdev->id_entry = id;
         return id;
      }
      id++;
   }
   return NULL;
}
```
   
#### uevent 回调
在该总线上的设备发生状态变化时，会通过该回调添加在发送 *uevent* 时的环境变量。
```c
// version: 4.1.15
// drivers/base/platform.c
static int platform_uevent(struct device *dev, struct kobj_uevent_env *env) {
   struct platform_device *pdev = to_platform_device(dev);
   int rc;
   /* Some devices have extra OF data and an OF-style MODALIAS */
   rc = of_device_uevent_modalias(dev, env);
   if (rc != -ENODEV)
      return rc;
   rc = acpi_device_uevent_modalias(dev, env);
   if (rc != -ENODEV)
      return rc;
   add_uevent_var(env, "MODALIAS=%s%s", PLATFORM_MODULE_PREFIX, 
      pdev->name);
   return 0;
}

// include/linux/mod_devicetable.h
#define PLATFORM_MODULE_PREFIX "platform:"
```
- 通过 `of_device_uevent_modalias()` 添加的环境变量：名字：“*MODALIAS*”，内容：`"of:N%sT%s", dev->of_node->name, dev->of_node->type`，并在该内容之后再添加上 `dev->of_node` 的 "*compatible*" 属性值。

#### pm 操作集
```c
// version: 4.1.15
// drivers/base/platform.c
static const struct dev_pm_ops platform_dev_pm_ops = {
   .runtime_suspend = pm_generic_runtime_suspend,
   .runtime_resume = pm_generic_runtime_resuem,
   USE_PLATFORM_PM_SLEEP_OPS
};

// 上述内容通过宏扩展为：
static const struct dev_pm_ops platform_dev_pm_ops = {
   .runtime_suspend = pm_generic_runtime_suspend,
   .runtime_resume = pm_generic_runtime_resuem,
   .suspend = platform_pm_suspend,
   .resume = platform_pm_resume,
   .freeze = platform_pm_freeze,
   .thaw = platform_pm_thaw, // 解冻、融化
   .poweroff = platform_pm_poweroff,
   .restore = platform_pm_restore,
};

// drivers/base/power/generic_ops.c
/**
   * pm_generic_runtime_suspend - Generic runtime suspend callback for subsystems.
   * @dev: Device to suspend.
   *
   * If PM operations are defined for the @dev's driver and they include
   * ->runtime_suspend(), execute it and return its error code.
   * Otherwise, return 0.
   */
int pm_generic_runtime_suspend(struct device *dev) {
   const struct dev_pm_ops *pm = dev->driver ? dev->driver->pm : NULL;
   int ret;
   ret = pm && pm->runtime_suspend ? pm->runtime_suspend(dev) : 0;
   return ret;
}
EXPORT_SYMBOL_GPL(pm_generic_runtime_suspend);

// pm_generic_runtime_resume() 的实现同上

// drivers/base/platform.c
int platform_pm_suspend(struct device *dev) {
   struct device_driver *drv = dev->driver;
   int ret = 0;
   if (!drv)
      return 0;
   if (drv->pm) {
      if (drv->pm->suspend)
         ret = drv->pm->suspend(dev);
   } else {
      ret = platform_legacy_suspend(dev, PMSG_SUSPEND);
   }
   return ret;
}

static int platform_legacy_suspend(struct device *dev, pm_message_t mesg) {
   struct platform_driver *pdrv = to_platform_driver(dev->driver);
   struct platform_device *pdev = to_platform_device(dev);
   int ret = 0;
   if (dev->driver && pdrv->suspend)
      ret = pdrv->suspend(pdev, mesg);
   return ret;
}

// platform_pm_resume() 的实现同上
```
- 关于 *pm* 的操作，优先使用 `dev->drv->pm` 字段的回调函数，其次考虑 *platform_driver* 的对应回调函数。 

#### 总线注册
```c
// version: 4.1.15
// drivers/base/platform.c
int __init platform_bus_init(void) {
   int error;
   early_platform_cleanup();
   error = device_register(&platform_bus);
   if (error)
      return error;
   error = bus_register(&platform_bus_type);
   if (error)
      device_unregister(&platform_bus);
   of_platform_register_reconfig_notifier();
   return error;
}
```
- 调用关系：
   ```c
   start_kernel()
      rest_init()
         kernel_thread(kernel_init, NULL, CLONE_FS);
   
   kernel_init()
      kernel_init_freeable()
         do_basic_setup()
            driver_init()
               platform_bus_init()
   ```
- 除了注册 `platform_bus_type` 外，还注册了 `platform_bus`（类型为 `struct device`），它将作为 *platform* 总线上设备的默认父设备（`pdev->dev.parent` 为空时）：
   ```c
   struct device platform_bus = {
      .init_name = "platform",
   };
   EXPORT_SYMBOL_GPL(platform_bus);
   ```
- 两个函数：`early_platform_cleanup()` 与 `of_platform_register_reconfig_notifier()` 的实现见源文件。前者与早期阶段注册的 *platform* 设备有关。

### platform 设备
#### 数据结构描述
```c
// version: 4.1.15
// include/linux/platform_device.h
struct platform_device {
   const char *name;
   int id;
   bool id_auto;
   struct device dev;
   u32 num_resourece;
   struct resource *resource;

   const struct platform_device_id *id_entry;
   char *driver_override;  // Driver name to force a match

   /* MFD cell pointer */
   struct mfd_cell *mfd_cell;

   /* arch specific additions */
   struct pdev_archdata *archdata;
}

// include/linux/mod_devicetable.h
struct platform_device_id {
   char name[PLATFORM_NAME_SIZE];
   kernel_ulong_t driver_data;
}

#define PLATFORM_NAME_SIZE 20
```
- `resource`：设备的资源列表
- `driver_override`：非空时，只匹配该字段指定的驱动；
- 卸载 *platform_device*（`platform_device_unregister()`）时会调用 `dev.release()`；

#### platform_device 的注册
1. 直接注册-`platform_device_register()`
   ```c
   // version: 4.1.15
   // drivers/base/platform.c
   /** 
    * platform_device_register - add a platform-level device
    * @pdev: platform device we're adding
    */
   int platform_device_register(struct platform_device *pdev) {
      device_initialize(&pdev->dev);
      arch_setup_pdev_archdata(pdev);
      return platform_device_add(pdev);
   }
   EXPORT_SYMBOL_GPL(platform_device_register)

   /**
    * platform_device_add - add a platform device to device hierarchy 
    * @pdev: platform device we're adding
    *
    * This is part 2 of platform_device_register(), thouth may be called
    * separately __iff__ pdev was allocated by platform_device_alloc().
    */
   int platform_device_add(struct platform_device *pdev) {
      int i, ret;
      if (!pdev)
         return -EINVAL;
      if (!pdev->dev.parent)
         pdev->dev.parent = &platform_bus;
      pdev->dev.bus = &platform_bus_type;
      switch (pdev->id) {
      default:
         dev_set_name(&pdev->dev, "%s.%d", pdev->name, pdev->id);
         break;
      case PLATFORM_DEVID_NONE:
         dev_set_name(&pdev->dev, "%s", pdev->name);
         break;
      case PLATFORM_DEVID_AUTO:
         /* 
          * Automatically allocated device ID. We mark it as such so
          * that we remember it must be freed, and we append a suffix
          * to avoid namespace collision with explicit IDs
          */
         ret = ida_sim_get(&platform_devid_ida, 0, 0, GFP_KERNEL);
         if (ret < 0)
            goto err_out;
         pdev->id = ret;
         pdev->id_auto = true;
         dev_set_name(&pdev->dev, "%s.%d.auto", pdev->name, pdev->id);
         break;
      }
      for (i = 0; i < pdev->num_resources; i++) {
         struct resource *p, *r &pdev->resource[i];
         if (r->name == NULL)
            r->name = dev_name(&pdev->dev);
         p = r->parent;
         if (!p) {
            if (resource_type(r) == IORESOURCE_MEM)
               p = &iomem_resource;
            else if (resource_type(r) == IORESOURCE_IO)
               p = ioport_resource;
         }
         if (p && insert_resource(p, r)) {
            dev_err(&pdev->dev, "failed to claim resource %d\n", i);
            ret = -EBUSY;
            goto failed;
         }
      }
      pr_debug("Registering platform device '%s'. Parent at %s\n",
         dev_name(&pdev->dev), dev_name(&pdev->dev.parent));
      ret = device_add(&pdev->dev);
      if (ret == 0)
         return ret;
   failed:
      if (pdev->id_auto) {
         ida_simple_remove(&platform_device_ida, &pdev->id);
         pdev->id = PLATFORM_DEVID_AUTO;
      }
      while (--i >= 0) {
         struct resource *r = &pdev->resource[i];
         if (r->parent)
            release_resource(r);
      }
   err_out:
      return ret;
   }
   EXPORT_SYMBOL_GPL(platform_device_add);

   // include/linux/platform_device.h
   #define PLATFORM_DEVICE_NONE (-1)
   #define PLATFORM_DEVICE_AUTO (-2)
   ```
   - *resource tree* 和 `ida_simple_get/remove()`，略。
   - `platform_device_register()` 对比 `device_register()` 的两个步骤，在中间增加了一些内容：
     - 设置 *device* 的 `parent` 字段，默认为 `platform_bus`。
     - 设置 *device* 的 `bus` 字段，设置为 `platform_bus_type`。
     - 设置 *device* 的名字，根据 `pdev->id` 来设置。
     - 将 `pdev` 管理的 `resource` 挂到 *resource tree* 上。

2. 先分配，再注册：
   `platform_device_alloc()` + `platform_device_add()`
   ```c
   // version: 4.1.15
   // drivers/base/platform.c
   /**
    * platform_device_alloc - create a platform device 
    * @name: base name of the device we're adding
    * @id: instance id
    *
    * Create a platform device object which can have other objects
    * attached to it, and which will have attached objects freed when
    * it is released.
    */
   struct platform_device *platform_device_alloc(const char *name, int id) {
      struct platform_object *pa;
      pa = kzalloc(sizeof(*pa) + strlen(name) + 1, GFP_KERNEL);
      if (pa) {
         strcpy(pa->name, name);
         pa->pdev.name = pa->name;
         pa->pdev.id = id;
         device_initialize(&pa->pdev.dev);
         pa->pdev.dev.release = platform_device_release;
         arch_setup_pdev_archdata(&pa->pdev);
      }
      return pa ? &pa->pdev : NULL;
   }
   EXPORT_SYMBOL_GPL(platform_device_alloc);

   static void platform_device_release(struct device *dev) {
      struct platform_object *pa = conatiner_of(dev, struct platform_object, pdev.dev);
      of_device_node_put(&pa->pdev.dev);
      kfree(pa->pdev.dev.platform_data);
      kfree(pa->pdev.mfd_cell);
      kfree(pa->pdev.resource);
      kfree(pa->pdev.driver_override);
      kfree(pa);
   }
   ```
   - 指定设备移除时的回调函数：`platform_device_release()`

#### platform_device 的注销
```c
// version: 4.1.15
// drivers/base/platform.c
/**
 * platform_device_unregister - unregister a platform-level device 
 * @pdev: platform device we're unregistering
 *
 * Unregistration is done in 2 steps. First we release all resources
 * and remove it from the subsystem, then we drop reference count
 * by calling platform_device_put().
 */
void platform_device_unregister(struct platform_device *pdev) {
   platform_device_del(pdev);
   platform_device_put(pdev);
}
EXPORT_SYMBOL_GPL(platform_device_unregister);

/**
 * platform_device_del - remove a platform-level device 
 * @pdev: platform device we're removing
 *
 * Note that this function will also release all memory- and port-based
 * resources owned by the device (@dev->resource). This function
 * must _only_ be externally called in error cases. All other usage
 * is a bug.
 */
void platform_device_del(struct platform_device *pdev) {
   int i;
   if (pdev) {
      device_del(&pdev->dev);
      if (pdev->id_auto) {
         ida_simple_remove(&platform_devid_ida, pdev->id);
         pdev->id = PLATFORM_DEVID_AUTO;
      }
      for (i = 0; i < pdev->num_resources; i++) {
         struct resource *r = &pdev->resource[i];
         if (r->parent)
            release_resource(r);
      }
   }
}
EXPORT_SYMBOL_GPL(platform_device_del);

/**
 * platform_device_put - destroy a platform device 
 * @pdev: platform device to free
 *
 * Free all memory associated with a platform device. This function
 * must _only_ be externally called in error cases. All other usage
 * is a bug.
 */
void platform_device_put(struct platform_device *pdev) {
   if (pdev)
      put_device(&pdev->dev);
}
EXPORT_SYMBOL_GPL(platform_device_put);
```

#### platform_device 的资源获取
`struct resource` 结构体描述和资源类型：
```c
// version: 4.1.15
// include/linux/ioport.h
/* Resources are tree-like, allowing 
 * nesting etc..
 */
struct resource {
   resource_size_t start;
   resource_size_t end;
   const char *name;
   unsigned long flags;
   struct resource *parent, *sibling, *child;
}
// start, end: 表示资源的起始和终止信息。对于内存类的资源，就表示内存起始和终止地址；
// flags:
// include/linux/ioport.h
#define IORESOURCE_BITS       0x000000ff     // Bus-specific bits
#define IORESOURCE_TYPE_BITS  0x00001f00     // Resource type
#define IORESOURCE_IO         0x00000100
#define IORESOURCE_MEM        0x00000200
#define IORESOURCE_REG        0x00000300
#define IORESOURCE_IRQ        0x00000400
#define IORESOURCE_DMA        0x00000800
#define IORESOURCE_BUS        0x00001000
...
#define IORESOURCE_PCI_FIXED  (1 << 4)       // Do not move resource
```

1. 基础接口-`platform_get_resource()`
   ```c
   // version: 4.1.15
   // drivers/base/platform.c
   /**
    * platform_get_resource - get a resource for a device 
    * @dev: platform device
    * @type: resource type
    * @num: resource index
    */
   struct resource *platform_get_resource(struct platform_device *dev,
      unsigned int type, unsigned int num) {
      int i;
      for (i = 0; i < dev->num_resources; i++) {
         struct resource *r = &dev->resource[i];
         if (type == resource_type(r) && num-- == 0)
            return r;
      }
      return NULL;
   }
   ```
2. 中断获取-`platform_get_irq()`
   ```c
   // version: 4.1.15
   // drivers/base/platform.c
   /** 
      * platform_get_irq - get an IRQ for a device 
      * @dev: platform device
      * @num: IRQ number index
      */
   int platform_get_irq(struct platform_device *dev, unsigned int num) {
   #ifdef CONFIG_SPARC
      /* sparc does not have irqs represented as IORESORUCE_IRQ resoruces */
      if (!dev || num >= dev->archdata.num_irqs)
         return -ENXIO;
      return dev->archdata.irqs[num];
   #else
      struct resource *r;
      if (IS_ENABLED(CONFIG_OF_IRQ) && dev->dev.of_node) {
         int ret;
         ret = of_irq_get(dev->dev.of_node, num);
         if (ret >= 0 || ret == -EPROBE_DEFER)
            return ret;
      }
      r = platform_get_resource(dev, IORESOURCE_IRQ, num);
      /* The resources may pass trigger flags to the irqs 
         * that need to be set up. It so happens that the trigger
         * flags for IORESOURCE_BITS correspond 1-to-1 to the
         * IRQF_TRIGGER* settings.
         */
      if (r && r->flags & IORESOURCE_BITS)
         irqd_set_trigger_type(irq_get_irq_data(r->start), 
            r->flags & IORESOURCE_BITS);
      return r ? r->start : -ENXIO;
   #endif
   }
   EXPORT_SYMBOL_GPL(platform_get_irq);
   ```

#### 从 device_node 到 platform_device
1. 函数 `of_platform_populate()` 解析：
   ```c
   // version: 4.1.15
   // arch/arm/mach-imx/mach-imx6ul.c
   // imx6ul_init_machine() -> of_platform_populate(NULL, of_default_bus_match_table, NULL, NULL)

   // drivers/of/platform.c
   const struct of_device_id of_default_bus_match_table[] = {
      { .compatible = "simple-bus", },
      { .compatible = "simple-mfd", },
   #ifdef CONFIG_ARM_AMBA
      { .compatible = "arm,amba-bus", },
   #endif
      {} /* Empty terminated list */
   };

   // drivers/of/platform.c
   /**
   * of_platform_populate() - Populate platform_devices from device tree data 
   * @root: parent of the first level to probe or NULL for the root of the tree
   * @matches: match table, NULL to use the default
   * @lookup: auxdata table for matching id and platform_data with devcie nodes
   * @parent: parent to hook devices from, NULL for toplevel
   *
   * Similar to of_platfrom_bus_probe(), this function walks the device tree
   * and creates devices from nodes. It differs in that if follows the modern
   * convention of requiring all device nodes to have a 'compatible' property,
   * and it is suitable for creating devices which are children of the root
   * node (of_platform_bus_probe will only create children of the root which 
   * are selected by the @matches argument).
   */
   int of_platform_populate(struct device_node *root,
      const struct of_device_id *matches,
      const struct of_dev_auxdata *lookup,
      struct device *parent) {
      struct device_node *child;
      int rc = 0;
      root = root ? of_node_get(root) : of_find_node_by_path("/");
      if (!root)
         return -EINVAL;
      for_each_child_of_node(root, child) {
         rc = of_platform_bus_create(child, matches, lookup, parent, true);
         if (rc)
            break;
      }
      of_node_set_flag(root, OF_POPULATED_BUS);
      of_node_put(root);
      return rc;
   }

   /**
   * of_platform_bus_create() - Create a device for a node and its children.
   * @bus: device node of the bus to instantiate
   * @matches: match table for bus nodes
   * @lookup: auxdata table for matching id and platform_data with device nodes
   * @parent: parent for new device, or NULL for top level.
   * @strict: require compatible property
   *
   * Creates a platform_device for the provided device_node, and optionally
   * recursively create devices for all the child nodes.
   */
   static int of_platform_bus_create(struct device_node *bus,
      const struct of_device_id *matches,
      const struct of_dev_auxdata *lookup,
      struct device *parent, bool strict) {
      const struct of_dev_auxdata *auxdata;
      struct device_node *child;
      struct platform_device *dev;
      struct char *bus_id = NULL;
      void *platform_data = NULL;
      int rc = 0;

      /* Make sure it has a compatible property */
      if (strict && (!of_get_property(bus, "compatile", NULL))) {
         pr_debug("%s() - skipping %s, no compatible prop\n",
            __func__, bus->full_name);
         return 0;
      }
      auxdata = of_dev_lookup(lookup, bus);  // 这里因为 lookup 为 NULL, auxdata 为 NULL
      if (auxdata) {
         ... // no executing
      }
      if (of_device_is_compatible(bus, "arm,primecell")) {
         /* 
         * Don't return an error here to keep compatibility with older 
         * device tree files.
         */
         of_amba_device_create(bus, bus_id, platform_data, parent);
         return 0;
      }
      dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
      if (!dev || !of_match_node(matches, bus))
         return 0;
      for_each_child_of_nodes(bus, child) {
         pr_debug("   create child: %s\n", child->full_name);
         rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
         if (rc) {
            of_node_put(child);
            break;
         }
      }
      of_node_set_flag(bus, OF_POPULATE_BUS);
      return rc;
   }
   ```
   - 设备树中将会转换为 *platform_device* 的节点：
     - 设备树根节点下的子节点，且要求子节点具有 *compatible* 属性。
     - 如果设备树根节点下的子节点具有 *compatible* 的属性值为 `"simple-bus"` 或 `"simple-mfd"`，则会对该子节点递归调用 `of_platform_bus_create()` 函数，直到 *platform_device* 创建失败或处理节点的 *compatible* 属性值不符合上述要求。
     - 另外，上述所提及节点还必须：没有 *status* 属性或者 *status* 的属性值为 `"okay"` 或 `"ok"`。
   - 如果设备树中的节点及其子节点均被处理，则其对应 `device_node->_flags` 的 `OF_POPULATE_BUS` 位将会被置位。

2. 由 *device_node* 到 *platform_device* 的转换详情：
   ```c
   // version: 4.1.15
   // drivers/of/platform.c
   /** 
    * of_platform_device_create_pdata - Alloc, initialize and register an of_device
    * @np: pointer to node to create device for
    * @bus_id: name to assign device
    * @platform_data: pointer to populate platform_data pointer with
    * @parent: Linux device model parent device.
    *
    * Returns pointer to created platform device, or NULL if a device was not
    * registered. Unavailable devices will not get registered.
    */
   static struct platform_device *of_platform_device_create_pdata(struct device_node *np,
      const char *bus_id, void *platform_data, struct device* parent) {
      struct platform_device *dev;
      if (!of_device_is_available(np) || of_node_test_and_set_flag(np, OF_POPULATED))
         return NULL;
      dev = of_device_alloc(np, bus_id, parent);
      if (!dev)
         goto err_clear_flag;
      dev->dev.bus = &platform_bus_type;
      dev->dev.platform_data = &platform_data;
      of_dma_configure(&dev->dev, dev->dev.of_node);
      if (of_device_add(dev) != 0) {
         of_dma_deconfigure(&dev->dev);
         platform_device_put(dev);
         goto err_clear_flag;
      }
      return dev;
   err_clear_flag:
      of_node_clear_flag(np, OF_POPULATED);
      return NULL;
   }

   /**
    * of_platform_device_is_available - check if a device is available for use 
    * @device: node to check for availability
    * 
    * Returns true if the status property is absent or set to "okay" or "ok",
    * false otherwise
    */
   bool of_device_is_available(const struct device_node *device) {
      unsigned long flags;
      bool ret;
      raw_spin_lock_irqsave(&devtree_lock, flags);
      res = __of_device_is_available(device);
      raw_spin_unlock_irqrestore(&devtree_lock, flags);
      return res;
   }
   EXPORT_SYMBOL(of_device_is_available);

   static bool __of_device_is_available(const struct device_node *device) {
      const char *status;
      int stalen;
      if (!device)
         return false;
      status = __of_get_property(device, "status", &stalen);
      if (status == NULL)
         return true;
      if (stalen > 0) {
         if (!strcmp(status, "okay") || !strcmp(status, "ok"))
            return true;
      }
      return false;
   }

   /**
    * of_device_alloc - Allocate and initialize an of_device 
    * @np: device node to assign to device
    * @bus_id: Name to assign to the device. May be NULL to use default name.
    * @parent: Parent device.
    */
   struct platform_device *of_device_alloc(struct device_node *np,
      const char *bus_id, struct device *parent) {
      struct platform_device *dev;
      int rc, i, num_reg = 0; num_irq;
      struct resource *res, temp_res;
      dev = platform_device_alloc("", -1);
      if (!dev)
         return NULL;
      /* count the io and irq resources */
      while (of_address_to_resource(np, num_reg, &temp_res) == 0)
         num_reg++;
      num_irq = of_irq_count(np);
      // Populate the resource table
      if (num_irq || num_reg) {
         res = kzalloc(sizeof(*res) * (num_irq + num_reg), GFP_KERNEL);
         if (!res) {
            platform_device_put(dev);
            return NULL;
         }
         dev->num_resources = num_reg + num_irq;
         dev->resource = res;
         for (i = 0; i < num_regs; i++, res++) {
            rc = of_address_to_resource(np, i, res);
            WARN_ON(rc);
         }
         if (of_irq_to_resource_table(np, res, num_irq) != num_irq)
            pr_debug("not all legacy IRQ resources mapped for %s\n", np->name);
      }
      dev->dev.of_node = of_node_get(np);
      dev->dev.parent = parent ? &platform_bus;
      if (bus_id)
         dev_set_name(&dev->dev, "%s", bus_id);
      else
         of_device_make_bus_id(&dev->dev);
      return dev;
   }
   EXPORT_SYMBOL(of_device_alloc);
   ```
   - 如果 `device_node->_flags` 中的 `OF_POPULATED` 已经被置位，则表示已将对应的设备树节点关联到创建完成的 *platform_device*：不会对 `OF_POPULATED` 已经被置位的设备树节点创建 *platform_device*；创建完成的设备树节点，其 `OF_POPULATED` 被置位。
   - 将 *address* 和 *irq* 转换为 `struct resource` 进行管理。*platform_device* 的 `num_resources` 和 `resource` 字段为这两者的和。
   - 调用链 + 功能：
      ```c
      of_platform_populate()
         root = root ? of_node_get(root) : of_find_node_by_path("/");
         for_each_child_of_node(root, child)
            rc = of_platform_bus_create(bus:child, matches, lookup, parent, strick:true)
               if (strick && (!of_get_property(bus, "compatible", NULL)))
                  return 0;
               struct platform_device *dev = of_platform_device_create_pdata(np:bus, bus_id, platform_data, parent)
                  // 检查设备树节点 status 属性及 flags 情况
                  if (!of_device_is_availabe(np) || of_node_test_and_set_flag(np, OF_POPULATED))
                     return NULL;
                  dev = of_device_alloc(np, bus_id, parent)
                     dev = platform_device_alloc("", -1)
                     while (of_address_to_resource(np, num_reg, &temp_res) == 0)
                        num_reg++
                     num_irq = of_irq_count(np)
                     // 将地址和中断资源解析为 platform_device 的 resource 进行管理
                     if (num_reg || num_reg) {
                        res = kzalloc(sizeof(*res) * (num_reg + num_reg), GFP_KERNEL)
                        dev->num_resources = num_reg + num_irq;
                        dev->resource = res;
                        for (i = 0; i < num_reg; i++, res++)
                           rc = of_address_to_resource(np, i, res)
                        if (of_irq_to_resource_table(np, res, num_irq) != num_irq)
                     }
                     dev->dev.of_node = of_node_get(np)
                     dev->dev.parent = parent ? &platform_bus;
                     if (bus_id)
                        dev_set_name(&dev->dev, "%s", bus_id)
                     else
                        of_device_make_bus_id(&dev->dev)
                  dev->dev.bus = &platform_bus_type
                  dev->dev.platform_data = platform_data
                  of_dma_configure(&dev->dev, dev->dev.of_node)
                     if (!dev->coherent_dma_mask)
                        dev->coherent_dma_mask = DMA_BIT_MASK(32);
                     if (!dev->dma_mask)
                        dev->dma_mask = &dev->coherent_dma_mask;
                     // 物理地址到 DMA 地址的转换关系及范围大小，设备树中属性为 "dma-ranges"
                     // 详情见 Documentation/devicetree/bindings/iommu
                     ret = of_dma_get_range(np, &dma_addr, &paddr, &size);
                     // offset 根据 of_dma_get_range 结果计算
                     // offset 是物理地址与 DMA 地址间页的偏移量
                     dev->dma_pfn_offset = offset;
                     // 对 *dma_mask 取有效范围（设备树 dma-ranges 属性内容）
                     dev->coherent_dma_mask = min(dev->coherent_dma_mask, DMA_BIT_MASK(ilog2(dma_addr + size)))
                     *dma->dma_mask = min((*dev->dma_mask), DMA_BIT_MASK(ilog2(dma_addr + size)))
                     // 如果设备树节点或其父节点（可能有多个）具有
                     // dma-coherent 属性，则 coherent 为 true
                     coherent = of_dma_is_coherent(np)
                     // NULL...
                     iommu = of_iommu_configure(dev, np)
                     arch_setup_dma_ops(dev, dma_addr, size, iommu, coherent)
                  if (of_device_add(dev) != 0) {
                     // ...
                  }
                     // of_device_add():
                     dev->name = dev_name(&dev->dev)
                     dev->id = -1
                     if (!dev->dev.parent)
                        set_dev_node(&dev->dev, of_node_to_nid(dev->dev.of_node))
                     device_add(&dev->dev)
                  return dev;
               if (!dev || !of_match_node(matches, bus))
                  return 0;
               for_each_child_of_node(bus, child)
                  rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict)
               of_node_set_flag(bus, OF_POPULATED_BUS)
         of_node_set_flag(root, OF_POPULATED_BUS)
         of_node_put(root)
      ```
      - 涉及的设备树属性：
        - *compatible*
        - *status*
        - *reg* 或者 *assigned-addresses* + *reg-names*
        - *interrupts-extended* 或者 *interrupts* + *interrupt-names*
        - *dma-ranges* + *dma-coherent*
      - `arch_setup_dma_ops()`：
         ```c
         // version: 4.1.15
         // arch/arm/include/asm/dma-mapping.h
         #define arch_setup_dma_ops arch_setup_dma_ops
         extern void arch_setup_dma_ops(struct device *dev, u64 dma_base,
            u64 size, struct iommu_ops *iommu, bool coherent);
         
         // arch/arm/mm/dma-mapping.c
         void arch_setup_dma_ops(struct device *dev, u64 dma_base,
            u64 size, struct iommu_ops *iommu, bool coherent) {
            struct dma_map_ops *dma_ops;
            dev->archdata.dma_coherent = coherent;
            if (arm_setup_iommu_dma_ops(dev, dma_base, size, iommu))
               dma_ops = arm_get_iommu_dma_map_ops(coherent);
            else
               dma_ops = arm_get_dma_map_ops(coherent);
            set_dma_ops(dev, dma_ops);
         }

         // arch/arm/include/asm/dma-mapping.h
         static inline void set_dma_ops(struct device *dev, struct dma_map_ops *ops) {
            BUG_ON(!dev);
            dev->archdata.dma_ops = ops;
         }

         // arch/arm/mm/dma-mapping.c
         static bool arm_setup_iommu_dma_ops(struct device *dev, u64 dma_base,
            u64 size, struct iommu_ops *iommu) {
            return false;
         }

         static struct dma_map_ops *arm_get_dma_map_ops(bool coherent) {
            return coherent ? &arm_coherent_dma_ops : &arm_dma_ops;
         }

         struct dma_map_ops arm_coherent_dma_ops = {
            .alloc = arm_coherent_dma_alloc,
            .free  = arm_coherent_dma_free,
            .mmap  = arm_dma_mmap,
            .get_sgtable = arm_dma_get_sgtable,
            .map_page = arm_coherent_dma_map_page,
            .map_sg = arm_dma_map_sg,
            .set_dma_mask = arm_dma_set_mask,
         };
         EXPORT_SYMBOL(arm_coherent_dma_ops);

         struct dma_map_ops arm_dma_ops = {
            .alloc   = arm_dma_alloc,
            .free    = arm_dma_free,
            .mmap    = arm_dma_mmap,
            .get_sgtable   = arm_dma_get_sgtable,
            .map_page      = arm_dma_map_page,
            .unmap_page    = arm_dma_unmap_page,
            .map_sg        = arm_dma_map_sg,
            .unmap_sg      = arm_dma_ummap_sg,
            .sync_single_for_cpu = arm_dma_sync_single_for_cpu,
            .sync_signle_for_device    = arm_dma_sync_single_for_device,
            .sync_sg_for_cpu = arm_dma_sync_sg_for_cpu,
            .sync_sg_for_device  = arm_dma_sync_sg_for_device,
            .set_dma_mask = arm_dma_set_mask,
         };
         EXPORT_SYMBOL(arm_dma_ops);
         ```

3. 调用链
   ```c
   // arch/arm/kernel/setup.c
   customize_machine()
      machine_desc->init_machine()

   // arch/arm/mach-imx/mach-imx6ul.c
   imx6ul_init_machine()   
      of_platform_populate()
         of_platform_bus_create()
            of_platform_device_create_pdata()
               of_device_alloc()
               of_dma_configure()
               of_device_add()
   ```
   - `customize_machine()` 也被放置在特定的段中：`".initcall3.init"`。
   - 关于 `customize_machine()` 的调用，在 `init/main.c` 中有如下内容：
      ```c
      // version: 4.1.15
      // init/main.c
      extern initcall_t __initcall_start[];
      extern initcall_t __initcall0_start[];
      extern initcall_t __initcall1_start[];
      ...
      extern initcall_t __initcall_end[];

      static initcall_t *initcall_levels[] __initdata = {
         __initcall0_start,
         ...
         __initcall_end,
      };

      static void __init do_initcall_level(int level) {
         initcall_t *fn;
         strcpy(initcall_command_line, saved_command_line);
         parse_args(initcall_level_names[level], 
            initcall_command_line, __start__param,
            __stop_param - __start_param,
            level, level,
            &repair_env_string);
         for (fn = initcall_levels[level]; fn < initcall_levels[level + 1]; fn++)
            do_one_initcall(*fn);
      }

      static void __init do_initcalls(void) {
         int level;
         for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
            do_initcall_level(level);
      }
      ```
      `do_initcalls()` 的调用链：
      ```c
      start_kernel()
         /* Do the rest non-__init'ed, we're now alive */
         rest_init()
            kernel_thread(kernel_init, NULL, CLONE_FS)
      
      kernel_init()
         kernel_init_freeable()
            do_basic_setup()
               do_initcalls()
      ```
   - 而在链接脚本文件 `arch/arm/kernel/vmlinux.lds` 中定义了如下内容：
      ```script
      .init.data : {
      *(.init.data) *(.meminit.data) *(.init.rodata) *(.meminit.rodata) . = ALIGN(8); __clk_of_table = .; *(__clk_of_table) *(__clk_of_table_end) . = ALIGN(8); __reservedmem_of_table = .; *(__reservedmem_of_table) *(__reservedmem_of_table_end) . = ALIGN(8); __clksrc_of_table = .; *(__clksrc_of_table) *(__clksrc_of_table_end) . = ALIGN(8); __cpu_method_of_table = .; *(__cpu_method_of_table) *(__cpu_method_of_table_end) . = ALIGN(8); __cpuidle_method_of_table = .; *(__cpuidle_method_of_table) *(__cpuidle_method_of_table_end) . = ALIGN(32); __dtb_start = .; *(.dtb.init.rodata) __dtb_end = .; . = ALIGN(8); __irqchip_of_table = .; *(__irqchip_of_table) *(__irqchip_of_table_end)
      . = ALIGN(16); __setup_start = .; *(.init.setup) __setup_end = .;
      __initcall_start = .; *(.initcallearly.init) __initcall0_start = .; *(.initcall0.init) *(.initcall0s.init) __initcall1_start = .; *(.initcall1.init) *(.initcall1s.init) __initcall2_start = .; *(.initcall2.init) *(.initcall2s.init) __initcall3_start = .; *(.initcall3.init) *(.initcall3s.init) __initcall4_start = .; *(.initcall4.init) *(.initcall4s.init) __initcall5_start = .; *(.initcall5.init) *(.initcall5s.init) __initcallrootfs_start = .; *(.initcallrootfs.init) *(.initcallrootfss.init) __initcall6_start = .; *(.initcall6.init) *(.initcall6s.init) __initcall7_start = .; *(.initcall7.init) *(.initcall7s.init) __initcall_end = .;
      __con_initcall_start = .; *(.con_initcall.init) __con_initcall_end = .;
      __security_initcall_start = .; *(.security_initcall.init) __security_initcall_end = .;
      . = ALIGN(4); __initramfs_start = .; *(.init.ramfs) . = ALIGN(8); *(.init.ramfs.info)
      }
      ```
      - 也就是说，函数 `customize_machine()` 可以通过 `__initcall3_start` 来调用。
---
来自与 *kimi* 的问答：
请你介绍一下在 *Linux* 内核启动过程中，设备树 -> `struct device_node` -> `struct platform_device` 这个转换过程都是在哪里发生的。

这个过程发生在内核启动的两个不同阶段，涉及 架构相关代码 和 设备驱动模型（*device core*）的协作。下面按时间线梳理。

1. 总体流程
```
start_kernel()
    └── setup_arch()                    [架构早期初始化]
            └── unflatten_device_tree()   [DTB → device_node 树]
    ...
    └── do_initcalls()                  [驱动子系统初始化]
            └── of_platform_default_populate_init()
                    └── of_platform_default_populate()
                            └── of_platform_populate()
                                    └── of_platform_device_create()  [device_node → platform_device]
```

1. 第一阶段：*DTB* → `struct device_node`
- 发生时机：
   非常早，在 `setup_arch()` 中，此时内存管理子系统尚未完全初始化，但 `memblock` 已经可用。

- 关键函数与文件
   | 项目 | 位置 |
   |-----|-----|
   | 入口 | `arch/arm/kernel/setup.c`（或 `arch/arm64/kernel/setup.c`）的 `setup_arch()` |
   | 核心函数 | `unflatten_device_tree()` |
   | 实现文件 | `drivers/of/fdt.c`（解析扁平设备树）→ `drivers/of/base.c`（建立节点树）|

- 过程简述
  1. *Bootloader*（如 *U-Boot*）将 *DTB*（扁平化的二进制设备树）的物理地址传给内核
  2. `setup_arch()` 调用 `unflatten_device_tree()`
  3. 内核遍历 *DTB*，为每个节点分配内存（通过 `memblock`），创建 `struct device_node`
  4. 建立父子关系（`parent`、`child`、`sibling` 指针），形成内存中的树结构
  5. 同时解析 `compatible`、`reg`、`interrupts` 等属性，存入 `device_node->properties`

   此时，内核中已经有了一棵完整的设备树，但还没有任何 `platform_device`。

3. 第二阶段：`struct device_node` → `struct platform_device`
- 发生时机
   较晚，在 `do_initcalls()` 阶段（`arch_initcall_sync` 级别），此时内存分配器、中断子系统、时钟子系统等已基本就绪。

- 关键函数与文件
   | 项目 | 位置 |
   |-----|-----|
   | 入口 | `drivers/of/platform.c` 中的 `of_platform_default_populate_init()` |
   | 注册方式 | `arch_initcall_sync(of_platform_default_populate_init)` |
   | 核心函数 | `of_platform_default_populate()` → `of_platform_populate()` |
   | 创建设备 | `of_platform_device_create()` → `of_device_alloc()` + `platform_device_register()` |

- 过程简述
  1. `do_initcalls()` 按级别调用各初始化函数
  2. 执行到 `arch_initcall_sync` 时，调用 `of_platform_default_populate_init()`
  3. 该函数调用 `of_platform_default_populate(NULL, NULL, NULL)`
  4. 从 根节点（`/`）开始，遍历其 直接子节点
  5. 对每个带有 `compatible` 属性的子节点，调用 `of_platform_device_create()`
  6. 分配 `struct platform_device`，填入：
     - `dev.of_node` → 指向对应的 `device_node`
     - `dev.parent` → 通常为 `&platform_bus`
     - `resource` → 从 `reg` 属性解析出的 `MMIO/IORESOURCE_MEM`
     - `irq` → 从 `interrupts` 属性解析
  7. 调用 `platform_device_register()` 将其注册到 *platform bus*

- 重要补充：哪些节点会变成 *platform_device*？
   不是所有 *device_node* 都会直接变成 *platform_device*。

   | 节点位置 | 处理方式 | 原因 |
   |-----|-----|-----|
   | 根节点 `/` 的直接子节点 | 默认创建为 *platform_device* | 由 `of_platform_default_populate()` 处理 |
   | *Bus* 节点下的子节点（如 `/soc/spi@...` 下的 `flash@0`） | 不直接创建 | 等父 *bus driver*（如 `spi-imx.c`）*probe* 后，由 *bus* 代码（如 `spi.c`）扫描子节点并创建 *spi_device* |
   | *I2C* 总线下的设备 | 由 *i2c-core* 在 *adapter* 注册后扫描创建 *i2c_client* | 属于 *bus* 子系统职责 |
   | *PCI* 设备 | 由 *PCI* 枚举代码创建 *pci_dev* | 不依赖设备树或走 *OF* 路径 |

   总结：
    - 根级设备（如 `/soc`、`/memory`、`/chosen` 的直接子设备）→ 变成 *platform_device*
    - *Bus* 从设备（如 *SPI Flash*、*I2C EEPROM*）→ 由各自的 *bus core* 在运行时创建

4. 总结
   *DTB* → *device_node* 发生在 `setup_arch()` 早期的 `unflatten_device_tree()` 中，是架构代码对设备树的"解析"；*device_node* → *platform_device* 发生在 `do_initcalls()` 的 *arch_initcall_sync* 阶段，由 `drivers/of/platform.c` 中的 `of_platform_default_populate_init()` 完成，是设备模型对设备树的"实例化"。前者建立的是"描述"，后者建立的是"可操作的内核对象"。

---

### platform 驱动
#### 数据结构描述
```c
// version: 4.1.15
// include/linux/platform_device.h
struct platform_driver {
   int (*probe)(struct platform_device *);
   int (*remove)(struct platform_device *)
   void (*shutdown)(struct platform_device *);
   int (*suspend)(struct platform_device *, pm_message_t state);
   int (*resume)(struct platform_device *);
   struct device_driver driver;
   const struct platform_device_id *id_table;
   bool prevent_deferred_probe;
};
```

#### platform_driver 的注册
```c
// version: 4.1.15
// include/linux/platform_device.h
/* use a macro to avoid include chaining to get THIS_MODULE */
#define platform_driver_register(drv) \
   __platform_driver_register(drv, THIS_MODULE)
extern int __platform_driver_register(struct platform_driver *, struct module *);
extern void platform_driver_unregister(struct platform_driver *);

// drivers/base/platform.c
/**
 * __platform_driver_register - register a driver for platform-level devices 
 * @drv: platform driver structure
 * @owner: owning module/driver
 */
int __platform_driver_register(struct platform_driver *drv, struct module *owner) {
   drv->driver.owner = owner;
   drv->driver.bus   = &platform_bus_type;
   if (drv->probe)
      drv->driver.probe = platform_drv_probe;
   if (drv->remove)
      drv->driver.remove = platform_drv_remove;
   if (drv->shutdown)
      drv->driver.shutdown = platform_drv_shutdown;
   return driver_register(&drv->driver);
}
EXPORT_SYMBOL_GPL(__platform_driver_register);

static int platform_drv_probe(struct device *_dev) {
   struct platform_driver *drv = to_platform_driver(_dev->driver);
   struct platform_device *dev = to_platform_device(_dev);
   int ret;

   ret = of_clk_set_default(_dev->of_node, false);
   if (ret < 0)
      return ret;
   ret = dev_pm_domain_attach(_dev, true);
   if (ret != -EPROBE_DEFER) {
      ret = drv->probe(dev);
      if (ret)
         dev_pm_domain_detach(_dev, true);
   }
   if (drv->prevent_deferred_probe && ret == -EPROBE_DEFER) {
      dev_warn(_dev, "probe deferral not supported");
      ret = -ENXIO;
   }
   return ret;
}

static int platform_drv_remove(struct device *_dev) {
   struct platform_driver *drv = to_platform_driver(_dev->driver);
   struct platform_device *dev = to_platform_device(_dev);
   int ret;

   ret = drv->remove(dev);
   dev_pm_domain_detach(_dev, true);
   return ret;
}

static void platform_drv_shutdown(struct device *_dev) {
   struct platform_driver *drv = to_platform_driver(_dev->driver);
   struct platform_device *dev = to_platform_device(_dev);

   drv->shutdown(dev);
   dev_pm_domain_detach(_dev, true);
}
```

#### platform_driver 的注销
```c
// version: 4.1.15
// drivers/base/platform.c
/**
 * platform_drvier_unregister - unregister a driver for platform-level devices 
 * @drv: platform driver structure
 */
void platform_driver_unregister(struct platform_driver *drv) {
   driver_unregister(&drv->driver);
}
EXPORT_SYMBOL_GPL(platform_driver_unregister);
```

## 参考
1. 韦东山-《嵌入式 Linux 应用开发完全手册 V5.2_IMX6ULL_Pro 开发板》
2. 正点原子-《I.MX6U嵌入式Linux驱动开发指南V1.81》
