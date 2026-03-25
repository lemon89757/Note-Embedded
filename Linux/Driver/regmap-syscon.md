# Note for Regmap and Syscon
## Regmap
Regmap 是 Linux 内核中引入的一种机制，**用于简化和统一对硬件寄存器的访问**。它抽象了 I2C、SPI 等总线的寄存器操作逻辑，提供了统一的接口，减少了驱动开发中的重复代码，提高了代码的可移植性和可维护性。

Regmap **在驱动和硬件之间添加了 cache**，降低了低速 I/O 的操作次数，提高了访问效率，缺点是实时性会降低。

使用 Regmap 的情况：
- 硬件寄存器操作，比如选用通过 I2C/SPI 接口来读写设备的内部寄存器，或者需要读写 SOC 内部的硬件寄存器；
- 提高代码复用性和驱动一致性，简化驱动开发过程；
- 减少底层 I/O 操作次数，提高访问效率；

### 框架
- [ ] 补充 Regmap 框架图

- 底层物理总线：Regmap 对不同的物理总线进行封装；
- Regmap 核心层：用于实现 Regmap；
- Regmap API 抽象层：向驱动编写人员提供 API 接口；

### 数据结构和 API
```c
// drivers/base/regmap/internal.h
struct regmap {
    //...
};

// include/linux/regmap.h
// read_flag_mask 和 write_flag_mask 寄存器地址相关的掩码
// 注意检查 spi 和 i2c 对于这两个掩码的默认设置是否符合要求
struct regmap_config {
    //...
};

// Regmap 的申请与初始化
// 对 regmap_init() 函数的进一步封装
// 一般在 probe 函数中初始化 regmap_config，然后申请并初始化 regmap
/* 
 * spi: 需要使用 regmap 的 spi_device 
 */
struct regmap *regmap_init_spi(struct spi_device *spi, const struct regmap_config *config);
/* 
 * i2c: 需要使用 regmap 的 i2c_client 
 */
struct regmap *regmap_init_i2c(struct i2c_client *i2c, const struct regmap_config *config);
/* Initialise register map */
static inline struct regmap *regmap_init_mmio(struct device *dev, void __iomem *regs, const struct regmap_config *config); 
/* Initialise managed register map */
static inline struct regmap *devm_regmap_init_mmio(struct device *dev, void __iomem *regs, const struct regmap_config *config);

// Regmap 的释放
void regmap_exit(struct regmap *map);

// 寄存器访问
// 统一的访问接口
int regmap_read(struct regmap *map, unsigned int reg, unsigned int *val);
int regmap_write(struct regmap *map, unsigned int reg, unsigned int val);
/* 
 * 修改寄存器指定的 bit 
 * mask: 需要更新的位必须在掩码中设置为 1
 */
int regmap_update_bits(struct regmap *map, unsigned int reg, unsigned int mask, unsigned int val);
/* 
 * 读取多个寄存器的值 
 * val_count: 要读取的寄存器数量
 */
int regmap_bulk_read(struct regmap *map, unsigned int reg, void *val, size_t val_count);
// 同上
int regmap_bulk_write(struct regmap *map, unsigned int reg, const void *val, size_t val_count);
```

### 示例代码
```c
// regmap 框架的核心就是 regmap 和 regmap_config 结构体，一般都是在自定义的设备结构体里面添加这两个类型的成员变量
struct icm20608_dev {
    struct spi_device *spi;
    dev_t devid;
    // ...
    struct regmap *regmap;
    struct regmap_config regmap_config;
}

// 在 probe 函数中初始化 regmap
static int icm20608_probe(struct spi_device *spi) {
    int ret;
    struct icm20608_dev *icm20608dev;

    icm20608dev = devm_kzalloc(&spi->dev, sizeof(*icm20608dev), GFP_KERNEL);
    if (!icm20608dev)
        return -ENOMEM;

    // 初始化 regmap_config
    icm20608dev->regmap_config.reg_bits = 8;    // 寄存器长度 8bit
    icm20608dev->regmap_config.val_bits = 8;    // 值长度 8bit
    icm20608dev->regmap_config.read_flag_mask = 0x80;   // 读掩码

    // 初始化 regmap
    icm20608dev->regmap = regmap_init_spi(spi, &icm20608dev->regmap_config);
    if (IS_ERR(icm20608->regmap))
        return PTR_ERR(icm20608->regmap);

    // 注册字符设备驱动
    // ...
}

// 在 remove 函数中删除 regmap
static int icm20608_remove(struct spi_device *spi) {
    struct icm20608_dev *icm20608 = spi_get_drvdata(spi);

    // ...

    class_destroy(icm20608_dev->class);
    regmap_exit(icm20608_dev->regmap);
    return 0;
}

// 读写设备内部寄存器
static unsigned char icm20608_read_onereg(struct imc20608_dev *dev, u8 reg) {
    u8 ret;
    unsigned int data;
    ret = regmap_read(dev->regmap, reg, &data);
    return (u8)data;
}

static void icm20608_write_onereg(struct icm20608_dev *dev, u8 reg, u8 value) {
    regmap_write(dev->regmap, reg, value);
}

// ...
```

## Syscon
System Control Driver, `drivers/mfd/syscon.c`. mfd, multiple function device.

### 示例程序
- 示例 1
    参考：[CSDN](https://blog.csdn.net/tiantao2012/article/details/73770520)
    如果在 dts 中有如下节点：
    ```dts
    dsaf0:dsa@c7000000 {
        #address-cells = <1>;
        #size-cells = <0>;
        compatible = "hisilicon,hns-dsaf-v2";
        mode = "6port-16rss";
        reg = <0x0 0xc5000000 0x0 0x890000
                0x0 0xc7000000 0x0 0x600000>;
        reg-names = "ppe-base", "dsaf-base";
        interrupt-parent = <&mbegin_dasf0>;
        subctrl-syscon = <&dsa_subctrl>;
    };
    ```
    可以通过 `of_parse_phandle()` 函数拿到 `subctrl-syscon`：
    ```c
    np_temp = of_parse_phandle(np, "subctrl-syscon", 0);
    ```
    `dsa_subctrl` 在设备树中的定义：
    ```dts
    dsa_subctrl: dsa_subctrl@c0000000 {
        compatible = "hisilicon,dsa-subctrl", "syscon";
        reg = <0x0 0xc0000000 0x0 0x10000>；
    };
    ```
    然后就可以通过 `syscon_node_to_regmap()` 来找到 `np_temp` 对应的 `regmap`：
    ```c
    struct regmap *syscon = syscon_node_to_regmap(np_temp);
    ```
    于是后续就可以通过 `syscon` 来读写 `dsa_subctrl` 中的寄存器。

- 示例 2
    参考：`/drivers/media/platform/mxc/subdev/mxc_mipi_csi.c`
    ```c
    static int mipi_csis_phy_reset(struct csi_state *state) {
        struct device_node *np = state->dev->of_node;
        struct device_node *node;
        phandle phandle;
        u32 out_val[3];
        int ret;

        ret = of_property_read_u32_array(np, "csis-phy-reset", out_val, 3);
        if (ret) {
            dev_dbg(state->dev, "no csis-hw-reset property found\n");
        } else {
            phandle = *out_val;

            node = of_find_node_by_phandle(phandle);
            if (!node) {
                dev_dbg(state->dev, "no find src node by phandle\n");
                ret = PTR_ERR(node);
            }
            state->hw_reset.src = syscon_node_to_regmap(node);
            if (IS_ERR(state->hw_reset.src)) {
                dev_err(state->dev, "failed to get src regmap\n");
                ret = PTR_ERR(state->hw_reset.src);
            }
            of_node_put(node);
            if (ret < 0)
                return ret;
            
            state->hw_reset.req_src = out_val[1];
            state->hw_reset.rst_bit = out_val[2];

            // reset mipi phy
            regmap_update_bits(state->hw_reset.src, state->hw_reset.req_src, 1 << state->hw_reset.rst_bit, 1 << state->hw_reset.rst_bit);
            msleep(20);
            regmap_update_bits(state->hw_reset.src, state->hw_reset.req_src, 1 << state->hw_reset.rst_bit, 0);
        }
        return ret;
    }
    ```
    设备树信息如下：
    ```dts
    // arch/arm/boot/dts/imx7d.dtsi
    // /soc:
    mipi_csi: mipi-csi@30750000 {
        compatible = "fsl, imx7d-mipi-csi";
        reg = <0x30750000 0x10000>;
        interrupts = <GIC_SPI 25 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&clks IMX7D_MIPI_CSI_ROOT_CLK>,
                <&clks IMX7D_MIPI_DPHY_ROOT_CLK>;
        clock-names = "mipi_clk", "phy_clk";
        mipi-phy-supply = <&reg_1p0d>;
        csis-phy-reset = <&src 0x28 2>;
        bus-width = <4>;
        status = "disable";
    };

    // /soc:
    src: src@30390000 {
        compatible = "fsl,imx7d-src", "fsl,imx51-src", "syscon";
        reg = <0x30390000 0x10000>;
        interrupts = <GIC_SPI 89 IRQ_TYPE_LEVEL_HIGH>;
        #reset-cells = <1>;
    };

    // /arch/arm/boot/dts/imx7d-sdb.dts
    // ...
    #include "imx7d.dtsi"

    &mipi_csi {
        clock-frequency = <240000000>;
        status = "okay";
        port {
            mipi_sensor_ep: endpoint1 {
                remote-endpoint = <&ov5640_mipi_ep>;
                data-lanes = <2>;
                csis-hs-settle = <13>;
                csis-clk-settle = <2>;
                csis-wclk;
            };

            csi_mipi_ep: endpoint2 {
                remote-endpoint = <&csi_ep>;
            };
        };
    };
    ```
    - `of_property_read_u32_array()` 找到 `<&src 0x28 2>`；
    - `phandle = *out_val` -> phandle => src；
    - `node = of_find_node_by_phandle(phandle)` 找到 src 对应的节点；
    - `syscon_node_to_regmap(node)` 通过 src 得到与之对应的 regmap；
    - `regmap_update_bits()` 在上述 regmap 上对寄存器进行读写；
 
### 源码分析
1. `syscon_node_to_regmap()` 的实现：
    ```c
    // /drivers/mfd/syscon.c
    static DEFINE_SPINLOCK(syscon_list_slock);
    static LIST_HEAD(syscon_list);

    struct syscon {
        struct device_node *np;
        struct regmap *regmap;
        struct list_head list;
    };

    static struct regmap_config syscon_regmap_config = {
        .reg_bits = 32;
        .val_bits = 32;
        // The register address stride.
        // Valid register addresses are a multiple of this value.
        .reg_stride = 4;
    }

    static struct syscon *of_syscon_register(struct device_node *np) {
        struct syscon *syscon;
        struct regmap *regmap;
        void __iomem *base;
        int ret;
        struct regmap_config syscon_config = syscon_regmap_config;

        if (!of_device_is_compatible(np, "syscon"))
            return ERR_PTR(-EINVAL);

        syscon = kzalloc(sizeof(*syscon), GFP_KERNEL);
        if (!syscon)
            return ERR_PTR(-ENOMEM);

        base = of_iomap(np, 0);
        if (!base) {
            ret = -ENOMEN;
            goto err_map;
        }

        // Parse the device's DT node for an endianness specification
        if (of_property_read_bool(np, "big-endian"))
            syscon_config.val_format_endian = REGMAP_ENDIAN_BIG;
        else if (of_property_read_bool(np, "little-endian"))
            syscon_config.val_format_endian = REGMAP_ENDIAN_LITTLE;
        
        regmap = regmap_init_mmio(NULL, base, &syscon_config);
        if (IS_ERR(regmap)) {
            pr_err("regmap init failed\n");
            ret = PTR_ERR(regmap);
            goto err_regmap;
        }

        syscon->regmap = regmap;
        syscon->np = np;

        spin_lock(&syscon_list_slock);
        list_add_tail(&syscon->list, &syscon_list);
        spin_unlock(syscon_list_slock);

        return syscon;
    
    err_regmap:
        iounmap(base);
    err_map:
        kfree(syscon);
        return ERR_PTR(ret);
    }

    struct regmap *syscon_node_to_regmap(struct device_node *np) {
        struct syscon *entry, *syscon = NULL;

        spin_lock(&syscon_list_slock);
        
        list_for_each_entry(entry, &syscon_list, list)
            if (entry->np == np) {
                syscon = entry;
                break;
            }
        
        spin_unlock(&syscon_list_slock);

        if (!syscon)
            syscon = of_syscon_register(np);

        if (IS_ERR(syscon))
            return ERR_CAST(syscon);
        
        return syscon->regmap;
    }
    EXPORT_SYSBOL_GPL(syscon_node_to_regmap);
    ```
    - 总的来说，`syscon_node_to_regmap` 的作用是通过设备树节点 `device_node` 获取与该节点对应的 `regmap`，从而能够对该节点的寄存器进行访问；
    - 要求：传入的设备树节点能够兼容："syscon"。syscon 框架会将新加入的 `struct syscon` 保存到自己的链表当中以便后续继续使用；
    - `struct syscon` 包含的信息有：`struct regmap*`、`struct device_node*` 以及 `struct list_head`；