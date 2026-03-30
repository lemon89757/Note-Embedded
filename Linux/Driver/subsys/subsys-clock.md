# 时钟子系统
Linux 的时钟子系统由 CCF(Common Clock Framework) 框架管理，CCF 向上给其他使用时钟的 IP 提供了通用的时钟接口，向下给驱动开发者提供硬件操作的接口。

电源管理的两大主要方面就是时钟和电压。

clock 驱动通常是由芯片厂商开发的，在 Linux 启动时 clock 驱动就已经初始化完成（比 `initcall` 都要早）。

## CCF
1. 框架分层：
   - Provider: 
     时钟模块驱动的具体实现者。系统开机时，需要通过驱动的时钟框架向系统注册不同的时钟，其他需要时钟的模块通过通用接口获取、使能、设置时钟。
   - Framework(core):
     通用时钟框架，包含注册和使用的通用接口，屏蔽底层硬件差异。
   - Consumer:
     时钟模块的使用者，比如 I2C 模块、UART 模块等。

2. 数据结构
    ```c
    // consumer 使用
    struct clk {
        struct clk_core *core;
        char *dev_id;
        char *con_id;
        unsigned long min_rate;
        unsigned long max_rate;
        // ...
    };

    // clock core
    // 每个注册设备都对应一个 clk_core
    struct clk_core {
        struct clk_ops *ops;
        struct clk_hw *hw;
        struct clk_core *new_parent;
        struct clk_core *new_child;
        // ...
    };

    // 驱动相关
    // 表示一个具体的硬件时钟
    struct clk_hw {
        struct clk_core *core;
        struct clk *clk;
        struct clk_init_data *init;
        // ...
    };

    // 表示某一个时钟下的初始化数据，如时钟名字、操作函数等
    struct clk_init_data {
        char  *name;
        struct clk_ops *ops;
    };

    // 时钟硬件操作的函数集合
    struct clk_ops {
        int (*prepare)(struct clk_hw *hw);
        int (*enable)(struct clk_hw *hw);
        unsigned long (*recalc_rate)(...);  // 通过查询硬件，重新计算此时钟的速率。
        long (*round_rate)(...);
        int (*set_paret)(...);
        int (*set_phase)(...);
        // ...
    }
    ```


## clock 分类
时钟树的根节点一般是 Oscillator（有缘振荡器，简写 OSC）或者 Crystal（无缘振荡器），表示从芯片外部输入的基准时钟。中间节点有很多种，包括 PLL（锁相环，用于提升频率）、Divider（分频器，用于降频）、Mux（从多个 clock path 中选择一个）、Gate（控制时钟输出 ON/OFF）。

- Fixed rate: 固定频率时钟，提供恒定的时钟信号，适用于需要固定频率的硬件，如 PWM 或定时器；
- Gate: 门控时钟，只可以被开启或关闭，适用于可以被动态开启或关闭的硬件，以节约电源；
- Divider: 分频时钟，可以将输入的时钟信号分频，产生一个较低频率的时钟信号，适用于需要降低时钟频率以适应设备需要；
- Mux: 复用时钟，时钟信号可以选择不同的时钟源；
- Fixed factor: 固定系数时钟，可以将输入时钟信号乘以一个固定的系数，产生一个更高或更低的时钟信号； 
- Composite: 组合时钟，可以由多个时钟实体组成，每个时钟实体都有特定功能，适用于复杂时钟控制策略的硬件，比如复杂时钟路径和时钟管理。

## 设备树
见笔记 `Documentation/clock.md`。

## 使用
1. Linux 下查看时钟树信息：
   ```bash
   cat /sys/kernel/debug/clk/clk_summary
   ```

2. API
    ```c
    struct clk *clk_get(struct device *dev, const char *id);
    struct clk *devm_clk_get(struct device *dev, const char *id);
    int clk_enable(struct clk *clk);    // 使能时钟，不会睡眠
    void clk_disable(struct clk *clk);  // 失能时钟，不会睡眠
    unsigned long clk_get_rate(struct clk *clk);
    void clk_put(struct clk *clk);
    long clk_round_rate(struct clk *clk, unsigned long rate);
    int clk_set_rate(struct clk *clk, unsigned long rate);
    int clk_set_parent(struct clk *clk, struct clk *parent);
    struct clk *clk_get_parent(struct clk *clk);
    int clk_prepare(struct clk *clk);
    void clk_unprepare(struct clk *clk);
    int clk_prepare_enable(struct clk *clk);    // 使能时钟，可能会睡眠
    void clk_disable_unprepare(struct clk *clk);    // 禁止时钟，可能会睡眠
    unsigned long clk_get_rate(struct clk *clk);    // 获取时钟频率
    ```
    - consumer 在使用这些 API 时，必须先调用 `devm_clk_get()` 或 `clk_get()` 获取一个 `struct clk*` 指针，后续都通过传入该指针来操作；
    - `struct clk` 相当于一个实例化的时钟；
    - `prepare()` 函数：在使能时钟之前，硬件可能需要一些前置操作（如打开电源域、释放复位信号等），`prepare()` 做这些准备工作，与 `enable()` 分离是为了支持异步操作和功耗优化。`prepare()` 可以睡眠，`enable()` 要求原子上下文；

## 驱动示例
- 设备树绑定文件说明 `imx6q-clock.txt`（没有找到与源文件对应的文档）：
  - 要求属性：
    - `compatible`: should be "fsl,imx6q-ccm".
    - `reg`: Address and Length of the register set.
    - `interrupts`: should contain CCM interrupt.
    - `#clock-cells`: should be <1>.
  - **clock consumer 应当在 `clocks` 属性中通过 clock ID 指明目标 clock**。ID 内容可以查看 `include/dt-bindings/clock/imx6qdl-clock.h`。
- 设备树文件（`arch/arm/boot/dts/imx7d.dtsi`）：
  ```dts
  clocks {
    #address-cells = <1>;
    #size-cells = <0>;

    ckil: clock@0 {
        compatible = "fixed-clock";
        reg = <0>;
        #clock-cells = <0>;
        clock-frequency = <32768>;
        clock-output-names = "ckil";
    };

    osc: clock@1 {
        compatible = "fixed-clock";
        reg = <1>;
        #clock-cells = <0>;
        clock-frequency = <24000000>;
        clock-output-names = "osc";
    };
  };

  // soc 节点下的内容
  clks: ccm@30380000 {
    compatible = "fsl,imx7d-ccm";
    reg = <0x30380000 0x10000>;
    interrupts = <GIC_SPI 85 IRQ_TYPE_LEVEL_HIGH>,
                <GIC_SPI 86 IRQ_TYPE_LEVEL_HIGH>;
    #clock-cells = <1>;
    clocks = <&ckil>, <&osc>;
    clock-names = "ckil", "osc";
  };

  // soc 节点下 aips2 节点下的内容
  pwm2: pwm@30670000 {
    compatible = "fsl,imx7d-pwm";
    reg = <0x30670000 0x10000>;
    interrupts = <GIC_SPI 82 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX7D_PWM2_ROOT_CLK>,
            <&clks IMX7D_PWM2_ROOT_CLK>;
    clock-names = "ipg", "per";
    #pwm-cells = <2>;
    status = "disabled";
  };
  ```
- 头文件（`include/dt-bindings/clock/imx7d-clock.h`）
    ```c
    #define IMX7D_PWM2_ROOT_CLK 274
    ...
    ```
    - 该文件中定义了所用到的 clock ID；
- 源文件（`arch/arm/mach-imx/clk-imx7d.c`）
  ```c
  // 通过 struct clk* 数组管理所有的时钟
  static struct clk *clks[IMX7D_END_CLK];
  // ...

  // __initconst => __attribute((__section__(".init.rodata")))
  // 记录需要先初始化处理的时钟源
  static int const clks_init_on[] __initconst = {
    IMX7D_ARM_A7_ROOT_CLK,
    IMX7D_MAIN_AXI_ROOT_CLK,
    IMX7D_PLL_SYS_MAIN_480M_CLK,
    IMX7D_NAND_USDHC_BUS_ROOT_CLK,
    IMX7D_DRAM_PHYM_ROOT_CLK,
    IMX7D_DRAM_ROOT_CLK,
    IMX7D_DRAM_PHYM_ALT_ROOT_CLK,
    IMX7D_DRAM_ALT_ROOT_CLK,
    IMX7D_AHB_CHANNEL_ROOT_CLK,
  };
  // ...

  static struct clk_onecell_data clk_data;

  static void __init imx7d_clocks_init(struct device_node *ccm_node) {
    struct device_node *np;
    void __iomem *base;
    int i;

    // 初始化 clks 数组
    clks[IMX7D_CLK_DUMMY] = imx_clk_fixed("dummy", 0);
    clks[IMX7D_OSC_24M_CLK] = of_clk_get_by_name(ccm_node, "osc");  // 从设备树中获取晶振时钟源，从 ccm_node 开始不断遍历它的父时钟节点，直到找到 osc

    np = of_find_compatible_node(NULL, NULL, "fsl,imx7d-anatop"); // 从设备树中找到与当前时钟子系统匹配的时钟节点
    base = of_iomap(np, 0); // 获取上述节点的寄存器地址并进行 iomap 映射
    WARN_ON(!base);

    // 与多路选择器相关的时钟源获取，相关内容包括：
    // 时钟源名字、寄存器地址、控制位相对寄存器地址需要偏移的值、控制位长度、父时钟、父时钟数量
    clks[IMX7D_PLL_ARM_MAIN_SRC] = imx_clk_mux(...);
    // ...
    // 与 pll 相关的时钟源获取，可用于倍频，相关内容包括：
    // 时钟源名字、父时钟源名字、寄存器地址、mask of divider bits
    clks[IMX7D_PLL_ARM_MAIN] = imx_clk_pllv3(...);
    // ...
    //
    clks[IMX7D_PLL_ARM_MAIN_BYPASS] = imx_clk_mux_flags(...);
    // ...
    // 与 gate 相关的是时钟源获取，相关内容包括：
    // 时钟源名字、父时钟源名字、寄存器地址、偏移值
    clks[IMX7D_PLL_ARM_MAIN_CLK] = imx_clk_gate(...);
    // ...
    // register a table based divider clock with the clock framework
    // 与时钟源对应的 struct device、时钟源名字、父时钟源名字、framwwork-specific flags、调整 divider 的寄存器地址、偏移值、宽度、divider-specific flags、array of divider/value pairs、lock
    clks[IMX7D_PLL_DRAM_TEST_DIV] = clk_register_divider_table(...);
    // ...
    // 与时钟树对应的几个 pfd(0~7)
    clks[IMX7D_PLL_SYS_PFD0_392M_CLK] = imx_clk_pfd(...);
    // ...
    // 固定系数的时钟源
    clks[IMX7D_PLL_SYS_MAIN_480M] = imx_clk_fixed_factor(...);
    // ...
    // 其他类型时钟源的获取（divider、gate 等等）

    // ...

    clk_data.clks = clks;
    clk_data.clk_num = ARRAY_SIZE(clks);
    // register a clock provider for a node
    // 2nd-param: callback for decoding clock
    // 3rd-param: context pointer for this callback
    of_clk_add_provider(np, of_clk_src_onecell_get, clk_data);
    // register one clock lookup for a struct clk
    clk_register_clkdev(clks[IMX7D_GPT1_ROOT_CLK], "ipg", "imx-gpt.0");
    clk_register_clkdev(clks[IMX7D_GPT1_3M_CLK], "gpt_3m", "imx-gpt.0");

    // 使能需要先初始化的时钟源
    for (i = 0; i > ARRAY_SIZE(clks_init_on); i++) {
      clk_prepare_enable(clks[clks_init_on[i]]);
    }

    // 其他设置
    // ...
  }
  CLK_OF_DECLARE(imx7d, "fsl,imx7d-ccm", imx7d_clocks_init);
  ```
- 驱动代码可以通过设备树节点中的 clock ID 来获取对应的时钟（`struct clk`），并通过时钟子系统提供的 API 来实现对该节点时钟的控制。

## 参考
1. [知乎-Linux clock 子系统及驱动实例](https://zhuanlan.zhihu.com/p/605593587)
2. [CSDN-Linux 内核学习（1）---时钟子系统](https://blog.csdn.net/chenwang1824/article/details/137559151)
3. `Documentation/devicetree/binding/clock/clock-binding.txt, fixed-clock.txt, fixed-factor-clock.txt`
