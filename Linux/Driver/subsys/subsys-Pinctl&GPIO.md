# PinCtrl & GPIO 子系统
Linux 驱动讲究驱动分离与分层。

## PinCtrl 子系统
关于 `IO Ports` 的使用往往涉及：
1. 复用模式选择（如将 `IO` 复用为 `GPIO`、 `UART` 的 `Tx port` 等）；
2. 电气属性配置（如 `IO` 的上下拉属性、驱动能力等）；
3. 中断配置（触发方式等）；

以 `IMX6ull` 为例，其 `IO Ports` 的电路框图如下：
// todo: 电路框图

不同的电路板的 `IO Ports` 数量、复用能力以及配置方式等不一致...

### 主要工作内容
- 获取设备树中的 pin 信息；
- 根据上述信息设置 pin 的复用功能、电气特性等；

### 参考
- 源码：`drivers/pinctrl`；
- 设备树格式：`Documentation/devicetree/bindings/pinctrl/fsl,imx-pinctrl.txt`；
参考需要根据具体芯片厂商提供的参考说明，例如上述设备树格式文件使用的是飞思卡尔的 pinctrl 子系统格式；

### 实现功能
`PinCtrl` 子系统通过软件分层，抽象出对 `IO Ports` 操作的公共部分并提供相应的接口。`BSP` 开发者在具体硬件的基础上实现这部分接口，驱动开发者通过调用接口，实现对硬件的操作。从而后者可以不必了解具体的 `IO Ports` 操作是如何完成的，实现了对驱动的 `IO Ports` 操作和具体硬件的解耦。即软件中的：用户不应该依赖于具体的实例，而是通过该类实例的抽象接口实现依赖反转。

## GPIO 子系统
### 主要工作内容
- 用于初始化 gpio 并且提供相应的 API 函数（如设置 gpio 为输入/输出，读取 gpio 值等）；
- Linux 内核向驱动程序开发者屏蔽掉了 gpio 的设置过程，方便了驱动开发者使用 gpio(这里前者应该是指 BSP 部分，后者才是驱动开发部分)；

### 参考
- 源码：`drivers/gpio/gpio_mxc.c`；
- 设备树格式：`Documentation/devicetree/bindings/gpio/fsl-imx-gpio.txt`；
参考需要根据具体芯片厂商提供的参考说明，例如上述设备树格式文件使用的是飞思卡尔的 pinctrl 子系统格式；

### API 函数
1. 在使用一个 gpio 之前一定要先通过 `gpio_request` 函数进行申请：
    ```c
    /* 
     * gpio: 要申请的 gpio 标号，使用 of_get_named_gpio() 函数从设备树中获取指定 gpio 属性信息，此函数会返回这个 gpio 的标号； 
     * label: 给 gpio 设置的名字；
     * 返回值：0-成功，其他值-失败；
     */
    int gpio_request(unsigned gpio, const char *label);
    ```
2. 如果不使用某个 gpio 了，则可以通过 `gpio_free` 函数进行释放：
    ```c
    void gpio_free(unsigned gpio);
    ```
3. 设置某个 gpio 为输入：
    ```c
    // 返回值：0-成功，负值-失败
    int gpio_direction_input(unsigned gpio);
    ```
4. 设置某个 gpio 为输出:
   ```c
   // 返回值：0-成功，负值-失败
   int gpio_direction_output(unsigned gpio);
   ```
5. 读写操作：
    ```c
    // 通过宏获取某个 gpio 的值（0/1）
    #define gpio_get_value __gpio_get_value
    // 非负值-得到的 gpio 值，负值-获取失败
    int __gpio_get_value(unsigned gpio);

    // 通过宏设置某个 gpio 的值
    #define gpio_set_value __gpio_set_value
    void __gpio_set_value(unsigned gpio, int value);
    ```
---
与 gpio 相关的 OF 函数：
1. 获取设备树某个属性里面定义了几个 gpio 信息：
   ```c
   // propname: 要统计的 gpio 属性；
   // 返回值：正值-统计到的 gpio 数量；负值-失败
   int of_gpio_named_count(struct device_node *np, const char *propname);
   ```
   空的 gpio 信息也会被统计到，例如：
   ```dts
   gpios = <0
            &gpio1 1 2
            0
            &gpio2 3 4>
   ```
   上述代码 `gpios` 节点一共定义了 4 个 gpio，但有两个是空的，没有实际含义，通过上述函数统计出来的结果是 4。
2. 与上述函数类型，`of_gpio_count` 函数用于统计 “gpios” 这个属性的 gpio 数量，而 `of_gpio_named_count` 函数可以统计任意属性的 gpio 信息：
    ```c
    // 返回值：正值，统计到的 gpio 数量；负值-失败
    int of_gpio_count(struct device_node *np);
    ```
3. 获取 gpio 编号。因为 Linux 内核中关于 gpio 的 API 函数都要使用 gpio 编号，此函数会将设备树中类似 `<&gpio5 7 GPIO_ACTIVATE_LOW>` 的属性信息转换为对应的 gpio 编号。此函数在驱动中使用很频繁。
- [ ] 如何理解 gpio 编号
    ```c
    /* index: gpio 索引，一个属性里可能包含多个 gpio，此参数指定要获取哪个 gpio，如果只有一个 gpio 信息的话此参数为 0 
     * 返回值：正值-获取到的 gpio 编号；负值，失败；
     */
    int of_get_named_gpio(struct device_node *np, const char *propname, int index);
    ```

## 用法与步骤
1. 需要注意 pin 是否被其他外设使用

## 代码框架

## 内核源码
1. 内核版本：

## references
1. [蜗窝科技](http://www.wowotech.net/sort/gpio_subsystem)