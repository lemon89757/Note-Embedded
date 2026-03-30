# 时钟子系统-设备树绑定
> `Documentation/devicetree/binding/clock/clock-binding.txt, fixed-clock.txt, fixed-factor-clock.txt`

在设备树中，任意节点都可以表示为时钟信号源，这些节点被称为 provider。时钟 consumer 使用 phandle 以及 clock 标识符对连接到 provider 的输出。与 gpio 标识符类似，clock 标识符是若干个 cells （可为 0 个）组成的队列，来识别输出到设备的 clock。clock 标识符长度通过 provider 节点的属性 `#clock-cells` 值来定义。

## clock provider
要求的属性：
- `#clock-cells`：clock 标识符的 cells 数量。典型值：0-provider 节点输出单个时钟信号、1-provider 节点输出多个时钟信号；

选项属性：
- `clock-output-name`：Recommended to be a list of strings of clock output signal names indexed by the first cell in the clock specifier. However, the meaning of clock-output-names is domain specific to the clock provider, and is only provided to encourage using the same meaning for the majority of clock providers. This format may not work for clock providers using a complex clock specifier format. In those cases it is recommended to omit this property and create a binding specific names property. **clock consumer nodes must never directly reference the provider's clock-output-names property**. Consumer nodes always reference clocks by index. The names should reflect the clock output signal names for the device.
- `clock-indices`：If the identifying number for the clocks in the node is not linear from zero, then this allows the mapping of identifiers into the clock-output-names array.
    ```dts
    oscillator {
        compatible = "myclocktype";
        #clock-cells = <1>;
        clock-indices = <1>, <3>;
        clock-output-names = "clka", "clkb";
    }
    ```
    This ensures we do not have any empty strings in clock-output-names.

## clock consumer
要求的属性：
- `clocks`：（phandle，clock 标识符）对的列表，一个对表示一个输入到此设备的时钟。当 provider 的 `#clock-cells` 值为 0 时，只用 phandle 部分来表示时钟源。

选项属性：
- `clock-names`：List of clock input name strings sorted in the same order as the clocks property. Consumers drivers will use clock-names to match clock input names with clock specifiers.
- `clock-ranges`：Empty property indicating that child nodes can inherit named clocks from this ndoe. Useful for bus nodes to provide a clock to their children.

## assigned clock parents and rates
一些 platforms 可能对默认父 clocks 以及时钟频率要求有初始配置，这样的配置信息可以通过在设备树节点中的 `assigned-clocks`、`assigned-clock-parents` 以及 `assigned-clock-rates` 属性来指定。
  - `assigned-clock-parents`：should contain a list of parent clocks in form of phandle and clock specifier pairs, the `assigned-clock-parents` property the list of assigned clock frequency values - corresponding to clocks listed in the `assigned-clocks` property.（`assigned-clock-parents` 是 `assigned-clocks` 中已经分配频率了的）
  - ~`assigned-clocks`：与上述 `assigned-clock-parents` 对应的时钟频率~；

示例：
```dts
uart@a000 {
    compatible = "fsl,imx-uart";
    reg = <0xa000 0x1000>;
    ...
    clocks = <&osc 0>, <&pll 1>;
    clock-names = "baud", "register";

    assigned-clocks = <&clkcon 0>, <&pll 2>;
    assigned-clock-parents = <&pll 2>;
    assigned-clock-rate = <0>, <460800>;
};
```
  - <&pll 2> 设备为 <&clkcon 0> 的父时钟；
  - <&pll 2> 被分配的频率是 460800 Hz；
  - 通过 consumer 设备节点来配置时钟的父节点以及频率只能执行一次，在多个 consumer 节点中指定冲突的父节点或者频率值是被禁止的；
  - 配置公共时钟，将会影响多个 consumer，这和直接在 provider 节点中进行设置类似。

## fixed-rate clock
要求属性：
- `compatible`: 应为 `"fixed-clock"`；
- `#clock-cells`：from common clock binding; shall be set to 0；
- `clock-frequency`：单个 cell 表示，单位为 Hz。

选项属性：
- `clock-accuracy`：accuracy of clock in ppb(parts per billion). should be a single cell.
- `clock-output-names`: From common clock binding.

示例：
```dts
clock {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <1000000000>;
    clock-accuracy = <100>;
};
```

## fixed-factor rate clock
要求属性：
- `compatible`：应为 `"fixed-factor-clock"`；
- `#clock-cells`：from common clock binding; shall be set to 0；
- `clock-div`：fixed divider；
- `clock-mut`：fixed multiplier；
- `clocks`：parent clock.

选项属性：
- `clock-output-names`：from common clock binding.

示例：
```dts
clock {
    compatible = "fixed-factor-clock";
    clocks = <&parentclk>;
    #clock-cells = <0>;
    clock-div = <2>;
    clock-mult = <1>;
};
```