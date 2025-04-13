# Linux's APP

## 关于函数详情、头文件
- 通过 `man page`(`man man`命令) 可以查找到相关内容，如 section 1 中是“可执行程序或者 `shell` 命令”、section 2 中是系统调用、section 3 中是库函数调用等等。
```bash
# open() 为系统调用函数，在 section 2 中
man 2 open
# 通过上述命令即可查找到关于 open() 的详细信息
```
- 另外，更加详细的信息可以通过 `info` 命令来查找，但是感觉 `info` 的内容过于多，反而不利于找到关键信息。

## 关于 `mmap` 函数
1. 在 `copy_mmap.c` 文件中，直接通过将文件描述符 `fd_old` 映射到内存空间进行对应文件数据的拷贝：
```C
...
if(fstat(fd_old, &stat) == -1){
    printf("can not get status of file %s.\n", argv[1]);
    return -1;
}

buf = mmap(NULL, stat.st_size, PROT_READ, MAP_SHARED, fd_old, 0);   // stat.st_size: size of file, in bytes.
if(buf = MAP_FAILED){
    printf("can not mmap file %s.\n", argv[1]);
    return -1;
}
...

if(write(fd_new, buf, stat.st_size) != stat.st_size){
    ...
}
```
2. 在 `showpixel.c` 和 `show_ascii.c` 中，通过将 frame buffer 映射到内存中，对 lcd 进行操作：
```C
...
fd_fb = open("dev/fb0", O_RDWR);
if(fd_fb < 0){
    ...
}
if(ioctl(fd_fb, FBIOGET_VSCREENINFO, &var)){
    ...
}
...
fb_base = (unsigned char *)mmap(NULL, screen_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd_fb, 0);
if(fb_base == MAP_FAILED){
    ...
}
// 清屏
memset(fb_base, 0xff, screen_size);      // 在这里为什么在操作完这块内存后显示就能更新，明明它还没有对相关设备进行写操作
...
```
- 帧缓冲设备的**物理内存通过 `mmap` 被映射到用户空间**后，任何对映射内存的写操作都会直接写入物理内存；
- 显示硬件（如 LCD 控制器）会**定期读取帧缓冲设备的物理内存**，并将其内容显示在屏幕上；
因此，当程序通过 `memset` 修改映射内存时，这些改动会立即反应到屏幕上。