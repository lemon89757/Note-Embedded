# proc
proc - process information pseudo-filesystem

在 Linux 系统中，proc 是一个特殊的**伪文件系统（pseudo-filesystem）**，它不占用实际磁盘空间，而是由内核实时生成，用于暴露内核数据结构、进程状态及系统配置信息。proc 是了解系统内部运行状态的窗口，也是调试、监控和调优的重要工具。

无论是查看 CPU 核心数、检查内存使用情况，还是调试一个异常进程的内存布局，proc 都能提供直接且**实时**的数据。

## 特性
1. 动态生成：文件内容由内核实时生成，不存储在磁盘中（可通过 `df -h /proc` 验证，显示容量为 0）；
2. 无持久化：系统重启后，`/proc` 的内容会重新初始化；
3. 权限控制：多数文件仅允许读取（如 `/proc/cpuinfo`），部分文件（如 `/proc/sys` 下的内核参数）可写入，但需 root 权限；
4. 文件格式：内容多数为文本格式，便于人类阅读和脚本解析（但需注意格式可能随内核版本变化）；

## proc 文件系统中的内容
### 系统全局信息文件
1. CPU 信息（`/proc/cpuinfo`）
   - 包含 CPU 的型号、厂商、核心数、频率、缓存大小等信息；
    ```bash
    # 处理器信息
    cat /proc/cpuinfo | grep 'processor'
    # 每个核心的运行频率
    cat /proc/cpuinfo | grep 'cpu MHz'
    # model name、逻辑 CPU 数（超线程）、物理核心数
    cat /proc/cpuinfo | grep -E "model name|siblings|cpu cores"
    ```
2. 内存信息（`/proc/meminfo`）
   - 包含总内存（MemTotal）、空闲内存（MemFree）、已用内存（MemTotal-MemFree）、可用内存（考虑缓存和缓冲区）（MemAvailable）、交换分区总量/空闲量（SwapTotal/SwapFree）、缓冲区（Buffers）和缓存（Cached）等信息；
   - `free -h` 命令的数据就来源于此；
    ```bash
    grep -E `MemTotal|MemFree|MemAvailable|Cached` /proc/meminfo
    ```
3. 系统平均负载（`/proc/loadavg`）
    - 记录系统在过去 1 分钟、5 分钟和 15 分钟内的平均负载，以及进程总数和最近运行进程 PID；
    - 平均负载是指系统处于可运行状态和不可中断状态的平均进程数；
    - 一般来说，如果平均负载持续高于 CPU 核心数，说明系统负载较重，可能存在性能瓶颈。
        ```bash
        cat /proc/loadavg
        # 输出
        0.85 0.62 0.41 2/189 12345
        # 1 分钟负载、5 分钟负载、15 分钟负载、2 个运行中进程/189 个总进程、最近 PID 12345
        ```
4. 内核版本（`/proc/version`）
    - 包括内核的主版本号、次版本号、修订版本号以及编译信息等。
5. `/proc/stat` 文件
    - 记录了系统启动以来的各种 CPU 统计信息；
    - 通过对这些信息的分析，能够计算出 CPU 在不同状态下的时间占比，从而判断 CPU 的负载情况。
        ```bash
        # 获取 CPU 的用户态时间、内核态时间、空闲时间等信息
        cat /proc/stat | grep 'cpu'
        # 输出结果可能如下：
        cpu 12345 6789 1234 56789 1234 567 89 123 0 0
        # 分别对应：用户态时间、nice 时间（低优先级用户态时间）、系统态时间、空闲时间、I/O 等待时间、硬中断时间、软中断时间等
        ```
        - 通过上述内容可以计算出 CPU 的使用率；
        - 根据使用率可以采取相应的优化措施。如果发现 CPU 使用率过高，且主要是用户态进程占用较多，可能需要优化应用程序的代码，减少不必要的计算；如果是内核态占用过高，可能需要检查系统调用是否频繁，或者是否存在内核模块的性能问题。
6. `proc/diskstats` 文件
   - 提供了磁盘的详细统计信息，包括读写次数、读写字节数、I/O 等待时间等；
        ```bash
        # 获取磁盘 sda 的相关信息
        grep 'sda' /proc/diskstats
        ```
   - 通过分析上述命令输出数据，可以判断磁盘的 I/O 性能是否正常。如果磁盘读写频繁，I/O 等待时间过长，可能需要优化磁盘 I/O 操作。比如，对于频繁读写的应用程序，可以考虑使用缓存机制，减少磁盘 I/O 次数；对于顺序读写的场景，可以调整文件系统的块大小，提高读写效率；等等。
7. `/proc/uptime` 文件
   - 系统运行时间
        ```bash
        cat /proc/uptime
        # 输出：系统启动至今的秒数 + 空闲进程占用的总秒数
        ```
8. `/proc/cmdline` 文件
   - 内核启动参数，记录系统启动时传递给内核的参数（如 quiet、splash、root=/dev/sda1 等）；
### 进程相关文件
前面的进程：init、systemd

1. 进程状态汇总（`/proc/[PID]/status`）
    - 该文件汇总了进程的关键信息，包括进程状态（运行、睡眠、僵尸等）、进程 ID（PID）和父进程 ID（PPID）、实际和有效的用户/组 ID（用于权限控制）、内存使用情况；
    - 通过该文件了解进程的基本状况，判断进程是否正常运行，是否存在内存泄漏等问题。
        ```bash
        cat /proc/$(pidof sshd)/status | grep -E "Name|VmRSS|Threads"
        Name: sshd
        VmRSS: 1234KB   # 常驻内存大小（实际物理内存使用）
        Thread: 4       # 线程数
        # VmSize: 虚拟内存大小（单位：KB）
        # Uid/Gid：进程的真实/有效 UID/GID
        ```
2. 进程状态的精简数字格式（`/proc/[PID]/stat`）
    - 供 `ps`、`top` 等工具解析；
    - 52 个字段，PID（1）、进程状态（3）（R：运行，S：睡眠，Z：僵尸等）、VmRSS（22）；
3. 内存映射（`/proc/[PID]/maps`）
    - 此文件描述进程的虚拟内存空间布局，列出了进程使用的每一段内存区域的起始和结束地址、访问权限（读-r、写-w、执行-x、私有-p）、偏移量、设备号、inode 以及映射的文件或资源（如堆栈、堆、共享库等）；
    - 可以从这些信息中看到进程的代码段、数据段、堆、栈以及共享库的内存映射情况；
    - 分析此文件对于调试内存泄漏、理解共享库加载或进行低级性能分析至关重要；
    - 如果发现某个进程存在内存泄漏问题，可以通过分析 `/maps` 文件，找出异常占用内存的区域，进而定位到问题代码。
4. 打开的文件描述符（`/proc/[PID]/fd`）
    - 该目录包含了进程打开的所有文件描述符的符号链接，每个符号链接以文件描述符编号命名（如 0，1，2 分别对应标准输入、输出、错误），并指向实际的文件、套接字或管道；
    - 通过查看这个目录，可以追踪进程的文件操作，查找文件锁的持有者，了解进程与外部资源的交互情况；
    - 当一个文件无法正常关闭或出现文件锁冲突时，就可以通过查看该进程的 `/fd` 目录，找出问题的根源。
5. 启动命令行（`proc/[PID]/cmdline`）
    - 这个文件保存了启动该进程的完整命令行参数，例如：
        ```bash
        # 一个启动的 MySQL 数据库进程，通过查看 cmdline 文件可能看到：
        /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib64/mysql/plugin --log-error=/var/log/mysqld.log --pid-file=/var/run/mysqld/mysqld.pid
        ```
    - 通过这些信息，可以重现进程的运行环境，检查配置参数是否正确传递，对于检查进程启动失败或运行异常等问题非常有帮助。

---
假设应用程序突然崩溃，此时可以通过查看 `/proc/[PID]/` 目录下的文件来获取关键信息。
   - 首先，查看 `/proc/[PID]/status` 文件，检查进程的状态、信号等信息。如果进程因为收到某个信号而终止，在 `status` 文件中会有相应的记录，比如 `SigQ:0/1234`，表示信号队列的使用情况、`SigPnd:0000000000000` 表示当前进程未处理的信号。通过分析这些信息，可以初步判断进程崩溃的原因是否与信号有关；
   - 接着，查看 `/proc/[PID]/coredump` 文件（如果启动了核心转储功能），这个文件包含了进程崩溃时的内存快照和堆栈信息。通过分析核心转储文件，可以定位导致进程崩溃的具体代码行。例如，使用 gdb 工具加载核心转储文件：
        ```bash
        gdb /path/to/executable /proc/[PID]/coredump
        ```
        - 执行上述命名后，使用 `bt` 命令查看堆栈回溯信息，就可以看到函数调用栈，从而找出问题所在。
### 内核可配置参数文件
`/proc/sys` 目录下的文件用于动态调整内核参数，这些参数涉及网络、内存管理、进程管理等多个方面，通过修改这些参数，可以在不重启系统的情况下，优化系统性能或满足特定的应用需求。

1. 网络参数调整
    ```bash
    # IP 转发控制，0-关闭转发，1-开发转发
    echo 1 > /proc/sys/net/ipv4/ip_forward

    # listen() 队列的最大长度的修改通过文件：
    /proc/sys/net/core/somaxconn 

    # 调整 TCP 链接超时时间
    echo 30 > /proc/sys/net/ipv4/tcp_fin_timeout # 减少 TIME_WAIT 状态超时（默认 60 秒）
    ```
2. 内存管理参数
    ```bash
    # 系统将内存数据交换到磁盘空间（swap）的倾向程度，取值范围为 0-100
    # 减少内存交换对性能的影响，将 swappiness 的值调小
    /proc/sys/vm/swappiness

    # 清理页面缓存
    echo 1 > /proc/sys/vm/drop_caches

    # 清理目录项和 inode 缓存
    echo 2 > /proc/sys/vm/drop_caches

    # 清理页面缓存、目录项和 inode 缓存
    echo 3 > /proc/sys/vm/drop_caches
    ```
3. 文件描述符限制
   ```bash
   # 下述文件定义了系统级别的文件描述符上限，设置文件描述符上限：
   echo 65535 > /proc/sys/fs/file-max   # 限制进程打开文件数（系统级）
   ```

注意：
   - 通过上述方式修改的参数，在系统重启后会恢复回默认值。如果希望参数永久生效，通常需要将参数配置写入 `/etc/sysctl.conf` 文件或 `/etc/sysctl.d` 目录下的配置文件中，然后执行 `sysctl -p` 命令使配置生效；
   - 需 root 权限；

## 使用
### 使用场景
1. 系统监控与诊断：
   - 查看内存使用：`cat /proc/meminfo`（替代 `free -m`）；
   - 检查磁盘 I/O：`/proc/diskstats` 记录磁盘读写次数和吞吐量；
   - 网络连接状态：`/proc/net/tcp` 展示所有 TCP 连接（本地/远程 IP、端口、状态）。
2. 进程调试与问题定位
   - 确认进程是否卡顿：通过 `/proc/[pid]/stat` 的状态字段（如 `D` 表示不可中断睡眠、可能是 I/O 阻塞）；
   - 定位内存泄漏：对比 `VmRSS`（`/proc/[pid]/status`）随时间的变化，或分析 `/proc/[pid]/maps` 中的异常内存增长区域；
   - 恢复误删文件：若进程仍打开已删除文件，可通过 `/proc/[pid]/fd/[fd]` 恢复（如 `cp /proc/1234/fd/5 /tmp/recovered.log`）。
3. 替代系统命令
   - 许多常用工具，如 `ps`（`ps aux` 读取 `/proc/[pid]/stat` 和 `/proc/[pid]/cmdline`）、`top`、`free`（读取 `/proc/meminfo`）、`netstat` 等，本质上是 `/proc` 的解析器。

### 实践与注意事项
1. 避免直接写入 `proc`
   除 `/proc/sys` 下部分文件外，多数 `/proc` 文件为只读。错误写入可能导致系统不稳定（如修改 `/proc/kcore` 会导致系统崩溃）。
2. 处理动态变化的 `/proc/[pid]`
   进程可能随时退出，导致 `/proc/[pid]` 目录消失。在脚本中需先检查目录是否存在：
   ```sh
    pid=1234
    if [ -d "/proc/$pid" ]; then
        cat /proc/$pid/cmdline
    else
        echo "进程 $pid 已退出"
    fi
   ```
3. 谨慎解析 `/proc` 文件格式
   内核版本升级可能导致 `/proc` 文件格式变化（如字段增减）。若需编程解析，建议优先使用封装库（如 `libprocps`），而非直接解析文本。
4. 避免读取大文件
   部分文件（如 `/proc/kcore`，表示物理内存镜像）体积巨大（通常等于物理内存大小），直接 `cat` 会导致内存耗尽。

### 自动化脚本示例
- 定期采集系统 CPU 使用率和内存使用率信息，并记录到日志文件中：
    ```sh
    #!/bin/bash
    log_file=/var/log/system_monitor.log
    while true
    do
        # 获取 CPU 使用率
        cpu_usage=$(top -bn1 | grep 'Cpu(s)' | awk '{print $2 + $4}')
        # 获取内存使用率
        mem_total=$(grep 'MemTotal' /proc/meminfo | awk '{print $2}')
        mem_free=$(grep 'MemFree' /proc/meminfo | awk '{print $2}')
        mem_usage=$(echo "scale=2; (1 - $mem_free / $mem_total) * 100" | bc)
        timestamp=$(date + "%Y-%m-%d %H:%M:%S")
        echo "$timestamp - CPU Usage: $cpu_usage%, Memory Usage: $mem_usage%" >> $log_file
        sleep 60 # 每 60 秒采集一次
    done
    ```
- 监控进程内存使用趋势
    ```sh
    #!/bin/bash
    pid=$1
    if ! [ -d "proc/$pid" ]; then
        echo "进程 $pid 不存在"
        exit 1
    fi
    while true; do
        timestamp=$(date +"%Y-%m-%d %H:%M:%S")
        rss=$(grep VmRSS /proc/%pid/status | awk "{print $2 " " $3}")
        echo "$timestamp: $rss"
        sleep 5
    done
    ```
- 统计系统中所有进程的线程总数
    ```sh
    #!/bin/bash
    total_threads=0
    for pid in /proc/[0-9]*; do
        if [ -f "$pid/status"]; then
            threads=$(grep Threads "$pid/status" | awk "{print $2}")
            total_threads=$((total_threads + threads))
        fi
    done
    echo "系统总线程数：$total_threads"
    ```

## 参考
1. [知乎-万字长文，彻底搞懂 Linux proc 文件系统](https://zhuanlan.zhihu.com/p/2017356315308090080)
2. [Linux proc 文件系统详解：深入内核与进程的窗口](https://geek-blogs.com/blog/linux-proc-folder/)
3. [内核文档](https://www.kernel.org/doc/html/latest/filesystems/proc.html)
4. man 手册：`man 5 proc`（详细描述 `/proc` 文件格式）；