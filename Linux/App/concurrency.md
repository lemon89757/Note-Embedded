# 并发编程
## 多进程
### 进程概述
1. 程序和进程
   两个不同的概念，状态和占用的系统资源都是不同的。
   - 程序：静态的概念，磁盘上的可执行文件，占用磁盘空间；
   - 进程：动态的概念（运行后的程序），消耗系统的内存和 CPU 资源，每个运行的进程都对应一个属于自己的虚拟地址空间；

2. 并行和并发
   - cpu 时间片
   - 并发：时间片分时复用，假的同时运行；
   - 并行：真正的进程同时运行；

3. PCB
   - processing control block，进程控制块；
   - Linux 内核的进程控制块本质上是一个 `struct task_struct` 的结构体，这个结构体记录了进程运行的相关信息：
     - 进程 id，PID，类型为 `pid_t`，本质是一个整型数；
     - 进程状态：就绪、运行、挂起、停止等状态；
     - 进程的虚拟地址空间信息；
     - 控制终端信息：进程在哪个终端启动默认就和哪个终端绑定；
     - 当前工作目录：默认情况下，启动进程的目录就是当前的工作目录；
     - umask 掩码：在创建新文件的时候，通过该掩码屏蔽某些用于对文件的操作权限；
     - 文件描述符表：每个被分配的文件描述符都对应一个已经打开的磁盘文件；
     - 和信号相关的信息：在 Linux 中 调用函数、键盘快捷键、执行 shell 命令 等操作都会产生信号。
       - 阻塞信号集：记录当前进程中阻塞哪些已产生的信号，使其不能被处理；
       - 未决信号集：记录在当前进程中产生的哪些信号还没有被处理掉；
     - 用户 id 和组 id：当前进程属于哪个用户，属于哪个用户组；
     - 会话（session）和进程组：多个进程的集合叫进程组，多个进程组的集合叫会话；
     - 进程可以使用的资源上限：可以使用 shell 命令 `ulimit -a` 查看详细信息；

4. 进程状态
   - 创建态
   - 就绪态
   - 运行态
   - 阻塞态（挂起态）
   - 退出态（停止态）

    创建态和退出态维持的时间是非常短的，稍纵即逝。

5. 进程命令
   1. 查看进程
    ```bash
    # a: 查看所有终端的信息，
    # u: 查看用户相关信息，
    # x: 显示和终端无关的进程信息。
    # 输出信息中 tty 表示进程关联的终端，如果为 ? 则表示进程不关联终端
    ps aux 
    ```

   2. 杀死进程
    ```bash
    # kill 命令用于发送某个信号到对应的进程，进程收到某些信号后默认的处理动作就是退出进程
    kill -l  # 查看 Linux 中的标准信号
    # # 9 号信号（SIGKILL）的行为是无条件杀死进程
    kill -9 PID
    kill -SIGKILL PID
    ``` 

### 进程创建
PID 的类型为 `pid_t`，其本质是一个正整数。PID 为 1 的进程是 Linux 系统中创建的第一个进程。

1. 获取当前进程的 PID
   ```c
   #include <sys/types.h>
   #include <unistd.h>

   pid_t getpid(void);
   ```
2. 获取当前进程的父进程 PID
   ```c
   #include <sys/types.h>
   #include <unistd.h>

   pid_t getppid(void);
   ```
3. 创建一个新的进程
   ```c
   #include <unistd.h>

   pid_t fork(void);
   ```

-----
1. `fork()` 函数解析：
   - 启动磁盘上的应用程序，得到一个进程，如果在这个启动的进程中调用 `fork` 函数，就会得到一个新的进程，我们习惯将其称之为**子进程**；
   - **子进程的虚拟地址空间是基于父进程的拷贝出来的**，虽然是拷贝，但是两个地址空间中存储的信息不可能是完全相同的；
   - [ ] 拷贝内容

   - `fork` 调用成功后，会返回两个值，父子进程的返回值是不同的：
     - 该函数调用成功之后，从一个虚拟地址空间变成了两个虚拟地址空间，每个地址空间中都会将 `fort()` 的返回值记录下来，这就是会得到两个返回值的原因；
     - **父进程的虚拟地址空间中将该返回值标记为一个大于 0 的整数（记录的是子进程的进程 ID）**；
     - **子进程的虚拟地址空间中将该返回值标记为 0**；
     - 在程序中需要通过 `fork()` 的返回值来判断当前进程是子进程还是父进程：
        ```c
        int main() {
            pid_t pid = fork();
            printf("当前进程 fork() 的返回值：%d\n", pid);
            if (pid > 0) {
                // 父进程执行逻辑
            } else if (pid == 0) {
                // 子进程执行逻辑
            } else {
                // 创建子进程失败
            }

            // 不加判断，父子进程都会执行这个循环
            for (int i = 0; i < 5; ++i) {
                printf("%d\n", i);
            }

            return 0;
        }
        ```

### 父子进程
**在父进程中成功创建了子进程，子进程就拥有父进程代码区的所有代码**（因为执行的是拷贝）。

1. 父子进程代码开始运行的位置
   - 父进程：从 `main()` 函数开始运行；
   - 子进程：从 `fork()` 函数开始运行；

2. 循环创建子进程
   ```c
   // 如下程序会最终得到 8 个进程：1 个父进程 + 3 个进程 + 4 个孙子进程
   #include <stdio.h>
   #include <stdlib.h>
   #include <unistd.h>
   #include <string.h>

   int main() {
    for (int i = 0; i < 3; ++i) {
        pid_t pid = fork();
        printf("当前进程 pid：%d\n", getpid());
    }
    return 0;
   }

   // 如下程序会最终得到 4 个进程：1 个父进程 + 3 个子进程
   #include <stdio.h>
   #include <stdlib.h>
   #include <unistd.h>
   #include <string.h>

   int main() {
    pid_t pid;
    for (int i = 0; i < 3; ++i) {
        pid = fork();
        if (pid == 0) break;
    }
    printf("当前进程 pid：%d\n", getpid());
    return 0;
   }
   ```

3. 终端显示问题
   ```c
   // 解决终端显示问题，需要让所有子进程退出之后再退出父进程
   // 比如在父进程代码中调用 sleep()：
   pid_t pid = fork();
   if (pid > 0) {
    sleep(3);   // 让父进程休眠 
   } else if (pid == 0) {
    // 子进程
   }
   ```

4. 关于父子进程通过全局变量实现数据交互的问题
   两个进程中是不能通过全局变量实现数据交互的，因为每个进程都有自己的地址空间。两个同名全局变量存储在不同的虚拟地址空间中，二者没有任何关联性。**如果要进行进程间通信，需要使用管道、共享内存、本地套接字、内存映射区、消息队列等方式**。

### `execl` 和 `execlp` 函数
通过运行的进程启动磁盘上另一个可执行文件程序。
```c
#include <unistd.h>

extern char **environ;
int execl(const char *path, const char *arg, ...);
int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg, ...);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execvpe(const char *file, char *const argv[], char *const envp[]);
```
- 这些函数成功后不会返回，**因为调用进程的实体，包括代码段、数据段和堆栈等都已经被新的内容取代**（也就是说用户区数据基本全部被替换掉了），只留下进程 ID 等一些表面上的信息保持原样；
- 调用失败时返回 -1，从原程序的调用点接着往下运行；
- 也就是说，**`exec` 族函数并没有创建新进程的能力**，只是让启动的新程序寄生到自己虚拟地址空间内，并且原有的用户空间数据被新的进程数据填充；
- 常用 `execl()` 和 `execlp()` 这两个函数。

1. `execl()`：可用于执行任意一个可执行程序，函数需要通过指定的文件路径才能找到这个可执行程序。
    ```c
    #include <unistd.h>

    int execl(const char *path, const char *arg, ...);
    ```
    - `path`: 要启动的可执行程序的路径，推荐使用绝对路径；
    - `arg`: `ps aux` 查看进程的时候，启动进程的名字，可以随意指定，一般和启动的可执行程序名相同；
    - `...`: 要执行的命令需要的参数，可以写多个，**最后以 NULL 结尾，表示参数指定完了**；
    - 返回值：成功，没有返回值；失败，返回 -1；
2. `execlp()`：常用于执行已经设置了环境变量的可执行程序，函数中的 `p` 就是 `path`，也就是说这个函数会自动搜索系统的环境变量 `PATH`，因此使用这个函数执行可执行程序不需要指定路径，只需要指定出名字即可。
    ```c
    // p = path
    int execlp(const char *file, const char *arg, ...);
    ```
    - `file`：可执行程序的名字，在环境变量 `PATH` 中，可执行程序可以不加路径；没有在环境变量 `PATH` 中，可执行程序需要指定绝对路径；
3. 使用示例
    `exec` 族函数，一般不会在进程中直接调用。如果直接调用，这个进程的代码区被替换，也就不能按照原来的流程工作了。**一般在调用这些函数的时候会先创建一个子进程，在子进程中调用 `exec` 族函数**：
    ```c
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>

    int main() {
        pid_t pid = fork();
        if (pid == 0) {
    #if 1
            execl("/bin/ps", "title", "aux", NULL);
    #else
            execlp("ps", "title", "aux", NULL);
    #endif
            // 如果成功，下面所有代码都不会执行
            perror("execl");
            printf("+++++++++++++++++++\n");
        } else if (pid > 0) {
            // 父进程逻辑
        }

        return 0;
    }
    ```

### 进程控制
进程控制主要指：进程的退出、进程的回收 和 进程的特殊状态 孤儿进程 和 僵尸进程。
1. 结束进程：
   - 如果想要直接退出某个进程，可以直接在程序的任何位置调用 `exit()` 或者 `_exit()` 函数。函数的参数相当于退出码：
        ```c
        // 专门退出进程的函数，在任何位置调用都可以
        // 标准 c 库函数
        #include <stdlib.h>
        void exit(int status);

        // linux 的系统函数
        // 可以这么理解，在 Linux 中 exit() 函数封装了 _exit()
        #include <unistd.h>
        void _exit(int status);
        ```
   - 在 `main` 函数中直接使用 `return` 也可以直接退出进程，假如在一个普通函数中调用 `return` 只能返回到调用者的位置，而不能退出进程。

2. 孤儿进程
   - 在一个启动的进程中创建子进程，这时候父子进程同时运行。但是父进程由于某种原因先退出了，子进程还在运行，这时候这个子进程就可以被称之为孤儿进程。
   - 操作系统在检测到一个进程变成了孤儿进程，这时候系统中就会有一个固定的进程领养这个孤儿进程。**如果使用 Linux 没有桌面终端，这个领养孤儿进程的进程就是 `init` 进程**。
   - 领养孤儿进程的原因：在子进程退出时，进程中的用户区可以自己释放，当时进程内核区的 pcb 资源自己无法释放，必须要由父进程来释放子进程的 pcb 资源。当孤儿进程被领养之后，pcb 资源能够正常被释放，从而避免系统资源的房费。

3. 僵尸进程
   - 在一个启动的进程中创建子进程，这时候就有了父子两个进程。父进程正常运行，子进程先于父进程结束，子进程无法释放自己的 pcb 资源，需要父进程来完成，如果此时父进程不进程该操作，这时候子进程就变成了僵尸进程；
   - 僵尸进程不能将它看成是一个正常的进程，这个进程已经死亡了，用户资源已经被释放了，只是还占用着一些内核资源（pcb）；
   - 僵尸进程的出现是由于这个已死亡进程的父进程不作为造成的（父进程一直在运行，一直不退出也不进行回收）；
   - **消灭僵尸进程的方法是，杀死这个僵尸进程的父进程**，这样僵尸进程的资源就被系统回收了。通过 `kill -9 僵尸进程 PID` 的方式是不能消灭僵尸进程的，这个命令只对活着的进程有效。

4. 进程回收
   为了避免僵尸进程，一般会在父进程中进行子进程的资源回收，回收方式有两种：
   - 一种阻塞 `wait()`；
     ```c
     // man 2 wait
     #include <sys/wait.h>

     pid_t wait(int *status);
     ```
     - `status`：传输参数，通过传递出的信息判断回收的进程是怎么退出的。如果不需要该信息，可以指定为 NULL。
     - `WIFEXITED(status)`：返回 1，进程是正常退出的；得到进程退出时候的状态码，相当于 `return` 后边的数值，或者 `exit()` 函数的参数；
     - `WIFSIGNALED(status)`：返回 1，进程是被信号杀死了；获得进程是被哪个信号杀死的，会得到信号的编号。 
     - 返回值：成功，返回被回收的子进程 PID，失败，-1（没有子进程资源可回收，函数的阻塞会自动解除；回收子进程资源时出现了异常）；
     ```c
     #include <sys/wait.h>

     int main() {
         pid_t pid;
         for (int i = 0; i < 5; ++i) {
             pid = fork();
             if (pid == 0) break;
         }

         // 父进程
         if (pid > 0) {
             while (1) {
                 // 回收子进程资源
                 // 子进程有多个，需要循环回收子进程资源
                 pid_t ret = wait(NULL);
                 if (ret > 0) {
                     printf("成功回收子进程资源，子进程 PID：%d\n", ret);
                 } else {
                     printf("回收失败，或者是没有子进程了...\n");
                     break;
                 }
                 printf("我是父进程，pid = %d\n", getpid());
             }
         } else if (pid == 0) {
             // 子进程，执行这句代码之后，子进程退出了
             printf("我是子进程，pid = %d，父进程 pid = %d\n", getpid(), getppid());
         }

         return 0;
     }
     ```
   - 一种非阻塞 `waitpid()`；
     ```c
     // man 2 waitpid
     #incldue <sys/wait.h>

     pid_t waitpid(pid_t pid, int *status, int options);
     ```
     - 该函数可以设置为阻塞，也可以设置非阻塞；
     - 该函数可以指定回收哪些子进程的资源；
     - `pid`：
       - -1，回收所有的子进程资源，和 `wait()` 是一样的，无差别回收，并不是一次性回收多个，也是需要循环回收的；
       - 大于 0，指定回收某一个进程的资源，为要回收的子进程的 PID；
       - 0：回收当前进程组的所有子进程 ID；
       - 小于 -1，绝对值表示进程组 ID，表示要回收这个进程组的所有子进程资源；
     - `option`：0，阻塞；`WHOHANG`：非阻塞；
     - 返回值：如果函数是非阻塞的，并且子进程还在运行，返回 0；成功，得到子进程的 PID；失败，-1，没有子进程资源可以回收了，函数如果是阻塞的，阻塞会接触，直接返回 -1，或者回收子进程资源的时候出现了异常了。

### 进程间通信-管道
1. 管道
- 管道的本质是内核中的一块内存（或者叫内核缓冲区），这个缓冲区中的数据存储在一个环形队列中。
- 因为管道在内核中，所以用户空间不能直接对其进行任何操作。
- 管道中数据的特点：
  - 管道对应的内核缓冲区大小是固定的，默认为 4K（也就是队列最大能存储 4K 数据）；
  - 管道分为两个部分：读端和写端（队列的两端），数据从写端进入管道，从读端流出管道；
  - **管道中的数据只能读一次**，做一次读操作后数据就没有了，读数据相当于出队列；
  - **管道是单工的**，数据只能单向流动，数据从写端流向读端；
  - 对管道的操作（读、写）默认是阻塞的；
- 管道的读写。**其实管道操作就是文件 IO 操作，内核中管道的两端分别对应两个文件描述符**，通过写端的文件描述符把数据写入到管道中，通过读端的文件描述符将数据从管道中读出来。
    ```c
    // 读管道
    ssize_t read(int fd, void *buf, size_t count);
    ssize_t write(int fd, const void *buf, size_t count);
    ```
- 假设父进程通过一些列操作可以通过文件描述符表中的文件描述符 fd3 写管道，通过 fd4 读管道，然后再通过 `fork()` 创建出子进程，那么在父进程中被分配的文件描述符 fd3 和 fd4 也就被拷贝到子进程中，子进程通过 fd3 可以将数据写入到内核的管道中，通过 fd4 将数据从管道中读出来。
- 也就是说，**管道是独立于任何进程的，并且充当了两个进程用于数据通信的载体，只要两个进程能够得到同一个管道的入口和出口（读端和写端的文件描述符），那么他们之间就可以通过管道进行数据的交互**。

2. 匿名管道
   - 拥有上述的所有特性，**匿名管道只能实现有血缘关系的进程间通信**；
   - 创建匿名管道：
        ```c
        #include <unistd.h>

        int pipe(int pipefd[2]);
        ```
        - `pipefd`：传输参数，0-读端，1-写端；
        - 返回值：成功返回 0，失败返回 -1；

### 进程间通信-信号
#### 概述
信号是事件发生时对进程的通知机制，也可以把它称为**软件中断**。信号与硬件中断的相似之处在于**能够打断程序当前执行的正常流程**，其实是在软件层次上对中断机制的一种模拟。大多数情况下，是无法预测信号达到的准确事件，所以信号提供了一种**处理异步事件**的方法。

信号的发送：信号的目的是用来通信的，一个具有合适权限的进程能够向另一个进程发送信号。信号的这一用法可作为一种同步技术，甚至是进程间通信（IPC）的原始形式。
   - 硬件发生异常，即硬件检测到错误条件并通知内核，随即再由内核发送相应的信号给相关进程；
   - 在终端上按下 ctrl+c 组合键可以产生中断信号（`SIGINT`），可以用来终止在前台运行的进程（与后台（可通过 `&` 设置）运行相对的运行方式）；按下 ctrl+z 组合键可以产生暂停信号（`SIGTSTP`），可以用来暂停当前前台运行的进程；
   - 调用 kill() 系统调用可以将任意信号发送给另一个进程或进程组，前提是权限允许；
   - 通过 kill 命令将信号发送给其他进程，其内部实现原理也是通过 kill() 系统调用来完成的；
   - 发生软件事件，即当检测到某种软件条件已经发生，例如定时器超时等。

信号的处理：当信号到达后，接收信号的进程需要做出相应的处理。
   - 忽略信号。有两种信号绝不能被忽略：`SIGINT` 和 `SIGSTOP`，用于使进程终止或停止的信号。另外，如果忽略某些由硬件异常产生的信号，则进程的运行行为是未定义的；
   - 捕获信号。当信号到达后执行预先绑定好的信号处理函数-signal() 系统调用用于注册信号处理函数；
   - 系统默认操作。进程不对信号做出处理，而是交由系统处理，每一种信号都有其对应的系统默认处理方式。对大多数信号来说，系统默认的处理方式是终止进程。

信号是异步的。产生信号的事件对进程而言是随机出现的，类似于硬件中断事件。

信号本质上是 int 类型数字编号。定义在 `signum.h` 头文件中，信号编号从 1 开始，编号 0 有着特殊应用。
```c
// <signum.h>

/* Signals */
#define SIGHUP      1       /* Hangup (POSIX) */
#define SIGINT      2       /* Interrupt (ANSI) */
#define SIGQUIT     3       /* Quit (POSIX) */
#define SIGILL      4       /* Illegal instruction (ANSI) */
#define SIGTRAP     5       /* Trace trap (POSIX) */
#define SIGABRT     6       /* Abort (ANSI) */
#define SIGIOT      6       /* IOT trap (4.2 BSD) */
#define SIGBUS      7       /* BUS error (4.2 BSD) */
#define SIGFPE      8       /* Floating-point exception (ANSI) */
#define SIGKILL     9       /* Kill, unblockable (POSIX) */
#define SIGUSR1     10      /* User-defined signal 1 (POSIX) */
#define SIGSEGV     11      /* Segmentation violation (ANSI) */
#define SIGUSR2     12      /* User-defined signal 2 (POSIX) */
#define SIGPIPE     13      /* Broken pipe (POSIX) */
#define SIGALRM     14      /* Alarm clock (POSIX) */
#define SIGTERM     15      /* Termination (ANSI) */
#define SIGSTKFLT   16      /* Stack fault */
#define SIGCLD      SIGCHLD /* Same as SIGCHLD (System V) */
#define SIGCHLD     17      /* Child status has changed (POSIX) */
#define SIGCONT     18      /* Continue (POSIX) */
#define SIGSTOP     19      /* Stop, unblockable (POSIX) */
#define SIGTSTP     20      /* Keyboard stop (POSIX) */
#define SIGTTIN     21      /* Background read from tty (POSIX) */
#define SIGTTOU     22      /* Background write to tty (POSIX) */
#define SIGURG      23      /* Urgent condition on socket (4.2 BSD) */
#define SIGXCPU     24      /* CPU limit exceeded (4.2 BSD) */
#define SIGXFSZ     25      /* File size limit exceeded (4.2 BSD) */
#define SIGVTALRM   26      /* Virtual alarm clock (4.2 BSD) */
#define SIGPROF     27      /* Profiling alarm clock (4.2 BSD) */
#define SIGWINCH    28      /* Window size change (4.3 BSD, sun) */
#define SIGPOLL     SIGIO   /* Pollable event occurred (System V) */
#define SIGIO       29      /* I/O now possible (4.2 BSD) */
#define SIGPWR      30      /* Power failure restart (System V) */
#define SIGSYS      31      /* Bad system call. */
#define SIGUNUSED   31
```

#### 分类
1. 可靠信号和不可靠信号
    UNIX 系统中信号机制的问题：
      - 进程每次处理完信号后，将对信号的响应设置为系统默认操作，导致对信号的错误处理；
      - 信号可能丢失（处理信号时又来了新的信号，则导致信号丢失）。

    UNIX 将可能导致上述问题的信号称为不可靠信号。UNIX 早期定义了 31 种信号。

    Linux 支持不可靠信号，但对不可靠信号进行了改进：在调用完信号处理函数后，不必再次调用 `signal()` 来重新设置。因此，Linux 下的不可靠信号主要指的是信号可能丢失。

    在 Linux 中，信号值小于 `SIGRTMIN (34)` 的信号都是不可靠信号。Linux 3.x 支持 64 种信号。前 31 种信号已有预定义值、用途和名字，并且有对应的系统默认操作。后 32 个信号表示可靠信号。

    可靠信号范围：`SIGRTMIN ~ SIGRTMAX`。可以通过如下命令查看：
    ```bash
    kill -l
    # 输出
    1) SIGHUP ...
    31) SIGSYS  34) SIGRTMIN    35) SIGRTMIN + 1 ...
    63) SIGRTMAX - 1    64) SIGTRMAX
    ```

    **可靠信号支持排队，不会丢失**。同时信号的发送和绑定也出现了新版本：信号发送函数 `sigqueue()` 和信号绑定函数 `sigaction()`。

2. 实时信号和非实时信号
   - 非实时信号都不支持排队，都是不可靠信号；实时信号都支持排队，都是可靠信号。
   - 实时信号保证了发送的多个信号都能被接收。
   - 实时信号是 POSIX 标准的一部分，可用于应用进程。
   - 一般将非实时信号（不可靠信号）称为标准信号。

#### 信号默认行为
| 信号名称 | 编号 | 描述 | 系统默认操作 |
| -----   | ----- | ----- | ----- |
| SIGINT | 2 | 终端终端符（ctrl + c）<br />内核将信号发给前台进程组中的每个进程 | term |
| SIGQUIT | 3 | 终端退出符 (ctrl + \\) <br />内核将信号发送给前台进程组中的每个进程 | term + core |
| SIGILL | 4 | 非法硬件指令 | term + core |
| SIGABRT | 6 | 异常终止 (abort) | term + core |
| SIGBUS | 7 | 总线错误，内存访问错误 | term + core |
| SIGFPE | 8 | 浮点运算异常，算术错误 | term + core |
| SIGKILL | 9 | 强制终止信号 <br /> 此信号无法被进程阻塞、忽略或者捕获 <br /> kill -9 pid 即表示发送此信号 | term |
| SIGUSR1 | 10 | 用户自定义信号 1 | term |
| SIGSEGV | 11 | 无效的内存引用 <br />如 C 中对错误地址（未初始化指针）解引用 | term + core |
| SIGUSR2 | 12 | 用户自定义信号 2 | term |
| SIGPIPE | 13 | 管道关闭 <br />向已经关闭的管道、FIFO 或套接字写入数据 | trem |
| SIGALRM | 14 | 定时器超时 (alarm) <br />与系统调用 alarm() 或 setitimer() 有关 <br />它们用于设置定时器 | term |
| SIGTERM | 15 | 终止进程 <br />用于终止进程的标准信号 <br />也是 kill 命令发送的默认信号 | term |
| SIGCHLD/SIGCLD | 17 | 子进程终止或停止 <br />子进程终止、停止或恢复时，内核向父进程发送此信号 | ignore |
| SIGCONT | 18 | 使停止状态的进程继续运行 <br />接收到此信号的进程不为停止状态则默认忽略 | cont |
| SIGSTOP | 19 | 停止进程 <br />应用程序无法忽略或捕获，故总能停止进程 | stop |
| SIGTSTP | 20 | 终端停止符 (ctrl + z) <br />系统将信号发送给前台进程组的每个进程 | stop |
| SIGXCPU | 24 | 超过 CPU 限制 <br />进程的 CPU 时间超出对应的资源限制 | term + core |
| SIGVTALRM | 26 | 虚拟定时器超时 <br />应用程序调用 setitimer() 设置一个虚拟定时器，<br />定时时间到后内核发送此信号给该进程 | term |
| SIGWINCH | 28 | 终端窗口尺寸变化 | ignore |
| SIGPOLL/SIGIO | 29 | 异步 I/O <br />用于提示一个异步 IO 事件的发生 | term/ignore |
| SIGSYS | 31 | 无效系统调用 | term + core |
- term-终止进程；
- core-生成核心转储文件；
- ignore-忽略信号；
- cont-继续运行进程；
- stop-停止进程（不等于终止，相当于暂停的意思）。

#### 进程对信号的处理
处理方式：
- 忽略信号；
- 捕获信号；
- 执行系统默认操作；

如果程序中没有设置信号的处理方式，或者程序刚启动还并没有运行至信号处理设置函数处，此时程序对信号的处理：
- 程序启动：通常情况下，进程对所有信号的处理方式都设置为系统默认操作；
- 进程创建：进程调用 `fork()` 创建子进程，子进程会继承父进程的信号处理方式。由于子进程复制了父进程的内存映像，所以信号捕获函数的地址在子进程中是有意义的。

一般而言，将信号处理函数设计越简单越好，这就好比中断处理函数越快越好，不要在处理函数中做大量消耗 CPU 时间的事情，一个重要的原因在于，设计得越简单这将降低引发信号竞争条件的风险。

Linux 系统调用 `signal()` 和 `sigaction()` 用于设置信号的处理方式：
```c
// signal
#include <signal.h>

// int 参数-当前触发该函数的信号
// 可以将多个信号绑定到同一个信号处理函数上
typedef void (*sig_t)(int);

/** 
 * @signum: 需要进行设置的信号
 * @handler: 对应的信号处理函数
 *          可以设置为用户自定义函数，也可以设置为：
 *          SIG_IGN-进程需要忽略该信号；
 *          SIG_DFL-设置为系统默认操作；
 * @return: 成功-此前的信号处理函数；出错-SIG_ERR，并设置 errno
 */
sig_t signal(int signum, sig_t handler);

#define SIG_ERR     ((sig_t) -1)        // Error return
#define SIG_DFL     ((sig_t) 0)         // Default action
#define SIG_IGN     ((sig_t) 1)         // Ignore signal

// sigaction
#include <signal.h>
/** 
 * @signum: 需要设置的信号，除了 SIGKILL 和 SIGSTOP 之外的任何信号
 * @act: 描述信号的处理方式。不为 NULL 时，表示设置新的处理方式；为 NULL 时，表示无需改变信号当前的处理方式；
 * @oldact: 不为 NULL 时，会将信号之前的处理方式等信息通过 oldact 返回出来；为 NULL 则表示不需要获取该信息
 * @return: 0-成功；-1-失败，并设置 errno
 */
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

/**
 * @sa_handler: 信号处理函数
 * @sa_sigaction: 也是信号处理函数，它提供了更多的参数，可以通过该函
 * 数获取到更多信息，这些信息通过 siginfo_t 参数获取。
 * sa_sigaction 和 sa_handler 是互斥的，不能同时设置。对于标准信号，
 * 使用 sa_handler 就可以了，可以通过 SA_SIGINFO 标志进行选择。
 * @sa_mask: 信号掩码。处理函数执行前置位相应信号，执行后再恢复。
 * 信号到达时，如果它的掩码已被置位则它会被阻塞，直到置位被取消。可用于防止处理函数被其他信号打断。
 * 信号处理函数被调用时，对应信号的掩码置位操作被自动执行。
 * 信号掩码可避免一些信号之间的竞争状态。
 * @sa_restore: 该成员已过时。
 * @sa_flags: 指定一组标志，这些标志用于控制信号的处理过程：
 *          SA_NOCLDSTOP: 如果 signum 为 SIGCHLD，则子进程停止时
 * （SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU）或恢复（SIGCONT）时不会收到 SIGCHLD 信号；
 *          SA_NOCLDWAIT: 如果 signum 为 SIGCHLD，则在子进程终止时不要将其转变为僵尸进程；
 *          SA_NODEFER: 信号处理函数执行时，不阻塞对应信号；
 *          SA_RESETHAND: 执行完信号处理函数后，将信号的处理方式设置为系统默认操作；
 *          SA_RESTART: 被信号中断的系统调用，在信号处理完成之后将自动重新发起；
 *          SA_SIGINFO: 当设置此标志，则表示使用 sa_sigaction 作为信号处理函数，而不是 sa_handler。
 */
struct sigaction {
    void (*sa_handler)(int);
    void (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t sa_mask;
    int sa_flags;
    void (*sa_restore)(void);
};

// 通过 man sigaction() 可获取相关信息
siginfo_t {
    int si_signo;       // Signal number
    int si_errno;       // An errno value
    int si_code;        // Signal code
    ...
}
```

#### 向进程发送信号
Linux 提供的系统调用：kill(), killpg()；库函数：raise()，可用于信号的发送。
```c
// kill
#include <sys/types.h>
#include <signal.h>

// 对于普通用户进程来说，基本规则是发送者进程的实际用户 ID 或有效用户 ID
// 必须等于接收者进程的实际用户 ID 或有效用户 ID。
/** 
 * @pid: 正数-信号发送到指定的进程；
 *       0-信号发送到当前进程的进程组中的每个进程；
 *       等于 -1-信号发送到当前进程有权发送信号的每个进程，但 1(init) 除外；
 *       小于 -1-信号发送到 ID 为 -pid 的进程组中的每个进程。
 * @sig: 如果 sig 设置为 0，则表示不发送信号，但仍执行错误检查
 * 这通常可用于检查参数 pid 指定的进程是否存在。不存在时，返回 -1 
 * 并设置 errno 为 ESRCH，表示进程不存在。
 * @return: 0-成功；-1-失败，并设置 errno
 */ 
int kill(pid_t pid, int sig);

// raise
#include <signal.h>

// 用于进程向自身发送信号
// 等价于 kill(getpid(), sig)
// @return: 0-成功；非零值-失败
int raise(int sig);
```

alarm() 和 pause() 系统调用。
```c
// alarm
#include <unistd.h>

/**
 * 用于设置一个定时器，当定时器时间到达时，内核向进程发送 SIGALRM 信号
 * 每个进程只能设置一个 alarm 闹钟
 * alarm 闹钟并不能循环触发，只能触发一次，可在处理函数中再次调用实现循环
 * @seconds: 如果为 0，则表示取消之前设置的 alarm 闹钟
 * @return: 如果在调用 alarm() 时，之前已经为该进程设置了 alarm 闹钟
 *          且还没有超时，则该闹钟的剩余值作为本次 alarm() 调用的返回
 *          值，之前的闹钟则被替代；否则返回 0
 */
unsigned int alarm(unsigned int seconds);

// pause
/**
 * 使得进程暂停运行，进入休眠状态，直到进程捕获到一个信号为止，只有执行
 * 了信号处理函数并从其中返回，pause() 才返回。在这种情况下，pause() 
 * 返回 -1，并将 errno 设置为 EINTR。
 */
#include <unistd.h>

int pause(void);

// 示例程序：通过 alarm 和 pause 模拟 sleep
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

static void sig_handler(int sig) {
    puts("Alarm timeout");
}

int main(int argc, char *argv[]) {
    int second;
    struct sigaction sig = {0};

    if (2 > argc)
        exit(-1);

    sig.sa_handler = sig_handler;
    sig.sa_flags = 0;
    if (-1 == sigaction(SIGALRM, &sig, NULL)) {
        perror("sigaction error");
        exit(-1);
    }

    second = atoi(argv[1]);
    printf("定时时长：%d 秒\n", second);
    alarm(second);

    pause();
    puts("休眠结束");

    exit(0);
}
```

#### 信号集
信号集就是 sigset_t 类型数据：
```c
#define _SIGSET_NWORDS  (1024 / (8 * sizeof(unsigned long int)))

typedef struct {
    unsigned long int __val[_SIGSET_NWORDS];
} sigset_t;

// 初始化信号集
#include <signal.h>
// 0-成功；-1-失败，并设置 errno
// 初始化为空信号集
int sigemptyset(sigset_t *set);
// 初始化信号集，使其包含所有信号
int sigfillset(sigset_t *set);

// 向信号集中添加/删除信号
#include <signal.h>
// 0-成功；-1-失败，并设置 errno
int sigaddset(sigset_t *set, int signum);
int sigdelset(sigset_t *set, int signum);

// 测试信号是否在信号集中
#include <signal.h>
// 1-在；0-不在；-1-失败，并设置 errno
int sigismember(const sigset_t *set, int signum);
```

#### 获取信号描述信息
信号描述信息位于 sys_siglist 数组中，类型为 char *。
```c
#include <signal.h>
#include <...>

int main(void) {
    printf("SIGINT 的描述信息：%s\n", sys_siglist[SIGINT]);
    ...
}
```

strsignal() 函数
```c
#include <string.h>

// 若传入的 sig 无效，则返回 "Unknown signal"。
char *stsignal(int sig);
```

psignal() 函数
```c
#include <signal.h>
// psignal 可以在标准错误（stderr）上输出信号描述信息
// psignal 会将参数 sig 指定的信号的描述信息输出到标准错误，并且允许
// 调用者添加一些输出信息，由参数 s 指定
void psignal(int sig, const char *s);

// eg:
psignal(SIGINT, "SIGINT 信号描述信息");
// out: SIGINT 信号描述信息：Interrupt
```

#### 信号掩码（阻塞信号传递）
内核为每个进程维护一个信号掩码（信号集），当进程接收到一个属于信号掩码中定义的信号时，该信号将会被阻塞、无法传递给进程进行处理，直到该信号从掩码中移除，内核才会把该信号传递给进程从而得到处理。

向信号掩码中添加一个信号的方式：
- 当应用程序调用 signal() 或 sigaction() （未设置 SA_NODEFER 标志）注册的处理函数时，进程会自动将对应信号添加到信号掩码中。当信号处理函数返回后，自动将对应信号从掩码中移除；
- 使用 sigaction() 函数为信号设置处理函数时，可以额外设置一组信号（通过 sa_mask 进行设置），当处理函数被调用时将该组信号自动添加到掩码中，处理结束后从掩码中移除；
- 使用 sigprocmask() 系统调用，随时可以显式地向信号掩码中添加/移除信号：
    ```c
    #include <signal.h>

    /**
     * @how:
     *      SIG_BLOCK: 将参数 set 所指向的信号集内的所有信号添加到进程的信号掩码中；
     *      SIG_UNBLOCK: 将参数 set 所指向的信号集内的所有信号从进程信号掩码中移除；
     *      SIG_SETMASK: 进程信号掩码直接设置为参数 set 所指向的信号集。
     * @set: 将参数 set 指向的信号集内的所有信号添加到信号掩码中或者从中移除；如果 set 为 NULL，则表示无需对当前信号掩码作出改动。
     * @oldset: 获取之前的信号掩码，如果为 NULL 则表示不获取。
     * @return: 0-成功；-1-失败，并设置 errno
     */
    int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
    ```

#### 阻塞等待信号（sigsuspend()）
更改进程的信号掩码可以阻塞所选择的信号，或解除对它们的阻塞。使用这种技术可以保护希望不被信号中断的关键代码段。

可能存在这样的需求：对一个信号解除阻塞后，然后调用 pause() 以等待之前被阻塞的信号的传递。

```c
sigset_t new_set, old_set;

sigempty(&new_set);
sigaddset(&new_set, SIGINT);

if (-1 == sigprocmask(SIG_BLOCK, &new_set, &old_set))
    exit(-1);
// 受保护的关键代码段
// ...
if (-1 == sigprocmask(SIG_SETMASK, &old_set, NULL))
    exit(-1);

// 可能在这里收到信号

pause();        // 等待信号唤醒
```
上述问题可通过 sigsuspend() 解决：
```c
#include <signal.h>

// 将参数 mask 指向的信号集替换进程的信号掩码，然后挂起进程，直到捕获到信号被唤醒，并从信号处理函数中返回。一旦从处理函数返回，sigsuspend() 会将进程的信号掩码恢复成调用前的值。
// 如果信号是 mask 中的成员，将不会唤醒，而是继续挂起。
// sigsuspend() 相当于以不可中断的方式执行以下操作：
//      sigprocmask(SIG_SETMASK, &mask, &old_mask);
//      pause();
//      sigprocmask(SIG_SETMASK, &old_mask, NULL);
// 返回值：始终返回 -1，并设置 errno 来指示错误（通常为 EINTR），表示被信号打断；如果调用失败，将 errno 设置为 EFAULT。
int sigsuspend(const sigset_t *mask);
```
示例程序：
```c
#include <signal.h>
#include <...>

static void sig_handler(int sig) {...}

int main(void) {
    struct sigaction sig = {0};
    sigset_t new_mask, old_mask, wait_mask;

    sigemptyset(&new_mask);
    sigaddset(&new_mask, SIGINT);
    sigemptyset(&wait_mask);

    sig.sa_handler = sig_handler;
    sig.sa_flags   = 0;
    if (-1 == sigaction(SIGINT, &sig, NULL))
        exit(-1);

    if (-1 == sigprocmask(SIG_BLOCK, &new_mask, &old_mask))
        exit(-1);
    // 保护的代码段
    // ...
    if (-1 == sigsuspend(&wait_mask))
        exit(-1);
    if (-1 == sigprocmask(SIG_SETMASK, &old_mask, NULL))
        exit(-1);

    exit(0);
}
```
#### 实时信号
如果进程当前正在执行信号处理函数，此时收到了新的信号，如果该信号是信号掩码中的成员，那么内核会将其阻塞，将该信号添加到进程的等待信号集中。为了确定进程中处于等待状态的是哪些信号，可以使用 sigpending() 函数获取。
```c
#include <signal.h>

// 处于等待状态的信号会通过 set 返回
// 0-成功；-1-失败，并设置 errno
int sigpending(sigset_t *set);
```

等待信号集指示一个掩码，仅表明一个信号是否发生，而不能表示其发生的次数。

实时信号相较于标准信号的优势：
- 实时信号的信号范围更大，可应用于应用程序自定义的目的，而标准信号仅提供了两个信号可用于应用程序自定义使用：SIGUSR1 和 SIGUSR2；
- 内核对实时信号采用的是队列化管理。如果将某一实时信号多次发送给另一个进程，那么将会多次传递此信号；
- 当发送实时信号时，可为信号指定伴随数据（整型数据或指针值），供接收方在处理函数中获取；
- 不同实时信号的传递顺序得到保障。如果有多个不同的实时信号处于等待状态，那么将率先传递具有最小编号的信号。即信号的编号越小，优先级越高。如果是同一类型的多个信号在排队，那么信号的传递顺序与接收时的顺序一致。

实时信号范围：34~64，`SIGRTMIN~SIGRTMAX`。

应用程序使用实时信号的要求：
- 发送进程使用 sigqueue() 系统调用向另一个进程发送实时信号以及伴随数据；
- 接收实时信号的进程要为该信号使用 sigaction 函数为信号建立处理函数，并加入 SA_SIGINFO，这样信号处理函数才能够接收到实时信号以及伴随数据。

使用 sigqueue() 发送实时信号：
```c
#include <signal.h>

/**
 * @pid: 接收信号进程的 pid
 * @sig: 需要发送的信号，也可以将 sig 设置为 0，用于检查 pid 指定进程是否存在
 * @value: 伴随数据
 * @return: 0-成功；-1-失败，并设置 errno
 */
int sigqueue(pid_t pid, int sig, const union sigval value);

// 伴随数据可以指定一个整型数据，也可以指定一个指针
typedef union sigval {
    int sival_int;
    void *sival_ptr;
} sigval_t;

// 示例程序

// 发送
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

int main(int argc, char *argv[]) {
    union sigval sig_val;
    int pid;
    int sig;

    if (3 > argc)
        exit(-1);

    pid = atoi(argv[1]);
    sig = atoi(argv[2]);
    printf("pid: %d, signal: %d.\n", pid, sig);

    sig_val.sival_int = 10;
    if (-1 == sigqueue(pid, sig, sig_val)) {
        perror("sigqueue error");
        exit(-1);
    }

    puts("信号发送成功！");
    exit(0);
}

// 接收
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

static void sig_handler(int sig, siginfo_t *info, void *context) {
    sigval_t sig_val = info->si_value;

    printf("接收到实时信号：%d.\n", sig);
    printf("伴随数据为：%d.\n", sig_val.sival_int);
}

int main(int argc, char *argv[]) {
    int num;
    struct sigaction sig = {0};

    if (2 > argc)
        exit(-1);

    num = atoi(argv[1]);
    sig.sa_sigaction = sig_handler;
    sig.sa_flats = SA_SIGINFO;
    if (-1 == sigaction(num, &sig, NULL)) {
        perror("sigaction error");
        exit(-1);
    }

    for (;;)
        sleep(1);
    
    exit(0);
}
```

#### 退出函数
1. abort() 函数
    对于异常退出程序，一般使用 abort() 库函数。使用 abort() 终止进程运行，会生成核心转储文件，可用于判断程序调用 abort() 时的状态。
    ```c
    #include <stdlib.h>

    void abort(void);
    ```
    函数 abort() 通常产生 SIGABRT 信号来终止调用该函数的进程。当 abort() 调用之后，内核会向进程发送 SIGABRT 信号。

    即使在程序中捕获了 SIGABRT 信号，仍然会在收到该信号后终止（先调用处理函数）并生成核心转储文件（无论阻塞或忽略都不影响）。

## 多线程编程
### 线程概念
   所谓线程，就是操作系统所能调度的最小单位。
### 特点
   - 普通的进程，只有一个线程在执行对应的逻辑。可以通过多线程编程，使一个进程可以去执行多个不同的任务。
   - 相比多进程编程而言，线程享有共享资源，即在进程中出现的全局变量，每个线程都可以去访问它，与进程共享“4G”内存空间，使得系统资源消耗减少。
### Linux 下 POSIX 线程
1. 线程的标识 `pthread_t`
   - 对于进程而言，每一个进程都有一个唯一对应的 PID 号来表示该进程。对于于线程而言，通过 TID 来表示，其本质上是一个 `pthread_t` 类型的变量。
   - 线程号与进程号是表示线程和进程的唯一标识。
   - **对于线程号而言，其仅仅在其所属的进程上下文中才有意义**。
2. 获取当前线程的线程号
   ```c
   #include <pthread.h>

   pthread_t pthread_self(void);    // 成功时返回当前线程号
   ```
3. 编译选项
   链接到 pthread 库
   ```bash
   gcc pthread_test1.c -lpthread
   ```

### 线程的创建
   ```c
   #include <pthread.h>

   int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine), void *arg);
   ```
   - `thread`：传出参数，用于保存新建线程的线程号；
   - `attr`：线程的属性，一般传入 NULL 表示默认属性；
   - `start_routine`：函数指针，即线程需要执行的函数；
   - `arg`：向线程处理函数传入的参数，若不传入，可用 NULL 填充。

    示例程序：
    ```c
    #include <pthread.h>
    #include <stdio.h>
    #include <unistd.h>
    #include <errno.h>

    void *fun(void *arg) {
        printf("pthread_new = %lu\n", (unsigned long)pthread_self());
    }

    int main() {
        pthread_t tid1;
        int ret = pthread_create(&tid, NULL, fun, NULL);
        if (ret != 0) {
            perror("pthread_create");
            return -1;
        }

        printf("tid_main = %lu, tid_new = %lu\n", (unsigned long)pthread_self(), (unsigned long)tid1);

        // 因线程执行顺序随机，不加 sleep 可能导致主线程先执行，导致进程结束，无法执行到子线程
        sleep(1);
        return 0;
    }
    ```
    - **当主线程伴随进程结束时，所创建出来的线程也会立即结束，不会继续执行**。
    - **创建出来的线程的执行顺序是随机竞争的，并不能保证哪一个线程会先运行**。

### 线程传参
   ```c
   #include <pthread.h>
   #include <stdio.h>
   #include <unistd.h>
   #include <error.h>

   void *fun1(void *arg){
    while (1) {
        printf("%s: arg = %d, addr = %p\n", __FUNCTION__, *(int *)arg, arg);
        sleep(1);
    }
   }

   void *fun2(void *arg) {
    while (1) {
        printf("%s: arg = %d, addr = %p\n", __FUNCTION__, (int)(long)arg, arg);
        sleep(1);
    }
   }

   int main() {
    int a = 50;
    pthread_t tid, tid2;

    int ret = pthread_create(&tid, NULL, fun1, (void*)&a);  // 传地址
    if (ret != 0) {
        perror("pthread_create");
        return -1;
    }
    sleep(1);

    ret = pthread_create(&tid2, NULL  fun2, (void *)(long)a);   // 传值
    if (ret != 0){
        perror("pthread_create");
        return -1;
    }

    while (1) {
        a++;
        sleep(1);
        printf("%s: a = %d, addr = %p\n", __FUNCTION__, a, &a);
    }

    return 0;
   }
   ```
   - 注意传值和传地址的区别。
   - **针对不同位数机器，指针对应的字数不同，需要 `int` 转换为 `long` 再转指针，否则可能会发生警告**。
   - 在实际项目中，往往会遇到传递多个参数的问题，我们可以通过结构体来进程传递以解决此问题。

### 线程的退出与回收
1. 线程退出的三种情况：
   - 进程结束，进程中所有的线程也会随之结束；
   - 主动退出线程：通过 `pthread_exit()` 函数；
   - 被动退出线程：其他线程通过调用 `pthread_cancel()` 函数来被动退出；
2. 线程主动退出
   ```c
   #include <pthread.h>

   void pthread_exit(void *retval);
   ```
   - `retval`：线程在退出时可以传递一个 `void *` 类型的数据带给主线程，若不选择传出数据，可将参数填充为 NULL。
3. 线程被动退出
   ```c
   #include <pthread.h>

   int pthread_cancel(pthread_t thread);
   ```
   - 该函数传入一个 TID 号，会强制退出该 TID 所指向的线程，若成功执行会返回 0；
4. 线程资源回收
    ```c
    #include <pthread.h>

    int pthread_join(pthread_t thread, void **retval);
    ```
    - 线程回收函数，**默认为阻塞状态**，直到成功回收线程后才返回；
    - `tid`：需要回收线程的 TID 号；
    - `retval`：线程回收后接收线程传出的数据；

    ```c
    #define _GNU_SOURCE
    #include <pthread.h>

    int pthread_tryjoin_np(pthread_t thread, void **retval);
    ```
    - **非阻塞模式回收函数，通过返回值判断是否回收掉线程，成功则返回 0**；

- [ ] 上述函数的线程安全性如何？

5. 示例程序
   - 阻塞式回收线程
        ```c
        #include <pthread.h>
        #include <stdio.h>
        #include <unistd.h>
        #include <errno.h>

        void *fun1(void *arg) {
            static int temp = 0;
            temp = *(int *)arg;
            temp += 100;
            printf("%s: addr = %p, tmp = %d", __FUNCTION__, &temp, temp);
            pthread_exit((void *)&temp);
        }

        int main() {
            pthread_t tid1;
            int a = 50;
            void *temp = NULL;
            int ret = pthread_create(&tid1, NULL, fun1, (void*)&a);
            if (ret != 0) {
                perror("pthread_create");
                return -1;
            }
            pthread_join(tid1, &temp);
            printf("%s: addr = %p, val = %d\n", __FUNCTION__, temp, *(int *)temp);
            return 0;
        }
        ```
        - **当变量从线程传出时，需要加 `static` 修饰，对生命周期做出延续，否则无法传出正确的变量值（`pthread_join` 无法获取到正确值）**；
   - 非阻塞方式回收线程
        ```c
        #define _GNU_SOURCE
        #include <pthread.h>
        #include <stdio.h>
        #include <unistd.h>
        #include <error.h>

        void *fun(void *arg) {
            printf("pthread: %d come!\n", (int)(long)(arg + 1));
            pthread_exit(arg);
        }

        int main() {
            int ret, i, flag = 0;
            void *temp = NULL;
            pthread_t tid[3];
            for (i = 0; i < 3; i++) {
                ret = pthread_create(&tid[i], NULL, fun, (void *)(long)i);
                if (ret != 0) {
                    perror("pthread_create");
                    return -1;
                }
            }
            while (1) {
                for (i = 0; i < 3; i++) {
                    if (pthread_tryjoin_np(tid[i], &temp) == 0) {
                        printf("pthrad: %d exit!\n", (int)(long)temp + 1);
                        flag++;
                    }
                }
                if (flag >= 3) break;
            }
            return 0;
        }
        ```
        - 多个线程可以指向同一个回调函数（可重入函数）；
        - 这里通过 `pthread_exit()` 传出的参数是位于 `main` 线程中申请的资源，所以不需要使用 `static` 来修饰；
        - **相比阻塞方式回收线程**：阻塞方式回收线程几乎规定了线程的回收顺序，若最先回收的线程未及时退出，则一直会被阻塞，导致后续先退出的线程无法及时回收；使用非阻塞回收，线程可以根据退出先后顺序自由地进行资源回收；
   - 通过被动方式退出线程并进行回收
        ```c
        #define _GUN_SOURCE
        #include <pthread.h>
        #include <stdio.h>
        #include <unistd.h>
        #include <error.h>

        void *fun1(void *arg) {
            printf("pthread: 1 come!"\n);
            while (1) {
                sleep(1);
            }
        }

        void *fun2(void *arg) {
            printf("pthread: 2 come!\n");
            pthread_cancel((pthread_t)(long)arg);   // 杀死线程 1,使之强制退出
            pthread_exit(NULL);
        }

        int main() {
            int ret, i, flag = 0;
            void *temp = NULL;
            pthread_t tid[2];
            ret = pthread_create(&tid[0], NULL, fun1, NULL);
            if (ret != 0) {
                perror("pthread_create");
                return -1;
            }
            sleep(1);

            ret = pthread_create(&tid[1], NULL, fun2, (void*)tid[0]);
            if (ret != 0) {
                perror("pthread_create");
                return -1;
            }

            while (1) {
                for (i = 0; i < 2; ++i) {
                    if (pthread_tryjoin_np(tid[i], NULL) == 0) {
                        printf("pthread: %d exit!\n", i + 1);
                        flag++;
                    }
                }
                if (flag >= 2) break;
            }
            return 0;
        }
        ```
        - 通过第二个线程来结束处于死循环睡眠逻辑的第一个线程；
        - 一定要确保第一个线程先执行（通过在第一个和第二个线程之间添加睡眠操作）；

### 同步与互斥
解决临界资源访问的竞争问题。

#### 互斥锁
1. 初始化互斥量
    ```c
    int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *restrict attr);
    ```
    - `mutex`：指向需初始化的互斥量；
    - `attr`：控制互斥量的属性，一般为 NULL；
    - 返回值：当函数成功后返回 0，代表初始化互斥量成功；

    通过宏来快速初始化：
    ```c
    pthread_mutex_t mutex = PTHREAD_MUXTE_INITIALIZER;
    ```
2. 互斥量加锁/解锁
    ```c
    #include <pthread.h>

    int pthread_mutex_lock(pthread_mutex_t *mutex);
    int pthread_mutex_unlock(pthread_mutex_t *mutex);
    ```
    - 成功返回 0；
    - `lock` 获取锁或者阻塞当前线程；`unlock` 释放锁并唤醒其他正在等待互斥量的线程；
    - 特别注意的是，当获取 `lock` 之后，必须在逻辑处理结束后执行 `unlock`，否则会发生死锁现象，导致其他线程一直处于阻塞状态，无法执行下去。在使用互斥量的时候，尤其要注意使用 `pthread_cancel()` 函数，防止发生死锁现象。

    ```c
    #include <pthread.h>

    int pthread_mutex_trylock(pthread_mutex_t *mutex);
    ```
    - 非阻塞式加锁；
    - 通过返回值来判断是否加锁成功，用法与上述非阻塞式加锁函数一致；
3. 销毁互斥量
    ```c
    #include <pthread.h>

    int pthread_mutex_destroy(pthread_mutex_t *mutex);
    ```
    - 成功返回 0；
4. 示例程序
   - 简单示例
    ```c
    #define _GNU_SOURCE
    #include <pthread.h>
    #include <stdio.h>
    #include <unistd.h>
    #include <errno.h>

    pthread_mutex_t mutex;

    int num = 0;    // 公共临界资源

    void *fun1(void *arg) {
        pthread_mutex_lock(&mutex);
        while (num < 3) {
            num++;
            printf("%s: num = %d", __FUNCTION__, num);
            sleep(1);
        }
        pthread_mutex_unlock(&mutex);
        pthread_exit(NULL);
    }

    void *fun2(void *arg) {
        pthread_mutex_lock(&mutex);
        while (num > -3) {
            num--;
            printf("%s: num = %d\n", __FUNCTION__, num);
            sleep(1);
        }
        pthread_mutex_unlock(&mutex);
        pthread_exit(NULL);
    }

    int main() {
        int ret;
        pthread tid1, tid2;
        ret = pthread_mutex_init(&mutex, NULL);
        if (ret != 0) {
            perror("pthread_mutex_init");
            return -1;
        }

        ret = pthread_create(&tid1, NULL, fun1, NULL);
        if (ret != 0) {
            perror("pthread_create");
            return -1;
        }

        ret = pthread_create(&tid2, NULL, fun2, NULL);
        if (ret != 0) {
            perror("pthread_create");
            return -1;
        }

        pthread_join(tid1, NULL);
        pthread_join(tid2, NULL);
        pthread_mutex_destory(&mutex);
        return 0;
    }
    ```
    - **互斥量一般申请为全局变量**；
    - 需要说明的是，线程加锁在循环内与循环外的情况。上述示例在进入 while 循环前进行了加锁操作，在循环结束后进行了解锁操作。**如果将加锁和解锁操作全部放入 while 循环内，作为单核的机器，执行解锁无异；当有多核机器执行代码时，可能会发生“枪锁”现象，这取决于操作系统底层的实现**。

#### 信号量
多线程执行顺序控制。

1. 初始化信号量
   ```c
   int sem_init(sem_t *sem, int pshared, unsigned int value);
   ```
   - `sem`：需要初始化的信号量指针；
   - `pshared`：传入 0 表示同一进程内的线程共享该信号量（`man` 手册，and should be located at some address that is visible to all threads (e.g., a global variable, or a variable allocated dynamically on the heap)），否则该信号量在进程间共享（and should be located int a region of shared memory）；
   - `value`：信号量的初始值（0 表示阻塞，1 代表运行）；
   - Initializing a semaphore that has already been initialized results in undefined behavior；
   - 返回值：成功返回 0，失败返回 -1；
2. 信号量 P/V 操作
   ```c
   #include <pthread.h>

   int sem_wait(sem_t *sem);
   int sem_post(sem_t *sem);
   ```
   - `sem_wait()`：
     - **P 操作**，检测指定信号量是否有资源可用，若无资源可用会阻塞等待，若有资源可用会自动地执行 `sem - 1` 操作。成功返回 0；
     - `sem_wait()` decrements (locks) the semaphore pointed to by `sem`. If the semaphore's value is greater than zero, then the decrement proceeds, and the function returns, immediately. If the semaphore currently has the value zero, then the call blocks until either it becomes possible to perform the decrement (i.e., the semaphore value rises above zero), or a signal handler interrupts the call.
   - `sem_post()`：
     - **V 操作**，释放指定信号量的资源，执行 `sem + 1` 操作。成功返回 0；
     - `sem_post()` increments (unlocks) the semaphore pointed by `sem`. If the semaphore's value consequently becomes greater than zero, the another process or thread blocked in a `sem_wait(3)` call will be woken up and proceed to lock the semaphore.

    ```c
    #include <pthread.h>

    int sem_trywait(sem_t *sem);
    ```
    - 非阻塞式进行信号量申请；
3. 销毁信号量
   ```c
   #include <pthread.h>

   int sem_destroy(sem_t *sem);
   ```
   - destroying a semaphore that other processes or threads are currently blocked on produces underfined behavior.
   - using a semaphore that has been destoryed produces undefined results, until the semaphore has been reinitialized using `sem_init(3)`.
   - return 0 on success; on error, -1 is returned.
4. 示例程序
   ```c
   #define _GNU_SOURCE
   #include <pthread.h>
   #include <stdio.h>
   #include <unistd.h>
   #include <errno.h>
   #include <semephore.h>

   sem_t sem1, sem2, sem3;

   void *fun1(void *arg) {
    sem_wait(&sem1);
    printf("%s: pthread come!\n", __FUNCTION__);
    sem_post(&sem2);
    pthread_exit(NULL);
   }

   void *fun2(void *arg) {
    sem_wait(&sem2);
    printf("%s: pthread come!\n", __FUNCTION__);
    sem_post(&sem3);
    pthread_exit(NULL);
   }

   void *fun3(void *arg) {
    sem_wait(&sem3);
    printf("%s: pthread come!\n", __FUNCTION__);
    sem_post(&sem1);
    pthread_exit(NULL);
   }

   int main() {
    int ret;
    pthread_t tid1, tid2, tid3;
    ret = sem_init(&sem1, 0, 1);
    if (ret < 0) {
        perror("sem init");
        return -1;
    }
    ret = sem_init(&sem2, 0, 0);
    if (ret < 0) {
        // ...
    }
    ret = sem_init(&sem3, 0, 0);
    if (ret < 0) {
        // ...
    }

    ret = pthread_create(&tid1, NULL, fun1, NULL);
    if (ret != 0) {
        perror("pthread create");
        return -1;
    }
    ret = pthread_create(&tid2, NULL, fun2, NULL);
    if (ret != 0) {
        // ...
    }
    ret = pthread_create(&tid3, NULL, fun3, NULL);
    if (ret != 0) {
        // ...
    }

    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
    pthread_join(tid3, NULL);

    sem_destroy(&sem1);
    sem_destroy(&sem2);
    sem_destroy(&sem3);

    return 0;
   }
   ```

#### 条件变量
条件变量是一种同步机制，用来通知其他线程条件满足了。一般用来通知对方共享数据的状态信息，因此条件变量是**结合互斥量来使用的**。

1. 创建和销毁条件变量
   ```c
   #include <pthread.h>

   // 初始化条件变量
   pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
   int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *cond_attr);  // cond_attr 通常为 NULL
   
   // 销毁条件变量
   int pthread_cond_destroy(pthread_cond_t *cond);
   ```
   - 上述函数调用成功时都返回 0；
2. 等待条件变量
   ```c
   int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
   ```
   
   **条件变量需要与互斥量配合使用**，示例程序：
   ```c
   pthread_mutex_lock(&g_tMutex);
   pthread_cond_wait(&g_tConVar, &g_tMutex);
   /* 操作临界资源 */
   pthread_mutex_unlock(&g_tMutex);
   ```
   - **在获取到互斥锁的情况下，再等待条件变量**；
   - **如果条件变量不满足，则会 `unlock` 互斥锁**（`pthread_cond_wait()`）；
   - **条件满足后被唤醒，会 `lock` 互斥锁**；
3. 通知条件变量
   > [参考](https://linux.die.net/man/3/pthread_cond_signal)
   ```c
   int pthread_cond_broadcast(pthread_cond_t *cond);
   int pthread_cond_signal(pthread_cond_t *cond);
   ```
   - `pthread_cond_broadcast()`: this function shall unblock all threads currently blocked on the specified condition variable cond.
   - `pthread_cond_signal()`: this function shall unblock at least one of the threads that are blocked on the specified condition variable cond (if any threads are blocked on cond).
   - **同样也需要先获取互斥锁再通知被当前条件变量阻塞的线程**，以确保线程安全，[其他情况](https://duoke360.com/post/33768)。

## 相关概念
1. 可重入性

## 应用示例
### 生产者-消费者模式
### 线程池

## 参考
1. 韦东山-《嵌入式 Linux 应用开发完全手册 V5.2》第 9 章；
2. [爱编程的大丙](https://subingwen.cn/linux/#%E7%AC%AC3%E7%AB%A0-%E8%BF%9B%E7%A8%8B%E5%92%8C%E7%BA%BF%E7%A8%8B)
3. 正点原子-《I.MX6U 嵌入式 Linux C 应用编程指南》