# Kernel Note
## Memory Allocation Guide
> [Memory Allocation Guide](https://www.kernel.org/doc/html/latest/core-api/memory-allocation.html#memory-allocation)

### GFP flags
`Get Free Page flags`

### GFP flags and reclaim behavior

### memory allocator

### Memory Management APIs
> [Memory Management APIs](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmalloc)

```c
// include/linux/device.h
static inline void *devm_kzalloc(struct device *dev, size_t size, gtp_t gfp) {
  return devm_kmalloc(dev, size, gfp | __GFP_ZERO);
}
```

## 内核空间和用户空间的数据交换
### 数据交换函数
```c
// arch/arm/include/asm/uaccess.h
static inline unsigned long __must_check copy_from_user(void *to, const void __user *from, unsigned long n) {
  if (access_ok(VERIFY_READ, from, n))
    n = __copy_from_user(to, from, n);
  else
    memset(to, 0, n);
  return n;
}

static inline unsigned long __must_check copy_to_user(void __user *to, const void *from, unsigned long n) {
  if (access_ok(VERIFY_WRITE, to, n))
    n = __copy_to_user(to, from, n);
  return n;
}
```
- 返回值情况（来自 AI 问答）：
`copy_from_user` 和 `copy_to_user` 的返回值设计遵循 Linux 内核的“剩余未拷贝字节数”约定：
1. 返回值定义
| 函数                                                                   | 成功    | 失败                 |
| -------------------------------------------------------------------- | ----- | ------------------ |
| `copy_from_user(void *to, const void __user *from, unsigned long n)` | **0** | **剩余未拷贝的字节数**（> 0） |
| `copy_to_user(void __user *to, const void *from, unsigned long n)`   | **0** | **剩余未拷贝的字节数**（> 0） |

2. 用法举例：
```c
// 驱动中常见的读写接口
static ssize_t my_read(struct file *filp, char __user *buf,
                       size_t count, loff_t *offp)
{
    char kbuf[256];
    
    // 从内核拷贝到用户空间
    if (copy_to_user(buf, kbuf, count))
        return -EFAULT;        // 失败：返回错误码
    
    return count;              // 成功：返回拷贝字节数
}

static ssize_t my_write(struct file *filp, const char __user *buf,
                        size_t count, loff_t *offp)
{
    char kbuf[256];
    
    // 从用户空间拷贝到内核
    if (copy_from_user(kbuf, buf, count))
        return -EFAULT;        // 失败
    
    // 处理 kbuf...
    
    return count;
}
```

- `access_ok` 的功能（来自 AI 问答）
`access_ok` 用于检查用户空间地址范围是否在当前进程的可访问地址空间内，但它不保证地址已映射或可读/写。
核心作用：
  | 检查项                   | 说明                             |
  | --------------------- | ------------------------------ |
  | **地址范围是否属于用户空间**      | 排除内核空间地址（`> TASK_SIZE`）        |
  | **地址范围是否在当前进程的内存映射内** | 粗略检查，非精确页表遍历                   |
  | **读写权限方向**            | `VERIFY_READ` / `VERIFY_WRITE` |

### 使用数据交换函数的原因


## io 相关
### io 内存和 io 端口
> 正点原子-linux 驱动开发指南，41 章：

I/O 端口和 I/O 内存。当外部寄存器或内存映射到 IO 空间时，称为 I/O 端口。当外部寄存器或内存映射到内存空间时，称为 I/O 内存。对于 ARM 来说，没有 IO 空间这个概念，因此 ARM 体系下只有 I/O 内存（可以直接理解为内存）。

### 物理地址到虚拟地址的转换
- Linux 下的任何外设驱动，最终都是要配置相应的硬件寄存器；
- 在老版本的 Linux 中要求处理器必须要有 MMU，但是现在 Linux 内核已经支持无 MMU 的处理器了，MMU 主要完成的功能如下：
  - 完成虚拟空间到物理空间的映射；
  - 内存保护，设置存储器的访问权限，设置虚拟存储空间的缓冲特性；
- Linux 内核启动的时候会初始化 MMU，设置好内存映射，**设置好以后 CPU 访问的都是虚拟地址**。需要通过 `ioremap` 和 `iounremap` 实现物理内存到虚拟内存之间的转换。通过 `ioremap` 实现物理地址到虚拟地址的转换，`iounremap` 释放这层映射关系。

1. `ioremap`
```c
// arch/arm/include/asm/io.h
#define ioremap(cookie, size) __arm_ioremap((cookie), (size), MT_DEVICE)

void __iomem *__arm_ioremap(phys_addr_t phys_addr, size_t size, unsigned int mtype) {
  return arch_ioremap_caller(phys_addr, size, mtype, __builtin_return_address(0));
}
```
   - 功能：从实际物理地址映射到进程中的逻辑地址
   - 实际上 `ioremap` 是**按页**（4K, 4096字节）进行映射的，即映射内存单位为页。eg: 若 `phys_addr = 0x10002, size = 4`, 则 `ioremap` 的内部实现是：
     1. `phys_addr` 按页取整（4096,0x1000），得到地址 0x10000；
     2. `size` 按页取整，得到 4096；
     3. 把起始地址 0x10000，大小为 4096 的这一块物理地址空间，映射到虚拟地址空间；
     4. 假设得到的虚拟空间起始地址为 0xf0010000，则 `phys_addr = 0x10002` 对应的 `virt_add = 0xf0010002`；

2. `iounmap`
  不再使用已映射的虚拟地址时，要 `iounmap(virt_addr)`。
  ```c
  void iounmap(volatile void __iomem *addr);
  
  // 内核源码
  #define iounmap				__arm_iounmap
  void __arm_iounmap(volatile void __iomem *io_addr)
  {
      arch_iounmap(io_addr);
  }
  ```

### io 内存访问函数
Linux 建议使用这组函数来操作通过 `ioremap` 映射得到的虚拟地址：
1. 读操作函数
   ```c
   u8 readb(const volatile void __iomem *addr);
   u16 readw(const volatile void __iomem *addr);
   u32 readl(const volatile void __iomem *addr);
   ```
2. 写操作函数
   ```c
   void writeb(u8 value, volatile void __iomem *addr);
   void writew(u16 value, volatile void __iomem *addr);
   void writel(u32 value, volatile void __iomem *addr);
   ```

## 位处理
### bitmap
1. `bitmap_set(unsigned long *map, unsigned int start, int len)`: `lib/bitmap.c`

### bitops
1. `set_bit(nr, p)`: `arch/arm/include/asm/bitops.h`

## devres-Managed Device Resource
来自 AI 对话。
### 参考
- [内核文档-Managed Device Resource](https://www.kernel.org/doc/html/latest/driver-api/driver-model/devres.html)
- 内核源码文档：`Documentation/driver-model/devres.txt`
- 文档内容结构
  ```txt
    devres 官方文档包含：
  ├── 1. Intro（介绍）
  ├── 2. Devres（核心机制）
  ├── 3. Devres Group（资源分组）
  ├── 4. Details（生命周期规则）
  ├── 5. Overhead（性能开销分析）
  └── 6. List of managed interfaces（完整 API 列表）
      ├── CLOCK
      ├── DMA
      ├── GPIO
      ├── IIO
      ├── INPUT
      ├── IO region
      ├── IOMAP
      ├── IRQ
      ├── LED
      ├── MDIO
      ├── MEM
      ├── MFD
      ├── MUX
      ├── PER-CPU MEM
      ├── PCI
      ├── PHY
      ├── PINCTRL
      ├── POWER
      ├── PWM
      ├── REGULATOR
      ├── RESET
      ├── RTC
      ├── SERDEV
      ├── SLAVE DMA ENGINE
      ├── SPI
      └── WATCHDOG
  ```
### 核心实现
1. devres 核心机制（统一实现）
  | 文件           | 路径                       | 内容                                  |
  | ------------ | ------------------------ | ----------------------------------- |
  | **devres.c** | `drivers/base/devres.c`  | devres 核心链表管理、分配、释放                 |
  | **device.h** | `include/linux/device.h` | `struct device` 中的 `devres_head` 定义 |
2. `drivers/base/devres.c` 中的核心函数：
  ```c
  // 资源分配与注册
  void *devres_alloc(dr_release_t release, size_t size, gfp_t gfp);
  void devres_add(struct device *dev, void *res);
  void devres_free(void *res);

  // 资源查找与释放
  void *devres_find(struct device *dev, dr_release_t release,
                    dr_match_t match, void *match_data);
  int devres_destroy(struct device *dev, dr_release_t release,
                    dr_match_t match, void *match_data);
  int devres_release(struct device *dev, dr_release_t release,
                    dr_match_t match, void *match_data);

  // 分组管理
  void *devres_open_group(struct device *dev, void *id, gfp_t gfp);
  void devres_close_group(struct device *dev, void *id);
  int devres_release_group(struct device *dev, void *id);
  void devres_remove_group(struct device *dev, void *id);

  // 全部释放（驱动 detach 时调用）
  int devres_release_all(struct device *dev);
  ```
3. 各子系统 `devm_*` 接口的分布：
  `devm_*` 接口不是集中在一个文件内，而是根据功能分散在各个子系统中，每个子系统实现自己的 managed 版本。
  | 功能类别                  | 实现文件                            | 主要接口                                                                   |
  | --------------------- | ------------------------------- | ---------------------------------------------------------------------- |
  | **内存 (MEM)**          | `drivers/base/devres.c`         | `devm_kmalloc()`, `devm_kzalloc()`, `devm_kfree()`, `devm_kstrdup()`   |
  | **中断 (IRQ)**          | `kernel/irq/devres.c`           | `devm_request_irq()`, `devm_request_threaded_irq()`, `devm_free_irq()` |
  | **IO 内存映射 (IOMAP)**   | `lib/devres.c`                  | `devm_ioremap()`, `devm_iounmap()`, `devm_ioremap_resource()`          |
  | **IO 区域 (IO region)** | `lib/devres.c`                  | `devm_request_mem_region()`, `devm_release_mem_region()`               |
  | **DMA**               | `kernel/dma/mapping.c`          | `dmam_alloc_coherent()`, `dmam_free_coherent()`                        |
  | **GPIO**              | `drivers/gpio/gpiolib-devres.c` | `devm_gpiod_get()`, `devm_gpio_request()`                              |
  | **时钟 (CLOCK)**        | `drivers/clk/clk-devres.c`      | `devm_clk_get()`, `devm_clk_put()`                                     |
  | **PINCTRL**           | `drivers/pinctrl/devm.c`        | `devm_pinctrl_get()`, `devm_pinctrl_register()`                        |
  | **PCI**               | `drivers/pci/pci.c`             | `pcim_enable_device()`, `devm_pci_remap_cfgspace()`                    |
  | **PHY**               | `drivers/phy/phy-core.c`        | `devm_usb_get_phy()`                                                   |
  | **REGULATOR**         | `drivers/regulator/devres.c`    | `devm_regulator_get()`                                                 |
  | **PWM**               | `drivers/pwm/core.c`            | `devm_pwm_get()`                                                       |
  | **SPI**               | `drivers/spi/spi.c`             | `devm_spi_alloc_host()`, `devm_spi_register_controller()`              |
### 实现模式
实现模式：统一的 devres 核心 + 分散的 release 函数
1. 核心机制（统一）
  ```c
  // drivers/base/devres.c
  // 所有 devm_* 共享的底层机制

  struct devres {
      struct devres_node node;      // 链表节点
      // 后面紧跟用户数据（柔性数组）
  };

  // 注册资源到设备
  void devres_add(struct device *dev, void *res)
  {
      struct devres *dr = addr_to_dr(res);
      unsigned long flags;
      
      spin_lock_irqsave(&dev->devres_lock, flags);
      add_dr(dev, &dr->node);
      spin_unlock_irqrestore(&dev->devres_lock, flags);
  }
  ```
2. 各个子系统的 release 函数（分散）
  ```c
  // kernel/irq/devres.c 示例
  struct irq_devres {
      unsigned int irq;
      void *dev_id;
  };

  static void devm_irq_release(struct device *dev, void *res)
  {
      struct irq_devres *this = res;
      free_irq(this->irq, this->dev_id);  // 子系统特定的释放逻辑
  }

  int devm_request_threaded_irq(struct device *dev, unsigned int irq,
                                  irq_handler_t handler,
                                  irq_handler_t thread_fn,
                                  unsigned long irqflags,
                                  const char *devname, void *dev_id)
  {
      struct irq_devres *dr;
      int rc;
      
      dr = devres_alloc(devm_irq_release, sizeof(*dr), GFP_KERNEL);
      if (!dr)
          return -ENOMEM;
      
      dr->irq = irq;
      dr->dev_id = dev_id;
      
      rc = request_threaded_irq(irq, handler, thread_fn, irqflags, devname, dev_id);
      if (rc) {
          devres_free(dr);
          return rc;
      }
      
      devres_add(dev, dr);  // 注册到 devres 链表
      return 0;
  }
  ```
### 数据结构关联
```txt
struct device (include/linux/device.h)
├── struct list_head devres_head;    ← 管理的资源链表头
│       │
│       ▼
│   ┌─────────────────┐
│   │ struct devres_node│
│   │   struct list_head entry    ← 链表指针
│   │   dr_release_t release       ← 释放函数指针
│   │   void *data                 ← 指向 struct devres
│   └─────────────────┘
│       │
│       ▼
│   ┌─────────────────┐
│   │ struct devres    │
│   │   struct devres_node node    ← 内嵌节点
│   │   u8 __data[0]               ← 柔性数组：实际资源数据
│   │                               (如 irq_devres、dma_devres 等)
│   └─────────────────┘
│
└── ... 其他字段
```
### 与 C++ RAII
Resource-managed（devres）的核心机制就是将资源生命周期与设备生命周期绑定，实现类似 C++ RAII 的自动资源管理。
1. 与 RAII 的类比
  | 特性       | C++ RAII                 | Linux devres                     |
  | -------- | ------------------------ | -------------------------------- |
  | **核心思想** | 资源获取即初始化，对象销毁时自动释放       | 资源注册即绑定，设备卸载时自动释放                |
  | **绑定对象** | 对象（class 实例）             | `struct device`                  |
  | **触发释放** | 析构函数 `~Class()`          | 设备 `remove()` 或 `probe()` 失败时的回滚 |
  | **异常安全** | 栈展开时自动调用析构               | `probe()` 中途返回错误时自动释放已注册资源       |
  | **手动释放** | 通常不需要（除非 `new`/`delete`） | 通常不需要（`devm_*` 系列）               |
2. devres 的两种核心场景
- `probe` 成功，设备正常卸载
  ```txt
  设备注册
      │
      ▼
  probe() 开始
      │
      ├── devm_kzalloc() ──► 内存分配，自动注册到 devres 链表
      ├── devm_request_irq() ──► 中断注册
      ├── devm_ioremap() ──► IO 内存映射
      ├── devm_clk_get() ──► 时钟获取
      │
      ▼
  probe() 成功返回 0
      │
      设备运行期...
      │
      ▼
  用户/系统触发卸载
      │
      ▼
  remove() 被调用
      │
      ▼
  设备核心层自动遍历 devres 链表
      │
      ├── 调用内存 release ──► kfree()
      ├── 调用中断 release ──► free_irq()
      ├── 调用 IO release ──► iounmap()
      ├── 调用时钟 release ──► clk_put()
      │
      ▼
  设备结构体释放
  ```
- `probe` 中途失败（devres 关键价值）
  ```c
  static int my_probe(struct platform_device *pdev)
  {
      void *buf;
      int irq;
      void __iomem *base;
      
      buf = devm_kzalloc(&pdev->dev, 1024, GFP_KERNEL);
      if (!buf)
          return -ENOMEM;           // 无资源需要释放（还没注册）
      
      irq = devm_request_irq(&pdev->dev, 10, handler, 0, "mydev", NULL);
      if (irq < 0)
          return irq;               // ⭐ 自动释放 buf！
      
      base = devm_ioremap(&pdev->dev, 0x10000000, 0x1000);
      if (!base)
          return -ENOMEM;           // ⭐ 自动释放 buf + irq！
      
      // ... 更多初始化
      
      // 如果这里返回错误：
      return -EINVAL;               // ⭐ 自动释放 buf + irq + base！
  }

  // 对比非 managed 代码（噩梦）
  static int my_probe_legacy(struct platform_device *pdev)
  {
      void *buf = kmalloc(1024, GFP_KERNEL);
      int irq = request_irq(10, handler, 0, "mydev", NULL);
      void __iomem *base = ioremap(0x10000000, 0x1000);
      int ret;
      
      // 每个错误路径都要手动释放前面成功的资源
      if (!buf) {
          return -ENOMEM;
      }
      if (irq < 0) {
          kfree(buf);           // 别忘了！
          return irq;
      }
      if (!base) {
          free_irq(irq, NULL);  // 别忘了！
          kfree(buf);           // 别忘了！
          return -ENOMEM;
      }
      
      // ... 更多初始化，更多错误路径，更多手动释放 ...
      
      return 0;
  }

  static int my_remove_legacy(struct platform_device *pdev)
  {
      // remove 中还要再写一遍释放逻辑
      iounmap(base);
      free_irq(irq, NULL);
      kfree(buf);
      return 0;
  }
  ```
3. devres 的分组机制（group）
更强大的功能：probe 中的阶段性回滚。
  ```c
  static int complex_probe(struct platform_device *pdev)
  {
      void *id;
      
      // 开启一个资源组
      id = devres_open_group(&pdev->dev, NULL, GFP_KERNEL);
      if (!id)
          return -ENOMEM;
      
      // 阶段 1：基础资源
      devm_kzalloc(&pdev->dev, 100, GFP_KERNEL);
      devm_request_irq(&pdev->dev, ...);
      
      // 阶段 1 验证
      if (some_check_failed()) {
          devres_release_group(&pdev->dev, id);   // ⭐ 只释放阶段 1 的资源
          return -EINVAL;
      }
      
      devres_close_group(&pdev->dev, id);          // 阶段 1 完成，锁定
      
      // 阶段 2：高级资源（如果阶段 1 失败，这些不会注册）
      devm_ioremap_resource(&pdev->dev, ...);
      devm_clk_get(&pdev->dev, ...);
      
      return 0;
  }
  ```
4. 关键实现：设备卸载时的自动释放
  ```c
  // drivers/base/dd.c
  static void __device_release_driver(struct device *dev)
  {
      struct device_driver *drv;
      
      drv = dev->driver;
      if (drv) {
          // 调用驱动的 remove()
          if (drv->remove)
              drv->remove(dev);
          
          // ⭐ 关键：自动释放所有 devres 资源
          devres_release_all(dev);
          
          // 断开设备与驱动的绑定
          dev->driver = NULL;
      }
  }

  // drivers/base/devres.c
  int devres_release_all(struct device *dev)
  {
      struct devres_node *node;
      struct devres *dr;
      unsigned long flags;
      
      // 遍历 devres 链表，逐个调用 release 函数
      spin_lock_irqsave(&dev->devres_lock, flags);
      while (!list_empty(&dev->devres_head)) {
          node = list_entry(dev->devres_head.next, struct devres_node, entry);
          dr = container_of(node, struct devres, node);
          
          list_del_init(&node->entry);    // 从链表移除
          spin_unlock_irqrestore(&dev->devres_lock, flags);
          
          node->release(dev, dr->data);    // ⭐ 调用注册的释放函数
          
          spin_lock_irqsave(&dev->devres_lock, flags);
      }
      spin_unlock_irqrestore(&dev->devres_lock, flags);
      
      return 0;
  }
  ```
5. devres 与手动管理的对比
  | 场景             | 非 managed（手动）        | managed（devres）    |
  | -------------- | -------------------- | ------------------ |
  | **probe 成功**   | `remove()` 中手动释放所有资源 | 自动释放，无需 `remove()` |
  | **probe 中途失败** | 每个错误路径手动回滚           | 自动回滚所有已注册资源        |
  | **remove 遗漏**  | 资源泄漏                 | 不可能遗漏              |
  | **代码复杂度**      | 高（大量错误处理代码）          | 低（专注业务逻辑）          |
  | **可读性**        | 差（释放逻辑分散）            | 好（释放逻辑内聚）          |
6. 总结
  devres 就是 Linux 内核中的 RAII 机制。它将资源（内存、中断、IO、时钟等）的生命周期绑定到 struct device 上，设备卸载或 probe 失败时自动释放所有已注册资源，彻底消除手动释放的遗漏风险，让驱动开发者专注于业务逻辑而非繁琐的错误处理。



## log
### `printk`
- 和 `printf` 的区别：
  - `printk` 运行在内核态，`printf` 运行在用户态；
  - `printk` 可以设置消息级别，从而控制可以显示的消息。
- 消息级别
  ```c
  // 一共八个消息级别
  // include/linux/kern_levels.h
  #define KERN_SOH        "\001"
  #define KERN_EMERG      KERN_SOH "0"    // 紧急事件，一般是内核崩溃
  #define KERN_ALERT      KERN_SOH "1"    // 必须立即采取行动
  #define KERN_CRIT       KERN_SOH "2"    // 临界条件，比如严重的软件或硬件错误
  #define KERN_ERR        KERN_SOH "3"    // 错误状态，一般设备驱动程序中使用 KERN_ERR 报告硬件错误
  #define KERN_WARNING    KERN_SOH "4"    // 警告信息，不会对系统造成严重影响
  #define KERN_NOTICE     KERN_SOH "5"    // 有必要进行提示的一些信息
  #define KERN_INFO       KERN_SOH "6"    // 提示性的信息
  #define KERN_DEBUG      KERN_SOH "7"    // 调试信息

  // 示例：
  printk(KERN_EMERG "gsmi: Log Shutdown Reason\n");
  printk("gsmi: Log Shutdown Reason\n");
  ```
  - 日志级别有：0-紧急，1-警报，2-严重，3-错误，4-警告，5-通知，6-信息，7-调试，**数字越小，表示消息越重要**；
  - 当不显式设置消息级别，那么 `printk` 将会采用默认级别； `MESSAGE_LOGLEVEL_DEFAULT`，`MESSAGE_LOGLEVEL_DEFAULT` 默认为 4；
  - 在 `include/linux/printk.h` 中，宏 `CONSOLE_LOGLEVEL_DEFAULT` 控制着哪些级别的消息可以显示在控制台上，默认为 7；
  - 当设置 `/proc/sys/kernel/printk` 的值为某个级别时，**只有该级别及更高级别的消息（数字更小的消息）才会输出到控制台**。

- 设置内核日志级别。内核日志级别决定了哪些信息会被输出到控制台，可以通过修改 `/proc/sys/kernel/printk` 文件或使用 `sysctl` 命令来设置日志级别。
  ```bash
  cat /proc/sys/kernel/printk         # 查看当前日志级别
  echo 7 > /proc/sys/kernel/printk    # 设置日志级别，7 表示所有级别的日志都会输出到控制台
  # 使用 sysctl 命令
  sysctl kernel.printk=8      # =8 还是 =7?
  ```

- 查看通过 `printk` 打印的内核日志信息：
  - 通过 `dmesg` 命令
    ```bash
    dmesg                   # 基本用法，用于显示或控制内核环形缓冲区的内容，即内核日志
    dmesg | tail            # 显示最新的日志
    dmesg | grep my_module  # 显示与 my_module 相关的日志信息
    # 实时查看内核日志的变化
    watch "dmesg | tail -n 50"
    dmesg -w
    ```
  - 查看 `/var/log/syslog` 文件
    ```bash
    sudo cat /var/log/syslog        # 查看完整日志
    sudo tail -f /val/log/syslog    # 实时查看日志
    ```  
  - 使用 `journalctl` 命令
    ```bash
    journalctl          # 查看所有日志
    journalctl -b       # 查看内核日志
    journalctl -f       # 实时查看日志
    ```

### pr
> 1. [Linux kernel printk 以及 pr_err, dev_err 等包装函数](https://cctags.github.io/posts/2021/12/05/linux-kernel-printk-alias-function.html)
> 2. Documentation/process/coding-style.rst
> 3. Documentation/core-api/printk-basics.rst

1. `pr_err` 和 `pr_debug` 等：
  ```c
  // /include/linux/printk.h
  
  /**
   * These can be used to print at the various log levels. 
   * All of these will print unconditionally, although note that
   * pr_deb() and other debug macros are compiled out unless either
   * DEBUG is defined or CONFIG_DYNAMIC_DEBUG is set.
   */
  #define pr_emerg(fmt, ...) \
    printk(KERN_EMERG pr_fmt(fmt), ##__VA_ARGS__)
  #define pr_alert(fmt, ...) \
    printk(KERN_ALERT pr_fmt(fmt), ##__VA_ARGS__)
  #define pr_crit(fmt, ...) \
    printk(KERN_CRIT pr_fmt(fmt), ##__VA_ARGS__)
  #define pr_err(fmt, ...) \
    printk(KERN_ERR pr_fmt(fmt), ##__VA_ARGS__)
  #define pr_warning(fmt, ...) \
    printk(KERN_WARNING pr_fmt(fmt), ##__VA_ARGS__)
  #define pr_warn pr_warning
  #define pr_notice(fmt, ...) \
    printk(KERN_NOTICE pr_fmt(fmt), ##__VA_ARGS__)
  #define pr_info(fmt, ...) \
    printk(KERN_INFO pr_fmt(fmt), ##__VA_ARGS__)

  /**
   * Like KERN_CONT, pr_cont should only be used when continuing
   * a line with no newline ('\n') enclosed. Otherwise it defaults
   * back to KERN_DEFAULT.
   */
  #define pr_cont(fmt, ...) \
    printk(KERN_CONT fmt, ##__VA_ARGS__)
  
  /* pr_devel() should produce zero code unless DEBUG is defined */
  #ifdef DEBUG
  #define pr_devel(fmt, ...) \
    printk(KERN_DEBUG pr_fmt(fmt), ##__VA_ARGS__)
  #else
  #define pr_devel(fmt, ...) \
    no_printk(KERN_DEBUG pr_fmt(fmt), ##__VA_ARGS__)
  #endif

  #include <linux/dynamic_debug.h>

  /* If you are writing a driver, please use dev_dbg instead */
  #if defined(CONFIG_DYNAMIC_DEBUG)
  /* dynamic_pr_debug() uses pr_fmt() internally so we 
   * don't need  it  here
   */
  #define pr_debug(fmt, ...) \
    dynamic_pr_debug(fmt, ##__VA_ARGS__)
  #elif defined(DEBUG)
  #define pr_debug(fmt, ...) \
    printk(KERN_DEBUG pr_fmt(fmt), ##__VA_ARGS__)
  #else
  #define pr_debug(fmt, ...) \
    no_printk(KERN_DEBUG pr_fmt(fmt), ##__VA_ARGS__)
  #endif
  ```
2. `no_printk`
  ```c
  /* 
   * Dummy printk for disable debugging statememts to use whilst 
   * maintaining gcc's format and side-effect checking.
   */
  static inline __printf(1, 2)
  int no_printk(const char *fmt, ...) {
    return 0;
  }
  ```
3. `pr_fmt`
   > 1.
  ```txt
  * This marco can be used to generate a unified format string for
  * pr_*() macros. A common use is to prefix all pr_*() meesages
  * in a file with a common string. For example, defining this at
  * the top of a source file:
  *     #define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
  * would prefix all pr_info, pr_emerg ... messages in the file
  * with the module name.
  ```
  - 内核好多文件里，在文件的开头位置，一般都有 `#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt` 这样的定义。
  - 比如，查看内核启动日志：
    ```bash
    smp: Bringing up secondary CPUs ...
    ```
    它对应的代码是：
    ```c
    // kernel/smp.c

    #define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
    ...

    pr_info("Bringing up secondary CPUs ...\n");
    ```
    所以日志里的 "smp: " 前缀是由 `pr_fmt` 定义的，这样可以进一步简化并统一内核模块里的打印。


### dev
> 1. [Linux kernel printk 以及 pr_err, dev_err 等包装函数](https://cctags.github.io/posts/2021/12/05/linux-kernel-printk-alias-function.html)
> 2. [linux kernel debug printk](https://www.cnblogs.com/WangYangkai/p/14898693.html)

```txt
Linux kernel coding style - Printing kernel messages:

There are a number of driver model diagnostic macros in <linux/device.h>
which you should use to make sure mesages are matched to the
right device and driver, and are tagged with the right level: dev_err(), dev_warn(), dev_info(), and so forth. For messages
that aren't associated with a particular device, <linux/printk.h>
defines pr_notice(), pr_info(), pr_warn, pr_err(), etc.
```

> 对于设备驱动实现代码，建议使用 dev_err(), dev_warn(), dev_info() 
> 等等打印函数，它们会在 printk 的基础上，加上当前模块相关的信息，使得打印更加清晰和通用。

1. `dev_info`, `dev_err`, `dev_warn` 等等：
  ```c
  // drivers/base/core.c

  #define define_dev_printk_level(func, kern_levl) \
  void func(const struct device *dev, const char *fmt, ...) { \
    struct va_format vaf; \
    va_list args; \
    \
    va_start(args, fmt); \
    \
    vaf.fmt = fmt; \
    vaf.va  = &args; \
    \
    __dev_printk(kern_level, dev, &vaf); \
    \
    va_end(args);
  }
  EXPORT_SYMBOL(func);

  define_dev_printk_level(dev_emerg, KERN_EMERG);
  define_dev_printk_level(dev_alert, KERN_ALERT);
  define_dev_printk_level(dev_crit, KERN_CRIT);
  define_dev_printk_level(dev_err, KERN_ERR);
  define_dev_printk_level(dev_warn, KERN_WARNING);
  define_dev_printk_level(dev_notice, KERN_NOTICE);
  define_dev_printk_level(_dev_info, KERN_INFO);

  // include/linux/device.h
  /*
   * Stupid hackaround for existing uses of non-printk uses 
   * dev_info
   * 
   * Note that the definition of dev_info below is actually
   * _dev_info and a macro is used to avoid redefining dev_info
   */
  #define dev_info(dev, fmt, arg...) __dev_info(dev, fmt, ##arg)
  ```
2. `dev_dbg`
  > 1. [Linux 内核中 dev_info, dev_dbg, dev_err 及动态调试](https://cloud.tencent.com/developer/article/1819284)

  ```c
  // include/linux/device.h

  #if defined(CONFIG_DYNAMIC_DEBUG)
  // 使能 CONFIG_DYNAMIC_DEBUG 后则根据 control 信息动态打印
  #define dev_dbg(dev, format, ...) \
  do { \
    dynamic_dev_dbg(dev, format, ##__VA_ARGS__); \
  } while (0)
  #elif defined(DEBUG)
  // 使能 DEBUG，则打印整个 kernel 的 dev_dbg 信息
  #define dev_dbg(dev, format, arg...) \
    dev_printk(KERN_DEBUG, dev, format, ##arg)
  #else
  // 都不使能，dev_dbg 不打印任何东西
  #define dev_dbg(dev, format, arg...) \
  ({  \
    if (0)  \
      dev_printk(KERN_DEBUG, dev, format, ##arg); \
  })
  #endif
  ```
  - [1] 目前在 kernel 驱动代码中，都不再建议直接使用 printk 添加打印信息，而是使用 dev_info，dev_dbg，dev_err 之类的函数代替，虽然这些 dev_xxx 函数的本质还是使用 printk 打印的，但是相比起 printk：
    - 支持打印模块信息、dev 信息；
    - 支持动态调试（dynamic debug）方式。
3. 动态调试使用方法 [1]
   - 打开内核动态调试开关：在 make menuconfig 中选中 CONFIG_DYNAMIC_DEBUG 以及 CONFIG_DEBUG_FS；
   - Linux 启动后，使用命令行挂载上 dbgfs：
      ```bash
      mkdir /mnt/dbg
      mount -t debugfs none /mnt/dbg
      ```
   - 通过如下方式控制输出 dev_dbg() 信息：
     - 控制某个文件所有 deb_dbg()：
        ```bash
        # 开启
        echo -n "file xxx.c +p" > /mnt/dbg/dynamic_debug/control
        # 关闭
        echo -n "file xxx.c -p" > /mnt/dbg/dynamic_debug/control
        # 示例
        echo -n "file ca_dsc_core.c +p" > /mnt/dbg/dynamic_debug/control # 打印 ca_dsc_core.c 所有的 dev_dbg() 信息
        ```
     - 控制某个函数所有 dev_dbg()：
        ```bash
        # 开启
        echo -n "func xxx +p" > /mnt/dbg/dynamic_debug/control
        # 关闭
        echo -n "func xxx -p" > /mnt/dbg/dynamic_debug/control
        # 示例
        echo -n "func ca_dsc_read +p" > /mnt/dbg/dynamic_debug/control # 打印 ca_dsc_read() 函数所有 dev_dbg() 信息
        ```
   - 运行程序，使用 `dmesg` 则可以看到相应 dev_dbg() 的输出信息。
   - 动态打印调试的基本原理：当编译选项 `CONFIG_DYNAMIC_DEBUG` 打开的时候，在编译阶段，kernel 会把所有使用 dev_dbg() 的信息记录在一个 table 中，这些信息可以从 `/mnt/dbg/dynamic_debug/control` 查到：
      ```bash
      cat /mnt/dbg/dynamic_debug/control
      ...
      drivers/alidrivers/modules/alidsc/ca_dsc_core.c:800 [alidsc]ca_dsc_probe_dt =_ "get dev-index error12" # 不会打印信息
      drivers/alidrivers/modules/alidsc/ca_dsc_core.c:292 [alidsc]ca_dsc_read =p "read: session#%d read returned %d bytes12" # 打印信息
      ...
      ```
      - 等号后面加下划线，表示低于默认日志级别，不打印；
      - 等号后面加 p，表示 printk 的 KERN_EMERG 级别，强制打印。
   - 优点：
     - 开发版本：打开 CONFIG_DYNAMIC_DEBUG 和 CONFIG_DEBUG_FS，配合 dbgfs 动态观测和调试内核代码；
     - 正式版本：关闭 CONFIG_DYNAMIC_DEBUG 和 CONFIG_DEBUG_FS，所有 dbgfs 以及 dev_dbg 信息都从编译阶段自动移除。

### 编译器定义相关宏

## err
### err check
1. ERR_PTR, PTR_ERR, IS_ERR
  ```c
  // include/linux/err.h

  /* Kernel pointers have redundant information, so we can use a
   * scheme where we can return either an error code or a normal
   * pointer with the same return value. 
   * 
   * This should be a per-architecture thing, to allow different
   * error and pointer decisions.
   */
   #define MAX_ERRNO  4095

   #ifndef __ASSEMBLY__

   #define IS_ERR_VALUE(x) unlikely((x) >= (unsigned long)-MAX_ERRNO)

   static inline void * __must_check ERR_PTR(long error) {
    return (void *) error;
   }

   static inline long __must_check PTR_ERR(__force const void *ptr) {
    return (long) ptr;
   }

   static inline bool __must_check IS_ERR(__force const void *ptr) {
    return IS_ERR_VALUE((unsigned long)ptr);
   }

   ...

   #endif
  ```
  - 检查获得的指针是否出错：`IS_ERR`；
  - 函数返回值类型为整型，返回：`PTR_ERR`，一般将出错的指针作为 `PTR_ERR` 的参数；
  - 函数返回值类型为指针，返回：`ERR_PTR`，一般将错误码，如 `-ENOMEM` 等，作为 `ERR_PTR` 的参数；

### errno base
```c
// include/uapi/asm-generic/errno-base.h

#ifndef _ASM_GENERIC_ERRNO_BASE_H
#define _ASM_GENERIC_ERRNO_BASE_H

#define	EPERM		 1	/* Operation not permitted */
#define	ENOENT		 2	/* No such file or directory */
#define	ESRCH		 3	/* No such process */
#define	EINTR		 4	/* Interrupted system call */
#define	EIO		 5	/* I/O error */
#define	ENXIO		 6	/* No such device or address */
#define	E2BIG		 7	/* Argument list too long */
#define	ENOEXEC		 8	/* Exec format error */
#define	EBADF		 9	/* Bad file number */
#define	ECHILD		10	/* No child processes */
#define	EAGAIN		11	/* Try again */
#define	ENOMEM		12	/* Out of memory */
#define	EACCES		13	/* Permission denied */
#define	EFAULT		14	/* Bad address */
#define	ENOTBLK		15	/* Block device required */
#define	EBUSY		16	/* Device or resource busy */
#define	EEXIST		17	/* File exists */
#define	EXDEV		18	/* Cross-device link */
#define	ENODEV		19	/* No such device */
#define	ENOTDIR		20	/* Not a directory */
#define	EISDIR		21	/* Is a directory */
#define	EINVAL		22	/* Invalid argument */
#define	ENFILE		23	/* File table overflow */
#define	EMFILE		24	/* Too many open files */
#define	ENOTTY		25	/* Not a typewriter */
#define	ETXTBSY		26	/* Text file busy */
#define	EFBIG		27	/* File too large */
#define	ENOSPC		28	/* No space left on device */
#define	ESPIPE		29	/* Illegal seek */
#define	EROFS		30	/* Read-only file system */
#define	EMLINK		31	/* Too many links */
#define	EPIPE		32	/* Broken pipe */
#define	EDOM		33	/* Math argument out of domain of func */
#define	ERANGE		34	/* Math result not representable */

#endif

// include/uapi/asm-generic/errno.h
...
#define EREMOTEIO   121
...
```

### WARN_ON...

## 修饰符相关
### 模块
1. `__init`：
2. `__exit`：

### 编译器
- compiler:
  `include/linux/compiler.h`
  1. `__user`：
  2. `__iomem`：
- compiler-gcc:
  1. `__must_check`
    ```c
    #define __must_check  __attribute__((warn_unused_result))
    ```

## misc
### EXPORT_SYMBOL
`EXPORT_SYMBOL` 是 linux 内核中用于将符号（函数或全局变量）导出到内核符号表，从而允许其他内核模块或内核本身访问这些符号的宏。

1. 核心作用
内核编译后，默认情况下模块之间的符号是隔离的。即使一个模块定义了某个函数，另一个模块也无法直接调用它，因为链接器不知道这些符号的存在。
`EXPORT_SYMBOL` 的作用就是打破这种隔离，显示声明：“这个符号可以被其他模块使用”。
```c
// drivers/char/my_driver.c
int my_helper_function(int x) {
  return x * 2;
}
EXPORT_SYMBOL(my_helper_function);  // 导出，其他模块可见

// 另一个模块就可以使用
extern int my_helper_function(int);
```

2. 为什么需要它
   1. 内核模块的动态加载机制
    内核模块（`.ko`）是动态加载的，不像用户空间程序在编译时静态链接。模块加载时，内核需要运行时解析未定义的符号。
      ```txt
      模块 A (a.ko)          内核符号表          模块 B (b.ko)
      ┌─────────────┐       ┌─────────────┐      ┌─────────────┐
      │ my_func()   │◄──────│ my_func     │◄─────│ 调用 my_func()│
      │ (已导出)     │       │ (EXPORT)    │      │ (未定义符号)  │
      └─────────────┘       └─────────────┘      └─────────────┘
      ```
   2. 如果没有 `EXPORT_SYMBOL`
      ```bash
      # 模块 A 定义了 helper()
      # 模块 B 想调用 helper()

      # 如果 A 没有 EXPORT_SYMBOL(helper)
      insmod b.ko
      # 输出：Unknown symbol in module: helper
      # 加载失败
      ```

3. 三种导出宏的区别
| 宏                                   | 可见范围            | 使用场景                |
| ----------------------------------- | --------------- | ------------------- |
| `EXPORT_SYMBOL(name)`               | **所有模块**        | 通用驱动函数，无许可证限制       |
| `EXPORT_SYMBOL_GPL(name)`           | **仅 GPL 许可证模块** | 强制要求调用者也是 GPL，保护开源  |
| `EXPORT_SYMBOL_NS(name, namespace)` | 指定命名空间内的模块      | 内核 5.4+ 引入，符号命名空间隔离 |
```c
// 示例
EXPORT_SYMBOL(usb_register)   // 任何模块都能使用
EXPORT_SYMBOL_GPL(schedule)   // 只有 GPL 模块能用
EXPORT_SYMBOL_NS(my_func, USB_STORAGE)  // 只有 USB_STORAGE 命名空间可见
```

4. 符号表查看
```bash
# 查看当前内核导出的所有符号
cat /proc/kallsyms | grep "T " | head

# 查看某个模块导出的符号
cat /proc/kallsyms | grep "my_module"

# 查看模块的依赖和未解析符号
modinfo my_module.ko
nm my_module.ko | grep " U " # 查看未定义符号（依赖其他模块）
```

5. 实际使用场景
   1. 驱动复用通用函数
    ```c
    // drivers/base/firmware_loader.c
    int firmware_request_nowarn(struct firmware **fw, const char *name, struct device *device) {
      // ...
    }
    EXPORT_SYMBOL(firmware_request_nowarn);
    // 任何需要加载固件的驱动都可以调用
    // request_firmware() 等函数都是这样导出的
    ```
   2. 子系统向驱动暴露 API
    ```c
    // net/core/dev.c
    int register_netdev(struct net_device *dev) {
      // ...
    }
    EXPORT_SYMBOL(register_netdev);
    // 网卡驱动通过调用 register_netdev() 注册自己
    ```

6. 与模块依赖的关系
`EXPORT_SYMBOL` 只是暴露符号，不自动建立模块加载顺序依赖。
如果模块 B 使用了模块 A 导出的符号：
```bash
# 手动加载时必须先加载 A，再加载 B
insmod a.ko
insmod b.ko

# 如果反过来
insmod b.ko     # 失败！A 还没有加载，符号不存在
```
解决方案：在模块 B 的代码中声明依赖：
```c
// 在 b.c 中
MODULE_IMPORT_NS(USB_STORAGE);    // 导入命名空间（可选）
// 或者让 modprobe 自动处理依赖：
// 在 Makefile 中配置，或在 /lib/modules/`uname -r`/modules.dep 中声明
```

7. 总结
`EXPORT_SYMBOL` 是 linux 内核模块间的“接口声明”，它将函数或变量注册到内核全局符号表，使得其他内核模块能够在运行时动态链接并调用这些符号。没有它，模块之间就是完全隔离的孤岛。

### likely 和 unlikely
> 1. [likely()/unlikely()宏的编译器优化机制分析](https://segmentfault.com/a/1190000046802665)
> 2. [linux内核中的likely与unlikely](https://zhuanlan.zhihu.com/p/600308066)
> 3. [深入理解 likely/unlikely](https://wenku.csdn.net/column/ow9nomkz3zk)

1. __builtin_expect 函数声明：
   - GCC v2.96 版本引入的，其声明如下：
      ```c
      long __builtin_expect(long exp, long c);
      ```
   - GCC 提供该内建函数来帮助程序员处理分支预测，它的作用是期望表达式 exp 的值等于 c。如果 exp == c 条件成立的概率高，那么性能将会得到提升，否则性能反而下降；
   - 原因是 GCC 在编译过程中，会将可能性更大的代码紧跟着前面的代码，从而减少指令跳转带来的性能上的下降，达到优化程序的目的：__builtin_expect 函数用来引导 GCC 进行条件分支复测、跳转指令相对于顺序执行的指令会多消耗 CPU 的时间。
2. likely() 和 unlikely 宏的本质是调用 GCC 内置函数：
  ```c
  #define likely()    __builtin_expect(!!(x), 1)
  #define unlikely()  __builtin_expect(!!(x), 0)
  ```
  **这些宏通过指导编译器进行分支预测优化，可以显著提升程序性能**。编译器基于下述提供的信息进行代码布局优化，将大概率执行的代码路径（热路径）放在条件判断后紧邻的位置，减少跳转指令的使用（self：同时也能减少流水线重新加载）。
   - likely(condition)：表示条件为真的概率很高；
   - unlikely(condition)：表示条件为真的概率很低。

3. 优化机制详解：
   1. 分支预测与流水线
    现代 CPU 采用流水线技术执行指令，当遇到条件分支时：
      - CPU 会预测分支走向；
      - 预测错误时需清空流水线（约 10~20 时钟周期惩罚）；
      - likely/unlikely 帮助编译器优化代码布局，提供预测准确性。
   2. 代码布局优化
    编译器根据概率提示调整代码块的位置：
      - 大概率分支：放在条件判断后紧邻的位置（无跳转）；
      - 小概率分支：通过跳转指令移到较远位置；
   3. 优化收益
      - 减少跳转指令：热路径顺序执行，减少 jmp 指令；
      - 提高指令缓存命中率：高频代码集中排列；
      - 降低分支预测错误率：配置 CPU 的分支预测器。

4. 适用场景：
  ```c
  // 1. 错误处理路径：使用 unlikely
  if (unlike(error_condition)) {
    // 错误处理
  }

  // 2. 高频执行路径：使用 likely
  while (likely(running)) {
    // 主循环体
  }

  // 3. 性能关键代码：如网络数据包处理、文件系统操作等
  ```

5. 注意事项：
   1. 平台兼容性：非 GCC 编译器可能不支持；
   2. 语义不变性：只影响性能，不改变程序行为；
   3. 避免滥用：在非性能关键路径避免使用；
   4. 概率准确性：确保提示与实际执行概率一致。
6. 总结：
  likely()/unlikely() 宏通过指导编译器优化代码布局：
   1. 将大概率执行的代码放在条件判断后紧邻的位置；
   2. 减少不必要的跳转指令；
   3. 提高 CPU 流水线效率和指令缓存命中率；
   4. 降低分支预测错误带来的性能惩罚。

这种优化在 Linux 内核等高性能场景中尤为重要，可能带来 10％ 以上的性能提升。但使用时需确保分支概率评估准确，避免在不必要的地方增加代码复杂性。

### 内核版本检查
通过内核源码检查内核版本：
1. 顶层 Makefile（前 5 行）：
  ```bash
  cat Makefile | head -20
  # 输出
  VERSION = 5         # 主版本号
  PATCHLEVEL = 15     # 次版本号（稳定版/开发版）
  SUBLEVEL = 32       # 修订号
  EXTRAVERSION =      # 额外版本标识（如 -rc1，-custom）
  NAME =              # 版本代号（趣味命名）
  ...
  # 完整版本号：5.15.32
  ```
2. 自动生成的 `include/generated/compile.h`：
  ```bash
  cat include/generated/compile.h | grep UTS_
  # 输出
  #define UTS_RELEASE "5.15.32"
  #define UTS_VERSION "#1 SMP PREEMPT Mon Apr 2 10:00:00 CST 2026"
  #define LINUX_COMPILE_BY "user"
  #define LINUX_COMPILE_HOST "build-server"
  ```
3. `include/config/kernel.release`
  ```bash
  cat include/config/kernel.release
  # 输出 5.15.32
  ```
补充：内核代码中使用的版本宏：
  ```c
  #include <linux/version.h>

  // 可用的宏：
  KERNEL_VERSION(a, b, c)     // 生成版本号整数：(((a) << 16) + ((b) << 8) + (c))
  LINUX_VERSION_CODE          // 当前内核版本整数，如 0x050F20 = 5.15.32
  KERNEL_VERSION_MAJOR        // 5
  KERNEL_VERSION_PATCHLEVEL   // 15
  KERNEL_VERSION_SUBLEVEL     // 32
  ```

编译后的内核二进制：
  ```bash
  # 查看已编译内核镜像的版本字符串
  strings vmlinux | grep "Linux version"
  # 输出：Linux version 5.15.32 (user@build-server) (gcc-11.2.0) #1 SMP PREEMPT

  # 或查看 /proc/version（运行中的系统）
  cat /proc/version
  ```

### kconfig
```c
/* IS_ENABLED(CONFIG_FOO)  evaluates to 1 if CONFIG_FOO is set 
 * to 'y' or 'm', 0 otherwise.
 */
#define IS_ENABLED(option) \
  (IS_BUILTIN(option) || IS_MODULE(option))
```

### 系统初始化顺序
1. module_init, subsys_init, postcore_initcall...
