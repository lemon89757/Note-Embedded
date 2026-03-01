# V4L2 子系统
`v4l2, video for linux 2` 子系统

## 框架简介
- [ ] 补充 v4l2 框架结构
- [ ] 补充与 v4l2 子系统与 media controller(media framework) 和 vb2 子系统之间的关系
1. `v4l2 core`：设备管理、提供统一接口等；
2. `media framework`：摄像头设备由各种组件构成，组件之间的拓扑连接可由 `media framework` 动态控制，形成各种 `pipeline`，从而可控制摄像头数据流；
3. `vb2`：摄像头数据的载体，由 `vb2` 管理被申请的缓冲内存；根据内存的分配者可以分为三种内存类型：...；

### 硬件框图
### 软件框架

## 摄像头硬件接口
**摄像头与主处理器之间的硬件接口**协议决定了数据传输的速率、复杂度与系统设计。

### MIPI CSI-2
`MIPI CSI-2(Camera Serial Interface 2)` 是移动产业处理器接口联盟制定的高速串行摄像头接口标准，广泛用于智能手机、平板电脑和嵌入式视觉系统中。

1. 核心特性
   - 高速串行传输：采用差分信号（1 对时钟线， 1~4 对数据线），抗干扰能力强；
   - 分层协议栈：
     - 物理层（`PHY`）：定义电气特性、时钟与数据通道；
     - 通道管理层（`Lane Management`）：数据分发与合并；
     - 协议层（`Protocol`）：数据包格式（长包、短包）；
     - 像素/字节到像素映射层（`Pixel/Byte to Pixel Mapping`）：将像素数据打包；
     - 应用层（`Application`）：实际图像数据格式（如 `RAW, YUV, RGB`）；
   - 支持高分辨率与高帧率：单通道可达 `2.5Gbps (CSI-2 v1.3)`，满足 4K/8K 视频需求；
   - 低功耗模式：支持 `ULPS`（超低功耗状态）； 
2. 软件架构
   `MIPI CSI-2 软件栈（Linux）`：
   用户空间应用（`v4l2 API`）-> `v4l2` 子系统和 `Video` 设备节点 -> `CSI-2` 控制器驱动 & 传感器驱动（`I2C` 控制） -> `MIPI CSI-2 PHY` 硬件

### DVP
`DVP, Digital Video Port`，也称为并行接口或 `BT.656/BT1120`，是一种传统的并行数字视频接口，常见于早期的 `CMOS` 传感器和低成本的嵌入式设备中。

1. 核心特性
   - 并行传输：使用多根数据线（通常 8/10/12/16 位）、像素时钟（`PCLK`）、行同步（`HSYNC`）和场同步（`VSYNC`）信号；
   - 时序简单直观：通过 `HSYNC` 和 `VSYNC` 信号界定一帧图像的行与场；
   - 带宽有限：受限于并行总线频率和信号完整性，通常用于较低分辨率（如 720p 以下）；
   - 无复杂协议开销：数据流即像素流，易于调试；
  
2. DVP 信号时序图
    ![](../../../img/subsys-v4l2/260301-165113.png)

### USB UVC
`USB Video Class, UVC` 是 `USB Implementers Forum` 定义的免驱动视频设备标准。任何符合 `UVC` 协议的摄像头（如常见的 USB 网络摄像头）都可以在支持的操作系统上即插即用，无需安装特定驱动。

1. 和新特性
   - 真正的即插即用：操作系统提供标准 `UVC` 驱动，设备厂商无需提供驱动；
   - 基于 USB 总线：利用 USB 的通用性和高带宽（`USB 2.0/3.0/3.1`）；
   - 灵活的格式与帧率协商：通过 USB 描述符报告设备支持的分辨率、像素格式（`MJPEG, YUY2, H264` 等）和帧率；
   - 支持控制功能：如亮度、对比度、白平衡、聚焦等，通过标准的 `UVC` 控制单元实现；

2. 设备类架构图
    ![](../../../img/subsys-v4l2/260301-165814.png)

### 三种接口对比
![](../../../img/subsys-v4l2/260301-165943.png)

## 内存类型与缓冲区队列
视频缓冲区管理是 `V4L2` 子系统的核心，它负责在用户空间和内核空间之间高效、稳定地传出视频数据。涉及到三种内存类型和用于管理这些缓冲区地核心数据结构 `struct vb2_queue`；

理解不同内存类型的特点和 `vb2_queue` 的工作机制，有助于开发者根据应用场景做出最佳选择，并编写出高效、稳定的 `V4L2` 摄像头驱动；

### 内存类型
`V4L2` 定义了三种内存类型，用于指定视频数据缓冲区的分配和管理方式。应用程序通过 `VIDIOC_REQBUFS` `ioctl` 的 `memory` 字段来选择：
```c
struct v4l2_requestbuffers reqbuf;
reqbuf.count    = 4;
reqbuf.type     = V4L2_BUF_TYPE_VIDEO_CAPTURE;
reqbuf.memory   = V4L2_MEMORY_MMAP;     // 指定 MMAP 类型

if (ioctl(fd, VIDIOC_REQBUFS, &reqbuf) == - 1) {
    perror("VIDIOC_REQBUFS");
    exit(EXIT_FAILURE);
}
...
```

1. `MMAP`
   内核负责分配缓冲区内存，用户空间通过 `mmap()` 系统调用将内核缓冲区映射到用户空间地址进行访问。
    - 工作流程：
      - 驱动分配物理连续或散列的缓冲区；
      - 用户调用 `VIDIOC_QUERYBUF` 获取缓冲区信息，如长度、偏移等；
      - 用户调用 `mmap()` 映射缓冲区到用户空间；
      - 通过 `VIDIOC_QBUF/VIDIOC_DQBUF` 进行缓冲区的入队和出队；
    - 优点：
        零拷贝（用户空间直接访问内核缓冲区），效率高；
    - 缺点：
        缓冲区大小和数量在内核分配后固定；

2. `USERPTR`
   缓冲区内存由用户空间应用程序分配（如 `malloc`），将用户空间指针传递给内核驱动使用。
   - 工作流程：
     - 应用程序分配内存，需满足对齐要求等；
     - 通过 `VIDIOC_QBUF` 将包含用户指针的缓冲区结构体提交给驱动；
     - 驱动通过 `DMA` 或 `CPU` 访问该内存区域；
   - 优点：
        应用程序完全控制内存的分配和生命周期；
   - 缺点：
        可能存在额外的拷贝开销，如果硬件 `DMA` 需要物理连续内存；

3. `DMABUF`
   基于 `Linux DMA-BUF` 框架，允许在不同驱动或子系统之间共享缓冲区，无需拷贝。是零拷贝和硬件加速的关键。
   - 工作流程：
     - 缓冲区由导出器（如 `GPU`、`ISP` 驱动）分配并导出为 `DMABUF` 文件描述符（`fd`）；
     - 导入器（如 `V4L2` 驱动）通过该 `fd` 导入并使用缓冲区；
     - 通过 `VIDIOC_QBUF` 传递包含 `dmabuf fd` 的缓冲区结构；
   - 优点：
        真正的零拷贝，支持跨设备、跨子系统共享，适合复杂图像处理流水线；
   - 缺点：
        需要硬件和驱动支持 `DMA-BUF`；

---
1. 对比
    ![](../../../img/subsys-v4l2/260301-190854.png)
   - `MMAP`：最常用，性能好，适合大多数标准视频采集应用；性能高；
   - `USERPTR`：当应用程序需要复用其自身自有内存池时使用，但需要注意性能和数据对齐；性能中等（需复制）；
   - `DMABUF`：`DMA` 缓冲区，可跨设备共享；现代复杂媒体应用的首选，尤其是涉及 `GPU, VPU`，显示等硬件加速和零拷贝流水线；性能最高；

2. 缓冲区生命周期
   1. 分配：通过 `vb2_queue_init()` 和 `vb2_reqbufs()` 分配缓冲区内存；
   2. 映射：用户空间调用 `mmap()` 将内核缓冲区映射到用户空间；
   3. 入队：`VIDIOC_QBUF` 将缓冲区交给驱动/硬件；
   4. 出队：硬件填充数据后，通过 `VIDIOC_DQBUF` 返回给用户；
   5. 释放：流停止或驱动卸载时释放所有缓冲区；

### 缓冲区管理
`videobuf2` 是 Linux `V4L2` 子系统的核心缓冲区管理框架，它为视频设备驱动提供了统一、高效且灵活的缓冲区管理机制。涉及 `vb2_ops` 操作集、缓冲区的分配与内存映射、以及流控制状态的工作原理，是驱动开发者掌握 `V4L2` 高级编程的关键。

`struct vb2_queue` 是 `V4L2` 核心提供的缓冲区队列抽象，**它封装了缓冲区的管理逻辑，驱动开发者只需实现特定操作回调**。

`vb2` 框架极大地简化了驱动开发，开发者只需关注硬件相关的操作，而无需重复实现缓冲区队列管理、同步、等待队列等复杂逻辑。

驱动开发者必须理解和使用的核心框架，正确实现其回调是驱动稳定的关键。

- 数据结构
  ```c
  struct vb2_queue {
    enum v4l2_buf_tye type;
    enum v4l2_memory memory;
    unsigned int io_modes;
    unsigned int num_buffers;
    unsigned int buf_struct_size;       // size of the driver-specific buffer structure
    u32 timestamp_flags;                // Timestamp flags; V4L2_BUF_FLAG_TIMESTAMP_* and V4L2_BUF_FLAG_TSTAMP_SRC_*
    struct list_head queued_list;       // 已排队缓冲区
    struct list_head done_list;         // 已处理缓冲区
    const struct vb2_mem_ops *mem_ops;  // 内存操作函数集
    const struct vb2_ops *ops;  // 缓冲区操作函数集
    void *drv_priv;                     // 驱动私有数据
    // ... 其他字段
  }
  ```

- 核心操作流程
  - 队列初始化：驱动填充 `vb2_queue` 结构体并调用 `vb2_queue_init()`（在驱动 probe 或 open 函数中初始化队列）；
  - 缓冲区申请：用户空间通过 `VIDIOC_REQBUFS` 触发，最终调用驱动的 `buf_init()` 等回调；
  - 缓冲区入队列：`QBUF`，用户将缓冲区交给驱动准备填充数据，对应 `vb2_qbuf()`；
  - 启动流：`VIDIOC_STREAMON` 触发 `vb2_streamon()`，驱动开始向缓冲区填充数据；
  - 缓冲区出队列：`DQBUF`，用户获取已填充数据的缓冲区，对应 `vb2_dqbuf()`；
  - 停止流：`VIDIOC_SREAMOFF` 触发 `vb2_streamoff()`；

- 流控制状态机
  `Videobuf2` 内部维护了一个严谨的状态机，确保流控制的有序性。

  1. 状态机
  - [ ] 补充状态机对应的状态转换图

    涉及的状态：
    1. `DEQUEUED`：缓冲区在用户空间，未提交给驱动；
    2. `PREPARED`：已调用 `buf_prepare`，缓冲区就绪；
    3. `QUEUED`：已提交给驱动，等待硬件处理；
    4. `ACTIVE`：硬件正在处理该缓冲区；
    5. `DONE`：硬件处理完成，数据就绪，等待用户取出；
    6. `ERROR`：处理过程中发生错误；

  2. 流控制序列
    `REQBUFS -> QUERYBUF -> QBUF(多个) -> STREAMON -> DQBUF/QBUF -> STREAMOFF`

  3. **驱动必须正确处理 `start_streaming` 和 `stop_streaming` 的调用，确保硬件状态与缓冲区状态的同步**。在 `stop_streaming` 中应将所有缓冲区状态重置为 `DEQUEUED`。


## 调用链
### 涉及的 ops 结构体
1. `ioctl` 操作
    ```c
    // 操作集数据结构
    ```

2. `vb2` 操作
   ```c
   // struct vb2_mem_ops: 用于操作内存
   // struct vb2_ops: 用于操作缓存区
   ```

### 调用流程

## 操作集实现
### ioctl 操作集
1. 操作流程概述
    典型的 `V4L2` **应用初始化流程**（**查询 -> 枚举 -> 获取 -> 设置**）：先查询设备能力，再枚举支持的格式，然后获取当前格式（或直接设置新格式）；

    **应用层（`ioctl` 调用） -> `V4L2` 核心（验证与路由） -> 驱动层（驱动回调）-> 硬件（寄存器设置）**

2. ioctl 支持的操作（相关宏）
    - `VIDIOC_QUERYCAP`：查询设备能力，如是否支持视频输入/输出、驱动信息等；
    - `VIDIOC_G_FMT`：获取当前视频格式、分辨率、像素格式等；
    - `VIDIOC_S_FMT`：设置...；
    - `VIDIOC_ENUM_FMT`：枚举支持的图像格式；
    - `VIDIOC_REQBUFS`：申请缓冲区（用于流格式 I/O 传输）；
    - `VIDIOC_QUERYBUFS`：查询缓冲区信息，如内存地址、大小、状态等；
    - `VIDIOC_QBUF`：将缓冲区放入队列，用于数据采集；
    - `VIDIOC_DQBUF`：从队列中取出缓冲区，以获取采集的数据；
    - `VIDIOC_STREAMON`：启动视频流，开始采集数据；
    - `VIDIOC_STREAMOFF`：停止视频流，停止采集数据；
    - `VIDIOC_DEFAULT`：处理未定义的 `ioctl` 请求，默认操作；

3. `VIDIOC_QUERYCAP`-查询设备能力
   准确填写驱动名、设备名、能力标志
   ```c
   struct v4l2_capability {
    __u8 driver[16];        // 驱动名称
    __u8 card[32];          // 设备名称
    __u8 bus_info[32];      // 总线信息
    __32 version;           // 内核版本
    __32 capabilities;      // 设备能力标志
    __32 device_caps;       // 设备特定能力
    __32 reserved[3];
   }

   // 关键能力标志-capabilities
   V4L2_CAP_VIDEO_CAPTURE       // 支持视频捕获
   V4L2_CAP_STREAMING           // 支持流式 I/O （mmap）
   V4L2_CAP_READWRITE           // 支持 read()/write I/O
   ```
   驱动实现示例：
   ```c
   static int my_v4l2_querycap(struct file *file, void *priv, struct v4l2_capability *cap) {
    struct my_device *dev = video_drvdata(file);

    strscpy(cap->driver, "my_camera_driver", sizeof(cap->driver));
    strscpy(cap->card, "My USB Camera", sizeof(cap->card));
    snprintf(cap->buf_info, sizeof(cap->bus_info), "usb-%s", dev->udev->bus->bus_name);

    cap->device_caps = V4L2_CAP_VIDEO_CAPTURE | V4L2_CAP_STREAMING | V4L2_CAP_READWRITE;
    cap->capabilities = cap->device_caps | V4L2_CAP_DEVICE_CAPS;

    return 0;
   }
   ```
   必须正确填充的字段（用户空间工具，如 `v4l2_ctl` 依赖这些信息显示设备）:
    - `driver`;
    - `card`;
    - `bus_info`;

4. `VIDIOC_ENUM_FMT`-枚举图像格式
   枚举设置支持的图像格式（如 `YUYV, MJPG, RGB24` 等），按 `index` 顺序返回，描述清晰；
   ```c
   struct v4l2_fmtdesc {
    __u32 index;            // 格式索引，从 0 开始
    __u32 type;             // 缓冲区类型，如 V4L2_BUF_TYPE_VIDEO_CAPTURE
    __u32 flags;            // 格式标志
    __u8 description[32];   // 格式描述字符串
    __u32 pixelformat;      // 四字符码（FOURCC）
    __u32 reserved[4];
   }
   ```
   驱动实现示例：
   ```c
   static int my_v4l2_enum_fmt(struct file *file, void *priv, struct v4l2_fmtdesc *f) {
    // 只支持视频捕获类型的枚举
    if (f->type != V4L2_BUF_TYPE_VIDEO_CAPTURE) {
        return -EINVAL;
    }

    switch (f->index) {
        case 0:
            f->pixelformat = V4L2_PIX_FMT_YUYV;
            strscpy(f->description, "YUYV 4:2:2", sizeof(f->description));
            break;
        case 1:
            f->pixelformat = V4L2_PIX_FMT_MJPEG;
            strscpy(f->description, "Motion-JPEG", sizeof(f->description));
            break;
        case 2:
            f->pixelformat = V4L2_PIX_FMT_RGB24;
            strscpy(f->description, "RGB 24-bit", sizeof(f->description));
            break;
        default:
            return -EINVAL; // 超出索引范围
    }

    f->flags = 0;
    return 0;
   }
   ```
   **应用会循环调用此 `ioctl`，递增 `index` 直到返回错误，从而获取所有支持的格式**；

5. `VIDIOC_G_FMT, VIDIOC_S_FMT`-获取/设置格式
   获取时，从设备状态返回真实参数；设置时，检查、调整参数，最后配置硬件；
    ```c
    struct v4l2_format {
        __u32 type;                     // 缓冲区类型
        union {
            struct v4l2_pix_format pix; // 用于视频捕获/输出
            // ...，其他格式，如 overlay,vbi 等
        } fmt;
    };

    struct v4l2_pix_format {
        __u32 width;
        __u32 height;
        __u32 pixelformat;          // 像素格式（FOURCC）
        __u32 field;                // 场序，如 V4L2_FIELD_NONE
        __u32 bytesperline;         // 每行字节数，包含填充
        __u32 sizeimgae;            // 图像缓冲区大小
        __u32 colorspace;           // 色彩空间
        // ... 其他字段
    };
    ```
    驱动实现示例：
    ```c
    static int my_v4l2_g_fmt(struct file *file, void *priv, struct v4l2_format *f) {
        struct my_device *dev = video_drvdata(file);

        if (f->type != V4L2_BUF_TYPE_VIDEO_CAPTURE) {
            return -EINVAL;
        }

        f->fmt.pix.width        = dev->width;
        f->fmt.pix.height       = dev->height;
        f->fmt.pix.pixelformat  = dev->pixelformat;
        f->fmt.pix.field        = V4L2_FIELD_NONE;
        f->fmt.pix.bytesperline = dev->bytesperline;
        f->fmt.pix.sizeimgage   = dev->imagesize;
        f->fmt.pix.colorspace   = V4L2_COLORSPACE_SRGB;

        return 0;
    }

    static int my_v4l2_s_fmt(struct file *file, void *priv, struct v4l2_format *f) {
        struct my_device *dev = video_drvdata(file);
        struct v4l2_pix_format *pix = &f->fmt.pix;

        // 检查缓冲区类型
        if (f->type != V4L2_BUF_TYPE_VIDEO_CAPTURE) {
            return -EINVAL;
        }

        // 检查是否支持该像素格式
        if (pix->pixelformat != V4L2_PIX_FMT_YUYV && pix->pixelformat != V4L2_PIX_FMT_MJPEG) {
            return -EINVAL;
        }

        // 调整宽度和高度以满足硬件对齐要求（例如 16 像素对齐）
        pix->width  = ALIGN(pix->width, 16);
        pix->height = ALIGN(pix->height, 2);

        // 计算 bytesperline 和 sizeimage
        if (pix->pixelformat == V4L2_PIX_FMT_YUYV) {
            pix->bytesperline = pix->width * 2; // YUYV 每个像素 2 字节
            pix->sizeimage = pix->bytesperline * pix->height;
        } else if (pix->pixelformat == V4L2_PIX_FMT_MJPEG) {
            pix->bytesperline = 0;      // MJPEG 是压缩格式，行跨度无意义
            pix->sizeimage = pix->width * pix->height * 3 / 8;  // 估算
        }

        pix->field = V4L2_FIELD_NONE;
        pix->colorspace = V4L2_COLORSPACE_SRGB;

        // 保存到设置结构体
        dev->width  = pix->width;
        dev->height = pix->height;
        dev->pixelformat = pix->pixelformat;
        dev->bytesperline = pix->bytesperline;
        dev->imagesize = pix->sizeimage;

        // 配置硬件寄存器，伪代码
        // write_reg(dev, WIDTH_REG, dev->width);
        // write_reg(dev, HEIGHT_REG, dev->height);
        // write_reg(dev, FORMAT_REG, dev->pixelformat);

        return 0;
    }
    ```
    要点：
    1. `VIDIOC_G_FMT`：返回当前设置的格式，驱动需从内部状态中填充结构体；
    2. `VIDIOC_S_FMT`：应用请求设置格式，驱动需检查并调整参数（如对齐要求），然后配置硬件；
    3. **驱动可以修改应用请求的参数，如将宽度对齐到 16 像素，但必须返回实际设置的格式**；
    补充：
    4. `ioctl` 的 `VIDIOC_TRY_FMT` 与 `VIDIOC_S_FMT` 类似，但是只测试格式而不实际配置硬件，常用于应用预览可用格式；

6. 建议：
   - 始终验证 `v4l2_format.type`（总体格式） 或 `v4l2_fmtdesc.type`（像素格式）字段，确保应用请求的是正确类型；
   - 在 `VIDIOC_S_FMT` 中，如果硬件不支持请求的分辨率，应将其调整为最接近的可用值；
   - 计算 `bytesperline` 和 `sizeimage` 时要考虑硬件对齐要求，避免缓冲区溢出；
   - 实现 `VIDIOC_ENUM_FMT` 时，确保索引从 0 开始连续，直到所有格式枚举完毕；

### vb2 操作集
`struct vb2_queue` 结构体涉及两种操作集：`struct vb2_mem_ops`（用于操作内存）和 `struct vb2_ops`（用于操作缓冲区）。

1. 与 `struct vb2_queue` 相关的 `ioctl` 操作：
    - `REQBUFS`：申请缓冲区；
    - `QBUF`：缓冲区入队列；
    - `DQBUF`：缓冲区出队列；
    - `STREAMON`：启动数据采集；

2. `struct vb2_ops` 是驱动开发者必须实现的一组回调函数，`Videobuf2` 框架通过它来与具体的硬件交互。
   - `queue_setup`：设置缓冲区数量、大小和内存类型；调用时机：流开启前；
   - `buf_init/buf_cleanup`：初始化/清理缓冲区私有数据；调用时机：缓冲区分配/释放时；
   - `buf_prepare/buf_finish`：准备/完成缓冲区（同步缓存、元数据）；调用时机：入队前/出队后；
   - `buf_queue`：将缓冲区加入硬件队列；调用时机：用户空间 `QBUF` 时；
   - `start_streaming/stop_streaming`：启动/停止硬件流；调用时机：流开启/停止时；
   - `wait_prepare/wait_finish`：释放/重新获取锁（避免死锁）；调用时机：等待事件前/后；

3. 驱动实现示例：
   ```c
    static int my_queue_setup(struct vb2_queue *vq, unsigned int *num_buffers, unsigned int *num_planes, unsigned int sizes[], struct device *alloc_devs[]) {
        // 计算缓冲区大小，设置 num_planes（平面数）
        size[0] = image_width * image_height * 2;   // 例如 YUV422
        *num_planes = 1;

        if (*num_buffers < 3) {
            *num_buffers = 3;       // 建议最小缓冲区数
        }

        return 0;
    }

    static const struct vb2_ops my_vb2_ops = {
        .queue_setup    = my_queue_setup,          // 设置缓冲区数量和大小
        .buf_init       = my_buf_init,             // 缓冲区初始化
        .buf_prepare    = my_buf_prepare,          // 缓冲区准备
        .buf_finish     = my_buf_finish,           // 缓冲区处理完成
        .buf_cleanup    = my_buf_cleanup,          // 缓冲区清理
        .buf_queue      = my_buf_queue,             // 缓冲区入队到驱动内部队列
        .start_streaming  = my_start_streaming,        // 启动流
        .stop_streaming   = my_stop_streaming,      // 停止流
        .wait_prepare   = vb2_ops_wait_prepare,
        .wait_finish    = vb2_ops_wait_finish,
    };

    // 在驱动 probe 或 open 函数中初始化队列
    static int my_video_open(struct file *file) {
        struct my_device *dev = video_drvdata(file);
        struct vb2_queue *q = &dev->vb2_q;

        q->type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        q->io_modes = VB2_MMAP | VB2_DMABUF;    // 支持的 I/O 模式
        q->drv_priv = dev;
        q->buf_struct_size = sizeof(struct my_buf);
        q->ops = &my_vb2_ops;
        q->mem_ops = &vb2_dma_contig_memops;    // 使用 DMA 连续内存操作
        q->timestamp_flags = V4L2_BUF_FLAG_TIMESTAMP_MONOTONIC;
        q->lock = &dev->lock;

        return vb2_queue_init(q);
   }
   ```

4. 最佳实践与调试技巧
   - checklist
     - 正确实现 `vb2_ops` 所有必需的回调函数；
     - 根据硬件能力选择合适的内存模型（通常 `MMAP` 或 `DMABUF`）；
     - 在 `buf_prepare` 中验证缓冲区大小，避免溢出；
     - 在 `start_streaming` 中启动硬件，并处理可能发生的错误；
     - 在 `stop_streaming` 中停止硬件，并返回所有缓冲区；
     - 使用 `vb2_buffer_done()` 正确设置缓冲区状态（`VB2_BUF_STATE_DONE` 或 `VB2_BUF_STATE_ERROR`）；
   - 调试工具
     - 查看 `videobuf2` 调试信息
        ```bash
        # 需要内核配置 CONFIG_VIDEOBUF2_DEBUG
        echo 0x1f > /sys/module/videobuf2_core/parameters/debug
        ```
     - 使用 `v4l2-ctl` 工具测试流
        ```bash
        v4l2-ctl --device /dev/video0 --set-fmt-video=width=1920,height=1080,pixelformat=YUYV
        v4l2-ctl --device /dev/video0 --stream-mmap --stream-count=100 --stream-to=test.raw
        ```
    - 常见问题与解决方案
      ![](../../../img/subsys-v4l2/260302-001742.png)

## 图像格式与色彩空间
### 像素格式
在 `V4L2` 子系统中，像素格式（`Pixel Format`）定义了图像数据在内存中的存储方式。**它是驱动与应用程序之间数据交换的“协议”**。`V4L2` 使用 `struct v4l2_fmtdesc` 和 `struct v4l2_format` 来描述和协商格式。
```c
struct v4l2_fmtdesc {
    __u32 index;            // 格式索引，从 0 开始
    __u32 type;             // 缓冲区类型，如 V4L2_BUF_TYPE_VIDEO_CAPTURE
    __u32 flags;            // 格式标志
    __u8 description[32];   // 格式描述字符串
    __u32 pixelformat;      // 四字符码（FOURCC）
    __u32 reserved[4];
}

struct v4l2_format {
    __u32 type;                     // 缓冲区类型
    union {
        struct v4l2_pix_format pix; // 图像格式，用于视频捕获/输出
        struct v4l2_window win;
        struct v4l2_vbi_format vbi;
        struct v4l2_sliced_vbi_format sliced;
        __u8 raw_data[200];
    } fmt;
};
```

- 常见 `FourCC` 码
| FourCC 码           | 格式名称                       | 描述 | 典型应用 |
| ------------ | ------------------------ | -------- | -------- |
| `YUYC` | `YUYV 4:2:2`   | 打包的 `YUV` 格式，Y 和 UV 分量交错存储 | 摄像头采集、视频流 |
| `NV12` | `YUY 4:2:0` 半平面 | Y 平面 + 交错的 UV 平面 | 硬件编解码、显示 |
| `RGB24` | `RGB 8:8:8` | 每个像素 24 位，R，G，B 各 8 位 | 图像处理、显示 |
| `MJPG`  | `Motion-JPEG` | 压缩的 `JPEG` 图像序列 | 网络摄像头、存储 |
| `H264`  | `H.264/AVC`   | 高级视频编码 | 高清视频录制 |

1. `YUV` 格式解析
   `YUV` 是一种颜色编码系统，将亮度信息（Y）与色度信息（UV）分离。这种分离使得 `YUV` 格式在视频压缩和传输中具有优势。
   - `YUV 4:2:2 (YUYV/YUY2)`: 每两个水平相邻的像素共享一组 UV 分量。内存布局示例：像素 1-`Y0 U0 Y1 V0`，像素 2-`Y2 U1 Y3 V1`，每个宏像素（2 个像素）占用 4 字节；
   - `YUV 4:2:0 (NV12/NV21)`: 每 `2x2` 的 Y 像素块共享一组 UV 分量， 数据量仅为 `RGB` 的 50％，UV 分量存储在一个单独的平面中交错排列。示例：`[Y 平面]: Y00 Y01 Y02 Y03 ... Y10 Y11 Y12 Y13 ...` `[UV 平面] U00 V00 U01 V01 ...`；常用于硬件加速；
   - `YUV 4:4:4`: 每个像素都有独立的 Y、U、V 分量，无下采样，质量最高但数据量最大。示例：像素 1-`Y0 U0 V0`，像素 2-`Y1 U1 V1`；主要用于专业图像处理；

2. `RGB` 格式解析
   `RGB` (红绿蓝)是加色模型，通过不同强度的红、绿、蓝光组合来呈现颜色。在数字图像中，`RGB` 通常以每个分量 8 位（24 位色）或更多位存储。
    | 格式名称           | 内存排列       | 字节/像素 | 特点 |
    | ------------ | ------------------------ | -------- | -------- |
    | `RGB24(RGB888)` | `R0 G0 B0 R1 G1 B1 ...` | 3 | 标准格式，兼容性好 |
    | `BGR24(BGR888)` | `B0 G0 R0 B1 G1 R1 ...` | 3 | OpenCV 默认格式 |
    | `RGB32(RGB8888)` | `R0 G0 B0 A0 R1 G1 B1 A1 ...` | 4 | 带 `Alpha` 通道或填充字节 |
    | `RGB565`  | `R[4:0] G[5:0] B[4:0](16位)` | 2 | 嵌入式系统常用，节省内存 |
    
    注：`RGB` 格式的内存排列与字节序（`Endianness`）密切相关。在大端系统和小端系统中，多字节像素值（如 `RGB565`）的存储顺序可能不同。

### 色彩空间转换
在实际应用中，经常需要在 `YUV` 与 `RGB` 之间进行转换，转换公式基于 `ITU-R` 标准（如 `BT.601`, `BT.709`）。

1. `YUV` 到 `RGB` 的转换-`BT.601` 标准
   ```bash
   # YUV (BT.601) 转 RGB24 公式
   # 假设 Y、U、V 范围均为 0-255
   R = 1.164 * (Y - 16) + 1.596 * (V - 128);
   G = 1.164 * (Y - 16) - 0.813 * (V - 128) - 0.391 * (U - 128);
   B = 1.164 * (Y - 16) + 2.018 * (U - 128);

   # 需要钳位到 [0, 255] 范围
   R = clamp(R, 0, 255);
   G = clamp(G, 0, 255);
   B = clamp(B, 0, 255);
   ```

2. `RGB` 到 `YUV` 的转换-`BT.601` 标准
   ```bash
   # RGB24 转 YUV (BT.601) 公式
   Y = 0.257 * R + 0.504 * G + 0.098 * B + 16;
   U = -0.148 * R - 0.291 * G + 0.439 * B + 128;
   V = 0.439 * R - 0.368 * G - 0.071 * B + 128;
   ```

3. 转换优化技巧
    - 整数运算：使用定点数运算避免浮点开销，例如将系数乘以 256 后使用整数乘除；
    - 查表法：对有限的宿儒（如 0~255）预先计算转换表，用空间换时间；
    - `SIMD` 指令：使用 `SSE/NEON` 等 `SIMD` 指令集并行处理多个像素；
    - 硬件加速：利用 `GPU` 或专用的图像处理单元 `IPU` 进行转换；

### 应用示例
示例展示如何在 `V4L2` 应用程序中查询和设置像素格式：
```c
#include <linux/videodev2.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <stdio.h>
#include <string.h>

int main() {
    int fd = open("/dev/video0", O_RDWR);
    if (fd < 0) {
        perror("打开设备失败");
        return -1;
    }

    // 1. 查询设备支持的格式
    struct v4l2_fmtdesc fmtdesc;
    memeset(&fmtdesc, 0, sizeof(fmtdesc));
    fmtdesc.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;

    printf("设备支持的格式：\n");
    for (fmtdesc.index = 0; ioctl(fd, VIDIOC_ENUM_FMT, &fmtdesc) == 0; fmtdesc.index++) {
        printf("%d: %s (FourCC: %.4s)\n", fmtdesc.index, fmtdesc.description, (char *)&fmtdesc.pixelformat);
    }

    // 2. 获取当前格式
    struct v4l2_format fmt;
    memset(&fmt, 0, sizeof(fmt));
    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;

    if (ioctl(fd, VIDIOC_G_FMT, &fmt) == 0) {
        printf("\n当前格式：%.4s，宽度：%d，高度：%d\n", (char *)&fmt.fmt.pix.pixelformat, fmt.fmt.pix.width, fmt.fmt.pix.height);
    }

    // 3. 尝试设置新格式（NV12）
    fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_NV12;
    fmt.fmt.pix.width = 1920;
    fmt.fmt.pix.height = 1080;
    fmt.fmt.pix.field = V4L2_FIELD_NONE;

    if (ioctl(fd, VIDIOC_S_FMT, &fmt) == 0) {
        printf("格式设置成功！\n");
    } else {
        printf("格式设置失败，可能设备不支持！\n");
    }

    close(fd);
    return 0;
}
```