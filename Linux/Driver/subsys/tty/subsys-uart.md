# uart 子系统
## 使用
串口驱动没有什么主机端和设备端之分，就只有一个串口驱动，而且这个驱动基本上由 SoC 厂商编写好了，驱动开发人员只需要在设备树中添加所要使用的串口节点信息。在驱动和设备匹配成功后，会生成 `/dev/ttyxxx` 入口文件（`ttyxxx` 根据不同的串口驱动程序有不同的命名方式）。

## uart 子系统
### 子系统架构

### 驱动描述
- `uart_driver`：
    ```c
    // version: 4.1.15
    // include/linux/serial_core.h
    struct uart_driver {
        struct module *owner;
        const char *driver_name;
        const char *dev_name;
        int major;
        int minor;
        int nr;                 // 设备数
        struct console *cons;   // 控制台

        /* 
         * these are private; the low level driver should not
         * touch these; they should be initialized to NULL 
         */
        struct uart_state *state;
        struct tty_driver *tty_driver;
    };
    ```
- 驱动注册与注销
    每个串口驱动都需要定义一个 `uart_driver`，加载驱动的时候通过 `uart_register_driver()` 函数向子系统注册：
    ```c
    // 返回值：0-成功；负值-失败
    int uart_register_driver(struct uart_driver *drv);
    ```
    驱动注销时通过 `uart_unregister_driver()` 函数：
    ```c
    void uart_unregister_driver(struct uart_driver *drv);
    ```

### 硬件信息描述
- `uart_port`
    ```c
    // version: 4.1.15
    // include/linux/serial_core.h
    struct uart_port {
        spinlock_t lock;        /* port lock */
        unsigned long iobase;   /* in/out [bwl] */
        unsigned char __iomem *membase;     /* read/write [bwl]，映射之后的虚拟地址 */
        ...
        const struct uart_ops *ops;
        unsigned int custom_divisor;
        unsigned int line;      /* port index */
        unsigned int minor;
        resource_size_t mapbase;    /* for ioremap，映射之前的物理地址 */
        resource_size_t mapsize;
        struct device *dev;
        ...
    };
    ```
    操作函数集字段：`ops`，包含了具体 UART 的驱动函数。uart 子系统的通信实现最终会调用到 `ops` 中的回调函数。
- 注册与注销：
    一个 `uart_driver` 可以支持多个 `uart_port` 的驱动、将 `uart_port` 添加到给定的 `uart_driver` 上：
    ```c
    // 返回值：0-成功；负值-失败
    int uart_add_one_port(struct uart_driver *drv, struct uart_port *uport);
    ```
    从 `uart_driver` 上将给定的 `uart_port` 删除：
    ```c
    // 返回值：0-成功；负值-失败
    int uart_remove_one_port(struct uart_driver *drv, struct uart_port *uport);
    ```
- `uart_port` 的操作函数集描述：
    ```c
    // version: 4.1.15
    // include/linux/serial_core.h
    struct uart_ops {
        unsigned int (*tx_empty)(struct uart_port *);
        void (*set_mctrl)(struct uart_port *, unsigned int mctrl);
        unsigned int (*get_mctrl)(struct uart_port *);
        void (*stop_tx)(struct uart_port *);
        void (*start_tx)(struct uart_port *);
        void (*throttle)(struct uart_port *);
        void (*unthrottle)(struct uart_port *);
        void (*send_xchar)(struct uart_port *);
        void (*stop_rx)(struct uart_port *);
        void (*enable_ms)(struct uart_port *);
        void (*break_ctrl)(struct uart_port *);
        int (*startup)(struct uart_port *);
        void (*shutdown)(struct uart_port *);
        void (*flush_buffer)(struct uart_port);
        void (*set_termios)(struct uart_port *, struct ktermios *new, struct ktermios *old);
        void (*set_ldisc)(struct uart_port *, struct ktermios *);
        void (*pm)(struct uart_port *, unsigned int state, unsigned int oldstate);

        /* Return a string describing the type of the port */
        const char *(*type)(struct uart_port *);

        /* 
         * Release IO and memory resources used by the port. 
         * This include iounmap if necessary.
         */
        void (*release_port)(struct uart_port *);

        /* 
         * Request IO and memory resources used by the port.
         * This include iomapping the port if necessary. 
         */
        int (*request_port)(struct uart_port *);
        
        void (*config_port)(struct uart_port *, int);
        int (*verify_port)(struct uart_port *, struct serial_struct *);
        int (*ioctl)(struct uart_port *, unsigned int, unsigned long);
    #ifdef CONFIG_CONSOLE_POLL
        int (*poll_init)(struct uart_port*);
        void (*poll_put_char)(struct uart_port *, unsigned char);
        int (*poll_get_char)(struct uart_port *);
    #endif // CONFIG_CONSOLE_POLL
    }
    ```
    - uart 驱动需要实现 `uart_ops`，它是最底层的 uart 驱动接口，实实在在和 uart 寄存器打交道的。
    - 回调函数说明见内核文档 `Documentation/serial/driver`。

## 源码阅读
### imx6ull
- 设备树描述
    ```dtsi
    // imx6ull.dtsi
    uart3: serial@021ec0000 {
        compatible = "fsl,imx6ul-uart", "fsl,imx6q-uart", "fsl,imx21-uart";
        reg = <0x021ec000 0x4000>;
        interrupts = <GIC_SPI 28 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&clks IMX6UL_CLK_UART3_IPG>, 
            <&clks IMX6UL_CLK_UART3_SERIAL>;
        clock-names = "ipg", "per";
        dmas = <&sdma 29 4 0>,
            <&sdma 30 4 0>;
        dma-names = "rx", "tx";
        status = "disabled";
    };
    ```

- 源文件
1. 入口/出口部分
    ```c
    // drivers/tty/serial/imx.c
    static struct platform_device_id imx_uart_devtype[] = {
        {
            .name = "imx1-uart",
            .driver_data = (kernel_ulong_t) &imx_uart_devdata[IMX1_UART],
        }, {
            .name = "imx21-uart",
            .driver_data = (kernel_ulong_t) &imx_uart_devdata[IMX21_UART],
        }, {
            .name = "imx6q-uart",
            .driver_data = (kernel_ulong_t) &imx_uart_devdata[IMX6Q_UART],
        }, {
            /* sentinel */
        }
    };
    MODULE_DEVICE_TABLE(platform, imx_uart_devtype);

    static const struct of_device_id imx_uart_dt_ids[] = {
        { 
            .compatible = "fsl,imx6q-uart", 
            .data = &imx_uart_devdata[IMX6Q_UART], 
        }, {
            .compatible = "fsl,imx1-uart",
            .data = &imx_uart_devdata[IMX1_UART],
        }, {
            .compatible = "fsl,imx21-uart",
            .data = &imx_uart_devdata[IMX21_UART],
        }, {
            /* sentinel */
        }
    };

    ...
    
    static struct platform_driver serial_imx_driver = {
        .probe = serial_imx_probe,
        .remove = serial_imx_remove,

        .suspend = serial_imx_suspend,
        .resume = serial_imx_suspend,
        .id_table = imx_uart_devtype,
        .driver = {
            .name = "imx-uart",
            of_match_table = imx_uart_dt_ids,
        },
    };

    static int __init imx_serial_init(void) {
        int ret = uart_register_driver(&imx_reg);

        if (ret)
            return ret;
        
        ret = platform_driver_register(&serial_imx_driver);
        if (ret != 0)
            uart_unregister_driver(&imx_reg);
        
        return ret;
    }

    static void __exit imx_serial_exit(void) {
        platform_driver_unregister(&serial_imx_driver);
        uart_unregister_driver(&imx_reg);
    }

    module_init(imx_serial_init);
    module_exit(imx_serial_exit);
    ```
    - 模块初始化顺序：`module_init`。
    - 基于 platform 驱动框架实现 uart 驱动。
    - 在入口函数中注册 `uart_driver`、注册 `platform_driver`；在出口函数中注销 `platform_driver`、注销 `uart_driver`。

2. probe/remove 部分
    `probe` 函数的重点工作就是分配、设置、注册 `uart_port`。在 imx6ull 源码中，`uart_port` 作为属性字段被包含在 `imx_port` 结构体中：
    ```c
    struct imx_port {
        struct uart_port port;
        struct timer_list timer;
        unsigned int old_status;
        unsigned int have_rtscts:1;
        unsigned int dte_mode:1;
        unsigned int irda_inv_rx:1;
        unsigned int irda_inv_tx:1;
        unsigned short trcv_delay;      // transceiver delay (收发器延时)
        ...
        unsigned long flags;
    };

    static int serial_imx_probe(struct platform_device *pdev) {
        struct imx_port *sport;
        void __iomem *base;
        int ret = 0;
        struct resource *res;
        int txirq, rxirq, rtsirq;

        sport = devm_kzalloc(&pdev->dev, sizeof(*sport), GFP_KERNEL);
        if (!sport)
            return -ENOMEM;
        
        ret = serial_imx_probe_dt(sport, pdev);
        if (ret > 0)
            serial_imx_probe_pdata(sport, pdev);
        else if (ret < 0)
            return ret;
    
        res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
        base = devm_ioremap_resource(&pdev->dev, res);
        if (IS_ERR(base)) 
            return PRT_ERR(base);
        
        rxirq = platform_get_irq(pdev, 0);
        txirq = platform_get_irq(pdev, 1);
        rtsirq = platform_get_irq(pdev, 2);

        sport->port.dev = &pdev->dev;
        sport->port.mapbase = res->start;
        sport->port.membase = base;
        sport->port.type = PORT_IMX,
        sport->port.iotype = UPIO_MEM;
        sport->port.irq = rxirq;
        sport->port.fifosize = 32;
        sport->port.ops = &imx_pops;
        sport->port.rs485_config = imx_rs485_config;
        sport->port.rs485.flags = 
            SER_RS485_RTS_ON_SEND | SER_RS485_RX_DURING_TX;
    }
    ```

3. `uart_driver` 驱动描述
    ```c
    static struct uart_driver imx_reg = {
        .owner = THIS_MODULE,
        .driver_name = DRIVER_NAME,
        .dev_name = DEV_NAME,
        .major = SERIAL_IMX_MAJOR,
        .minor = MINOR_START,
        .nr = ARRAY_SIZE(imx_ports),
        .cons = IMX_CONSOLE,
    };
    ```

## 参考
1. 正点原子-I.MX6U 嵌入式驱动开发指南
2. 内核文档：`Documentation/serial/driver`