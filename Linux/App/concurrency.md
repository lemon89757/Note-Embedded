# 并发编程
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