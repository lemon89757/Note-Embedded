# Note for ARM
1. *AAPCS*
   - *ARM Architecture Procedure Call Standard*，*ARM* 架构过程调用标准，是 *ARM ABI* 的一部分，用于定义 ARM 架构下函数调用的统一规则，包括：
     1. 参数如何传递
     2. 返回值如何返回
     3. 哪些寄存器由调用者保存，哪些由被调用者保存
     4. 栈的使用方式与对齐要求
   - 函数参数传递（如 *R0~R3*）
   - 返回值置位（*R0* 或 *R0~R1*）
   - *R14*（*LR*）：函数返回地址
   - 寄存器保存规则（如 *R4~R11* 为被调用者保存）
   - 栈对齐要求（通常 8 字节）
2. *ATPCS*
   > [ATPCS 和 AAPCS](https://www.cnblogs.com/jontian/p/6115312.html)
   - *ATPCS, ARM-Thumb Procedure Call Standard*
3. *CAS* 原子操作的 *ARM* 实现
   - *CAS，Compare-And-Swap*，是一种无锁原子操作，通过比较内存值与预期值并在相等时更新，实现多线程环境下的线程安全。
   - 包含三个操作数：内存值-*V*、预期值-*A*、新值-*B*
   - *ARM* 架构通过 `LL/SC, load-linked/Store-Conditional` 指令实现，`LDREX` 加载内存并标记，`STREX` 在标记未失效时写入新值。

## 协处理器寄存器指令

## MMU
1. *TLB*，地址转换后备缓冲器，常称为快表，用以缓存虚拟地址到物理地址的映射结果

2. *TTBR, Translation Table Base Register*，即翻译表基址寄存器
   - 是 *ARM* 架构 *MMU* 中用于存放页表基地址的寄存器
   - *CPU* 进行虚拟地址到物理地址翻译时，需要先知道页表在内存中的位置，*TTBR* 就是告诉 *MMU*“页表从哪里开始”的那个寄存器
   - 在 *ARMv7* 中，通常有两个 *TTBR*：
     - *TTBR0*：存放用户空间页表的基地址。进程切换时，它会被更新为新进程的 *PGD* 地址
     - *TTBR1*：存放内核空间页表的基地址。内核空间在所有进程中共享，所以 *TTBR1* 通常固定不变
   - *CPU* 翻译一个虚拟地址时，会根据地址的高位判断它属于用户空间还是内核空间，然后选用 *TTBR0* 或 *TTBR1* 作为页表遍历的起点
   - 在 *ARMv7* 中，*TTBR* 不只包含页表基地址，还包含 *ASID* 和若干控制位

3. *ASID, Address Space Identifier*，即地址空间标识符
   - 是 *ARM MMU* 中用于区分不同进程地址空间的一个标识，它被写入 *TTBR* 的高位，与页表基址一起参与地址翻译
   - 它的核心作用是避免每次进程切换都刷新整个 *TLB*
     - 没有 *ASID* 时，进程切换必须清空 *TLB*，否则新进程可能命中旧进程的地址翻译缓存，导致错误
     - 有了 *ASID*，*TLB* 表项会带上“这个翻译属于哪个 *ASID*”的标记。切换进程时，*CPU* 只需换一个 *ASID*，旧进程的 *TLB* 表项仍然保留，新进程不会误用它们。当旧进程再次运行时，如果 *ASID* 未变，它的 *TLB* 表项可能还在，从而加速地址翻译
   - *ASID* 的位数由硬件决定
