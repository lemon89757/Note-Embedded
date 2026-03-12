# platform 设备驱动

## 设备树中节点转化为内核中的 `platform_device`
1. 会转化为 `platform_device`：
   - 根节点下含有 `compatible` 属性的子节点；
   - 含有特定 `compatible` 属性的**节点的子节点**：
     - 如果一个节点的 `compatible` 属性值为这 4 者之一：`"simple-bus", "simple-mfd", "isa", "arm,amba-bus"`；
     - 含有上述 `compabile` 属性值的节点的子节点、且子节点含有 `compatible` 属性可以转化；
2. 不会转化为 `platform_device`：
   - 总线 `I2C`、`SPI` 节点下的子节点不转化为 `plaform_device`（某个总线下的子节点，应该交给对应的总线驱动程序来处理，它们不应该被转化为 `platform_device`）；

## API
1. `module_platform_driver()`

## 参考
1. 韦东山-《嵌入式 Linux 应用开发完全手册 V5.2_IMX6ULL_Pro 开发板》