# dma 子系统

## Note:Dynamic DMA mapping Guide
### GPU and DMA address
There are several kinds of addresses involved in thre DMA API, and it's important to understand the differencees.
- 虚拟地址（`virtual address`）
  - 例子：
    - 快表（`TLB`）；
    - 页表（`page tables`）；
- 物理地址（`physical address`）
  - 例子：
    - `CPU`；
    - 内核管理像寄存器这样的设备资源使用物理地址；
  - 表示（数据结构）：
    - `phys_addr_t`；
    - `resource_size_t`；
  - 信息查找：
    - `/proc/iomem`；
- `bus address`
  - 例子：
    - `I/O` 设备（if a device has registers at an MMIO address, or if it performs DMA to read or write system memory, the address used by the device are bus address）；


### 三者间的关系
- 虚拟地址和物理地址
  - 驱动程序是不能够直接使用物理地址的，需要通过 `ioremap()` 函数进行映射从而得到一个虚拟地址（以供驱动使用）；

- 物理地址和 bus address
  - 在一些系统中，`bus address` 和物理地址是相同的。但是在通常情况下，它俩是不同的。
  - `IOMMUs` 和 `host bridges` 能够在物理地址和总线地址之间生成任意的映射关系；


从设备的角度触发，`DMA` 使用的是总线地址，或者是总线地址空间的子集。例如，即使一个系统的主存和 `PCI BARs` 支持 `64-bit` 的地址，但它可能使用 `IOMMU` 从而只需要使用 `32-bit` 的 `DMA` 地址。
```
          CPU                  CPU                  Bus
           Virtual              Physical             Address
           Address              Address               Space
            Space                Space

          +-------+             +------+             +------+
          |       |             |MMIO  |   Offset    |      |
          |       |  Virtual    |Space |   applied   |      |
        C +-------+ --------> B +------+ ----------> +------+ A
          |       |  mapping    |      |   by host   |      |
+-----+   |       |             |      |   bridge    |      |   +--------+
|     |   |       |             +------+             |      |   |        |
| CPU |   |       |             | RAM  |             |      |   | Device |
|     |   |       |             |      |             |      |   |        |
+-----+   +-------+             +------+             +------+   +--------+
          |       |  Virtual    |Buffer|   Mapping   |      |
        X +-------+ --------> Y +------+ <---------- +------+ Z
          |       |  mapping    | RAM  |   by IOMMU
          |       |             |      |
          |       |             |      |
          +-------+             +------+
```
- During the enumeration process, the kernel learns about I/O devices and their MMIO space and the host bridges that connect them to the system. For example, if a PCI device has a BAR, the kernel reads the bus address(A) from the BAR and converts it to a CPU physical address(B). The address B is stored in a struct resource and usually exposed via /proc/iomem. when a driver claims a device, it typically uses ioremap() to map physical address B at a virtual address(C). It can then use, e.g., ioread32(C), to access the device registers at bus address A.
- if the device supports DMA, the driver sets up a buffer using kmalloc() or s similar interface, which returns a virtual address(X). The virtual system maps X to a physical address(Y) in system RAM. **The driver can use virtual address X to access the buffer, but the device itself cannot because DMA doesn't go through the CPU virtual memory system**.
- In some simple systems, the device can do DMA directly to physical address(Y). But in many others, there is IOMMU hardware that translates DMA addresses to physical addresses, e.g., it translates Z to Y. This is part of the reason for the DMA API: the driver can give a virtual address X to an interface like dma_map_single(), which sets up any required IOMMU mapping and returns the DMA address Z. The driver then tells the device to do DMA to Z, and the IOMMU maps it to the buffer at address Y in system RAM.

### 使用
- So that Linux can use the dynamic DMA mapping, it needs some help from the drivers, namely it has to take into account that **DMA addresses should be mapped only for the time they are actually used and unmapped after the DMA transfer**.
- The following API will work of course even on platforms where no such hardware exists.
- Note that the DMA API works with any bus independent of the underlying microprocessor. **You should use the DMA API rather than the bus-specific DMA API, i.e., use the dma_map_*() interface rather than the pci_map_*() interfaces**.
- 包含头文件：
    ```c
    #include <linux/dam-mapping.h>
    ```
    - 提供数据类型 `dma_addr_t` 的定义，它可以用以保存任何有效的 DMA 地址；
    - `dma_addr_t` 类型用以保存从 DMA mapping 函数返回的任意 DMA 地址； 
- 可以用于 DMA 的内存（`memory`）
  acquired your memory:
  - via the page allocator(i.e. __get_free_page*());
  - via the generic memory allocators(i.e. kmalloc() or kmem_cache_alloc)
  
  you may DMA to/from that memory using the address returned from those routines.
- 不可用于 DMA 的内存（`memory`）
  - vmalloc()(It is possible to DMA to the memory mapped into a vmalloc() area, but this requires walking page tables to get the physical addresses, and then translating each of those pages back to kernel address using something like __va());
  - kmap() return;
  - kernel image addresses(items in data/text/bss segments);
  - module image addresses;
  - stack addresses;
- `__dma_from_device_group_begin/end annotations`
这句话解释了 DMA（直接内存访问）缓存一致性问题导致的数据损坏机制：
  - 核心概念
    | 术语 | 含义 |
    |------|------|
    | DMA_FROM_DEVICE | 设备向内存写入数据（设备→内存） |
    | DMA_BIDIRECTIONAL | 双向 DMA 传输 |
    | DMA-incoherent caches | DMA 非一致性缓存（CPU 缓存与设备看到的内存内容可能不一致） |
  - 问题场景
    ```
    结构体布局示例：
    ┌─────────────────┬─────────────────┐
    │  DMA缓冲区(设备写)  │  CPU写字段       │
    │  (DMA_FROM_DEVICE) │                 │
    └─────────────────┴─────────────────┘
        ↑ 共享同一缓存行 ↑
    ```
  - 数据损坏原因
    **缓存行共享**（cache line sharing）导致的问题：
    1. **设备写入 DMA 缓冲区** → 写入主内存
    2. **CPU 写入相邻字段** → 由于共享缓存行，CPU缓存加载整行
    3. **缓存不一致** → CPU 可能用旧缓存覆盖设备新写入的数据，或设备看到CPU 脏数据
  - 解决方案暗示
    需要确保 DMA 缓冲区与 CPU 写字段**位于不同缓存行**（通过填充/对齐），防止缓存行共享。

    ```c
    struct my_device {
        spinlock_t lock1;
        __dma_from_device_group_begin();
        char dma_buffer1[16];
        char dma_buffer2[16];
        __dma_from_device_group_end();
        spinlock_t lock2;
    };
    ```
    - To isolate a DMA buffer from adjacent fields, use `__dma_from_device_group_begin(GROUP)` before the first DMA buffer field and `__dma_from_device_group_end(GROUP)` after the last DMA buffer field (with the same GROUP name). **This protects both the head and tail of the buffer from cache line sharing**.
    - The GROUP parameter is an optional identifier that names the DMA buffer group (int case you have several in the same structure):

    ```c
    struct my_device {
        spinlock_t lock1;
        __dma_from_device_group_begin(buffer1);
        char dma_buffer1[16];
        __dma_from_device_group_end(buffer1);
        spinlock_t lock2;
        __dma_from_device_group_begin(buffer2);
        char dma_buffer2[16];
        __dma_from_device_group_end(buffer2);
    };
    ```
- DMA addressing capabilities
    **By default, the kernel assumes that your device can address 32-bits of DMA addressing**. For 64-bit capable device, this needs to be increased, and for a device with limitations, it needs to be decreased.

    For correct operation, you must set the DMA mask to inform the kernel about your devices DMA addressing capabilities.

    ```c
    // set the mask for both streaming and coherent APIs together
    // never return fail when DMA_BIT_MASK(64) and never return failure when bigger than 32.
    int dma_set_mask_and_coherent(struct device *dev, u64 mask);
    // setup for streaming mappings
    int dma_set_mask(struct device *dev, u64 mask);
    // setup for coherent allocations
    // one would have to check the return value from dma_set_coherent_mask
    int dma_set_coherent_mask(struct device *dev, u64 mask);

    // eg:
    // 当设置出错时产生日志信息
    if (dma_set_mask_and_coherent(dev, DMA_BIT_MASK(24))) {
        dev_warn(dev, "mydev: No suitable DMA available\n");
        goto ignore_this_device;
    }

    if (dma_set_mask(dev, DMA_BIT_MASK(24))) {
        dev_warn(dev, "mydev: 24-bit DMA addressing not available\n");
        goto ignore_this_device;
    }

    // if the device only supports 32-bit addressing for descriptors in the coherent allocations, but supports full 64-bits for streaming mappings it would look like this:
    if (dam_set_mask(dev, DMA_BIT_MASK(64))) {
        dev_warn(dev, "...");
        goto ignore_this_device;
    }
    // the coherent mask will always be able to set the same or a smaller mask as the streaming mask.
    ```
- Types of DMA mappings:
  - Coherent DMA mappings
    - 通常在驱动初始化时进行 `map`，在结束时进行 `unmap`；
    - 硬件应当保证设备和 CPU 可以在并发情况下获取数据；并且在没有显示软件 `flushing` 的情况下，彼此都能够看见对方（对数据）的更新操作；
    - 适用情况：
      - network card DMA ring descriptos;
      - SCSI adapter mailbox command data structures;
      - device firmware microcode executed out of main memory.
  - Streaming DMA mapping
    - mapped for one DMA transfer, unmapped right afer it (unless you use dma_sync_* below);
    - 适用情况：
      - networking buffers transmitted/received by a device;
      - filesystem buffers written/read by a SCSI device;

- Using Coherent DMA mmapings
  - To allocate and map large (PAGE_SIZE or so) coherent DMA regions:
    ```c
    dma_addr_t dma_handle;
    // cpu_addr 保存函数返回的虚拟地址；
    // dma_handle 用以保存 DMA 地址
    // 两者都是传出参数，都以 PAGE_SIZE 进行对齐
    cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, gfp);
    ```
    - flag 为 GFP_ATOMIC，该函数可以在中断上下文中调用；
    - 该函数将会分配 RAM；
    - 如果需要的 regions 大小小于一个页（的大小），更倾向于适用 `dma_pool` 的相关接口函数；
  - To unmmap and free such DMA region:
    ```c
    dma_free_coherent(dev, size, cpu_addr, dma_handle);
    ```
    - 参数与 `dma_alloc_coherent` 中参数相等；
    - 不可再中断上下文中调用；
  - To create a dma_pool and use:
    ```c
    struct dma_pool *pool;
    // size: ；
    // align: 分配元素的对齐单位；
    // boundary: 分配元素大小的上限；
    pool = dma_pool_create(name, dev, size, align, boundary);

    // allocate memory from a DMA pool
    // flags are GFP_KERNEL if blocking is permitted (not in_interrupt nor holding SMP locks), GFP_ATOMIC otherwise.
    cpu_addr = dma_pool_alloc(pool, flags, &dma_handle);

    // free memory
    dma_pool_free(pool, cpu_addr, dma_handle);

    // destory dma_pool
    // make sure you've called dma_pool_free() for all memory allocated from a pool before you destory the pool.
    // this function may not be called in interrupt context.
    dma_pool_destory(pool);
    ```

- DMA Diretion
  - DMA_BIDIRECTIONAL
  - DMA_TO_DEVICE: from main memory to device;
  - DMA_FROM_DEVIE: from the device to main memory;
  - DMA_NONE

    - you are strongly encouraged to specify this as precisely as you possibly can.
    - DMA_NONE is to be used for debugging.
    - DMA_BIDIRECTIONAL will work, but this may be at the cost of performance for example.
    - only streaming mappings specify a direction, coherent mappings implicitly have a direction attribute setting of DMA_BIDIRECTIONAL.
- Using Streaming DMA mappings
  The streaming DMA mapping routines can be called from interrupt context. There are two versions of each map/unmap, one which will **map/unmap a single memory region**, and one which will **map/unmap a scatterlist**.
  - single memory region
    ```c
    struct device *dev = &my_dev->dev;
    dma_addr_t dma_handle;
    void *addr = buffer->ptr;
    size_t size = buffer->len;

    dma_handle = dma_map_single(dev, addr, size, direction);
    if (dma_mapping_error(dev, dma_handle)) {
        // reduce current DMA mapping usage,
        // delay and try again later or
        // reset driver.
        goto map_error_handling;
    }

    // to unmap it
    dma_unmap_single(dev, dma_handle, size, direction);
    ```
  - with scatterlists, you map a region gathered from several regions:
    ```c
    int i, count = dma_map_sg(dev, sglist, nents, direction);
    struct scatterlist *sg;

    for_each_sg(sglist, sg, count, i) {
        hw_addressp[i] = sg_dma_address(sg);
        hw_len[i] = sg_dma_len(sg);
    }
    ```

## 补充内容
### `MMIO`
### `IOMMU`

## 参考
1. [Linux DMA 详解：原理、子系统与实践指南](https://geek-blogs.com/blog/linux-dma/#7-%E7%A4%BA%E4%BE%8Blinux-%E5%86%85%E6%A0%B8%E6%A8%A1%E5%9D%97%E4%B8%AD%E7%9A%84-dma-%E4%BD%BF%E7%94%A8)
2. [DMA 子系统](https://malism.net/linux/advanced/dma.html)
3. [Dynamic DMA mapping Guide](https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html)