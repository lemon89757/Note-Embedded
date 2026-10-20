# Note for Time Management
> 《深入 Linux 内核架构》-第十五章 时间管理

## 时钟设备初始化
1. 入口函数
    ```c
    // version: 4.1.15
    // arch/arm/kernel/time.c
    void __init time_init(void) {
        if (machine_desc->init_time) {
            machine_desc->init_time();
        } else {
    #ifdef CONFIG_COMMON_CLK;
            of_clk_init(NULL);
    #endif
            clocksource_of_init();
        }
    }
    ```
    - 调用时机：系统启动初始化时
        ```c
        start_kernel()
            time_init()
        ```
    - 背景：*i.mx6ull* 的设备描述（*machine_desc*）中并没有设置 `init_time()` 回调，且配置了选项 `CONFIG_COMMON_CLK`。另外，关于 *i.mx6ull* 的时钟设备的设备树描述信息如下：
        ```dts
        // arch/arm/boot/dts/imx6ul.dtsi
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

            ipp_di0: clock@2 {
                compatible = "fixed-clock";
                reg = <2>;
                #clock-cells = <0>;
                clock-frequency = <0>;
                clock-output-names = "ipp_di0";
            };

            ipp_di1: clock@3 {
                compatible = "fixed-clock";
                reg = <3>;
                #clock-cells = <0>;
                clock-frequency = <0>;
                clock-output-names = "ipp_di1";
            };
        };
        ```

2. `of_clk_init()` 
    ```c
    // version: 4.1.15
    // drivers/clk/clk.c

    /**
     * of_clk_init() - Scan and init clock provides from the DT
     * @matches: array of compatible values and init functions for providers.
     *
     * This function scans the device tree for matching clock providers
     * and calls their initialization functions. It also does it by trying
     * to follow the dependencies.
     */
    void __init of_clk_init(const struct of_device_id *matches) {
        const struct of_device_id *match;
        struct device_node *np;
        struct clock_provider *clk_provider, *next;
        bool is_init_done;
        bool force = false;

        if (!matches)
            matches = &__clk_of_table;
        
        /* First prepare the list of the clocks providers */
        for_each_matching_node_and_match(np, matches, &match) {
            struct clock_provider *parent = 
                kzalloc(sizeof(struct clock_provider), GFP_KERNEL);
            parent->clk_init_cb = match->data;
            parent->np = np;
            list_add_tail(&parent->node, &clk_provider_list);
        }

        while (!list_empty(&clk_provider_list)) {
            is_init_done = false;
            list_for_each_entry_safe(clk_provider, next, &clk_provider_list, node) {
                if (force || parent_ready(clk_provider->np)) {
                    clk_provider->clk_init_cb(clk_provider->np);
                    of_clk_set_defaults(clk_provider->np, true);

                    list_del(&clk_provider->node);
                    kfree(clk_provider);
                    is_init_done = true;
                }
            }

            /* 
             * We didn't manage to initialize any of the remaining providers during
             * last loop, so now we initialize all the remaining ones unconditionally
             * in case the clock parent was not mandatory (强制的)
             */
            if (!is_init_done)
                force = true;
        }
    }
    ```
    - 在设置树中寻找与 `__clk_of_table` 数组中兼容性匹配的设备结点，匹配的结点使用 `np` 记录，匹配信息通过 `match` 传出。识别到匹配结点后，分配 `struct clock_provider` 实例的内存、设置其设备树结点、初始化回调等字段信息，并将该实例连接到由全局变量 `clk_provider_list` 管理的链表上；
    - `parent_ready(), drivers/clk/clk.c`: *This function looks for a parent clock. If there is one, then it checks that the provider for this parent clock was initialized, in this case the parent clock will be ready.*
    - 执行 `clk_provider->clk_init_cb()` 进行时钟设备的初始化
    - 对 `__clk_of_table` 的分析：
      - `__clk_of_table` 符号由链接脚本文件（`arch/arm/kernel/vmlinux.lds`）定义：
        ```script
        .init.data : {
        *(.init.data) *(.meminit.data) *(.init.rodata) *(.meminit.rodata) . = ALIGN(8); 
        __clk_of_table = .; *(__clk_of_table) *(__clk_of_table_end) . = ALIGN(8); 
        ...
        }
        ```
      - 但是从内核源码中查找并找不到直接与 *__clk_of_table* 段相关的内容，而位于 *__clk_of_table_end* 段的变量只找到 `__clk_of_table_sentinel, drivers/clk/clk.c`，很容易判断它并不是与开发板设备树结点进行匹配的兼容性描述
      - 进一步通过内核映像对应的映射文件（`System.map`）查找到：
        ```map
        80bbe330 T __clk_of_table
        80bbe330 t __of_table_imx53_ccm
        80bbe3f4 t __of_table_imx51_ccm
        80bbe4b8 t __of_table_imx50_ccm
        80bbe57c t __of_table_imx6q
        80bbe640 t __of_table_imx6sl
        80bbe704 t __of_table_imx6sll
        80bbe7c8 t __of_table_imx6sx
        80bbe88c t __of_table_imx6ul
        80bbe950 t __of_table_imx7d
        80bbea14 t __of_table_vf610
        80bbead8 t __of_table_fixed_factor_clk
        80bbeb9c t __of_table_fixed_clk
        80bbec60 t __of_table_gpio_gate_clk
        80bbed24 t __clk_of_table_sentinel
        ```
        - 根据映射文件可知由 `__clk_of_table` 表示的数组包括 `__of_table_imx53_ccm` 等等
      - 进一步定位到 `__clk_of_table` 数组中的内容：
        ```c
        // version: 4.1.15

        // arch/arm/mach-imx/clk-imx51-imx53.c
        CLK_OF_DECLARE(imx50_ccm, "fsl,imx50-ccm", mx50_clocks_init);
        ...
        CLK_OF_DECLARE(imx51_ccm, "fsl,imx51-ccm", mx51_clocks_init);
        ...
        CLK_OF_DECLARE(imx53_ccm, "fsl,imx53-ccm", mx53_clocks_init);

        ...
        // arch/arm/mach-imx/clk-imx6ul.c
        CLK_OF_DECLARE(imx6ul, "fsl,imx6ul-ccm", imx6ul_clocks_init);

        ...
        // drivers/clk/clk-fixed-rate.c
        CLK_OF_DECLARE(fixed_clk, "fixed-clock", of_fixed_clk_setup);
        ```
        - `CLK_OF_DECLARE(imx53_ccm, "fsl,imx53-ccm", mx53_clocks_init)` 表示兼容性描述为 *"fsl,imx53-ccm"*，传入 `struct of_device_id` 中 `data` 字段的数据是一个函数指针，指向 `mx53_clocks_init()` 函数;
        - 根据 *i.mx6ull* 的设备树信息，`of_fixed_clk_setup()` 将会是完成匹配工作后调用的函数

3. `of_fixed_clk_setup()`
    ```c
    // version: 4.1.15
    // drivers/clk/clk-fixed-rate.c
    /* 
     * of_fixed_clk_setup() - Setup function for simple fixed rate clock 
     */
    void of_fixed_clk_setup(struct device_node *node) {
        struct clk *clk;
        struct char *clk_name = node->name;
        u32 rate;
        u32 accuracy = 0;

        if (of_property_read_u32(node, "clock-frequency", &rate))
            return;
        
        of_property_read_u32(node, "clock-accuracy", &accuracy);

        of_property_read_string(node, "clock-output-names", &clk_name);

        clk = clk_register_fixed_rate_with_accuracy(NULL, clk_name, NULL,
            CLK_IS_ROOT, rate, accuracy);
        
        if (!IS_ERR(clk))
            of_clk_add_provider(node, of_clk_src_simple_get, clk);
    }
    EXPORT_SYMBOL_GPL(of_fixed_clk_setup);
    ```