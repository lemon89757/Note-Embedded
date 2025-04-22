# make usage

## 编译内核模块（驱动文件）
1. `Makefile` 文件内容：
```Makefile
# 指定 Linux 源码路径
## arm
KERN_DIR = /home/melodic/alientek/Linux-src/src/
## x86-64
# KERN_DIR = /lib/modules/5.15.0-130-generic/build
# KERN_DIR = /lib/modules/$(shell uname -r)/build

# PWD:=(shell pwd)

all:
# -C $(KERN_DIR): 切换到内核源码目录；内核模块的编译依赖于内核源码
# M=`pwd`		: 指定当前目录作为模块的源码目录
# modules		: 告诉 make 只编译模块
# 由于在环境变量中定义了 CROSS_COMPILE，内核构建系统从环境变量中获取到
# 该值后，编译驱动模块使用的编译器即 ${CROSS_COMPILE}-gcc
# 下述命令会执行 obj-m 中的内容
	make -C $(KERN_DIR) M=`pwd` modules
# make -C $(KERN_DIR) M=$(PWD) modules
	$(CROSS_COMPILE)gcc -o hello_drv_test hello_drv_test.c

clean:
	make -C $(KERN_DIR) M=`pwd` modules clean
	rm -rf modules.order
	rm -f  hello_drv_test
# rm -f *.ko *.o *.symvers *.cmd *cmd.o modules.* *.mod.c

# 指定需要编译的内核模块对象文件，追加内容 +=
obj-m += hello_drv.o
```

2. `make` 输出信息：
- `CC`: 表示正在执行 C 编译器（通常是 gcc）来编译源代码；
- `CC [M]`: `[M]` 表示当前正在编译的是一个内核模块；
- `LD`: 表示正在执行链接器 (通常是 `ld`) 来链接对象，生成最终的内核模块；
- `LD [M]`: 与 `CC [M]` 同理；