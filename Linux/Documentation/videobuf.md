# videobuf 文档阅读笔记

## 参考
1. `Documentation/video4linux/videobuf.txt`
2. Developers who are interested in more information can go into the relevant header files. there are a few low-level functions declared there which have not been talked about here. Also worthwhile is the vivi driver (drivers/media/platform/vivi.c), which is maintained as an example of how V4L2 drivers should be written. Vivi only uses the vmalloc() API, but it's good enough to get started with. Note also that all of these calls are exported GPL-only, so they will not be available to non-GPL kernel modules.

## 概述
The videobuf layer functions as a sort of glue(粘合剂) layer between **a V4L2 driver and user space**. It handles the **allocation and management of buffers for the storage of video frames**. 
   - There is a set of functions which can be used to implement many of the standard POSIX I/O system calls, including read(), poll(), and, happily, mmap().
   - Another set of functions can be used to implement the bulk of the V4L2 ioctl() calls related to streaming I/O, including buffer allocation, queueing and dequeueing, and streaming control.

Using videobuf imposes(强制) a few design decision on the driver author, but the payback comes in the form of reduced code in the driver and a consistent implementation of the V4L2 user-space API.

1. 功能：提供一系列函数，用于 V4L2 驱动和用户空间的交互。提供视频帧缓冲区的分配和管理的功能。这些函数涉及：
   - POSIX I/O 系统调用的实现，例如 read()、poll() 以及 mmap() 等；
   - V4L2 涉及 streaming I/O 的 ioctl() 的实现，包括：缓冲区分配、入队、出队以及 streaming 控制；
2. 减少了驱动实现的差异性、增强了 V4L2 用户空间 API 实现的一致性；

## buffer 类型
Not all video devices use the same kind of buffers. In fact, there are (at least) three common variations:

1. 缓冲区在物理上以及（内核）虚拟地址空间上都是分散的（scattered）。（几乎）所有的用户空间缓冲区都是如此，对于内核使用同样的方式来分配缓冲区也是有意义的。但是分散的缓冲区并非总是可取的，因为这需要硬件提供 scatter/gather DMA 操作的支持。
2. 缓冲区在物理上分散（scattered），在虚拟地址空间上连续。或者说通过调用 `vmalloc()` 得到的缓冲区。这种缓冲区要使用 DMA 操作是很困难的，它们适合不需要使用 DMA 但是要求虚拟地址连续的缓冲区。
3. 缓冲区在物理上是连续的。Allocation of this kind of buffer can be unreliable on fragmented systems, but simpler DMA controllers cannot deal with anything else.

Videobuf can work with all three types of buffers, but the driver author must pick one at the outset and design the driver around that decision.

4. 第四种缓冲区："overlay"，位于系统的视频内存，但是几乎已被弃用。

## 数据结构、回调以及初始化
取决于所使用的缓冲区类型，驱动需要包含如下之一的头文件：
```
<media/videobuf-dam-sg.h>       // 物理上分散的缓冲区
<media/videobuf-vmalloc.h>      // 使用 vmalloc() 得到的缓冲区
<media/videobuf-dma-contig.h>   // 物理上连续的缓冲区
```
驱动中用于描述 V4L2 device 的数据结构应当包含一个 
   - **`struct videobuf_queue`** 的实例，以用于管理缓冲区队列（buffer queue）；
   - 同时包含一个 **`list_head`** 用于在该队列中可获取的缓冲区（available buffers）；
   - 此外，还需要一个中断安全（interrupt-safe）的 **spinlock**，用于保护此队列；

下一步就是实现 4 个简单的回调函数，来帮助 videobuf 进行缓冲区的管理：
```c
struct videobuf_queue_ops {
    int (*buf_setup)(struct videobuf_queue *q, unsigned int *count, unsigned int *size);
    int (*buf_prepare)(struct videobuf_queue *q, struct videobuf_buffer *vb, enum v4l2_field field);
    void (*buf_queue)(struct videobuf_queue *q, struct videobuf_buffer *vb);
    void (*buf_release)(struct videobuf_queue *q, struct videobuf_buffer *vb);
}
```
   - `buf_setup`：在早于 I/O 处理时被调用，当 streaming 已完成初始化，它的目的是告诉 videobuf 关于 I/O stream 的一些信息。
     - `count`：作为建议，表示使用的缓冲区数量。驱动应当检查此值并在需要的情况下做出修改（传出参数）。经验：缓冲区数量最少需要有 2 个，最大的缓冲区数量（不超过 32 个）对于设备而言也是具有意义的。
     - `size`：每一个帧数据的期望或者最大大小；
   - `buf_prepare`：每个缓冲区（以 `struct videobuf_buffer` 指针表示）将会被传递给 `buf_prepare()`，用以设置缓冲区的 size、width、height 以及 fields 等属性。如果缓冲区的 state 是 VIDEOBUF_NEEDS_INIT，驱动应当将其传递给：
        ```c
        int videobuf_iolock(struct videobuf_queue *q, struct video_buffer *vb, struct v4l2_framebuffer *fbuf);
        ```
        上述函数通常会给缓冲区分配空间。最后，`buf_prepare()` 函数应当将缓冲区的 state 设置为 VIDEOBUF_PREPARED。
   - `buf_queue`：当缓冲区入队时，它被传递给 `buf_queue()` 函数。`buf_queue()` 应当把该缓冲区放到驱动的 available buffers 列表并把它的状态设备为 VIDEOBUF_QUEUED。需要注意的是，该函数在调用之前需要先获取队列的自旋锁（spinlock），这可能会导致对应线程停在此处（不断轮询）（screeching halt）。还需要注意的是，videobuf 可能在等待队列中的第一个缓冲区，将其他缓冲区放至队列前面会导致错误，因此需要使用 `list_add_tail()` 来完成入队工作。
   - `buf_release`：当缓冲区不再需要使用的时候调用 `buf_release()`。驱动应当确保没有该缓冲区上的 I/O 操作，然后再把它传递给适当的释放函数：
        ```c
        // scatter/gather drivers
        int videobuf_dma_unmap(struct videobuf_queue *q, struct videobuf_dambuf *dma);
        int videobuf_dma_free(struct videobuf_dmabuf *dma);

        // vmalloc drivers
        void videobuf_vmalloc_free(struct videobuf_buffer *buf);

        // contiguous drivers
        void videobuf_dma_contig_free(struct videobuf_queue *q, struct videobuf_buffer *buf);
        ```
        一种确保缓冲区上没有 I/O 操作的方法是把它传递给如下函数：
        ```c
        // vb: buffer
        // non_blocking: 是否使用阻塞 I/O，在 buf_release() 下应当为 0
        // intr: 控制等待是否可被中断
        int videobuf_waiton(struct videobuf_buffer *vb, int non_blocking, int intr);
        ```
## 文件操作
### open
At this point, much of the work is done; much of the rest is slipping videobuf calls into implementation of the other driver callbacks. 第一步就是在 `open()` 函数中必须初始化 videobuf 队列，需要使用到的函数取决于所使用的缓冲区类型：
```c
void videobuf_queue_sg_init(struct videobuf_queue *q, 
                    struct videobuf_queue_ops *ops,
                    struct device *dev,
                    spinlock_t *irqlock,
                    enum v4l2_buf_type type,
                    enum v4l2_field field,
                    unsigned int msize,
                    void *priv);

void videobuf_queue_vmalloc_init(struct videobuf_queue *q,
                    struct videobuf_queue_ops *ops,
                    struct device *dev,
                    spinlock_t *irqlock,
                    enum v4l2_buf_type type,
                    enum v4l2_field field,
                    unsigned int msize,
                    void *priv);

void videobuf_queue_dma_contig_init(struct videobuf_queue *q,
                    struct videobuf_queue_ops *ops,
                    struct device *dev,
                    spinlock_t *irqlock,
                    enum v4l2_buf_type type,
                    enum v4l2_field field,
                    unsigned int msize,
                    void *priv);
```
上述三种情况下的参数都一样：
- `q`：queue structure for the device；
- `ops`：上述提到的回调函数集合；
- `dev`：is the device structure for this video device；
- `irqlock`：an interrupt-safe spinlock to protect access to the data structures；
- `type`：buffer type used by the device (cameras will use V4L2_BUF_TYPE_VIDEO_CAPTURE, for example)；
- `field`：describes which field is being captured (often V4L2_FIELD_NONE for progressive devices)；
- `msize`：is the size of any containing structure used around struct videobuf_buffer；
- `priv`：is a private data pointer which shows up in the priv_data field of struct videobuf_queue.

### read
V4L2 捕获（capture）驱动被实现为支持两种 API：`read()` 系统调用和一个相当复杂的 streaming 机制。为实现 `read()` 函数，驱动只需要调用如下函数中的一个：
```c
ssize_t videobuf_read_one(struct videobuf_queue *q, char __user *data, size_t count, loff_t *ppos, int nonblocking);

ssize_t videobuf_read_stream(struct videobuf_queue *q, char __user *data, size_t count, loff_t *ppos, int vbihack, int nonblocking);
```
上述函数都将读取到的帧数据到 data 中，并返回读到的数据量。不同的是，`videobuf_read_one()` 只会读取单帧，而 `videobuf_read_stream` 将会读取多个帧，以满足应用程序要求的 `count`。
一个典型的驱动 `read()` 实现是先开始捕获引擎（start the capture engine）、调用上述函数、在返回之前终止引擎（stop the engine）（一个更加明知的实现方式是保留 engine 运行一小段时间，以防止另一个很快的 `read()` 调用的发生）。

### poll
`poll()` 调用通常可以以一个对如下函数直接调用的方式实现：
```c
unsigned int videobuf_poll_stream(struct file *file, struct videobuf_queue *q, poll_table *wait);
```
注意：实际最终使用的等待队列（wait queue）是与第一个缓冲区（first available buffer）相关联的。

### mmap
当基于内核空间缓冲区的 stream I/O 已完成，驱动必须支持 `mmap()` 系统调用，以让用户空间能够获取到数据。在许多 V4L2 驱动程序中，经常复杂的 `mmap()` 实现被简化到一个单独的调用：
```c
int videobuf_mmap_mapper(struct videobuf_queue *q, struct vm_area_struct *vma);
```
其他的所有事情都由 videobuf 的代码处理。

### release
`release()` 函数要求两个分开的 videobuf 调用：
```c
void videobuf_stop(struct videobuf_queue *q);
int videobuf_mmap_free(struct videobuf_queue *q);
```
`videobuf_stop()` 终止所有在运行的 I/O，仍然依赖驱动 stop the capture engine。
`videobuf_mmap_free()` 将会确保所有的缓冲区都被 unmapped。如果正常处理完成，在这之后将会调用 `buf_release()` 回调函数。如果仍有处于 mapped 状态的缓冲区，`videobuf_mmap_free()` 会返回一个错误码。这样做的目的是显而易见的：让文件描述符的关闭操作失败，如果仍有缓冲区处于 mapped 状态。但是在内核版本 2.6.32 中的驱动程序忽略该函数的返回值。

### ioctl 操作
The V4L2 API includes a very long list of driver callbacks to respond to many ioctl() commands made available to user space. A number of these - those associated with streaming I/O - turn almost directly into videobuf calls. The relevant helper function are:
```c
int videobuf_reqbufs(struct videobuf_queue *q, struct v4l2_requestbuffers *req);
int videobuf_querybuf(struct videobuf_queue *q, struct v4l2_buffer *b);
int videobuf_qbuf(struct videobuf_queue *q, struct v4l2_buffer *b);
int videobuf_dqbuf(struct videobuf_queue *q, struct v4l2_buffer *b, int nonblocking);
int videobuf_streamon(struct videobuf_queue *q);
int videobuf_streamoff(struct videobuf_queue *q);
```
例如，一个 VIDIOC_REQBUFS 调用将会转接到调用驱动程序的 `vidioc_reqbufs()` 回调上。在该函数中，通常只需要定位到正确的 `struct videobuf_queue` 类型指针并将其传递给 `videobuf_reqbufs()` 函数即可。在大量的 V4L2 驱动中，这些支持函数可以取代大量的缓冲区管理样板。

`vidioc_streamon()` 和 `vidioc_streamoff` 函数会比较复杂一点，以为涉及到 starting and stopping the capture engine 的处理。

## Buffer allocation
scatter/gather 的 case 是最复杂的。
- 对于缓冲区分配而言，驱动可以把它完全交给 videobuf 层。在这种情况下，缓冲区将会以匿名的用户空间页来分配并且会很分散。
- 如果应用程序使用用户空间缓冲区，驱动不需要执行分配工作。videobuf 层将会调用 `get_user_pages()` 并且填充 scatterlist array。
- 如果驱动需要执行自己的内存分配，那么这应该在 `vidioc_reqbufs()` 函数中执行，并之后调用 `videobuf_reqbuf()`。第一步就是调用：
    ```c
    struct videobuf_dmabuf *videobuf_to_dma(struct videobuf_buffer *buf);
    ```
    返回的 `videobuf_dmabuf` 结构体（`media/videobuf-dma-sg.h`）包含一对相关的字段：
    ```
    struct scatterlist *sglist;
    int sglen;
    ```
    驱动程序必须分配一个合适大小的 scatterlist array 并让它指向分配好的缓冲区。sglen 应当被设置为 array 的长度。

Drivers using the vmalloc() method need not (and cannot) concern themselves with buffer allocation at all; videobuf will handle those details. The same is normally true of contiguous-DMA drivers as well; videobuf will allocate the buffers (with dma_alloc_coherent()) when it sees fit. That means that these drivers may be trying to do high-order allocations at any time, an operation which is not always guaranteed to work. Some drivers play tricks by allocating DMA space at system boot time; videobuf does not currently play well with those drivers.

在内核 2.6.31 版本，contiguous-DMA 驱动能够和用户提供的缓冲区正常工作，前提是该缓冲区在物理上是连续的。通常情况下用户空间分配的结果不能满足这一要求，但是从其他内核驱动程序上获取的缓冲区或者是在大页（huge pages）内获取的，能够和这些驱动程序工作。

## Filling the buffers
videobuf 实现的最后一部分没有直接的回调，它通常是将帧数据放置到缓冲区的代码的一部分，常在中断服务程序中进行。对于所有类型的驱动程序，这部分的处理过程大致如下：
    - 获取下一个 available 缓冲区，并且确认 somebody 正在等待它；
    - 获取一个内存指针并将视频数据放在那里；
    - 标记缓冲区为 done，同时唤醒等待它的进程。

步骤 1 是在由驱动管理的 `list_head` 结构体，即由 `buf_queue` 回调填充的队列中寻找。因为 starting the engine 和缓冲区入队是在不同的步骤中进行的，这可能导致 engine 正在运行但是没有任何的缓冲区 available，特别是在 `vmalloc()` 的情况下。因此驱动程序应当有列表为空的对应处理。类似地，可能 nobody 对缓冲区感兴趣，驱动程序不应该将其移除队列或者填充它，直到某一进程在等待它为止。That test can be done by examining the buffer's done flag (a wait_queue_head_t structure) with waitqueue_active().

在被 mapped 用于DMA 之前，缓冲区 state 应当被设置为 VIDEOBUF_ACTIVE，这保证了当设备在进行数据传输时 videobuf 层将不会对其进行任何操作。

对于 scatter/gather 驱动，目标内存指针可以从 scatterlist structure 中获取。使用 `vmalloc()` 函数的驱动程序，可以通过如下函数获取内存指针：
```c
void *videobuf_to_vmalloc(struct videobuf_buffer *buf);
```
如果是 contiguous DMA 驱动程序，可以使用如下函数：
```c
dma_addr_t videobuf_to_dma_contig(struct videobuf_buffer *buf);
```
contiguous DMA API 通过这种方式对驱动程序隐藏了 DMA 缓冲区的内核空间地址。

最后一步就是设置相关的 videobuf_buffer 结构体的 size 字段，设置为实际捕获图像的大小，并且将它的 state 设置为 VIDEOBUF_DONE，然后调用 `wake_up()` on the done queue。在这时，缓冲区的所有者是 videobuf 层，驱动不应该 touch it again。

Developers who are interested in more information can go into the relevant header files; there are a few low-level functions declared there which have not been talked about here. Also worthwhile is the vivid driver (drivers/media/platform/vivi.c), which is maintained as an example of how V4L2 drivers should be written. Vivi only uses the vmalloc() API, but it's good enough to get started with. Note also that all of these calls are exported GPL-only, so they will not be available to non-GPL kernel modules.
