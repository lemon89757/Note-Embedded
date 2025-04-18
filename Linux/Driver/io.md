# About IO

## `ioremap`
- 位置：`asm/io.h`
- 功能：从实际物理地址映射到进程中的逻辑地址
- 宏定义原型：`virt_addr = ioremap(phys_addr, size)`
- 注意事项：
  - 实际上 `ioremap` 是**按页**（4K, 4096字节）进行映射的，是整页整页地映射的。eg: 若 `phys_addr = 0x10002, size = 4`, 则 `ioremap` 的内部实现是：1. `phys_addr` 按页取整（4096,0x1000），得到地址 0x10000; 2. `size` 按页取整，得到 4096; 3. 把起始地址 0x10000，大小为 4096 的这一块物理地址空间，映射到虚拟地址空间; 4. 假设得到的虚拟空间起始地址为 0xf0010000，则 `phys_addr = 0x10002` 对应的 `virt_add = 0xf0010002`.
  - 不再使用该段虚拟地址时，要 `iounmap(virt_addr)`: `void iounmap(volatile void __iomem *cookie)`