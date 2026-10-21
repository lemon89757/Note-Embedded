# 汇编笔记

## GNU 汇编
> [GAS, GNU Assembler](https://sourceware.org/binutils/docs/as/index.html)
> as.pdf

1. `.type` - 声明符号类型
   - 在 *GNU ARM* 汇编（*gas*）中，`.type` 和 `.size` 是 *ELF* 符号属性伪指令，用于向链接器（*ld*）和调试器描述符号的性质与范围。
   - 语法：
    ```asm
    .type <symbol>, #<type>
    ```
    - 它把符号标记为 ELF 符号表中的特定类型，常见取值：

    | 类型 | 含义 | 典型场景 |
    |-----|-----|-----|
    | `#object` | 数据对象（变量/结构体/常量表） | 定义 `struct proc_info_list`、`struct machine_desc` 等 |
    | `#function` | 可执行函数 | 定义汇编函数入口 |
    | `#notype` | 无特定类型 | 默认 |
   - 示例代码
    ```asm
    /* arch/arm/mm/proc-v7.S */
    .type __v7_proc_info, #object
    __v7_proc_info:
        .long   0x000f0000      @ CPU ID mask
        .long   0x000f0000      @ CPU ID value
        .long   PMD_TYPE_SECT | \
                PMD_SECT_AP_WRITE | \
                PMD_SECT_AP_READ | \
                PMD_FLAGS
        .long   MMU_FLAGS
        ...
    .size __v7_proc_info, . - __v7_proc_info
    ```

2. `.size` - 声明符号大小
   - 语法：
    ```asm
    .size <symbol>, <expression>
    ```
   - 通常配合 `.`（当前地址计数器）使用：
    ```asm
    .size __v7_proc_info, . - __v7_proc_info
    ```
    - 含义：该符号占用的字节数 = 当前地址 − 符号起始地址。链接器据此填充 *ELF* 符号表的 `st_size` 字段。

3. 宏定义
   - *macro* 格式：
       ```asm
       .macro 宏名 参数名列表   @伪指令.macro定义一个宏
       指令序列
       .endm                   @.endm表示宏结束
       ```
   - *define* 格式
       ```asm
       #define 标识符 值
       ```

## ARM 汇编
> [Linux GNU ARM 汇编语法指南](https://softool.cn/read/gnu_arm_assembly_syntax_guide/macro.html)

1. *CPS* 指令
   > 5.6.3 PSR modification: In a privileged mode the Change Processor State(CPS) instruction can be used to directly modify the mode and interrupt-enable or disable (I and F) bits in the CPSR.
   
   补充：也可以使用 *MRS、MSR* 指令通过读、修改、写回的方式修改 *CPSR* 寄存器的 *M[4:0]*，从而让 *CPU* 进入不同的模式。

2. *WFI* 和 *WFE* 指令
    > [ARM hint instruction-WFI(Wait For Interrupt)指令详解](https://blog.csdn.net/luolaihua2018/article/details/126773265)

3. 内存屏障指令
    > [ARM基础(6)：内存屏障指令之DMB、DSB和ISB详解](https://blog.csdn.net/tilblackout/article/details/131949061)
    > [ARM64中的内存屏障指令（DMB,DSB,ISB）](https://zhuanlan.zhihu.com/p/601037646)
   - `DMB`
   - `DSB`
   - `ISB`

## 内联汇编
现代编译器已经足够优秀，大部分的 C 代码转成汇编码后，效率都很高。但是，有些使用汇编代码的情况：
- 特殊的算法需要手工优化，这时就需要手写汇编代码；
- 有时需要调用特殊的汇编指令，比如使用 `ldrex` 和 `strex` 实现互斥访问；

1. 反汇编：
2. 单独汇编文件 `add.S`
    ```asm
    .text
    .global add
    .thumb

    add:
        add r0, r0, r1
        bx lr
    ```
    - `.text`：指定程序放在代码段；
    - `.global add`：指定 `add` 为全局符号，可供其他文件使用；
    - `.thumb`：使用 thumb 指令。main.c 默认使用 thumb 指令，所以这里也使用 thumb 指令；
    - 这里，`add()` 函数连栈都没有使用，非常高效。在工作中并不使用汇编来进行“加法优化”，**在计算量非常大的地方可以考虑单独编写汇编函数实现优化**。
3. 内联汇编语法：
   ```
   asm asm-qualifiers (AssemblerTemplate
                    : OutputOperands
                    [ : InputOperands
                    [ : Clobbers]])
   ```
   - `asm`，也可以写成 `__asm__`，表示这是一段内联汇编；
   - `asm-qualifiers`：有 3 个取值：
     - `volatile`：易变的、不稳定的。用以告诉编译器不要随便优化这段代码。
     - `inline`
     - `goto`
   - `AssemblerTemplate`：汇编指令，用双引号包含起来，每条指令用 `\n` 分开，示例：
        ```asm
        "mov %0, %1\n"
        "add %0, %1, %2\n"
        ```
   - `OutputOperands`：输出操作数。内联汇编执行时，输出的结果保存在这里。当有多个变量时用逗号隔开，格式如下：
        ```asm
        [[asmSymbolicName] constraint (cvariablename)]
        ```
        - `asmSymbolicName`：符号名，可以省略；
        - `constraint`：表示约束，常用值有：
          - `m`：`memory operand`，表示要传入有效的地址，只要 CPU 能支持该地址，就可以传入；
          - `r`：`register operand`，寄存器操作数，使用寄存器来保存这些操作数；
          - `i`：`immediate integer operand`：表示可以传入一个立即数；
        - `constraint` 还可以加上一些修饰字符：
          - `=`：表示内联汇编会修改这个操作数，即：写（输出参数常用）；
          - `+`：这个操作数既被读、被写；
          - `&`：它是一个 earlyclobber 操作数；
        - `cvariablename`：C 语言的变量名；
        ```asm
        [result] "=r" (sum)
        ```
        - 上述代码的意思是，汇编代码中会通过某个寄存器（`r`）把结果写入（`=`）sum （`cvariablename`）变量。在汇编代码中可以使用 `%[result]`（`[asmSymbolName]`）来引用它。
    - `InputOperands`：输入操作数，内联汇编执行前，输入的数据保存在这里。格式如下：
        ```asm
        [[asmSymbolName] constraint (cexpression)]
        ```
        - `asmSymbolName`：同上；
        - `constraint`：同上；
        - `cexpression`：C 语言的表达式。
        ```asm
        [a_val] "r" (a), [b_val] "r" (b)
        ```
        - 上述代码表示，变量 a、b 的值会放入某些寄存器。在汇编代码中可以通过 `%[a_val]` 和 `%[b_val]` 来使用它们。
    - `Clobbers`：在汇编代码中，可能需要使用的寄存器、内存会更多，必须在 `Clobbers` 中进行声明。常用的有：
      - `cc`：表示汇编代码会修改 `flags register`；
      - `memory`：表示汇编代码中，除了 `InputOperands` 和 `OutputOperands` 中指定的之外，还会读、写更多的内存。比如 `"r0", "r1", "r2", "r3", "r4", "r5", "memory"`。
4. 示例程序
   - 使用内联汇编实现加法
        ```c
        int add(int a, int b) {
            int sum;
            __asm__ volatile (
                "add %0, %1, %2" // 将第 1，2 个操作数相加，存入第 0 个操作数
                :"=r"(sum)
                :"r"(a), "r"(b)
                :"cc"
            );
            return sum;
        }

        // 另一种写法，在 Linux 内核中这种用法比较少见
        int add(int a, int b) {
            int sum;
            __asm__ volatile (
                "add %[result], %[val1], %[val2]"
                :[result]"=r"(sum)
                :[val1]"r"(a), [val2]"r"(b)
                :"cc"
            );
            return sum;
        }
        ```
5. 关于输出操作数的约束修饰 `&` 的说明
   - 编译器可能会对输出操作数和输入操作数使用同一个寄存器（`add r3, r2` 中输出使用的寄存器为 `r3`，一个输入也使用了 `r3` 寄存器），如果在返回结果之前继续操作这个寄存器会导致返回错误的结果。
   - 解决方式，明确告诉编译器分配单独的寄存器，这可以通过修饰符 `&` 进行说明：
    ```c
    int add(int a, int b) {
        int sum;
        __asm__ volatile (
            "add %0, %1, %2\n"
            "add %1, #1\n"
            "add %2, #1\n"
            :"=&r"(sum)
            :"r"(a), "r"(b)
            :"cc"
        );
        return sum;
    }
    ```

## 参考
### ARM 汇编指令
1. ARM Cortex-A(armV7)编程手册V4.0

### 内联汇编
1. 韦东山-驱动大全
2. [GNU C 扩展汇编](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)
3. [ARM GCC 内嵌（inline）汇编手册](http://blog.chinaunix.net/uid-20543672-id-3194385.html)
4. [C 内联汇编](https://akaedu.github.io/book/ch19s05.html)

