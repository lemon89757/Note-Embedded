# 完全公平调度类
> 《深入 Linux 内核架构》-第二章 进程管理和调度

完全公平调度类，即 *completely fair scheduling, cfs*

## CFS 就绪队列
主调度器的每个就绪队列中嵌入了一个该结构的实例。

```c
// version: 4.1.15
// kernel/sched/sched.h
/* CFS-related fields in a runqueue */
struct cfs_rq {
    struct load_weight load;
    unsigned int nr_running, h_nr_running;

    u64 exec_clock;
    u64 min_vruntime;
#ifndef CONFIG_64BIT
    u64 min_vruntime_copy;
#endif

    struct rb_root tasks_timeline;
    struct rb_node *rb_leftmost;

    /* 
     * 'curr' points to currently running entity on this cfs_rq.
     * It is set to NULL otherwise (i.e when none are currently running).
     */
    struct sched_entity *curr, *next, *last, *skip;

#ifdef CONFIG_SCHED_DEBUG
    unsigned int nr_spread_over;
#endif

#ifdef CONFIG_SMP
    /* 
     * CPU load tracking 
     * Under CFS, load is tracked on a per-entity basis and aggregated up.
     * This allows for the description of both thread and group usage (in
     * the FAIR_GROUP_SCHED case).
     * runnable_load_avg is the sum of the load_avg_contrib of the
     * sched_entities on the rq.
     * blocked_load_avg is similar to runnable_load_avg except that
     * its the blocked sched_entities on the rq.
     * utilization_load_avg is the sum of the average running time of
     * the sched_entities on the rq.
     */
    unsigned long runnable_load_avg, blocked_load_avg, utilization_load_avg;
    atomic64_t decay_counter;
    u64 last_decay;
    atomic_long_t removed_load;

#ifdef CONFIG_FAIR_GROUP_SCHED
    /* Required to track per-cpu representation of task group */
    u32 tg_runnable_contrib;
    unsigned long tg_load_contrib;

    /* 
     * h_load = weight * f(tg) 
     * 
     * where f(tg) is the recursive weight fraction assigned to
     * this group.
     */
    unsigned long h_load;
    u64 last_h_load_update;
    struct sched_entity *h_load_next;
#endif
#endif

#ifdef CONFIG_FAIR_GROUP_SCHED
    struct rq *rq;  /* cpu runqueue to which this cfs_rq is attached */

    /* 
     * leaf cfs_rqs are those that hold tasks (lowest schedulable
     * entity in a hierarchy). Non-leaf lrqs hold other higher schedulable
     * entities (like users, containters etc.)
     *
     * leaf_cfs_rq_list ties together list of leaf cfs_rq's in a cpu.
     * This list is used during load balance.
     */
    int on_list;
    struct list_head leaf_cfs_rq_list;
    struct task_group *tg;  /* group that "owns" this runqueue */

#ifdef CONFIG_CFS_BANDWIDTH
    int runtime_enabled;
    u64 runtime_expires;
    s64 runtime_remaining;

    u64 throttled_clock, throttled_clock_task;
    u64 throttled_clock_task_time;
    int throttled, throttle_count;
    struct list_head throttled_list;
#endif
#endif
};
```
- `nr_running`：计算了队列上可运行进程的数目。`h_nr_running` 会递归统计整个子层级（*hierarchy*）中实际可运行任务（*task*）的总数，`nr_running` 只统计当前层级调度实体的数量
- `load`：维护了所有这些进程的累积负荷值。
- `min_vruntime`：跟踪记录队列上所有进程的最小虚拟运行时间。这个值是实现与就绪队列相关的虚拟时钟的基础。其名字很容易会产生一些误解，因为 `min_vruntime` 实际上可能比最左边的树节点的 `vruntime` 大些。它是单调递增的。
- `tasks_timeline`：用于在按时间排序的红黑树中管理所有进程。
- `rb_leftmost`：总是设置为指向树最左边的节点，即最需要被调度的进程。该成员理论上可以通过遍历红黑树获得，但由于我们通常只对最左边的节点感兴趣，因为这可以减少搜索树花费的平均时间。
- `curr`：指向当前执行进程的可调度实体。

对 `min_vruntime` 字段的理解：
1. 它的数值等于队列上某个可运行进程的 `vruntime`
2. 它是单调递增的，但是它不一定是队列上可运行进程的 `vruntime` 中的最小值，因为它只反映当前通过调度器已经选择的进程的 `vruntime`
3. 它在 `update_curr()` 中被更新，根据队列红黑树中最靠左的进程或当前运行进程的 `vruntime` 以及自身 `min_vruntime` 来更新（取最大值）
4. 它是虚拟时钟的基础
   - 睡眠进程唤醒后的“时钟同步”的基准值是 `min_vruntime` (`place_entity(), kernel/sched/fair.c`)
   - 红黑树上进程的排序也是以 `min_vruntime` 为基准的，比它小的应该优先被调度（因为它受到更多不公平地对待），反之同理
   - 每个实体都有 `vruntime`，它记录了实体运行的虚拟时长。当实体在特定 *cfs rq* 上时，`vruntime` 需要是相对于该 *cfs rq* 的一个值（`vruntime += cfs_rq->min_vruntime`）；当实体需要独立存在时（便于实体迁移）才恢复回原值：`vruntime -= cfs_rq->min_vruntime`

## CFS 操作
### 虚拟时钟
完全公平调度算法依赖于虚拟时钟，用以度量等待进程在完全公平系统中所能得到的 *CPU* 时间。

数据结构中没有虚拟时钟，因此所有的必要信息都可以根据现存的实际时钟和每个进程相关的负荷权重推算出来。

所有与虚拟时钟有关的计算都在 `update_curr()` 中执行，该函数在系统中各个不同地方调用，包括周期性调度器之内。
```c
// version: 4.1.15
// kernel/sched/fair.c
/* Update the current task's runtime statistics */
static void update_curr(struct cfs_rq *cfs_rq) {
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;

    if (unlikely(!curr))
        return;
    
    delta_exec = now - curr->exec_start;
    if (unlikely((s64)delta_exec <= 0))
        return;

    curr->exec_start = now;

    schedstat_set(curr->statistics.exec_max,
        max(delta_exec, curr->statistic.exec_max));
    
    curr->sum_exec_runtime += delta_exec;
    schedstat_add(cfs_rq, exec_clock, delta_exec);

    curr->vruntime += calc_delta_fair(delta_exec, curr);
    update_min_vruntime(cfs_rq);

    if (entity_is_task(curr)) {
        struct task_struct *curtask = task_of(curr);

        trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
        cpuacct_charge(curtask, delta_exec);
        account_group_exec_runtime(curtask, delta_exec);
    }

    account_cfs_rq_runtime(cfs_rq, delta_exec);
}
```
- 语义：*Update the current task's runtime statistics*，即更新（*CFS* 就绪队列的）当前进程（调度实例）的运行时间统计数据（开始运行时间、实际运行时间 + 虚拟运行时间）

- 首先，该函数确定 *CFS* 就绪队列（*cfs_rq*）的当前执行进程，并获取主调度器就绪队列（*rq*）的实际时钟值，该值在每个调度周期都会更新（`rq_of()` 是一个辅助函数，用于确定与 *CFS* 就绪队列相关的 `struct rq` 实例，可根据 `container_of()` *back_casting* 出来）
- 如果就 *CFS* 绪队列上当前没有进程正在执行，则无事可做。否则，内核会计算当前进程从上次更新以来的实际运行时间，保存在变量 `delta_exec` 中。同时更新当前进程的开始运行时间（`exec_start`）为当前时钟值、当前进程在 *CPU* 上执行花费的物理时间和虚拟时间：
  - 物理时间只要将 `delta` 加到累积运行时间 `sum_exec_runtime` 即可
  - 虚拟时间：
    - 通过 `calc_delta_fair()` 基于当前进程的权重和 `delta_exec` 来计算，计算结果累加到 `vruntime`
    - **对于运行在 *nice* 级别为 0（优先级为 120）的进程来说，根据定义虚拟时间和物理时间是相等的**
    - 在使用不同的优先级时，必须根据进程的负荷权重重新衡定时间：
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        /* delta /= w */
        static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se) {
            if (unlikely(se->load.weight != NICE_0_LOAD))
                delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
            return delta;
        }

        // kernel/sched/sched.h
        #define SCHED_LOAD_SHIFT    (10 + SCHED_LOAD_RESOLUTION)
        #define SCHED_LOAD_SCALE    (1L << SCHED_LOAD_SHIFT)

        #define NICE_0_LOAD         SCHED_LOAD_SCALE    // 等于 1024，与数组 prio_to_weight 中 nice 为 0 的值一致
        #define NICE_0_SHIFT        SCHED_LOAD_SHIFT

        // kernel/sched/fair.c
        /* 
         * delta_exec * weight / lw.weight 
         * OR
         * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHFIT
         * 
         * Either weight := NICE_0_LOAD and lw \e prio_to_wmult[],
         * in which case we're guaranteed shift stays positive because
         * inv_weight is guaranteed to fit 32 bits, and NICE_0_LOAD
         * gives another 10 bits; therefore shift >= 22.
         *
         * Or, weight =< lw.weight (because lw.weight is the runqueue
         * weight), thus weight/lw.weight <= 1, and therefor our
         * shift will also be positive.
         */
        static u64 __calc_delta(u64 delta_exec, unsigned long weight,
            struct load_weight *lw) {
            u64 fact = scale_load_down(weight);
            int shift = WMULT_SHIFT;

            __update_inv_weight(lw);

            if (unlikely(fact >> 32)) {
                while (fact >> 32) {
                    fact >>= 1;
                    shift--;
                }
            }

            /* hint to use a 32x32->64 mul */
            fact = (u64)(u32)fact * lw->inv_weight;

            while (fact >> 32) {
                fact >>= 1;
                shift--;
            }

            return mul_u64_u32_shr(delta_exec, fact, shift);
        }
        ```
    - **忽略舍入和溢出检查，`calc_delta_fair()` 所作的就是根据公式计算：`delta_exec x (NICE_0_LOAD / curr->load.weight)`**
    - 由于逆向权重值，越重要的进程会有越高的优先级（*nice* 值越低），会得到更大的权重（可见 `prio_to_weight` 数组定义），因此累加的虚拟运行时间会小一些
- 内核需要设置 `min_vruntime`，必须小心保证该值是单调递增的：
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static void update_min_vruntime(struct cfs_rq *cfs_rq) {
        u64 vruntime = cfs_rq->min_vruntime;

        if (cfs_rq->curr)
            vruntime = cfs_rq->curr->vruntime;

        if (cfs_rq->rb_leftmost) {
            struct sched_entity *se = rb_entry(cfs_rq->rb_leftmost, struct sched_entity, run_node);
            if (!cfs_rq->curr)
                vruntime = se->vruntime;
            else
                vruntime = min_vruntime(vruntime, se->vruntime);
        }

        /* ensure we never gain time by being placed backwards */
        cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);
    #ifndef CONFIG_64BIT
        smp_wmb();
        cfs_rq->min_vruntime_copy = cfs_rq->min_vruntime;
    #endif
    }
    ```
    - `vruntime` 的取值情况：`cfs_rq->min_vruntime`（初始值）-> `cfs_rq->curr->vruntime`（如果 *cfs_rq* 当前有进程在运行）-> `se->vruntime`（*cfs_rq* 管理的进程红黑树非空且没有正在运行的进程） 或者 `min_vruntime(vruntime, se->vruntime)`（*cfs_rq* 管理的进程红黑树非空且有正在运行的进程）。
    - 为保证每个队列（*cfs_rq*）的 `min_vruntime` 是单调递增的，内核将其设置为二者中的较大者： `cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime)`。这意味着，每个队列的 `min_vruntime` 只有被树上某个节点的 `vruntime` 超出时才会更新。利用该策略，内核确保 `min_vruntime` 只能增加，不能减少。
- 完全公平调度器的真正关键点是，红黑树的排序过程是根据下列键进行的：
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static inline int entity_before(struct sched_entity *a, struct sched_entity *b) {
        return (s64)(a->vruntime - b->vruntime) < 0;
    }
    ```
    - 键值较小的节点，排序位置就更靠左，因此会被更快地调度。
    - 通过这种方法，内核实现了如下机制：
      - 在进程运行时，其 `vruntime` 稳定地增加，它在红黑树中总是向右移动的。因为越重要的进程 `vruntime` 增加越慢，因此它们向右移动的速度也越慢，这样其被调度的机会要大于次要进程。
      - 如果进程进入睡眠，则其 `vruntime` 保持不变。因为每个队列 `min_vruntime` 同时会增加（它是单调的），那么睡眠进程醒来后，在红黑树中的位置会更靠左，因为其键值变得更小了。
- 如果当前调度实例（*sched_entity*）对应是一个 *task_struct*，则：
  - `account_group_exec_runtime()`: *Maintain exec runtime for a thread group*
- 调用链：
    ```c
    // 1.
    update_curr_fair()  // 作为 sched_class->update_curr() 的实现
        update_curr()
    
    // 2.
    enqueue_task_fair() // 作为 sched_class->enqueue_task() 的实现
        enqueue_entity()
            update_curr()
    
    // 3.
    dequeue_task_fair() // 作为 sched_class->dequeue_task() 的实现
        dequeue_entity()
            update_curr()
    
    // 4.
    put_prev_task_fair() // 作为 sched_class->put_prev_task() 的实现
        put_prev_entity()
            update_curr()
    
    // 5.
    task_tick_fair() // 作为 sched_class->task_tick() 的实现
        entity_tick()
            update_curr()
    
    // 6. 
    check_preempt_wake()    // 作为 sched_class->check_preempt_curr() 的实现
        update_curr()
    
    // 7.
    yield_task_fair() // 作为 sched_class->yield_task() 的实现
                      // 另外 yield_to_task_fair() 会调用它
        update_curr()
    
    // 8.
    task_fork_fair()    // 作为 sched_class->task_fork() 的实现
        update_curr()
    ```
    - 概括来说，当有任务需要入队或出队、被切换之前、周期调度器处理、检查是否抢占、当前进程让出 *CPU* 使用权、*fork* 进程时都会调用到它，以更新 *CFS* 就绪队列的当前调度实例的运行时间统计数据

### 延迟跟踪
内核有个固有概念，称之为良好的调度延迟，即保证每个可运行的进程都应该至少运行一次的某个时间间隔。
- 它在 `sysctl_sched_latency` 给出；
- 可通过 `/proc/sys/kernel/sched_latency_ns` 控制（纳秒），默认值为 20 000 000，即 20 毫秒。

第二个控制参数 `sched_nr_latency`，控制在一个延迟周期中处理的最大活动进程数目。
- 如果活动进程的数目超出该上限，则延迟周期也成比例地线性扩展；
- `sched_nr_latency` 可以通过 `sysctl_sched_min_granularity`（颗粒度、尺寸间隔）间接控制，后者可以通过 `/proc/sys/kernel/sched_min_granularity_ns` 设置。默认值是 4 000 000，即 4 毫秒。每次 `sysctl_sched_latency`/`sysctl_sched_min_granularity` 之一改变时，都会重新计算 `sched_nr_latency`。

`__sched_period()` 确定延迟周期的长度，通常就是 `sysctl_sched_latency`，但如果有更多进程在运行，其值有可能按比例线性扩展。在这种情况下，周期长度是：`sysctl_sched_latency * (nr_running / sched_nr_latency)`。
```c
// version: 4.1.15
// kernel/sched/fair.c
/* 
 * The idea is to set a period in which each task runs once. 
 * 
 * When there are too many tasks (sched_nr_latency) we have to stretch
 * this period because otherwise the slices get too small.
 *
 * p = (nr <= nl) ? l : l * nr/nl
 */
static u64 __sched_period(unsigned long nr_running) {
    u64 period = sysctl_sched_latency;
    unsigned long nr_latency = sched_nr_latency;

    if (unlikely(nr_running > nr_latency)) {
        period = sysctl_sched_min_granularity;
        period *= nr_running;
    }

    return period;
}
```

通过考虑各个进程的相对权重，将一个延迟周期的时间在活动进程之间进行分配。对于由某个可调度实体表示的给定进程，分配到的时间如下计算：
```c
// version: 4.1.15
// kernel/sched/fair.c
/* 
 * We allocate the wall-time (实际时间) slice from the period by 
 * taking a part proportional to the weight.
 *
 * s = p *P[w/rw]
 */
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se) {
    u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);

    for_each_sched_entity(se) {
        struct load_weight *load;
        struct load_weight *lw;

        cfs_rq = cfs_rq_of(se);
        load = &cfs_rq->load;

        if (unlikely(!se->on_rq)) {
            lw = cfs_rq->load;

            update_load_add(&lw, se->load.weight);
            load = &lw;
        }
        slice = __calc_delta(slice, se->load.weight, load);
    }
    return slice;
}

// kernel/sched/fair.c
#ifdef CONFIG_FAIR_GROUP_SCHED
...
#else
#define for_each_sched_entity(se) \
    for (; se; se = NULL)
#endif

static inline struct cfs_rq *cfs_rq_of(struct sched_entity *se) {
    // ...
    // 大致流程 sched_entity -> task_struct -> cpu -> cpu's rq -> cfs_rq
}
```
- 计算结果就是：输入进程的 `load.weight` 与 *cfs_rq* 队列上所有进程的 `load.weight` 之和的比（*CFS* 就绪队列的负荷权重是队列上所有活动进程负荷权重的累加和），再与 `slice` 的乘积。
- `for_each_sched_entity()` 和调度控制组（*cgroup*）有关，它会遍历由 `sched_entity->parent` 定义的调度层次结构，每个实体都加入到就绪队列。另外，如果未选择支持组调度，则会退化为只执行一次循环体中的代码。

`sched_slice()` 的结果作为进程分配到的时间。注，结果时间段是按实际时间给出的，但内核有时候也需要知道等价的虚拟时间：
```c
// version: 4.1.15
// kernel/sched/fair.c
/* 
    * We allocate the vruntime slice of a to-be-inserted task. 
    * 
    * vs = s/w
    */
static u64 sched_vslice(struct cfs_rq *cfs_rq, struct sched_entity *se) {
    return calc_delta_fair(sched_slice(cfs_rq, se), se);
}
```
- 对权重 `weight` 的进程来说，实际时间段 `time` 对应的虚拟时间长度为 `time * (NICE_0_LOAD / weight)`。

## CFS 调度类实现
```c
// version: 4.1.15
// kernel/sched/fair.c
/* All the scheduling class methods: */
const struct sched_class fair_sched_class = {
    .next = &idle_sched_class,
    .enqueue_task = enqueue_task_fair,
    .dequeue_task = dequeue_task_fair,
    .yield_task   = yield_task_fair,
    .yield_to_task = yield_to_task_fair,

    .check_preemt_curr = check_preempt_wakeup,

    .pick_next_task = pick_next_task_fair,
    .put_prev_task  = put_prev_task_fair,

#ifdef CONFIG_SMP
    .select_task_rq = select_task_rq_fair,
    .migrate_task_rq = migrate_task_rq_fair,

    .rq_online = rq_online_fair,
    .rq_offline = rq_offline_fair,

    .task_waking = task_waking_fair,
#endif
    .set_curr_task = set_curr_task_fair,
    .task_tick = task_tick_fair,
    .task_for  = task_fork_fair,

    .prio_changed = prio_changed_fair,
    .switched_from = switched_from_fair,
    .switched_to = switched_to_fair,

    .get_rr_interval = get_rr_interval_fair,

    .update_curr = update_curr_fair,
#ifdef CONFIG_FAIR_GROUP_SCHED
    .task_move_group = task_move_group_fair,
#endif
};
```

### 核心调度路径
包括的调度类回调有：
- `enqueue_task()`
- `dequeue_task()`
- `pick_next_task()`
- `check_preempt_curr()`
- `update_curr()`
- `put_prev_task()`

1. `enqueue_task_fair()` - `enqueue_task()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /* 
    * The enqueue_task method is called before nr_running is
    * increased. Here we update the fair scheduling stats and
    * then put the task into the rbtree.
    */
    static void enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags) {
        struct cfs_rq *cfs_rq;
        struct sched_entity *se = &p->se;

        for_each_sched_entity(se) {
            if (se->on_rq)
                break;
            cfs_rq = cfs_rq_of(se);
            enqueue_entity(cfs_rq, se, flags);

            /* 
            * end evaluation on encountering a throttled cfs_rq
            *
            * note: in the case of encountering a throttled cfs_rq we
            * will post the final h_nr_running increment below.
            */
            if (cfs_rq_throttled(cfs_rq))
                break;
            cfs_rq->h_nr_running++;

            flags = ENQUEUE_WAKEUP;
        }

        for_each_sched_entity(se) {
            cfs_rq = cfs_rq_of(se);
            cfs_rq->h_nr_running++;

            if (cfs_rq_throttled(cfs_rq))
                break;
            
            update_cfs_shares(cfs_rq);
            update_entity_load_avg(se, 1);
        }

        if (!se) {
            update_rq_runnable_avg(rq, rq->nr_running);
            add_nr_running(rq, 1);
        }
        hrtick_update(rq);
    }

    ```
    - `flags`:
      - `ENQUEUE_WAKEUP`: 表示这次入队动作是一次睡眠任务的唤醒。只要任务是从睡眠状态被唤醒（而非新建或迁移），这个标志通常就会被设置
      - `ENQUEUE_WAKING`: 表示这次入队是因为调用了调度类的 `task_waking()` 回调，因而该调度实体的 `vruntime` 需要被特殊处理。在 `try_to_wake_up()` 流程中，如果 `p->sched_class->task_waking()` 存在并被调用，就会额外设置这个标志
    - 如果通过 `struct sched_entity` 的 `on_rq` 成员判断进程已经在就绪队列上，则无事可做；否则，具体的工作委托给 `enqueue_entity()` 完成，其中内核会借机调用 `update_curr()` 更新统计量。
    - 当没有选择组调度配置项时， `for_each_sched_entity()` 之后 `se` 被设置为 `NULL`
    - 在 `add_nr_running()` 中，除了更新 *rq* 就绪队列的 `nr_running` 字段以外，如果 *rq* 的状态是从少于 2 个可运行实体到 2 个或以上的转换，且选择了动态时钟相关的配置项 `CONFIG_NO_HZ_FULL`，则会尝试重启该 *rq* 对应 *CPU* 的时钟 *tick*。`tick_nohz_full_kick_cpu(rq->cpu)` 负责完成该任务，这涉及到 *SMP* 通信：
        ```c
        /* 
         * kick the CPU if it's full dynticks in order to force it
         * to re-evaluate its dependency on the tick and restart it
         * if necessary.
         */
        tick_nohz_full_kick_cpu()
            irq_work_queue_on()
                arch_send_call_function_single_ipi()
                    smp_cross_call()
        ```
    - 如果 *rq* 的当前实体已经运行超过了分配的时间，则会在 `hrtick_update()` 中通过 `hrtick_start_fair()` 进一步调用 `resched_curr()` 以重新调度。这可能是让当前 *CPU* 重新调度，也可能是让其他 *CPU* 重新调度
   1. `enqueue_entity()`
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        static void enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags) {
            /* 
            * Update the normalized vruntime before updating min_vruntime 
            * through calling update_curr().
            */
            if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_WAKING))
                se->vruntime += cfs_rq->min_vruntime;

            /* 
            * update run-time statistics of the 'current'. 
            */
            update_curr(cfs_rq);
            enqueue_entity_load_avg(cfs_rq, se, flags & ENQUEUE_WAKEUP);
            account_entity_enqueue(cfs_rq, se);
            update_cfs_shares(cfs_rq);

            if (flags & ENQUEUE_WAKEUP) {
                place_entity(cfs_rq, se, 0);
                enqueue_sleeper(cfs_rq, se);
            }

            update_stats_enqueue(cfs_rq, se);
            check_spread(cfs_rq, se);
            if (se != cfs_rq->curr)
                __enqueue_entity(cfs_rq, se);
            se->on_rq = 1;

            if (cfs_rq->nr_running == 1) {
                list_add_lead_cfs_rq(cfs_rq);
                check_enqueue_throttle(cfs_rq);
            }
        }
        ```
        - 如果队列的调度实体不是从睡眠状态唤醒，或者是从 `task_waking()` 触发的唤醒，则会在自身 `vruntime` 的基础上再加上 `cfs_rq->min_vruntime`，从而避免它们获得不合理的调度优势
        - 通过 `update_curr()` 更新 *cfs rq* 的 `min_vruntime` 以及它当前实体的执行开始时间、总计运行时间、总计虚拟运行时间（`vruntime`）
        - `account_entity_enqueue()` 会更新 *cfs rq* 的负载权重以及其对应 *rq* 的负载权重，它们都是在原有基础上加上入队实体的权重值。另外，还会 `cfs_rq->nr_running++`
        - 如果当前进程是从睡眠中唤醒的进程，则会通过 `place_entity()` 重新设置它的 `vruntime`，目的是让睡眠进程在被唤醒之后不至于拥有过分的调度优势。`place_entity()` 的 `initial` 参数为 0，表示入队实体不是新建的。`enqueue_sleeper()` 在没有配置 `CONFIG_SCHEDSTATS` 为空函数
        - `update_stats_enqueue()` 的主要作用是更新入队实体的 `se->statistics.wait_start` 为 *rq* 的当前时间（如果它不是 *cfs rq* 的当前实体），即记录入队实体的开始等待时间
        - `__enqueue_entity()` 会将入队实体放入 *cfs rq* 的红黑树中，这里调用了排序的判断方式 `entity_before()`
        - 设置入队实体的 `on_rq` 字段，表明其已经放入 *cfs rq* 队列中
        - 后面两个都是空函数
   2. `hrtick_update()`
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        /* 
         * called from enqueue/dequeue and updates the hrtick when
         * the current task is from our class and nr_running is low
         * enough to matter.
         */
        static void hrtick_update(struct rq *rq) {
            struct task_struct *curr = rq->curr;
            if (!hrtick_enable(rq) || curr->sched_class != &fair_sched_class)
                return;
            if (cfs_rq_of(&curr->se)->nr_running < sched_nr_latency)
                hrtick_start_fair(rq, curr);
        }

        static void hrtick_start_fair(struct rq *rq, struct task_struct *p) {
            struct sched_entity *se = &p->se;
            struct cfs_rq *cfs_rq = cfs_rq_of(se);

            WARN_ON(task_rq(p) != rq);

            if (cfs_rq->nr_running > 1) {
                u64 slice = sched_slice(cfs_rq, se);
                u64 ran = se->sum_exec_runtime - se->prev_sum_exec_runtime;
                u64 delta = slice - ran;

                if (delta < 0) {
                    if (rq->curr == p)
                        resched_curr(rq);
                    return;
                }
                hrtick_start(rq, delta);
            }
        }
        ```
        - `resched_curr()` 对 `rq` 进程重新调度
        - `hrtick_start()` 重新设置高精度定时器的过期时间（当前时间基础上加上 10 微秒与 `delta` 间的最大值），然后重启
   3. `place_entity()`
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        static void place_entity(struct cfs_rq *cfs_rq, struct schced_entity *se, int initial) 
        {
            u64 vruntime = cfs_rq->min_vruntime;

            /* 
             * The 'current' period is already promised to the current 
             * tasks, however the extra weight of the new task will
             * slow them down a little, place the new task so that
             * it fits in the slot that stays open at the end.
             */
            if (initial && sched_feat(START_DEBIT))
                vruntime += sched_vslice(cfs_rq, se);
            
            /* sleeps up to a single latency don't count. */
            if (!initial) {
                unsigned long thresh = sysctl_sched_latency;

                /* 
                 * Halve (减半) their sleep time's effect, to allow
                 * for a gentler (更缓和的) effect of sleepers:
                 */
                if (sched_feat(GENTLE_FAIR_SLEEPERS))
                    thresh >> 1;
                
                vruntime -= thresh;
            }

            /* ensure we never gain time by being placed backwards */
            se->vruntime = max_vruntime(se->vruntime, vruntime);
        }
        ```
        - 注释：原本当前的这段时间是属于现在 *cfs rq* 上的可运行实体（因为 *CFS* 承诺在这段时间每个实体都能够得到运行），新入队实体导致 *cfs rq* 权重的更新会缩减它们原本能够得到的运行时间。因此，这里需要将新入队实体往后放一放，让原本在这段时间的可运行实体先运行。
        - *debit*: 借记
        - 让 `vruntime = cfs_rq->min_runtime`，让睡眠唤醒的实体和新建实体与当前 *cfs rq* 进行虚拟时钟的同步，防止它们拥有过分的调度优先。如果是睡眠唤醒的实体，还会在这基础上削减一个延迟周期（即 `sysctl_sched_latency`），使其还是能够优先一点执行
        - 在最后对 `se->vruntime` 赋值时，取最大值，可以防止占有便宜的实体因睡眠后由于 `vruntime` 的缩小又重新获得了更大的调度优势

2. `dequeue_task_fair()` - `dequeue_task()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /* 
     * The dequeue_task method is called before nr_running is
     * decreased. We remove the task from the rbtree and update
     * the fair scheduling stats
     */
    static void dequeue_task_fair(struct rq *rq, struct task_struct *p, int flags) 
    {
        struct cfs_rq *cfs_rq;
        struct sched_entity *se = &p->se;
        int task_sleep = flags & DEQUEUE_SLEEP;

        for_each_sched_entity(se) {
            cfs_rq = cfs_rq_of(se);
            dequeue_entity(cfs_rq, se, flags);

            /* 
             * end evaluation on encountering a throttled cfs_rq
             * 
             * note: in the case of encountering a throttled cfs_rq
             * we will post the final h_nr_running decrement below.
             */
            if (cfs_rq_throttled(cfs_rq))
                break;
            cfs_rq->h_nr_running--;

            /* 
             * Don't dequeue parent if it has other entities 
             * besides us
             */
            if (cfs_rq->load.weight) {
                /* 
                 * Bias pick_next to pick a task from this cfs_rq,
                 * as p is sleeping when it is within its sched_slice.
                 */
                if (task_sleep && parent_entity(se))
                    set_next_buddy(parent_entity(se));
                
                /* avoid re-evaluating load for this entity */
                se = parent_entity(se);
                break;
            }
            flags |= DEQUEUE_SLEEP;
        }

        for_each_sched_entity(se) {
            cfs_rq = cfs_rq_of(se);
            cfs_rq->h_nr_running--;
            
            if (cfs_rq_throttled(cfs_rq))
                break;
            
            update_cfs_shares(cfs_rq);
            update_entity_load_avg(se, 1);
        }

        if (!se) {
            sub_nr_running(rq, 1);
            update_rq_runnable_avg(rq, 1);
        }
        hrtick_update(rq);
    }
    ```
    - 通过 `dequeue_entity()` 将实体从 *cfs rq* 拿走
    - `sub_nr_running(rq, 1)` 将 *rq* 对可运行实体的统计数量（`nr_running`）减一
    - `hrtick_update()` 的解析见上文
   1. `dequeue_entity()`:
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        static void dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags) 
        {
            /* Update run-time statistics of the 'current' */
            update_curr(cfs_rq);
            dequeue_entity_load_avg(cfs_rq, se, flags & DEQUEUE_SLEEP);

            update_stats_dequeue(cfs_rq, se);
            if (flags & DEQUEUE_SLEEP) {
        #ifdef CONFIG_SCHEDSTATS
                ...  
        #endif
            }

            clear_buddies(cfs_rq, se);

            if (se != cfs_rq->curr)
                __dequeue_entity(cfs_rq, se);
            se->on_rq = 0;
            account_entity_dequeue(cfs_rq, se);

            /* 
             * Normalize the entity after updating the min_vruntime
             * because the update can refer to the ->curr item and we
             * need to reflect this movement in our normalized position
             */
            if (!(flags & DEQUEUE_SLEEP))
                se->vruntime -= cfs_rq->min_vruntime;
            
            /* return excess runtime on last dequeue */
            return_cfs_rq_runtime(cfs_rq);

            update_min_vruntime(cfs_rq);
            update_cfs_shares(cfs_rq);
        }
        ```
        - 首先，通过 `update_curr(cfs_rq)` 更新 *cfs rq* 的 `min_vruntime` 及其当前实体的运行时间统计数据
        - 如果将要移出的实体不是 *cfs rq* 的当前实体，则 `update_stats_dequeue(cfs_rq, se)` 会更新移出实体的等待标记
        - `clear_buddies` 更新 *cfs rq* 的 `last`、`next`、`skip` 字段（因为它们可能指向将要移出的实体）
        - 如果将要移出的实体不是 *cfs rq* 的当前实体，则通过 `__dequeue_entity()` 将其从 *cfs rq* 的红黑树上移出
        - 标记 `on_rq` 字段为 0，表示它已经从就绪队列上移出
        - `account_entity_dequeue()` 相当于是 `account_entity_enqueue()` 的逆操作：减小 *cfs rq* 的负载权重和 *rq* 的负载权重，同时将 `cfs_rq->nr_running--`
        - 如果移出实体不是因为睡眠原因导致的，则缩减其 `vruntime`（这可能是将实体加入队列时需要在 `vruntime` 的基础上增加一个值是同样的道理）
        - 通过 `update_min_vruntime()`（见上文）更新 *cfs rq* 的 `min_vruntime`

3. `pick_next_task_fair()` - `pick_next_task()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static struct task_struct *pick_next_task_fair(struct rq *rq, struct task_struct *prev) 
    {
        struct cfs_rq *cfs_rq = &rq->cfs;
        struct sched_entity *se;
        struct task_struct *p;
        int new_task;
    
    again:
    #ifdef CONFIG_FAIR_GROUP_SCHED
        ...
    #endif
        if (!cfs_rq->nr_running)
            goto idle;
        
        put_prev_task(rq, prev);

        do {
            se = pick_next_entity(cfs_rq, NULL);
            set_next_entity(cfs_rq, se);
            cfs_rq = group_cfs_rq(se);
        } while(cfs_Rq);

        p = task_of(se);

        if (hrtick_enable(rq))
            hrtick_start_fair(rq, p);
        return p;
    
    idle:
        new_tasks = idle_balance(rq);
        /* 
         * Because idle_balance() releases (and re-acquires) rq->lock
         * it is possible for any higher priority task to appear.
         * In that case we must re-start the pick_next_entity() loop.
         */
        if (new_tasks < 0)
            return RETRY_TASK;
        if (new_tasks > 0)
            goto again;
        return NULL;
    }
    ```
    - 如果 *cfs rq* 上没有可运行的实体（`nr_running` 为 0），则跳到 `idle` 处开始运行。`idle_balance()` 会尝试从其他 *CPU* 拉取实体来运行（触发负载均衡 `load_balance()`）。如果有从其他 *CPU* 拉取到实体，则重新尝试选择下一个实体（`pick_next_entity()`），否则返回 `RETRY_TASK`（负载均衡后当前 *CPU* 获取了更高优先级的调度类） 或 `NULL`
    - 否则，通过调用 `put_prev_task()` 做任务切换前的准备，它会进一步调用 `sched_class->put_prev_tasks()` 回调。然后，选择下一个进程（注意，下一个进程可能还是 *cfs rq* 的当前进程）、将 *cfs rq* 的当前进程设置为上述选择的进程并更新其时间方面的信息，如运行开始时间等。如果当前 *rq* 上使能了高精度 *tick*，则调用 `hrtick_start_fair()` 重新开始 *tick* 的计时。另外，`hrtick_start_fair()` 也可能会将 `pick_next_entity()` 选择的下一个进程标记为需要重新调度，如果其运行时长已经超出了分配给它的 *CPU* 时间（`sched_slice()`）。最后，返回选择的下一个进程。
   1. `pick_next_entity()`
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        /* 
         * Pick the next process, keeping these things in mind, in this order:
         * 1) keep things fair between processes/task groups
         * 2) pick the "next" process, since someone really wants that to run
         * 3) pick the "last" process, for cache locality
         * 4) do not run the "skip" process, if something else is available
         */
        static struct sched_entity *
        pick_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *curr) 
        {
            struct sched_entity *left = __pick_first_entity(cfs_rq);
            struct sched_entity *se;

            /* 
             * If curr is set we have to see if its left of the leftmost
             * entity still in the tree, provided there was anything
             * in the tree at all.
             */
            if (!left || (curr && entity_before(curr, left)))
                left = curr;
            
            se = left;  /* ideally we run the leftmost entity */

            /* 
             * Avoid running the skip buddy, if running something
             * else can be done without getting too unfair.
             */
            if (cfs_rq->skip == se) {
                struct sched_entity *second;
                if (se == curr) {
                    second = __pick_first_entity(cfs_rq);
                } else {
                    second = __pick_next_entity(se);
                    if (!second || (curr && entity_before(curr, second)))
                        second = curr;
                }
                if (second && wakeup_preempt_entity(second, left) < 1)
                    se = second;
            }

            /* Prefer last buddy, try to return the CPU to a preempted task */
            if (cfs_rq->last && wakeup_preempt_entity(cfs_rq->last, left) < 1)
                se = cfs_rq->last;
            
            /* Someone really wants this to run. If it's not unfair */
            if (cfs_rq->next && wakeup_preempt_entity(cfs_rq->next, left) < 1)
                se = cfs_rq->next;
            
            clear_buddies(cfs_rq, se);
            return se;
        }
        ```
        - 注释中的 *next*，*last*，*skip* 是 *cfs rq* 的字段
        - `__pick_next_entity(cfs_rq)`: 如果 *cfs rq* 红黑树的最左边结点非空，则返回该结点；否则返回 `NULL`
        - 实体的选择：
          - 理想情况下，选择的下一个结点就是红黑树中最左边的结点
          - 尽可能不选择需要跳过的实体，即 `cfs_rq->skip`。这时可能选择 `cfs_rq->skip` 的下一个实体（`rb_next()`）
          - 可能是 `cfs_rq->last`，如果它能抢占前面选择的实体（`wakeup_preempt_entity()`，它与系统控制参数 `sysctl_sched_wakeup_granularity` 有关，其返回值小于 1 表示第二个参数实体不抢占第一个）
          - 可能是 `cfs_rq->next`，如果它能抢占前面选择的实体
        - 最后，如果选择的下一个实体与 *next*，*last*，*skip* 其中之一相等，则 `clear_buddies()` 会将 *cfs rq* 的对应字段清空（设置为 `NULL`）
   2. `set_next_entity()`
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        static void set_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *se) 
        {
            /* 'current' is not kept within the tree */
            if (se->on_rq) {
                /* 
                 * Any task has to be enqueued before it get to execute
                 * on a CPU. So account for the time it spent waiting
                 * on the runqueue.
                 */
                update_stats_wait_end(cfs_rq, se);
                __dequeue_entity(cfs_rq, se);
                update_entity_load_avg(se, 1);
            }
            update_stats_curr_start(cfs_rq, se);
            cfs_rq->curr = se;
        #ifdef CONFIG_SCHEDSTATS
            ...
        #endif
            se->prev_sum_exec_runtime = se->sum_exec_runtime;
        }
        ```
        - *cfs rq* 的当前实体并不在其红黑树上
        - `update_stats_wait_end()` 会将 `se->statistics.wait_start` 设置为 0，同时更新其他等待相关的统计信息
        - `update_stats_curr_start()` 记录该实体开始运行的时间，涉及字段为 `se->exec_start`
        - 设置 *cfs rq* 的当前实体为 `se`、更新 `se->prev_sum_exec_runtime`

4. `check_preempt_wakeup()` - `check_preempt_curr()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /* 
     * Preempt the current task with a newly woken task if needed
     */
    static void check_preempt_wakeup(struct rq *rq, struct task_struct *p, int wake_flags) 
    {
        struct task_struct *curr = rq->curr;
        struct sched_entity *se = &curr->se, *pse = &p->se;
        struct cfs_rq *cfs_rq = task_of_rq(curr);
        int scale = cfs_rq->nr_running >= sysctl_nr_latency;
        int next_buddy_marked = 0;

        if (unlikely(se == pse))
            return;
        
        /* 
         * This is possible from callers such as attach_tasks(), in
         * which we unconditionally (无条件地) check_preempt_curr()
         * after an enqueue (which may have lead to a throttle). This
         * both saves work and prevents false next-buddy nomination (提名，任命) below.
         */
        if (unlikely(throttled_hierarchy(cfs_rq_of(pse))))
            return;
        if (sched_feat(NEXT_BUDDY) && scale && !(wake_flags & WF_FORK)) {
            set_next_buddy(pse);
            next_buddy_marked = 1;
        }
        /* 
         * We can come here with TIF_NEED_RESCHED already set from new
         * task wake up path.
         *
         * Note: this also catches the edge-case of curr being in a
         * throttled group (e.g. via set_curr_task), since update_curr()
         * (in the enqueue of curr) will have resulted in resched being
         * set. This prevents us from potentially nominating it as
         * a false LAST_BUDDY below.
         */
        if (test_tsk_need_resched(curr))
            return;
        
        /* Idle tasks are by definition preempted by non-idle tasks. */
        if (unlikely(curr->policy == SCHED_IDLE) && 
            likely(p->policy != SCHED_IDLE))
            goto preempt;
        
        /* 
         * Batch and idle tasks do not preempt non-idle tasks (their
         * preeption is driven by the tick):
         */
        if (unlikely(p->policy != SCHED_NORMAL) || !sched_feat(WAKEUP_PREEMPTION))
            return;
        
        find_matching_se(&se, &pse);    // 调度组相关，当前为空
        update_curr(cfs_rq_of(se));
        BUG_ON(!pse);
        if (wakeup_preempt_entity(se, pse) == 1) {
            /* 
             * Bias pick_next to pick the sched entity that is 
             * triggering this preemption.
             */
            if (!next_buddy_marked)
                set_next_buddy(pse);
            goto preempt;
        }
        return;
    preempt:
        resched_curr(rq);
        /* 
         * Only set the backward buddy when the current task is still
         * on the rq. This can happen when a wakeup gets interleaved (交替)
         * with schedule on the ->pre_schedule() or idle_balance()
         * point, either of which can * drop the rq lock.
         *
         * Also, during early boot the idle thread is in the fair
         * class, for obvious reasons its a bad idea to schedule back to it.
         */
        if (unlikely(!se->on_rq || curr = rq->idle))
            return;
        if (sched_feat(LAST_BUDDY) && scale && entity_is_task(se))
            set_last_buddy(se);
    }
    ```
    - `set_next_buddy(pse)` 将 `pse` 对应的 *cfs rq* 的 `next` 字段设置为指向 `pse`
    - `SCHED_IDLE` 调度策略的进程无条件被其他类型的进程抢占
    - `SCHED_BATCH` 和 `SCHED_IDLE` 调度策略的进程不会抢占其他调度策略的进程
    - `update_curr()`（在抢占前）更新 *cfs rq* 的时间信息
    - 当 `wakeup_preempt_entity(se, pse)` 返回 1 时，表明 `pse` 可以抢占 `se`
    - 发生抢占时，通过 `reshed_curr()` 将当前进程的 `TIF_NEED_RESCHED` 置位，以便调度 `pse` 上 *CPU* 运行
    - 总结：结合 `pick_next_entity()` 的实现逻辑，如果进程 `pse` 可以抢占当前进程，则设置 `cfs_rq->next` 为 `pse`、`cfs_rq->last` 为 `se`，并设置当前进程的 `TIF_NEED_RESCHED` 标志。这样，当触发调度时，通过 `pick_next_task_fair()` 会首先选择由 `cfs_rq->next` 指向的 `pse`。同时，当该抢占进程结束运行时，由 `cfs_rq->last` 指向的 `se` 也能够优先恢复运行。
    1. `wakeup_preempt_entity()`
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        /* 
         * Should 'se' preempt 'curr' 
         *              | s1
         *       |s2
         * |s3
         *        g
         *    |<------->|c
         * w(c, s1) = -1
         * w(c, s2) = 0
         * w(c, s3) = 1
         *
         */
        satic int wakeup_preempt_entity(struct sched_entity *curr, struct sched_entity *se) 
        {
            s64 gran, vdiff = curr->vruntime - se->vruntime;
            if (vdiff <= 0)
                return -1;
            gran = wakeup_gran(curr, se);
            if (vdiff > gran)
                return 1;
            return 0;
        }

        static unsigned long
        wakeup_gran(struct sched_entity *curr, struct sched_entity *se) {
            unsigned long gran = sysctl_sched_wakeup_granularity;
            return calc_delta_fair(gran, se);
        }
        ```
        - 注释：从实体的 `vruntime` 方面考虑，如果 `se->vruntime` 在 `curr->vruntime` 一定范围内，函数会返回 0；如果 `se->vruntime` 超出这个范围且比 `curr->vruntime` 大，则返回 -1；如果 `se->vruntime` 超出这个范围且比 `curr->vruntime` 晓，则返回 1。当函数返回 1 时，表明 `se` 可以抢占 `curr`
        - `wakeup_gran()` 的返回值：`sysctl_sched_wakeup_granularity * (NICE_0_LOAD / se->load.weight)`

5. `update_curr_fair()` - `update_curr()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static void update_curr_fair(struct rq *rq) 
    {
        update_curr(cfs_rq_of(&rq->curr->se));
    }
    ```
    - `update_curr()` 的详细信息见上文（这里不直接使用 `rq->cfs` 的原因是为了兼容组调度，来自 *ds* 问答）

6. `put_prev_task_fair()` - `put_prev_task()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /* Account for a descheduled task */
    static void put_prev_task_fair(struct rq *rq, struct task_struct *prev) 
    {
        struct sched_entity *se = &prev->se;
        struct cfs_rq *cfs_rq;

        for_each_sched_entity(se) {
            cfs_rq = cfs_rq_of(se);
            put_prev_entity(cfs_rq, se);
        }
    }

    static void put_prev_entity(struct cfs_rq *cfs_rq, struct sched_entity *prev) {
        /* 
         * If still on the runqueue then deactive_task() 
         * was not called and update_curr() has to be done
         */
        if (prev->on_rq)
            update_curr(cfs_rq);
        /* throttle cfs_rqs exceeding runtime */
        check_cfs_rq_runtime(cfs_rq);
        
        check_spread(cfs_rq, prev);
        if (prev->on_rq) {
            update_stats_wait_start(cfs_rq, prev);
            __enqueue_entity(cfs_rq, prev);
            update_entity_load_avg(prev, 1);
        }
        cfs_rq->curr = NULL;
    }
    ```
    - `check_cfs_rq_runtime()` 与配置选项 `CONFIG_CFS_BANDWIDTH` 有关，如果没有选择此配置，则该函数返回 `false`
    - 总的来说，`put_prev_task_fair()` 的作用就是：通过 `update_curr()` 更新 *cfs rq* 的时间统计数据、重新设置 `prev` 的开始等待时间、将 `prev` 重新放回 *cfs rq*、设置 `cfs_rq->curr` 为 `NULL`。从而为切换到下一个进程做准备。

### 时间驱动与生命周期
包括的调度类回调有：
- `task_tick()`
- `task_fork()`
- `switch_from()`
- `switch_to()`

1. `task_tick_fair()` - `task_tick()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /* scheduler tick hitting a task of our scheduling class: */
    static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued) 
    {
        struct cfs_rq *cfs_rq;
        struct sched_entity *se = &curr->se;

        for_each_sched_entity(se) {
            cfs_rq = cfs_rq_of(se);
            entity_tick(cfs_rq, se, queued);
        }

        if (numabalancing_enabled)
            task_tick_numa(rq, curr);
        
        update_rq_runnable_avg(rq, 1);
    }

    static void entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued) 
    {
        /* update run-time statistics of the 'current' */
        update_curr(cfs_rq);
        /* Ensure that runnable average is periodically updated */
        update_entity_load_avg(curr, 1);
        update_cfs_rq_blocked_load(cfs_rq, 1);
        update_cfs_shares(cfs_rq);
    
    #ifdef CONFIG_SCHED_HRTICK
        /* 
         * queued ticks are scheduled to match the slice, so don't bother 
         * validating it and just reschedule.
         */
        if (queued) {
            resched_curr(rq_of(cfs_rq));
            return;
        }
        /* don't let the period tick interface with the hrtick preemption */
        if (!sched_feat(DOUBLE_TICK) && 
            hrtimer_active(&rq_of(cfs_rq)->hrtick_timer))
            return;
    #endif
        if (cfs_rq->nr_running > 1)
            check_preempt_tick(cfs_rq, curr);
    }
    ```
    - "*queued ticks are scheduled to match the slice, so don't bother validating it and just reschedule.*"，当 `queued` 为真时，即 *hrtick*，说明本次 *tick* 是专门为匹配调度时间片而精确安排的，因此无需再做常规的时间片检查，直接标记需要重新调度即可。
    - 当 *cfs rq* 上的可运行实体数量大于 1（`nr_running > 1`）时，通过 `check_preempt_tick()` 检查当前进程是否可以被抢占。
    - 总结，由 `task_tick_fair()` 触发的三种重新调度的情形：
      - 该 *tick* 是由 *hrtick* 触发的，则直接重新调度
      - `curr` 已经运行了足够长的时间，重新调度
      - `curr` 比 *cfs rq* 红黑树中 *vruntime* 最小的实体还占有比它自身本次运行的时长还大的便宜时，重新调度
   1. `check_preempt_tick()`
        ```c
        // version: 4.1.15
        // kernel/sched/fair.c
        /* Preempt the current task with a newly woken task if needed */
        static void check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr) 
        {
            unsigned long ideal_runtime, delta_exec;
            struct sched_entity *se;
            s64 delta;

            ideal_runtime = sched_slice(cfs_rq, curr);
            delta_exec = curr->sun_exec_runtime - curr->prev_sum_exec_runtime;
            if (delta_exec > ideal_runtime) {
                resched_curr(rq_of(cfs_rq));
                /* 
                 * The current task ran long enough, ensure it doesn't get
                 * re-elected due to buddy favours.
                 */
                clear_buddies(cfs_rq, curr);
                return;
            }

            /* 
             * Ensure that a task that missed wakeup preemption by a narrow 
             * margin doesn't have to wait for a full slice.
             * This also mitigates (减轻、缓解) buddy induced (导致、引起) latencies under load.
             */
            if (delta_exec < sysctl_sched_min_granularity)
                return;
            
            se = __pick_first_entity(cfs_rq);
            delta = curr->vruntime - se->vruntime;

            if (delta < 0)
                return;
            if (delta > ideal_runtime)
                resched_curr(rq_of(cfs_rq));
        }
        ```
        - 如果 `curr` 实体已经运行了足够长的时间，则通过 `resched_curr()` 重新调度。为了防止 *cfs rq* 的 *last, next* 可能指向 `curr`，从而为其提供了调度优势，通过 `clear_buddies()` 清除指向 `curr` 的字段。
        - 否则，检查 `curr` 本次执行时长是否小于最小的调度颗粒度（`sysctl_sched_min_granularity`）。如果是，则直接返回。当不满足这个条件时，意味从调度的时间间隔上满足调度要求，*Ensure that a task that missed wakeup preemption by a narrow margin doesn't have to wait for a full slice. This also mitigates (减轻、缓解) buddy induced (导致、引起) latencies under load*. 首先，从 *cfs rq* 中选择最左边的结点，让其和 `curr->vruntime` 进行比较。如果 `curr` 本就执行的虚拟时间更短，则不应该调度，直接返回。否则，当 `curr` 比 *cfs rq* 红黑树中最小 *vruntime* 的实体还多占 `ideal_runtime` 以上的便宜时，触发调度（`resched_curr()`）

2. `task_fork_fair()` - `task_fork()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /* 
     * called on fork with the child task as argument from the parent's context 
     * - child not yet on the tasklist
     * - preemption disable
     */
    static void task_fork_fair(struct task_struct *p) 
    {
        struct cfs_rq *cfs_rq;
        struct sched_entity *se = *&p->se, *curr;
        int this_cpu = smp_processor_id();
        struct rq *rq = this_rq();
        unsigned long flags;

        raw_spin_lock_irqsave(&rq->lock, flags);
        
        update_rq_clock(rq);

        cfs_rq = task_cfs_rq(current);
        curr = cfs_rq->curr;

        /* 
         * Not only the cpu but also the task_group of the parent might have 
         * been changed after parent->se.parent, cfs_rq were copied to child->se.parent,cfs_rq.
         * So call __set_task_cpu() to make those of child point to valid ones.
         */
        rcu_read_lock();
        __set_task_cpu(p, this_cpu);
        rcu_read_unlock();

        update_curr(cfs_rq);

        if (curr)
            se->vruntime = curr->vruntime;
        place_entity(cfs_rq, se, 1);

        if (sysctl_sched_child_runs_first && curr && entity_before(curr, se)) {
            /* 
             * Upon rescheduling, sched_class::put_prev_task() will palce 'current'
             * within the tree based on its new key value.
             */
            swap(curr->vruntime, se->vruntime);
            resched_curr(rq);
        }
        se->vruntime -= cfs_rq->min_vruntime;

        raw_spin_unlock_irqrestore(&rq->lock, flags);
    }
    ```
    - 由于执行 `task_fork_fair()` 时的环境为调用 *fork* 的进程的上下文，因此 `curr` 指向该进程
    - 首先，通过 `update_rq_clock()` 更新了 *rq* 的时钟信息
    - 调用 `__set_task_cpu()` 设置与子进程的 *CPU* 相关信息，比如 `task_struct->cpu` 和 `thread_info->cpu`
    - 随后，更新 *cfs rq* 的运行时间统计数据、设置 `p` 对应的 *vruntime*
    - 如果通过系统控制参数 `sysctl_sched_child_runs_first` 规定了让子进程先运行，为保证这一点，在必要时交换 `curr` 和 `p` 对应的 *vruntime* 并重新调度
    - （来自 *ds* 的问答）让子进程在原有 *vruntime* 的基础上缩减 `cfs_rq->min_vruntime`，因为之前设置的 *vruntime* 都是基于当前 *CPU* 的 *cfs rq* 的。这是为了让子进程的 *vruntime* 以相对值的形式保存，而不是一个绝对的、绑定在当前 *CPU* 运行队列时间基准上的值。
      - 这个操作的关键背景在于 `fork()` 之后、子进程真正被唤醒并加入运行队列之前，它有可能被迁移到其他 *CPU* 上。内核提交记录明确指出了这一点：`task_fork_fair()` 将 *vruntime* 留在相对状态（*relative state*），这样在 `wake_up_new_task()` 之前的时间延迟或 *CPU* 迁移就不会造成问题。
      - 在 `enqueue_entity()` 的 `flags` 中看到的 `ENQUEUE_WAKEUP` 逻辑，正是这个机制的配套部分。对于新 *fork* 的进程，在唤醒入队时，通常会走类似的路径，把之前减去的 *min_vruntime* 重新加回去，完成从“相对值”到“目标队列绝对值”的转换。
      - 因此，这一步本质上是在为子进程的 “可迁移性” 做准备。
    1. `__set_task_cpu()`
        ```c
        // version: 4.1.15
        // kernel/sched/sched.h
        static inline void __set_task_cpu(struct task_struct *p, unsigned int cpu)
        {
            set_task_rq(p, cpu);
        #ifdef CONFIG_SMP
            /* 
             * After ->cpu is set up to a new value, task_rq_lock(p, ...) can be successfully
             * executed on another CPU. We must ensure that updates of per-task data
             * have been completed by this moment.
             */
            smp_wmb();
            task_thread_info(p)->cpu = cpu;
            p->wake_cpu = cpu;
        #endif
        }
        ```
        - `set_task_rq()` 在未启用 `CONFIG_CGROUP_SCHED` 时为空

3. `switched_from_fair()` - `switch_from()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static void switched_from_fair(struct rq *rq, struct task_struct *p)
    {
        struct sched_entity *se = &p->se;
        struct cfs_rq *cfs_rq = cfs_rq_of(se);

        /*
        * Ensure the task's vruntime is normalized, so that when it's
        * switched back to the fair class the enqueue_entity(.flags=0) will
        * do the right thing.
        *
        * If it's queued, then the dequeue_entity(.flags=0) will already
        * have normalized the vruntime, if it's !queued, then only when
        * the task is sleeping will it still have non-normalized vruntime.
        */
        if (!task_on_rq_queued(p) && p->state != TASK_RUNNING) {
            /*
            * Fix up our vruntime so that the current sleep doesn't
            * cause 'unlimited' sleep bonus.
            */
            place_entity(cfs_rq, se, 0);
            se->vruntime -= cfs_rq->min_vruntime;
        }

    #ifdef CONFIG_SMP
        /*
        * Remove our load from contribution when we leave sched_fair
        * and ensure we don't carry in an old decay_count if we
        * switch back.
        */
        if (se->avg.decay_count) {
            __synchronize_entity_decay(se);
            subtract_blocked_load_contrib(cfs_rq, se->avg.load_avg_contrib);
        }
    #endif
    }
    ```
    - 主要是完成对迁出任务 *vruntime* 的 *normalize* 工作，即在 `task_fork_fair()` 中提到的相对值、绝对值问题。而且注释中说明了：如果迁出任务已经在就绪队列上，那么 `dequeue_entity()` 就会完成 *vruntime* 的 *normalize* 工作。当其不在就绪队列上时，只有它在睡眠时才会有需要 *normalize* 操作的问题

4. `switch_to()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /*
     * We switched to the sched_fair class.
     */
    static void switched_to_fair(struct rq *rq, struct task_struct *p)
    {
    #ifdef CONFIG_FAIR_GROUP_SCHED
        ...
    #endif
        if (!task_on_rq_queued(p))
            return;

        /*
        * We were most likely switched from sched_rt, so
        * kick off the schedule if running, otherwise just see
        * if we can still preempt the current task.
        */
        if (rq->curr == p)
            resched_curr(rq);
        else
            check_preempt_curr(rq, p, 0);
    }
    ```

### 主动控制与优先级变更
包括的调度类回调有：
- `yield_task()`
- `prio_changed()`
- `set_curr_task()`
- `get_rr_interval()`

1. `yield_task_fair()` - `yield_task()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /*
     * sched_yield() is very simple
     *
     * The magic of dealing with the ->skip buddy is in pick_next_entity.
     */
    static void yield_task_fair(struct rq *rq)
    {
        struct task_struct *curr = rq->curr;
        struct cfs_rq *cfs_rq = task_cfs_rq(curr);
        struct sched_entity *se = &curr->se;

        /*
        * Are we the only task in the tree?
        */
        if (unlikely(rq->nr_running == 1))
            return;

        clear_buddies(cfs_rq, se);

        if (curr->policy != SCHED_BATCH) {
            update_rq_clock(rq);
            /*
            * Update run-time statistics of the 'current'.
            */
            update_curr(cfs_rq);
            /*
            * Tell update_rq_clock() that we've just updated,
            * so we don't do microscopic update in schedule()
            * and double the fastpath cost.
            */
            rq_clock_skip_update(rq, true);
        }

        set_skip_buddy(se);
    }
    ```
    - `sys_sched_yield(), kernel/sched/core.c` 会调用调度类实例的 `yield_task()` 回调，并在之后通过 `schedule()` 完成主动调度。
    - 在上述函数中最主要的还是 `clear_buddies(cfs_rq, se)` 和 `set_skip_buddy(se)`，它们让当前运行实体没那么容易被调度器选中了。

2. `prio_changed_fair()` - `prio_changed()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /*
     * Priority of the task has changed. Check to see if we preempt
     * the current task.
     */
    static void prio_changed_fair(struct rq *rq, struct task_struct *p, int oldprio)
    {
        if (!task_on_rq_queued(p))
            return;

        /*
        * Reschedule if we are currently running on this runqueue and
        * our priority decreased, or if we are not currently running on
        * this runqueue and our priority is higher than the current's
        */
        if (rq->curr == p) {
            if (p->prio > oldprio)
                resched_curr(rq);
        } else
            check_preempt_curr(rq, p, 0);
    }
    ```
    - 如果 `p` 是 *rq* 当前运行实体且优先级变更后更小，触发重新调度；否则，`p` 尝试抢占当前运行实体

3. `set_curr_task_fair()` - `set_curr_task()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /* Account for a task changing its policy or group.
     *
     * This routine is mostly called to set cfs_rq->curr field when a task
     * migrates between groups/classes.
     */
    static void set_curr_task_fair(struct rq *rq)
    {
        struct sched_entity *se = &rq->curr->se;

        for_each_sched_entity(se) {
            struct cfs_rq *cfs_rq = cfs_rq_of(se);

            set_next_entity(cfs_rq, se);
            /* ensure bandwidth has been allocated on our new cfs_rq */
            account_cfs_rq_runtime(cfs_rq, 0);
        }
    }
    ```
    - `set_next_entity()` 具体信息见上文，最主要的是它让 `cfs_rq->curr = se`，即将 `se` 设置为 *cfs rq* 的当前运行实体。上述的注释直接说明了它的功能：“*This routine is mostly called to set cfs_rq->curr field when a task migrates between groups/classes*.”。也就是说，当一个任务因为在调度组或调度类间迁移等原因，需要被“认定”为当前任务时，这个回调会被触发

4. `get_rr_interval_fair()` - `get_rr_interval()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static unsigned int get_rr_interval_fair(struct rq *rq, struct task_struct *task)
    {
        struct sched_entity *se = &task->se;
        unsigned int rr_interval = 0;

        /*
        * Time slice is 0 for SCHED_OTHER tasks that are on an otherwise
        * idle runqueue:
        */
        if (rq->cfs.load.weight)
            rr_interval = NS_TO_JIFFIES(sched_slice(cfs_rq_of(se), se));

        return rr_interval;
    }
    ```
    - 将 *task* 分配到的执行时间（以纳秒为单位）转换成 *jiffies* 为单位

### SMP 相关
包括的调度类回调有
- `select_task_rq()`
- `migrate_task_rq()`
- `rq_online()`
- `rq_offline()`
- `task_waking()`

1. `select_task_rq_fair` - `select_task_rq()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /*
     * select_task_rq_fair: Select target runqueue for the waking task in domains
     * that have the 'sd_flag' flag set. In practice, this is SD_BALANCE_WAKE,
     * SD_BALANCE_FORK, or SD_BALANCE_EXEC.
     *
     * Balances load by selecting the idlest cpu in the idlest group, or under
     * certain conditions an idle sibling cpu if the domain has SD_WAKE_AFFINE set.
     *
     * Returns the target cpu number.
     *
     * preempt must be disabled.
     */
    static int select_task_rq_fair(struct task_struct *p, int prev_cpu, int sd_flag, int wake_flags)
    {
        struct sched_domain *tmp, *affine_sd = NULL, *sd = NULL;
        int cpu = smp_processor_id();
        int new_cpu = cpu;
        int want_affine = 0;
        int sync = wake_flags & WF_SYNC;

        if (sd_flag & SD_BALANCE_WAKE)
            want_affine = cpumask_test_cpu(cpu, tsk_cpus_allowed(p));

        rcu_read_lock();
        for_each_domain(cpu, tmp) {
            if (!(tmp->flags & SD_LOAD_BALANCE))
                continue;

            /*
            * If both cpu and prev_cpu are part of this domain,
            * cpu is a valid SD_WAKE_AFFINE target.
            */
            if (want_affine && (tmp->flags & SD_WAKE_AFFINE) &&
                cpumask_test_cpu(prev_cpu, sched_domain_span(tmp))) {
                affine_sd = tmp;
                break;
            }

            if (tmp->flags & sd_flag)
                sd = tmp;
        }

        if (affine_sd && cpu != prev_cpu && wake_affine(affine_sd, p, sync))
            prev_cpu = cpu;

        if (sd_flag & SD_BALANCE_WAKE) {
            new_cpu = select_idle_sibling(p, prev_cpu);
            goto unlock;
        }

        while (sd) {
            struct sched_group *group;
            int weight;

            if (!(sd->flags & sd_flag)) {
                sd = sd->child;
                continue;
            }

            group = find_idlest_group(sd, p, cpu, sd_flag);
            if (!group) {
                sd = sd->child;
                continue;
            }

            new_cpu = find_idlest_cpu(group, p, cpu);
            if (new_cpu == -1 || new_cpu == cpu) {
                /* Now try balancing at a lower domain level of cpu */
                sd = sd->child;
                continue;
            }

            /* Now try balancing at a lower domain level of new_cpu */
            cpu = new_cpu;
            weight = sd->span_weight;
            sd = NULL;
            for_each_domain(cpu, tmp) {
                if (weight <= tmp->span_weight)
                    break;
                if (tmp->flags & sd_flag)
                    sd = tmp;
            }
            /* while loop will break here if sd == NULL */
        }
    unlock:
        rcu_read_unlock();

        return new_cpu;
    }
    ```
    - 考虑 “在“普通”的 *SMP* 系统上，所有的处理器都包含在一个调度域中”（《深入 *Linux* 内核架构》），记这个调度域为 `unique_sd`：
      - 如果是由唤醒触发的负载均衡，即 `sd_flag` 中设置了 `SD_BALANCE_WAKE`：
        ```txt
        情况 1：
            1. p 可以在当前 cpu 上运行 -> want_affine = 1
            2. unique_sd 设置了 SD_WAKE_AFFINE -> affine_sd = unique_sd
            3. p 之前运行的 CPU 不是当前的 cpu，且可以在当前 cpu 上唤醒 p (wake_affine() 返回 1) -> prev_cpu = cpu
            4. new_cpu 通过 select_idle_sibling() 取值，它返回一个 p 允许在上面运行的 idle CPU，否则返回 prev_cpu 自身
        情况 2：
            1. p 不可以在当前 cpu 上运行 -> want_affine = 0
            2. 同情况 1 第 4 点
        ```
        - 总的来说，只要通过 `sd_flags` 中设置了 `SD_BALANCE_WAKE`，则目标 *CPU* 是通过 `select_idle_sibling()` 获取的
      - 如果不是由唤醒触发的负载均衡，目标 *CPU* 通过 `find_idlest_cpu()` 获取，它返回一个 `p` 允许在上面运行的 *CPU*，这个 *CPU* 要么是 *idle* 的，要么是这些 *CPU* 中负载最小的（这里的负载指的是平均负载，即 `rq->cfs.runnable_load_avg`）
    1. `cfs_rq->runnable_load_avg` 的更新
        - 实体入队时更新 `enqueue_entity_load_avg()`
            ```c
            // version: 4.1.15
            // kernel/sched/fair.c
            cfs_rq->runnable_load_avg += se->avg.load_avg_contrib;
            ```
        - 实体出队时更新 `dequeue_entity_load_avg()`
            ```c
            // version: 4.1.15
            // kernel/sched/fair.c
            cfs_rq->runnable_load_avg -= se->avg.load_avg_contrib;
            ```
        - *cfs rq* 上实体的负载平均值更新时更新 `update_entity_load_avg()`
            ```c
            // version: 4.1.15
            // kernel/sched/fair.c
            /* Update a sched_entity's runnable average */
            static inline void update_entity_load_avg(struct sched_entity *se,
                                int update_cfs_rq)
            {
                struct cfs_rq *cfs_rq = cfs_rq_of(se);
                long contrib_delta, utilization_delta;
                int cpu = cpu_of(rq_of(cfs_rq));
                u64 now;

                /*
                * For a group entity we need to use their owned cfs_rq_clock_task() in
                * case they are the parent of a throttled hierarchy.
                */
                if (entity_is_task(se))
                    now = cfs_rq_clock_task(cfs_rq);
                else
                    now = cfs_rq_clock_task(group_cfs_rq(se));

                if (!__update_entity_runnable_avg(now, cpu, &se->avg, se->on_rq,
                                cfs_rq->curr == se))
                    return;

                contrib_delta = __update_entity_load_avg_contrib(se);
                utilization_delta = __update_entity_utilization_avg_contrib(se);

                if (!update_cfs_rq)
                    return;

                if (se->on_rq) {
                    cfs_rq->runnable_load_avg += contrib_delta;
                    cfs_rq->utilization_load_avg += utilization_delta;
                } else {
                    subtract_blocked_load_contrib(cfs_rq, -contrib_delta);
                }
            }
            ```
            - `__update_entity_load_avg_contrib()` 函数计算负载平均值的基本原理：
              - 整体的计算公式是：`load_avg = u_0 + u_1*y + u_2*y^2 + ... [re-labeling u_i --> u_{i+1}]`，其中 `u_i` 表示第前 *i* 个 *period* 内的（可）运行时间；`y` 表示衰减因子，且具有 `y^32 = 0.5` 的关系
              - 将 *period* 按 *1ms* 长度分段。不过在代码实现时，将 *ms <-> us <-> ns* 间的转换关系用 `2^10 = 1024` 来近似
              - 负载平均值通过两个数值来记录：实体处于可运行状态的平均值（`se->avg.runnable_avg_sum`）和实体真正得到运行的平均值（`se->avg.running_avg_sum`）。另外，还通过 `se->agv.avg_period` 来记录未满一个 *period* 所经过的时间。它们的更新都满足上述计算公式。不过，在计算每个 *period* 对 `se->avg.running_avg_sum` 的贡献值时，还会乘上一个比例因子，该因子与 `arch_scale_freq_capacity()` 和 `SCHED_CAPACITY_SHIFT` 有关
              - 从 *rq* 上获取的时间值以 *ns* 为单位，在函数中首先会判断：与前一次负载平均值更新时间（记录在 `se->last_runnable_update`）比较，是否已经过去有 *1ms*。只有超过 *1ms* 才会继续后面的更新操作
              - 更新负载平均值的更新时间、滚动更新（`decay_load()` + `__compute_runnable_contrib`）若干个完整 *period* 和当前非整 *period* 时间内对平均负载的贡献。对跨过的完整 *period* 做衰减时，会假设这些跨过的周期内实体一直是可以运行的
              - 结合计算公式可知，随着时间的推移，距离当前越久远的负载值占比负载均值越小，当前的负载情况占有较大的比例
              - 返回值情况：如果过了完整 *period*，返回 1；否则，返回 0。即在该函数中是否涉及到了衰减计算
            - 通过 `__update_entity_load_avg_contrib()` 计算当前实体负载对 *cfs rq* 的 `runnable_load_avg` 的贡献值
              - 实体 `se` 的贡献值记录在 `se->avg.load_avg_contrib` 中
              - 首先，记录上一次的贡献值
              - 通过 `__update_task_entity_contrib()` 更新的贡献值，它与 `se->avg.runnable_avg_sum` 以及 `se->load.weight` 成正比，与 `se->agv.avg_period + 1` 成反比
              - 最后 `update_entity_load_avg_contrib()` 的返回值是新贡献值与旧贡献值之差。
            - 综上，得到如下推论：
              - 如果实体的负载均值减小，会导致 *cfs rq* 的可运行负载均值减小；反之亦然
              - 权重大的实体对 *cfs rq* 的可运行负载均值影响更大
              - 实体在当前 *period* 周期已经运行的时间越长，则它的负载贡献值越小
              - 如果两个 *cfs rq* 上面的可运行实体数量一致，则具体较大权重的 *cfs rq* 的可运行负载均值更大

2. `migrate_task_rq_fair()` - `migrate_task_rq()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    /*
     * Called immediately before a task is migrated to a new cpu; task_cpu(p) and
     * cfs_rq_of(p) references at time of call are still valid and identify the
     * previous cpu.  However, the caller only guarantees p->pi_lock is held; no
     * other assumptions, including the state of rq->lock, should be made.
     */
    static void migrate_task_rq_fair(struct task_struct *p, int next_cpu)
    {
        struct sched_entity *se = &p->se;
        struct cfs_rq *cfs_rq = cfs_rq_of(se);

        /*
        * Load tracking: accumulate removed load so that it can be processed
        * when we next update owning cfs_rq under rq->lock.  Tasks contribute
        * to blocked load iff they have a positive decay-count.  It can never
        * be negative here since on-rq tasks have decay-count == 0.
        */
        if (se->avg.decay_count) {
            se->avg.decay_count = -__synchronize_entity_decay(se);
            atomic_long_add(se->avg.load_avg_contrib,
                            &cfs_rq->removed_load);
        }

        /* We have migrated, no longer consider this task hot */
        se->exec_start = 0;
    }
    ```
    - 任务迁移之前先将其对应的 *exec_start* 清空（设置为 0）

3. `rq_online_fair()` - `rq_online()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static void rq_online_fair(struct rq *rq)
    {
        update_sysctl();

        update_runtime_enabled(rq);
    }
    ```
    - `update_sysctl()` 更新了 `sysctl_sched_min_granularity`、`sysctl_sched_latency` 和 `sysctl_sched_wakeup_granularity` 三个值，它们在原值的基础上乘上了一个系统，该系数是通过 `get_update_sysctl_factor()` 获取的。系数值与参数 `sysctl_sched_tunable_scaling` 有关，该参数的取值与系统的关系如下：
      - `SCHED_TUNABLESCALING_NONE` 时，系数为 1
      - `SCEHD_TUNABLESCALING_LINEAR` 时，系数同当前在线的 *CPU* 数量，最大为 8
      - `SCHED_TUNABLESCALING_LOG` 时（默认时也是），系数是 `1 + log2(cpus)`，`cpus` 为在线的 *CPU* 数量，且 `cpus` 最大为 8
    - 在未选择配置项 `CONFIG_CFS_BANDWIDTH` 时，`update_runtime_enabled()` 为空

4. `rq_offline_fair()` - `rq_offline()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static void rq_offline_fair(struct rq *rq)
    {
        update_sysctl();

        /* Ensure any throttled groups are reachable by pick_next_task */
        unthrottle_offline_cfs_rqs(rq);
    }
    ```
    - `update_sysctl()` 详细信息见上文
    - 在未选择配置项 `CONFIG_CFS_BANDWIDTH` 时，`unthrottle_offline_cfs_rqs()` 为空

5. `task_waking_fair()` - `task_waking()`
    ```c
    // version: 4.1.15
    // kernel/sched/fair.c
    static void task_waking_fair(struct task_struct *p)
    {
        struct sched_entity *se = &p->se;
        struct cfs_rq *cfs_rq = cfs_rq_of(se);
        u64 min_vruntime;

    #ifndef CONFIG_64BIT
        u64 min_vruntime_copy;

        do {
            min_vruntime_copy = cfs_rq->min_vruntime_copy;
            smp_rmb();
            min_vruntime = cfs_rq->min_vruntime;
        } while (min_vruntime != min_vruntime_copy);
    #else
        min_vruntime = cfs_rq->min_vruntime;
    #endif

        se->vruntime -= min_vruntime;
        record_wakee(p);
    }
    ```
    - 该函数更新实体的虚拟运行时间，将其修正（*normalize*）为相对值。在 32 位平台上，只需要在实体的虚拟运行时间上减少 *cfs rq* 的 *min_vruntime* 即可
    - `record_wakee()` 会通过当前运行进程记录这次唤醒信息，涉及的 *task_struct* 字段有：`last_wakee`、`wakee_flips` 等