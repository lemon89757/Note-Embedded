# video-interfaces
> `Documentation/devicetree/bindings/media/video-interfaces.txt`

**Common bindings for video receiver and transmitter interfaces**

## 基本概念
视频数据链路通常由多个**外部**设备组成，例如包括摄像头传感器、I2C/SPI/UART 总线控制器、SoC **内部** IP 以及视频 DMA engines 和视频数据处理器等。

SoC 内部模块通过设备树节点来描述，这部分在 SoC 之间是相似的。外部设备由它们所在总线控制器，比如 I2C，下的子节点来表示。

**所有视频设备的数据接口通过它们的 `port` 子节点来描述**。`port` 在设备树中的描述依赖于其他参与数据传输的设备，它通过 `endpoint` 子节点来描述。

```dts
device {
    ...
    ports {
        #address-cells = <1>;
        #size-cells = <0>;

        port@0 {
            ...
            endpoint@0 {...};
            endpoint@1 {...};
        };
        port@1 {...};
    };
};
```
如果一个 `port` 能够被配置为与多个 `remote device` 在同一总线上工作，每个 `remote device` 都应该由一个单独的 `endpoint` 子节点来描述。如果在一个设备节点中存在多个（大于等于 2）`port` 或者在一个 `port` 中有一个以上 `endpoint`，又或者 `port` 节点需要与一个可选择的硬件接口联系起来，一个共同的策略就是使用属性：`#address-cells`、`#size-cells`、`#reg`。

所有的 `port` 节点可以组合在 `ports` 节点下（可选），从而允许单独指定 `port` 节点以及 `endpoint` 节点的 `#address-cells`、`#size-cells` 属性，以及一个设备可能有的任何子设备节点。

**两个 `endpoint` 节点通过彼此的 `remote-endpoint` 句柄（phandle）连接。一个设备的 `endpoint` 子节点包含用于配置该设备与其他设备数据交换的所有所需属性**。大多数情况下，同级 `endpoint` 节点下属性是一样的，然而当在总线上的两个设备之间有任何信号修改的情况下，即在线上（on lines）有逻辑信号转换，也有可能导致属性的差异。

在设备支持的情况下，允许位于同一 `port` 下的多个 `endpoint` 同时激活。例如，一个设备数据接口被分到多个数据总线上，例如一个 16 位输入的 port 被分到单独的两个 ITU-R BT.656 8 位的总线上。在这种情况下，`bus-width` 以及 `data-shift` 属性可以被用来分配物理数据线到每个 `endpoint` 节点（逻辑总线）。

## 要求的属性
如果有超过一个 `port`、`endpoint` 或者 `reg` 属性出现在 `port` 以及/或者 `endpoint` 节点下，在其父节点下如下属性需要存在：
- `#address-cells`：number of cells required to define port/endpoint identifier, should be 1.
- `#size-cells`：should be 0.

## endpoint 的可选属性
- `remote-endpoint`：句柄（phandle），用以描述远程设备节点的 `endpoint` 子节点；
- `slave-mode`：布尔属性，表示链接是否工作在 slave 模式。当该属性没有指定时，默认是 master 模式。在 slave 模式下，master 设备（data sink）提供水平和垂直同步信号给从设备（data source）。在 master 模式下，数据源（data source）设备同样也是同步信号源设备；
- `bus-width`：并行总线（parallel busses）实际使用的有效的数据线（lines）总量；
- `data-shift`：在并行数据总线（parallel data busses）上，如果 `bus-width` 被用于指定数据线（data line）的数量，`data-shift` 可以被用于指定哪些数据线被使用。例如，"bus-width=<8>; data-shift=<2>;" 表示 9:2 的线被使用；
- `hsync-active`：HSYNC 信号激活的状态，0/1 分别表示 LOW/HIGH；
- `vsync-active`：VSYNC 信号激活的状态，0/1 分别表示 LOW/HIGH。
  - 注意，如果 HSYNC 和 VSYNC 信号的极性没有指定，embedded synchronization may be required, where supported；
- `data-active`：与 HSYNC 和 VSYNC 相似，指定数据线（data line）的极性；
- `field-even-active`：field signal level during the even field data transmission；
- `pclk-sample`：在像素时钟的上升沿（1）或者下降沿（0）进行数据采样；
- `sync-on-green-active`：Sync-on-green (SOG) 信号的激活状态，0/1 分别表示 LOW/HIGH；
- `data-lanes`：物理数据通道的索引数组，入口的位置（position of an entry）决定了逻辑通道的数字，但是入口的值表示的是物理通道。例如，对于两个通道的 MIPI CSI-2 总线来说，我们可以指定 "data-lanes=<1 2>;"，假设时钟通道是硬件通道 0。该属性只对串行总线（serial busses）（e.g. MIPI CSI-2）有效；
- `clock-lanes`：物理时钟通道的索引数组。入口的位置决定了逻辑通道的数字，但是入口的值表示的是物理通道。例如，对于一个 MIPI CSI-2 总线来说，我们可以指定 "clock-lanes=<0>;"，这表示时钟通道是在硬件通道 0 上。该属性只对串行总线有效（例如 MIPI CSI-2）。注意，对于 MIPI CSI-2 来说，该（属性的）数组只包含一个入口；
- `clock-noncontinuos`：布尔属性，（表示是否）允许 MIPI CSI-2（使用）非连续时钟模式；
- `link-frequencies`：被允许的数据总线频率。对于 MIPI CSI-2 来说，是表示实际的总线频率，并非一个时钟一个通道的位数量。用 64 位无符号整数的数组表示；
- `lane-polarities`：通道极性的数组，以时钟通道开始，后面依次是数据通道的极性，顺序和 `data-lanes` 一致。有效的值是 0（normal）和 1（inverted）。数组的长度应当是 `data-lanes` 和 `clock-lanes` 属性的长度和。如果忽略该属性，则相当于指定 0（normal）。该属性只对串行总线有效。

## 示例
如下代码块示例描述的是两个数据链路（two data pipelines），ov772x 和 imx074 为摄像头传感器，它们分别具有并行和串行（MIPI CSI-2）视频总线。两个传感器都是通过 I2C 控制总线连接，对应到 i2c0 控制器节点上。ov772x sensor 直接连接到 ceu0 视频 host 接口，imx074 通过 MIPI CSI-2 接收端（csi2）连接到 ceu0。ceu0 has a (single) DMA engine 将捕获的数据写入内存。ceu0 节点有一个 port 节点，这表示任何时候都只有一个数据链路，可以是 ov772x -> ceu0 或者 imx074 -> csi2 -> ceu0。
```dts
ceu0: ceu@0xfe910000 {
    compatible = "renesas,sh-mobile-ceu";
    reg = <0xfe910000 0xa0>;
    interrupts = <0x880>;

    mclk: master_clock {
        compatible = "renesas,ceu-clk";
        #clock-cells = <1>;
        clock-frequency = <50000000>;   /*max clock frequency*/
        clock-output-names = "mclk";
    };

    port {
        #address-cells = <1>;
        #size-cells = <0>;

        /*parallel bus endpoint*/
        ceu0_1: endpoint@1 {
            reg = <1>;                  /*local endpoint*/
            remote = <&ov772x_1_1>;     /*remote phandle*/
            bus-width = <8>;            /*used data lines*/
            data-shift = <2>;           /*lines 9:2 are used*/

            /*if hsync-active/vsync-active are missing, embedded BT.656 sync is used*/
            hsync-active = <0>;         /*Active low*/
            vsync-active = <0>;         /*Active low*/
            data-active = <1>;          /*Active high*/
            pclk-sample = <1>;          /*Rising*/
        };

        /*MIPI CSI-2 bus endpoint*/
        ceu0_0: endpoint@0 {
            reg = <0>;
            remote = <&csi2_2>;
        };
    };
};

i2c0: i2c@0xfff20000{
    ...
    ov772x_1: camera@0x21 {
        compatible = "ovti,ov772x";
        reg = <0x21>;
        vddio-supply = <&regulator1>;
        vddcore-supply = <&regulator2>;

        clock-frequency = <20000000>;
        clocks = <&mclk 0>;
        clock-names = "xclk";

        port {
            /*with 1 endpoint per port no need for addresss.*/
            ov772x_1_1: endpoint {
                bus-width = <8>;
                remote-endpoint = <&ceu0_1>;
                hsync-active = <1>;
                vsync-active = <0>;     /*who came up with an inverter here ?...*/
                data-active = <1>;
                pclk-sample = <1>;
            };
        };
    };

    imx074: camera@0x1a {
        compatible = "sony,imx074";
        reg = <0x1a>;
        vddio-supply = <&regulator1>;
        vddcore-supply = <&regulator2>;

        clock-frequency = <30000000>;   /*shared clock with ov772x_1*/
        clocks = <&mclk 0>;
        clock-names = "sysclk";         /*assuming this is the name in the datasheet*/
        port {
            imx074_1: endpoint {
                clock-lanes = <0>;
                data-lanes = <1 2>;
                remote-endpoint = <&csi2_1>;
            };
        };
    };
};

csi2: csi2@0xffc90000 {
    compatible = "renesas,sh-mobile-csi2";
    reg = <0xffc90000 0x1000>;
    interrupts = <0x17a0>;
    #address-cells = <1>;
    #size-cells = <0>;

    port@1 {
        compatible = "renesas,csi2c";   /*one of CSI2I and CSI2C*/
        reg = <1>;                      /*CSI-2 PHY #1 of 2: PHY_S, PHY_M has port address 0, is unused*/
        csi2_1: endpoint {
            clock-lanes = <0>;
            data-lanes = <2 1>;
            remote-endpoint = <&imx074_1>;
        };
    };
    port@2 {
        reg = <2>;                      /*port 2: link to the CEU*/
        csi2_2: endpoint {
            remote-endpoint = <&ceu0_0>;
        };
    };
};
```
