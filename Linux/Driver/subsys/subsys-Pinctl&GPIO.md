# PinCtrl & GPIO 子系统
关于 `IO Ports` 的使用往往涉及：
1. 复用模式选择（如将 `IO` 复用为 `GPIO`、 `UART` 的 `Tx port` 等）；
2. 电气属性配置（如 `IO` 的上下拉属性、驱动能力等）；
3. 中断配置（触发方式等）；

以 `IMX6ull` 为例，其 `IO Ports` 的电路框图如下：
// todo: 电路框图

不同的电路板的 `IO Ports` 数量、复用能力以及配置方式等不一致...

## 实现功能
`PinCtrl` 子系统通过软件分层，抽象出对 `IO Ports` 操作的公共部分并提供相应的接口。`BSP` 开发者在具体硬件的基础上实现这部分接口，驱动开发者通过调用接口，实现对硬件的操作。从而后者可以不必了解具体的 `IO Ports` 操作是如何完成的，实现了对驱动的 `IO Ports` 操作和具体硬件的解耦。即软件中的：用户不应该依赖于具体的实例，而是通过该类实例的抽象接口实现依赖反转。

## 用法与步骤

## Kernel-Documentation

## 代码框架

## 内核源码
1. 内核版本：

## references
1. [蜗窝科技](http://www.wowotech.net/sort/gpio_subsystem)