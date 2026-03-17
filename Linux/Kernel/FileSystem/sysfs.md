# sysfs
## 使用
`/sys` 目录下有很多内核、驱动的信息。

1. 查看设备树信息
   ```bash
   ls /sys/firmware
   # 输出 devicetree fdt
   ```
   - `/sys/firmware/device` 目录下是以目录结构呈现的 `dtb` 文件，根节点对应 `base` 目录，每一个节点对应一个目录，每一个属性对应一个文件；
   - 设备正常加载在 `/sys/firmware/devicetree/base` 下有对应节点；
   - 属性查看：
     - 字符串类型：通过 `cat` 命令打印；
     - 数值类型：通过 `hexdump` 打印；
2. platform_device
   在 `/sys/devices/platform` 目录下存在目标 `platform_device`。
   - 一个设备对应一个目录，进入某个目录后，如果它有“driver”子目录，就表示这个 `platform_device` 跟某个 `platform_driver` 配对了；
   - 一个 `platform_device` 只能配对一个 `platform_driver`；
   - 查看驱动信息，可以找到驱动所在的位置：
    ```bash
    ls driver -ld
    ```
3. platform_driver
   在 `/sys/bus/platform/drivers` 目录下含有注册进内核的所有 `platform_driver`。
   - 一个 `driver` 对应一个目录，进入某个目录后，如果它有配对的设备，可以直接看到。
   - 一个 `platform_driver` 可以配对多个 `platform_device`；