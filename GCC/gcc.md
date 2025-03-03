# gcc的基本使用
预处理-> 编译 -> 汇编 -> 链接 -> 反汇编
.c -> .i -> .S -> .o(OBJ文件) -> .elf(可执行链接文件，包含多种) -> .dis

## 关于头文件和库
- `<>` 类头文件
  在编译器默认路径下寻找，通过编译添加 `-v` 选项可以查找到头文件 `include` 目录路径。
- `""` 类头文件
  默认在当前位置下寻找该头文件，编译时可以通过 `-I` 选项指定需要包含的头文件路径。
- 库路径
  默认为编译器中的 `lib` 目录或者为别的 `.c` 文件，编译时可以通过 `-L` 选项指定目录。`-l` 是用来指定库名称。

## 常用命令
1. 默认编译
   ```bash
   gcc hello.c  # 输出一个名为 a.out 的可执行程序
   ```
2. 输出指定文件名称
    ```bash
    gcc -o hello hello.c    # 指定输出名为 hello 的执行程序，经过了预处理、编译、汇编等步骤
    ```
3. 静态链接
    ```bash
    gcc -o hello hello.c -static
    ```
4. 先编译，不链接
    ```bash
    gcc -c -o hello.o hello.c
    ```
5. 单独链接
    ```bash
    gcc -o hello hello.o
    ```
```bash
gcc -E -o hello.i hello.c   # 预处理
gcc -S -o hello.s hello.i   # 编译
gcc -c -o hello.o hello.s   # 汇编
gcc -o hello hello.o        # 链接
```

## 常用选项
- `-E` 预处理
- `-S` 编译
- `-c` 把预处理、编译、汇编都做了，不链接
- `-o` 指定输出文件
- `-I` 指定头文件目录
- `-L` 指定链接时库文件目录
- `-l` 指定链接时的库文件名称

## 执行 `gcc -o hello hello.c -v`时运行过程：
1. `cc1 hello.c -o /tmp/ccjSVLTM.s`
   预处理和编译被放在了一个命令 `cc1` 中进行，可以把它拆分为两步：
    ```bash
    cpp -o hello.i hello.c
    cc1 hello.i -o /tmp/ccjSVLTM.s
    ```
2. `as -v --64 -o /tmp/ccqEzh6u.o /tmp/ccjSVLTM.s`
3. `collect2 -o hello /tmp/ccqEzh6u.o`
    - `ld` 被 `collect2` 调用来链接程序。`ld`，是 `GNU` 链接器，是 `GNU Binutils` 工具集的一部分，主要负责将多个目标文件（`.o` 文件）和库文件链接成可执行文件或共享库。`collect2` 是 `GCC` 编译系统的一部分，主要用于处理 C++ 程序的全局静态对象的构造和析构，同时还能通过调用 `ld` 来完成最终的链接过程。
    - `ld 的 -static 选项`：
    ```bash
    # 编译源文件为对象文件
    gcc -c source1.c -o source1.o
    gcc -c source2.c -o source2.o
    gcc -c main.c -o main.o
    # 创建静态库
    ar rcs libmylib.a source1.o source2.o
    # 使用 -static 选项链接静态库
    ld -static -o myprogram main.o -L. -lmylib
    ```
    - 当指定目录下已存在需要链接的静态库时：
    ```bash
    gcc -static -o myprogram main.c source1.c source2.c -L. lmylib
    ```
- 当需要编译并链接多个文件时，重复 .1 .2 中的的内容，并在 .3 中将生成的多个 .o 文件链接成目标文件，直接将 .o 文件写在后面。

## 预处理结果查看、宏展开、依赖文件生成、列出头文件目录、库文件目录
1. `gcc -E hello.c`
2. `gcc -E -dM hello.c > 1.txt`：将宏展开内容保存到 1.txt 文件中
3. `gcc -Wp,-MD,hello.dep -c -o hello.o hello.c`：将依赖信息内容保存到 hello.dep 文件中
4. `echo 'main(){}' | gcc -E -v -`：列出头文件目录、库文件目录

## 创建共享库
- `fPIC` 选项
  1. **位置无关代码**（PIC，Position-Independent Code， PIC），是指代码在运行时可以被加载到内存的任意位置而不需要修改。这种代码不依赖于固定的内存地址，因此可以在不同的内存位置运行。这在共享库（如 `.so` 文件）中非常重要，因为共享库可以在不同的程序中被加载到不同的内存位置。
  2. `-fPIC`：当使用 `-fPIC` 选项时，GCC 会生成完全位置无关的代码。这种代码可以在任何地址空间中运行，而不需要进行重定位。这使得生成的代码可以被共享库使用，因为共享库需要在不同的程序中被加载到不同的内存位置。
  3. `-fpic`：与 `-fPIC` 的区别：GCC 还提供了 `-fpic` 选项，它生成的是小位置无关代码（Small Position-Independent Code）。这种代码在某些情况下可能比 `-fPIC` 生成的代码更小，但它的适用范围更有限，可能无法在所有地址空间中运行。`-fpic` 通常用于目标平台的地址空间较小或对代码大小有严格限制的情况。
  4. 使用场景：
     1. 共享库：在创建共享库时，通常会使用 `-fPIC` 选项，以确保生成的代码可以在不同的程序中被加载到不同的内存位置；
     2. 动态加载模块：在某些情况下，程序可能会动态加载模块。这些模块也需要是位置无关的，因此会使用 `-fPIC` 选项。
    ```bash
    gcc -fPIC -c source1.c source2.c -o source1.o source2.o
    gcc -shared -o libexample.so source1.o source2.o
    ```

- 静态库 `.a`
    ```bash
    gcc -c source1.c source2.c -o source1.o source2.o
    # 或者分别编译
    gcc -c source1.c -o source1.o
    gcc -c source2.c -o source2.o
    # 使用 ar 工具将对象文件打包为静态库
    ar rcs libmylib.a source1.o source2.o       # r: 插入或替换成员；c：创建一个新的归档文件；s：生成索引
    # 链接静态库到可执行文件
    gcc main.c -o main -L. -lmylib              # -L.:指定当前目录作为库的搜索路径；-lmylib:链接名为 libmylib.a 的库
    ```

- 动态库 `.so`
    ```bash
    gcc -fPIC -c source1.c source2.c -o source1.o source2.o
    # 或者分别编译每个源文件
    gcc -fPIC -c source1.c -o source1.o
    gcc -fPIC -c source2.c -o source2.o
    # 创建动态库
    gcc -shared -o libmylib.so source1.o source2.o
    # 链接动态库到可执行文件
    gcc main.c -o main -L. -lmylib
    # 在运行可执行文件时，确保动态库的路径包含在 LD_LIBRARY_PATH 环境变量中：
    export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH
    ./main
    ```