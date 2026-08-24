# SPI 子系统
## 概述

## 使用
### 设备树相关节点描述
- *pinctrl* 节点
    ```dts
    pinctrl_ecspi3: ecspi3grp {
            fsl,pins = <
                    MX6UL_PAD_UART2_RTS_B__ECSPI3_MISO        0x100b1  /* MISO*/
                    MX6UL_PAD_UART2_CTS_B__ECSPI3_MOSI        0x100b1  /* MOSI*/
                    MX6UL_PAD_UART2_RX_DATA__ECSPI3_SCLK      0x100b1  /* CLK*/
                    MX6UL_PAD_UART2_TX_DATA__GPIO1_IO20       0x100b0  /* CS*/
            >;
    };
    ```
- *spi* 控制器和设备节点
    ```dts
    // imx6ull.dtsi
    ecspi3: ecspi@02010000 {
        #address-cells = <1>;
        #size-cells = <0>;
        compatible = "fsl,imx6ul-ecspi", "fsl,imx51-ecspi";
        reg = <0x02010000 0x4000>;
        interrupts = <GIC_SPI 33 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&clks IMX6UL_CLK_ECSPI3>,
                <&clks IMX6UL_CLK_ECSPI3>;
        clock-names = "ipg", "per";
        dmas = <&sdma 7 7 1>, <&sdma 8 7 2>;
        dma-names = "rx", "tx";
        status = "disabled";
    };
    // imx6ull-alientek-emmc.dts
    &ecspi3 {
            fsl,spi-num-chipselects = <1>;
            cs-gpios = <&gpio1 20 GPIO_ACTIVE_LOW>;
            pinctrl-names = "default";
            pinctrl-0 = <&pinctrl_ecspi3>;
            status = "okay";

        spidev: icm20608@0 {
        compatible = "alientek,icm20608";
            spi-max-frequency = <8000000>;
            reg = <0>;
        };
    };

    // imx6qdl-sabresd.dtsi
    &ecspi1 {
        fsl,spi-num-chipselects = <1>;
        cs-gpios = <&gpio4 9 0>;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_ecspi>;
        status = "okay";

        flash: m25p80@0 {
            #address-cells = <1>;
            #size-cells = <1>;
            compatible = "st,m25p32";
            spi-max-frequency = <20000000>;
            reg = <0>;
        };
    };
    ```
    - *spi* 控制器节点描述
      - *fsl,spi-num-chipselects*：片选引脚的数量；
      - *cs-gpios*： 片选引脚集合（*gpio* 控制器、*gpio* 引脚、有效电平）；
      - *clocks*：*spi* 控制器具有两个时钟源输入端口（使用了两个时钟源），*ipg* 对应到控制器自身内部的时钟输入，*per* 对应到与 *spi* 设备进行通信时使用的时钟源。
      - 关于 *dma* 属性节点的描述查看 `Documentation/devicetree/bindings` 部分（笔记 `Linux/Documentation/devicetree-bindings/dma.md`）。
    - *spi* 设备节点描述
      - *spi* 设备节点作为 *spi* 控制器的子节点。
      - *spi-max-frequency*：*spi* 设备的最高频率，这个要根据所使用的 *spi* 设备来设置。
      - *reg*：表示 *m25p80* 这个设备所使用的片选信号，对应到 `spi_device->chip_select` 字段的值。
    - 设备树绑定要求：查看 `Documentation/devicetree/bindings` 部分（笔记 ``Linux/Documentation/devicetree-bindings/spi.md``）。

### 设备驱动
#### 注册与注销
- 注册：
    ```c
    // 注册
    // 0-成功；负值-失败
    int spi_register_driver(struct spi_driver *sdrv);
    ```
- 注销：
    ```c
    // 注销
    void spi_unregister_driver(struct spi_driver *sdrv);
    ```
- 注册示例：
    ```c
    static int xxx_probe(struct spi_device *spi) {}

    static int xxx_remove(struct spi_device  *spi) {}

    static const struct spi_device_id xxx_id[] = {
        {"xxx", 0},
        {/* sentinel */}
    };

    static const struct of_device_id xxx_of_match[] = {
        {.compatible = "xxx"},
        {/* sentinel */}
    };

    static struct spi_driver xxx_driver = {
        .probe = xxx_probe,
        .remove = xxx_remove,
        .driver = {
            .owner = THIS_MODULE,
            .name  = "xxx",
            .of_match_table = xxx_of_match,
        },
        .id_table = xxx_id;
    };

    static int __init xxx_init(void) {
        ...
        spi_register_driver(&xxx_driver);
        ...
    }

    static void __exit xxx_exit(void) {
        ...
        spi_unregister_driver(&xxx_driver);
        ...
    }

    module_init(xxx_init);
    module_exit(xxx_exit);
    MODULE_LICENSE("GPL");
    ```

#### 通信数据表示与传输
- 需要通过 *spi* 主机才能与设备（从机）进行通信，*spi* 设备驱动只需将需要传输的数据提交给 `struct spi_master` 表示的 *spi* 主机即可。常用的提交 *API* 有：
  - `spi_async()` 和 `spi_sync()`
      ```c
      // version: 4.1.15
      // include/linux/spi/spi.h

      // 返回值：成功-0，其他（负值）-失败
      extern int spi_async(struct spi_device *spi, struct spi_message *message);
      
      /* All these synchronous SPI transfer routines are utilities layered
       * over the core async transfer primitive. Here, "synchronous" means 
       * they will sleep uninterruptibly until the async transfer completes.
       */
      // It returns zero on success, else a negative error code.
      extern int spi_sync(struct spi_device *spi, struct spi_message *message);
      ```
  - *wrapper*：`spi_write()`、`spi_read()`、`spi_sync_transfer()`、`spi_write_then_read()`、`spi_w8r8()`、`spi_w8r16()`、`spi_w8r16be()`
      ```c
      // version: 4.1.15
      // include/linux/spi/spi.h

      /**
      * spi_write - SPI synchronous write
      * @spi: device to which data will be written
      * @buf: data buffer
      * @len: data buffer size
      * Context: can sleep
      *
      * This writes the buffer and returns zero or a negative error code.
      * Callable only from contexts that can sleep.
      */
      static inline int
      spi_write(struct spi_device *spi, const void *buf, size_t len)
      {
          struct spi_transfer	t = {
                  .tx_buf		= buf,
                  .len		= len,
              };
          struct spi_message	m;

          spi_message_init(&m);
          spi_message_add_tail(&t, &m);
          return spi_sync(spi, &m);
      }

      /**
      * spi_read - SPI synchronous read
      * @spi: device from which data will be read
      * @buf: data buffer
      * @len: data buffer size
      * Context: can sleep
      *
      * This reads the buffer and returns zero or a negative error code.
      * Callable only from contexts that can sleep.
      */
      static inline int
      spi_read(struct spi_device *spi, void *buf, size_t len)
      {
          struct spi_transfer	t = {
                  .rx_buf		= buf,
                  .len		= len,
              };
          struct spi_message	m;

          spi_message_init(&m);
          spi_message_add_tail(&t, &m);
          return spi_sync(spi, &m);
      }

      /**
      * spi_sync_transfer - synchronous SPI data transfer
      * @spi: device with which data will be exchanged
      * @xfers: An array of spi_transfers
      * @num_xfers: Number of items in the xfer array
      * Context: can sleep
      *
      * Does a synchronous SPI data transfer of the given spi_transfer array.
      *
      * For more specific semantics see spi_sync().
      *
      * It returns zero on success, else a negative error code.
      */
      static inline int
      spi_sync_transfer(struct spi_device *spi, struct spi_transfer *xfers,
          unsigned int num_xfers)
      {
          struct spi_message msg;

          spi_message_init_with_transfers(&msg, xfers, num_xfers);

          return spi_sync(spi, &msg);
      }

      /* this copies txbuf and rxbuf data; for small transfers only! */
      // 分为 2 个 transfer 进行传输，第一个用于发送数据，第二个用于接收数据
      extern int spi_write_then_read(struct spi_device *spi,
              const void *txbuf, unsigned n_tx,
              void *rxbuf, unsigned n_rx);

      /**
      * spi_w8r8 - SPI synchronous 8 bit write followed by 8 bit read
      * @spi: device with which data will be exchanged
      * @cmd: command to be written before data is read back
      * Context: can sleep
      *
      * This returns the (unsigned) eight bit number returned by the
      * device, or else a negative error code.  Callable only from
      * contexts that can sleep.
      */
      static inline ssize_t spi_w8r8(struct spi_device *spi, u8 cmd)
      {
          ssize_t			status;
          u8			result;

          status = spi_write_then_read(spi, &cmd, 1, &result, 1);

          /* return negative errno or unsigned value */
          return (status < 0) ? status : result;
      }

      /**
      * spi_w8r16 - SPI synchronous 8 bit write followed by 16 bit read
      * @spi: device with which data will be exchanged
      * @cmd: command to be written before data is read back
      * Context: can sleep
      *
      * This returns the (unsigned) sixteen bit number returned by the
      * device, or else a negative error code.  Callable only from
      * contexts that can sleep.
      *
      * The number is returned in wire-order, which is at least sometimes
      * big-endian.
      */
      static inline ssize_t spi_w8r16(struct spi_device *spi, u8 cmd)
      {
          ssize_t			status;
          u16			result;

          status = spi_write_then_read(spi, &cmd, 1, &result, 2);

          /* return negative errno or unsigned value */
          return (status < 0) ? status : result;
      }

      /**
      * spi_w8r16be - SPI synchronous 8 bit write followed by 16 bit big-endian read
      * @spi: device with which data will be exchanged
      * @cmd: command to be written before data is read back
      * Context: can sleep
      *
      * This returns the (unsigned) sixteen bit number returned by the device in cpu
      * endianness, or else a negative error code. Callable only from contexts that
      * can sleep.
      *
      * This function is similar to spi_w8r16, with the exception that it will
      * convert the read 16 bit data word from big-endian to native endianness.
      *
      */
      static inline ssize_t spi_w8r16be(struct spi_device *spi, u8 cmd)

      {
          ssize_t status;
          __be16 result;

          status = spi_write_then_read(spi, &cmd, 1, &result, 2);
          if (status < 0)
              return status;

          return be16_to_cpu(result);
      }
      ```

- *spi* 设备驱动在将数据提交给 *spi* 主机时，需要将数据表示为 `struct spi_message`，而 `struct spi_message` 包含由若干个 `struct spi_transfer` 组成的链表。设备驱动首先需要将数据表示为若干个 `struct spi_transfer` 实例，然后将这些实例依次添加到 `struct spi_message` 的链表上，最后通过上述 *API* 完成提交。
  - `struct spi_transfer` 结构体用于描述 *spi* 传输信息，它是 *spi* 子系统用于描述传输数据的单位。
  - `struct spi_transfer` 需要组织成 `struct spi_message`，而后者表示一次完整的 *spi* 通信过程。可以在 `struct spi_message` 中设置回调函数相关内容，从而在传输任务完成后调用。
  
- 操作顺序及函数接口如下：
  ```c
  // 使用 spi_message 之前需初始化
  void spi_message_init(struct spi_message *m);
  // 初始化完成后，将 spi_transfer 添加到 spi_message 队列中
  void spi_message_add_tail(struct spi_transfer *t, struct spi_message *m);
  // spi_message 准备好后即可传输
  // 同步传输，阻塞等待直到 spi 传输完成
  int spi_sync(struct spi_device *spi, struct spi_message *message);
  // 异步传输，非阻塞
  // 需设置 complete 成员变量（回调函数），当 spi 异步传输完成后调用此函数
  int spi_async(struct spi_device *spi, struct spi_message *message);
  ```

- 示例代码：
    ```c
    // spi 多字节发送
    static int spi_send(struct spi_device *spi, u8 *buf, int len) {
        int ret;
        struct spi_message m;

        struct spi_transfer t = {
            .tx_buf = buf,
            .len = len,
        };

        spi_message_init(&m);
        spi_message_add_tail(&t, &m);
        ret = spi_sync(spi, &m);
        return ret;
    }

    // spi 多字节接收
    static int spi_receive(struct spi_device *spi, u8 *buf, int len) {
        int ret;
        struct spi_message m;

        struct spi_transfer t = {
            .rx_buf = buf,
            .len = len,
        };

        spi_message_init(&m);
        spi_message_add_tail(&t, &m);
        ret = spi_sync(spi, &m);
        return ret;
    }
    ```

- 综上，*spi* 数据传输步骤：
  1. 申请并初始化 `struct spi_transfer` 实例，设置它的 `tx_buf`、`rx_buf`、`len` 成员变量，如果不需要发送/接收数据，则可以将相应成员变量的初始值设置为 `NULL`（不初始化）。
  2. 使用 `spi_message_init()` 初始化 `struct spi_message` 实例。
  3. 使用 `spi_message_add_tail()` 将设置好的 *spi_transfer* 实例添加到 *spi_message* 链表队列中。
  4. 通过 `spi_sync()/spi_async()` 调用完成 spi 数据同步/异步传输（设置相应回调）。

#### 设备驱动示例
- *spi nor flash*: `/drivers/misc/eeprom/at25.c`
- *spi eeprom*: `/drivers/mtd/devices/m25p80.c`

## SPI 子系统
### SPI Core
`drivers/spi/spi.c`: *spi init/core code*.

#### 入口
```c
// version: 4.1.15
// drivers/spi/spi.c
static int __init spi_init(void) {
    int status;
    buf = kmalloc(SPI_BUFSIZ, GFP_KERNEL);
    if (!buf) {
        staut = -ENOMEM;
        goto err0;
    }
    status = bus_register(&spi_bus_type);
    if (status < 0)
        goto err1;
    status = class_register(&spi_master_class);
    if (status < 0)
        goto err2;
    if (IS_ENABLED(CONFIG_OF_DYNAMIC))
        WARN_ON(of_reconfig_notifier_register(&spi_of_notifier));
    return 0;
err2:
    bus_unregister(&spi_bus_type);
err1:
    kfree(buf);
    buf = NULL;
err0:
    return status;
}
/* board_info is normally registered in arch_initcall(), 
    * but event essential drivers wait till later 
    *
    * REVISIT only boardinfo really needs static linking. the
    * rest (device and driver registration) _could_ be dynamically
    * linked (modular) ... costs include needing to have boardinfo
    * data structures be much more public.
    */
postcore_initcall(spi_init);
```
  - 全局变量 `buf` 在 `spi_write_then_read()` 函数中使用，是为 *rx* 和 *tx* 提前分配的内存。
  - 注册 *bus_type*：`spi_bus_type`，对应的入口目录为：`/sys/bus/spi`。
  - 注册 *class*：`spi_master_class`，对应的入口目录为：`/sys/class/spi_master`。
  - [ ] `CONFIG_OF_DYNAMIC`，动态设备树的配置项？
  - 入口的启动顺序为 `postcore_initcall()`。

1. `spi_bus_type`
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    struct bus_type spi_bus_type = {
        .name = "spi",
        .dev_groups = spi_dev_groups,
        .match = spi_match_device,
        .uevent = spi_uevent,
    };

    static ssize_t modalias_show(struct device *dev, struct device_attribute *a, char *buf) {
        const struct spi_device *spi = to_spi_dev(dev);
        int len;
        len = acpi_device_modalias(dev, buf, PAGE_SIZE - 1);
        if (len != -ENODEV)
            return len;
        return sprintf(buf, "%s%s\n", SPI_MODULE_PREFIX, spi->modalias);
    }
    static DEVICE_ATTR_RO(modalias);

    static struct attribute *spi_dev_attrs[] = {
        &dev_attr_modalias.attr,
        NULL,
    };
    ATTRIBUTE_GROUPS(spi_dev);

    static int spi_match_device(struct device *dev, struct device_driver *drv) {
        const struct spi_device *spi = to_spi_device(dev);
        const struct spi_driver *sdrv = to_spi_driver(drv);

        // Attempt an OF style match
        if (of_driver_match_device(dev, drv))
            return 1;
        // Then try ACPI
        if (acpi_driver_match_device(dev, drv))
            return 1;
        if (sdrv->id_table)
            return !!spi_match_id(sdrv->id_table, spi);
        
        return strcmp(spi->modalias, drv->name) == 0;
    }

    static int spi_uevent(struct device *dev, struct kobject kobj_uevent_env *env) {
        const struct spi_device *spi = to_spi_device(dev);
        int rc;
        rc = acpi_device_uevent_modalias(dev, env);
        if (rc != -ENODEV)
            return rc;
        add_uevent_var(env, "MODALIAS=%s%s", SPI_MODULE_PREFIX, spi->modalias);
        return 0;
    }
    ```
    - `spi_bus_type->dev_groups` 字段为总线上的设备添加的属性，这里 *groups* 的数量为 1，且该 *group* 未设置 `name` 字段，因此该属性文件会直接添加到设备所在目录下。属性的名字为 *modalias*，它为只读文件。由于源码中 `acpi_device_modalias()` 直接返回 `-ENODEV`，因此读 *modalias* 文件时，返回的内容是：“*spi:*`spi_device->modalias`”（`#define SPI_MODULE_PREFIX "spi:"`）。
    - 设备和驱动是否匹配的比较顺序：*OF->ACPI->id_table-> spi->modalias 和 drv->name*。
      - *OF*: 比较设备树结点 *compatible* 属性值与 `drv->of_match_table` 中元素的 `compatible` 值是否匹配；
      - *id_table*: 比较 `spi_device->modalias` 与 `spi_device_id->name` 是否相等；
      - `spi_device->modalias` 值的设备树来源是 *compatible* 属性值“,”的后面部分或全部（没有“,”时）。
    - `apci_device_uevent_modalias()` 也会直接返回 `-ENODEV`，因此 `spi_uevent()` 会为该总线下的设备，当需要产生 uevent 事件时，增加名为 `"MODALIAS"` 的环境变量，该环境变量的值与该 *spi* 设备目录下 *modalias* 文件的读出值一致。

2. `spi_master_class`
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    static struct class spi_master_class = {
        .name = "spi_master",
        .owner = THIS_MODLE,
        .dev_release = spi_master_release,
    };

    static void spi_master_release(struct device *dev) {
        struct spi_master *master;
        master = container_of(dev, struct spi_master, dev);
        kfree(master);
    }
    ```
    - 在 *SoC* 上可能具有多个 *spi* 控制器，它们都能被归类到 `spi_master_class`。对应到 *sysfs* 中为 `/sys/class/spi_master` 下的子目录。

#### 注册控制器
```c
// version: 4.1.15
// drivers/spi/spi.c
/**
 * spi_alloc_master - allocate SPI master controller 
 * @dev: the controller, possibly using the platform_bus
 * @size: how much zeroed driver-private data to allocate; the pointer
 *  to this memory is in the driver_data field of the returned device,
 *  accessible with spi_master_get_devdata().
 * Context: can sleep
 * 
 * This call is used only by SPI master controller drivers, which are
 * the only ones directly touching chip registers. It's how they allocate
 * an spi_master structure, prior to calling spi_register_master().
 *
 * This must be called from context that can sleep. It returns the SPI
 * master structure on success, else NULL.
 *
 * The caller is responsible for assigning the bus number and initiliazing
 * the master's methods before calling spi_register_master(); and (after
 * errors adding the device) calling spi_master_put() to prevent a memory leak
 */
 struct spi_master *spi_alloc_master(struct device *dev, unsigned size) {
    struct spi_master *master;
    if (!dev)
        return NULL;
    master = kzalloc(size + sizeof(*master), GFP_KERNEL);
    if (!master)
        return NULL;
    device_initialize(&master->dev);
    master->bus_num = -1;
    master->num_chipselect = 1;
    master->dev.class = &spi_master_class;
    master->dev.parent = get_device(dev);
    spi_master_set_devdata(master, &master[1]);
 }

 static inline void spi_master_put(struct spi_master *master) {
    if (master)
        put_device(&master->dev);
 }
```
- 在注册 *spi_master* 之前需要通过 `spi_alloc_master()` 函数分配得到需要注册的 *spi_master*，并基于此设置其 *bus number* 和相关函数（主要是 `transfer_one_message()`）。
- `spi_alloc_master()` 的功能：
  - 分配 *spi_master* 所需内存，内存布局图示见 `Linux/Driver/subsys/image.drawio` 中 *spi_master-memLayout*。
  - 调用 `device_initialize(&master->dev)`，是将 `master->dev` 完成注册的第一步（后续通过 `device_add()` 完成注册）。
  - 总线编号为 -1（初始值，后续根据设备树指定或自动分配）；
  - 片选引脚数量为 1（要求片选引脚数量至少为 1）；
  - 指定 *spi_master* 对应的 *class* 为 `spi_master_class`，后续可在 *sysfs* 中的 `/sys/class/spi_master` 目录下找到。
  - 指定 `spi_master->dev.parent` 构建层级关系。
  - 通过 `spi_master_set_devdata()` 将 `master->dev.drvdata` 指向用户指定内存区。

```c
// version: 4.1.15
// drivers/spi/spi.c
/**
 * spi_register_master - register SPI master controller
 * @master: initialized master, originally from spi_alloc_master()
 * Context: can sleep
 *
 * SPI master controllers connect to their drivers using some non-SPI
 * bus, such as the platform bus. The final stage of probe() in that
 * code includes calling spi_register_master() to hook up (连接) to this SPI
 * bus glue.
 *
 * SPI controllers use board specific (often SOC specific) bus numbers,
 * and board-specific addressing for SPI devices combines those numbers
 * with chip select numbers. Since SPI does not directly support dynamic
 * device identification, boards need configuration tables telling which
 * chip is at which address.
 *
 * This must be called from context that can sleep. It returns zero
 * on success, else a negative error code (dropping the master's refcount).
 * After a successful return, the caller is responsible for calling 
 * spi_unregister_master().
 */
int spi_register_master(struct spi_master *master) {
    static atomic_t dyn_bus_id = ATOMIC_INIT((1<<15) - 1);
    static device *dev = master->dev.parent;
    struct boardinfo *bi;
    int status = -ENODEV;
    int dynamic = 0;

    if (!dev)
        return -ENODEV;

    status = of_spi_register_master(master);
    if (status)
        return status;
    /* even if it's just one always-selected device, there must
     * be at least one chipselect.
     */
    if (master->num_chipselect == 0)
        return -EINVAL;
    if ((master->bus_num < 0) && master->dev.of_node)
        master->bus_num = of_alias_get_id(master->dev.of_node, "spi");
    /* convention: dynamically assigned bus IDs count down from the max */
    if (master->bus_num < 0) {
        /* FIXME switch to an IDR based scheme, something like
         * I2C now uses, so we don't run out of "dynamic" IDs 
         */
        master->bus_num = atomic_dec_return(&dyn_bus_id);
        dynamic = 1;
    }

    INIT_LIST_HEAD(&master->queue);
    spin_lock_init(&master->queue_lock);
    spin_lock_init(&master->bus_lock_spinlock);
    mutex_init(&master->bus_lock_mutex);
    master->bus_lock_flag = 0;
    init_completion(&master->xfer_completion);
    if (!master->max_dma_len)
        master->max_dma_len = INT_MAX;

    /* register the device, then userspace will see it. 
     * registration fails if the bus ID is in use.
     */
    dev_set_name(&master->dev, "spi%u", master->bus_num);
    status = device_add(&master->dev);
    if (status < 0)
        goto done;
    dev_dbg(dev, "registered master %s%s\n", dev_name(&master->dev), dynamic ? " (dynamic)" : "");

    /* if we're using a queue driver, start the queue */
    if (master->transfer)
        dev_info(dev, "master is unqueued, this is deprecated\n");
    else {
        status = spi_master_initialize_queue(master);
        if (status) {
            device_del(&master->dev);
            goto done;
        }
    }

    mutex_lock(&board_lock);
    list_add_tail(&master->list, &spi_master_list);
    list_for_each_entry(bi, &board_list, list)
        spi_match_master_to_boardinfo(master, &bi->board_info);
    mutex_unlock(&board_lock);

    /* Register devices from the device tree and ACPI */
    of_register_spi_devices(master);
    acpi_register_spi_devices(master);
done:
    return status;
}
EXPORT_SYMBOL_GPL(spi_register_master);

/** 
 * spi_unregister_master - unregister SPI master controller
 * @master: the master being unregistered
 * Context: can sleep
 *
 * This call is used only by SPI master controller drivers, which
 * are the only ones directly touching chip registers.
 *
 * This must be called from context that can sleep.
 */
void spi_unregister_master(struct spi_master *master) {
    int dummy;
    if (master->queued) {
        if (spi_destroy_queue(master))
            dev_err(&master->dev, "queue remove failed\n");
    }
    mutex_lock(&board_lock);
    list_del(&master->list);
    mutex_unlock(&board_lock);
    dummy = device_for_each_child(&master->dev, NULL, __unregister);
    device_unregister(&master->dev);
}

static int of_spi_register_master(struct spi_master *master) {
    int nb, i, *cs;
    struct device_node *np = master->dev.of_node;
    
    if (!np)
        return 0;
    
    nb = of_gpio_named_count(np, "cs-gpios");
    master->num_chipselect = max_t(int, nb, master->num_chipselect);
    /* Return error only for an incorrectly formed cs-gpios property */
    if (nb == 0 || nb == -ENOENT)
        return 0;
    else if (nb < 0)
        return nb;
    cs = devm_kzalloc(&master->dev, sizeof(int) * master->num_chipselect, GFP_KERNEL);
    master->cs_gpios = cs;
    if (!master->cs_gpios)
        return -ENOMEM;
    for (i = 0; i < master->num_chipselect; i++)
        cs[i] = -ENOENT;
    for (i = 0; i < nb; i++)
        cs[i] = of_get_named_gpio(np, "cs-gpios", i);
    return 0;
}

static int __unregister(struct device *dev, void *null) {
    spi_unregister_device(to_spi_device(dev));
    return 0;
}
```
- 调用 `of_spi_register_master()`：
  - 该函数的主要功能是片选引脚信息的获取和设置。
  - 根据设备树（*cs-gpios* 属性值中描述 *gpio* 的个数）和 *master* 原本设置的片选引脚数量计算有效的片选引脚数量（取两者中的最大值），并赋值给 `master->num_chipselect`。
  - 分配 `master->cs_gpios` 对应的内存（`int` 列表），并将从设备树 *cs-gpios* 解析得到的若干个 *gpio* 值赋值给内存（其他未赋值的为默认值 `-ENOENT`）。
- *spi master* 被要求至少有一个片选引脚。
- 设置 *spi master* 的总线编号：
  - 如果 `master->bus_num < 0`，会根据 *master* 对应的设备树别名节点和 *"spi"* 词干找到对应的 *id* 来设置 `master->bus_num`，例如设备树节点 *aliases* 下 *"spi0"* 的后缀 *"0"* 就是这个 *id*。
  - 如果取得的 *bus_num* 仍小于 0，则会根据函数的静态变量 `dyn_bus_id` 来设置。
  - 后续注册的 *device*，即 `master->dev`，将会以 *"spi%u"* 的形式命名，后续数字就是 `master->bus_num`。
- 进行一些初始化设置，初始化的内容包括：
  - `master->queue`（*msg* 链表头）
  - `master->queue_lock`（*msg* 队列的自旋锁）
  - `master->bus_lock_spinlock`（总线自身的自旋锁）
  - `master->bus_lock_mutex`（总线自身的互斥锁）
  - `master->bus_lock_flag`（总线上锁标志）
  - `master->xfer_completion`（传输完成量）
  - `master->max_dma_len`，如果驱动未设置，这里默认取最大值 `INT_MAX`
- 注册 `master->dev`，由于在 `spi_alloc_master()` 中的设置：`master->dev.class = &spi_master_class` 和 `master->dev.parent = get_device(dev)`，因此创建相关的入口目录。如果 `spi_alloc_master()` 传入的 `dev` 参数不为 `NULL`，且 `dev->class` 已经被设置，则：
  - `master->dev->kobject.parent` 是 `dev->kobj`。于是会在该 *kobject* 对应目录下创建名为 *"spi%u"* (数字以 `master->bus_num` 填充) 的目录，并在该目录下创建文件：*uevent* 文件、*subsystem* (链接，指向 `master->dev->class->p->subsys.kobj`，即 `/sys/class/spi_master`)、*device* (链接，指向 `master->dev->parent->kobj`)...
  - 在 `sys/class/spi_master` 目录下创建名为 *"spi%u"* 的链接，它指向 `master->dev->kobject`。
- 关于 `master->transfer()` 函数。
  - 推荐的选择是使用 *spi core* 提供的默认实现函数，以实现 *queued driver*（注册时 *master* 的该字段为空，交给 *spi core* 来设置）。
  - 如果 *spi master* 提供了此函数，则会在注册时提示“*master is unqueued, this is deprecated*”。已弃用的方法。对该函数的要求：见 [2]-Linux 内核 SPI 支持概述-这绝对不能睡眠。其职责是安排传输发生并发出其 `complete()` 回调。这两者通常会在稍后、在其他传输完成后发生，并且如果控制器处于空闲状态，则需要将其重新启动（*kickstarted*）。此方法不用于已排队的控制器，并且如果实现了 `transfer_one_message()` 和 `(un)prepare_transfer_hardware()`，则必须为 `NULL`。
- 通常 `master->transfer()` 使用 *spi core* 默认实现，此时调用 `spi_master_initialize_queue()`：
  - 通过 *kthread*（注释中称为 *message pump thread*）执行 *spi msg* 传输任务，*spi* 设备驱动只需将传输需求提交给对应的 *spi master* 即可（将 *msg* 挂到 `spi_master->queue` 上），后续传输执行由该线程完成。
  - *spi master* 可以通过设置自身的 `rt` 字段（在注册之前），将上述 *kthread* 的调度策略提升至实时调度，从而减少传输的延时。
- 将 *spi master* 挂到全局变量 `spi_master_list` 表示的链表上。
- 以多种方式注册 *spi_device*：
  - 源码形式：被提前挂载在全局变量 `board_list` 表示的链表上，元素类型为 `struct boardinfo`，具有两个字段：`struct list_head list` 和 `struct spi_board_info board_info`。详细信息见源文件 `drivers/spi/spi.c`。
  - 设备树形式：`of_register_spi_devices()`：
    - 调用 `spi_alloc_device()` 分配 *spi device* 内存，并对其进行初始化设置；
    - 设置 `spi_device->modalias`，内容取自设备树结点的 *compatible* 属性值；
    - 设置 `spi_device->chip_select`，内容取自设备树结点的 *reg* 属性值；
    - 设置 `spi_device->mode`，设备树结点提供了 *spi-cpha*、*spi-cpol*、*spi-cs-high*、*spi-3wire*、*spi-lsb-first* 属性值时对应的模式位置位。另外，根据 *spi-tx-bus-width* 和 *spi-rx-bus-width* 的属性值，置位 `SPI_TX_DUAL` 等模式位。
    - 设置 `spi_device->max_speed_hz`，内容取自设备树结点的 *spi-max-frequency* 属性值；
    - 设置 `spi_device->irq`，对应设备树结点中的第一个中断号；
    - 将 `spi_device->dev.of_node` 设置为当前设备树结点；
    - 调用 `spi_add_device()` 将当前 *spi* 设备注册到 *spi* 总线上（注册时会判断 *spi master* 是否支持该 *spi* 设备）。
  - ACPI 形式：`acpi_register_spi_devices()`；
- 总结该注册函数的功能：
  - 解析设备树，获取并设置片选引脚信息；
  - 设置总线编号；
  - 初始化相关字段，如链表、互斥锁、自旋锁、完成量等；
  - 注册 *spi_master* 对应的 *device*；
  - （默认情况下，`master->transfer` 为空）启用 *message pump thread*，初始化相关内容，包括：*kthread_worker*、*kthread_work* 和 *task_struct* 等；
  - 以多种方式解析并注册在此控制器上的 *spi device*。

#### 注册设备驱动
```c
// version: 4.1.15

// include/linux/spi/spi.h
/** 
 * module_spi_driver() - Helper macro for registering a SPI driver 
 * @__spi_driver: spi_driver struct
 *
 * Helper macro for SPI drivers which do not do anything special in
 * module init/exit. This eliminates a lot of boilerplate (样板文件).
 * Each module may only use this macro once, and calling it replaces
 * module_init() and module_exit()
 */
#define module_spi_driver(__spi_driver) \
    module_driver(__spi_driver, spi_register_driver, spi_unregister_driver)

// drivers/spi/spi.c
/**
 * spi_register_driver - register a SPI driver 
 * @sdrv: the driver to register
 * Context: can sleep
 */
int spi_register_driver(struct spi_driver *sdrv) {
    sdrv->driver.bus = &spi_bus_type;
    if (sdrv->probe)
        sdrv->driver.probe = spi_drv_probe;
    if (sdrv->remove)
        sdrv->driver.remove = spi_drv_remove;
    if (sdrv->shutdown)
        sdrv->driver.shutdown = spi_drv_shutdown;
    return driver_register(&sdrv->driver);
}

static int spi_drv_probe(struct device *dev) {
    const struct spi_driver *sdrv = to_spi_driver(dev->driver);
    int ret;
    ret = of_clk_set_defaults(dev->of_node, false);
    if (ret)
        return ret;
    ret = dev_pm_domain_attach(dev, true);
    if (ret != -EPROBE_DEFER) {
        ret = sdrv->probe(to_spi_device(dev));
        if (ret)
            dev_pm_domain_detach(dev, true);
    }
    return ret;
}

static int spi_drv_remove(struct device *dev) {
    const struct spi_driver *sdrv = to_spi_driver(dev->driver);
    int ret;
    ret = sdrv->remove(to_spi_device(dev));
    dev_pm_domain_detach(dev, true);
    return ret;
}

static void spi_drv_shutdown(struct device *dev) {
    const struct spi_driver *sdrv = to_spi_driver(dev->driver);
    sdrv->shutdown(to_spi_device(dev));
}

// include/linux/spi/spi.h
/* spi_unregister_driver - reverse effect of spi_register_driver 
 * @sdrv: the driver to unregister
 * Context: can sleep
 */
static inline void spi_unregister_driver(struct spi_driver *sdrv) {
    if (sdrv)
        driver_unregister(&drv->driver);
}
```
- `of_clk_set_defaults()`，会解析设备树结点中 *assgined-{clocks/clock-parents/clock-rates}* 等属性，设置设备的时钟源和时钟频率。具体信息源码：`drivers/clk/clk-config.c`。
- `dev_pm_domain_attach()` 是与 *Power Management* 相关的 *API*，这里提供的参数 `true` 表示需要将设备 *power on*。查看源码时发现其调用的 `apci_dev_pm_attach()` 直接返回 `-ENODEV`（未配置选项 `CONFIG_ACPI`）。`dev_pm_domain_detach()` 中的 `true` 表示将设备 *power off*。这里会继续执行到 `sdrv->probe()`，即 *spi driver* 的 `probe()` 回调被调用。

#### 通信接口
1. `spi_async()`:
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /**
     * spi_async - asynchronous SPI transfer
     * @spi: device with which data will exchanged
     * @message: describes the data transfers, including completion
     *  callback
     * Context: any (irqs may be blocked, etc)
     * 
     * This call may be used in_irq and other contexts which can't
     * sleep, as well as from task contexts which can sleep.
     *
     * The completion callback is invoked in a context which can't
     * sleep. Before that invocation, the value of message->status
     * is undefined. When the callback is issued, message->status
     * holds either zero (to indicate complete success) or a negative
     * error code. After that callback returns, the driver which issued
     * the transfer request may deallocate the associated memory;
     * it's no longer in use by any SPI core or controller driver code.
     *
     * Note that although all messages to a spi_device are handled
     * in FIFO order, messages may go to different devices in other
     * in other orders. Some device might be higher priority, or
     * have various "hard" access time requirements, for example.
     *
     * On detection of any fault during the transfer, processing of
     * the entire message is aborted, and the device is deselected.
     * Until returning from the associated message completion callback,
     * no other spi_message queued to that device will be processed.
     * (This rule applies equally to all the synchronous transfer
     * calls, which are wrappers around this core asynchronous primitive.)
     */
    int spi_async(struct spi_device *spi, struct spi_message *message) {
        struct spi_master *master = spi->master;
        int ret;
        unsigned long flags;
        ret = __spi_validate(spi, message);
        if (ret != 0)
            return ret;
        spi_lock_irqsave(&master->bus_lock_spinlock, flags);
        if (master->bus_lock_flag)
            ret = -EBUSY;
        else
            ret = __spi_async(spi, message);
        spin_unlock_irqrestore(&master->bus_lock_spinlock, flags);
        return ret;
    }
    EXPORT_SYMBOL_GPL(spi_async);

    static int __spi_validate(struct spi_device *spi, struct spi_message *message) {
        struct spi_master *master = spi->master;
        struct spi_transfer *xfer;
        int w_size;
        if (list_empty(&message->transfers))
            return -EINVAL;
        /* Half-duplex links include original MicroWire, and ones
         * with only one data pin like SPI_3WIRE (switches direction)
         * or where either MOSI or MISO is missing. They can also be
         * caused by software limitations.
         */
        if ((master->flags & SPI_MASTER_HALF_DUPLEX) || 
            (spi->mode & SPI_3WIRE)) {
            unsigned flags = master->flags;
            list_for_each_entry(xfer, &message->transfers, transfer_list) {
                if (xfer->rx_buf && xfer->tx_buf)
                    return -EINVAL;
                if ((flags & SPI_MASTER_NO_TX) && xfer->tx_buf)
                    return -EINVAL;
                if ((flags & SPI_MASTER_NO_RX) && xfer->rx_buf)
                    return -EINVAL;
            }
        }
        /* Set transfer bits_per_word and max speed as spi device 
         * default if it is not set for this transfer.
         * Set transfer tx_nbits and rx_nbits as single transfer
         * default (SPI_NBITS_SINGLE) if it is not set for this transfer.
         */
        ...
        /* SPI transfer length should be multiple of SPI word size 
         * where SPI word size should be power-of-two multiple
         */
        ...
        /* Check transfer tx/rx_nbits */
        ...
        message->status = -EINPROGRESS;
        return 0;
    }

    static int __spi_async(struct spi_device *spi, struct spi_message *message) {
        struct spi_master *master = spi->master;
        message->spi = spi;
        trace_spi_message_submit(message);
        return master->transfer(spi, message);
    }
    ```
    - 对传输的 *spi_message* 的要求：
      - *message* 中包含的 *transfer* 数量不能为 0；
      - 如果 *spi* 控制器（*master*）只能进行半双工传输或 *spi* 设备（*device*）本身要求通过三线进行传输，则每个 *transfer* 都不能违反这个限制；
      - 如果 *transfer* 没有提供 *bits_per_word* 和 *max speed* 信息，则默认使用 *spi* 设备的信息；如果 *transfer* 没有指定是 *DSPI* 还是 *QSPI*，则默认使用 *single SPI*。最后，*transfer* 的 *bits_per_word* 需要能够被 *spi* 控制器支持且不能超过 *32* 位。
      - *transfer* 的字节长度能够以整数次进行传输，即能够被字长整除。字长根据 *bits_per_word* 计算得到：小于等于 8 *bit* 为 1，9 到 16（包含 16）为 2，其他为 4。
      - *transfer* 的 *speed* 要求需要在 *spi* 控制器的支持范围内。如果超过控制器的最大速度，则使用控制器的最大速度进行传输；如果小于控制器的最小速度，则不支持。
      - 对 *tx/rx_nbits* 的要求：只能是 *SINGLE*、*DUAL*、*QUAD* 之一，其他不支持；另外 *transfer* 的 *tx/rx_nbits* 要求需要 *spi* 设备支持。
    - `spi_async()` 的功能总结：
      - 验证 *message* 的有效性；
      - 默认情况下 `master->transfer()` 的实现为 `spi_queued_transfer()`，后者将 *message* 挂到 *spi* 控制器的传输队列上，并申请异步传输（当前默认都使用消息队列功能，即 *message pump thread*）。
2. `spi_sync()`
    ```c
    // version: 4.1.15
    // /drivers/spi/spi.c
    /**
     * spi_sync - blocking/synchronous SPI data transfers 
     * @spi: device with which data will be exchanged
     * @message: describes the data transfers
     * Context: can sleep
     *
     * This call may only be used from a context that may sleep.
     * The sleep is non-interruptible, and has no timeout. Low-overhead
     * controller drivers may DMA directly into and out of the message buffers.
     *
     * Note that the SPI device's chip select is active during the
     * message, and then is normally disabled between messages. Drivers
     * for some frequently-used devices may want to minimize costs
     * of selecting a chip, by leaving it selected in anticipation
     * that the next message will go to the same chip. (That may
     * increase power usage.)
     *
     * Also, the caller is guaranteeing that the memory associated
     * with the message will not be freed before this call returns.
     *
     * It returns zero on success, else a negative error code.
     */
    int spi_sync(struct spi_device *spi, struct spi_message *message) {
        return __spi_sync(spi, message, 0);
    }
    EXPORT_SYMBOL_GPL(spi_sync);

    static int __spi_sync(struct spi_device *spi, struct spi_message *message, int bus_locked) {
        DECLARE_COMPLETION_ONSTACK(done);
        int status;
        struct spi_master *master = spi->master;
        unsigned long flags;
        status = __spi_validate(spi, message);
        if (status != 0)
            return status;
        message->complete = spi_complete;
        message->context = &done;
        message->spi = spi;
        if (!bus_locked)
            mutex_lock(&master->bus_lock_mutex);
        /* If we're not using the legacy transfer method then we
         * will try to transfer in the calling context so special
         * case. This code would be less tricky (难办的，棘手的) if we could remove
         * the support for driver implemented message queues.
         */
        if (master->transfer == spi_queued_transfer) {
            spin_lock_irqsave(&master->bus_lock_spinlock, flags);
            trace_spi_message_submit(message);
            status = __spi_queued_transfer(spi, message, false);
            spin_unlock_irqrestore(&master->bus_lock_spinlock, flags);
        } else {
            status = spi_async_locked(spi, message);
        }
        if (!bus_locked)
            mutex_unlock(&master->bus_lock_mutex);
        if (status == 0) {
            /* Push out the messages in the calling context if we can */
            if (master->transfer == spi_queued_transfer)
                __spi_pump_messages(master, false);
            wait_for_completion(&done);
            status = message->status;
        }
        message->context = NULL;
        return status;
    }
    ```
    - 若 `transfer()` 函数使用的是 `spi_queued_transfer()`，则将 *message* 挂到 *master* 的传输队列上（不调用 `queue_kthread_work()`）；否则通过 `spi_async_locked()` 进一步调用 `transfer()` 完成传输；
    - 若上述操作成功：当 `transfer()` 函数使用的是 `spi_queued_transfer()` 时，则继续通过 `__spi_pump_messages()` 在 *pump* 线程上完成传输。当前线程通过 `wait_for_completion()` 实现同步。
3. 调用链-`spi_async()`
    ```c
    spi_async(spi_device, spi_message)
        __spi_validate(spi_device, spi_message)
        __spi_async(spi_device, spi_message)
            master->transfer(spi_device, spi_message) // transfer default: spi_queued_transfer
            // 上述相当于 spi_queue_transfer(spi_device, spi_message)，进一步调用：
                __spi_queued_transfer(spi_device, spi_message, need_pump:true)
                    spi_message->actual_length = 0;
                    spi_message->status = -EINPROGRESS;
                    list_add_tail(&spi_message->queue, &spi_device->master->queue)
                    if (!master->busy && need_pump)
                        queue_kthread_work(&spi_device->master->kworker, &spi_device->master->pump_messages)
    
    // spi_master->kworker 对应的 kthread 根据 spi_master->rt 字段值决定其是否按照实时策略进行调度
    // spi_master->pump_messages - default: spi_pump_messages

    // 异步线程 pump kthread: 执行 spi_pump_messages 函数的调用
    spi_pump_messages(kthread_work)
        __spi_pump_messages(spi_master, in_kthread:true) // spi_master 由 kthread_worker 通过 back-casting 获取
        // 该函数可以分为如下三个部分：
        // 1. 状态判断：
            spi_master->cur_msg、spi_master->idling
        // 2. 收尾工作：
            成立条件：spi_master->queue 为空或者 spi_master->running 为 false
            条件判断：!master->busy、!in_ktread
            idling 处理：
                busy=false,idling=true
                释放锁
                spi_master->unprepare_transfer_hardware(spi_master)
                spi_master->auto_runtime_pm -> pm_runtime_mark_last_busy(spi_master->dev.parent) + pm_runtime_put_autosuspend(spi_master->dev.parent)
                获取锁
                idling=false
                释放锁
        // 3. 消息队列处理
    
    // __spi_pump_messages 中的消息队列处理：
    1. spi_master->cur_msg = spi_master->queue 队列中第一个 msg，并将 spi_master->cur_msg 从 spi_master->queue 中删除
    2. 如果之前 spi_master not busy
        spi_master->auto_runtime_pm -> pm_runtime_get_sync(spi_master->dev.parent)
        spi_master->prepare_transfer_hardware(spi_master)
    3. spi_master->prepare_message(spi_master, spi_master->cur_msg) + master->cur_msg_prepared = true
    // 从第 3 步开始，message 处理结束（成功或失败）调用 spi_finalize_current_message(spi_master)
    4. spi_map_msg(spi_master, spi_master->cur_msg)
        a. 根据情况（spi_master MUST_RX/MUST_TX）给 spi_master->dummy_tx 和 spi_master->dummy_rx 分配零内存，并将其赋值给 msg 中的 transfer
        b. __spi_map_msg(spi_master, spi_message)
            1. 判断 spi_master->can_dma 是否为空
            2. 对 msg 中的每个 tranfser：
                检验：spi_master->can_dma(spi_master, spi_device:spi_message_spi, spi_transfer)
                通过：如果 spi_transfer->tx_buf 不为 NULL，则 spi_map_buf(spi_master, 
                    dma_chan->dam_device->dev:spi_master->dma_tx->device->dev, sg_table:
                    &spi_transfer->tx_sg, (void *)spi_transfer->tx_buf, spi_transfer->len, DMA_TO_DEVICE)
                     如果 spi_transfer->rx_buf 不为 NULL，则 spi_map_buf(spi_master, 
                        spi_master->dma_rx->device->dev, &spi_transfer->rx_sg, (void*)spi_transfer->rx_buf,
                        spi_transfer->len, DMA_FROM_DEVICE)
                对 spi_transfer 的 rx_buf 和 tx_buf 进行映射，结果保存在 spi_transfer 的 rx_sg 和 tx_sg 中
            3. spi_master->cur_msg_mapped = true
    5. spi_master->transfer_one_message(spi_master, spi_master->cur_msg)
    // transfer_one_message() 可由 master 自己提供，也可使用 spi core 提供的默认实现
    ```
4. 调用链-`spi_sync()`
    与 `spi_async()` 的调用链大致相同，略。

#### SPI Core 相关函数实现
1. *pump* 线程管理
   - 初始化：`spi_master_initialize_queue()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    static int spi_master_initialize_queue(struct spi_master *master) {
        int ret;
        master->transfer = spi_queued_transfer;
        if (!master->transfer_one_message)
            master->transfer_one_message = spi_transfer_one_message;
        /* Initialize and start queue */
        ret = spi_init_queue(master);
        if (ret){
            ... // error handling
        }
        master->queued = true;
        ret = spi_start_queue(master);
        if (ret) {
            .... // error handling
        }
        return 0;
    err_start_queue:
        spi_destroy_queue(master);
    err_init_queue:
        return ret;
    }

    static int spi_init_queue(struct spi_master *master) {
        struct sched_param param = { .sched_priority = MAX_RT_PRI0 - 1 };
        master->running = false;
        master->busy = false;
        init_kthread_worker(&master->kworker);
        master->kworker_task = kthread_run(kthread_worker_fn, &
            master->kworker, "%s", dev_name(&master->dev));
        if (IS_ERR(master->kworker_task)) {
            ... // error handling
        }
        init_kthread_work(&master->pump_messages, spi_pump_messages);
        /* Master config will indicate if this controller should run
        * the message pump with high (realtime) priority to reduce the
        * transfer latency on the bus by minimising the delay between
        * a transfer request and the scheduling of the message pump
        * thread. Without this setting the message pump thread wil
        * remain at default priority.
        */
        if (master->rt) {
            dev_info(&master->dev, "will run message pump with realtime priority\n");
            sched_setscheduler(master->kworker_task, SCHED_FIFO, &param);
        }
        return 0;
    }
    ```
   - 开启：`spi_start_queue()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    static int spi_start_queue(struct spi_master *master) {
        unsigned long flags;
        spin_lock_irqsave(&master->queue_lock, flags);
        if (master->running || master->busy) {
            spin_unlock_irqrestore(&master->queue_lock, flags);
            return -EBUSY;
        }
        master->running = true;
        master->cur_msg = NULL;
        spin_unlock_irqrestore(&master->queue_lock, flags);
        queue_kthread_work(&master->kworker, &master->pump_messages);
        return 0;
    }
    ```
   - 停止：`spi_destroy_queue()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    static int spi_destroy_queue(struct spi_master *master) {
        int ret;
        ret = spi_stop_queue(master);
        /* flush_kthread_worker will block until all work is done.
        * If the reason that stop_queue timed out is that the work will
        * never finish, then it does no good to call flush/stop thread,
        * so return anyway.
        */
        if (ret) {
            dev_err(&master->dev, "problem destroying queue\n");
            return ret;
        }
        flush_kthread_worker(&master->kworker);
        kthread_stop(master->kworker_task);
        return 0;
    }

    static int spi_stop_queue(struct spi_master *master) {
        unsigned long flags;
        unsigned limit = 500;
        int ret = 0;
        spin_lock_irqsave(&master->queue_lock, flags);
        /* 
        * This is a bit lame (站不住脚的，无说服力的), but is optimized
        * for the common execution path. A wait_queue on the master->busy
        * could be used, but then the common execution path (pump_messages)
        * wold be required to call wake_up or friends on every SPI message.
        * Do this instead.
        */
        while ((!list_empty(&master->queue) || master->busy) && limit--) {
            spin_unlock_irqrestore(&master->queue_lock, flags);
            usleep_range(10000, 11000);
            spin_lock_irqsave(&master->queue_lock, flags);
        }
        if (!list_empty(&master->queue) || master->busy)
            ret = -EBUSY;
        else
            master->running = false;
        spin_unlock_irqrestore(&master->queue_lock, flags);
        if (ret) {
            dev_warn(&master->dev, "could not stop message queue\n");
            return ret;
        }
        return ret;
    }
    ```
2. *spi* 设备管理
   - `of_register_spi_devices()`：设备树解析
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /** 
     * of_register_spi_devices() -  Register child devices onto the SPI bus
     * @master: Pointer to spi_master device
     *
     * Register an spi_device for each child node of master node which
     * has a 'reg' property.
     */
    static void of_register_spi_devices(struct spi_master *master) {
        struct spi_device *spi;
        struct device_node *nc;
        if (!master->dev.of_node)
            return;
        for_each_available_child_of_node(master->dev.of_node, nc) {
            spi = of_register_spi_device(master, nc);
            if (IS_ERR(spi)) {
                ... // error handling
            }
        }
    }

    static struct spi_device *of_register_spi_device(struct spi_master *master, struct device_node *nc) {
        struct spi_device *spi;
        int rc;
        u32 value;
        // Alloc an spi_device
        spi = spi_alloc_device(master);
        if (!spi) {
            ... // error handling
        }
        // Select device driver
        rc = of_modalias_node(nc, spi->modalias, sizeof(spi->modalias));
        if (rc < 0) {
            ... // error handling
        }
        rc = of_property_read_u32(nc, "reg", &value);
        if (rc) {
            ... // error handling
        }
        spi->chip_select = value;
        // Mode (clock phase/polarity/etc.)
        if (of_find_property(nc, "spi-cpha", NULL))
            spi->mode |= SPI_CPHA;
        if (of_find_property(nc, "spi-cpol", NULL))
            spi->mode |= SPI_CPOL;
        if (of_find_property(nc, "spi-cs-high", NULL))
            spi->mode |= SPI_CS_HIGH;
        if (of_find_property(nc, "spi-3wire", NULL))
            spi->mode |= SPI_3WIRE;
        if (of_find_property(nc, "spi-lsb-first", NULL))
            spi->mode |= SPI_LSB_FIRST;
        // Device DUAL/QUAD mode
        if (!of_property_read_u32(nc, "spi-tx-bus-width", &value)) {
            switch (value) {
            case 1:
                break;
            case 2:
                spi->mode |= SPI_TX_DUAL;
            case 4:
                spi->mode |= SPI_TX_QUAD;
            default:
                dev_warn(&master->dev, "spi-tx-bus-width %d not supported\n", value);
                break;
            }
        }
        if (!of_property_read_u32(nc, "spi-rx-bus-width, &value")) {
            switch (value) {
                ... // the same with above: SPI_RX_DUAL, SPI_RX_QUAD
            }
        }
        // Device speed
        rc = of_property_read_u32(nc, "spi-max-frequency", &value);
        if (rc) {
            ... // error handling
        }
        spi->max_speed_hz = value;
        // IRQ
        spi->irq = irq_of_parse_and_map(nc, 0);
        // Store a pointer to the node in the device structure
        of_node_get(nc);
        spi->dev.of_node = nc;
        // Register the new device
        rc = spi_add_device(spi);
        if (rc) {
            ... // error handling
        }
        return spi;
    err_out:
        spi_dev_put(spi);
        return ERR_PTR(rc);
    }
    ```
   - `spi_setup()`：检查当前 *spi* 设备是否能够与其 *spi master* 兼容
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /** 
     * spi_setup - setup SPI mode and clock rate 
     * @spi: the device whose settings are being modified
     * Context: can sleep, and no requests are queued to the device
     *
     * SPI protocol drivers may need to update the transfer mode if
     * the device doesn't work with its default. They may likewise (同样)
     * need to update clock rates or word sizes from intial values.
     * This function changes those settings, and must be called from
     * a context that can sleep. Except for SPI_CS_HIGH, which takes
     * effect immediately, the changes take effect the next time the
     * device is selected and data is transferred to or from it. When
     * this function returns, the spi device is deselected.
     *
     * Note that this call will fail if the protocol driver specifies
     * an option that the underlying controller or its drivers does
     * not support. For example, not all hardware supports wire transfers
     * using nine bit words, LSB-first wire encoding, or active-high
     * chipselects.
     */
    int spi_setup(struct spi_device *spi) {
        unsigned bad_bits, ugly_bits;
        int status = 0;
        /* check mode to prevent that DUAL and QUAD set at the 
         * same time.
         */
        if (((spi->mode & SPI_TX_DUAL) && (spi->mode && SPI_TX_QUAD)) || 
            ((spi->mode & SPI_RX_DUAL) && (spi->mode & SPI_RX_QUAD))) {
            dev_err(&spi->dev, "setup: can not select dual and quad at the same time\n");
            return -EINVAL;
        }
        /* if it is SPI_3WIRE mode, DUAL and QUAD should be forbidden */
        if ((spi->mode & SPI_3WIRE) && (spi->mode & 
            (SPI_TX_DUAL | SPI_TX_QUAD | SPI_RX_DUAL | SPI_RX_QUAD)))
            return -EINVAL;
        /* help drivers fail *cleanly* when they need options
         * that aren't supported with their current master
         */
        bad_bits = spi->mode & ~spi->master->mode_bits;
        ugly_bits = bad_bits & (SPI_TX_DUAL | SPI_TX_QUAD | SPI_RX_DUAL | SPI_RX_QUAD);
        if (ugly_bits) {
            dev_warn(&spi->dev, "setup: ignoring unsupported mode  bits %x\n", ugly_bits);
            spi->mode &= ~ugly_bits;
            bad_bits &= ~ugly_bits;
        }
        if (bad_bits) {
            dev_err(&spi->dev, "setup: unsupported mode bits %x\n", bad_bits);
            return -EINVAL;
        }
        if (!spi->bits_per_word)
            spi->bits_per_word = 8;
        if (!spi->max_speed_hz)
            spi->max_speed_hz = spi->master->max_speed_hz;
        spi_set_cs(spi, false);
        if (spi->master->setup)
            status = spi->master->setup(spi);
        dev_dbg(&spi->dev, "setup mode %d, %s%s%s%s%u bits/w, %u Hz max --> %d\n",
            (int) (spi->mode & (SPI_CPOL | SPI_CPAH)),
            (spi->mode & SPI_CS_HIGH) ? "cs_high, " : "",
            (spi->mode & SPI_LSB_FIRST) ? "lsb, " : "", 
            (spi->mode & SPI_3WIRE) ? "3wire, " : "",
            (spi->mode & SPI_LOOP) ? "loopback" : "",
            spi->bits_per_word, spi->max_speed_hz, status);
        return status;
    }
    EXPORT_SYMBOL_GPL(spi_setup);
    ```
   - `spi_set_cs()`：*spi* 设备的片选信号处理
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    static void spi_set_cs(struct spi_device *spi, bool enable) {
        if (spi->mode & SPI_CS_HIGH)
            enable = !enable;
        if (spi->cs_gpio >= 0)
            gpio_set_value(spi->cs_gpio, !enable);
        else if (spi->master->set_cs)
            spi->master->set_cs(spi, !enable);
    }
    ```
3. *msg* 处理
   - `spi_queued_transfer()` - `spi_master->transfer()` 的 *spi core* 实现
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /**
     * spi_queued_transfer - transfer function for queued transfers 
     * @spi: spi device which is requesting transfer
     * @msg: spi message which is to handled is queued to driver queue
     */
    static int spi_queued_transfer(struct spi_device *spi, struct spi_message *msg) {
        return __spi_queued_transfer(spi, msg, true);
    }

    static int __spi_queued_transfer(struct spi_device *spi, struct spi_message *msg, bool need_pump) {
        struct spi_master *master = spi->master;
        unsigned long flags;

        spin_lock_irqsave(&master->queue_lock, flags);

        if (!master->running) {
            spin_unlock_irqrestore(&master->queue_lock, flags);
            return -ESHUTDOWN;
        }
        msg->actual_length = 0;
        msg->status = -EINPROGRESS;

        list_add_tail(&msg->queue, &master->queue);
        if (!master->busy && need_pump)
            queue_kthread_work(&master->kworker, &master->pump_messages);

        spin_unlock_irqrestore(&master->queue_lock, flags);
        return 0;
    }
    ```
    - `spi_queued_transfer()` 函数的作用就是将 *spi_message* 挂到 *spi_master* 的消息队列上，并在必要时唤醒 *pump* 线程以进行 *spi_message* 的发送。
   - `spi_transfer_one_message()` - `spi_master->transfer_one_message()` 的 *spi core* 实现
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /* 
     * spin_transfer_one_message - Default implementation of transfer_one_message() 
     * 
     * This is a standard implementation of transfer_one_message() for drivers
     * which implement a transfer_one() operation. It provides standard handling
     * of delays and chip select management.
     */
    static int spi_transfer_one_message(struct spi_master *master, struct spi_message *msg) {
        struct spi_transfer *xfer;
        bool keep_cs = false;
        int ret = 0;
        unsigned long ms = 1;

        spi_set_cs(msg->spi, true);

        list_for_each_entry(xfer, &msg->transfers, transfer_list) {
            trace_spi_transfer_start(msg, xfer);

            if (xfer->tx_buf || xfer->rx_buf) {
                reinit_completion(&master->xfer_completion);

                ret = master->transfer_one(master, msg->spi, xfer);
                if (ret < 0) {
                    dev_err(&msg->spi->dev, "SPI transfer failed: %d\n", ret);
                    goto out;
                }
                if (ret > 0) {
                    ret = 0;
                    ms = xfer->len * 8 * 1000 / xfer->speed_hz;
                    ms += ms + 100;     /* some tolerance */
                    ms = wait_for_completion_timeout(&master->xfer_completion, msecs_to_jiffies(ms));
                }

                if (ms == 0) {
                    dev_err(&msg->spi->dev, "SPI transfer timed out\n");
                    msg->status = -ETIMEDOUT;
                }
            } else {
                if (xfer->len)
                    dev_err(&msg_spi_dev, "Bufferless transfer has length %u\n", xfer->len);
            }

            trace_spi_transfer_stop(msg, xfer);

            if (msg->status != -EINPROGRESS)
                goto out;
            if (xfer->delay_usecs)
                udelay(xfer->delay_usecs);
            if (xfer->cs_change) {
                if (list_is_last(&xfer->transfer_list, &msg->transfers)) {
                    keep_cs = true;
                } else {
                    spi_set_cs(msg->spi, false);
                    udelay(10);
                    spi_set_cs(msg->spi, true);
                }
            }
            msg->actual_length += xfer->len;
        }
    out:
        if (ret != 0 || !keep_cs)
            spi_set_cs(msg->spi, false);
        if (msg->status == -EINPROGRESS)
            mgs->status = ret;
        if (msg->status && master->handler_err)
            master->handle_err(master, msg);
        
        spi_finalize_current_message(master);
        
        return ret;
    }
    ```
    - [ ] trace_spi_message_xxx 相关内容
    - [ ] 延迟函数相关笔记
    - [ ] dma 相关内容
    - 提供了标准的延迟和片选的管理；
   - `spi_pump_messages()`：处理 *spi master* 上的 *msg* 队列
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /**
     * spi_pump_messages - kthread work function which processes spi message queue 
     * @work: pointer to kthread work struct contained in the master struct
     */
    static void spi_pump_messages(struct kthread_work *work) {
        struct spi_master *master = container_of(work, struct spi_master, pump_messages);
        __spi_pump_messages(master, true);
    }
    ```
   - `__spi_pump_messages()`：*spi master* 的 *msg* 队列处理，根据提供的参数 `in_kthread` 区分当前环境是否在 *pump kthread* 中。
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /**
     * __spi_pump_messages - function which processes spi message queue 
     * @master: master to process queue for
     * @in_kthread: true if we are in the context of the message pump thread
     *
     * This function checks if there is any spi message in the queue that needs processing
     * and if so call out to the driver to initialize hardware and transfer each message.
     *
     * Note that it is called both from the kthread itself and also from inside spi_sync().
     * the queue extraction handling at the top of the function should deal with this safely.
     */
    static void __spi_pump_messages(struct spi_master *master, bool in_kthread) {
        unsigned long flags;
        bool was_busy = false;
        int ret;
        /* Lock queue */
        spin_lock_irqsave(&master->queue_lock, flags);
        /* Make sure we are not already running a message */
        if (master->cur_msg) {
            spin_unlock_irqrestore(&master->queue_lock, flags);
            return;
        }
        /* If another context is idling the device then defer */
        if (master->idling) {
            queue_kthread_work(&master->kworker, &master->pump_messages);
            spin_unlock_irqrestore(&master->queue_lock, flags);
            return;
        }
        /* Check if the queue is idle */
        if (list_empty(&master->queue) || !master->running) {
            if (!master->busy) {
                spin_unlock_irqrestore(&master->queue_lock, flags);
                return;
            }
            /* Only do teardown in the thread */
            if (!in_kthread) {
                queue_kthread_work(&master->kworker, &master->pump_messages);
                spin_unlock_irqrestore(&master->queue_lock, flags);
                return;
            }
            master->busy = false;
            master->idling = true;
            spin_unlock_irqrestore(&master->queue_lock, flags);
            kfree(master->dummy_rx);
            master->dummy_rx = NULL;
            kfree(master->dummy_tx);
            master->dummy_tx = NULL;
            if (master->unprepare_transfer_hardware && 
                master->unprepare_transfer_hardware(master)) {
                    dev_err(&master->dev, "failed to unprepare transfer hardware\n");
                }
            if (master->auto_runtime_pm) {
                pm_runtime_mark_last_busy(master->dev.parent);
                pm_runtime_put_autosuspend(master->dev.parent);
            }
            trace_spi_master_idle(master);
            spin_lock_irqsave(&master->queue_lock, flags);
            master->idling = false;
            spin_unlock_irqrestore(&master->queue_lock, flags);
            return;
        }
        /* Extract head of queue */
        master->cur_msg = list_first_entry(&master->queue, struct spi_message, queue);
        list_del_init(&master->cur_msg->queue);
        if (master->busy)
            was_busy = true;
        else
            master->busy = true;
        spin_unlock_irqrestore(&master->queue_lock, flags);
        if (!was_busy && master->auto_runtime_pm) {
            ret = pm_runtime_get_sync(master->dev.parent);
            if (ret < 0) {
                dev_err(&master->dev, "failed to power device: %d\n", ret);
                return;
            }
        }
        if (!was_busy)
            trace_spi_master_busy(master);
        if (!was_busy && master->prepare_transfer_hardware) {
            ret = master->prepare_transfer_hardware(master);
            if (ret) {
                dev_err(&master->dev, "failed to prepare transfer hardware\n");
                if (master->auto_runtime_pm)
                    pm_runtime_put(master->dev.parent);
                return;
            }
        }
        trace_spi_message_start(master->cur_msg);
        if (master->prepare_message) {
            ret = master->prepare_message(master, master->cur_msg);
            if (ret) {
                dev_err(&master->dev, "failed to prepare message: %d\n", ret);
                master->cur_msg->status = ret;
                spi_finalize_current_message(master);
                return;
            }
            master->cur_msg_prepared = true;
        }
        ret = spi_map_msg(master, master->cur_msg);
        if (ret) {
            master->cur_msg->status = ret;
            spi_finalize_current_message(master);
            return;
        }
        ret = master->transfer_one_message(master, master->cur_msg);
        if (ret) {
            dev_err(&master->dev, "failed to transfer one message from queue\n");
            return;
        }
    }
    ```
    - [ ] 如果 kwork 已经在 kworker 上，则函数 queue_kthread_work 不会将 kwork 再次放到 kworker 上。由于在 pump 线程上只有 master->pump_messages 对应的 func 即 spi_pump_messages 在运行...那 queue_kthread_work 的调用时机有哪些？或者说 spi core 是如何保证的 master->pump_messages 在反复地被 queue_kthread_work 时不失败的？或者说不在乎成功与否，只需要将 message 挂到 master 队列上即可？
    - 有 spi 通信需求时，调用 transfer->...-> queue_kthread_work() 让 spi_pump_messages 在 pump 线程上运行。所有的 message 都会挂到 master 的队列上，而 spi_pump_messages 处理这里队列上的传输需求。根据 __spi_queued_transfer，在提交传输需求时，只有当 spi 控制器不 busy 且当前 message 需要在 pump 线程上传输才会调用 queue_kthread_work。那也就是说，spi_pump_messages 需要处理的 message 不光是当前提交的，也包括之前由于 spi 控制器 busy 等原因挂在队列上但没有处理的 message。
    - 该函数从三方面进行分析：整体逻辑、idling 的处理以及消息队列的处理，相关流程图见 `image.drawio:spi-__spi_pump_message`。
    - [ ] 收尾工作中，`if (!in_kthread) {...}` 的作用是什么？为什么要这样做？
   - `spi_map_msg()` 和 `spi_unmap_msg()`：*msg* 的中所有 *transfer* 的 *DMA* 映射和取消映射操作
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    #ifdef CONFIG_HAS_DMA
    static int spi_map_msg(struct spi_master *master, struct spi_message *msg) {
        struct spi_transfer *xfer;
        void *tmp;
        unsigned int max_tx, max_rx;
        if (master->flags & (SPI_MASTER_MUST_RX | SPI_MASTER_MUST_TX)) {
            max_tx = 0;
            max_rx = 0;
            list_for_each_entry(xfer, &msg->transfers, transfer_list) {
                if ((master->flags & SPI_MASTER_MUST_TX) && !xfer->tx_buf)
                    max_tx = max(xfer->len, max_tx);
                if ((master->flags & SPI_MASTER_MUST_RX) && !xfer->rx_buf)
                    max_rx = max(xfer->len, max_rx); 
            }
            if (max_tx) {
                tmp = krealloc(master->dummy_tx, max_tx, GFP_KERNEL | GFP_DMA);
                if (!tmp)
                    return -ENOMEM;
                master->dummy_tx = tmp;
                memset(tmp, 0, max_tx);
            }
            if (max_rx) {
                tmp = krealloc(master->dummy_rx, max_rx, GFP_KERNEL | GFP_DMA);
                if (!tmp)
                    return -ENOMEM;
                master->dummy_rx = tmp;
            }
            if (max_tx || max_rx) {
                list_for_each_entry(xfer, &msg->transfers, transfer_list) {
                    if (!xfer->tx_buf)
                        xfer->tx_buf = master->dummy_tx;
                    if (!xfer->rx_buf)
                        xfer->rx_buf = master->dummy_rx;
                }
            }
        }
        return __spi_map_msg(master, msg);
    }

    static int __spi_map_msg(struct spi_master, struct spi_message *msg) {
        struct device *tx_dev, *rx_dev;
        struct spi_transfer *xfer;
        int ret;
        if (!master->can_dma)
            return 0;
        tx_dev = master->dma_tx->device->dev;
        rx_dev = master->dma_rx->device->dev;
        list_for_each_entry(xfer, &msg->transfers, transfer_list) {
            if (!master->can_dma(master, msg->spi, xfer))
                continue;
            if (xfer->tx_buf != NULL) {
                ret = spi_map_buf(master, tx_dev, &xfer->tx_sg, 
                    (void *)xfer->tx_buf, xfer->len, DMA_TO_DEVICE);
                if (ret != 0)
                    return ret;
            }
            if (xfer->rx_buf != NULL) {
                ret = spi_map_buf(master, rx_dev, &xfer->rx_sg,
                    (void *)xfer->rx_buf, xfer->len, DMA_TO_FROM_DEVICE);
                if (ret != 0) {
                    spi_unmap_buf(master, tx_dev, &xfer->tx_gs, DMA_TO_DEVICE);
                    return ret;
                }
            }
        }
        master->cur_msg_mapped = true;
        return 0;
    }

    static int spi_map_buf(struct spi_master *master, struct device *dev, 
        struct sg_table *sgt, void *buf, size_t len, enum dam_data_direction dir) {
        const bool vmalloced_buf = is_vmalloc_addr(buf);
        const int desc_len = vmalloced_buf ? PAGE_SIZE : master->max_dma_len;
        const int sgs = DIV_ROUND_UP(len, desc_len);
        struct page *vm_page;
        struct scatterlist *sg;
        void *sg_buf;
        size_t min;
        int i, ret;

        ret = sg_alloc_table(sgt, sgs, GFP_KERNEL);
        if (ret != 0)
            return ret;
        sg = &sgt->sgl[0];
        for (i = 0; i < sgs; i++) {
            min = min_t(size_t, len, desc_len);
            if (vmalloced_buf) {
                vm_page = vmalloc_to_page(buf);
                if (!vm_page) {
                    sg_free_table(sgt);
                    return -ENOMEM;
                }
                sg_set_page(sg, vm_page, min, offset_in_page(buf));
            } else {
                sg_buf = buf;
                sg_set_buf(sg, sg_buf, min);
            }
            buf += min;
            len -= min;
            sg = sg_next(sg);
        }
        ret = dma_map_sg(dev, sgt->sgl, sgt->nents, dir);
        if (!ret)
            return = -ENOMEM;
        if (ret < 0) {
            sg_free_table(sgt);
            return ret;
        }
        sgt->nents = ret;
        return 0;
    }

    static int spi_unmap_msg(struct spi_master *master, struct spi_message *msg) {
        struct spi_transfer *xfer;
        struct device *tx_dev, *rx_dev;
        if (!master->cur_msg_mapped || !master->can_dma)
            return 0;
        tx_dev = master->dma_tx->device->dev;
        rx_dev = master->dma_rx->device->dev;
        list_for_each_entry(xfer, &msg->transfers, transfer_list) {
            /* Restore the original value of tx_buf or rx_buf if 
             * they are NULL.
             */
            if (xfer->tx_buf == master->dummy_tx)
                xfer->tx_buf = NULL;
            if (xfer->rx_buf == master->dummy_rx)
                xfer->rx_buf = NULL;
            if (!master->can_dma(master, msg->spi, xfer))
                continue;
            spi_unmap_buf(master, rx_dev, &xfer->rx_sg, DMA_FROM_DEVICE);
            spi_unmap_buf(master, tx_dev, &xfer->tx_sg, DMA_TO_DEVICE);
        }
        return 0;
    }

    static void spi_unmap_buf(struct spi_master *master, struct dev *dev, 
        struct sg_table *sgt, enum dma_data_direction dir) {
        if (sgt->orig_nents) {
            dma_unmap_sg(dev, sgt->sgl, sgt->orig_nents, dir);
            sg_free_table(sgt);
        }
    }
    ```
    - 位于 *vmalloc* 区域的地址并不是不能进行 *DMA* 操作，而是需要按页拆分，以页为单位进行 *DMA*（通过 `vmalloc()` 分配的内存在虚拟地址空间连续，但不保证在物理地址空间上连续；按页分配，页帧和页帧之间不一定连续，但页帧内是连续的）。
    - 根据需要 *DMA* 的 *buf* 是否位于 *vmalloc* 区域，计算单次 *DMA* 操作允许搬运的最大数据长度为 `desc_len`。
    - [ ] 为什么不需要从 `PAGE_SIZE` 和 `master->max_dma_len` 取最小值？
    - 需要将 *buf* 拆分成 *scatterlist* 的个数-`sgs`。
    - 在确定了需要将 *buf* 拆分的数量、单次传输的数据长度、*buf* 是否位于 *vmalloc* 区域等信息后，设置 *sg_table* 中 *scatterlist* 的数量并依次对各个 *scatterlist* 所包含的地址和传输长度进行设置。
    - 设置完成后，通过宏 `dma_map_sg()` 进行统一的映射操作。
    - [ ] 为什么 `dma_map_sg` 返回 0 后不直接调用和返回值小于 0 时一样的操作呢？
   - `spi_finalize_current_message()`：每个 *msg* 被处理完成后（成功或失败）时调用此函数，以继续处理 *master* 中剩余的 *msg*，或者处理一些收尾工作。
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /** spi_finalize_current_message() - the current message is complete. 
     * @master: the master to return the message to
     *
     * Called by the driver to notify the core that the message in the front
     * of the queue is complete and can be removed from the queue.
     */
    void spi_finalize_current_queue(struct spi_master *master) {
        struct spi_message *mesg;
        unsigned long flags;
        int ret;

        spin_lock_irqsave(&master->queue_lock, flags);
        mesg = master->cur_msg;
        spin_unlock_irqrestore(&master->queue_lock, flags);

        spin_unmap_msg(master, mesg);

        if (master->cur_msg_prepared && master->unprepare_message) {
            ret = master->unprepare_message(master, mesg);
            if (ret) {
                dev_err(&master->dev, "failed to unprepare message: %d\n", ret);
            }
        }

        spin_lock_irqsave(&master->queue_lock, flags);
        master->cur_msg = NULL;
        master->cur_msg_prepared = false;
        queue_kthread_work(&master->kworker, &master->pump_messages);
        spin_unlock_irqrestore(&master->queue_lock, flags);

        trace_spi_message_done(mesg);

        mesg->state = NULL;
        if (mesg->complete)
            mesg->complete(mesg->context);
    }
    EXPORT_SYMBOL_GPL(spi_finalize_current_message);
    ```

#### 相关数据结构及其关系
*spi_master、spi_device、spi_message、spi_transfer*
1. `struct spi_message`：
    ```c
    // version: 4.1.15
    // include/linux/spi/spi.h
    /**
     * struct spi_message - one multi-segment SPI transaction 
     * @transfers: list of transfer segments in this transaction
     * @spi: SPI device to which the transaction is queued
     * @is_dma_mapped: if true, the caller provided both dma and cpu
     *  virtual addresss for each transfer buffer
     * @complete: called to report transaction completions
     * @context: the argument to complete() when it's called
     * @frame_length: the total number of bytes in the message
     * @actual_length: the total number of bytes that were transferred
     *  in all successful segments
     * @status: zero for success, else negative errno
     * @queue: for use by whichever driver currently owns the message
     * @state: for use by whichever driver currently owns the message
     *
     * A @spi_message is used to execute an atomic sequence of data
     * transfers, each represented by a struct spi_transfer. The sequence
     * is "atomic" in the sense that no other spi_message may use that
     * SPI bus until that suquence completes. On some systems, many
     * such sequences can execute as single programmed DMA transfer.
     * On all systems, these messages are queued, and might complete
     * after transactions to other devices. Messages sent to a given
     * spi_device are always executed in FIFO order.
     *
     * The code that submits an spi_message (and its spi_transfers)
     * to the lower layers is responsible for managing its memory.
     * Zero-Initialized every field you don't set up explicitly, to
     * insulate against future API updates. After you submit a message
     * and its transfers, ignore them until its completion callback.
     */
    struct spi_message {
        struct list_head transfers;
        struct spi_device *spi;
        unsigned is_dma_mapped:1;
        /* REVISIT: we might want a flag affecting the behavior of
         * the last transfer ... allowing things like "read 16 bit
         * length L" immediately followed by "read L byte". Basically 
         * imposing a specific message scheduling algorithm.
         *
         * Some controller drivers (message-at-a-time queue processing)
         * could provide that as their default scheduling algorithm.
         * But others (with multi-message pipelines) could need a flag
         * to tell them about such special cases.
         */
         /* completion is reported through a callback */
         void (*complete)(void *context);
         void *context;
         unsigned frame_length;
         unsigned actual_length;
         int status;
         /* for optional use by whatever driver currently owns the
          * spi_message ... between calls to spi_async and then later
          * complete(), that's the spi_master controller driver.
          */
         struct list_head queue;
         void *state;
    };
    ```
2. `struct spi_transfer`
    ```c
    /**
    * struct spi_transfer - a read/write buffer pair
    * @tx_buf: data to be written (dma-safe memory), or NULL
    * @rx_buf: data to be read (dma-safe memory), or NULL
    * @tx_dma: DMA address of tx_buf, if @spi_message.is_dma_mapped
    * @rx_dma: DMA address of rx_buf, if @spi_message.is_dma_mapped
    * @tx_nbits: number of bits used for writing. If 0 the default
    *      (SPI_NBITS_SINGLE) is used.
    * @rx_nbits: number of bits used for reading. If 0 the default
    *      (SPI_NBITS_SINGLE) is used.
    * @len: size of rx and tx buffers (in bytes)
    * @speed_hz: Select a speed other than the device default for this
    *      transfer. If 0 the default (from @spi_device) is used.
    * @bits_per_word: select a bits_per_word other than the device default
    *      for this transfer. If 0 the default (from @spi_device) is used.
    * @cs_change: affects chipselect after this transfer completes
    * @delay_usecs: microseconds to delay after this transfer before
    *	(optionally) changing the chipselect status, then starting
    *	the next transfer or completing this @spi_message.
    * @transfer_list: transfers are sequenced through @spi_message.transfers
    * @tx_sg: Scatterlist for transmit, currently not for client use
    * @rx_sg: Scatterlist for receive, currently not for client use
    *
    * SPI transfers always write the same number of bytes as they read.
    * Protocol drivers should always provide @rx_buf and/or @tx_buf.
    * In some cases, they may also want to provide DMA addresses for
    * the data being transferred; that may reduce overhead, when the
    * underlying driver uses dma.
    *
    * If the transmit buffer is null, zeroes will be shifted out
    * while filling @rx_buf.  If the receive buffer is null, the data
    * shifted in will be discarded.  Only "len" bytes shift out (or in).
    * It's an error to try to shift out a partial word.  (For example, by
    * shifting out three bytes with word size of sixteen or twenty bits;
    * the former uses two bytes per word, the latter uses four bytes.)
    *
    * In-memory data values are always in native CPU byte order, translated
    * from the wire byte order (big-endian except with SPI_LSB_FIRST).  So
    * for example when bits_per_word is sixteen, buffers are 2N bytes long
    * (@len = 2N) and hold N sixteen bit words in CPU byte order.
    *
    * When the word size of the SPI transfer is not a power-of-two multiple
    * of eight bits, those in-memory words include extra bits.  In-memory
    * words are always seen by protocol drivers as right-justified, so the
    * undefined (rx) or unused (tx) bits are always the most significant bits.
    *
    * All SPI transfers start with the relevant chipselect active.  Normally
    * it stays selected until after the last transfer in a message.  Drivers
    * can affect the chipselect signal using cs_change.
    *
    * (i) If the transfer isn't the last one in the message, this flag is
    * used to make the chipselect briefly go inactive in the middle of the
    * message.  Toggling chipselect in this way may be needed to terminate
    * a chip command, letting a single spi_message perform all of group of
    * chip transactions together.
    *
    * (ii) When the transfer is the last one in the message, the chip may
    * stay selected until the next transfer.  On multi-device SPI busses
    * with nothing blocking messages going to other devices, this is just
    * a performance hint; starting a message to another device deselects
    * this one.  But in other cases, this can be used to ensure correctness.
    * Some devices need protocol transactions to be built from a series of
    * spi_message submissions, where the content of one message is determined
    * by the results of previous messages and where the whole transaction
    * ends when the chipselect goes intactive.
    *
    * When SPI can transfer in 1x,2x or 4x. It can get this transfer information
    * from device through @tx_nbits and @rx_nbits. In Bi-direction, these
    * two should both be set. User can set transfer mode with SPI_NBITS_SINGLE(1x)
    * SPI_NBITS_DUAL(2x) and SPI_NBITS_QUAD(4x) to support these three transfer.
    *
    * The code that submits an spi_message (and its spi_transfers)
    * to the lower layers is responsible for managing its memory.
    * Zero-initialize every field you don't set up explicitly, to
    * insulate against future API updates.  After you submit a message
    * and its transfers, ignore them until its completion callback.
    */
    struct spi_transfer {
        /* it's ok if tx_buf == rx_buf (right?)
        * for MicroWire, one buffer must be null
        * buffers must work with dma_*map_single() calls, unless
        *   spi_message.is_dma_mapped reports a pre-existing mapping
        */
        const void	*tx_buf;
        void		*rx_buf;
        unsigned	len;

        dma_addr_t	tx_dma;
        dma_addr_t	rx_dma;
        struct sg_table tx_sg;
        struct sg_table rx_sg;

        unsigned	cs_change:1;
        unsigned	tx_nbits:3;
        unsigned	rx_nbits:3;
    #define	SPI_NBITS_SINGLE	0x01 /* 1bit transfer */
    #define	SPI_NBITS_DUAL		0x02 /* 2bits transfer */
    #define	SPI_NBITS_QUAD		0x04 /* 4bits transfer */
        u8		bits_per_word;
        u16		delay_usecs;
        u32		speed_hz;

        struct list_head transfer_list;
    };
    ```
    - `len` 是要进行传输的数据长度，*spi* 为全双工通信、一次通信中发送和接收的字节数都是一样的，因此也就没有发送长度和接收长度之分。
- 各数据结构间的关系见图 `Linux/Driver/subsys/image.drawio-spi_master_overview`。

### Master 驱动
#### 数据结构描述
*spi* 控制器驱动通常借助 *platform* 总线设备驱动来实现，在这里只关注 *spi* 控制器的数据结构描述，其驱动相关内容见 *platform* 笔记。
```c
// version: 4.1.15
// include/linux/spi/spi.h
/**
 * struct spi_master - interface to SPI master controller
 * @dev: device interface to this driver
 * @list: link with the global spi_master list
 * @bus_num: board-specific (and often SOC-specific) identifier for
 *  a given SPI controller.
 * @num_chipselect: chipselects are used to distinguish individual
 *  SPI slaves, and are numbered from zero to num_chipselects.
 *  each slave has a chipselect signal, but it's common that not 
 *  every chipselect is connected to a slave.
 * @dma_alignment: SPI controller constraint on DMA buffers alignment.
 * @mode_bits: flags understood by this controller driver
 * @bits_per_word_mask: A mask indicating which values of bits_per_word are
 *  supported by the driver. Bit n indicates that a bits_per_word n+1 is supported.
 *  If set, the SPI core will reject any transfer with an unsupported bits_per_word.
 *  If not set, this value is simply ignored, and it's up to the individual driver 
 *  to perform any validation.
 * @min_speed_hz: Lowest supported transfer speed
 * @max_speed_hz: Highest supported transfer speed
 * @flags: other constrains relevant to this driver
 * @bus_lock_spinlock: spinlock for SPI bus locking
 * @bus_lock_mutex: mutex for SPI bus locking
 * @bus_lock_flags: indicates that the SPI bus is locked for exclusive use
 * @setup: updates the devcie mode and clocking records used by a device's
 *  SPI controller; protocol code may call this. This must fail if an unrecognized
 *  or unsupported mode is requested. It's always safe to call this unless transfers
 *  are pending on the device whose settings are being modified.
 * @transfer: adds a message to the controller's transfer queue.
 * @cleanup: frees controller-specific state
 * @can_dma: determine whether this master supports DMA
 * @queued: whether this master is providing an internal message queue.
 * @kworker: thread struct for message pump (输送)
 * @kworker_task: pointer to task for message pump kworker thread
 * @pump_messages: work struct for scheduling work to the message pump
 * @queue_lock: spinlock to syncronise access to message queue
 * @queue: message queue
 * @idling: the device is entering idle state
 * @cur_msg: the currently in-flight message
 * @cur_msg_prepared: spi_prepare_message was called for the currently 
 *  in-flight message
 * @cur_msg_mapped: message has been mapped for DMA
 * @xfer_completion: used by core transfer_one_message()
 * @busy: message pump is busy
 * @running: message pump is running
 * @rt: whether this queue is set to run as a realtime task
 * @auto_runtime_pm: the core should ensure a runtime PM reference is held
 *  while the hardware is prepared, using the parent device for the spidev
 * @max_dma_len: Maximum length of a DMA transfer for the device.
 * @prepare_transfer_hardware: a message will soon arrive from the queue
 *  so that subsystem requests the driver to prepare the transfer hardware
 *  by issuing this call.
 * @transfer_one_message: the subsystem calls the driver to transfer a signle
 *  message while queuing transfers that arrive in the meantime. when the driver
 *  is finished with this message, it must call spi_finalize_currrent_message()
 *  so the subsystem can issue the next message.
 * @unprepare_transfer_hardware: there are currently no more messages on the queue
 *  so the subsystem notifies the driver that it may relax the hardware by issuing
 *  this call
 * @set_cs: set the logic level of the chip select line. May be called from interrupt
 *  context.
 * @prepare_message: set up the controller to transfer a single message, for example
 *  doing DMA mapping. Called from threaded context.
 * @transfer_one: transfer s single spi_transfer
 *  - return 0 if the transfer is finished.
 *  - return 1 if the transfer is still in progress. When the driver is finished with 
 *    this transfer it must call spi_finalize_current_transfer() so the subsystem can
 *    issue the next transfer. Note: transfer_one and transfer_one_message are mutually (互相、彼此)
 *    exclusive; when both are set, the generic subsystem does not call your transfer_one
 *    callback.
 * @handle_err: the subsystem calls the driver to handle an error that occurs in the
 *  generic implementation of transfer_one_message().
 * @unprepare_message: undo any work done by prepare_message().
 * @cs_gpios: Array of GPIOs to use as chip select lines; one per CS number. Any 
 *  individual value may be -ENOENT for CS lines that are not GPIOs (driven by the SPI
 *  controller itself).
 * @dma_tx: DMA transmit channel.
 * @dma_rx: DMA receive channel.
 * @dummy_rx: dummy receive buffer for full-duplex (全双工) devices
 * @dummy_tx: dummy receive buffer for full-duplex devices
 *
 * Each SPI master controller can communicate with one or more @spi_device children.
 * These make a small bus, sharing MOSI, MISO and SCK signals but not chip select signals.
 * Each device may be configured to use a different clock rate, since those shared signals
 * are ignored unless the chip is selected.
 *
 * The driver for an SPI controller manages access to those devices through a queue of
 * spi_message transactions, copying data between CPU memory and an SPI slave device.
 * For each such message it queues, it calls the message's completion function when
 * the transaction completes.
 */
struct spi_master {
    struct device dev;
    struct list_head list;
    /* other than negative (== assign one dynamically), bus_num is fully board-specific. 
     * usually that simplifies to being SOC-specific.
     * example: one SOC has three SPI controllers, numbered 0...2,
     * and one board's schematics might show it using SPI-2.
     * software would normally use bus_num=2 for that controller.
     */
    s16 bus_num;
    /* chipselects will be integral (必须的，不可或缺的) to many controllers; 
     * some others might use board-specific GPIOs. 
     */
    u16 num_chipselect;
    /* some SPI controllers pose alignment requirements on DMAable
     * buffers; let protocol drivers know about these requirements
     */
    u16 dma_alignment;
    /* spi_device.mode flags understood by this controller driver  */
    u16 mode_bits;
    /* bitmask of supported bits_per_word for transfers. */
    u32 bits_per_word_mask;
#define SPI_BPW_MASK(bits) BIT((bits) - 1)
#define SPI_BIT_MASK(bits) (((bits) == 32) ? ~0U : (BIT(bits) - 1))
#define SPI_BPW_RANGE_MASK(min, max) (SPI_BIT_MASK(max) - SPI_BIT_MASK(min - 1))
    /* limits on transfer speed */
    u32 min_speed_hz;
    u32 max_speed_hz;
    /* other constraints relevant to this driver. */
    u16 flags;
#define SPI_MASTER_HALF_DUPLEX BIT(0)   /* can't do full duplex */
#define SPI_MASTER_NO_RX       BIT(1)   /* can't do buffer read */
#define SPI_MASTER_NO_TX       BIT(2)   /* can't do buffer write */
#define SPI_MASTER_MUST_RX     BIT(3)   /* requires rx */
#define SPI_MASTER_MUST_TX     BIT(4)   /* requires tx */
    /* lock and mutex for SPI bus locking */
    spinlock_t bus_lock_spinlock;
    struct mutex bus_lock_mutex;
    /* flag indicating that the SPI bus is locked for exclusive use */
    bool bus_lock_flag;
    /* setup mode and clock, etc (spi driver may call many times). 
     * IMPORTANT: this may be called when transfer to another
     * device are active. DO NOT UPDATE SHARED REGISTERS in ways
     * which could break those transfer.
     */
    int (*setup)(struct spi_device *spi);
    /* bidirectional bulk transfer 
     * 
     * + The transfer() method may not sleep; its main role is
     *  just to add the message to the queue.
     * + For now there's no remove-from-queue operation, or any
     *  other request management
     * + To a given spi_device, message queueing is pure fifo
     *
     * + The master's main job is to process its message queue,
     *  selecting a chip then transferring data
     * + If there are multiple spi_device children, the i/o queue
     *  arbitration (仲裁) algorithm is unspecified (round robin (轮询), fifo,
     *  priority, reservations, preemption, etc)
     *
     * + Chipselect stays active during the entire message
     *  (unless modified by spi_transfer.cs_change != 0).
     * + The message transfer use clock and SPI mode parameters
     *  previously established by setup() for this device
     */
    int (*transfer)(struct spi_device *spi, struct spi_message *mesg);
    /* called on release() to free memory provided by spi_master */
    void (*cleanup)(struct spi_device *spi);
    /* Used to enable core support for DMA handling, if can_dma()
     * exists and returns true then the transfer will be mapped
     * prior to transfer_one() being called. The drivers should 
     * not modify or store xfer and dma_tx and dma_rx must be set
     * while the device is prepared.
     */
    bool (*can_dma)(struct spi_master *master, struct spi_device *spi, 
        struct spi_transfer *xfer);
    /* These hooks are for drivers that want to use the generic
     * master transfer queueing mechanism. If these are used, the
     * transfer() function above must NOT be specified by the driver.
     * Over time we except SPI drivers to be phased over to this API.
     */
    bool queued;
    struct kthread_worker kworker;
    struct task_struct kworker_task;
    struct kthread_work pump_messages;
    spinlock_t queue_lock;
    struct list_head queue;
    struct spi_message *cur_msg;
    bool idling;
    bool busy;
    bool running;
    bool rt;
    bool auto_runtime_pm;
    bool cur_msg_prepared;
    bool cur_msg_mapped;
    struct completion xfer_completion;
    size_t max_dma_len;
    int (*prepare_transfer_hardware)(struct spi_master *master);
    int (*transfer_one_message)(struct spi_master *master, struct spi_message *mesg);
    int (*unprepare_transfer_hardware)(struct spi_master *master);
    int (*prepare_message)(struct spi_master *master, struct spi_message *message);
    int (*unprepare_message)(struct spi_master *master, struct spi_message *message);
    /* These hooks are for drivers that use a generic implementation
     * of transfer_one_message() provided by the core.
     */
    void (*set_cs)(struct spi_device *spi, bool enable);
    int (*transfer_one)(struct spi_master *master, struct spi_device *spi, struct spi_transfer *transfer);
    void (*handle_err)(struct spi_master *master, struct spi_message *message);
    /* gpio chip select */
    int *cs_gpios;
    /* DMA channels for use with core dmaengine helpers */
    struct dma_chan *dma_tx;
    struct dma_chan *dma_rx;
    /* dummy data fro full duplex devices */
    void *dummy_rx;
    void *dummy_tx;
};
```
- `bits_per_word_mask`:
  - 参考：`bing-spi bits per word`、[spi word 宽度](https://www.cnblogs.com/wangwangever/p/17989480)、[linux基础之spi驱动程序理解](https://blog.csdn.net/liuhuahan/article/details/41773937)
  - 在 SPI 通信中，`bits_per_word` 表示一次传输中每个“字”的比特数，即在一次片选有效期间传输的位宽。常见值有 8 位或 16 位，但具体支持范围取决于 SPI 控制器驱动，有的可支持 1-32 位，也有的仅支持 8 位。
  - 在 `struct spi_master` 中的成员 `bits_per_word_mask` 用于表示支持的字长有哪些。如果该成员的 *n* 位被置位，则表示支持 `bits_per_word` 为 *n + 1* 的数据传输。如果该成员被设置，则 SPI 控制器会拒绝传输不支持的字长，即 `spi_mask->bits_per_work_mask & spi_device->bits_per_work` 的结果为 0。
  - 在 `struct spi_device` 中的成员 `bits_per_word` 用于指定默认字长，如果为 0 则默认使用 8 位。
  - 在用户空间，通过 *spidev* 设备节点，可使用 *ioctl* 命令：`SPI_IOC_WR_BITS_PER_WORD` 和 `SPI_IOC_RD_BITS_PER_WORD` 来设置或读取该值。
- `rt`: 表示在进行 *spi* 传输时，其对应的异步线程是否作为实时进程来进行调度。在 `spi_init_queue()` 函数中会根据此字段的值进行相应的处理。
- [ ] 自旋锁 bus_lock_spinlock 和互斥锁 bus_lock_mutex 同步的内容有哪些？它们有交集吗？
- `cur_msg`: 表示当前正在处理的 *message*，也可以通过该字段来判断 *spi* 控制器是否繁忙。
- `idling`: *spi* 控制器正在进入 *idle* 状态。
- `running` 和 `busy`: 前者表示当前 *spi* 控制器是否正在运行（是否可用），而后者则表示当前 *spi* 控制器是否正在进行数据传输。
- `transfer()` 将 *msg* 挂到对应的 *spi master* 上。`transfer_one_message()` 函数用于 *spi* 数据传输，实现一个 *msg* 的 *spi* 通信。*spi* 的数据会打包成 *spi_message*，*pump kthread* 在处理 *spi master* 上的 *msg* 时会通过调用此函数实现传输。
- *spi master* 驱动的核心就是申请、设置（初始化）、注册 *spi_master*。
- *spi master* 驱动作为 *platform_driver* 注册到 *platform* 总线上。
```c
// 申请
/* @dev: 设备，一般是 platform_device 中的 dev 成员变量。 
 * @size: 私有数据大小，可以通过 spi_master_get_devdata() 获取私有数据
 * @return: 申请到的 spi_master
 */
struct spi_master *spi_alloc_master(struct device *dev, unsigned size);

// 释放
void spi_master_put(struct spi_master *master);

// 注册
// 0-成功；负值-失败
int spi_register_master(struct spi_master *master);
// spi_bitbang_start

// 注销
void spi_unregister_master(struct spi_master *master);
// spi_bitbang_stop
```

#### spi-bitbang
##### 概述
以下内容来自于 kimi 的对话：
> 1. "*Bitbang*" 是什么意思？
> *Bitbang* 是一种**软件模拟硬件协议**的技术。
> 在 *spi* 的语境下，它指的是：**不使用专用的 *spi* 硬件控制器，而是通过软件直接控制 *GPIO* 引脚的电平翻转，来模拟 *spi* 的时钟（*SCLK*）、数据输出（*MOSI*）和数据输入（*MISO*）时序。**
```
硬件 SPI:  SPI 控制器 ──→ 自动产生 SCLK/MOSI/MISO 时序
Bitbang:   CPU 软件 ──→ GPIO 引脚 ──→ 手动翻转电平模拟时序
```
> *Bitbang* 的优势是**灵活性**（任何 *GPIO* 都能当 *spi* 用），劣势是**速度慢、*CPU* 占用高**（每个 *bit* 都需要 *CPU* 干预）。


> 2. *spi_bitbang* 是什么？
> *spi_bitbang* 是 *Linux spi* 子系统中的一个**中间层框架**，位于 `drivers/spi/spi-bitbang.c`。
> 2.1 功能
> 它为"用软件模拟 *spi*"的驱动提供**通用基础设施**：
| 功能 | 说明 |
|------|------|
| ***Message* 队列管理** | 继承 *spi* 核心层的 *message pump* 机制，管理 *spi_message* 的排队和调度 |
| **片选（*CS*）管理** | 自动控制 *CS* 引脚的拉低/拉高 |
| **时序控制** | 处理 *spi_transfer* 的速度（*speed_hz*）、延迟（*delay_usecs*）等 |
| **半双工/全双工抽象** | 将上层驱动的 *tx_buf*/*rx_buf* 转换为底层的移位操作 |
| **提供回调接口** | 要求底层驱动只需实现"发送/接收一个字节（或一个 *word*）"的硬件操作 |

> 2.2 核心接口
> 底层驱动（如 *spi_gpio*）需要向 *spi_bitbang* 注册一个 `setup_transfer()` 和 `txrx_bufs()`（或 `txrx_word()`）回调。

> 3. 在 SPI 子系统中的位置
```
┌─────────────────────────────────────┐
│         SPI 核心层 (spi.c)           │  ← 提供 spi_master/spi_device/spi_transfer
│    队列管理、设备模型、用户态接口       │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│      spi_bitbang 框架层              │  ← 软件模拟 SPI 的通用逻辑
│   (drivers/spi/spi-bitbang.c)       │
│   - Message pump 适配               │
│   - CS 控制                         │
│   - 时序/delay 处理                  │
│   - 调用底层 txrx 回调               │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│      具体 Bitbang 驱动实现            │  ← 控制实际引脚
│   - spi_gpio (GPIO 模拟)            │
│   - spi_butterfly (并口模拟)         │
│   - 各平台自定义 GPIO SPI            │
└─────────────────────────────────────┘
```
> **位置总结**：*spi_bitbang* 是 ***spi* 核心层与具体 *GPIO*/软件实现之间的适配层**。它让底层驱动不需要重复实现 *message* 队列、*CS* 切换等通用逻辑，只需关注"如何在硬件上移位一个 *bit*"。

> 4. *spi_bitbang* 与 *spi_gpio* 的关系
> ***spi_gpio* 是基于 *spi_bitbang* 框架的具体驱动实现**。
| 对比项 | *spi_bitbang* | *spi_gpio* |
|--------|--------------|-----------|
| **角色** | 框架/中间层 | 具体驱动 |
| **文件** | `drivers/spi/spi-bitbang.c` | `drivers/spi/spi-gpio.c` |
| **功能** | 提供通用 *message* 处理、时序、*CS* 管理 | 用 *GPIO* 引脚模拟 *SCLK/MOSI/MISO/CS* |
| **复用性** | 可被任何软件模拟 *spi* 的驱动使用 | 只能用于 *GPIO* 场景 |

> 5. 代码关系
> *spi_gpio* 在初始化时会：
>   a. 分配 `struct spi_bitbang`
>   b. 设置 `bitbang->txrx_bufs = spi_gpio_txrx_buf;`（真正的 GPIO 翻转函数）
>   c. 调用 `spi_bitbang_start(bitbang)`，让 *spi_bitbang* 框架接管后续的 *message pump*
```c
// spi-gpio.c 简化示意
static int spi_gpio_probe(struct platform_device *pdev)
{
    struct spi_bitbang *bitbang;
    ...
    bitbang->master = master;
    bitbang->chipselect = spi_gpio_chipselect;
    bitbang->txrx_bufs = spi_gpio_txrx_buf;  // ← 真正的 GPIO 操作
    
    status = spi_bitbang_start(bitbang);      // ← 交给框架
    ...
}
```

> 6. 总结
> ***spi_bitbang* 是 *Linux spi* 子系统中为"软件模拟 *spi*"提供的通用框架层，负责 *message* 队列、片选和时序管理；*spi_gpio* 是基于它的具体驱动，用 *GPIO* 引脚实现 *spi* 时序。"*Bitbang*" 本身就是"用软件逐位翻转 *GPIO* 来模拟硬件协议"的意思。**

##### 数据结构描述
1. `struct spi_bitbang`
    ```c
    // version: 4.1.15
    // include/linux/spi/spi_bitbang.h
    struct spi_bitbang {
        spinlock_t lock;
        u8 busy;
        u8 use_dma;
        u8 flags;       // extra spi->mode support
        struct spi_master *master;
        /* setup_transfer() changes clock and/or wordsize to match
         * settings for this transfer; zeroes restore defaults from
         * spi device
         */
        int (*setup_transfer)(struct spi_device *spi, struct spi_transfer *t);
        void (*chipselect)(struct spi_device *spi, in is_on);
    #define BITBANG_CS_ACTIVE   1   /* normally nCS, active low*/
    #define BITBANG_CS_INACTIVE 0
        /* txrx_bufs() may handle dma mapping for transfers that
         * don't already have one (transfer.{tx,rx}_dma is zero),
         * or use PIO
         */
        int (*txrx_bufs)(struct spi_device *spi, struct spi_transfer *t);
        /* txrx_word[SPI_MODE_*]() just looks like a shift register */
        u32 (*txrx_word[4])(struct spi_device *spi, unsigned nsecs, u32 word, u8 bits);
    }
    ```
    - *spi_bitbang has a spi_master*
    - `txrx_word[4]` 对应 *spi* 协议中的四种模式，即 *CPOL* 和 *CPAH* 的四种组合。*spi_bitbang* 框架提供四种模式的接口，`struct spi_bitbang` 实例可自行实现具体函数，也可使用 *spi_bitbang* 提供的默认实现。在与具体 *spi* 设备通信时，会根据设备的模式要求，从这个集合中选择对应的函数进行通信。
    - `txrx_bufs()` 实现对 *spi* 设备的通信，通信内容为参数 `t`，它会继续调用 *spi device* 对应的 `spi_bitbang_cs->txrx_bufs()` 来进行通信。
2. `struct spi_bitbang_cs`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    /* 
     * FIRST PART (OPTIONAL): word-at-a-time spi_transfer support.
     * Use this for GPIO or shift-register level hardware APIs.
     *
     * spi_bitbang_cs is in spi_device->controller_state, which is
     * unavailable to glue code. These bitbang setup() and cleanup()
     * routines are always used, though maybe they're called from
     * controller-aware code.
     *
     * chipselect() and friends may use spi_device->controller_data
     * and controller registers as appropriate.
     *
     * NOTE: SPI controller pins can often be used as GPIO pins instead,
     * which means you could use a bitbang driver either to get hardware
     * working quickly, or testing for differences that aren't speed
     * related.
     */
    struct spi_bitbang_cs {
        unsigned nsecs;     /* (clock cycle time) / 2 */
        u32 (*txrx_word)(struct spi_device *spi, unsigned nsecs, u32 word, u8 bits);
        unsigned (*txrx_bufs)(struct spi_device *, u32 (*txrx_word)
            (struct spi_device *spi, unsigned nsecs, u32 word, u8 bits), 
            unsigned, struct spi_transfer *);
    };
    ```
    - `struct spi_bitbang` 是对 *spi bitbang* (软件模拟的 *spi* 控制器) 的描述，提供了通信模式的集合、设置时钟和位宽的接口。而 `struct spi_bitbang_cs` 则描述与具体 *spi* 设备通信时使用的是哪个通信模式（`txrx_word`）、时钟大小（`nsecs`）、位宽（`txrx_bufs`）等信息。

##### SPI 子系统与 spi_bitbang 关系图示
`drivers/spi/spi_bitbang.c`: *polling/bitbanging SPI master controller driver utilities*

##### 入口和出口
1. `spi_bitbang_start()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    /**
     * spi_bitbang_start - start up a polled/bitbanging SPI master driver
     * @bitbang: driver handle
     *
     * Caller should have zero-initialized all parts of the structure, and
     * then provided callbacks for chip selection and I/O loops. If the master
     * has a transfer method, its final step should call spi_bitbang_transfer;
     * or, that's the default if the transfer routine is not initialized. It should
     * also set up the bus number and number of chipselects.
     *
     * For i/o loops, provide callbacks either per-word (for bitbanging, or for
     * hardware that basically exposes a shift register) or per-spi_transfer
     * (which takes better advantage of hardware like fifos or DMA engines).
     *
     * Drivers using per-word I/O loops should use (or call) spi_bitbang_setup,
     * spi_bitbang_cleanup and spi_bitbang_setup_transfer to handle those spi
     * master methods. Those methods are the defaults if the bitbang->txrx_bufs
     * routine isn's initalized.
     * 
     * This routine register the spi_master, which will process requests in a
     * dedicated task, keeping IRQs unblocked most of the time. To stop processing
     * those requests, call spi_bitbang_stop().
     *
     * On success, this routine will take a reference to master. The caller is
     * responsible for calling spi_bitbang_stop() to decrement the reference and
     * spi_master_put() as counterpart of spi_alloc_master() to prevent a memory leak.
     */
    int spi_bitbang_start(struct spi_bitbang *bitbang) {
        struct spi_master *master = bitbang->master;
        int ret;
        if (!master || !bitbang->chipselect)
            return -EINVAL;
        spin_lock_init(&bitbang->lock);
        if (!master->mode_bits)
            master->mode_bits = SPI_CPOL | SPI_CPHA | bitbang->flags;
        if (master->transfer || master->transfer_one_message)
            return -EINVAL;
        master->prepare_transfer_hardware = spi_bitbang_prepare_hardware;
        master->unprepare_transfer_hardware = spi_bitbang_prepare_hardware;
        master->transfer_one_message = spi_bitbang_transfer_one;
        if (!bitbang->txrx_bufs) {
            bitbang->use_dma = 0;
            bitbang->txrx_bufs = spi_bitbang_bufs;
            if (!master->setup) {
                if (!bitbang->setup_transfer)
                    bitbang->setup_transfer = spi_bitbang_setup_transfer;
                master->setup = spi_bitbang_setup;
                master->cleanup = spi_bitbang_cleanup;
            }
        }
        /* driver may get busy before register() returns, especially
         * if someone registered boardinfo for devices.
         */
        ret = spi_register_master(spi_master_get(master));
        if (ret)
            spi_master_put(master);
        return 0;
    }
    EXPORT_SYMBOL_GPL(spi_bitbang_start);
    ```
    - 使用 *spi bitbang* 的 *spi*（模拟）控制器驱动程序，被强制要求使用：
      - `master->prepare_transfer_hardware()` 为 `spi_bitbang_prepare_hardware()`
      - `master->unprepare_transfer_hardware()` 为 `spi_bitbang_unprepare_hardware()`
      - `master->transfer_one_message()` 为 `spi_bitbang_transfer_one()`，主要的 *message* 传输逻辑都在该函数中。
2. `spi_bitbang_stop()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    /* spi_bitbang_stop - stops the task providing spi communication */
    void spi_bitbang_stop(struct spi_bitbang *bitbang) {
        spi_unregister_master(bitbang->master);
    }
    EXPORT_SYMBOL_GPL(spi_bitbang_stop);
    ```

##### 相关接口函数的默认实现
1. `spi_bitbang_setup_transfer()` - `bitbang->setup_transfer()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    int spi_bitbang_setup_transfer(struct spi_device *spi, struct spi_transfer *t) {
        struct spi_bitbang_cs *cs = spi->controller_state;
        u8 bits_per_word;
        u32 hz;
        if (t) {
            bits_per_word = t->bits_per_word;
            hz = t->speed_hz;
        } else {
            bits_per_word = 0;
            hz = 0;
        }
        /* spi_transfer level calls that work per-word */
        if (!bits_per_word)
            bits_per_word = spi->bits_per_word;
        if (bits_per_word <= 8)
            cs->txrx_bufs = bitbang_txrx_8;
        else if (bits_per_word <= 16)
            cs->txrx_bufs = bitbang_txrx_16;
        else if (bits_per_word <= 32)
            cs->txrx_bufs = bitbang_txrx_32;
        else
            return -EINVAL;
        /* nsecs = (clock period) / 2 */
        if (!hz)
            hz = spi->max_speed_hz;
        if (hz) {
            cs->nsecs = (1000000000/2) / hz;
            if (cs->nsecs > (MAX_UDELAY_MS * 1000 * 1000))
                return -EINVAL;
        }
        return 0;
    } 
    EXPORT_SYMBOL_GPL(spi_bitbang_setup_transfer);
    ```
    - 当满足 *spi_transfer*/*spi_device* 要求的传输频率时，每发送一个 *bit* 需要的周期时间为 `1 / hz` 秒，换算为纳秒为 `10^9 * (1 / hz)` 纳秒。将周期时间等分，则可以等到一个高/低电平时钟的时间，即延时时间，为 `(10^9/hz)/2`。
    - 由于使用的 `udelay()` 等延时接口函数为忙等，因此不宜超过 `MAX_UDELAY_MS` 时长。
    - 综上，`spi_bitbang_setup_transfer()` 函数的功能是：设置 `spi_device->controller_state` 作为 *spi_bitbang_cs* 时的相关字段，包括：
      - `spi_bitbang_cs->txrx_bufs()`，它根据 `bits_per_word` 来设置，而 `bits_per_word` 又是根据 *spi_transfer*/*spi_device* 来得到；
      - `spi_bitbang_cs->nsecs`，即每发送一个 *bit* 需要延时的时长，单位为纳秒。如果该值不满足小于等于最大延时时间 `MAX_UDELAY_MS`，则设置失败，返回错误。
2. `bitbang_txrx_8()`、`bitbang_txrx_16()`、`bitbang_txrx_32()`: - `bitbang_cs->txrx_bufs()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    static unsigned bitbang_txrx_8(struct spi_device *spi,
        u32 (*txrx_word)(struct spi_device *spi, unsigned nsecs,
            u32 word, u8 bits), unsigned ns, struct spi_transfer *t) {
        unsigned bits = t->bits_per_word;
        unsigned count = t->len;
        const u8 *tx = t->tx_buf;
        u8 *rx = t->rx_buf;

        while (likely(count > 0)) {
            u8 word = 0;
            if (tx)
                word = *tx++;
            word = txrx_word(spi, ns, word, bits);
            if (rx)
                *rx++ = word;
            count -= 1;
        }
        return t->len - count;
    }

    static unsigned bitbang_txrx_16(...) {
        ...
        const u16 *tx = t->tx_buf;
        u16 *rx = t->rx_buf;

        while (likely(count > 1)) {
            u16 word = 0;
            if (tx)
                word = *tx++;
            word = txrx_word(spi, ns, word, bits);
            if (rx)
                *rx++ = word;
            count -= 2;
        }
        return t->len - count;
    }

    static unsigned bitbang_txrx_32(...) {
        ...
    }
    ```
3. `spi_bitbang_bufs()` - `bitbang->txrx_bufs()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    static int spi_bitbang_bufs(struct spi_device *spi, struct spi_transfer *t) {
        struct spi_bitbang_cs *cs = spi->controller_state;
        unsigned nsecs = cs->nsecs;
        return cs->txrx_bufs(spi, cs->txrx_word, nsecs, t);
    }
    ```
4. `spi_bitbang_setup()` - *default setup for per-word I/O loops* - `master->setup()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    int spi_bitbang_setup(struct spi_device *spi) {
        struct spi_bitbang_cs *cs = spi->controller_state;
        struct spi_bitbang *bitbang;
        unsigned long flags;

        bitbang = spi_master_get_devdata(spi->master);
        if (!cs) {
            cs = kzalloc(sizeof(*cs), GFP_KERNEL);
            if (!cs)
                return -ENOMEM;
            spi->controller_state = cs;
        }
        /* per-word shift register access, in hardware or bitbanging */
        cs->txrx_word = bitbang->txrx_word[spi->mode & (SPI_CPOL | SPI_CPHA)];
        if (!cs->txrx_word)
            return -EINVAL;
        if (bitbang->setup_transfer) {
            int retval = bitbang->setup_transfer(spi, NULL);
            if (retval < 0)
                return retval;
        }
        dev_dbg(&spi->dev, "%s, %u nsec/bit\n", __func__, 2 * cs->nsecs);
        
        /* NOTE we __need__ to call chipselect() early, ideally with adapter
         * setup, unless the hardware defaults cooperate to avoid confusion
         * between normal (active low) amd inverted chipselects.
         */
        
        /* deselect chip (low or high) */
        spin_lock_irqsave(&bitbang->lock, flags);
        if (!bitbang->busy) {
            bitbang->chipselect(spi, BITBANG_CS_INACTIVE);
            ndelay(cs->nsecs);
        }
        spin_unlock_irqrestore(&bitbang->lock, flags);
        return 0;
    }
    EXPORT_SYMBOL_GPL(spi_bitbang_setup);
    ```
    - 可作为 `master->setup()` 的实现（在 `spi_bitbang_start()` 中），设置与 *spi* 设备的通信模式、时钟、位宽等。
5. `spi_bitbang_cleanup()` - *default cleanup for per-word I/O* - `master->cleanup()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    void spi_bitbang_cleanup(struct spi_device *spi) {
        kfree(spi->controller_state);
    }
    EXPORT_SYMBOL_GPL(spi_bitbang_cleanup);
    ```
6. `spi_bitbang_prepare_hardware()` 和 `spi_bitbang_unprepare_hardware()` - `master->prepare_transfer_hardware()` 和 `master->unprepare_transfer_hardware()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi-bitbang.c
    /* SECOND PART ... simple transfer queue runner. 
     * This costs a task context per controller, running the queue by
     * performing each transfer in sequence. Smarter hardware can queue
     * several DMA transfers at once, and process several controller queues
     * in parallel; this driver doesn't match such harware very well.
     *
     * Drviers can provide word-at-a-time i/o primitives, or provide
     * transfer-at-a-time ones to leverage (杠杆作用，影响力) dma or fifo hardware.
     */
    static int spi_bitbang_prepare_hardware(struct spi_master *spi) {
        struct spi_bitbang *bitbang;
        unsigned long flags;
        bitbang = spi_master_get_devdata(spi);
        spin_lock_irqsave(&bitbang->lock, flags);
        bitbang->busy = 1;
        spin_unlock_irqrestore(&bitbang->lock, flags);
    }

    static int spi_bitbang_unprepare_hardware(struct spi_master *spi) {
        struct spi_bitbang *bitbang;
        unsigned long flags;
        bitbang = spi_master_get_devdata(spi);
        spin_lock_irqsave(&bitbang->lock, flags);
        bitbang->busy = 0;
        spin_unlock_irqrestore(&bitbang->lock, flags);
        return 0;
    }
    ```
7. `spi_bitbang_transfer_one()` - `master->transfer_one_message()`
    ```c
    static int spi_bitbang_transfer_one(struct spi_master *master, struct spi_message *m) {
        struct spi_bitbang *bitbang;
        unsigned nsecs;
        struct spi_transfer *t = NULL;
        unsigned cs_change;
        int status;
        int do_setup = -1;
        struct spi_device *spi = m->spi;

        bitbang = spi_master_get_devdata(master);

        /* FIXME this is made-up (编造的，虚构的) ... the correct value is known to 
         * word-at-a-time bitbang code, and presumably (大概，很可能) chipselect()
         * should enforce these requirements too?
         */
        nsecs = 100;

        cs_change = 1;
        status = 0;
        
        list_for_each_entry(t, &m->transfers, transfer_list) {
            /* override speed or wordsize? */
            if (t->speed_hz || t->bits_per_word)
                do_setup = 1;
            /* init (-1) or override (1) transfer params */
            if (do_setup != 0) {
                if (bitbang->setup_transfer) {
                    status = bitbang->setup_transfer(spi, t);
                    if (status < 0)
                        break;
                }
                if (do_setup == -1)
                    do_setup = 0;
            }
            /* set up default clock polarity, and activate chip, 
             * this implicitly updates clock and spi modes as
             * previously recorded for this device via setup().
             * (and also deselects any other chip that might be
             * selected ...)
             */
            if (cs_change) {
                bitbang->chipselect(spi, BITBANG_CS_ACTIVE);
                ndelay(nsecs);
            }
            cs_change = t->cs_change;
            if (!t->tx_buf && !t->rx_buf && t->len) {
                status = -EINVAL;
                break;
            }
            /* transfer data. the lower level code handles any 
             * new dma mappings it needs. our caller always gave
             * us dma-safe buffers.
             */
            if (t->len) {
                /* REVISIT dma API still needs a designated (指定的)
                 * DMA_ADDR_INVALID, ~0 might be better.
                 */
                if (!m->is_dma_mapped)
                    tx->rx_dma = tx->tx_dma = 0;
                status = bitbang->txrx_bufs(spi, t);
            }
            if (status > 0)
                m->actual_length += status;
            if (status != t->len) {
                /* always report some kind of error */
                if (status >= 0)
                    status = -EREMOTEIO;
                break;
            }
            status = 0;
            /* protocol tweaks (调整，微调) before next transfer */
            if (t->delay_usecs)
                udelay(t->delay_usecs);
            if (cs_change && !list_is_last(&t->transfer_list, &m->transfers)) {
                /* sometimes a short mid-message deselect of the chip
                 * may be needed to terminate a mode or command
                 */
                ndelay(nsecs);
                bitbang->chipselect(spi, BITBANG_CS_INACTIVE);
                ndelay(nsecs);
            }
        }
        m->status = status;
        /* normally deactivate chipselect ... unless no error and 
         * cs_change has hinted that the next message will probably
         * be for this chip too.
         */
        if (!(status == 0 && cs_change)) {
            ndelay(nsecs);
            bitbang->chipselect(spi, BITBANG_CS_INACTIVE);
            ndelay(nsecs);
        }
        spi_finalize_current_message(master);
        return status;
    }
    ```
    - 这里片选信号的变化时的延时时间选用了一个固定值：`nsecs = 100`，在注释中也提到这是一个可修改的地方，具体延时多长的时间应当由具体 *bitbang* (模拟的 *spi* 控制器) 代码来决定。
    - 对 *message* 中的若干个 *transfer* 进行传输时，当 *transfer* 设置了传输频率或位宽要求就会触发传输前的 `setup_tranfser()`。这里比较有意思的点是，如果第一个 *transfer* 并未指定传输频率或位宽要求，则对后续的所有 *transfer* 都不执行 `setup_transfer()`（使用第一个 *transfer* 触发的默认配置），直到碰到有要求的第一个 *transfer*。
    - `cs_change` 初始值为 1。考虑到前一次 *message* 可能不是当前的 *spi* 设备，默认为 1，在开始第一个 *transfer* 前 *active* 当前的 *spi* 设备。后续是否需要改变当前 *spi* 设备的片选信号，由当前的 `transfer->cs_change` 来决定（`cs_change`: *transfer* 处理前 *active*，处理后 *inactive*）。如果当前 *message* 成功传输，并且最后一个 `transfer->cs_change` 不为 0 时才会将当前片选信号保留（明确知道当前 *message* 与下一个 *message* 是对应与同一各 *spi* 设备）。与之相反的是，只要当前 *message* 传输失败或最后一个 *transfer* 并未要求片选信号不变换，则当 *message* 处理完后都要将片选信号取消。
    - [ ] 这里应该是 `status == 0 && cs_change == 0` 吧？
    - 通过调用 `bitbang->txrx_bufs()` 来完成对 *spi* 设备的通信，通信内容为当前 *transfer*。
    - 当前 *message* 处理完（不一定成功传输）后调用 `spi_finalize_current_message()` 完成收尾工作。

### 设备驱动
#### 数据结构描述
1. *spi* 驱动数据结构描述
    ```c
    // version: 4.1.15
    // include/linux/spi/spi.h

    /**
    * struct spi_driver - Host side "protocol" driver
    * @id_table: List of SPI devices supported by this driver
    * @probe: Binds this driver to the spi device.  Drivers can verify
    *	that the device is actually present, and may need to configure
    *	characteristics (such as bits_per_word) which weren't needed for
    *	the initial configuration done during system setup.
    * @remove: Unbinds this driver from the spi device
    * @shutdown: Standard shutdown callback used during system state
    *	transitions such as powerdown/halt and kexec
    * @driver: SPI device drivers should initialize the name and owner
    *	field of this structure.
    *
    * This represents the kind of device driver that uses SPI messages to
    * interact with the hardware at the other end of a SPI link.  It's called
    * a "protocol" driver because it works through messages rather than talking
    * directly to SPI hardware (which is what the underlying SPI controller
    * driver does to pass those messages).  These protocols are defined in the
    * specification for the device(s) supported by the driver.
    *
    * As a rule, those device protocols represent the lowest level interface
    * supported by a driver, and it will support upper level interfaces too.
    * Examples of such upper levels include frameworks like MTD, networking,
    * MMC, RTC, filesystem character device nodes, and hardware monitoring.
    */
    struct spi_driver {
        const struct spi_device_id *id_table;
        int (*probe)(struct spi_device *spi);
        int (*remove)(struct spi_device *spi);
        void (*shutdown)(struct spi_device *spi);
        struct device_driver driver;
    };
    ```
2. *spi* 设备数据结构描述
    ```c
    // version: 4.1.15
    // include/linux/spi/spi.h
    /**
     * struct spi_device - Master side proxy for an SPI slave device
     * @dev: Driver model representation of the device.
     * @master: SPI controller used with the device.
     * @max_speed_hz: Maximum clock rate to be used with this chip
     *  (on this board); may be changed by the device's driver.
     *  The spi_transfer.speed_hz can override this for each transfer.
     * @chip_select: Chipselect, distinguishing chips handled by @master
     * @mode: The spi mode defines how data is clocked out and in.
     *  This may be changed by the device's driver.
     *  The "active low" default for chipselect mode can be overridden
     *  (by specifying SPI_CS_HIGH) as can the "MSB first" default
     *  for each word in a transfer (by specifying SPI_LSB_FIRST).
     * @bits_per_word: Data transfers involve one or more words; word sizes
     *  like eight or 12 bits are common. In-memory wordsizes are
     *  powers of two bytes (e.g. 20 bit samples use 32 bits).
     *  This may be changed by the device's driver, or left at the
     *  default (0) indicating protocol words are eight bit bytes.
     *  The spi_transfer.bits_per_word can override this for each
     *  transfer.
     * @irq: Negative, or the number passed to request_irq() to receive
     *  interrupts from this device.
     * @controller_state: Controller's runtime state
     * @controller_data: Board-specific definitions for controller,
     *  such as FIFO initialization parameters; from board_info.controller_data
     * @modalias: Name of the driver to use with this device, or
     *  an alias for that name. This appears in the sysfs "modalias" attribute for driver coldplugging, and in uevents used
     * for hotplugging
     * @cs_gpio: gpio number of the chipselect line (optional, -ENOENT
     *  when not using a GPIO line) 
     *
     * A @spi_device is used to interchange (交换) data between an
     * SPI slave (usually a discrete chip) and CPU memmory.
     *
     * In @dev, the platform_data is used to hold information about
     * this device that's meaningful to the device's protocol driver,
     * but not to its controller. One example might be an identifier
     * for a chip variant with slightly different functionality;
     * another might be information about how this particular board
     * wires the chip's pins.
     */
    struct spi_device {
        struct device dev;
        struct spi_master *master;
        u32 max_speed_hz;
        u8 chip_select;
        u8 bits_per_word;
        u16 mode;
    #define SPI_CPHA    0x01    /* clock phase */
    #define SPI_CPOL    0x02    /* clock polarity */
    #define SPI_MODE_0  (0|0)   /* original MicroWire */
    #define SPI_MODE_1  (0|SPI_CPHA)
    #define SPI_MODE_2  (SPI_CPOL|0)
    #define SPI_MODE_3  (SPI_CPOL|SPI_CPHA)
    #define SPI_CS_HIGH 0x04    /* chipselect active high? */
    #define SPI_LSB_FIRST 0x08  /* per-word bits-on-wire */
    #define SPI_3WIRE   0x10    /* SI/SO signals shared */
    #define SPI_LOOP    0x20    /* loopback mode */
    #define SPI_NO_CS   0x40    /* 1 dev/bus, no chipselect */
    #define SPI_READY   0x80    /* slave pulls low to pause */
    #define SPI_TX_DUAL 0x100   /* transmit with 2 wires */
    #define SPI_TX_QUAD 0x200   /* transmit with 4 wires */
    #define SPI_RX_DUAL 0x400   /* receive with 2 wires */
    #define SPI_RX_QUAD 0x800   /* receive with 4 wires */
        int irq;
        void *controller_state;
        void *controller_data;
        char modalias[SPI_NAME_SIZE];
        int cs_gpio;

        /* likely need more hooks for more protocol options affecting
         * how the controller talks to each chip, like:
         * - memory packing (12 bit samples into low bits, others zeroed)
         * - priority
         * - drop chipselect after each word
         * - chipselect delays
         * - ...
         */
    };

    #define SPI_NAME_SIZE 32
    ```

#### 设备分配与注册
通常这部分函数都不需要 *spi* 设备驱动来调用，在注册（注销） *spi master* 时会调用这部分函数来注册（注销）*spi* 设备。

1. `spi_alloc_device()`
    ```c
    // version: 4.1.15
    // drivers/spi/spi.c
    /**
     * spi_alloc_device - Allocate a new SPI device
     * @master: Controller to which device is connected
     * Context: can sleep
     * 
     * Allows a driver to allocate and initialize a spi_device without
     * registering it immediately. This allows a driver to directly
     * fill the spi_device with device parameters before calling 
     * spi_add_device() on it.
     *
     * Caller is responsible to call spi_add_device() on the returned
     * spi_device structure to add it to the SPI master. If the
     * caller needs to discard the spi_device without adding it,
     * then it should call spi_dev_put() on it.
     *
     * Returns a pointer to the new device, or NULL
     */
    struct spi_device *spi_alloc_device(struct spi_master *master) {
        struct spi_device *spi;
        if (!spi_master_get(master))
            return NULL;
        spi = kzalloc(sizeof(*spi), GFP_KERNEL);
        if (!spi) {
            spi_master_put(master);
            return NULL;
        }
        spi->master = master;
        spi->dev.parent = &master->dev;
        spi->dev.bus = &spi_bus_type;
        spi->dev.release = spidev_release;
        spi->cs_gpio = -ENOENT;
        device_initialize(&spi->dev);
        return spi;
    }
    ```
2. `spi_add_device()`
    ```c
    /**
     * spi_add_device - Add spi_device allocated with spi_alloc_device 
     * @spi: spi_device to register
     *
     * Companion function to spi_alloc_device. Devices allocated with
     * spi_alloc_device can be added onto the spi bus with this
     * function.
     *
     * Returns 0 on success; negative errno on failure
     */
    int spi_add_device(struct spi_device *spi) {
        static DEFINE_MUTEX(spi_add_lock);
        struct spi_master *master = spi->master;
        struct device *dev = master->dev.parent;
        int status;
        /* Chipselects are numbered 0..MAX; validate. */
        if (spi->chip_select >= master->num_chipselect) {
            dev_err(dev, "cs%d >= max %d\n", spi->chip_select,
                master->num_chipselect);
            return -EINVAL;
        }
        /* Set the bus ID string */
        spi_dev_set_name(spi);
        /* We need to make sure there's no other device with this
         * chipselect **BEFORE** we call setup(), else we'll trash
         * its configuration. Lock against concurrent add() calls.
         */
        mutex_lock(&spi_add_lock);
        status = bus_for_each_dev(&spi_bus_type, NULL, spi, spi_dev_check);
        if (status) {
            dev_err(dev, "chipselect %d already in use\n", spi->chip_select);
            goto done;
        }
        if (master->cs_gpios)
            spi->cs_gpio = master->cs_gpios[spi->chip_select];
        /* Drivers may modify this initial i/o setup, but will 
         * normally rely on the device being setup. Devices using
         * SPI_CS_HIGH can't coexist (共存) well otherwise...
         */
        status = spi_setup(spi);
        if (status < 0) {
            dev_err(dev, "can't setup %s, status %d\n", 
                dev_name(&spi->dev), status);
            goto done;
        }
        /* Device may be bound to an active driver when this returns */
        status = device_add(&spi->dev);
        if (status < 0)
            dev_err(dev, "can't add %s, status %d\n", dev_name(&spi->dev), status);
        else
            dev_dbg(dev, "registered child %d\n", dev_name(&spi->dev));
    done:
        mutex_unlock(&spi_add_lock);
        return status;
    }
    EXPORT_SYMBOL_GPL(spi_add_device);
    ```
    - `spi_dev_set_name()` 的功能是设置 `spi->dev` 的名字，设置的格式为 “`dev_name(&spi->master->dev).spi->chip_select`”。一般情况下类似于 “*spi0.0*”、“*spi0.1*”...。
    - 为防止片选信号冲突，在将 *spi* 设备注册到 *spi* 总线上之前进行检查：`spi_dev_check()`，方式是对应的 `spi->master` 上是否已经有该片选信号的 *spi device* 存在。由于所有的 *spi* 设备都挂在 *spi* 总线上（不区分 *spi master*），因此这里使用了 `bus_for_each_dev()` 来遍历总线上的所有设备。
    - 在 *spi* 设备的设备树描述中，*reg* 属性的值会对应到 `spi_device->chip_select` 字段上，而 `spi_device->cs_gpio` 会根据此字段的值从其 *spi master* 的 `cs_gpios` 获取 *gpio*。
    - `spi_setup()` 会：
      - 检查 *spi device* 的兼容性，主要检查 `spi_device->mode`：是否有自相矛盾的地方，例如要求 `SPI_3WIRE` 但是又要求 `SPI_TX_DUAL`、要求 `SPI_TX_DUAL` 又要求 `SPI_TX_QUAD` 等；是否能够得到其 *spi master* 的支持，例如 *CPOL*、*CPAH*、*CS_HIGH*、*LSB_FIRST* 和 *3WIRE* 等（`spi->mode & ~spi->master->mode_bits` 为 0 才支持）。
      - 如果 *spi_device* 的 `bits_per_word` 和 `max_speed_hz` 字段未设置，则指定默认值。前者是 8，后者是其 *spi master* 的 `max_speed_hz`。
      - 同时取消该 *spi* 设备的片选信号。
      - 如果 *spi master* 设置了 `setup()` 回调函数，那么会在这里调用，以进一步检查 *spi master* 是否支持该 *spi* 设备。
    - 通过 `device_add()` 将 *spi* 设备注册到系统中。
3. `spi_unregister_device()`
    ```c
    // version: 4.1.15
    // include/linux/spi/spi.h
    static inline void void spi_unregister_device(struct spi_device *spi) {
        if (spi)
            device_unregister(&spi->dev);
    }
    ```

#### spi-dev

### SPI 子系统架构总结

### I2C 和 SPI 子系统对比

## 源码阅读
### spi-gpio
### imx6ull
#### 硬件描述
见 `Hardware/SPI.md`

#### 设备树描述
```dts
// imx6ull.dtsi
ecspi1: ecspi@02008000 {
    #address-cells = <1>;
    #size-cells = <0>;
    compatible = "fsl,imx6ul-ecspi", "fsl,imx51-ecspi";
    reg = <0x02008000 0x4000>;
    interrupts = <GIC_SPI 31 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_ECSPI1>,
            <&clks IMX6UL_CLK_ECSPI1>;
    clock-names = "ipg", "per";
    dmas = <&sdma 3 7 1>, <&sdma 4 7 2>;
    dma-names = "rx", "tx";
    status = "disabled";
};
```
#### 源文件
> 内核版本 kernel: 4.1.15
> 源文件：drivers/spi/spi-imx.c
1. 入口/出口部分
```c
static struct platform_device_id spi_imx_devtype[] = {
    ... 
    {
        .name = "imx6ul-ecspi",
        .driver_data = (kernel_ulong_t) &imx6ul_ecspi_devtype_data,
    }, {
        /* sentinel */
    }
};

static const struct of_device_id spi_imx_dt_ids[] = {
    { .compatible = "fsl,imx-cspi", .data = &imx1_cspi_devtype_data, },
    ...
    { .compatible = "fsl,imx6ul-ecspi", .data = &imx6ul_ecspi_devtype_data, },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, spi_imx_dt_ids);

static struct platform_driver spi_imx_driver = {
    .driver = {
        .name = DRIVER_NAME,
        .of_match_table = spi_imx_dt_ids,
        .pm = IMX_SPI_PM,
    };
    .id_table = spi_imx_devtype,
    .probe = spi_imx_probe,
    .remove = spi_imx_remove,
};
module_platform_driver(spi_imx_driver);
```

2. *SoC-specific* 数据结构描述及关于 *imx6ull* 的实例
```c
struct spi_imx_master {
    int *chipselect;
    int num_chipselect;
};

struct spi_imx_config {
    unsigned int speed_hz;
    unsigned int bpw;
    unsigned int mode;
    u8 cs;
};

struct spi_imx_devtype_data {
    void (*intctrl)(struct spi_imx_data *, int);
    int (*config)(struct spi_imx_data *, struct spi_imx_config *);
    void (*trigger)(struct spi_imx_data *);
    int (*rx_available)(struct spi_imx_data *);
    void (*reset)(struct spi_imx_data *);
    enum spi_imx_devtype devtype;       // 关于此类型的定义略，只需知道这里使用的是 IMX6UL_ECSPI 即可
};

struct spi_imx_data {
    // 将 spi_bitbang 放置开头
    // 后续即使通过 API 获取的数据是 spi_imx_data 指针，也可以通过
    // 强制类型转换获取 spi_bitbang 指针
    struct spi_bitbang bitbang;

    struct completion xfer_done;
    void __iomem *base;
    struct clk *clk_per;
    struct clk *clk_ipg;
    unsigned long spi_clk;

    unsigned int count;
    void (*tx)(struct spi_imx_data *);
    void (*rx)(struct spi_imx_data *);
    void *rx_buf;
    const void *tx_buf;
    unsigned int txfifo;    /* number of words pushed in tx FIFO */

    /* DMA */
    unsigned int dma_is_inited;
    unsigned int dma_finished;
    bool usedma;
    u32 rx_wml;             /* wml, watermark level, 水线 */
    u32 tx_wml;
    u32 rxt_wml;
    struct completion dma_rx_completion;
    struct completion dma_tx_completion;
    struct dma_slave_config rx_config;
    struct dma_slave_config tx_config;

    const struct spi_imx_devtype_data *devtype_data;
    int chipselect[0];
}

// imx6ull 的实例
static struct spi_imx_devtype_data imx6ul_ecspi_devtype_data = {
    .intctrl = mx51_ecspi_intctrl,
    .config  = mx51_ecspi_config,
    .trigger = mx51_ecspi_trigger,
    .rx_available = mx51_ecspi_rx_available,
    .reset = mx51_ecspi_reset,
    .devtype = IMX6UL_ECSPI,
};

/* 
 * 中断控制，这里只使用了三个中断
 * 1. 发送为空中断
 * 2. 存在接收内容中断
 * 3. 发送完成中断
 */
static void __maybe_unused mx51_ecspi_intctrl(struct spi_imx_data *spi_imx, int enable) {
    unsigned val = 0;
    if (enable & MXC_INT_TE)
		val |= MX51_ECSPI_INT_TEEN;
	if (enable & MXC_INT_RR)
		val |= MX51_ECSPI_INT_RREN;
	if (enable & MXC_INT_TCEN)
		val |= MX51_ECSPI_INT_TCEN;
	writel(val, spi_imx->base + MX51_ECSPI_INT);
}


/* 
 * spi 控制器配置：
 * 1. 配置 CONFIG、CONTROL、DMA 三个寄存器
 * 2. 配置内容：spi 使能、通道、片选、频率、burst length、CPHA、CPOL 等等
 * 3. 配置来源： 默认-使能 SPI、并将四个通道配置为 Master mode、（若使用 DMA）使能 TX FIFO 为空时
 *                的 DMA 请求、RX FIFO DMA 请求.
 *              config->speed_hz: 修改 CONFIG 中的两个分频器以配置时钟频率
 *              config->cs: 配置片选引脚、multiple/single burst (使用 multiple)
 *              config->bpw: 配置 BURST_LENGTH
 *              config->mode + config->cs: 配置 CPAH 和 CPOL，以及未激活状态下 CLK 的极性
 *              config->mode: 配置片选是否高电平有效
 *              spi_imx->spi_clk: spi 时钟源频率，计算分频器的值时用到
 *              spi_imx->base: 写入各个寄存器时的基地址
 *              spi_imx->rx_wml 和 tx_wml: 发送和接收的水线，配置 DMA 的门限值
 *             
 */
static int __maybe_unused mx51_ecspi_config(struct spi_imx_data *spi_imx,
		struct spi_imx_config *config)
{
	u32 ctrl = MX51_ECSPI_CTRL_ENABLE, cfg = 0, dma = 0;
	u32 clk = config->speed_hz, delay;
	/*
	 * The hardware seems to have a race condition when changing modes. The
	 * current assumption is that the selection of the channel arrives
	 * earlier in the hardware than the mode bits when they are written at
	 * the same time.
	 * So set master mode for all channels as we do not support slave mode.
	 */
	ctrl |= MX51_ECSPI_CTRL_MODE_MASK;
	/* set clock speed */
	ctrl |= mx51_ecspi_clkdiv(spi_imx->spi_clk, config->speed_hz, &clk);
	/* set chip select to use */
	ctrl |= MX51_ECSPI_CTRL_CS(config->cs);
	ctrl |= (config->bpw - 1) << MX51_ECSPI_CTRL_BL_OFFSET;
	cfg |= MX51_ECSPI_CONFIG_SBBCTRL(config->cs);
	if (config->mode & SPI_CPHA)
		cfg |= MX51_ECSPI_CONFIG_SCLKPHA(config->cs);
	if (config->mode & SPI_CPOL) {
		cfg |= MX51_ECSPI_CONFIG_SCLKPOL(config->cs);
		cfg |= MX51_ECSPI_CONFIG_SCLKCTL(config->cs);
	}
	if (config->mode & SPI_CS_HIGH)
		cfg |= MX51_ECSPI_CONFIG_SSBPOL(config->cs);
	writel(ctrl, spi_imx->base + MX51_ECSPI_CTRL);
	writel(cfg, spi_imx->base + MX51_ECSPI_CONFIG);
	/*
	 * Wait until the changes in the configuration register CONFIGREG
	 * propagate into the hardware. It takes exactly one tick of the
	 * SCLK clock, but we will wait two SCLK clock just to be sure. The
	 * effect of the delay it takes for the hardware to apply changes
	 * is noticable if the SCLK clock run very slow. In such a case, if
	 * the polarity of SCLK should be inverted, the GPIO ChipSelect might
	 * be asserted before the SCLK polarity changes, which would disrupt
	 * the SPI communication as the device on the other end would consider
	 * the change of SCLK polarity as a clock tick already.
	 */
	delay = (2 * 1000000) / clk;
	if (likely(delay < 10))	/* SCLK is faster than 100 kHz */
		udelay(delay);
	else			/* SCLK is _very_ slow */
		usleep_range(delay, delay + 10);
	/*
	 * Configure the DMA register: setup the watermark
	 * and enable DMA request.
	 */
	if (spi_imx->dma_is_inited) {
		if (spi_imx->devtype_data->devtype != IMX6UL_ECSPI)
			spi_imx->tx_wml = 1;
		dma = (spi_imx->rx_wml - 1) << MX51_ECSPI_DMA_RX_WML_OFFSET
		      | (spi_imx->tx_wml - 1) << MX51_ECSPI_DMA_TX_WML_OFFSET
		      | (1 << MX51_ECSPI_DMA_TEDEN_OFFSET)
		      | (1 << MX51_ECSPI_DMA_RXDEN_OFFSET);
		writel(dma, spi_imx->base + MX51_ECSPI_DMA);
	}
	return 0;
}

/* 
 * 触发 spi 传输 
 * 1. 如果没有使用 DMA，则直接将控制寄存器的 XCH 置为 1（进行数据传输）
 * 2. 如果使用 DMA，但是 dma 操作未完成且设备类型是 IMX6UL_ECSPI，将控制寄存器 SMC 置位（立即进行数据传输）
 * 3. 如果使用 DMA，且不符合 2 中的其他条件，就将控制寄存器的 SMC 清空（由 XCH 位控制是否进行数据传输）
 * 注：从注释来看，SDMA 需要使用 XCH 信号而不是 SMC（因此在 dma 操作完成时需要等待 XCH 置位？）
 */
static void __maybe_unused mx51_ecspi_trigger(struct spi_imx_data *spi_imx)
{
	u32 reg = readl(spi_imx->base + MX51_ECSPI_CTRL);
	/*
	 * To workaround ERR008517, SDMA script need use XCH instead of SMC
	 * just like PIO mode and it fix on i.mx6ul
	 */
	if (!spi_imx->usedma)
		reg |= MX51_ECSPI_CTRL_XCH;
	else if (!spi_imx->dma_finished &&
		 spi_imx->devtype_data->devtype == IMX6UL_ECSPI)
		reg |= MX51_ECSPI_CTRL_SMC;
	else
		reg &= ~MX51_ECSPI_CTRL_SMC;
	writel(reg, spi_imx->base + MX51_ECSPI_CTRL);
}

/* 
 * spi 接收数据判断
 * 通过状态寄存器判断 RX FIFO 中是否有数据
 */
static int __maybe_unused mx51_ecspi_rx_available(struct spi_imx_data *spi_imx)
{
	return readl(spi_imx->base + MX51_ECSPI_STAT) & MX51_ECSPI_STAT_RR;
}

/*
 * spi reset 
 * 相当于将 RX FIFO 中的数据清空
 */
static void __maybe_unused mx51_ecspi_reset(struct spi_imx_data *spi_imx)
{
	/* drain receive buffer */
	while (mx51_ecspi_rx_available(spi_imx))
		readl(spi_imx->base + MXC_CSPIRXDATA);
}
```
- `mx51_ecspi_clkdiv()` 根据给定的时钟频率和目标频率，计算得到实际使用的频率，并保存在变量 `clk` 中。计算过程的实现见源文件。

3. *probe/remove* 部分
```c
static int spi_imx_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    const struct of_device_id *of_id =
            of_match_device(spi_imx_dt_ids, &pdev->dev);
    struct spi_imx_master *mxc_platform_info =
            dev_get_platdata(&pdev->dev);
    struct spi_master *master;
    struct spi_imx_data *spi_imx;
    struct resource *res;
    int i, ret, num_cs, irq;
    ...
    ret = of_property_read_u32(np, "fsl,spi-num-chipselects", &num_cs);
    ...
    master = spi_alloc_master(&pdev->dev,
            sizeof(struct spi_imx_data) + sizeof(int) * num_cs);
    ...
    platform_set_drvdata(pdev, master);
    master->bits_per_word_mask = SPI_BPW_RANGE_MASK(1, 32);
    master->bus_num = pdev->id;
    master->num_chipselect = num_cs;
    spi_imx = spi_master_get_devdata(master);
    spi_imx->bitbang.master = master;
    for (i = 0; i < master->num_chipselect; i++) {
        int cs_gpio = of_get_named_gpio(np, "cs-gpios", i);
        ...
        spi_imx->chipselect[i] = cs_gpio;
        ...
        ret = devm_gpio_request(&pdev->dev, spi_imx->chipselect[i],
                    DRIVER_NAME);
        ...
    }
    spi_imx->bitbang.chipselect = spi_imx_chipselect;
    spi_imx->bitbang.setup_transfer = spi_imx_setupxfer;
    spi_imx->bitbang.txrx_bufs = spi_imx_transfer;
    spi_imx->bitbang.master->setup = spi_imx_setup;
    spi_imx->bitbang.master->cleanup = spi_imx_cleanup;
    spi_imx->bitbang.master->prepare_message = spi_imx_prepare_message;
    spi_imx->bitbang.master->unprepare_message = spi_imx_unprepare_message;
    spi_imx->bitbang.master->mode_bits = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
    init_completion(&spi_imx->xfer_done);
    spi_imx->devtype_data = of_id ? of_id->data :
        (struct spi_imx_devtype_data *) pdev->id_entry->driver_data;
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    spi_imx->base = devm_ioremap_resource(&pdev->dev, res);
    ...
    irq = platform_get_irq(pdev, 0);
    ...
    ret = devm_request_irq(&pdev->dev, irq, spi_imx_isr, 0,
                dev_name(&pdev->dev), spi_imx);
    ...
    spi_imx->clk_ipg = devm_clk_get(&pdev->dev, "ipg");
    ...
    spi_imx->clk_per = devm_clk_get(&pdev->dev, "per");
    ...
    ret = clk_prepare_enable(spi_imx->clk_per);
    ...
    ret = clk_prepare_enable(spi_imx->clk_ipg);
    ...
    spi_imx->spi_clk = clk_get_rate(spi_imx->clk_per);
    /*
    * Only validated on i.mx6 now, can remove the constrain if validated on
    * other chips.
    */
    if ((spi_imx->devtype_data == &imx51_ecspi_devtype_data
        || spi_imx->devtype_data == &imx6ul_ecspi_devtype_data)
        && spi_imx_sdma_init(&pdev->dev, spi_imx, master, res))
        dev_err(&pdev->dev, "dma setup error,use pio instead\n");
    spi_imx->devtype_data->reset(spi_imx);
    spi_imx->devtype_data->intctrl(spi_imx, 0);
    master->dev.of_node = pdev->dev.of_node;
    ret = spi_bitbang_start(&spi_imx->bitbang);
    ...
    dev_info(&pdev->dev, "probed\n");
    clk_disable_unprepare(spi_imx->clk_ipg);
    clk_disable_unprepare(spi_imx->clk_per);
    return ret;
...
}

static int spi_imx_remove(struct platform_device *pdev)
{
    struct spi_master *master = platform_get_drvdata(pdev);
    struct spi_imx_data *spi_imx = spi_master_get_devdata(master);
    spi_bitbang_stop(&spi_imx->bitbang);
    writel(0, spi_imx->base + MXC_CSPICTRL);
    clk_unprepare(spi_imx->clk_ipg);
    clk_unprepare(spi_imx->clk_per);
    spi_imx_sdma_exit(spi_imx);
    spi_master_put(master);
    return 0;
}

/* 
 * spi 设备片选函数 
 * 根据 spi_device 的片选引脚（对应到 spi_master 中片选引脚的索引）和 spi_device->mode
 * （片选有效的高低电平）以及 is_active 参数值
 * 来设置对应的 gpio 输出
 */
static void spi_imx_chipselect(struct spi_device *spi, int is_active)
{
	struct spi_imx_data *spi_imx = spi_master_get_devdata(spi->master);
	int gpio = spi_imx->chipselect[spi->chip_select];
	int active = is_active != BITBANG_CS_INACTIVE;
	int dev_is_lowactive = !(spi->mode & SPI_CS_HIGH);

	if (!gpio_is_valid(gpio))
		return;

	gpio_set_value(gpio, dev_is_lowactive ^ active);
}

/* 
 * spi 进行传输前的配置 
 * 1. 会结合 spi_device 和 spi_transfer 中的要求进行设置 bpw 和 speed_hz
 *    后者的优先级更高
 * 2. mode 和 cs 从 spi_device 获取
 * 3. 根据 bpw 设置传输函数 tx 和 rx，并设置 dma_slave_config 的地址宽度
 * 4. 如果可以，进行 DMA 的相关设置
 */
static int spi_imx_setupxfer(struct spi_device *spi,
				 struct spi_transfer *t)
{
	struct spi_imx_data *spi_imx = spi_master_get_devdata(spi->master);
	struct spi_imx_config config;
	int ret;
	config.bpw = t ? t->bits_per_word : spi->bits_per_word;
	config.speed_hz  = t ? t->speed_hz : spi->max_speed_hz;
	config.mode = spi->mode;
	config.cs = spi->chip_select;
	if (!config.speed_hz)
		config.speed_hz = spi->max_speed_hz;
	if (!config.bpw)
		config.bpw = spi->bits_per_word;
	/* Initialize the functions for transfer */
	if (config.bpw <= 8) {
		spi_imx->rx = spi_imx_buf_rx_u8;
		spi_imx->tx = spi_imx_buf_tx_u8;
		spi_imx->tx_config.dst_addr_width = DMA_SLAVE_BUSWIDTH_1_BYTE;
		spi_imx->rx_config.src_addr_width = DMA_SLAVE_BUSWIDTH_1_BYTE;
	} else if (config.bpw <= 16) {
		...
	} else {
		...
	}
	if (spi_imx->bitbang.master->can_dma &&
	    spi_imx_can_dma(spi_imx->bitbang.master, spi, t)) {
		ret = dmaengine_slave_config(spi_imx->bitbang.master->dma_tx,
						&spi_imx->tx_config);
		...
		ret = dmaengine_slave_config(spi_imx->bitbang.master->dma_rx,
						&spi_imx->rx_config);
		...
	}
	spi_imx->devtype_data->config(spi_imx, &config);
	return 0;
}

/* 
 * spi 通信函数 
 * 根据当前 transfer 能否进行 dma 操作有两种不同的传输方式
 * 1. spi_imx_dma_transfer
 * 2. spi_imx_pio_transfer
 */
static int spi_imx_transfer(struct spi_device *spi,
				struct spi_transfer *transfer)
{
	int ret;
	struct spi_imx_data *spi_imx = spi_master_get_devdata(spi->master);
	if (spi_imx->bitbang.master->can_dma &&
	    spi_imx_can_dma(spi_imx->bitbang.master, spi, transfer)) {
		spi_imx->usedma = true;
		ret = spi_imx_dma_transfer(spi_imx, transfer);
		spi_imx->usedma = false; /* clear the dma flag */
		if (ret != -EAGAIN)
			return ret;
	}
	spi_imx->usedma = false;
	return spi_imx_pio_transfer(spi, transfer);
}

/* spi 设备注册时，会通过 spi_setup 进而调用 master->setup() 
 * 执行到此函数（判断 spi 设备对应的 master 能否支持该设备）
 */
static int spi_imx_setup(struct spi_device *spi) {
    struct spi_imx_data *spi_imx = spi_master_get_devdata(spi->master);
    int gpio = spi_imx->chipselect[spi->chip_select];
    dev_dbg(&spi->dev, "%s: mode %d, %u bpw, %d hz\n", 
        __func__,
        spi->mode,
        spi->bits_per_word,
        spi->max_speed_hz);
    if (gpio_is_valid(gpio))
        gpio_direction_output(gpio, spi->mode & SPI_CS_HIGH ? 0 : 1);
    spi_imx_chipselect(spi, BITBANG_CS_INACTIVE);
    return 0;
}

static void spi_imx_cleanup(struct spi_device *spi) {

}

/* spi message 传输前调用此函数进行一些准备工作 */
static int spi_imx_prepare_message(struct spi_master *master, struct spi_message *msg) {
    struct spi_imx_data *spi_imx = spi_master_get_devdata(master);
    int ret;
    ret = clk_prepare_enable(spi_imx->clk_per);
    if (ret)
        return ret;
    ret = clk_prepare_enable(spi_imx->clk_ipg);
    if (ret) {
        clk_disable_unprepare(spi_imx->clk_per);
        return ret;
    }
    return 0;
}

/* spi message 传输后调用此函数进行一些收尾工作 */
static int spi_imx_unprepare_message(struct spi_master *master, struct spi_message) {
    struct spi_imx_data *spi_imx = spi_master_get_devdata(master);
    clk_disable_unprepare(spi_imx->clk_ipg);
    clk_disable_unprepare(spi_imx->clk_per);
    return 0;
}

static int spi_imx_sdma_init(struct device *dev, struct spi_imx_data *spi_imx, 
    struct spi_master *master, const struct resource *res) {
    int ret;
    /* Prepare for TX DMA: */
    master->dma_tx = dma_request_slave_channel(dev, "tx");
    if (!master->dma_tx) {
        ... // error handling
    }
    spi_imx->tx_config.direction = DMA_MEM_TO_DEV;
    spi_imx->tx_config.dst_addr = res->start + MXC_CSPITXDATA;
    spi_imx->tx_config.dst_addr_width = DMA_SLAVE_BUSWIDTH_1_BYTE;
    spi_imx->tx_config.dst_maxburst = spi_imx_get_fifosize(spi_imx) / 4;
    ret = dmaengine_slave_config(master->dma_tx, &spi_imx->tx_config);
    if (ret) {
        ... // error handling
    }
    /* Prepare for RX DMA: */
    master->dma_rx = dma_request_slave_channel(dev, "rx");
    if (!master->dma_rx) {
        ... // error handling
    }
    spi_imx->rx_config.direction = DMA_DEV_TO_MEM;
    spi_imx->rx_config.src_addr = res->start + MXC_CSPIRXDATA;
    spi_imx->rx_config.src_addr_width = DMA_SLAVE_BUSWIDTH_1_BYTE;
    spi_imx->rx_config.src_maxburst = spi_imx_get_fifosize(spi_imx) / 2;
    ret = dmaengine_slave_config(master->dma_rx, &spi_imx->rx_config);
    if (ret) {
        // error handling
    }
    init_completion(&spi_imx->dma_rx_completion);
    init_completion(&spi_imx->dma_tx_completion);
    master->can_dma = spi_imx_can_dma;
    master->max_dma_len = MAX_SDMA_BD_BYTES;
    spi_imx->bitbang.master->flags = SPI_MASTER_MUST_RX | SPI_MASTER_TX;
    spi_imx->rx_wml = spi_imx->rx_config.src_maxburst;
    spi_imx->tx_wml = spi_imx->tx_config.dst_maxburst;
    spi_imx->dma_is_inited = 1;
    return 0;
err:
    spi_imx_sdma_exit(spi);
    return ret;
}

static void spi_imx_sdma_exit(struct spi_imx_data *spi_imx) {
    struct spi_master *master = spi_imx->bitbang.master;
    if (master->dma_rx) {
        dma_release_channel(master->dma_rx);
        master->dma_rx = NULL;
    }
    if (master->dma_tx) {
        dma_release_channel(master->dma_tx);
        master->dma_tx = NULL;
    }
    spi_imx->dma_is_inited = 0;
}

/* The maximum bytes that a sdma can transfer */
#define MAX_SDMA_BD_BYTES (1 << 15)
```
- `spi_imx_can_dma()`:
    ```c
    static bool spi_imx_can_dma(struct spi_master *master, struct spi_device *spi,
			 struct spi_transfer *transfer)
    {
        struct spi_imx_data *spi_imx = spi_master_get_devdata(master);

        if (transfer && spi_imx->dma_is_inited &&
            (transfer->len > spi_imx_get_fifosize(spi_imx)))    // imx6ull 返回 64
            return true;
        return false;
    }
    ```
- `spi_imx_dma_transfer()` - 通过 *DMA* 进行数据传输
    ```c
    static int spi_imx_dma_transfer(struct spi_imx_data *spi_imx, struct spi_transfer *transfer) {
        struct dma_async_tx_descriptor *desc_tx = NULL, *desc_rx = NULL;
        int ret;
        int left = 0;
        struct spi_master *master = spi_imx->bitbang.master;
        struct sg_table *tx = &transfer->tx_sg, *rx = &transfer->rx_sg;
        if (tx) {
            desc_tx = dmaengine_prep_slave_sg(master->dma_tx, 
            tx->sgl, tx->nents, DMA_MEM_TO_DEV, 
            DMA_PREP_INTERRUPT | DMA_CTRL_ACK);
            if (!desc_tx)
                goto tx_nodma;
            desc_tx->callback = spi_imx_dma_tx_callback;
            desc_tx->callback_param = (void *)spi_imx;
            dmaengine_submit(desc_tx);
        }
        if (rx) {
            struct scatterlist *sgl_last = &rx_sgl[rx->nents - 1];
            unsigned int orig_length = sgl_last->length;
            int wml_mask = ~(spi_imx->rx_wml - 1);
            /* 
             * Adjust the transfer length of the last scatterlist 
             * if there are some tail data, use PIO read to get the
             * tail data since DMA sometimes miss the last tail interrupt
             */
            left = transfer->len % spi_imx->rx_wml;
            if (left)
                sgl_last->length = orig_length & wml_mask;
            desc_rx = dmaengine_prep_slave_sg(master->dma_rx,
            rx->sgl, rx->nents, DMA_DEV_TO_MEM,
            DMA_PREP_INTERRUPT | DMA_CTRL_ACK);
            if (!desc_rx)
                goto rx_nodma;
            desc_rx->callback = spi_imx_dma_rx_callback;
            desc_rx->callback_param = (void *)spi_imx;
            dmaengine_submit(desc_rx);
        }
        reinit_completion(&spi_imx->dma_tx_completion);
        reinit_completion(&spi_imx->dma_rx_completion);
        /* Trigger the cspi module */
        spi_imx->dma_finished = 0;
        spi_imx->devtype_dat->trigger(spi);
        dma_async_issue_pending(master->dma_tx);
        dma_async_issue_pending(master->dma_rx);
        /* wait SDMA to finish the data transfer */
        ret = wait_for_completion_timeout(&spi_imx->dma_tx_completion, 
        IMX_DMA_TIMEOUT(transfer->len));
        if (!ret) {
            pr_warn("%s %s: I/O Error in DMA TX:%x\n", 
                dev_driver_string(&master->dev),
                dev_name(&master->dev), transfer->len);
            dmaengine_terminate_all(master->dma_tx);
        } else {
            ret = wait_for_completion_timeout(&spi_imx->dma_rx_completion,
            IMX_DMA_TIMEOUT(transfer->len));
            if (!ret) {
                pr_warn("%s %s: I/O Error in DMA RX:%X\n",
                    dev_driver_string(&master->dev),
                    dev_name(&master->dev), transfer->len);
                spi_imx->devtype_data->reset(spi_imx);
                dmaengine_terminate_all(master->dma_rx);
            } else if (left) {
                /* read the tail data by PIO */
                dma_sync_sg_for_cpu(master->dma_rx->device->dev, 
                    &rx->sgl[rx->nents - 1], 1, DMA_FROM_DEVICE);
                spi_imx->rx_buf = transfer->rx_buf + (transfer->len - left);
                spi_imx_tail_pio_set(spi_imx, left);
                reinit_completion(&spi_imx->xfer_done);
                spi_imx->devtype_data->intctrl(spi_imx, MXC_INT_TCEN);
                ret = wait_for_completion_timeout(&spi_imx->xfer_done, IMX_DMA_TIMEOUT(transfer->len));
                if (!ret) {
                    ...
                }
            }
        }
        spi_imx->dma_finished = 1;
        if (spi_imx->devtype_data->devtype == IMX6UL_ECSPI)
            spi_imx->devtype_dat->trigger(spi_imx);
        if (!ret)
            ret = -ETIMEOUT;
        else if (ret > 0)
            ret = transfer->len;
        return ret;
    rx_nodma:
        dmaengine_terminate_all(master->dma_tx);
    tx_nodma:
        pr_warn_once("%s %s: DMA not available, falling back to PIO\n", 
        dev_driver_string(&master->dev),
        dev_name(&master->dev));
        return -EAGAIN;
    }

    static void spi_imx_tail_pio_set(struct spi_imx_data *spi_imx, int left) {
        switch (spi_imx->rx_config.src_addr_width) {
        case DMA_SLAVE_BUSWIDTH_1_BYTE:
            spi_imx->rx = spi_imx_buf_rx_u8;
            break;
        case DMA_SLAVE_BUSWIDTH_2_BYTES:
            ...
        case DMA_SLAVE_BUSWIDTH_4_BYTES:
            ...
        default:
            spi_imx->rx = spi_imx_buf_rx_u8;
            break;
        }
        spi_imx->txfifo = left / spi_imx->tx_config.dst_addr_width;
    }
    ```
    - 在申请 *dma* 描述符时指定 *flags* 为 `DMA_PREP_INTERRUPT | DMA_PREP_ACK`，前者使得在此次 *dma* 操作完成后触发中断下半部（*tasklet*）执行后续设置的回调函数，`DMA_CTRL_ACK` 表明没有依赖链的需求。
    - 注意，在通过 *dma* 对 *spi rx* 进行处理时，设置了 *rx fifo* 的水线（当 *rx* 数据量超过水线值时触发 *dma* 数据搬运操作），因此要求通过 *dma* 转运的 *rx* 数据量可能得不到满足，水线以下的部分需要通过 *PIO* 进行处理。上述代码通过 `wml_mask` 获取最后一个 *scatterlist* 水线以上部分，剩余部分长度保存在 `left` 变量中，并且将最后一个 *scatterlist* 的 *dma* 传输长度修改为水线以上的长度。
    - `IMX_DMA_TIMEOUT`，当传输的数据长度小于 *1 MB* 时，超时时间为 *3s*。
    - [ ] `dma_sync_sg_for_cpu()` 的作用是什么，为什么这里的参数只是最后一个 scatterlist ？
    - 在对尾部需要接收的数据进行处理时，通过 `spi_imx_tail_pio_set()` 修改了 `spi_imx` 的接收处理函数和 `spi_imx->txfifo` 的值（这里表示仍有数据需要接收，见后面对该字段的分析）。而 `intctrl(spi_imx, TCEN)` 是等待前面通过 *DMA* 进行 *spi* 传输的完成，完成后调用中断服务函数 `spi_imx_isr`，发现仍有数据需要接收（`spi_imx->txfifo` 不为 0）...
- `spi_imx_pio_transfer` - 通过 中断（*PIO，Programmed I/O*，程序控制输入/输出） 进行数据传输
    ```c
    static int spi_imx_pio_transfer(struct spi_device *spi, struct spi_transfer *transfer) {
        struct spi_imx_data *spi_imx = spi_master_get_devdata(spi->master);
        spi_imx->tx_buf = transfer->tx_buf;
        spi_imx->rx_buf = transfer->rx_buf;
        spi_imx->count  = transfer->len;
        spi_imx->txfifo = 0;
        reinit_completion(&spi_imx->xfer_done);
        spi_imx_push(spi_imx);
        spi_imx->devtype_data->intctrl(spi_imx, MXC_INT_TE);
        wait_for_completion(&spi_imx->xfer_done);
        return transfer->len;
    }

    static void spi_imx_push(struct spi_imx_data *spi_imx) {
        while (spi_imx->txfifo < spi_imx_get_fifosize(spi_imx)) {
            if (!spi_imx->count)
                break;
            spi_imx->tx(spi_imx);
            spi_imx->txfifo++;
        }
        spi_imx->devtype_data->trigger(spi_imx);
    }

    // 接收函数定义
    #define MXC_SPI_BUF_RX(type) \
    static void spi_imx_buf_rx_##type(struct spi_imx_data *spi_imx) { \
        unsigned int val = readl(spi_imx->base + MXC_CSPIRXDATA); \
        if (spi_imx->rx_buf) {  \
            *(type *)spi_imx->rx_buf = val; \
            spi_imx->rx_buf += sizeof(type);    \
        }   \
    }

    // 发送函数定义
    #define MXC_SPI_BUF_TX(type) \
    static void spi_imx_buf_tx_##type(struct spi_imx_data *spi_imx) {   \
        type val = 0;   \
        if (spi_imx->tx_buf) {  \
            val = *(type *)spi_imx->tx_buf; \
            spi_imx->tx_buf += sizeof(type);    \
        }   \
        spi_imx->count -= sizeof(type); \
        writel(val, spi_imx->base + MXC_CSPITXDATA);    \
    }

    MXC_SPI_BUF_RX(u8)
    MXC_SPI_BUF_RX(u16)
    MXC_SPI_BUF_RX(u32)
    MXC_SPI_BUF_TX(u8)
    MXC_SPI_BUF_TX(u16)
    MXC_SPI_BUF_TX(u32)

    // 中断处理函数
    static irqreturn_t spi_imx_isr(int irq, void *dev_id) {
        struct spi_imx_data *spi_imx = dev_id;
        while (spi_imx->devtype_data->rx_available(spi_imx)) {
            spi_imx->rx(spi_imx);
            spi_imx->txfifo--;
        }
        if (spi_imx->count) {
            spi_imx_push(spi_imx);
            return IRQ_HANDLED;
        }
        if (spi_imx->txfifo) {
            /* No data left to push, but still waiting for rx
             * data, enable receive data available interrupt.
             */
            spi_imx->devtype_data->intctrl(spi_imx, MXC_INT_RR);
            return IRT_HANDLED;
        }
        spi_imx->devtype_data->intctrl(spi_imx, 0);
        complete(&spi_imx->xfer_done);
        return IRQ_HANDLED;
    }
    ```
    - *spi* 通信的特点使得接收和发送是同步进行的，接收和发送的数据量是一致的。
    - 数据传输分为几个阶段：将数据写入 *spi tx fifo*、操作 *CONTROL* 寄存器开始数据传输（即 `trigger()` 函数的调用）、从 *rx fifo* 中读取数据。其中需要明确两个变量的含义：
      - `spi_imx->txfifo`，指当前（已（发送）/要（接收））使用 *spi* 控制器 *fifo* 的 *entry* 个数。因此将数据写入到 *tx fifo* 时都要求 `spi_imx->txfifo` 的值要小于 `spi_imx_get_fifosize()` 函数的返回值。这也是 `spi_imx_push()` 中 "push" 的来源，因为它只是将数据写入到 *fifo* 中，并未开始传输；
      - `spi_imx->count`，指当前未放入 *tx fifo* 中数据量的字节数，也可以理解为还需要从 *rx fifo* 中读取的字节数，两者的值是相等的。
    - 综上，中断处理函数 `spi_imx_isr()` 中对应的几种情况：
      - 前：*tx fifo* 还存在未发送完成的数据，对应到接收端就是还有可从 *rx fifo* 中读取的数据。这部分处理 `spi_imx_push()` 的输入。
      - 中：此时，*tx fifo* 中没有可处理的数据，但还存在需要传输（发送/接收）的剩余数据，此时需要将发送数据（有效/无效，0）继续 "*push*" 到 *tx fifo* 中，以实现后续的传输。
      - 后：所有需要发送的数据已经写入 *tx fifo*，但 *tx fifo* 中的数据还未来得及发送完。此时的状态可能是还有有效数据需要发送，可能是还有数据需要接收，也可能两者都是。
- `spi_imx_chipselect()`
    ```c
    static void spi_imx_chipselect(struct spi_device *spi, int is_active) {
        struct spi_imx_data *spi_imx = spi_master_get_devdata(spi->master);
        int gpio = spi_imx->chipselect[spi->chip_select];
        int active = is_active != BITBANG_CS_INACTIVE;
        int dev_is_lowactive = !(spi->mode & SPI_CS_HIGH);
        if (!gpio_is_valid(gpio))
            return;
        gpio_set_value(gpio, dev_is_lowactive ^ active);
    }
    ```

#### 总结
1. 关于 DMA 的相关设置
   1. 设置 dma 的源地址和目的地址等参数
      - 在 spi 控制器初始化时（`spi_imx_probe()->spi_imx_sdma_init()`），就申请了 DMA 通道（`dma_request_slave_channel()`）并进行了设置 `dmaengine_slave_config()`，在这里将发送通道的目的地址设置为 SPI 控制器的 TX FIFO 地址、目的地址宽度为 1 字节、方向为 DMA_MEM_TO_DEV、dst_maxburst 设置为 fifo 数量的四分之一。接收通道同理。
      - 接收通道指定了目的端信息，发送通道指定了源端的信息，两者中的各自另一端的设置都在 `spi_map_msg()->spi_map_buf` 中完成（关于此部分的详细信息见 spi core 的解析）。
   2. 在 `spi_imx_setupxfer->spi_imx->devtype_data->config()` 中设置 dma 控制寄存器，使能或禁止发送和接收等 DMA 请求。即进行 spi_tranfer 的传输前，设置 dma 控制寄存器。只要在之前的 `spi_imx_sdma_init()` 中已正确完成DMA 的初始化（`spi_imx->dma_is_inited = 1`），在此处就设置 SPI 控制器发送和接收 FIFO 的水线和 DMA 使能。在这里水线是根据 fifo 中 entry 的个数设置的。
   3. 在 `spi_imx_setupxfer()` 设置 dma 的源地址和目的地址宽度。在 spi_transfer 传输前的设置函数（`spi_imx_setupxfer()`）中会进一步地根据 spi_transfer 或者 spi_device 的 bits per word 设置目的/源地址宽度，并调用 `dmaengine_slave_config()` 完成设置。
   4. 在 `spi_imx_dma_transfer()` 中申请 DMA 描述符（`dmaengine_prep_slave_sg()`）及设置回调相关内容、提交 DMA 传输任务（`dmaegine_submit()`）、开启传输（`dma_async_issue_pending()`）。
   5. 通过完成量（`wait_for_completion_timeout()`）等待完成传输。超时或失败时调用 `dmaengine_terminate_all()` 终止传输。
   6. 当前 transfer 能否进行 DMA 的判断：根据 transfer 中的字节数量与 spi fifo 的数量进行比较，前者大于后者则可以进行 DMA。
2. 调用链-设置
    ...
   1. spi_imx_sdma_init - 初始设置，申请 DMA 通道、设置 DMA 的源/目的端的地址、宽度、maxburst 以及方向等信息
   2. spi_map_msg - 设置目的/源端的地址
   3. spi_imx_setupxfer - 根据 spi_transfer 和 spi_device 进一步修正满足传输的源/目的端的宽度
    ...
3. 调用链-通信
   1. 
4. 通信部分
   - `struct spi_imx_data` 类型的数据 `spi_imx` 通过 `tx` 和 `rx` 成员函数分别实现 SPI 数据的发送和接收。
   - 通过 `spi_imx_setupxfer()` 函数来设置 `spi_imx` 的 `tx` 和 `rx` 函数。
   - 根据要处理的数据位宽有不同的通信函数：
        ```c
        // 接收
        spi_imx_buf_rx_u8
        spi_imx_buf_rx_u16
        spi_imx_buf_rx_u32
        // 发送
        spi_imx_buf_tx_u8
        spi_imx_buf_rx_u16
        spi_imx_buf_tx_u32

        // spi-imx.c
        #define MXC_SPI_BUF_TX(type)    \
        static void spi_imx_buf_tx_##type(struct spi_imx_data *spi_imx) \
        { \
            type val = 0; \
            if (spi_imx->tx_buf) { \
                val = *(type *)spi_imx->tx_buf; \
                spi_imx->tx_buf += sizeof(type); \
            }
            spi_imx->count -= sizeof(type); \
            writel(val, spi_imx->base + MXC_CSPITXDATA); \
        }
        MXC_SPI_BUF_RX(u8);
        MXC_SPI_BUF_TX(u8);
        ```
        - 函数的实现方式类似于 C++ 的模板函数。
        - 通信最终通过将数据写入 spi 的发送数据寄存器中实现。

### at25

## 参考
1. [Linux 系统的 SPI 设备编程](https://shaocheng.li/posts/2021/04/10/)-综合参考：电路、通信原理、时序、Linux...
2. [Linux 内核文档 - 串口外设接口 SPI](https://docs.linuxkernel.org.cn/spi/index.html)