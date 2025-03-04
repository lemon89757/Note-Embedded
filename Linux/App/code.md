# Linux's APP

## 关于函数详情、头文件
- 通过 `man page`(`man man`命令) 可以查找到相关内容，如 section 1 中是“可执行程序或者 `shell` 命令”、section 2 中是系统调用、section 3 中是库函数调用等等。
```bash
# open() 为系统调用函数，在 section 2 中
man 2 open
# 通过上述命令即可查找到关于 open() 的详细信息
```
- 另外，更加详细的信息可以通过 `info` 命令来查找，但是感觉 `info` 的内容过于多，反而不利于找到关键信息。