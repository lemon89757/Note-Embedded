# SPI 子系统
## 概述

## 使用
### 设备树相关节点描述
- `pinctrl` 节点
- `spi` 设备节点
    ```c
    // imx6qdl-sabresd.dtsi
    &ecspi1 {
        fsl,spi-num-chipselects = <1>;
        cs-gpios = <&gpio4 9 0>;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_ecspi>;
        status = "okay";

        flash: m25p80@0 {
            #address-celss = <1>;
            #size-celss = <1>;
            compatible = "st,m25p32";
            spi-max-frequency = <20000000>;
            reg = <0>;
        };
    };
    ```
    - spi 控制器节点描述
      - [ ] `fsl,spi-num-chipselects` 是片选引脚的数量吗？（1，表示只有一个设备）；
      - `cs-gpios`: 片选引脚；
    - spi 设备节点描述
      - spi 设备节点作为 spi 控制器的子节点。
      - `spi-max-frequency` 设置 SPI 控制器的最高频率，这个要根据所使用的 SPI 设备来设置。
      - `reg` 表示 m25p80 这个设备所使用的 ECSPI 通道，这里为通道 0。

### 设备驱动- `struct spi_driver`
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

### 通信数据表示-`spi_transfer`
- 通过 `Master` 与设备进行通信。

- `spi_transfer` 结构体用于描述 SPI 传输信息。
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
    - `tx_buf` 保存着要发送的数据。
    - `rx_buf` 用于保存接收到的数据。
    - `len` 是要进行传输的数据长度，SPI 为全双工通信、一次通信中发送和接收的字节数都是一样的，因此也就没有发送长度和接收长度之分。

- `spi_transfer` 需要组织成 `spi_message`：
    ```c
    /**
    * struct spi_message - one multi-segment SPI transaction
    * @transfers: list of transfer segments in this transaction
    * @spi: SPI device to which the transaction is queued
    * @is_dma_mapped: if true, the caller provided both dma and cpu virtual
    *	addresses for each transfer buffer
    * @complete: called to report transaction completions
    * @context: the argument to complete() when it's called
    * @frame_length: the total number of bytes in the message
    * @actual_length: the total number of bytes that were transferred in all
    *	successful segments
    * @status: zero for success, else negative errno
    * @queue: for use by whichever driver currently owns the message
    * @state: for use by whichever driver currently owns the message
    *
    * A @spi_message is used to execute an atomic sequence of data transfers,
    * each represented by a struct spi_transfer.  The sequence is "atomic"
    * in the sense that no other spi_message may use that SPI bus until that
    * sequence completes.  On some systems, many such sequences can execute as
    * as single programmed DMA transfer.  On all systems, these messages are
    * queued, and might complete after transactions to other devices.  Messages
    * sent to a given spi_device are always executed in FIFO order.
    *
    * The code that submits an spi_message (and its spi_transfers)
    * to the lower layers is responsible for managing its memory.
    * Zero-initialize every field you don't set up explicitly, to
    * insulate against future API updates.  After you submit a message
    * and its transfers, ignore them until its completion callback.
    */
    struct spi_message {
        struct list_head	transfers;

        struct spi_device	*spi;

        unsigned		is_dma_mapped:1;

        /* REVISIT:  we might want a flag affecting the behavior of the
        * last transfer ... allowing things like "read 16 bit length L"
        * immediately followed by "read L bytes".  Basically imposing
        * a specific message scheduling algorithm.
        *
        * Some controller drivers (message-at-a-time queue processing)
        * could provide that as their default scheduling algorithm.  But
        * others (with multi-message pipelines) could need a flag to
        * tell them about such special cases.
        */

        /* completion is reported through a callback */
        void			(*complete)(void *context);
        void			*context;
        unsigned		frame_length;
        unsigned		actual_length;
        int			status;

        /* for optional use by whatever driver currently owns the
        * spi_message ...  between calls to spi_async and then later
        * complete(), that's the spi_master controller driver.
        */
        struct list_head	queue;
        void			*state;
    };
    ```
    - 操作顺序及函数接口
    ```c
    // 使用 spi_message 之前需初始化
    void spi_message_init(struct spi_message *m);
    // 初始化完成后，将 spi_transfer 添加到 spi_message 队列中
    void spi_message_add_tail(struct spi_transfer *t, struct spi_message *m);
    // spi_message 准备好后即可传输
    // 同步传输，阻塞等待直到 SPI 传输完成
    // 返回值-无
    int spi_sync(struct spi_device *spi, struct spi_message *message);
    // 异步传输，非阻塞
    // 需设置 complete 成员变量，回调函数，当 SPI 异步传输完成后调用此函数
    // 返回值-无
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
- 综上，SPI 数据传输步骤：
  1. 申请并初始化 `spi_transfer`，设置 `spi_transfer` 的 `tx_buf`、`rx_buf`、`len` 成员变量，如果不需要发送/接收数据，则可以将相应成员变量的初始值设置为 `NULL`（不初始化）。
  2. 使用 `spi_message_init()` 初始化 `spi_message`。
  3. 使用 `spi_message_add_tail()` 将设置好的 `spi_transfer` 添加到 `spi_message` 队列中。
  4. 通过 `spi_sync()/spi_async()` 调用完成 SPI 数据同步/异步传输（设置相应回调）。


## SPI 子系统
内核版本为：`4.1.15`。
### SPI Core
1. spi_bus_type
    ```c
    // drivers/spi/spi.c
    struct bus_type spi_bus_type = {
        .name = "spi",
        .dev_groups = spi_dev_groups,
        .match = spi_match_device,
        .uevent = spi_uevent,
    };

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
    ```
    - 设备和驱动是否匹配的比较顺序：`OF->ACPI->id_table-> spi->modalias 和 drv->name`。


### Master 驱动
```c
// include/linux/spi/spi.h
struct spi_master {
    struct device dev;
    struct list_head list;
    ...
    s16 bus_num;
    /* chipselects will be integral to many controllers; 
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
    ...
    /* limits on transfer speed */
    u32 min_speed_hz;
    u32 max_speed_hz;
    /* other constraints relevant to this driver. */
    u16 flags;
    ...
    /* lock and mutex for SPI bus locking */
    spinlock_t bus_lock_spinlock;
    struct mutex bus_lock_mutex;

    /* flag indicating that the SPI bus is locked for exclusive use */
    bool bus_lock_flag;
    ...
    int (*setup)(struct spi_device *spi);
    ...
    int (*transfer)(struct spi_device *spi, struct spi_message *mesg);
    ...
    int (*transfer_one_message)(struct spi_master *master, struct spi_message *mesg);
    ...
};
```
- `transfer()` 即控制器数据传输函数，`transfer_one_message()` 函数也用于 SPI 数据发送，用于发送一个 `spi_message`，SPI 的数据会打包成 `spi_message`，然后以队列方式发送出去。
- SPI Master 驱动的核心就是申请 `spi_master`，然后初始化 `spi_master`，最后注册 `spi_master`。
- SPI Master 驱动作为 `platform_driver` 注册到 `platform` 总线上。
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

### 设备驱动

### I2C 和 SPI 子系统对比

## 源码阅读
### imx6ull
- 设备树描述
```dts
ecspi3: ecspi@02010000 {
    #address-cells = <1>;
    #size-cells = <0>;
    compatible = "fsl,imx6ul-ecspi", "fsl,imx51-ecspi";
    reg = <0x02010000 0x4000>;
    interrupts = <GIC_SPI 33 IRQ_TYPE_LEVEL_HIGH>;
    clock = <&clks IMX6UL_CLK_ECSPI3>,
        <&clks IMX6UL_CLK_ECSPI3>;
    clock-names = "ipg", "per";
    dmas = <&sdma 7 7 1>, <&sdma 8 7 2>;
    dma-names = "rx", "tx";
    status = "disabled";
};
```
- 源文件
1. 入口/出口部分
```c
// drivers/spi/spi-imx.c
static struct platform_device_id spi_imx_devtype[] = {
    {
        .name = "imx1-cspi",
        .driver_data = (kernel_ulong_t) &imx1_cspi_devtype_data,
    }, {
        .name = "imx21-cspi",
        .driver_data = (kernel_ulong_t) &imx21_cspi_devtype_data,
    }, {
        .name = "imx27-cspi",
        .driver_dat = (kernel_ulong_t) &imx27_cspi_devtype_data,
    },
    ... {
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

...

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

2. probe/remove 部分
    ```c
    ```
3. 通信部分
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