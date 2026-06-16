# I2C 子系统
## I2C 子系统的模块划分
背景：
划分模块：

## I2C Core
1. init-exit
    ```c
    static int __init i2c_init(void) {
    ...
    retval = bus_register(&i2c_bus_type);
    if (retval)
        return retval;
    ...
    }

    static void __exit i2c_exit(void) {
        ...
        bus_unregister(&i2c_bus_type);
        ...
    }

    /* We must initialize early, because some subsystems register 
     * i2c drivers in subsys_initcall() code, but are linked
     * (and initialized) before i2c.
     */
    postcore_initcall(i2c_init);
    module_exit(i2c_exit);
    ```
    - `postcore_initcall` 表明 i2c 子系统的初始化在 `subsys_init` 之前。
2. `i2c_bus_bype`
    ```c
    // drivers/i2c/i2c-core.c
    struct bus_type i2c_bus_type = {
        .name       = "i2c",
        .match      = i2c_device_match,
        .probe      = i2c_device_probe,
        .remove     = i2c_device_remove,
        .shutdown   = i2c_device_shutdown,
    };

    static int i2c_device_match(struct device *dev, struct device_driver *drv) {
        struct i2c_client *client = i2c_verify_client(dev);
        struct i2c_driver *driver;

        if (!client)
            return 0;
        
        // Attempt an OF style match
        if (of_driver_match_device(dev, drv))
            return 1;
        // Then ACPI style match
        if (acpi_driver_match_device(dev, drv))
            return 1;
        driver = to_i2c_driver(drv);
        if (driver->id_table)
            return i2c_match_id(driver->id_table, client) != NULL;
        return 0;
    }

    static int i2c_device_probe(struct device *dev) {
        struct i2c_client *client = i2c_verify_client(dev);
        struct i2c_driver *driver;
        int status;

        ... // 设置 client 的 irq，设置时钟，电源相关...
        if (!driver->probe || !driver->id_table)
            return -ENODEV;

        if (status != -EPROBE_ERROR)
            status = driver->probe(client, i2c_match_id(driver->id_table, client));
        ...
    }

    struct i2c_client *i2c_verify_client(struct device *dev) {
        return (dev->type == &i2c_client_type) ? to_i2c_client(dev) : NULL;
    }
    ```
    - 设备和驱动是否匹配的比较顺序：`OF->ACPI->i2c_device_id`；
    - 在设备和驱动匹配完成后，`probe` 函数被调用：`i2c_bus_type.probe -> i2c_driver.probe`；
    - 无论是在 `match` 还是 `probe` 中，都要求设置 `i2c_driver` 的 `id_table` 字段，即设置 `i2c_device_id`。

2. [ ] 架构图，注册、匹配流程。

## Adapter 驱动
```c
// include/linux/i2c.h
struct i2c_adapter {
    struct module *owner;
    unsigned int class;                 // class to allow probing for
    const struct i2c_algorithm *algo;   // 总线访问算法
    void *alog_data;

    //data fields that are valid for all devices
    struct rt_mutex bus_lock;

    int time_out;                       // in jiffies
    int retries;
    struct device dev;                  // the adapter device

    in nr;
    char name[48];
    struct completion dev_released;

    struct mutex userspace_clients_lock;
    struct list_head userspace_clients;

    struct i2c_bus_recovery_info *bus_recovery_info;
    const struct i2c_adapter_quirks *quirks;
};

struct i2c_algorithm {
    ...
    // I2C 适配器传输函数
    int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
    // SMBUS 总线传输函数
    int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr, unsigned short flags, char read_write, u8 command, int size, union i2c_smbus_data *data);
    // To determine what the adapter supports
    u32 (*functionality)(struct i2c_adapter *);
    ...
};

// 注册 i2c_adapter
// 0-成功；负数-失败
int i2c_add_adapter(struct i2c_adapter *adapter);               // 使用动态的总线号
// 0-成功；负数-失败
int i2c_add_numbered_adapter(struct i2c_adapter *adap);     // 使用静态总线号

// 删除 i2c_adapter
void i2c_del_adapter(struct i2c_adapter *adap);
```
- I2C 适配器驱动的主要工作就是初始化 `i2c_adapter` 结构体变量，然后设置 `i2c_algorithm` 中的 `master_xfer` 函数，最后注册到系统中。
- I2C 适配器驱动作为 `platform_driver` 注册到 `platform` 总线上。
- 设置、获取 I2C Adapter 驱动数据：`i2c_set_adapdata()`，`i2c_get_adapdata()`。

## Client 驱动
1. 设备硬件信息描述（消费端）-`i2c_client`
    ```c
    // include/linux/i2c.h
    struct i2c_client {
        unsigned short flags;
        unsigned short addr;            // 芯片地址，位于低 7 位
        ...
        char name[I2C_NAME_SIZE];       
        struct i2c_adapter *adapter;    // 对应的 I2C 适配器
        struct device dev;
        int irq;
        struct list_head detected;
        ...
    };
    ```
    - 当 Adapter 向系统中注册时，会解析设备树中位于该 Adapter 节点下的所有子节点为 `struct i2c_client`，同时将 `adapter` 指向该 Adapter。

2. 设备硬件信息描述（生产端）
   - 未使用设备树时
        ```c
        struct i2c_board_info {
            char type[I2C_NAME_SIZE];
            unsigned short flags;
            unsigned short addr;
            void *platform_data;
            struct dev_archdata *archdata;
            struct device_node *of_node;
            struct fwnode_handle *fwnode;
            int irq;
        };
        ```
      - `type` 和 `addr` 是必须要设置的，一个是 I2C 设备名字，一个是 I2C 设备的器件地址；
      - 定义好的 `i2c_board_info` 可作为自定义设备结构体的一个指针字段的值，同时该自定义设备结构体又作为 `platform_device.dev.platorm_data` 的值，最后被添加到系统中（参考 `arch/arm/mach-imx/mach-mx27_3ds.c`）。
   - 使用设备树时
        ```dts
        // imx6ull-14x14-evk.dts
        &i2c1 {
            clock-frequency = <100000>;
            pinctrl-names   = "default";
            pinctrl-0       = <&pinctrl_i2c1>;
            status          = "okay";

            mag3110@0e {
                compatible = "fsl,mag3110";
                reg        = <0x0e>;
                position   = <2>;
            };

            ...
        };
        ```

3. 驱动结构体-`i2c_driver`
    ```c
    // include/linux/i2c.h
    struct i2c_driver {
        unsigned int class;

        /* Notifies the driver that a new bus has appeared. You should
        avoid using this, it will be removed in a near future. */
        int (*attach_adapter)(struct i2c_adapter *) __deprecated;

        /* Standard driver model interfaces */
        int (*probe)(struct i2c_client *, const struct i2c_device_id *);
        int (*remove)(struct i2c_client *);

        /* driver model interfaces that don't relate to enumeration */
        void (*shutdown)(struct i2c_client *);

        /* Alert callback, for example for the SMBus alert protocol. 
         * The format and meaning of the data value depends on the
         * protocol. For the SMBus alert protocol, there is a single
         * bit of data passed as the alert response's low bit ("event flag").
         */
        void (*alert)(struct i2c_client *, unsigned int data);

        /* a ioctl like command that can be used to perform specific 
         * functions with the device.
         */
        int (*command)(struct i2c_client *client, unsigned int cmd, void *arg);

        struct device_driver driver;
        const struct i2c_device_id *id_table;

        /* Device detection callback for automatic device creation */
        int (*detect)(struct i2c_client *, struct i2c_board_info *);
        const unsigned short *address_list;
        struct list_head clients;
    };

    // 注册
    // 0-成功；负值-失败
    int i2c_register_driver(struct module *owner, struct i2c_driver *driver);

    #define i2c_add_driver(driver) \
        i2c_register_driver(THIS_MODULE, driver)
    
    // 注销
    void i2c_del_driver(struct i2c_driver *driver);
    ```
    - 一般需要在 `probe` 函数中初始化 I2C 设备。

4. `i2c_driver` 注册示例：
    ```c
    static int xxx_probe(struct i2c_client *client, const struct i2c_device_id *id) {}

    static int xxx_remove(struct i2c_client *client) {}

    // 传统匹配方式 ID 列表
    static const struct i2c_device_id xxx_id[] = {
        {"xxx", 0},
        { /* sentinel */ }
    };

    // 设备树匹配表
    static const struct of_device_id xxx_of_match[] = {
        {.compatible = "xxx" },
        { /* sentinel */ }
    };

    static struct i2c_driver xxx_driver = {
        .probe = xxx_probe,
        .remove = xxx_remove,
        .driver = {
            .owner = THIS_MODULE,
            .name  = "xxx",
            .of_match_table = xxx_of_match,
        },
        .id_table = xxx_id,
    };

    static int __init xxx_init(void) {
        int ret = 0;
        ...
        ret = i2c_add_driver(&xxx_driver);
        if (ret)
            return -EFAULT;
        ...
    }

    static void __exit xxx_exit(void) {
        ...
        i2c_del_driver(&xxx_driver);
        ...
    }

    module_init(xxx_init);
    module_exit(xxx_exit);
    MODULE_LICENSE("GPL");
    ```

5. 通过 Adapter 与 Client 进行通信
   - 通信函数
    ```c
    // 负值-失败；其他非负值-发送 msgs 数量
    int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);

    // 对 i2c_transfer 的进一步封装
    // 数据发送
    /* @count: 要发送的数据字节数，要求：小于 64KB，因为 i2c_msg 的 len 成员类型为无符号 16 位。 
     * @return: 负值-失败；其他非负值-发送的字节数。
     */
    int i2c_master_send(const struct i2c_client *client, const char *buf, int count);

    // 数据接收
    /* @count: 要接收的数据字节数，要求：同上 
     * @return: 负值-失败；其他非负值-接收的字节数
     */
    int i2c_master_recv(const struct i2c_client *client, char *buf, int count);
    ```
   - 数据表示
    ```c
    // include/uapi/linux/i2c.h
    struct i2c_msg {
        __u16 addr;
        __u16 flags;
        #define I2C_M_TEN       0x0010
        #define I2C_M_RD        0x0001
        #define I2C_M_STOP      0x8000
        #define I2C_M_NOSTART   0x4000
        #define I2C_M_REV_DIR_ADDR  0x2000
        #define I2C_M_IGNORE_NAK    0x1000
        #define I2C_M_NO_RD_ACK     0x0800
        #define I2C_M_RECV_LEN      0x0400
        __u16 len;
        __u8 *buf;
    };
    ```
   - 通信示例
    ```c
    struct xxx_dev {
        ...
        void *private_data;     // 私有数据，一般会设置为 i2c_client
    };

    static int xxx_read_regs(struct xxx_dev *dev, u8 reg, void *val, int len) {
        int ret;
        struct i2c_msg msg[2];
        struct i2c_client *client = (struct i2c_client *)dev->private_data;

        // 第一条消息，发送数据，发送要读取的寄存器首地址
        msg[0].addr = client->addr;
        msg[0].flags = 0;           // 标记为发送数据
        msg[0].buf = &reg;
        msg[0].len = 1;

        // 第二条消息，读取寄存器数据
        msg[1].addr = client->addr;
        msg[1].flags = I2C_M_RD;    // 标记为读取数据
        msg[1].buf = val;
        msg[1].len = len;

        ret = i2c_transfer(client->adapter, msg, 2);
        if (ret == 2)
            ret = 0;
        else
            ret = -EREMOTEIO;
        return ret;
    }

    static s32 xxx_write_regs(struct xxx_dev *dev, u8 reg, u8 *buf, u8 len) {
        u8 b[256];
        struct i2c_msg msg;
        struct i2c_client *client = (struct i2c_client *)dev->private_data;

        b[0] = reg;                 // 第一个字节数据为寄存器首地址
        memcpy(&b[1], buf, len);    // 从第二个字节开始为需要发送的数据
        msg.addr = client->addr;
        msg.flags = 0;              // 标记为写数据
        msg.buf = b;
        msg.len = len + 1;          // 长度为需要发送数据的长度 + 1

        return i2c_transfer(client->adapter, &msg, 1);
    }
    ```

## 源码阅读
### I.MX6UL 的 I2C 适配器驱动
- 设备树描述
    ```dts
    i2c1: i2c@021a0000 {
        #address-cell = <1>;
        #size-cells = <0>;
        compatible = "fsl,imx6ul-i2c", "fsl,imx21-i2c";
        reg = <0x021a0000 0x4000>;
        interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&clk IMX6UL_CLK_I2C1>;
        status = "disabled";
    };
    ```
- 源代码
    ```c
    // drivers/i2c/buses/i2c-imx.c
    static struct platform_device_id imx_i2c_devtype[] = {
        {
            .name = "imx1-i2c",
            .driver_data = (kernel_ulong_t)&imx1_i2c_hwdata,
        },
        {
            .name = "imx21-i2c",
            .driver_data = (kernel_ulong_t)&imx21_i2c_hwdata,
        },
        {
            /* sentinel */
        }
    };
    MODULE_DEVICE_TABLE(platform, imx_i2c_devtype);

    static const struct of_device_id i2c_imx_dt_ids[] = {
        { .compatible = "fsl,imx1-i2c", .data = &imx1_i2c_hwdata, },
        { .compatible = "fsl,imx21-i2c", .data = &imx21_i2c_hwdata, },
        { .compatible = "fsl,vf610-i2c", .data = &vf610_i2c_hwdata, },
        { /* sentinel */}
    };
    MODULE_DEVICE_TABLE(of, i2c_imx_dt_ids);
    ...

    static u32 i2c_imx_func(struct i2c_adapter *adapter) {
        return I2C_FUNC_I2C | I2C_FUNC_SMBUS_EMUL | I2C_FUNC_SMBUS_READ_BLOCK_DATA;
    }

    static int i2c_imx_xfer(struct i2c_adapter *adapter, struct i2c_msg *msgs, int num) {
        unsigned int i, temp;
        int result;
        bool is_lastmsg = false;
        struct imx_i2c_struct *i2c_imx = i2c_get_adapdata(adapter);

        dev_dbg(&i2c_imx->adapter.dev, "<%s>\n", __func__);

        /* Start I2C transfer */
        result = i2c_imx_start(i2c_imx);
        if (result)
            goto fail0;

        /* read/write data */
        for (i = 0; i < num; i++) {
            if (i == num - 1)
                is_lastmsg = true;
            
            if (i) {
                dev_dbg(&i2c_imx->adapter.dev, "<%s> repeated start\n", __func__);
                temp = imx_i2c_read_reg(i2c_imx, IMX_I2C_I2CR);
                temp |= I2CR_RSTA;
                imx_i2c_write_reg(temp, i2c_imx, IMX_I2C_I2CR);
                result = i2c_imx_bus_busy(i2c_imx, 1);
                if (result)
                    goto fail0;
            }
            dev_dbg(&i2c_imx->adapter.dev, "<%s> transfer message: %d\n", __func__, i);

            // write/read data
            ...
            if (msgs[i].flags & I2C_M_RD)
                result = i2c_imx_read(i2c_imx, &msgs[i], is_lastmsg);
            else {
                if (i2c_imx->dma && msgs[i].len >= DMA_THRESHOLD)
                    result = i2c_imx_dma_write(i2c_imx, &msgs[i]);
                else
                    result = i2c_imx_write(i2c_imx, &msgs[i]);
            }
            if (result)
                goto fail0;
        }
    fail0:
        /* stop I2C transfer */
        i2c_imx_stop(i2c_imx);
        dev_dbg(&i2c_imx->adapter.dev, "<%s> exit with: %s: %d\n", __func__, (result < 0) ? "error" : "success msg", (result < 0) ? result : num);
        return (result < 0) ? result : num;
    }

    static struct i2c_algorithm i2c_imx_algo = {
        .master_xfer = i2c_imx_xfer,
        .functionality = i2c_imx_func,
    };

    static int i2c_imx_probe(struct platform_device *pdev) {
        const struct of_device_id *of_id = of_match_device(i2c_imx_dt_id, &pdev->dev);
        struct imx_i2c_struct *i2c_imx;
        struct resource *res;
        struct imxi2c_platform_data *pdata = dev_get_platdata(&pdev->dev);
        void __iomem *base;
        int irq, ret;
        dma_addr_t phy_addr;

        dev_dbg(&pdev->dev, "<%s>\n", __func__);

        irq = platform_get_irq(pdev, 0);
        ...
        res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
        base = devm_ioremap_resource(&pdev->dev, res);
        if (IS_ERR(base))
            return PTR_ERR(base);
        
        phy_addr = (dma_addr_t)res->start;
        i2c_imx = devm_kzalloc(&pdev->dev, sizeof(*i2c_imx), GFP_KERNEL);
        if (!i2c_imx)
            return -ENOMEM;
        
        if (of_id)
            i2c_imx->hwdata = of_id->data;
        else
            i2c_imx->hwdata = (struct imx_i2c_hwdata *)platform_get_device_id(pdev)->driver_data;
        
        /* setup i2c_imx driver structure */
        i2c_imx->adapter.owner = THIS_MODULE;
        i2c_imx->adapter.algo  = &i2c_imx_algo;
        i2c_imx->adapter.dev.parent = &pdev->dev;
        i2c_imx->adapter.nr    = pdev->id;
        i2c_imx->adapter.dev.of_node = pdev->dev.of_node;
        i2c_imx->base          = base;

        /* get i2c clock */
        i2c_imx->clk = devm_clk_get(&pdev->dev, NULL);
        ...
        ret = clk_prepare_enable(i2c_imx->clk);
        ...

        /* request IRQ */
        ret = devm_request_irq(&pdev->dev, irq, i2c_imx_isr, IRQF_NO_SUSPEND, pdev->name, i2c_imx);

        /* init queue */
        init_waitqueue_head(&i2c_imx->queue);

        /* set up adapter data */
        i2c_set_adapdata(&i2c_imx->adapter, i2c_imx);

        /* set up clock divider */
        i2c_imx->bitrate = IMX_I2C_BIT_RATE;
        ret = of_property_read_u32(pdev->dev.of_node, "clock-frequency", &i2c_imx->bitrate);
        if (ret < 0 && pdata && pdata->bitrate)
            i2c_imx->bitrate = pdata->bitrate;
        
        /* set up chip registers to defaults */
        // 设置 I2C1 控制器的 I2CR 和 I2SR 寄存器
        imx_i2c_write_reg(i2c_imx->hwdata->i2cr_ien_opcode ^ I2CR_IEN, i2c_imx, IMX_I2C_I2CR);
        imx_i2c_write_reg(i2c_imx->hwdata->i2sr_clr_opcode, i2c_imx, IMX_I2C_I2SR);

        /* add i2c adapter */
        ret = i2c_add_numbered_adapter(&i2c_imx->adapter);
        if (ret < 0) {
            dev_err(&pdev->dev, "registration failed\n");
            goto clk_disable;
        }

        /* set up platform driver data */
        platform_set_drvdata(pdev, i2c_imx);
        clk_disable_unprepare(i2c_imx->clk);
        ...
        /* init dma config if supported */
        i2c_imx_dma_request(i2c_imx, phy_addr);
        
        return 0;   // return ok

    clk_disable:
        clk_disable_unprepare(i2c_imx->clk);
        return ret;
    }

    static struct platform_driver i2c_imx_driver = {
        .probe = i2c_imx_probe,
        .remove = i2c_imx_remove,
        .driver = {
            .name = DRIVER_NAME,
            .owner = THIS_MODULE,
            .of_match_table = i2c_imx_dt_ids,
            .pm = IMX_I2c_PM,
        },
        .id_table = imx_i2c_devtype,
    };

    static int __init i2c_adap_imx_init(void) {
        return platform_driver_register(&i2c_imx_driver);
    }
    subsys_initcall(i2c_adap_imx_init);

    static void __exit i2c_adap_imx_exit(void) {
        platform_driver_unregister(&i2c_imx_driver);
    }
    module_exit(i2c_adap_imx_exit);
    ```
    - 适配器驱动的启动顺序：`subsys_initcall()`。
    - 在 `i2c_imx` 中对 `adpater->nr` 字段进行了设置：`i2c_imx->adapter.nr = pdev->id;`，因此后续将 `adapter` 注册到总线上时使用的 API 是：`i2c_add_numbered_adapter()`。如果希望由 I2C 总线动态分配 `nr` 字段的值，则需要调用 `i2c_add_adapter()` API。
    - `i2c_algorithm` 的 `functionality` 回调用于返回 I2C 适配器支持的通信协议。
    - `i2c_imx_start`, `i2c_imx_read`, `i2c_imx_write` 和 `i2c_imx_stop` 等函数是 I2C 寄存器的操作函数。

## 参考
1. 正点原子-I.MX6U 嵌入式 Linux 驱动开发指南
