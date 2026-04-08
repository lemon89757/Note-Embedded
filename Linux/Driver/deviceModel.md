# Linux 内核设备模型
## kojbect
- `kojbect`：统一设备模型中最基本的对象。
    ```c
    /* 
    * @name：该 kobject 的名称，同时也是 sysfs 中的目录的名称。
    *        由于 kobject 添加到内核时，需要根据名字注册到 sysfs 中，之后就不能再直接修改该字段。
    *        如果需要修改 kobject 的名字，需要调用 kobject_rename() 接口，该接口会主动处理 sysfs 的相关事宜。
    * @entry：用于将 kobject 加入到 kset 中的 list_head。
    * @parent：指向 parent kobject，以此形成层次结构（在 sysfs 中表现为目录结构）。
    * @kset：该 kobject 属于的 kset，可以为 NULL。
    *        如果存在，且没有指定 parent，则会把 kset 作为 parent（kset 是一个特殊的 kobject）。
    * @ktype：该 kobject 属于的 kobj_type。每个 kobject 必须有一个 ktype，否则内核会提示错误。
    * @sd：该 kobject 在 sysfs 中的表示。
    * @kref：为一个可以用于原子操作的引用计数。
    * @state_initialized：指示该 kobject 是否已经初始化，以在 kobject 的 Init、Put、Add 等操作时进行异常校验。
    * @state_in_sysfs：指示该 kobject 是否已经在 sysfs 中呈现，以便在自动注销时从 sysfs 中移除。
    * @state_add_uevent_sent：与下述字段组成一对，记录是否已经向用户空间发送 ADD uevent。
    *                         如果有，且没有发送 REMOVE uevent，则在自动注销时，补发 REMOVE uevent，以便让用户空间正确处理。
    * @state_remove_uevent_sent
    * @uevent_suppress：如果该字段为 1，则表示忽略所有上报的 uevent 事件。
    */
    struct kobject {
        const char *name;
        struct list_head entry;
        struct kobject *parent;
        struct kset *kset;
        struct kobj_type *ktype;
        struct sysfs_dirent *sd;
        struct kref kref;
        unsigned int state_initialized:1;
        unsigned int state_in_sysfs:1;
        unsigned int state_add_uevent_sent:1;
        unsigned int state_remove_uevent_sent:1;
        unsigned int uevent_suppress:1;
    }
    ```
    - uevent 提供了“用户空间通知”的功能实现，通过该功能，当内核中有 kobject 增加、删除、修改等动作时，会通知用户空间。
- `kobj_type`：代表 `kobject` 的属性操作集合，严格地讲，是包含了 `kobject` 的数据结构。由于通用性，多个 `kobject` 可能共用同一个属性操作集，因此把 `ktype` 独立出来了。
    ```c
    /* 
     * @release：通过该回调函数，可以将包含该种类型 kobject 的数据结构的内存空间释放掉。
     * @sysfs_ops：该种类型的 kobject 的 sysfs 文件系统接口。
     * @default_attrs：该种类型的 kobject 的 attribute 列表，
     *                 attribute 在 sysfs 中表现为 kobject 对应目录下的一个文件，
     *                 在 kobject 添加到内核时，一并注册到 sysfs 中。
     * @child_ns_type：和文件系统（sysfs）的命名空间有关。
     * @namespace：和文件系统（sysfs）的命名空间有关。
     */
    struct kobj_type {
        void (*release)(struct kobject *kobj);
        const struct sysfs_ops *sysfs_ops;
        struct attribute **default_attrs;
        const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
        const void *(*namespace)(struct kobject *kobj);
    }
    ```
    - 每一个内嵌 `kobject` 的数据结构，例如 `kset`、`device`、`device_driver` 等，都要实现自己的 `kobj_type`，并定义其中的回调函数。
- `kset`：一个特殊的 `kobject`（因此它也会在 `sysfs` 文件系统中以目录的形式出现），它用来集合相似的 `kobject`（这些 `kobject` 可以是相同属性的，也可以不同属性的）。
    ```c
    /* 
     * @list：用于保存该 kset 下所有的 kobject 的链表。
     * @list_lock：list 自旋锁。
     * @kobj：该 kset 自己的 kobject（kset 是一个特殊的 kobject，也会在 sysfs 中以目录的形式出现）。
     * @uevent_ops：该 kset 的 uevent 操作函数集，
     *              当任何 kobject 需要上报 uevent 时，都要调用它所从属的 kset 的 uevent_ops，添加环境变量，或者过滤 event（kset 可以决定哪些 event 可以上报），
     *              因此，如果一个 kobject 不属于任何 kset 时，是不允许发送 uevent 的。
     */
    struct kset {
        struct list_head list;
        spinlock_t list_lock;
        struct kobject kobj;
        const struct kset_uevent_ops *uevent_ops;
    }

    /* 
     * @filter：当任何 kobject 需要上报 uevent 时，它所属的 kset 可以通过该接口过滤，阻止不希望上报的 uevent，从而达到从整体上管理的目的。
     * @name：该接口可以返回 kset 的名称。如果一个 kset 没有合法的名称，则其下所有的 kobject 将不允许上报 uevent。
     * @uevent：当任何 kobject 需要上报 uevent 时，它所属的 kset 可以通过该接口统一为这些 event 添加环境变量。
     *          因为很多时候上报 uevent 时的环境变量都是相同的，因此可以由 kset 统一处理，就不需要让每个 kobject 独自添加了。
     */
    struct kset_uevent_ops {
        int (*const filter)(struct kset *kset, struct kobject *kobj);
        const char *(*const name)(struct kset *kset, struct kobject *obj);
        int (*const uevent)(struct kset *kset, struct kobject *obj, struct kobj_uevent_env *env);
    }
    ```
    - 注意 `kset` 和 `ktype` 的关联，`kobject` 会利用成员 `kset` 找到自己所属的 `kset`，然后才设置自身的 `ktype` 为 `kobj.ktype`。当没有指定 `kset` 成员时，才会利用 `ktype` 来建立关系。
    - 由于 `kobject` 调用的是它所属 `kset` 的 `uevent` 操作函数，所以 `kset` 可以对其行为进行控制。如果 `kobject` 不属于任何 `kset`，则无法发送 `uevent`。
- 总结：
  1. `kobject` 的核心功能是：保持一个引用计数，当该计数减为 0 时，自动释放 `kobject` 所占用的内存空间（由 kobject 模块负责）。这就决定了 `kobject` 必须是动态分配的（只有这样才能动态释放）（`kobject` 不能被静态声明或者定义在栈上，见笔记 `Documentation/kobject.md`）。
  2. `kobject` 的大多数使用场景是内嵌在大型的数据结构中（如 `kset`、`device_driver` 等），因此这些大型数据结构也必须动态分配、动态释放。释放时机与内嵌的 `kobject` 一致。
  3. 大型数据结构内存空间的一并释放需要借助 `ktype`。`ktype` 中的 `release()` 回调函数负责释放 `kobject`（甚至是包含 `kobject` 的数据结构）的内存空间。那么 `ktype` 及其内部函数，是由谁实现呢？是由上层数据结构所在的模块！因为只有它，才清楚 `kobject` 内嵌在哪个数据结构中，并通过 `kobject` 指针以及自身的数据结构类型，找到需要释放的上层数据结构的指针，然后释放它。
  4. 综上，每一个内嵌 `kobject` 的数据结构，都要实现一个 `ktype`，并且定义其中的回调函数。同理，**sysfs 相关的操作也一样，必须经过 `ktype` 的中转，因为 sysfs 看到的是 `kobject`，而真正的文件操作的主体，是内嵌 `kobject` 的上层数据结构**。 

## mdev
- mdev 和 sysfs 的关系
    mdev 在 busybox 的代码包中能找到，位于 `busybox/util-linux/mdev.c` 文件中，它通过 `uevent_helper()` 函数被调用。在 mdev 主要完成两件事情：
    - 执行 `mdev -s` 命令时，mdev 扫描 `/sys/block` 和 `/sys/class` 两个目录下的 dev 属性文件，从该 dev 属性文件中获取到设备编号，并以包含该 dev 属性文件的目录名称作为设备名 device_name，在 `/dev` 目录下创建相应的设备文件。例如，`cat /sys/class/tty/tty0/dev` 会得到 `4:0`，subsystem 为 tty，device_name 为 tty0。
      - 补充-1：块设备保存在 `/sys/block` 目录下，内核 2.6.25 版本以后，块设备也保存在 `/sys/class/block` 目录下。mdev 扫描 `/sys/block` 是为了实现向后兼容。
      - 补充-2：dev 属性文件以 `major:minor` 的形式保存设备编号。
      - 补充-3：包含 dev 属性文件的目录称为 device_name，而 `/sys/class/` 和 devie_name 之间的那部分目录称为 subsystem。也就是每个 dev 属性文件所在的路径都可以表示为 `/sys/class/subsystem/device_name/dev`。
    - 当 mdev 因 uevent 事件（以前叫 hotplug 事件）被调用时，mdev 通过由 uevent 事件传递给他的环境变量获取到：引起该 uevent 事件的设备 action 以及该设备所在的路径 device path。然后判断引起该 uevent 事件的 action 是什么，并根据 action 的不同做相应操作。
      - 若 action 是 add，即有新设备加入到系统中，不管该设备是虚拟设备还是实际物理设备，mdev 都会通过 device path 路径下的 dev 属性文件获取到设备编号，然后以 device path 路径最后一个目录作为设备名，在 `/dev` 目录下创建相应的设备文件。
      - 若 action 是 remote，即设备已从系统中移除，则删除 `/dev` 目录下以 device path 路径最后一个目录名称作为文件名的设备文件。
      - 若 action 既不是 add，也不是 remote，mdev 则什么都不做。

    由上可知，如果需要在设备加入到系统中或从系统中移除时，由 mdev 自动地创建和删除设备文件，则前提条件是：
    1. 在 `/sys/class` 的某一 subsystem 目录下，创建一个以设备名 device_name 作为名称的目录；
    2. 此目录下还必须包含一个 dev 属性文件，该 dev 属性文件以 `major:minor` 形式输出设备编号。

    由上可知，**sysfs 为 uevent 机制做前期的准备工作，即创建相应的目录**，而 mdev 则是在 sysfs 的基础上通过调用 sysfs 创建的目录或文件来实现设备节点的创建。

## uevent
uevent 本质上是内核对象（kobject）**状态变化时向用户空间广播的事件通知**。当硬件设备插入、移除或状态改变时，内核通过 uevent 机制通知用户空间的守护进程（如 udev、systemd），后者据此创建设备节点、加载驱动、设置权限或执行自定义脚本---[电子发烧友网](https://www.elecfans.com/d/5948085.html)。

uevent 是实现热插拔（Hotplug）和动态设备管理的基础。

1. uevent 机制创建设备节点
    ```c
    class_create(THIS_MODULE, "buttonsdrv");
        class_register(cls);
            kobject_set_name(&cls->subsys.kobj, "%s", cls->name);   // 将类的名字 led_class 赋值给对应的 kset
        subsys_set_kset(cls, class_subsys);
        subsystem_register(&cls->subsys);
            kset_register(s);   // 创建 class 设备目录
            kset_add(k);
                    kobject_add(&k->kobj);
                kobject_shadow_add(kobj, NULL);
                        parent = kobject_get(kobj->parent);
                    list_add_tail(&kobj->entry, &kobj->kset->list);
                    create_dir(kobj, shadow_parent);    // 创建一个 class 设备类目录
                    sysfs_create_dir(kobj, shadow_parent);  // 该接口是 sysfs 文件系统接口，代表创建一个目录
    ```
    - kobject 在 sysfs 中对应的是目录；
    - 当我们注册一个 kobject 时，会调用 `kobject_add(&k->obj);`，紧接着创建 class 设备目录。

2. uevent 将事件上报到用户空间的两种方式：
   - 通过 kmod 模块，直接调用用户空间的可执行程序或脚本。
   - 通过 netlink 通信机制，将事件从内核空间传递到用户空间。

    通过 kmod 模块，直接调用用户空间的可执行程序或脚本：
    ```c
    // uevent 通过 kobject.h 提供了如下 API
    // 在 lib/kobject_uevent.c 中实现
    int kobject_uevent(struct kobject *kobj, enum kobject_action action);
    int kobject_uevent_env(struct kobject *kobj, enum kobject_action action, char *envp[]);
    int add_uevent_var(struct kobject_uevent_env *env, const char *format, ...);
    int kobject_action_type(const char *buf, size_t count, enum kobject_action *type);

    
    // kobject_action：
    // /include/linux/kobject.h
    enum kobject_action {
        KOBJ_ADD,
        KOBJ_REMOVE,
        // kobject 或其上层数据结构的状态或者内容发生改变。
        // 如果设备驱动需要上报的事件不在上面事件的范围内，或者是自定义的事件，可以使用该 event，并携带相应的参数
        KOBJ_CHANGE,
        // kobject 或其上层数据结构更改名称或者更改 parent（意味着在 sysfs 中更改了目录结构）
        KOBJ_MOVE,
        // ONLINE、OFFLINE，kobject 或其上层数据结构的上线、下线事件，其实是是否使能
        KOBJ_ONLINE,
        KOBJ_OFFLINE,
        KOBJ_MAX
    };


    int kobject_uevent(struct kobject *kobj, enum kobject_action action) {
        return kobject_uevent_env(kobj, action, NULL);
    }

    /* 
     * 发送一个带有环境变量的事件
     * @kobj：事件发送的 kobject 结构体
     * @action：发生的事件
     * @envp_ext：环境变量数据指针 
     */
     int kobject_uevent_env(struct kobject *kobj, enum kobject_acton action, char *envp_ext[]) {
        action_string = action_to_string(action);

        top_kobj = kobj;
        while (!top_kobj->kset && top_kobj->parent) {
            top_kobj = top_kobj->parent;
        }
        if (!top_kobj->kset) {
            pr_debug("kobject attempted to send uevent without kset!\n");
            return -EINVAL;
        }

        kset = top_kobj->set;
        uevent_ops = kset->uevent_ops;

        if (uevent_ops && uevent_ops->filter) {
            if (!uevent_ops->filter(kset, kobj)) {
                pr_debug("kobject filter function caused the event to drop!\n");
                return 0;
            }
        }

        if (uevent_ops && uevent_ops->name)
            subsystem = uevent_ops->name(kset, kobj);
        else
            subsystem = kobject_name(&kset->kobj);
        if (!subsystem) {
            pr_debug("unset subsystem caused the event to drop!\n");
            return 0;
        }

        // 分配一个此次上报的环境变量
        envp = kzalloc(NUM_ENVP * sizeof(char *), GFP_KERNEL);
        if (!envp)
            return !ENOMEN;

        buffer = kmalloc(BUFFER_SIZE, GFP_KERNEL);
        if (!buffer) {
            retval = -ENOMEN;
            goto exit;
        }

        // 获得该 kojbect 在 sysfs 中路径
        devpath = kobject_get_path(kobj, GFP_KERNEL);
        if (!devpath) {
            retval = -ENOMEN;
            goto exit;
        }

        // uevent_helper 的环境变量
        envp[i++] = "HOME=/";
        envp[i++] = "PATH=/sbin:/bin:/usr/sbin:/usr/bin";
        // 添加环境变量
        scratch = buffer;
        envp[i++] = scratch;
        scratch += sprintf(scratch, "ACTION=%s", action_string) + 1;
        envp[i++] = scratch;
        scratch += sprintf(scratch, "DEVPATH=%s", devpath) + 1;
        envp[i++] = scratch;
        scratch += sprintf(scratch, "SUBSYSTEM=%s", subsystem) + 1;
        for (j = 0; envp_ext && evnp_ext[j]; j++) {
            envp[i++] = envp_ext[j];
        }
        // just reserve the space, overwrite it after kset call has returned
        envp[i++] = seq_buf = scratch;
        scratch += strlen("SEQNUM=188446744073709551616") + 1;

        // 如果 uevent_ops->uevent 存在，调用该接口，添加 kset 统一的环境变量到 env 指针
        if (uevent_ops && uevent_ops->uevent) {
            retval = uevent_ops->uevent(kset, kobj, &envp[i], NUM_ENVP - i, scratch, BUFFER_SIZE - (scratch - buffer));
            if (retval) {
                pr_debug("%s - uevent() returned %d\n", __FUNCTION__, retval);
                goto exit;
            }
        }

        // 调用 add_uevent_var 接口，添加格式为 "SEQNUM=%llu" 的序列号
        spin_lock(&sequence_lock);
        seq = ++uevent_seqnum;
        spin_lock(&sequence_lock);
        sprintf(seq_buff, "SEQNUM=%llu", (unsigned long long)seq);

        // 以 uevent_helper、subsystem 以及添加了环境变量的 env 指针为参数
        // 调用 kmod 模块提供的 call_usermodehelper 函数，上报 uevent
        if (uevent_helper[0]) {
            char *argv[3];
            argv[0] = uevent_helper;
            argv[1] = (char*)subsystem;
            argv[2] = NULL;
            call_usermodehelper(argv[0], argv, envp, 0);
        }
     }
    ```

## 参考
1. [CSDN-嵌入式 Linux--uevent 机制：uevent 原理分析](https://blog.csdn.net/W1107101310/article/details/80211885)
2. [蜗窝科技-统一设备模型](http://www.wowotech.net/sort/device_model/page/2)
