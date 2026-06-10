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