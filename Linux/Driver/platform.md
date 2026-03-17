# platform 设备驱动

## 总线-设备-驱动模型
引入 `platform_device` 和 `platform_driver`，**将“资源”与“驱动”*分离***，使得代码易于扩展。

资源的表示方式：
- `.c` 文件：冗余代码多，修改硬件配置时需要重新编译；
- `.dts` 文件：无冗余代码，修改硬件配置时只需修改设备树文件并重新编译后传给内核即可（无需重新编译内核/驱动）；

1. 总线 `Bus`
   ```c
   struct bus_type platform_bus_type = {
      .name       = "platform",
      .dev_groups = platform_dev_groups,
      .match      = platform_match,
      .uevent     = platform_uevent,
      .dma_configure = platform_dam_configure,
      .pm         = &platform_dev_pm_ops,
   };
   ```
   - `match`：判断（设备，驱动）是否匹配，如果匹配则调用**驱动的 `probe` 函数**；
2. 设备 `Device`
   ```c
   struct resource {
      resource_size_t start;
      resource_size_t end;
      const char *name;
      unsigned long flags;    // 指哪类资源
      struct resource *parent, *sibling, &child;
   };
   // flags:
   #define IORESOURCE_IO   ...
   // ...

   struct platform_device {
      const char *name;
      int id;
      bool id_auto;
      struct device dev;
      u32 num_resources;
      struct resource *resource;

      const struct platform_device_id *id_entry;
      char *driver_override;        // Driver name to force a match

      // MFD cell pointer
      struct mfd_cell *mfd_cell;

      // arch specific additions
      struct pdev_archdata archdata;
   };
   ```
   - `resource`：指定资源；
   - `driver_override`：只匹配这个指定的驱动；
   - 卸载 `platform_device`（`platform_device_unregister`）时会调用 `dev.release()`；
3. 驱动 `Driver`
   ```c
   struct platform_driver {
      int (*probe)(struct platform_device*);
      int (*remove)(struct platform_device*);
      void (*shutdown)(struct platform_device*);
      int (*suspend)(struct platform_device*, pm_message_t stat);
      int (*resume)(struct platform_device*);
      struct device_driver driver;
      const strut platform_device_id *id_table;
      bool prevent_deferred_probe;
   };
   ```
   - `probe`：设备与驱动匹配成功后被调用，一般在 `probe` 函数中完成：
     - 分配、设置、注册 `file_operations` 结构体；
     - 根据设备确定硬件；
     - 使用 `ioremap` 映射寄存器，操作寄存器；
     - ...；
   - `driver`：会用匹配过程中被调用；

## 设备与驱动匹配过程
1. 设备注册时
   ```c
   platform_device_register
   platform_device_add
      device_add
         bus_add_device // 放入链表
         bus_probe_device  // probe 枚举设备，即找到匹配的（dev, drv）
            device_initial_probe
               __device_attach
                  bus_for_each_drv(..., __device_attach_driver, ...)
                     _device_attach_driver
                        driver_match_device(drv, dev) // 是否匹配
                        driver_probe_device           // 调用 drv 的 probe
   ```
2. 驱动注册时
   ```c
   platform_driver_register
   __platform_driver_register
      driver_register
         bus_add_driver // 放入链表
            driver_attch(drv)
               bus_for_each_dev(drv->bus, NULL, drv, __driver_attach)
                  __driver_attach
                     driver_match_device(drv, dev) // 是否匹配
                     driver_probe_device           // 调用 drv 的 probe

   ```
3. 判断设备与驱动是否匹配的过程：
   ```c
   // drivers/base/platform.c
   static int platform_match(struct device *dev, struct device_driver *drv) {
      struct platform_device *pdev = to_platform_device(dev);
      struct platform_driver *pdrv = to_platform_driver(drv);

      // when driver_override is set, only bind to the matching driver
      if (pdev->driver_override)
         return !strcmp(pdev->driver_override, drv->name);

      // Attempt an OF style match first
      if (of_driver_match_device(dev, drv))
         return 1;

      // Then try ACPI(Advanced Configurable Power Interface) style match
      if (acpi_driver_match_device(dev, drv))
         return 1;
      
      // Then try to match against the id table
      if (pdrv->id_table)
         return platform_match_id(pdrv->id_table, pdev) != NULL;
      
      // fall-back to driver name match
      return (strcmp(pdev->name, drv->name) == 0);
   }
   ```
   比较顺序：
   1. `platform_device.driver_override` 和 `platform_driver.driver.name`；
   2. `OF style`，设备树对应的匹配方式：`platform_driver.driver.of_match_table`，`struct device_driver` 有个名为 `of_match_table` 的字段，对应着 `struct of_device_id` 类型的指针，可以用于表示支持的设备列表。`{.compatible = xxx}` 会和设备树中节点的 `compatible` 属性相比较，如果相同的话则实现设备和驱动的匹配；
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
   3. `ACPI` 方式，略；
   4. `platform_device.name` 和 `platform_driver.id_table[i].name`：`platform_device.id_table` 是 `struct platform_device_id` 类型的指针，表示该驱动支持若干个设备。`{.name = xxx, .driver_data=xxx, }` 中 `name` 字段表示该驱动支持的设备的名字，`driver_data` 字段是提供给该设备的私有数据；
   5. `platform_device.name` 和 `platform_driver.driver.name`：`platform_driver.id_table` 可能为空，这时可以根据 `platform_driver.driver.name` 来寻找同名的 `platform_device`；

## 设备树相关
### 设备树中节点转化为内核中的 `platform_device`
1. 会转化为 `platform_device`：
   - 根节点下含有 `compatible` 属性的子节点；
   - 含有特定 `compatible` 属性的**节点的子节点**：
     - 如果一个节点的 `compatible` 属性值为这 4 者之一：`"simple-bus", "simple-mfd", "isa", "arm,amba-bus"`；
     - 含有上述 `compabile` 属性值的节点的子节点、且子节点含有 `compatible` 属性可以转化；
2. 不会转化为 `platform_device`：
   - 总线 `I2C`、`SPI` 节点下的子节点不转化为 `plaform_device`（某个总线下的子节点，应该交给对应的总线驱动程序来处理，它们不应该被转化为 `platform_device`）；

### 设备树中的资源信息
- `platform_device` 中含有 `resource` 数组，它来自设备树节点 `device_node` 的 `reg`、`interrupts` 属性；
- `platform_device.dev.of_node` 指向 `device_node`，可以通过它获得其他属性；

### 设备树与 `platform_device` 相关函数
1. `of_find_device_by_node`：
   ```c
   extern struct platform_device *of_find_device_by_node(struct device_node *np);
   ```
   设备树中的每一个节点，在内核中都有一个 `device_node`，可以通过它去找到对应的 `platform_device`。

## API
1. `module_platform_driver()`
2. `EXPORT_SYMBOL(xxx)`：`a.c` 编译为 `a.ko`，里面定义了 `func_a`。如果它想让 `b.ko` 使用该函数，那么 `a.c` 里需要导出此函数（如果 `a.c`，`b.c` 都编进内核，则无需导出）；

### 获取资源
1. 返回该设备（`dev`）中类型（`type`）资源中的第几个（`num`）：
   ```c
   struct resource *platform_get_resource(struct platform_device *dev, unsigned int type, unsigned int num);
   ```
2. 返回该设备（`dev`）所用的第几个（`num`）中断：
   ```c
   int platform_get_irq(struct platform_device *dev, unsigned int num);
   ```
3. 通过名字（`name`）返回该设备（`dev`）的某类型（`type`）资源：
   ```c
   struct resource *platform_get_resource_byname(struct platform_device *dev, unsigned int type, const char *name);
   ```
4. 通过名字（`name`）返回该设备（`dev`）的中断号：
   ```c
   int platform_get_irq_byname(struct platform_device *dev, const char *name);
   ```

## 示例程序
1. 通过 `.c` 记录设备硬件信息（实现 `platform_devce`）：
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
   - 在入口函数中向内核注册 `platform_device`；
   - 注意：如果在 `platform_device` 结构体初始化时未提供 `dev.release`，则在调用 `platform_device_unregister` 时会出现警告。可以提供一个 `release` 函数，如果实在无事可做，把这个函数写为空；
   - 要根据依赖关系顺序加载各个模块；

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
2. 通过设备树 `.dts` 文件记录设备硬件信息：
   驱动要求设备树节点提供什么，就得按照要求去编写设备树。驱动程序中要根据 pin 属性来确定引脚，因此需要在设备树节点中添加 pin 属性：
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
   在驱动程序中，可以从 platform_device 中得到 device_node，再用 `of_property_read_u32` 得到属性的值。
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
3. 调试
   见 `sysfs.md` 相关内容。

## 参考
1. 韦东山-《嵌入式 Linux 应用开发完全手册 V5.2_IMX6ULL_Pro 开发板》