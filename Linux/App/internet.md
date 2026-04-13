# 网络编程

## 基础概念
### 网络相关
1. 局域网和广域网
2. IP（Internet Protocol）：**本质是一个整型数，用于表示计算机在网络中的地址**。IP 协议版本有两个：
   - IPv4：
     - 使用一个 32 的整型数描述一个 IP 地址，4 个字节，int 型；
     - 也可以使用一个点分**十进制**字符串描述这个 IP 地址，分成 4 份，每份 1 字节，最大值为 255；
     - 可使用的 IP 地址共有 2^32 个；
   - IPv6：
     - 使用一个 128 位的整型数描述一个 IP 地址，16 字节；
     - 也可以使用一个字符串描述这个 IP 地址，分成 8 份，每份 2 字节，每份以 **16 进制**的方式表示；
     - 可以使用的 IP 地址共有 2^128 个；
   ```bash
   # 查看 ip 地址
   # linux
   ifconfig

   # windows
   ipconfig

   # 特殊 ip 地址：127.0.0.1 -> 和本地的 IP 地址是等价的
   ```
3. 端口：**端口的作用是定位到主机上的某一个进程**，通过这个端口，进程就可以接收到对应的网络数据了。
   - 端口也是一个整型数（unsigned short），一个 16 位整型数，**有效端口的取值范围是：0~65535**；
   - **通多 IP 地址可以定位到某一台主机，通过端口就可以定位到主机上的某一个进程**；
   - 计算机中，并不是所有的进程都需要关联一个端口（如果这个进程不需要网络通信，那么该进程就不需要绑定端口）；
   - 一个端口只能给某一个进程使用，多个进程不能同时使用同一个端口；

### 数据传输三要素
- 源地址；
- 目的地址；
- 数据传输长度；

在网络传输中使用“IP 和端口”来表示源或目的。

在网络传输中，涉及两个对象：server 和 client，即 server （服务器）程序和 client （客户端）程序。后者主动发起请求，前者被动地响应请求。

## 字节序
> 字节序，顾名思义字节的顺序，就是大于一个字节类型的数据在内存中的存放顺序，也就是说对于单个字符来说是没有字节序问题的，字符串是单字符的集合，**因此字符串也没有字节序问题**。

字节序问题只出现在：**一个多字节数值内部**的字节如何排列。

字符串的每个字符是独立编码端元，字符本身不涉及多字节数值的字节序问题。字符在字节流中的排列顺序就是其逻辑顺序，不存在因字节序不同而产生的歧义解释。字符串只有：字符 A 在字符 B 前面，这种逻辑顺序。

目前，在各种体系的计算机中通常采用的字节存储机制主要有两种：`Big-Endian` 和 `Little-Endian`。

### 小端-`Little-Endian`
- 主机字节序；
- 数据的**低位字节**存储到内存的**低地址位**，数据的**高位字节**存储到内存的**高位地址**；
- PC 机，数据的存储默认使用的是小端；

### 大端-`Big-Endian`
- 网络字节序；
- 数据的**低位字节**存储到内存的**高位地址**，数据的**高位字节**存储到内存的**低地址位**；
- **套接字通信过程中操作的数据都是大端存储的，包括接收、发送的数据、IP 地址、端口**。

### 转换接口函数
```c
// 网络通信中 IP 和端口的转换
// 主机字节序到网络字节序
uint16_t htons(uint16_t hostshort);
uint32_t htonl(uint32_t hostlong);

// 网路字节序到主机字节序
uint16_t ntohs(uint16_t netshort);
uint32_t ntohl(uint32_t netlong);

// ip 地址转换
// 主机字节序的 IP 地址转换位网络字节序
// 主机字节序的 IP 地址为字符串，网络字节序 IP 地址是整型
/* 
 * @af: 地址族，AF_INET（ipv4 格式的 ip 地址）、AF_INET6（ipv6 格式的 ip 地址）；
 * @src: 传入参数，对应要转换的点分十进制的 ip 地址，如 192.168.1.100
 * @dst: 传出参数，函数调用完成，转换得到的大端整型 ip 被写入到这块内存中
 * 返回值：成功返回 1，失败返回 0 或者 -1
 */
int inet_pton(int af, const char *src, void *dst);

// 将大端的整型数，转换为小端的点分十进制的 ip 地址
#include <arpa/inet.h>
/* 
 * @af: 同上；
 * @src: 传入参数，指向内存中存储了大端的整型 ip 地址；
 * @dst: 传出参数，存储转换后得到的小端的点分十进制的 ip 地址；
 * @size: 修饰 dst 参数的，标记 dst 指向的内存中最多可以存储多少个字节
 * 返回值：成功，指向第三个参数对应的内存地址；失败，NULL
 */
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);

// 只能用于处理 ipv4 地址大小端的转换的一组函数
// 点分十进制 IP -> 大端整型
in_addr_t inet_addr(const char *cp);
// 大端整型 -> 点分十进制 IP
char *inet_ntoa(struct in_addr in);
```

## 分层架构
![](../../img/internet/-%20[%20]%20OSI（Open%20System%20Interconnect），补充分层架构信息.png)
> [上述图片来源](https://subingwen.cn/linux/socket/#1-%E6%A6%82%E5%BF%B5)

1. 应用层：
   - 直接为应用进程（如电子邮件、文件传输等）提供服务；
   - 应用层涉及的协议很多，比如 HTTP（网络应用）、SMTP（电子邮件）、FTP（文件传送）以及 DNS、POP3、SNMP、Telnet 等；
2. 表示层：
   - 对应用层数据编码和转化，确保以一个系统应用层发送的信息可以被另一个系统应用层识别；
3. 会话层：
   - 建立、管理和终止表示层与实体之间的通信会话；
   - 建立一个连接（自动的收集信息、自动的网络寻址）；
4. 传输层：
   - 向高层提供可靠的端到端的网络数据流服务；
   - 每个应用程序都会在网卡注册一个端口号，该层就是端口与端口的通信；
   - 基于传输层协议编写应用程序，应用程序位于“应用层”；
   - 传输层协议可以选择 TCP 协议，也可以选择 UDP 协议；
5. 网络层：
   - 负责在源点和终点之间建立连接；
   - 此处需要确定计算机的位置，通过 IPv4，IPv6 格式的 IP 地址来找到对应的主机；
6. （数据）链路层：
   - 规定了 0 和 1 的分包形式，确定了网络数据包的形式等；
7. 物理层：
   - 负责最后将信息编码成电流脉冲或其他信号用于网上传输；
   - 在物理层上所传数据的单位是比特，物理层的任务就是透明地传送比特流。

## 通信操作流程
![](../../img/internet/-%20[%20]%20补充数据封装.png)
> [上述图片来源](https://subingwen.cn/linux/socket/#2-%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE)

- 在网络通信的时候，程序员需要负责应用层数据的处理（最上层）：
  - 应用层的数据可以使用某些协议进行封装，也可以不封装；
  - 调用发送数据的接口函数，将数据发送出去；
  - 接口函数做底层数据处理：
    - 传输层使用传输层协议打包数据；
    - 网络层使用网络协议打包数据；
    - 网络接口层使用网络接口层协议打包数据；
    - 数据被发送到 Internet

- 程序员调用接收数据的接口函数获取数据，如果应用层也使用了协议对数据进行了封装，数据包的解析需要在应用层中执行。

网络协议指的是计算机网络中互相通信的对等实体之间交换信息时所必须遵守的规则的集合。一般系统网络协议包括五分部分：通信环境，传输服务，词汇表，信息的编码格式，时序、规则和过程。

## 网络接口层协议：IP 协议和以太网协议
- [ ] 补充相关内容

## 传输层协议：TCP 和 UDP
### TCP
#### 概述
向应用程序提供面向连接地服务。

- 特点
  - **面向连接**：是一个双向连接，通过三次握手完成，断开连接需要通过四次挥手完成；
  - **安全**：tcp 通信过程中，会对发送的每一数据包都进行校验，如果发现数据丢失，会自动重传；
  - **流式传输**：发送端和接收端处理数据的速度，数据的量都可以不一致；

- 功能
  - 可靠传输（重传机制）、流量控制（即发送方/接收方速率匹配）；
  - 应用层报文划分为短报文，并提供拥塞控制机制；

- 通信交互流程
![](../../img/internet/260413-222435.png)
> 图片来源：[1]

#### 文件描述符相关
1. 服务器端的两类文件描述符
  - 监听的文件描述符：
    - 只需要有一个；
    - 不负责和客户端通信，负责检测客户端的连接请求，检测到后调用 `accept()` 就可以建立新的连接；
  - 通信的文件描述符：
    - 负责和建立连接的客户端通信；
    - 如果有 N 个客户端和服务器建立了新的连接，通信的文件描述符就有 N 个，每个客户端和服务器都对应一个通信的文件描述符；

2. 文件描述符对应的内存结构：
  - **一个文件描述符对应两块内存，一块内存是读缓冲区，一块内存是写缓冲区**；
  - 监听的文件描述符：
    - 客户端的连接请求会发送到服务器端监听的文件描述符的读缓冲区中；
    - 读缓冲区中有数据，说明有新的客户端连接；
    - 调用 `accept()` 函数，这个函数会检测监听文件描述符的读缓冲区。检测不到数据，该函数阻塞；如果检测到数据，接触阻塞，建立新的连接；
  - 通信的文件描述符：
    - 客户端和服务器端都有通信的文件描述符；
    - 发送数据（调用 `write()/send()`），数据进入到内核中。数据并没有被发送出去，而是将数据写入到了通信的文件描述符对应的写缓冲区中。内核检测到通信的文件描述符写缓冲区中有数据，内核会将数据发送到网络中；
    - 接收数据（调用 `read()/recv()`），从内核读取数据。数据如何进入到内核程序员不需要处理，数据进入到通信的文件描述符的读缓冲区中。数据进入到内核，必须使用通信的文件描述符，将数据从读缓冲区中读出即可。

#### tcp 协议中的重要字段：
  ![](../../img/internet/260413-222838.png)
    > [图片来源](https://subingwen.cn/linux/three-four/)
   - 源端口：表示发送端端口号，字段长 16 位，2 个字节；
   - 目的端口：表示接收端端口号，字段长 16 位，2 个字节；
   - 序号：字段长 32 位，4 个字节，序号范围为 0~2^32 - 1：
     - 由于 tcp 是面向字节流的，在一个tcp 连接中传送的字节流中的每个字节都按顺序编号；
     - 首部中的序号字段则是指本报文所发送的数据的第一个字节的序号，这是随机生成的；
     - 序号循环使用，当序号增加到最大值时，下一个序号就又回到了 0；
   - 确认序号：字段长 32 位，4 字节。表示收到的下一个报文段的第一个数据字节的序号，如果确认序号为 N，序号为 S，则表明到序号 N-S 为止的所有数据？？？
   - 8 个标志位：
      - CWR:
      - ECE:
      - URG:
      - ACK:
      - PSH:
      - RST:
      - SYN:
      - FIN:
   - 窗口大小：字段长 16 位，表示从确认序号所指位置开始能够接收的数据大小，tcp 不允许发送超过该窗口大小的数据。
   - [ ] 没有理解 tcp 相关的字段

#### 三次握手
所谓的面向连接就是三次握手，在客户端调用 `connect()` 函数，三次握手就自动进行了。

tcp 连接是双向连接，客户端和服务器需要分别向对方发送连接请求，并且建立连接，三次握手成功之后，二者之间的双向连接也就成功建立了。

1. 前提条件
   - 服务器端：已经启动，并且启动了监听；
   - 客户端：基于服务器端监听的 IP 和端口，向服务器端发起连接请求；

#### 四次挥手
四次挥手是断开连接的过程，需要双向断开。关于哪一方先断开连接是没有要求的。通信的两端如果想要断开连接就需要调用 `close()` 函数。当两端都调用该函数，四次挥手也就完成了。


#### 流量控制
流量控制可以让发送端根据接收端的实际能力控制发送的数据量。它的具体操作是，接收端主机向发送端主机通知自己可以接收数据的大小，于是发送端会发送不会超过该大小的数据量，该限制大小即为窗口大小。**窗口大小由接收端主机决定**。

tcp 首部中，专门有一个字段来通知窗口大小，接收主机将自己可以接收的缓冲区大小放在该字段中通知发送端。当接收端的缓冲区面临数据溢出时，窗口大小的值也随之改变，设置一个更小的值通知发送端。从而控制数据的发送量，实现流量的控制。**这个控制流程的窗口也可以称作滑动窗口**。

#### 拥塞控制

#### 粘包和拆包
1. 粘包
   - tcp 是基于流的传输协议，数据发送端和接收端每次处理的数据量、处理的频率可以是不对等的。
   - 通信过程中，客户端会不定期地给服务器发送一个不定长度的有特定含义的字符串；而服务器端需要接收这个不定长的字符串，并对其进行解析。
   - 服务器端在接收数据时可能出现的情况：
     - 一个完整数据包；
     - N 个数据包，由于每个包的长度不定，无法将各个数据包拆开；
     - N 个数据包 + 下一个数据包的一部分；
     - 不足一个数据包，下一次收到剩余部分 + 下个数据包的一部分；
     - 另外，还有一些不可抗因素：比如客户端和服务器端的网速不一样，发送和接收的数据量也会不一致；
2. 解决方案：
   - 使用标准的应用层协议（比如 http、https）来封装要传输的不定长的数据包；
   - 在每条数据的尾部添加特殊字符，如果遇到特殊字符，代表当条数据接收完毕：有缺陷（效率低，需要一个字节一个字节接收，接收一个字节判断一次，判断是不是那个特殊字符）；
   - 在发送数据块之前，在数据块最前边添加一个固定大小的数据头，这时候数据由两部分组成：数据头 + 数据块；
     - 数据头：存储当前数据包的总字节数，接收端先接收数据头，然后再根据数据头接收对应大小的字节；
     - 数据块：当前数据包的内容；
3. 拆包
   **通常使用添加包头的方式轻松地解决掉这个问题**。关于数据包的包头大小可以根据实际情况进行设定。可以规定包头的固定大小为 4 个字节，用于存储当前数据块的总字节数。
4. 实现时注意事项（以包头占 4 字节为例）：
   - 发送端：
     - 申请的内存大小为 N + 4（N 是数据块所需内存大小）；
     - 将数据块总长度写入申请内存的头 4 个字节中，**需要先将总长度转换为网络字节序（大端）**；
     - 将待发送的数据拷贝到包头后边的内存中，将完整的数据包发送出去（**字符串没有字节序问题**）；
     - 释放申请的堆内存；
   - 接收端
     - 首先接收 4 字节数据，**并将其从网络字节序转换为主机字节序**，从而得到将要接收的数据总长度；
     - 根据得到的数据总长度申请固定大小的堆内存，用于存储待接收数据；
     - 根据得到的数据块长度接收固定数目的数据保存到申请的堆内存中；
     - 处理接收的数据（**需等到数据块完整接收到本地才能进行数据处理**）；
     - 释放存储数据的堆内存；

#### 状态转换

#### 服务器并发与 IO 多路复用
在 tcp 通信过程中，服务器端启动之后可以同时和多个客户端建立连接，并进行网络通信。

1. 单线程/单进程的缺点：
   - 引起程序阻塞的函数：
     - `accept()`：...
     - `read()`：如果通信的套接字对应的读缓冲区没有数据，阻塞当前进程/线程；检测到数据解除阻塞，接收数据；
     - `write()`：如果通信的套接字写缓冲区被写满了，阻塞当前进程/线程；
   - 使用单线程/进程无法满足同时、高效地完成新连接的建立、与已建立连接的客户端之间的通信；
2. 解决方案：
   - 多线程
   - 多进程
   - IO 多路复用实现
   - IO 多路复用 + 多线程实现
3. 多进程并发
4. 多线程并发
5. IO 多路复用-select, poll, epoll

#### 套接字通信封装

### UDP
向应用程序提供无连接服务。

- 特点
  - 数据传输没有可靠性；
  - 没有流量控制，也没有拥塞控制；
- 使用特点：
  - 关于何时发送什么数据控制得更为精细：采用 UDP 时只要应用进程将数据传递给 UDP，UDP 就会立即将其传递给网络层；而 TCP 有重传机制，不管可靠传输需要多长时间。在实际应用时，有些场景不希望过分的延迟报文传输，且能容忍一部分数据丢失；
  - 无需建立连接，不会引入建立连接时的延迟；
  - 无连接状态，能支持更多的活跃客户；
  - 分组首部开销较小；

- 通信交互流程
  ![](../../img/internet/260414-001331.png)
  > 图片来源：[1]

### 对比
1. 适用场景：
   - TCP：
   - UDP：视频通话（可以接收偶尔的丢包、偶尔的花屏；如果使用 TCP，每个数据包都要确保可靠传输，当它出错时就重传，这会导致后续的数据包被阻滞，视频效果反而不好）

## 网络编程接口
套接字（Socket）是一套网络通信的接口，使用这套接口来完成网络通信。

### `socket` 函数
  ```c
  int socket(int domain, int type, int protocol);
  ```
  - 功能：create an endpoint for communication，创建一个用于通信的套接字；
  - 返回值：returns a file descriptor that refers to that endpoint，返回一个文件描述符，通过它来引用套接字；失败时返回 -1，查看 `errno` 可知道出错的详细情况；
  - 参数：
    - `domain`：The domain argument specific a communication domain; **this selects the protocol family which will be used for communication**. These family are defined in <sys/socket.h>. The formats currently understood by the Linux kernel include:
      - `AF_UNIX`: local communication，只能用于单一的 Unix 系统进程间通信；
      - `AF_LOCAL`: Synonym（同物异名，同义词） for `AF_UNIX`；
      - `AF_INET`: IPv4 Internet protocols。针对网络通信，可用于远程通信；
      - `AF_INET6`: IPv6 Internet protocols；
      - 其他，见 `man` 手册；
    - `type`：The socket has the indicated types（socket 具有指定的类型）,which specifies the communication semantics. Currently defined types are:
      - `SOCK_STREAM`: Provides **sequenced, reliable, two-way, connection-based type stream**. An out-of-band data transmission mechanism may be supported;
      - `SOCK_DGRAM`: Supports **datagrams（数据包）**(**connectionless, unreliable messages of a fixed maximum length**);
      - 其他，见 `man` 手册；
      - Since Linux 2.6.27, the type argument servers a second purpose: in addition to specifying a socket type, it may include the bitwise（逐位） OR of any of the following values, to modify the behavior of socket()：
        - `SOCK_NONBLOCK`: Set the `O_NONBLOCK` file status flag on the open file description referred to by the new file descriptor. Using this flag saves extra calls to `fcntl` to achieve the same result.
        - `SOCK_CLOEXEC`: Set the close-on-exec (FD_CLOEXEC) flag on the new file descriptor. See the description of the `O_CLOEXEC` flag in `open` for reasons why this may be useful. 
    - `protocol`: The protocol specifies a particular protocol to be used with the socket. **Normally only a single protocol exists to support a particular socket type within a given protocol family, in which case protocol can be specified as 0**. 其他信息见 `man` 手册、`protocol(5)`、`getprotoent(3)`；

### `bind` 函数
  ```c
  int bind(int sockfd, struct sockaddr *addr, int addrlen);
  ```
  - 概述：when a socket is created with `socket(2)`, it exists in a name space (address family) but has no address assigned to it. `bind()` assigns the address specified by `addr` to the socket referred to by the file descriptor `sockfd`. `addrlen` specifies the size, in bytes, of the address struct pointed to by `addr`. Traditionally, this operation is called "assigning a name to a socket".
  - **It is normally necessary to assign a local address using `bind()` before a `SOCK_STREAM` socket may receive connections (seet `accept(2)`)**.
  - The rules used in name binding vary between address families. Consult the manual entries in Section 7 for detailed information. For `AF_INET`, see `ip(7)`; for `AF_INET6`, see `ip6(7)`; for `AF_UNIX`, see `unix(7)`; 其他，见 `man` 手册；
  - **The actual struct passed for the `addr` argument will depend on the address family**. The `sockaddr` structure is defined as something like:
    ```c
    struct socketaddr {
      sa_family_t sa_family;
      char sa_data[4];
    };
    ```
    - **The only purpose of this structure is to cast the structure pointer passed in `addr` in order to avoid compiler warnings**.
    - 示例程序：
      ```c
      #include <sys/socket.h>
      #include <sys/un.h>
      #include <stdlib.h>
      #include <stdio.h>
      #include <string.h>

      #define MY_SOCK_PATH  "/somepath"
      #define LISTEN_BACKLOG  50

      #define handle_error(msg) \
        do {perror(msg); exit(EXIT_FAILURE); } while(0)

      int main(int argc, char *argv[]) {
        int sfd, cfd;
        struct sockaddr_un my_addr, peer_addr;
        socklen_t peer_addr_size;

        sfd = socket(AF_UNIX, SOCK_STREAM, 0);
        if (sfd == -1)
          handle_error("socket");

        memset(&my_addr, 0, sizeof(my_addr));
        my_addr.sun_family = AF_UNIX;
        strncpy(my_addr.sun_path, MY_SOCK_PATH, sizeof(my_addr.sun_path) - 1);

        if (bind(sfd, (struct sockaddr*)&my_addr, sizeof(my_addr)) == -1)
          handle_error("bind");
        
        if (listen(sfd, LISTEN_BACKLOG) == -1)
          handle_error("listen");
        
        /*Now we can accept incoming connections one at a time using accept(2)*/

        peer_addr_size = sizeof(peer_addr);
        cfd = accept(sfd, (struct sockaddr*)&peer_addr, &peer_addr_size);
        if (cfd == -1)
          handle_error("accept");

        /* Code to deal with incoming connection(s) */

        /* When no longer required, the socket pathname, MY_SOCK_PATH should be deleted using unlink(2) or remove(3) */
      }
      ```
    - 不过由于系统的兼容性，我们一般使用另外一个结构（`struct sockaddr_in`）来代替：
      ```c
      struct sockaddr_in {
        unsigned short sin_family;
        unsigned short sin_port;
        struct in_addr sin_addr;
        unsigned char sin_zero[8];
      };
      ```
      - 如果使用 `Inernet`，则 `sin_family` 一般为 `AF_INET`；
      - `sin_addr` 设置为 `INADDR_ANY` 表示可以和任何的主机通信；
      - `sin_port` 是要监听的端口号；
      - `bind` 将本地的端口同 `socket()` 返回的文件描述符捆绑在一起，成功返回 0，失败则返回 -1。

### `listen` 函数
```c
int listen(int sockfd, int backlog);
```
- 功能：`listen()` marks the socket referred to by `sockfd` as a **passive socket**, that is, as **a socket that will be used to accept incoming connection requests using `accept(2)`**。
- 返回值：成功时返回 0；失败时返回 -1，同时 `errno` 也被设置。
- 参数：
  - `sockfd`：The `sockfd` argument is a file descriptor that **refers to a socket of type `SOCK_STREAM` or `SOCK_SEQPACKET`**.
  - `backlog`：The `backlog` argument **defines the maximum length to which the queue of pending connections for `sockfd` may grow**. If a connection request arrives when the queue is full, the client may receive an error with an indication of `ECONNREFUSED` or, if the underlying protocol supports retransmission, the request may be ignored so that a later reattempt at connection succeeds.

### `accept` 函数
```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```
- 功能：The `accept()` system call **is used with connection-based socket types** (`SOCK_STREAM`, `SOCK_SEQPACKET`). **It extracts the first connection request on the queue of pending connections for the listening socket, `sockfd`, creates a new connected socket, and returns a new file descriptor referring to that socket. The newly created socket is not in the listening state. The original socket `sockfd` is unaffected by this call**.
- 参数：
  - `sockfd`：The argument `sockfd` is a socket that has been created with `socket(2)`, bound to a local address with `bind(2)`, and is listening for connections after a `listen(2)`.
  - `addr`: The argument `addr` is a pointer to a `sockaddr` structure. **This structure is filled in with the address of the peer socket（可以理解为 `addr` 中传出的是客户端地址）, as known to the communications layer. The exact format of the address returned `addr` is determined by the socket's address family** (see socket(2) and the respective protocol man pages). When `addr` is NULL, nothing is filled in; in this case, `addrlen` is not used, and should also be NULL.
  - `addrlen`: The `addrlen` argument is a value-result argument; **the caller must initialize it to contain the size (in bytes) of the structure pointed to by `addr`; on return it will contain the actual size of the peer address**.
- 返回值：
  - 返回值：On success, this system call returns a file descriptor for the accepted socket （根据功能描述，该 socket 是新创建的，用以表示与客户端（peer socket）建立连接的 socket）(a nonnegative integer). On error, -1 is returned, `errno` is set appropriately, and **`addrlen` is left unchanged**.
  - 传出参数：The returned address is truncated if the buffer provided is too small; in this case, `addrlen` will return a value greater than was supplied to the call.
  - 根据是否 `nonblocking` 返回值：**If no pending connections are present on the queue, and the socket is not marked as nonblocking, `accept()` blocks the caller until a connetion is present**. If the socket is marked nonblocking and no pending connections are present on the queue, `accept()` fails with the error `EAGIN` or `EWOULDBLOCK`.
- poll 机制（IO 复用）的使用：In order to be notified of incoming connections on a socket, you can use `select(2)`, `poll(2)`, or `epoll(7)`. A readable event will be delivered when a new connection is attempted and you may then call `accept()` to get a socket for that connection. Alternatively, you can set the socket to deliver `SIGIO` when activity occurs on a socket; see `socket(7)` for details.

### `connect` 函数
```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
- 功能：
  - The `connect()` system call connects the socket referred to by the file descriptor `sockfd` to the address specified by `addr`. The `addrlen` argument specifies the size of `addr`. The format of the address in `addr` is determined by the address space of the socket `sockfd`; see `socket(2)` for further details.
  - IF the socket `sockfd` is of type `SOCK_DGRAM`, then `addr` is the address to which datagrams are sent by default, and the only address from which datagrams are received（当 socket 类型为 UDP 时，发送端调用 connect 指定接收端的地址，并且只有这个地址的数据包会被接收）. If the socket is of type `SOCK_STREAM` or `SOCK_SEQPACKET`, this call attemps to make a connection to the socket that is bound to the address specified by `addr`.
  - Some protocol sockets (e.g., `UNIX` domain stream sockets) may successfully `connect()` only once.
  - Some protocol sockets (e.g., datagrams sockets in the `UNIX` and Internet domains) may use `connect()` multiple times to change their association.
  - Some protocol sockets (e.g, TCP sockets as well as datagram socket in the UNIX and Internet domains) may dissolve（终止、解除、解散） the association by connecting to an address with the `sa_family` member of `sockaddr` set to `AF_UNSPEC`; thereafter, the socket can be connected to another address. (`AF_UNSPEC` is supported to Linux since kernel 2.2.)
- 返回值：If the connectin or binding succeeds, zero is returned. On error, -1 is returned, and `errno` is set appropriately.

### `send` 相关函数
```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);

// 其他相关函数，可通过 man 2 send 来找到
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);

ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
```
- 功能：The system calls `send()`, `sendto()`, and `sendmsg()` are used to transmit a message to another socket.
  - `send`: **The `send()` call may be used only when the socket is in a `connected` state** (so that the intended recipient iss known-预期的接收者是已知的). The only difference between `send()` and `write(2)` is the presence of `flags`. With a zero `flags` argument, `send()` is equivalent to `write(2)`. 另外，如下调用也是等效的：
    ```c
    send(sockfd, buf, len, flags);
    // 和
    sendto(sockfd, buf, len, flags, NULL, 0);
    ```
  - `sendto`: If `sendto()` is used on a connection-mode (`SOCK_STREAM`, `SOCK_SEQPACKET`) socket, the arguments `dest_addr` and `addrlen` are ignored (and **the error `EISCONN` may be returned when they are not NULL and 0**), and the error `ENOTCONN` is returned when the socket was not actually connected. Otherwise, the address of the target is given by `dest_addr` with `addrlen` specifying its size.
  - `senbmsg`: For `sendmsg()`, the address of the target is given by `msg.msg_name`, with `msg.msg_namelen` specifying its size. 
- 参数：
  - `sockfd`: The argument `sockfd` is the file descriptor of the sending socket（发送方对应的 sockfd）.
  - `buf` and `len`: For `send()` and `sendto()`, the message is found in `buf` and has length `len`. For `sendmsg()`, the message is pointed to by the elements of the array `msg.msg_inv`. The `sendmsg()` call also allows sending ancillary（辅助的、补充的、附属的） data (also known as control information).
  - `flags`: The flags argument is the bitwise OR of zero or more of the following flags.
    - `MSG_CONFIRM`: Tell the link layer that forward progress happened: you got a successful reply from the other size. 其他信息，见 `man` 手册；
    - `MSG_DONTROUTE`: Don't use gateway to send out the packet, send to hosts only on directly connected networks. 其他信息，见 `man` 手册；
    - `MSG_MORE`：略；
    - `MSG_NOSIGNAL`：略；
    - `MSG_OOB`：略；
  - `msghdr` 的定义：
    ```c
    struct msghdr {
      void *msg_name;         // optinal address
      socklen_t msg_namelen;  // size of address
      struct iovec *msg_iov;  // scatter/gather array
      size_t msg_iovlen;      // elements in msg_iov
      void *msg_control;      // ancillary data
      size_t msg_controllen;  // ancillary data buffer len
      int msg_flags;          // flags (unused)
    }
    // 其他信息见 man 手册
    ```
- 返回值：No indication of failure to deliver is implicit in a `send()`. Locally detected errors are indicated by a return value of -1.
- 注意：
  - **IF the message is too long to pass atomically through the underlying protocol, the error `EMSGSIZE` is returned, and the message is not transimitted**.
  - When the message does not fit into the send buffer of the socket, **`send()` normally blocks**, unless the socket has been placed in nonblocking I/O mode. In nonblocking mode it would fail with the error `EAGAIN` or `EWOULDBLOCK` in this case. **The `select(2)` call may be used to determine when it is possible to send more data**.

### `recv` 相关函数
```c
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```
- 功能：The `recv()`, `recvfrom()` and `recvmsg()` calls are used to receive messages from a scoket. **They may be used to receive data on both connectionless and connection-oriented sockets**.
  - `recv`: The only difference between `recv()` and `read(2)` is the presence of `flags`. With a zero `flags` argument, `recv()` is equivalent to `read(2)`(but see NOTES). **The `recv()` call is normally used only on a connected socked**. 另外，如下调用也是等效的：
    ```c
    recv(sockfd, buf, len, flags);
    // 和
    recvfrom(sockfd, buf, len, flags, NULL, NULL);
    ```
  - `recvfrom`: `recvfrom()` places the received message into the buffer `buf`. The caller must specify the size of the buffer in `len`. 其他内容略，详见 `man` 手册；
  - `recvmsg()`: The `recvmsg()` call uses a `msghdr` structure to minimize the number of directly supplied arguments. This structure is defined as follows in <sys/socket.h>：其他内容略，见 `man` 手册；
- 参数：
  - `flags`: The `flags` argument is formed by ORing one or more of the following values: 略，详见 `man` 手册；
- 返回值：上述三个函数都返回成功接收数据的长度。If a message is too long to fit in the supplied buffer, excess bytes may be discarded depending on the type of socket the message is recevived from. 如果产生错误，则返回 -1，同时设置 `errno`.
- 注意： 
  - If no messages are avaiable at the socket, the receive calls wait for a message to arrive, unless the socket is nonblocking (see `fcntl(2)`), in which case the value -1 is returned and the external variable `errno` is set to `EAGAIN` or `EWOULDBLOCK`. 
  - The receive calls normaly return any data avaialbe, up to the requested amount, rather than waiting for receipt of the full amount requested（receive 调用只会返回可获取的数据，这取决于调用时指定的接收数据长度，而不会等待到所有要求的数据都可获取）.
  - An application can use `select(2)`, `poll(2)`, or `epoll(7)` to determine when more data arrives on a socked.


## 示例程序

## 参考
1. 韦东山-《嵌入式 Linux 应用开发完全手册 V5.2》第 8 章；
2. [爱编程的大丙-Linux](https://subingwen.cn/linux/)