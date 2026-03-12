# Kernel Note
## Memory Allocation Guide
> [Memory Allocation Guide](https://www.kernel.org/doc/html/latest/core-api/memory-allocation.html#memory-allocation)

### GFP flags
`Get Free Page flags`

### GFP flags and reclaim behavior

### memory allocator

### Memory Management APIs
> [Memory Management APIs](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmalloc)

## log
### `printk`
- 查看通过 `printk` 打印的内核日志信息：
  - 通过 `dmesg` 命令
    ```bash
    dmesg                   # 基本用法，用于显示或控制内核环形缓冲区的内容，即内核日志
    dmesg | tail            # 显示最新的日志
    dmesg | grep my_module  # 显示与 my_module 相关的日志信息
    # 实时查看内核日志的变化
    watch "dmesg | tail -n 50"
    dmesg -w
    ```
  - 查看 `/var/log/syslog` 文件
    ```bash
    sudo cat /var/log/syslog        # 查看完整日志
    sudo tail -f /val/log/syslog    # 实时查看日志
    ```
  - 设置内核日志级别。内核日志级别决定了哪些信息会被输出到控制台，可以通过修改 `/proc/sys/kernel/printk` 文件或使用 `sysctl` 命令来设置日志级别。
    ```bash
    cat /proc/sys/kernel/printk         # 查看当前日志级别
    echo 7 > /proc/sys/kernel/printk    # 设置日志级别，7 表示所有级别的日志都会输出到控制台，覆盖
    # 使用 sysctl 命令
    sysctl kernel.printk=8      # =8 还是 =7?
    ```
    日志级别有：0-紧急，1-警报，2-严重，3-错误，4-警告，5-通知，6-信息，7-调试，**数字越小，表示消息越重要**。当设置 `/proc/sys/kernel/printk` 的值为某个级别时，**只有该级别及更高级别的消息（数字更小的消息）才会输出到控制台**。
  - 使用 `journalctl` 命令
    ```bash
    journalctl          # 查看所有日志
    journalctl -b       # 查看内核日志
    journalctl -f       # 实时查看日志
    ```