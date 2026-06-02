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
/**
 * Configuration for the register map of a device. 
 * @name: Optional name of the regmap. Useful when a device has multiple register regions.
 * @reg_bits: Number of bits in a register address, mandatory（强制的）.
 * @reg_stride: 步伐，步幅。The register address stride. Valid register address are a multiple of this value. If set to 0, a value of 1 will be used.
 * @pad_bits: Number of bits of padding between register and value.
 * @val_bits: Number of bits in a register value, mandatory.
 * @writeable_reg: Optional callback returning true if the register can be written to. If this field is NULL but wr_table is not, the check is performed on such table (a register is writeable if it belongs to one of the specified by wr_able).
 * @readable_reg: Optional callback returning true if the register can be read from. ...
 * @volatile_reg: Optional callback returning true if the register value can't be cached. ...
 * @precious_reg: Optioanl callback returning true if the register should not be read outside of a call from the driver (e.g., a clear on read interrupt status register). ...
 * @lock: Optional lock callback (overrides regmap's default lock function, based on spinlock or mutex).
 * @unlock: As above for unlocking.
 * @lock_arg: this field is passed as the only argument of lock/unlock functions (ignored in case regular lock/unlock functions are not overridden).
 * @reg_read: Optional callback that if filled will be used to perform all the reads from the registers. Should only be provided for devices whose read operation cannot be represented as a simple read operation on a bus such as SPI, I2C, etc. Most of devices do not need this.
 * @reg_write: Same as above for writing.
 * @fast_io: Register IO is fast. Use a spinlock instead of a mutex to perform locking. This field is ignored if custom lock/unlock functions are used. This field is a duplicate of a similar file in 'struct regmap_bus' and serves exact same purpose. Use it only for "no-bus" cases.
 * @max_register: Optional, specifies the maximum valid register index.
 * @wr_table: Optional, points to a struct regmap_access_table specifying valid ranges for write access.
 * @rd_table: As above, for read access.
 * @volatile_table: As above, for volatile registers.
 * @precious_table: As above, for precious registers.
 * @reg_defaults: Power on reset values for registers (for use with register cache support).
 * @num_reg_defaults: Number of elements in reg_defaults.
 * @read_flag_mask: Mask to be set in the top byte of the register when doing a read.
 * @write_flag_mask: Mask to be set in the top byte of the register when doing a write. If both read_flag_mask and write_flag_mask are empty the regmap_bus default masks are used.
 * @use_single_rw: If set, converts the bulk read and write operations into a series of single read and write operations. This is useful for device that does not support bulk read and write.
 * @can_multi_write: If set, the device supports the multi write mode of bulk write operations, if clear multi write requests will be split into individual write operations.
 * @cache_type: the actual cache type.
 * @reg_defaults_raw: Power on reset values for registers (for use register cache support).
 * @num_reg_defaults_raw: Number of elements in reg_defaults_raw.
 * @reg_format_endian: Endianness for formatted register address. If this is DEFAULT, the @reg_format_endian_default value from the regmap bus is used.
 * @val_fromat_endian: Endianness for formatted register value. ...
 * @ranges: Array of configuration entries for virtual address ranges.
 * @num_ranges: Number of range configuration entries.
 */
struct regmap_config {
    const char *name;
    int reg_bits;
    int reg_stride;
    int pad_bits;
    int val_bits;

    bool (*writeable_reg)(struct device *dev, unsigned int reg);
    bool (*readalbe_reg)(struct device *dev, unsigned int reg);
    bool (*volatile_reg)(struct device *dev, unsigned int reg);
    bool (*precious_reg)(struct device *dev, unsigned int reg);
    regmap_lock lock;
    regmap_unlock unlock;
    void *lock_arg;

    int (*reg_read)(void *context, unsigned int reg, unsigned int *val);
    int (*reg_write)(void *context, unsigned int reg, unsigned int val);

    bool fast_io;

    unsigned int max_register;
    const struct regmap_access_table *wr_table;
    const struct regmap_access_table *rd_table;
    const struct regmap_access_table *volatile_table;
    const struct regmap_access_table *precious_table;
    const struct reg_default *reg_default;
    unsigned int num_reg_default;
    enum regcache_type cache_type;
    const void *reg_default_raw;
    unsigned int num_reg_defaults_raw;

    u8 read_flag_mask;
    u8 write_flag_mask;

    bool use_signle_rw;
    bool can_multi_write;

    enum regmap_endian reg_format_endian;
    enum regmap_endian val_format_endian;

    const struct regmap_range_cfg *ranges;
    unsigned int num_ranges;
};

// Regmap 的申请与初始化
// 对 regmap_init() 函数的进一步封装
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
```
- 驱动需要先分配一个具体的 `regmap` 结构体，并通过 `regmap_config` 来完成初始化；
  - 一般在 `probe` 函数中初始化 `regmap_config`，然后申请并初始化 regmap；
  - 在退出驱动时释放申请到的 `regmap`；
- 对于 `regmap`，`read_flag_mask` 和 `write_flag_mask` 是寄存器地址相关的掩码，注意检查 spi 和 i2c 对于这两个掩码的默认设置是否符合要求；
- `regmap_config`：
  - `precious_reg`：当寄存器值不能被读出来的时候此回调函数会被调用，比如很多中断状态寄存器读清零，读这些寄存器就可以清除中断标志位，但是并没有读出这些寄存器内部的值。
  - `reg_defaults`：为 `reg_default` 结构体类型，包含两个成员变量：reg 和 def，reg 为寄存器地址，def 是默认值。

```c
// 寄存器访问，统一的访问接口

// 返回值：0-成功；其他值-失败
int regmap_read(struct regmap *map, unsigned int reg, unsigned int *val);
int regmap_write(struct regmap *map, unsigned int reg, unsigned int val);
/* 
 * 修改寄存器指定的 bit 
 * mask: 需要更新的位必须在掩码中设置为 1
 * 返回值：0-成功；其他值-失败
 */
int regmap_update_bits(struct regmap *map, unsigned int reg, unsigned int mask, unsigned int val);
/* 
 * 读取多个寄存器的值 
 * val_count: 要读取的寄存器数量
 * 返回值：0-成功；其他值-失败
 */
int regmap_bulk_read(struct regmap *map, unsigned int reg, void *val, size_t val_count);
// 同上
int regmap_bulk_write(struct regmap *map, unsigned int reg, const void *val, size_t val_count);
```

### 源码阅读
```c
// regmap-spi.c
static struct regmap_bus regmap_spi = {
    .write = regmap_spi_write,
    ...
    .read_flag_mask = 0x80,
    .reg_format_endian_default = REGMAP_ENDIAN_BIG,
    .val_format_endian_default = REGMAP_ENDIAN_BIG,
};

struct regmap *regmap_init_spi(struct spi_device *spi, const struct regmap_config *config) {
    return regmap_init(&spi->dev, &regmap_spi, &spi->dev, config);
}

// 在 regmap_init 有如下代码：
if (config->read_flag_mask || config->write_flag_mask) {
    map->read_flag_mask = config->read_flag_mask;
    map->write_flag_mask = config->write_flag_mask;
} else if (bus) {
    map->read_flag_mask = bus->read_flag_mask;
}
```
- 当 `regmap_config` 中没有设置 `read_flag_mask` 和 `write_flag_mask` 时，使用 `regmap_spi (regmap_bus)` 上设置的 `read_flag_mask (0x80)`。因此，当使用的 spi 设备不需要读掩码时，在初始化 `regmap_config` 的时候将 `read_flag_mask` 显式设置为 0x00。

### 示例代码
当使用 spi 读取 icm20608 寄存器的时候，地址最高位必须设置为 1；写内部寄存器的时候，地址最高位要设置为 0。

在使用 regmap 的时候不需要手动将寄存器地址的 bit7 置 1，在初始化 `regmap_config` 的时候直接将 `read_flag_mask` 设置为 0x80 即可，这样通过 regmap 读取 spi 内部寄存器的时候就会将寄存器地址与 `read_flag_mask` 进行或运算，结果就是将 bit7 置 1。它们由 regmap 框架来完成。
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