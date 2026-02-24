# Note for DeviceTree
## 硬件描述方式
   - 在没有设备树的时候（内核版本 3.x ），Linux 内核源码中大量的 `arch/arm/mach-xxx` 和 `arch/arm/plat-xxx` 文件夹，这些文件夹里面的文件就是对应平台下的板级信息；（它们被硬编码到 `Linux` 内核中，导致内核臃肿）
   - `ARM` 社区引入了 `PowerPC` 等架构已经采用的设备树（`Flattened Device Tree`），将这些描述板级硬件信息的内容从 `Linux` 中分离开来，用一个专属的文件格式来描述。这个专属的文件就叫设备树，文件扩展名为 `.dts`；

## 设备树相关的文件扩展名
   - `.dts`：   设备树文件；
   - `.dtsi`：  类似于 C 语言中的头文件。一个 SOC 可以作出很多不同的板子，这些不同的板子肯定是有共同的信息，将这些共同的信息提取出来作为一个通用的文件，其他的 `.dts` 文件可以直接引用这个通用文件（`.dtsi`）；
   一般 `.dts` 描述板级信息（例如开发板上的 IIC 设备、SPI 设备等），`.dtsi` 描述 SOC 级信息（例如 SOC 有几个 CPU、主频是多少、各个外设控制器信息等）；
   此外，`.dts` 文件也可以直接引用 `.dts` 文件以及 C 语言的 `.h` 头文件，例如：
   ```dts
   #include <dt-bindings/input/input.h>
   #include "imx6ull.dtsi"
   ...
   #include "imx6ull-14x14-evk.dts"
   ```
   - `.dtb`：`.dts` 文件编译以后得到的二进制文件的扩展名；
   将 `.dts` 编译为 `.dtb` 需要用到 `dtc` 工具，它的源码在 Linux 内核的 `scripts/dtc` 目录下，在 `scripts/dtc/Makefile` 中有如下信息说明了 `dtc` 工具的依赖：
    ```Makefile
    hostprogs-y := dtc
    always := $(hostprogs-y)

    dtc-objs := dtc.o flattree.o fstree.o data.o livetree.o treesource.o srcpos.o checks.o util.o
    dtc-objs += dtc-lexer.lex.o dtc-parser.tab.o
    ```

## 设备树的编译
   根据 Linux 内核顶层 Makefile 文件，可以找到用于编译设备树的相关目标。
   ```bash
   make all # 编译 Linux 源码中的所有东西，包括 zImage，.ko 驱动模块以及设备树
   ```
   或者
   ```bash
   make dtbs # 只编译设备树
   ```
   在 `arch/arm/boot/dts/Makefile` 中可以找到需要编译的设备树源文件详情，例如：
   ```Makefile
   dtb-$(CONFIG_SOC_IMX6UL) += \
    imx6ul-14x14-ddr3-arm2.dtb \
    imx6ul-14x14-drr3-arm2-emmc.dtb
   ```
   由上述 Makefile 代码可知，当 `#define CONFIG_SOC_IMX6UL y` 后，即可编译其依赖的设备树文件。

## 设备树语法
参考资料：
    1. 《Devicetree Specification.pdf》
    2. 《Power_ePAPR_APPROVED_vxx.pdf》
    3. 内核文档 Documentation/bind，其中会有详细的规定以及各个厂商对于自己设备树的具体说明；

1. 每个设备树文件有且仅有一个根节点，以 `/` 表示，被 `#include` 的头文件中的根节点会与当前文件合并成一个；
2. 设备树中节点命名格式：
    ```dts
    label:node-name@unit-address
    ```
    - 可以通过 `&label` 来访问其对应的节点；
    - `label:` 可以省略；
3. 标准属性
   不同的设备需要的属性不同，用户可以自定义属性。除了用户自定义属性，有很多属性是标准属性。
   - `compatible`：用于匹配内核中的驱动程序。根节点的 `compatible` 属性可知所使用的设备（"硬件设备名字", "设备所使用的 SOC"："fsl,imx6ull-14x14-evk", "fsl,imx6ull"），内核会通过根节点的 `compatible` 属性查看是否支持此设备，如果支持的化就会启动内核；
   - `mode`：描述设备模块信息，例如名字；
   - `status`："okay"（设备可操作）、"disable"（设备当前不可操作，但未来可变为可操作的）、"fail"（设备不可操作，设备检测到了一系列错误）、"fail-sss"（同 "fail"，"sss" 部分是检测到的错误内容）；
   - `#address-cells` 和 `#size-cell`：前者决定了**子节点** `reg` 属性中地址信息所占用的**字长**（32 位）；后者决定了子节点 `reg` 属性中长度信息所占的**字长**（32 位）；
   - `reg`：`reg` 属性的值一般是 `(address, length)` 对，用于描述设备地址空间资源信息，一般都是某个外设寄存器地址范围信息；
   - `ranges`：`ranges` 属性值可以为空或者按照 `(child-bus-address, parent-bus-address, length)` 格式编写的数字矩阵，表示一个地址映射/转换表，由子地址、父地址和地址空间长度组成，例如：
    ```dts
    soc {
        compatible = "simple-bus";
        #address-cells = <1>;
        #size-cells = <1>;
        ranges = <0x0 0xe0000000 0x00100000>;

        serial {
            device_type = "serial";
            compatible = "ns16550";
            reg = <0x4600 0x100>;
            clock-frequency = <0>;
            interrupts = <0xA 0x8>;
            interrupt-parent = <&ipic>;
        };
    }
    ```
    上述设备树指定了一个 1024KB(0x00100000) 的地址范围，子地址空间的物理起始地址为 0x0，父地址空间的物理起始地址为 0xe0000000。子节点 `serial` 定义了设备寄存器的起始地址为 0x4600，寄存器长度为 0x100，经过地址转换，`serial` 设备可以从 0xe0004600 开始进行读写操作；
    - `name`：用于记录节点名字，已被弃用；
    - `device_type`：已被弃用；
4. 特殊节点
   - `aliases`，定义别名；
   - `chosen`，并不是一个真实的设备，主要是为了 `uboot` 向内核传递数据，重点是 `bootargs` 参数：
        ```dts
        chosen {
            stdout-path = &uart1;
        };
        ```
        - 进入到 `proc/device-tree/chosen` 目录中，会发现存在 `bootargs` 这个属性，通过 `cat` 命令查看内容可得到 `uboot` 启动内核时 `bootargs` 的环境变量；
        - `uboot` 在 `chosen` 节点中添加了 `bootargs` 属性，参考 `common/fdt_support.c` 中的 `fdt_chosen` 函数；
        - `bootz` 命令 -> `do_bootz()` -> `do_bootm_states()` -> `boot_selected_os()` -> `do_bootm_linux()` (`boot_fn() -> boot_prep_linux() -> image_setup_linux() -> fdt_chosen()`)；

## 关于 Linux 内核是否支持某设备的判断
1. 使用设备树前
   - `uboot` 会向内核传递一个叫做 `machine id` 的值，即设备 ID，告诉内核当前的设备信息，内核据此判断是否支持此设备；
   - 针对每个设备，Linux 内核都用 `MACHINE_START` 和 `MACHINE_END` 来定一个 `machine_desc` 结构体变量来描述这个设备（eg:`arch/arm/mach-imx/mach-mx35_3ds.c`）；
   - 这个 `machine_desc` 变量存储在 `.arch.info.init` 段中，变量中存储了此设备的 `machine id`；
   - `include/generated/mach-types.h` 中定义了大量的 `machine id`，`uboot` 传进来的 `machine id` 会于此文件中定义的相关宏进行对比，看有没有相等的，如果相等的话就表示内核支持该设备；
2. 使用设备树后
   - 不再使用 `MACHINE_START`，而是 `DT_MACHINE_START`（定义在 `arch/arm/include/asm/mach/arch.h`，它也对应到一个 `machine_desc` 结构体），不再根据 `machine id` 来检查内核是否支持该设备；
   - `machine_desc` 中有一个成员变量 `.dt_compat`，表示设备兼容性，它会与设备树中根节点的 `compatible` 属性进行对比，若有任何一个值相等，则表示内核支持该设备；
3. 匹配流程：
   - 内核调用 `start_kernel` 函数来启动内核；
   - `start_kernel` 调用 `setup_arch` 函数（位于 `arch/arm/kernel/setup.c`）来匹配 `machine_desc`；
   - `setup_arch` 继续调用 `setup_machine_fdt() -> of_flat_dt_match_machine()` 来实现匹配； 

## 设备树在 Linux 中文件目录的体现
Linux 内核启动的时候会解析设备树中各个节点的信息，并且在根文件系统中的 **`/proc/device-tree`** 目录下根据节点名字创建不同的文件夹。

- 内核解析 `dtb` 文件
  - 解析流程：`start_kernel -> setup_arch -> unflatten_device_tree -> __unflatten_device_tree -> unflatten_dt_node(解析 DTB 文件中的各个节点)`

## 向设备树中添加节点信息
- 参考：
  - Linux 源码目录：`/Documentation/devicetree/bindings`。例如在 I.MX6ULL 这颗 SOC 的 I2C 下添加一个节点，可以查看 `/Documentation/decicetree/bingdings/i2c/i2c-imx.txt`（由具体芯片厂商提供各自的绑定参考指南，目录下找不到也是需要联系芯片厂商提供）。

## 设备树操作相关函数和数据结构
- 节点描述结构体
  linux 内核使用 `device_node` 结构体来描述一个节点，位于 `include/linux/of.h` 文件中：
  ```c
  struct device_node {
    const char *name;
    const char *type;
    phandle phandle;
    const char *full_name;
    struct fwnode_handle fwnode;

    struct property *properties;
    struct property *deadprops; // removed 属性
    struct device_node *parent;
    struct device_node *child;
    struct device_node *sibling;
    struct kobject kobj;
    unsigned long _flags;
    void *data;
  #ifdef defined(CONFIG_SPARC)
    const char *path_component_name;
    unsigned int unique_id;
    struct of_irq_controller *irq_trans;
  #endif // defined(CONFIG_SPARC)
  };
  ```
- 属性描述结构体
  linux 内核使用 `property` 结构体来描述属性信息，位于 `include/linux/of.h` 文件中：
  ```c
  struct property {
    char *name;     // 属性名
    int length;     // 属性长度
    void *value;    // 属性值
    struct property *next;
    unsigned long _flags;
    unsigned int unique_id;
    struct bin_attribute attr;
  };
  ```
- 内存空间描述结构体
  IIC、SPI、GPIO 等这些外设都有相应的寄存器，这些寄存器其实就是一组内存空间，Linux 内核使用 resource 结构体来描述一段内存空间，定义在文件 `include/linux/ioport.h` 中：
  ```c
  struct resource {
    resource_size_t start;
    resource_size_t end;
    const char *name;
    unsigned long flags;
    struct resource *parent, *sibling, *child;
  };
  ```
  对于 32 位的 SOC 来说，`resource_size_t` 是 u32 类型的。其中，`flags` 表示资源类型，可选的资源标志定义在 `include/linux/ioport.h` 中：
  ```c
  #define IORESOURCE_BITS ...
  #define IORESOURCE_TYPE_BITS ...
  #define IORESOURCE_IO ...
  #define IORESOURCE_MEM ...
  #define IORESOURCE_REG ...
  #define IORESOURCE_IRQ ...
  #define IORESOURCE_DMA ...
  #define IORESOURCE_BUS ...
  ...
  ```

- 函数
Linux 内核提供了一系列的函数来获取设备树中的节点或属性信息，这一系列函数都有一个统一的前缀 `of_`，因此也常被称为 `OF` 函数，原型位于 `include/linux/of.h` 文件中。
  - 查找节点函数：
      ```c
      struct device_node* of_find_node_by_name(struct device_node *from, const char *name)`;
      struct device_node* of_find_node_by_type(struct device_node *from, const char *type);
      /* 
       * type 可以为 NULL，表示忽略掉 device_type 属性 
       * compatible 表示要查找的节点所对应的 compatible 属性列表
       */
      struct device_node* of_find_compatible_node(struct device_node *from, const char *type, const char *compatible);
      /* 
       * matches: of_device_id 匹配表，以检查兼容性
       * match: 传出参数，找到的匹配的 of_device_id 
       */
      struct device_node* of_find_matching_node_and_match(struct device_node *from, const struct of_device_id *matches, const struct of_device_id **match);
      /* path: 带有全路径的节点名，可以使用节点的别名，比如 "/backlight" 就是 backlight 这个节点的全路径 */
      inline struct device_node* of_find_node_by_path(const char *path);
      ```
      其中：
      - `from` 表示开始查找的节点，当为 `NULL` 时表示从根节点开始查找；
      - 返回值为 `NULL` 时表示查找失败；
  - 查找父/子节点
      ```c
      struct device_node* of_get_parent(const struct device_node *node);
      /* prev 为 NULL 时表示从第一个节点开始 */
      struct device_node* of_get_next_child(const struct device_node *node, struct device_node *prev);
      ```
  - 提取属性值
      ```c
      /* 
       * np: 设备节点；
       * name: 属性名字；
       * lenp: 属性值的字节数 
       */
      struct property* of_find_property(const struct device_node *np, const char *name, int lenp);
      /* 获取属性中元素的数量，比如 reg 属性值是一个数组，那么使用此函数可以获取到这个数组的大小 */
      int of_property_count_elems_of_size(const struct device_node *np, const char *propname, int elem_size);
      /* 
       * 从属性中获取指定标号的 u32 类型数据值 
       * out_value: 传出值，表示读取到的值
       * 返回值：0-读取成功，负值-读取失败，-EINVAL 表示属性不存在，-ENODATA 表示没有要读取的数据，-EOVERFLOW 表示超出索引
       */
      int of_property_read_u32_index(const struct device_node *np, const char *propname, u32 index, u32 *out_value);
      /* 读取数组数据，比如多数的 reg 属性都是数组数据，可以使用函数一次性地读出 reg 属性中地所有数据 */
      int of_property_read_u8_array(const struct device_node *np, const char *propname, u8 *out_values, size_t sz);
      int of_property_read_u16_array(const struct device_node *np, const char *propname, u16 *out_values, size_t sz);
      int of_property_read_u32_array(const struct device_node *np, const char *propname, u32 *out_values, size_t sz);
      int of_property_read_u64_array(const struct device_node *np, const char *propname, u64 *out_values, size_t sz);
      /* 读取只有一个整形值的属性 */
      int of_property_read_u8(const struct device_node *np, const char *propname, u8 *out_value);
      int of_property_read_u16(const struct device_node *np, const char *propname, u16 *out_value);
      int of_property_read_u32(const struct device_node *np, const char *propname, u32 *out_value);
      int of_property_read_u64(const struct device_node *np, const char *propname, u64 *out_value);
      /* 读取属性中字符串值 */
      int of_property_read_string(struct device_node *np, const char *propname, const char **out_string);
      /* 获取 #address-cells 属性值 */
      int of_n_addr_cells(struct device_node *np);
      /* 获取 #size-cells 属性值 */
      int of_n_size_cells(struct device_node *np);
      ```
  - 其他
    ```c
    /* 
     * 检查设备节点的兼容性，查看节点的 compatible 属性是否包含 compat 指定的字符串
     * 返回值：0-不包含，正数-节点的 compatible 属性中包含 compat 指定的字符串 
    */
    int of_device_is_compatible(const struct device_node *device, const char *compat);
    /* 
     * 获取地址相关属性，主要是 reg 或者 assigned-address 属性值 
     * index: 要读取的地址标号；
     * size: 地址长度
     * flags: 参数，比如 IORESOURCE_IO, IORESOURCES_MEM 等
     * 返回值为地址数据的首地址，为 NULL 表示读取失败
     */
    const __be32 *of_get_address(struct device_node *dev, int index, u64 *size, unsigned int *flags);
    /* 
     * 将从设备树中读取到的地址转换为物理地址
     * in_addr: 要转换的地址 
     * 返回值：得到的物理地址，如果为 OF_BAD_ADDR 的话表示转换失败
     */
     u64 of_translate_address(struct device_node *dev, const __be32 *in_addr);
    /* 
     * 从设备树中提取资源值，本质上是将 reg 属性值转换为 resource 结构体类型 
     * 返回值：0-成功，负值-失败
     */
    int of_address_to_resource(struct device_node *dev, int index, struct resource *r);
    /* 直接内存映射，之前通过 ioremap 函数来完成物理地址到虚拟地址的映射，采用设备树后就可以直接通过 of_iomap 函数来获取内存地址所对应的虚拟地址。of_iomap 函数本质上也是将 reg 属性中的地址信息转换为虚拟地址。如果 reg 属性有多段的话，可以通过参数指定要完成内存映射的是哪一段 */
    /* 返回值：NULL 时表示内存映射失败 */
    void __iomem *of_iomap(struct device_node *np, int index);
    ```