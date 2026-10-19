# Linux 内核设备模型
## 概述
内核设备模型架构图见：`Linux/Driver/image.drawio:bus-device-driver arch`。

内核对象的创建：从顶至底的过程；内核对象的删除：从底至顶的过程

## 内核对象、类型和集合
### kobject
#### 概述
统一设备模型中最基本的对象。

#### 数据结构描述
```c
// include/linux/kobject.h
/**
 * @name: 该 kobject 的名称，同时也是 sysfs 中的目录的名称。
 *        由于 kobject 添加到内核时，需要根据名字注册到 sysfs 中，因此该字段不能直接修改。
 *        如果需要修改 kobject 的名字，需要调用 kobject_rename()，该接口会主动处理 sysfs 的相关事宜。
 * @entry: 用于将 kobject 加入到 kset 中的 list_head。
 * @parent: 指向 parent kobject，以此形成层次结构（在 sysfs 中表现为目录结构）。
 * @kset: 该 kobject 属于的 kset，可以为 NULL。
 *        如果存在，且没有指定 parent，则会把 kset 作为 parent（kset 是一个特殊的 kobject）。
 * @ktype: 该 kobject 属于的 kobj_type。每个 kobject 必须有一个 ktype，否则内核会提示错误。
 * @sd: 该 kobject 在 sysfs 中的表示。
 * @kref: 一个可以用于原子操作的引用计数。
 * @state_initialized: 指示该 kobject 是否已经初始化，以在 kobject 的 Init、Put、Add 等操作时进行异常校验。
 * @state_in_sysfs: 指示该 kobject 是否已经在 sysfs 中呈现，以便在注销时从 sysfs 中移除。
 * @state_add_uevent_sent: 与下述字段组成一对，记录是否已经向用户空发送 ADD uevent。
 *                         如果有，且没有发送 REMOVE uevent，则在注销时，发送 REMOVE uevent，以便让用户空间正确处理。
 * @state_remove_uevent_sent: 
 * @uevent_suppress: 如果该字段为 1，则表示忽略所有上报的 uevent 事件。
 */
struct kobject {
    const char *name;
    struct list_head entry;
    struct kobject *parent;
    struct kset *kset;
    struct kobj_type *ktype;
    struct kernfs_node *sd;
    struct kref kref;
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};
```

#### 注册
- 注册流程
  1. 初始化
  2. 注册
  - 初始化 - `kobject_init()`
    ```c
    // version: 4.1.15
    // lib/kobject.c
    /**
     * kobject_init - initialize a kobject structure 
     * @kobj: pointer to the kobject to initialize
     * @ktype: pointer to the ktype for this kobject.
     *
     * This function will properly initialize a kobject such
     * that it can then be passed to the kobject_add() call.
     * 
     * After this function is called, the kobject MUST be cleaned
     * up by a call to kobject_put(), not by a call to kfree
     * directly to ensure that all of the memory is cleaned up
     * properly.
     */
    void kobject_init(struct kobject *kobj, struct kobj_type *ktype) {
        char *err_str;
        if (!kobj) {
            err_str = "invalid kobject pointer!";
            goto error;
        }
        if (!ktype) {
            err_str = "must have a ktype to be initialized properly!\n";
            goto error;
        }
        if (kobj->state_initialized) {
            /* do not error out as sometimes we can recover */
            printk(KERN_ERR "kobject (%p): tried to init an initialized "
            "object, something is seriously wrong.\n", kobj);
            dump_stack();
        }

        kobject_init_internal(kobj);
        kobj->ktype = ktype;
        return;
    error:
        printk(KERN_ERR "kobject (%p): %s\n", kobj, err_str);
        dump_stack();
    }
    EXPORT_SYMBOL(kobject_init);

    static void kobject_init_internal(struct kobject *kobj) {
        if (!kobj)
            return;
        kref_init(&kobj->kref);
        INIT_LIST_HEAD(&kobj->entry);
        kobj->state_in_sysfs = 0;
        kobj->state_add_uevent_sent = 0;
        kobj->state_remove_uevent_sent = 0;
        kobj->state_initialized = 1;
    }
    ```
    - `kobject_init()` 要求给定的参数都要有效（不能为空）。
    - `kobject_init()` 会设置 *kobject* 实例的标志位，只有 `state_initialized` 被置位；
    - 初始化 `entry` 字段以便于连接到 *kset* 的链表上；
    - 将 *kobject* 实例的引用计数设置为 1；
    - 同时指定 *kobj_type* 类型为给定的 `ktype`。
    - 经过 `kobject_init()` 的 *kobject* 实例如果需要释放，不能直接调用 `kfree()`，唯一正确的方式是调用 `kobject_put()`。
  - 注册 - `kobject_add()`
    ```c
    // version: 4.1.15
    // lib/kobject.c
    /** 
    * kobject_add - the main kobject add function 
    * @kobj: the kobject to add
    * @parent: pointer to the parent of the kobject
    * @fmt: format to name the kobject with
    *
    * The kobject name is set and added to the kobject hierarchy
    * in this function.
    * If @parent is set, then the parent of the @kobj will be
    * set to it. If @parent is NULL, then the parent of the 
    * @kobj will be set to the kobject associated with the kset
    * assigned to this kobject. If no kset is assigned to the kboject,
    * then the kobject will be located in the root of the sysfs tree.
    *
    * If this function returns an error, kobject_put() must be 
    * called to properly clean up the memory associated with the
    * kobject. Under no instance should the kobject that is passed
    * to this function be direclty freed with a call to kfree(),
    * that can leak memory.
    *
    * Note, no "add" uevent will be created with this call, the
    * caller should set up all of the necessary sysfs files for
    * the object and then call kobject_uevent() with the UEVENT_ADD
    * parameter to ensure that userspace is properly notified
    * of this kobject's creation.
    */
    int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...) {
        va_list args;
        int retval;
        if (!kobj)
            return -EINVAL;
        if (!kobj->state_initialized) {
            printk(KERN_ERR "kobject '%s' (%p): tried to add an"
            "uninitialized object, something is seriously wrong.\n",
            kobject_name(kobj), kobj);
            dump_stack();
            return -EINVAL;
        }
        va_start(args, fmt);
        retval = kobject_add_varg(kobj, parent, fmt, args);
        va_end(args);
        return retval;
    }
    EXPORT_SYMBOL(kobject_add);

    static int kobject_add_varg(struct kobject *kobj, struct kobject *parent, const char *fmt, va_list args) {
        int retval;
        retval = kobject_set_name_vargs(kobj, fmt, vargs);
        if (retval) {
            printk(KERN_ERR "kobject: can not set name properly!\n");
            return retval;
        }
        kobj->parent = parent;
        return kobject_add_internal(kobj);
    }

    static int kobject_add_internal(struct kobject *kobj) {
        int error = 0;
        struct kobject *parent;

        if (!kobject)
            return -ENOENT;
        if (!kobj->name || !kobj->name[0]) {
            // ... error handling
        }
        
        parent = kobject_get(kobj->parent);
        /* join kset if set, use it as parent if we do not 
            * already have one
            */
        if (kobj->kset) {
            if (!parent)
                parent = kobject_get(&kobj->kset->kobj);
            kobj_kset_join(kobj);
            kobj->parent = parent;
        }

        pr_debug(...);

        error = create_dir(kobj);
        if (error) {
            kobj_kset_leave(kobj);
            kobject_put(parent);
            kobj->parent = NULL;
            // ... error handling
        } else {
            kobj->state_in_sysfs = 1;
        }
        return error;
    }

    /* add the kobject to its kset's list */
    static void kobj_kset_join(struct kobject *kobj) {
        if (!kobj->kset)
            return;
        kset_get(kobj->kset);
        spin_lock(&kobj->kset->list_lock);
        list_add_tail(&kobj->entry, &kobj->kset->list);
        spin_unlock(&kobj->kset->list_lock);
    }

    static int create_dir(struct kobject *kobj) {
        const struct kobj_ns_type_operations *ops;
        int error;
        error = sysfs_create_dir_ns(kobj, kobject_namespace(kobj));
        if (error)
            return error;
        error = populate_dir(kobj);
        if (error) {
            sysfs_remove_dir(kobj);
            return error;
        }
        /* 
            * @kobj->sd may be deleted by an ancestor going away.
            * Hold an extra reference so that it stays until @kobj
            * is gone.
            */
        sysfs_get(kobj->sd);
        /* 
            *If @kobj has ns_ops, its children need to be filtered 
            * based on their namespace tags. Enable namespace support
            * on @kboj->sd.
            */
        ops = kobj_child_ns_ops(kobj);
        if (ops) {
            BUG_ON(ops->type <= KOBJ_NS_TYPE_NONE);
            BUG_ON(ops->type >= KOBJ_NS_TYPES);
            BUG_ON(!kobj_ns_type_registered(ops->type));
            sysfs_enable_ns(kobj->sd);
        }
        return 0;
    }
    ```
    - 通过 `kobject_add()` 设置 *kobject* 的名字，并与父对象间建立层级关系，然后在 *sysfs* 中创建入口目录。
    - 如果提供了 `parent` 参数，则它将会被设置为 *kobject* 的父对象；如果未提供 `parent` 参数，即 `parent = NULL`，那么 *kobject* 的父对象则会指向 *kobject* 的 `kset->kobj`。如果 *kobject* 的 `kset` 未被设置，则 *kobject* 位于 *sysfs* 的顶层目录，即用户空间的 `/sys` 中。
    - 如果使用了该函数，则不能使用 `kfree()` 来直接释放 *kobject* 的内存资源，而是使用 `kobject_put()`。
    - 该函数不会向用户空间发送 *uevent* 事件，调用者应当设置好对象的所有必要 *sysfs* 文件，然后再调用 `kobject_uevent()` 并提供 `UEVENT_ADD` 参数来通知用户空间该 *kobject* 的创建。
    - 设置 *kobject* 的 `name`：`kobject_set_name_vargs()` 会检查 name 中是否含有 *'/'*，有的话将其替换为 *'!'*。
    - 设置 *kobject* 的 `parent`：如上所述。
    - 将 *kobject* 挂到它的 *kset* 的链表上（如果 *kobject* 已被设置 *kset*）：`kobj_kset_join()`。
    - 在 sysfs 中创建目录入口：`create_dir`。
      - `sysfs_create_dir_ns()`, `sysfs_remove_dir()` 和 `sysfs_enable_ns()` 见笔记 `sysfs.md`。
      - `populate_dir()`：`lib/kobject.c`，Most subsystems have a set of default attributes that are associated with an object that registers with them. This is a helper called during object registration that loops through the default attributes of the subsystem and creates attributes files for them in sysfs. 它会获取 kobject 对应的 kobj_type，并创建 kobj_type 提供的默认属性文件。
      - [ ] `kobj_child_ns_ops()` 和 `kobj_ns_type_registered()`。

- *helper func*
  1. `kobject_create()`：内存分配 + `kobject_init()`，指定的 *kobj_type* 为 `dynamic_kobj_ktype`。
    ```c
    // version: 4.1.15
    // lib/kobject.c
    /** 
     * kobject_create - create a struct kobject dynamically 
     *
     * This function creates a kobject struct dynamically and 
     * sets it up to be a "dynamic" kobject with a default 
     * release fucntion set up.
     * 
     * If the kobject was not able to be created, NULL will be returned.
     * The kobject structure returned from here must be cleaned up
     * with a call to kobject_put() and not kfree(), as kobject_init()
     * has already been called on this structure.
     */
    struct kobject *kobject_create(void) {
        struct kobject *kobj;

        kobj = kzalloc(sizeof(*kobj), GFP_KERNEL);
        if (!kobj)
            return NULL;
        kobject_init(kobj, &dynamic_kobj_ktype);
        return kobj;
    }
    ```
  
  2. `kobject_create_and_add()`：`kobject_create()` + `kobject_add()`
    ```c
    // version: 4.1.15
    // lib/kobject.c
    /**
     * kobject_create_and_add - create a struct kobject 
     * dynamically and register it with sysfs
     *
     * @name: the name for the kobject
     * @parent: the parent kobject of this kobject, if any.
     *
     * This function creates a kobject structure dynamically and
     * registers it with sysfs. When you are finished with this
     * structure, call kobject_put() and the structure will be
     * dynamically freed when is is no longer beging used.
     *
     * If the kobject was not able to be created, NULL will be
     * returned.
     */
    struct kobject *kobject_create_and_add(const char *name, struct kobject *parent) {
        struct kobject *kobj;
        int retval;
        kobj = kobject_create();
        if (!kobj)
            return NULL;
        retval = kobject_add(kobj, parent, "%s", name);
        if (retval) {
            printk(KERN_WARNING "%s: kobject_add error: %d\n", 
                __func__, retval);
            kobject_put(kobj);
            kobj = NULL;
        }
        return kobj;
    }
    EXPORT_SYMBOL_GPL(kobject_create_and_add);
    ```

  3. `kobject_get_path()`：获取 *kobject* 的层级路径。
    ```c
    // version: 4.1.15
    // lib/kobject.c
    /** 
     * kobject_get_path - generate and return the path 
     * associate with a given kobj and kset pair.
     * @kobj: kobject in question, with which to build the path
     * @gfp_mask: the allocation type used to allocate the path
     *
     * The result must be freed by the caller with kfree().
     */
    char *kobject_get_path(struct kobject *kobj, gfp_t gfp_mask) {
        char *path;
        int len;
        len = get_kobj_path_length(kobj);
        if (len == 0)
            return NULL;
        path = kzalloc(len, gfp_mask);
        if (!path)
            return NULL;
        fill_kobj_path(kobj, path, len);
        return path;
    }
    EXPORT_SYMBOL_GPL(kobject_get_path);
    ```
    - 自身 `kobject_name()` 长度加上父对象方向回溯时所有 *kobject* 的 `kobject_name()` 长度，并在每次累加时进行额外的加 1 操作，用来放置 *"/"*。
    - 再次从 *kobject* 自身出发，往父对象方向遍历，并将所有的 `kobject_name()` 和 *"/"* 填入申请的内存中。填写方向是从后向前。

- `dynamic_kobj_ktype` 和 `kset_ktype`
    ```c
    // lib/kobject.c
    /* default kobject attribute operations */
    static ssize_t kobj_attr_show(struct kobject *kobj, struct attribute *attr, char *buf) {
        struct kobj_attribute *kattr;
        ssize_t ret = -EIO;

        kattr = container_of(attr, struct kobj_attribute, attr);
        if (kattr->show)
            ret = kattr->show(kobj, kattr, buf);
        return ret;
    }

    static ssize_t kobject_attr_store(struct kobject *kobj, struct attribute *attr, const char *buf, size_t count) {
        struct kobj_attribute *kattr;
        ssize_t ret = -EIO;

        kattr = container_of(attr, struct kobj_attribute, attr);
        if (kattr->store)
            ret = kattr->store(kobj, kattr, buf, count);
        return ret;
    }

    const struct sysfs_ops kobj_sysfs_ops = {
        .show = kobj_attr_show,
        .store = kobj_attr_store,
    };
    EXPORT_SYMBOL_GPL(kobj_sysfs_ops);

    static void kset_release(struct kobject *kobj) {
        struct kset *kset = container_of(kobj, struct kset, kobj);
        pr_debug("kobject: '%s' (%p): %s\n", kobject_name(kobj), kobj, __func__);
        kfree(kset);
    }

    static struct kobj_type kset_ktype = {
        .sysfs_ops = &kobj_sysfs_ops,
        .release = kset_release,
    };

    static void dynamic_kobj_release(struct kobject *kobj) {
        pr_debug("kobject: (%p): %s\n", kobj, __func__);
        kfree(kobj);
    }

    static struct kobj_type dynamic_kobj_ktype = {
        .release = dynamic_kobj_release,
        .sysfs_ops = &kobj_sysfs_ops,
    };
    ```

#### 注销
注销步骤（*remove* 机制）：
  1. 将 *kobject* 的引用计数减 1
  2. 当 *kobject* 的引用计数变为 0 时，释放其内存

- `kobject_put()`
    ```c
    // version: 4.1.15
    // lib/kobject.c
    /**
     * kobject_put - decrement refcount for object. 
     * @kobj: object.
     *
     * Decrement the refcount, and if 0, call kobject_cleanup().
     */
    void kobject_put(struct kobject *kobj) {
        if (kobj) {
            if (!kobj->state_initialized)
                WARN(1 KERN_WARNING "kobject: '%s' (%p): is not"
                " initialized, yet kobject_put() is being "
                "called.\n", kobject_name(kobj), kobj);
            kref_put(&kobj->kref, kobject_release);
        }
    }

    // include/linux/kref.h
    /**
     * kref_put - decrement refcount for object. 
     * @kref: object.
     * @release: pointer to the function that will clean up the
     *  object when the last reference to the object is released.
     *  This pointer is required, and it is not acceptable to pass
     *  kfree in as this function. If the caller does pass kfree
     *  to this function, you will be publicly mocked mercilessly
     *  by the kref maintainer, and anyone else who happens to
     *  notice it. You have been warned.
     *
     * Decrement the refcount, and if 0, call release().
     * Return 1 if the object was removed , otherwise return 0.
     * Beware, if this function returns 0, you still can not count
     * on the kref from remaining in memory. Only use the return
     * value if you want to see if the kref is now gone, not present.
     */
    static inline int kref_put(struct kref *kref, void (*release)(struct kref *kref)) {
        return kref_sub(kref, 1, release);
    }

    // 注释略，与 kref_put 大致相同
    static inline int kref_sub*(struct kref *kref, unsigned int count, void (*release)(struct kref *kref)) {
        WARN_ON(release == NULL);
        if (atomic_sub_and_test((int)count, &kref->refcount)) {
            release(kref);
            return 1;
        }
        return 0;
    }

    // lib/kobject.c
    static void kobject_release(struct kref *kref) {
        struct kobject *kobj = container_of(kref, struct kobject, kref);
    #ifdef CONFIG_DEBUG_KOBJECT_RELEASE
        // ...
    #else
        kobject_cleanup(kobj);
    #endif
    }

    static void kobject_cleanup(struct kobject *kobj) {
        struct kobj_type *t = get_ktype(kobj);
        const char *name = kobj->name;
        pr_debug("kobject: '%s' (%p): %s, parent %p\n", 
            kobject_name(kobj), kobj, __func__, kobj->parent);
        if (t && !t->release)
            pr_debug("kobject: '%s' (%p): does not have a release()"
                " function, it is broken and must be fixed.\n", 
                kobject_name(kobj), kobj);
        /* send "remove" if the caller did not do it but send "add" */
        if (kobj->state_add_uevent_sent &&
            !kobj->state_remove_uevent_sent) {
            pr_debug("kobject: '%s' (%p): auto cleanup 'remove' event\n", 
                kobject_name(kobj), kobj);
            kobject_uevent(kobj, KOBJ_REMOVE);
        }
        /* remove from sysfs if the caller did not do it */
        if (kobj->state_in_sysfs) {
            pr_debug("kobject: '%s' (%p): auto cleanup kobject_del\n", 
                kobject_name(kobj), kobj);
            kobject_del(kobj);
        }
        if (t && t->release) {
            pr_debug("kobject: '%s' (%p): calling ktype release\n", 
                kobject_name(kobj), kobj);
            t->release(kobj);
        }
        /* free name if we allocated it */
        if (name) {
            pr_debug("kobject: '%s': free name\n", name);
            kfree(name);
        }
    }

    // version: 4.1.15
    // lib/kobject.c
    /* kobject_del - unlink kobject from hierarchy. */
    void kobject_del(struct kobject *kobj) {
        struct kernfs_node *sd;
        if (!kobj)
            return;
        sd = kobj->sd;
        sysfs_remove_dir(kobj);
        sysfs_put(sd);
        kobj->state_in_sysfs = 0;
        kobj_kset_leave(kobj);
        kobject_put(kobj->parent);
        kobject->parent = NULL;
    }
    ```
    - 当引用计数为 0 时，释放 *kobject*。该功能通过 `kobject->ktype->release()` 回调完成，同时释放 `kobject->name` 对应内存。

#### sysfs
每个 *kobject* 通过 *sysfs* 文件系统向用户空间提供一个 `/sys` 下目录入口，与该 *kobject* 相关的属性信息（`/sys` 对应目录下的文件）创建于此目录下。 

详细信息见笔记： `Linux/Kernel/FileSystem/sysfs.md`

#### uevent
*uevent* 提供了“用户空间通知”的功能实现。通过该功能，当内核中有 *kobject* 增加、删除、修改等动作时，会通知用户空间。

*uevent* 本质上是内核对象（*kobject*）状态变化时向用户空间广播的事件通知。当硬件设备插入、移除或状态改变时，内核通过 *uevent* 机制通知用户空间的守护进程（如 *udev*、*systemd*），后者据此创建设备节点、加载驱动、设置权限或执行自定义脚本---[电子发烧友网](https://www.elecfans.com/d/5948085.html)。

*uevent* 是实现热插拔（*Hotplug*）和动态设备管理的基础。

- *uevent* 发送及其相关接口
    ```c
    // version: 4.1.15
    // uevent 通过 include/linux/kobject.h 提供了如下 API
    // 在 lib/kobject_uevent.c 中实现
    int kobject_uevent(struct kobject *kobj, enum kobject_action action);
    int kobject_uevent_env(struct kobject *kobj, enum kobject_action action, char *envp[]);
    int add_uevent_var(struct kobject_uevent_env *env, const char *format, ...);
    int kobject_action_type(const char *buf, size_t count, enum kobject_action *type);
    ```

- *uevent* 的类型
    ```c
    // version: 4.1.15
    // lib/kobject_uevent.c
    /* the strings here must match the enum in include/linux/
     * kobject.h 
     */
    static const char *kobject_actions[] = {
        [KOBJ_ADD] = "add",
        [KOBJ_REMOVE] = "remove",
        [KOBJ_CHANGE] = "change",
        [KOBJ_MOVE] = "move",
        [KOBJ_ONLINE] = "online",
        [KOBJ_OFFLINE] = "offline",
    };

    // /include/linux/kobject.h
    /* The actions here must match the index to the string 
     * array in lib/kobject_uevent.c
     * 
     * Do not add new actions here without checking with
     * driver-core maintainers. Action strings are not meant
     * to express subsystem or device specific properties.
     * In most case you want to send a kobject_uevent_env(kobj, 
     * KOBJ_CHANGE, env) with additional event specific
     * variables added to the event environment.
     */
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
    ```

- `kobject_uevent()`
    ```c
    // version: 4.1.15
    // lib/kobject_uevent.c
    /**
     * kobject_uevent - notify userspace by sending an uevent
     * @action: action that is happening
     * @kobj: struct kobj that the action is happening to
     *
     * Returns 0 If kobject_uevent() is completed with success or the
     * corresponding error when it fails.
     */
    int kobject_uevent(struct kobject *kobj, enum kobject_action action) {
        return kobject_uevent_env(kobj, action, NULL);
    }
    EXPORT_SYMBOL_GPL(kobject_uevent);

    /** 
     * kobject_uevent_env - send an uevent with environmental 
     * data.
     * 
     * @action: action that is happening
     * @kobj: struct kobject that the action is happening to
     * @envp_ext: pointer to environmental data
     *
     * Returns 0 if kobject_uevent_env() is completed with success
     * or the corresponding error when if fails.
     */
    int kobject_uevent_env(struct kobject *kobj, enum kobject_action action, char *envp_ext[]) {
        struct kobj_uevent_env *env;
        const char *action_string = kobject_actions[action];
        const char *devpath = NULL;
        const char *subsystem;
        struct kobject *top_kobj;
        struct kset *kset;
        const struct kset_uevent_ops *uevent_ops;
        int i = 0;
        int retval = 0;
    #ifdef CONFIG_NET
        struct uevent_sock *ue_sk;
    #endif
        pr_debug("kobject: '%s' (%p): %s\n", kobject_name(kobj), kobj, __func__);

        /* search the kset we belong to */
        top_kobj = kobj;
        while (!top_kobj->kset && top_kobj->parent)
            top_kobj = top_kobj->parent;
        if (!top_kobj->kset) {
            pr_debug("kobject: '%s' (%p): %s: attempted to send uevent without kset!\n", kobject_name(kobj), kobj, __func__);
            return -EINVAL;
        }

        kset = top_kobj->kset;
        uevent_ops = kset->uevent_ops;

        /* skip the event, if uevent_suppress is set */
        if (kobj->uevent_suppress) {
            pr_debug("kobject: '%s' (%p): %s: uevent_suppress "
                "caused the event to drop!\n", kobject_name(kobj),
                kobj, __func__);
            return 0;
        }
        /* skip the event, if the filter returns zero */
        if (uevent_ops && uevent_ops->filter)
            if (!uevent_ops->filter(kset, kobj)) {
                pr_debug("kobject: '%s' (%p): %s filter function "
                    "caused the event to drop!\n", kobject_name(kobj),
                    kobj, __func__);
                return 0;
            }
        
        /* originating subsystem */
        if (uevent_ops && uevent_ops->name)
            subsystem = uevent_ops->name(kset, kobj);
        else
            subsystem = kobject_name(&kset->kobj);
        if (!subsystem) {
            pr_debug("kobject: '%s' (%p): %s: unset subsystem caused"
            " the event to drop!\n", kobject_name(kobj), kobj, __func__);
            return 0;
        }
        /* environment buffer */
        env = kzalloc(sizeof(struct kobj_uevent_env), GFP_KERNEL);
        if (!env)
            return -ENOMEM;
        /* complete object path */
        devpath = kobject_get_path(kobj, GFP_KERNEL);
        if (!devpath) {
            retval = -ENOENT;
            goto exit;
        }
        /* default keys */
        retval = add_uevent_var(env, "ACTION=%s", action_string);
        if (retval)
            goto exit;
        retval = add_uevent_var(env, "DEVPATH=%s", devpath);
        if (retval)
            goto exit;
        retval = add_uevent_var(env, "SUBSYSTEM=%s", subsystem);
        if (retval)
            goto exit;
        /* keys passed in from the caller */
        if (envp_ext) {
            for (i = 0; envp_ext[i]; i++) {
                retval = add_uevent_var(env, "%s", envp_ext[i]);
                if (retval)
                    goto exit;
            }
        }
        /* let the kset specific function add its stuff */
        if (uevent_ops && uevent_ops->uevent) {
            retval = uevent_ops->uevent(kset, kobj, env);
            if (retval) {
                pr_debug("kobject: '%s' (%p): %s: uevent() returned"
                " %d\n", kobject_name(kobj), kobj, __func__, retval);
                goto exit;
            }
        }

       /*
        * Mark "add" and "remove" events in the object to ensure 
        * proper events to userspace during automatic cleanup. If 
        * the object did send an "add" event, "remove" will automatically
        * generated by the core, if not already done by the caller.
        */
        if (action == KOBJ_ADD)
            kobj->state_add_uevent_sent = 1;
        else if (action == KOBJ_REMOVE)
            kobj->state_remove_uevent_sent = 1;
        
        mutex_lock(&uevent_sock_mutex);
        /* we will send an event, so request a new sequence number */
        retval = add_uevent_var(var, "SEQNUM=%llu", (unsigned long long )++uevent_seqnum);
        if (retval) {
            mutex_unlock(&uevent_sock_mutex);
            goto exit;
        }
    #if defined(CONFIG_NET)
        /* send netlink message */
        list_for_each_entry(ue_sk, &uevent_sock_list, list) {
            struct sock *uevent_sock = ue_sk->sk;
            struct sk_buff *skb;
            size_t len;
            if (!netlink_has_listener(uevent_sock, 1))
                continue;
            /* allocate message with maximum possible size */
            len = strlen(action_string) + strlen(devpath) + 1;
            skb = alloc_skb(len + env->buflen, GFP_KERNEL);
            if (skb) {
                char *scratch;
                /* add header */
                scratch = skb_put(skb, len);
                sprintf(scratch, "%s@%s", action_string, devpath);
                /* copy keys to our continuous event payload buffer */
                for (i = 0; i < env->envp_idx; i++) {
                    len = strlen(env->envp[i]) + 1;
                    scratch = skb_put(skb, len);
                    strcpy(scratch, env->envp[i]);
                }
                NETLINK_CB(skb).dst_group = 1;
                retval = netlink_broadcast_filtered(uevent_sock, 
                    skb, 0, 1, GFP_KERNEL, kobj_bcast_filter, kobj);
                if (retval == -ENOBUFS || retval == -ESRCH)
                    retval = 0;
            } else
                retval = -ENOMEM;
        }
    #endif
        mutex_unlock(&uevent_sock_mutex);
    #ifdef CONFIG_UEVENT_HELPER
        /* call uevent_helper, usually only enabled during early boot */
        if (uevent_helper[0] && !kobj_usermode_filter(kobj)) {
            struct subprocess_info *info;
            retval = add_uevent_var(env, "HOME=/");
            if (retval)
                goto exit;
            retval = add_uevent_var(env, "PATH=/sbin:/bin:/usr/sbin:/usr/bin");
            if (retval)
                goto exit;
            retval = init_uevent_argv(env, subsystem);
            if (retval)
                goto exit;
            retval = -ENOMEM;
            info = call_usermodehelper_setup(env->argv[0], env->argv, env->envp, GFP_KERNEL, NULL, cleanup_uevent_env, env);
            if (info) {
                retval = call_usermodehelper_exec(info, UMH_NO_WAIT);
                env = NULL;     /* freed by cleanup_uevent_env */
            }
        }
    #endif
    exit:
        kfree(devpath);
        kfree(env);
        return retval;
    }
    EXPORT_SYMBOL_GPL(kobject_uevent_env);
    ```
    - 寻找 *kobject* 对应的 *kset*：一直往当前 *kobject* 的父对象方向（包括 *kobject* 自己）寻找，直到第一个设置了 *kset* 字段的 *kobject*，将该 *kset* 作为寻找结果。如果找不到 *kset*，则直接返回错误代码。
    - 利用 *kset* 提供的 *uevent_ops* 进行后续操作。
    - 过滤 *uevent*：如果 *kobject* 已置位了 `uevent_suppress`，或者 *uevent_ops* 提供的 `filter()` 回调过滤掉了该 *uevent*，或者子系统名未设置，会导致 *uevent* 无法发送。
    - 默认提供的环境变量有：`ACTION`, `DEVPATH`, `SUBSYSTEM`，其余的由调用者以及子系统 `uevent_ops->uevent()` 提供，环境变量结构体的定义：
        ```c
        // version: 4.1.15
        // include/linux/kobject.h
        struct kobject_uevent_env {
            char *argv[3];
            char *envp[UEVENT_NUM_ENVP];
            int envp_idx;
            char buf[UEVENT_BUFFER_SIZE];
            int buflen;
        };
        ```
    - 当发送的 *uevent* 事件为 *add* 和 *remove* 时，在 `kobject` 上做相应的标记。
    - `add_uevent_var()`：*add key value string to the environment buffer*.
    - `CONFIG_UEVENT_HELPER` 等配置项：在 *linux* 内核源码目录下的 `.config` 文件中有对应设置（通过查看其他配置文件（`arch/arm/configs/` 目录下），发现很多都将 `CONFIG_UEVENT_HELPER_PATH` 指定为 `/sbin/hotplug`）：
        ```config
        // imx6ull 的 Linux 内核源码配置文件
        CONFIG_NET=y    // imx_alientek_emmc_defconfig 选择了该配置项
        ...
        # Generic Driver Options
        CONFIG_UEVENT_HELPER=y
        CONFIG_UEVENT_HELPER_PATH=""
        ```
    - 通过 *subprocess_info* 来调用 `uevent_helper` 指定的程序。这里路径为空，没有对应的程序（用户空间）又怎么向用户空间发送通知的呢？（见下文）

- *uevent* 将事件上报到用户空间的两种方式（从上述 `kobject_uevent()` 的源码可知）：
  1. 通过异步运行用户空间的应用程序
    ```c
    // version:4.1.15
    // kernel/kmod.c
    /* prepare to call a usermode helper */
    info = call_usermodehelper_setup()
        INIT_WORK(&sub_info->work, __call_usermodehelper);  // 初始化工作队列的 work
        subinfo->path = path;   // 用户空间应用程序路径
        subinfo->argv = argv;   // 上述程序运行参数
        subinfo->envp = envp;   // 上述程序运行环境变量
        ...
    /* Runs a user-space application. The application
     * is started asynchronously if wait is
     * not set, and runs as a child of keventd.
     * (ie. it runs with full root capabilities)
     */
    call_usermodehelper_exec(info, UMH_NO_WAIT);
        queue_work(khelper_wq, &info->work);
    
    // __call_usermodehelper(): This is run by khelper thread
    __call_usermodehelper(struct work_struct *work)
        struct subprocess_info *sub_info = container_of(...)
        ...
        pid_t pid = kernel_thread(____call_usermodehelper, sub_info, SIGCHLD);
    
    // ____call_usermodehelper(): 
    // This is the task which runs the usermode application
    ____call_usermodehelper(void *data)
        struct subprocess_info *sub_info = data;
        ...
        /* Our parent is keventd, which runs 
         * with elevated (偏高的) scheduling priority.
         * Avoid propagating (传播) that into the userspace child.
         */
        ...
        retval = do_execve(getname_kernel(subinfo->path), 
            (const char __user *const __user *)sub_info->argv,
            (const char __user *const __user *)sub_info->envp);
        ...
    // 执行指定的用户空间应用程序 with
    // 指定的参数和环境变量
    ```
  2. 通过 *netlink* 通信机制，用户空间程序监听相关内容

- 总结，`kobject_uevent()` 的大致流程就是：
  - 通过 *kobject* 找到其对应的 *kset*；
  - 判断当前 *uevent* 是否需要跳过（不发送，`kobject->uevent_suppress` 置位、被 `kset->uevent_ops->filter()` 过滤掉、`subsystem`（名字）未设置）；
  - 设置环境变量，包括默认环境变量、调用者提供的环境变量（`kobject_uevent()` 不提供）、`kset->uevent_ops->uevent()` 提供的环境变量；
  - 根据事件类型置位 *kobject* 的相关标志位；
  - 根据配置文件，需要选择配置项 `CONFIG_UEVENT_HELPER`，并结合其中指定 `CONFIG_UEVENT_HELPER_PATH` (用户空间下的 `uevent_helper` 可执行程序的路径)，（subprocess-新建子进程来？）调用可执行程序完成 *uevent* 的响应工作。这里也可以通过 *netlink* 机制实现。

---
来自与 *kimi* 的问答，问：我在阅读内核源码关于 *uevent* 的部分时发现，实际上的设备节点文件创建工作是通过创建一个子进程来运行用户空间程序来完成的。这个用户空间程序通过 `CONFIG_UEVENT_HELPER` 和 `CONFIG_UEVENT_HELPER_PATH` 来指定，我在内核源码中发现一些开发板的配置中指出了 `CONFIG_UEVENT_HELPER_PATH` 为 `/sbin/udev`。但是我现在使用的开发板配置中对 `CONFIG_UEVENT_HELPER_PATH` 的配置为 `“”`，即值为空。但是我的开发板却能正常响应 *uevent* 事件，在 `/dev` 目录下创建设备节点。我当前使用的是 *busybox* 构建的根文件系统，你能帮我解释一下这是为什么吗？

这是一个关于 Linux 内核设备热插拔机制演进的问题。让我解释两种机制的区别：

1. 两种 *uevent* 处理机制
    ```
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                    Linux 内核 uevent 处理机制演进                             │
    ├─────────────────────────────────────────────────────────────────────────────┤
    │                                                                             │
    │  机制 1：CONFIG_UEVENT_HELPER（旧机制，已废弃）                               │
    │  ─────────────────────────────────────────────                              │
    │  内核通过 fork/exec 执行用户空间程序处理每个 uevent                            │
    │  由 CONFIG_UEVENT_HELPER_PATH 指定程序路径（如 /sbin/udev）                   │
    │                                                                             │
    │       内核 uevent                                                      │
    │          ↓                                                              │
    │    ┌─────────────┐                                                      │
    │    │ fork() 子进程 │ ← 每个 uevent 都 fork 一次，性能差                      │
    │    │ exec("/sbin/udev") │                                               │
    │    └─────────────┘                                                      │
    │          ↓                                                              │
    │    用户空间程序处理设备节点创建                                              │
    │                                                                             │
    │  机制 2：netlink socket（现代机制，推荐）                                    │
    │  ─────────────────────────────────────                                    │
    │  内核通过 netlink 广播 uevent，用户空间守护进程监听并处理                      │
    │                                                                             │
    │       内核 uevent                                                      │
    │          ↓                                                              │
    │    ┌─────────────┐                                                      │
    │    │ netlink 广播  │ ← 通过 socket 发送，无 fork 开销                         │
    │    └─────────────┘                                                      │
    │          ↓                                                              │
    │    用户空间守护进程（udevd / mdev -d）监听并处理                               │
    │                                                                             │
    └─────────────────────────────────────────────────────────────────────────────┘
    ```

2. 为什么 `CONFIG_UEVENT_HELPER_PATH=""` 仍能工作
    关键：内核优先使用 *netlink*，*uevent_helper* 是后备。
    ```c
    // kernel/lib/kobject_uevent.c 简化逻辑
    static int uevent_net_broadcast(struct sk_buff *skb, ...)
    {
        // 1. 首先尝试通过 netlink 发送 uevent
        //    用户空间守护进程（udevd / mdev）在监听
        retval = netlink_broadcast(uevent_sock, skb, ...);
        
        // 2. 如果 netlink 没有用户监听，且配置了 uevent_helper
        //    才回退到 fork/exec 方式
        if (retval == -ESRCH && uevent_helper[0]) {
            call_usermodehelper(uevent_helper, ...);
        }
    }
    ```
    流程：
    ```
    内核产生 uevent
        ↓
    尝试 netlink 广播
        ↓
    ├─ 有守护进程监听？──→ 守护进程处理（udevd / mdev -d）
    │                      设备节点创建成功
    │
    └─ 没有监听？        ─→ 检查 uevent_helper[0]
                        ├─ 非空 → fork/exec 执行 helper
                        └─ 空   → 放弃处理（设备节点不会创建）
    ```

3. 你的系统实际工作方式
   - *BusyBox mdev* 的两种运行模式
    | 模式 | 命令 | 机制 | 说明 |
    |------|------|------|------|
    | *daemon* 模式 | `mdev -d` | *netlink* | 监听内核 *netlink socket*，实时响应 *uevent* |
    | 单次扫描模式 | `mdev -s` | 扫描 `/sys` | 启动时一次性创建所有设备节点 |
   - 典型 *init* 脚本配置
    ```bash
    # /etc/init.d/S10mdev 或 /etc/inittab

    # 方式 1：设置热插拔代理（旧机制，配合 uevent_helper）
    echo /sbin/mdev > /proc/sys/kernel/hotplug

    # 方式 2：mdev daemon 模式（现代机制，使用 netlink）
    mdev -d &
    ```
   - 你的系统很可能是：
    ```bash
    # 启动脚本中执行了
    mdev -d &

    # 这会让 mdev 作为守护进程运行，通过 netlink 监听 uevent
    # 因此不需要 uevent_helper 机制
    ```

4. 验证你的系统使用哪种机制
    ```bash
    # 1. 检查是否有 mdev 守护进程在运行
    ps | grep mdev
    # 输出示例：  123 root     mdev -d

    # 2. 检查 netlink socket 监听
    cat /proc/net/netlink | grep -i uevent
    # 或
    ss -x -a | grep -i uevent

    # 3. 检查 hotplug 代理设置
    cat /proc/sys/kernel/hotplug
    # 如果为空，说明没有使用 uevent_helper 机制

    # 4. 检查 mdev 启动方式
    cat /etc/inittab | grep mdev
    # 或
    ls /etc/init.d/ | grep mdev
    ```

5. `CONFIG_UEVENT_HELPER_PATH=""` 的含义
    | 配置值 | 含义 |
    |--------|------|
    | `"/sbin/udev"` | 使用 *uevent_helper* 机制，回退时执行 *udev* |
    | `"/sbin/mdev"` | 使用 *uevent_helper* 机制，回退时执行 *mdev* |
    | `""`（空） | 禁用 *uevent_helper* 回退机制，完全依赖 *netlink* |

    空字符串是故意的设计：强制系统使用 *netlink* + 守护进程模式，避免 *fork/exec* 的性能开销。

6. 机制对比
    | 特性 | *uevent_helper (fork/exec)* | *netlink + daemon* |
    |------|---------------------------|------------------|
    | 性能 | 差（每个 *uevent fork* 一次） | 好（*socket* 通信）|
    | 延迟 | 高 | 低 |
    | 资源占用 | 高 | 低（一个守护进程）|
    | 可靠性 | 低（可能 *fork* 失败）| 高 |
    | 现代系统 | ❌ 废弃 | ✅ 推荐 |
    | 嵌入式系统 | 仍支持（兼容性）| 主流 |

7. 总结
    | 问题 | 答案 |
    |------|------|
    | 为什么 `CONFIG_UEVENT_HELPER_PATH=""` 仍能工作 | 系统使用 *netlink + mdev daemon* 模式，不依赖 *uevent_helper* |
    | *mdev* 如何接收 *uevent* | `mdev -d` 启动守护进程，监听 *netlink socket* |
    | 空配置是错误吗 | 不是，是现代系统的推荐配置 |
    | 如果 `mdev -d` 没启动会怎样 | `/dev` 下不会动态创建设备节点，只有启动时 `mdev -s` 创建的节点 |

    你的系统工作正常，说明 *init* 脚本正确启动了 `mdev -d`，这是现代嵌入式 *Linux* 的标准做法。

### kobj_type
#### 概述
代表一类 *kobject* 的所具有的共同属性（*attr*）和方法（*callback*）。

每一个内嵌 *kobject* 的数据结构，例如 *device* 等，都要实现自己的 *kobj_type*，并定义其中的回调函数。

#### 数据结构描述
```c
// version: 4.1.15
// include/linux/kobject.h
/** 
 * @release: 通过该回调函数，可以将该种类型的 kobject 的数据结构内存空间释放掉。
 * @sysfs_ops: 该种类型的 kobject 的 sysfs 文件系统操作函数集。
 * @default_attrs: 该种类型的 kobject 的 attribute 列表，
 *                 attribute 在 sysfs 中表现为 kobject 目录下的文件，
 *                 在 kobject 添加到内核时，一并注册到 sysfs 中。
 * @child_ns_type: 和文件系统（sysfs）的命名空间有关。
 * @namespace: 和文件系统（sysfs）的命名空间有关。
 */
struct kobj_type {
    void (*release)(struct kobject *kobj);
    const struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;
    const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
    const void *(*namespace)(struct kobject *kobj);
};
```
- 关于 `release` 的一般实现可见 [3]。
- 子系统会自定义 `struct sysfs_ops` 的实现，然后再调用本子系统下的 *attribute*（通过 `struct attribute` *back-casting*） 的 `show()` 或者 `store()` 回调。这在笔记 `Linux/Kernel/FileSystem/sysfs.md` 中也有说明。
  - 在用户空间对 `/sys` 下的文件进行操作，它的调用链大概是：用户空间的读写操作 -> *sysfs_ops* -> *attribute* 的 `show()/store()`。
  - 对属性文件的读写操作也体现了“**驱动/内核提供机制，但不提供策略**”的思想。机制通过最底层的数据结构，如 *kobject、kset、kobj_type、attribute* 等，来实现。驱动的实现需要借助机制，基于这些底层数据结构并找到自身关注的内容构建数据结构，然后进一步地实现驱动自身的业务内容，即策略部分。

### kset
#### 概述
一个特殊的 *kobject*（因此它也会在 *sysfs* 文件系统中以目录的形式出现），它用来集合相似的 *kobject*（这些 *kobject* 可以是相同的 *kobj_type*，也可以是不同的）。

由于 *kobject* 调用的是它所属 *kset* 的 *uevent* 操作函数，所以 *kset* 可以对其行为进行控制。如果 *kobject* 不属于任何 *kset*，则无法发送 *uevent*。

#### 数据结构描述
```c
// include/linux/kobject.h
/**
 * @list: 用于保存该 kset 下所有的 kobject 的链表。the list of all kobjects for this kset.
 * @list_lock: list 自旋锁。a lock for iterating over the kobjects.
 * @kobj: 该 kset 自己的 kobject（kset 是一个特殊的 kobject， 会在 sysfs 中以目录的形式出现）。
 * @uevent_ops: 该 kset 的 uevent 操作函数集，
 *              当任何 kobject 需要上报 uevent 时，都要调用它从属
 *              的 kset 的 uevent_ops：添加环境变量，或者过滤 
 *              event（kset 可以决定哪些 event 可以上报）。
 *              因此，如果一个 kobject 不属于任何 kset 时，是不允许发送 uevent 的。
 *              the set of uevent operations for this kset. These are called whenever
 *              a kobject has something happen to it so that the kset can add new
 *              environment variables, or filter out the uevents if so desired.
 */
 /*
  * struct kset - a set of kobjects of a specific type, belonging to a specific subsystem.
  * A kset defines a group of kobjects. They can be individually different "type" but
  * overall these kobjects all want to be grouped together and operated on in the 
  * same manner. ksets are used to define the attribute callbacks and other common
  * events that happen to a kobject.
  */
struct kset {
    struct list_head list;
    spinlock_t list_lock;
    struct kobject kobj;
    const struct kset_uevent_ops *uevent_ops;
};
```

#### 注册
与 *kobject* 的注册过程大致相同，因为 *kset* 本就是一个特殊的 *kobject*。
```c
// version: 4.1.15
// lib/kobject.c
/**
 * kset_register - initialize and add a kset 
 * @k: kset.
 */
int kset_register(struct kset *k) {
    int err;
    if (!k)
        return -EINVAL;
    kset_init(k);
    err = kobject_add_internal(&k->kobj);
    if (err)
        return err;
    kobject_uevent(&k->obj, KOBJ_ADD);
    return 0;
}

/** 
 * kset_init - initialize a kset for use 
 * @k: kset
 */
void kset_init(struct kset *k) {
    kobject_init_internal(&k->kobj);
    INIT_LIST_HEAD(&k->list);
    spin_lock_init(&k->list_lock);
}
```

- *helper func*
  1. `kset_create()` 
     - 分配 *kset* 所需内存
     - 设置 *kset* 的 *uevent* 操作集
     - 为 *kobject* 设置名字 + 父 kobject + *kobj_type* (`kset_ktype`) + kset (`NULL`) 
    ```c
    // version:4.1.15
    // lib/kobject.c
    /**
     * kset_create - create a struct kset dynamically 
     * @name: the name for the kset
     * @uevent_ops: a struct kset_uevent_ops for the kset
     * @parent_kobj: the parent kobject of this kset, if any.
     * This function creates a kset structure dynamically. This 
     * structure can then be registered with the system and show up
     * in sysfs with a call to kset_register(). When you are finished
     * with this structure, if kset_register() has been called, call
     * kset_unregister() and the structure will be dynamically freed
     * when it is no longer beging used.
     * If the kset was not able to be created, NULL will be returned.
     */
    static struct kset *kset_create(const char *name, const struct kset_uevent_ops *uevent_ops, struct kobject *parent_kobj) {
        struct kset *kset;
        int retval;
        kset = kzalloc(sizeof(*kset), GFP_KERNEL);
        if (!kset)
            return NULL;
        retval = kobject_set_name(&kset->kobj, "%s", name);
        if (retval) {
            kfree(kset);
            return NULL;
        }
        kset->uevent_ops = uevent_ops;
        kset->kobj.parent = parent_kobj;
        /* The kobject of this kset will have a type of kset_ktype 
         * and belong to no kset itself. That way we can properly free
         * it when it is finished being used.
         */
        kset->kobj.ktype = &kset_ktype;
        kset->kobj.kset = NULL;
        return kset;
    }
    ```

  2. `kset_create_and_add()`：`kset_create()` + `kset_register()`
    ```c
    // version: 4.1.15
    // lib/kobject.c
    /**
     * kset_create_add_add - create a struct kset dynamically and 
     * add it to sysfs
     * @name: the name for the kset
     * @uevent_ops: a struct kset_uevent_ops for the kset
     * @parent_kobj: the parent kobject of this kset, if any.
     * This function creates a kset structure dynamically and registers
     * it with sysfs. When you are finished with this structure, call
     * kset_unregister() and the structure will be dynamically freed
     * when it is no longer being used.
     * If the kset was not able to create, NULL will be returned.
     */
    struct kset *kset_create_and_add(const char *name, const struct kset_uevent_ops *uevent_ops, struct kobject *parent_kobj) {
        struct kset *kset;
        int error;
        kset = kset_create(name, uevent_ops, parent_kobj);
        if (!kset)
            return NULL;
        error = kset_register(kset);
        if (error) {
            kfree(kset);
            return NULL;
        }
        return kset;
    }
    EXPORT_SYMBOL_GPL(kset_create_and_add);
    ```

#### 注销
```c
// version: 4.1.15
// lib/kobject.c
/* kset_unregister - remove a kset */
void kset_unregister(struct kset *k) {
    if (!k)
        return;
    kobject_del(&k->kobj);
    kobject_put(&k->kobj);
}
```

#### uevent 操作集
```c
/**
 * @filter: 当任何 kobject 需要上报 uevent 时，它所属的 kset 
 * 可以通过该接口过滤，阻止不希望上报的 uevent，从而达到从整体上
 * 管理的目的。
 * @name: 该接口可以返回 kset 的名称。如果一个 kset 没有合法的
 * 名称，则其下所有的 kobject 将不允许上报 uevent。
 * @uevent: 当任何 kobject 需要上报 uevent 时，它所属的 kset 
 * 可以通过该接口统一为这些 event 添加环境变量。
 * 因为很多时候上报 uevent 时的环境变量都是相同的，因此可以由 
 * kset 统一处理，就不需要让每个 kobject 独自添加了。
 */
struct kset_uevent_ops {
    int (*const filter)(struct kset *kset, struct kobject *kobj);
    const char *(*const name)(struct kset *kset, struct kobject *obj);
    int (*const uevent)(struct kset *kset, struct kobject *obj, struct kobj_uevent_env *env);
}
```

### 内核对象、类型和集合的关系
*kobject*, *kobj_type* 和 *kset* 三者之间的关系，见图 `./image.drawio-kobject`。

1. *kobject* 的核心功能是：保持一个引用计数，当该计数减为 0 时，自动释放 *kobject* （及其嵌入的数据结构）所占用的内存空间。这就决定了 *kobject* 必须是动态分配的（只有这样才能动态释放）（*kobject* 不能被静态声明或者定义在栈上，见 [3]）。
2. *kobject* 的大多数使用场景是内嵌在大型的数据结构中（如 *kset*、*device_driver* 等），因此这些大型数据结构也必须动态分配、动态释放。释放时机与内嵌的 *kobject* 一致。
3. 大型数据结构内存空间的一并释放需要借助 *kobj_type*。*kobj_type* 中的 `release()` 回调函数负责释放 *kobject*（甚至是包含 *kobject* 的数据结构）的内存空间。那么 *kobj_type* 及其内部函数，是由谁实现呢？是由上层数据结构所在的模块！因为只有它，才清楚 *kobject* 内嵌在哪个数据结构中，并通过 *kobject* 指针以及自身的数据结构类型，找到需要释放的上层数据结构的指针，然后释放它。
4. 综上，每一个内嵌 *kobject* 的数据结构，都要实现一个 *kobj_type*，并且定义其中的回调函数。同理，***sysfs* 相关的操作也一样，必须经过 *kobject* 的中转，因为 *sysfs* 看到的是 *kobject*，而真正的文件操作的主体，是内嵌 *kobject* 的上层数据结构**。 

### 内核示例代码
1. `samples/kobject/kobject-example.c`
    ```c
    /* Sample kobject implementation */
    #include <linux/kobject.h>
    #include <linux/string.h>
    #include <linux/sysfs.h>
    #include <linux/module.h>
    #include <linux/init.h>

    /* 
     * This module shows how to create a simple subdirectory in 
     * sysfs called /sys/kernel/kobject-example. In that directory,
     * 3 files are created: "foo", "baz", and "bar". If an integer
     * is written to these files, it can be later read out of it.
     */
    static int foo;
    static int baz;
    static int bar;

    /* 
     * The "foo" file where a static variable is read from and written to.
     */
    static ssize_t foo_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf) {
        return sprintf(buf, "%d\n", foo);
    }

    static ssize_t foo_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count) {
        int ret;
        ret = kstrtoint(buf, 10, &foo);
        if (ret < 0)
            return ret;
        return count;
    }

    /* Sysfs attributes cannot be world-writable. */
    static struct kobj_attribute foo_attribute = __ATTR(foo, 0664, foo_show, foo_store);

    /* 
     * More complex function where we determine which variable is 
     * being accessed by looking at the attribute for the "baz" and 
     * "bar" files.
     */
    static ssize_t b_show(struct kobject *kobj, struct kobj_attribute, char *buf) {...}

    static ssize_t b_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count) {...}

    static struct kobj_attribute baz_attribute = __ATTR(baz, 0664, b_show, b_store);
    static struct kobj_attribute bar_attribute = __ATTR(bar, 0664, b_show, b_store);

    /* 
     * Create a group of attributes so that we can create and 
     * destroy them all at once.
     */
    static struct attribute *attr[] = {
        &foo_attribute.attr,
        &baz_attribute.attr,
        &bar_attribute.attr,
        NULL,   /* need to NULL terminate the list of attributes */
    };

    /* 
     * An unnamed attribute group will put all of the attributes
     * directly in the kobject directory. If we specify a name, a
     * subdirectory will be created for the attributes with the 
     * directory being the name of the attribute group.
     */
    static struct attribute_group attr_group = {
        .attrs = attrs,
    };

    static struct kobject *example_kobj;

    static int __init example_init(void) {
        int retval;
        /**
         * Create a simple kobject with the name of "kobject_example",
         * located under /sys/kernel
         * 
         * As this is a simple directory, no uevent will be sent to
         * userspace. That is why this function should not be used
         * for any type of dynamic kobjects, where the name and number
         * are not known ahead of time.
         */
        example_kobj = kobject_create_and_add("kobject_example", kernel_kobj);
        if (!example_kobj)
            return -ENOMEM;
        
        /* Create the files associated with this kobject */
        retval = sysfs_create_group(example_kobj, &attr_group);
        if (retval)
            kobject_put(example_kobj);
        
        return retval;
    }

    static void __exit example_exit(void) {
        kobject_put(example_kobj);
    }

    module_init(example_init);
    module_exit(example_exit);
    MODULE_LICENSE("GPL v2");
    MODULE_AUTHOR("Greg Kroah-Hartman <greg@kroah.com>");
    ```
    - 通过 `__ATTR()` 的第二个参数 `0664` (8 进制) 指定文件的访问权限信息；
    - 由注释信息可知：
      - 通过将多个 `struct attribute` 组织成 `struct attribute_group` 可以方便地将这些属性一起创建、销毁；
      - *need to NULL terminate the list of attributes*;
      - **如果 `struct attribute_group` 未指定名字，则会将所有的 `struct attribute` 放到 *kobject* （即 `sysfs_creatre_group()` 中的参数）目录下；如果指定了名字，则会新建一个目录，名字就是 *attribute_group* 的名字，然后将所有 *attribute* 放到这个新建目录下**；
      - `kobject_create_and_add()` 函数在 `/sys/` 下创建目录入口后并不会向用户空间发送 *uevent* 事件，所以对于事先并不知道 *name and number* 的 *dynamic kobject* 来说不要使用此函数。
    - `foo_attribute` 的类型为 `struct kobj_attribute`，对 *kobj_attribute* 的读写操作：通过 `kobject_create()` 函数创建的 *kobject* 实例的 `ktype` 指向 `dynamic_kobj_ktype`，因此在 *sysfs* 中对它的操作最终会执行到 `kobj_sysfs_ops` 指定的操作集上。在 `kobj_sysfs_ops` 的回调函数中，会进一步地调用 *kobj_attribute* 的 `show()/store()` 函数。
        ```c
        // include/linux/kobject.h
        struct kobj_attribute {
            struct attribute attr;
            ssize_t (*show)(struct kobject *kobj, struct kobj_attribute *attr, char *buf);
            ssize_t (*store)(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count);
        };

        extern const struct sysfs_ops kobj_sysfs_ops;
        ```
    - `example_kobj` 的父节点为 `kernel_kobj`：
        ```c
        // kernel/ksysfs.c
        struct kobject *kernel_kobj;
        EXPORT_SYMBOL_GPL(kernel_kobj);

        // 关于 kernel_attr_group 的定义，略

        static int __init ksysfs_init(void) {
            int err;
            kernel_kobj = kobject_create_add_add("kernel", NULL);
            if (!kernel_kobj) {
                error = -ENOMEM;
                goto exit;
            }
            error = sysfs_create_group(kernel_kobj, &kernel_attr_group);
            if (error)
                goto kset_exit;
            ...
        group_exit:
            sysfs_remove_group(kernel_kobj, &kernel_attr_group);
        kset_exit:
            kobject_put(kernel_kobj);
        exit:
            return error;
        }

        core_initcall(ksysfs_init);
        ```
        - `kernel_kobj` 入口位于 `/sys` 的顶层目录，名为 "kernel"，因此完整路径是：`/sys/kernel`，这与源码中的注释对应上。

## 总线设备驱动
### 总线
#### 概述
总线可以视为一个子系统：
- 分离驱动、设备，实现两者的动态匹配。
- 为总线上的驱动、设备提供一致的属性、回调函数、操作函数。
- 依靠总线实现子系统的公共逻辑业务，提供统一的 *API* 给上层使用、规定底层实现。
- 一个总线也可以 *plug into* 其他的总线。

关于总线、设备、驱动以及各个 *private* 数据结构的关系见图：`Linux/Driver/image.drawio:bus-device-driver overview`

#### 数据结构描述
```c
// version: 4.1.15
// include/linux/device.h
/**
 * struct bus_type: The bus type of the device 
 * @name: The name of the bus.
 * @dev_name: Used for subsystems to enumerate devices like ("foo%u", dev->id)
 * @dev_root: Default device to use as the parent
 * @dev_attrs: Default attributes of the devices on the bus
 * @bus_groups: Default attributes of the bus.
 * @dev_groups: Default attributes of the devices on the bus
 * @drv_groups: Default attributes of the device drivers on the bus
 * @match: Called, perhaps multiple times, whenever a new device or driver
 *      is added for this bus. It should return a nonzero value if the given
 *      device can be handled by the given driver.
 * @uevent: Called when a device is added, removed, or a few other things
 *      that generate uevents to add the environment variables.
 * @probe: Called when a new device or driver add to this bus, and callback
 *      the specific driver's probe to initial the matched device
 * @remove: Called when a device removed from this bus.
 * @shutdown: Called at shut-down (关闭，关机) time to quiesce the device
 * 
 * @online: Called to put the device back online (after offlining it).
 * @offline: Called to put the device offline for hot-removal. May fail.
 *
 * @suspend: Called when a device on this bus wants to go to sleep mode.
 * @resume: Called to bring a device on this bus out of sleep mode.
 * @pm: Power management operations of this bus, callback the specific device
 *      driver's pm-ops.
 * @iommu_ops: IOMMU specific operations for this bus, used to attach IOMMU
 *      driver implementations to a bus and allow the driver to do bus-specific
 *      setup.
 * @p: The private data of the driver core, only the driver core can touch this.
 * @lock_key: Lock class key for use by the lock validator
 *
 * A bus is channel between the processor and one or more devices. For the
 * purpose of the device model, all devices are connected via a bus, even if
 * it is an internal, virtual, "platform" bus. Buses can plug into each other.
 * A USB controller is usually a PCI device, for example. The device model
 * represents the actual connections between buses and the devices they control.
 * A bus is represents by the bus_type structure. It contains the name, the 
 * default attributes, the bus's methods, PM operations, and the driver core's
 * private data.
 */
struct bus_type {
    const char *name;
    const char *dev_name;
    struct device *dev_root;
    struct device_attribute *dev_attrs;  /* use dev_groups instead */
    const struct attribute_group **bus_groups;
    const struct attribute_group **dev_groups;
    const struct attribute_group **drv_groups;

    int (*match)(struct device *dev, struct device_driver *drv);
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);
    int (*shutdown)(struct device *dev);

    int (*online)(struct device *dev);
    int (*offline)(struct device *dev);

    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    const struct dev_pm_ops *pm;

    const struct iommu_ops *iommu_ops;

    struct subsys_private *p;
    struct lock_class_key lock_key;
};
```
1. 各个回调函数总结、对比：
    | 回调 | 触发时机 | 作用 | 典型场景 |
    | ----- | ----- | ----- | ----- |
    | `match()` | 设备或驱动添加到总线时 | 判断设备与驱动是否匹配 | 设备或驱动注册 |
    | `uevent()` | `kobject_uevent()` 调用时 | 向用户空间发送 *uevent* 时添加环境变量 | 设备添加或移除 |
    | `probe()` | 设备与驱动匹配后 | 会进一步调用驱动的 `probe()` 以初始化设备 | 设备或驱动注册 |
    | `remove()` | 设备从总线解绑或注销时 | 与 `probe()` 相反，释放设备占用的总线级资源，调用驱动 `remove()` | 模块卸载、热拔除、驱动重绑 |
    | `shutdown()` | 系统关机/重启前 | 让设备进入安全状态，如刷写缓存、停止 *DMA* | *poweroff*/*reboot* 时 |
    | `suspend()` | 系统进入睡眠状态 | 保存设备上下文，停止 *I/O*，切低功耗模式 | 笔记本合盖、`echo mem > /sys/power/state` |
    | `resume()` | 系统从睡眠唤醒 | 恢复时钟、寄存器状态，重新使能设备 | 唤醒事件后 |
    | `online()` | 设备被显式标记为可用 | 将离线设备重新纳入系统管理，恢复其功能 | *CPU* 热插上线、*PCI* 设备重新启用、内存热添加 |
    | `offline()` | 设备被显式标记为不可用 | 优雅停止设备，将其从系统中隔离，为移除做准备 | *CPU* 热拔除前、*PCI* 设备 `remove()` 前的隔离、故障内存下线 |

    `online()/offline()` 和 `suspend()/resume()` 的区别：
    | 维度 | `online()/offline()` | `suspend()/resume()` |
    | ----- | ----- | ----- |
    | 管理对象 | 设备的可用性/存在性 | 设备的电源状态 |
    | 设备是否仍在工作 | `offline()` 后完全停止工作，对上层不可见或不可访问 | `suspend()` 后逻辑上仍属于系统，只是硬件在"睡觉" |
    | 资源占用 | 通常会释放 *IRQ*、*DMA*、总线带宽等资源 | 保留资源分配（如 *IRQ* 号、*DMA* 通道），仅硬件断电/门控 |
    | 触发原因 | 用户显式操作、热插拔事件、故障隔离 | 电源管理策略、系统级睡眠 |
    | 恢复复杂度 | 可能需要重新 `probe()` 或重新初始化部分结构 | 通常只需恢复寄存器上下文和时钟 |
    | 内核路径 | `device_online()/device_offline()` | `device_suspend()/device_resume()` |

2. 总线通过 `struct subsys_private` 与设备、驱动建立联系，从而管理总线上的设备和驱动。从如下注释中可知，该数据结构不仅用于总线与设备、驱动之间，也用于 `struct class` 与其设备之间。
    ```c
    // version: 4.1.15
    // 
    /**
    * struct subsys_private - struture to hold the private to the
    *      driver core portions of the bus_type/class structure.
    * @subsys: the struct kset that defines this subsystem.
    * @devices_kset: the subsystem's 'devices' directory.
    * @interface: list of subsystem interface associated
    * @mutex: protect the devices, and interfaces lists.
    *
    * @drivers_kset: the list drivers associated 
    * @klist_devices: the klist to iterate over the @devices_kset
    * @klist_drivers: the klist to iterate over the @drivers_kset
    * @bus_notifier: the bus notifier list for anything that cares about
    *  things on this bus.
    * @bus: pointer back to the struct bus_type that this structure is
    *   associated with.
    * @glue_dirs: "glue" directory to put in-between the parent device
    *  to avoid namespace conflicts.
    * @class: pointer back to the struct class that this structure is
    *  associated with.
    *
    * This structure is the one that is the actual kobject allowing 
    * struct bus_type/class to be statically allocated safely. Nothing
    * outside of the driver core should ever touch these fields.
    */
    struct subsys_private {
        struct kset subsys;
        struct kset *devices_kset;
        struct list_head interfaces;
        struct mutex mutex;

        struct kset *drivers_kset;
        struct klist klist_devices;
        struct klist klist_drivers;
        struct blocking_notifier_head bus_notifier;
        unsigned int drivers_autoprobe:1;
        struct bus_type *bus;
        struct kset glue_dirs;
        struct class *class;
    };

    #define to_subsys_private(obj) container_of(obj, struct subsys_private, subsys.kobj);
    ```
   - 子系统（总线/类）通过 `subsys` 字段（类型为 `struct kset`）建立该子系统在 *sysfs* 中的入口，通常表示为 `sys/bus/xxx`（以总线为例）。
   - `devices_kset` 和 `drivers_kset` 两个字段分别建立该子系统中设备和驱动在 *sysfs* 中的入口，通常表示为：`sys/bus/xxx/devices` 和 `sys/bus/xxx/drivers`（以总线为例）。
   - 子系统对设备和驱动的管理实际上是通过 `klist_devices` 和 `klist_drivers` 两个字段实现的，它们分别表示在该子系统上的设备链表和驱动链表。
   - 如果子系统表示总线，则可以通过 `drivers_autoprobe` 字段控制在设备添加到总线上时是否自动与总线上的驱动进行匹配。一般而言，总线实例总是将该字段设置为有效。
   - `bus_notifier` 字段用于实现总线的 *notifier* 机制，当总线上某部分需要关注总线上的设备变化时，可以通过 `bus_register_notifier(struct bus_type *, struct notifier_block *)` 接口注册 *notifier*。例如：`/drivers/i2c/i2c-dev.c-i2c_dev_init()`。
   - `interfaces`, *list of subsystem interface associated*，该接口链表的元素类型为：
       ```c
       // version: 4.1.15
       // include/linux/device.h
       /** 
        * struct subsys_interface - interfaces to device functions 
        * @name: name of the device function
        * @subsys: subsystem of the devices to attach to
        * @node: the list of functions registered at the subsystem
        * @add_dev: device hookup to device function handler
        * @remove_dev: device hookup to device function handler
        *
        * Simple interfaces attached to a subsystem. Multiple interfaces can
        * attach to subsystem and its devices. Unlike drivers, they do not
        * exclusively claim or control devices. Interfaces usually represent
        * a specific functionality of a subsystem/class of devices.
        */
       struct subsys_interface {
           const char *name;
           struct bus_type *subsys;
           struct list_head *node;
           int (*add_dev)(struct device *dev, struct subsys_interface *sif);
           int (*remove)(struct device *dev, struct subsys_interface *sif);
       };
       ```
   - 从注释得到：
     - 根据 `@subsys: the struct kset that defines this subsystem.` 推测：子系统可能是一个针对 *kset* 的概念，且 `struct subsys_private` 才是 *subsystem* 的核心，因此在内核中 *bus* 和 *class* 都可以作为一个子系统。
     - 可能会存在一个 “*glue*” 的目录，用于避免命名空间的冲突。

#### 总线子系统的实现
1. 每个子系统会定义一个 `struct kobj_type`，总线子系统定义了该类型的变量：`bus_ktype`。在通过 `bus_register()` 接口注册总线实例时，会将其赋值给 `bus->p->subsys.kobj.ktype`。
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    static struct kobj_type bus_ktype = {
        .sysfs_ops = &bus_sysfs_ops,
        .release   = bus_release,
    };

    static const struct sysfs_ops bus_sysfs_ops = {
        .show = bus_attr_show,
        .store = bus_attr_store,
    }

    static ssize_t bus_attr_show(struct kobject *kobj, struct attribute *attr, char *buf) {
        struct bus_attribute *bus_attr = to_bus_attr(attr);
        struct subsys_private *subsys_priv = to_subsys_private(kobj);
        ssize_t ret = 0;
        if (bus_attr->show)
            ret = bus_attr->show(subsys_priv->bus, buf);
        return ret;
    }

    static ssize_t bus_attr_store(struct kobject *kobj, struct attribute *attr, const char *buf, size_t count) {
        struct bus_attribute *bus_attr = to_bus_attr(attr);
        struct subsys_private *subsys_priv = to_subsys_private(kboj);
        ssize_t ret = 0;
        if (bus_attr->store)
            ret = bus_attr->store(subsys_priv->bus, buf, count);
        return ret;
    }
    
    // include/linux/device.h
    struct bus_attribute {
        struct attribute attr;
        ssize_t (*show)(struct bus_type *bus, char *buf);
        ssize_t (*store)(struct busy_type *bus, const char *buf, size_t count);
    };

    // 几个 bus_attr 属性读写函数
    // drivers/base/bus.c
    /* func - 将设备与总线上的驱动进行匹配
     * buf - 设备名字 
     * return: 匹配成功时返回 count
     */
    static ssize_t store_drivers_probe(struct bus_type *bus, const char *buf, size_t count);
    /* func - 获取总线是否进行驱动的自动 probe
     * buf - bus->p->drivers_autoprobe 的值被复制到 buf 中
     * return: 返回复制到 buf 中的数据长度
     */
    static ssize_t show_drivers_autoprobe(struct bus_type *bus, char *buf);
    /* func - 控制总线是否进行驱动的自动 probe 
     * buf - 写入值，根据第一个字节是否为 '0' 判断是否自动 probe
     * return: 返回 count
     */
    static ssize_t store_drivers_autoprobe(struct bus_type *bus, const char *buf, size_t count);

    static void bus_release(struct kobject *kobj) {
        struct subsys_private *priv = container_of(kobj, typeof(*priv), subsys.kobj);
        struct bus_type *bus = priv->bus;
        kree(priv);
        bus->p = NULL;
    }
    ```
    - `bus_release()` 回调只将 `bus->p` 对应（所指向）的内存释放掉。

2. 在 `drivers/base/bus.c` 中还定义了两个 `struct kset *` 类型的变量：`bus_kset` 和 `system_kset`。它们在 `buses_init()` 完成初始化：
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    
    /* /sys/devices/system */
    static struct kset *system_kset;

    static struct kset *bus_kset;

    int __init buses_init(void) {
        bus_kset = kset_create_and_add("bus", &bus_uevent_ops, NULL);
        if (!bus_kset)
            return -ENOMEM;
        system_kset = kset_create_and_add("system", NULL, &devices_kset->kobj);
        if (!system_kset)
            return -ENOMEM;
        return 0;
    }

    static const struct kset_uevent_ops bus_uevent_ops = {
        .filter = bus_uevent_filter,
    };

    static int bus_uevent_filter(struct kset *kset, struct kobject *kobj) {
        struct kobj_type *ktype = get_ktype(kobj);
        if (ktype == &bus_ktype)
            return 1;
        return 0;
    }
    ```
    - 入口函数 `buses_init()` 在内核的启动函数 `driver_init(), /drivers/base/init.c` 中被调用。
    - `bus_kset` 对应的入口目录为 `sys/bus`。在通过 `bus_register()` 接口注册总线实例时，会将其赋值给 `bus->p->subsys.kobj.kset`。
      - `bus_uevent_filter()` 会将非 `bus_ktype` 的对象过滤掉。
    - `system_kset` 对应的入口目录为 `sys/devices/system`，它的父目录 `sys/devices`（对应 `devices_kset`） 在 `devices_init(), drivers/base/core.c` 中被创建。`devices_init()` 同样在 `driver_init()` 中被调用，且位于 `buses_init()` 之前。调用 `subsys_system_register()` 注册的总线会使用到 `system_kset`，此类总线的入口目录为 `/sys/devices/system/<name>`。

#### 总线注册
```c
// version: 4.1.15
// drivers/base/bus.c
/**
    * bus_register - register a driver-core subsystem 
    * @bus: bus to register
    *
    * Once we have that, we register the bus with the kobject
    * infrastructure, then register the children subsystems it has:
    * the devices and drivers that belong to the subsystem.
    */
int bus_register(struct bus_type *bus) {
    int retval;
    struct subsys_private *priv;
    struct lock_class_key *key = &bus->lock_key;

    priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;
    
    priv->bus = bus;
    bus->p = priv;
    
    BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);

    retval = kobject_set_name(&priv->subsys.kobj, "%s", bus->name);
    if (retval)
        goto out;
    
    priv->subsys.kobj.kset = bus_kset;
    priv->subsys.kobj.ktype = &bus_ktype;
    priv->drivers_autoprobe = 1;

    retval = kset_register(&priv->subsys);
    if (retval)
        goto out;
    
    retval = bus_create_file(bus, &bus_attr_uevent);
    if (retval)
        goto bus_uevent_fail;
    
    priv->devices_kset = kset_create_and_add("devices", NULL, &priv->subsys.kobj);
    if (!priv->devices_kset) {
        retval = -ENOMEM;
        goto bus_devices_fail;
    }
    priv->drivers_kset = kset_create_and_add("drivers", NULL, &priv->subsys.kobj);
    if (!priv->drivers_kset) {
        retval = -ENOMEM;
        goto bus_drivers_fail;
    }

    INIT_LIST_HEAD(&priv->interfaces);
    __mutex_init(&priv->mutex, "subsys mutex", key);
    klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
    klist_init(&priv->klist_drivers, NULL, NULL);

    retval = add_probe_files(bus);
    if (retval)
        goto bus_probe_files_fail;
    
    retval = bus_add_groups(bus, bus->bus_groups);
    if (retval)
        goto bus_groups_fail;
    
    pr_debug("bus: '%s': register\n", bus->name);
    return 0;
bus_groups_fail:
    remove_probe_files(bus);
bus_probe_files_fail:
    kset_unregister(bus->p->drivers_kset);
bus_drivers_fail:
    kset_unregister(bus->p->devices_kset);
bus_devices_fail:
    bus_remove_file(bus, &bus_attr_uevent);
bus_uevent_fail:
    kset_unregister(&bus->p->subsys);
out:
    kfree(bus->p);
    bus->p = NULL;
    return retval;
}

static void klist_devices_get(struct klist_node *n) {
    struct device_private *dev_prv = to_dev_private_bus(n);
    struct device *dev = dev_prv->device;
    get_device(dev);
}

static void klist_devices_put(struct klist_node *n) {
    struct device_private *dev_prv = to_dev_private_bus(n);
    struct device *dev = dev_prv->device;
    put_device(dev);
}
```
- `bus_register()` 的实现上有点类似于 `class_register()`（可能两者的地位在内核上是类似的？）。
- 创建入口目录，名为 `bus->name`（`/sys/bus/<name>`），并在此目录下创建 *devices* 和 *drivers* 两个子目录，它们分别对应到 `priv->subsys.kobj`、`priv->devices_kset` 和 `priv->drivers_kset`，从而向用户空间暴露该总线上的设备和驱动。
- 创建总线目录下的文件：
  - *uevent* 文件
  - *drivers_probe* 和 *drivers_autoprobe* 文件
  - `bus->bus_groups` 对应的属性集
- `bus_create_file()` 和 `bus_remove_file()`
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    int bus_create_file(struct bus_type *bus, struct bus_attribute *attr) {
        int error;
        if (bus_get(bus)) {
            error = sysfs_create_file(&bus->p->subsys.kobj, &attr->attr);
            bus_put(bus);
        } else
            error = -EINVAL;
        return error;
    }
    EXPORT_SYMBOL_GPL(bus_create_file);

    void bus_remove_file(struct bus_type *bus, struct bus_attribute *attr) {
        if (bus_get(bus)) {
            sysfs_remove_file(bus->p->subsys.kobj, &attr->attr);
            bus_put(bus);
        }
    }
    EXPORT_SYMBOL_GPL(bus_remove_file);

    static ssize_t bus_uevent_store(struct bus_type *bus, const char *buf, size_t count) {
        enum kobject_action action;
        if (kobject_action_type(buf, count, &action) == 0)
            kobject_uevent(&bus->p->subsys.kobj, action);
        return count;
    }
    static BUS_ATTR(uevent, S_IWUSR, NULL, bus_uevent_store);
    ```
    - 往 *bus* 所在目录 *uevent* 文件写入内容，会产生对应的 *uevent* 事件。
- `add_probe_files()` 和 `remove_probe_files()`
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    static int add_probe_files(struct bus_type *bus) {
        int retval;
        retval = bus_create_file(bus, &bus_attr_drivers_probe);
        if (retval)
            goto out;
        retval = bus_create_file(bus, &bus_attr_drivers_autoprobe);
        if (retval)
            bus_remove_file(bus, &bus_attr_drivers_probe);
    out:
        return retval;
    }
    
    static void remove_probe_files(struct bus_type *bus) {
        bus_remove_file(bus, &bus_attr_drivers_autoprobe);
        bus_remove_file(bus, &bus_attr_drivers_probe);
    }

    static ssize_t store_drivers_probe(struct bus_type *bus, const char *buf, size_t count) {
        struct device *dev;
        int err = -EINVAL;

        dev = bus_find_device_by_name(bus, NULL, buf);
        if (!dev)
            return -ENODEV;
        if (bus_rescan_devices_helper(dev, NULL) == 0)
            err = count;
        put_device(dev);
        return err;
    }

    static ssize_t show_drivers_autoprobe(struct bus_type *bus, char *buf) {
        return sprintf(buf, "%d\n", bus->p->drivers_autoprobe);
    }

    static ssize_t store_drivers_autoprobe(struct bus_type *bus, const char *buf, size_t count) {
        if (buf[0] == '0')
            bus->p->drivers_autoprobe = 0;
        else
            bus->p->drivers_autoprobe = 1;
        return count;
    }

    static BUS_ATTR(drivers_probe, S_IWUSR, NULL, store_drivers_probe);
    static BUS_ATTR(drivers_autoprobe, S_IWUSR | S_IRUGO, show_drivers_autoprobe, store_drivers_autoprobe);
    ```
    - 往 *bus* 所在目录的 *drivers_probe* 中写入此总线上的设备名字，如果此设备并未匹配驱动，则可以让此设备重新进行匹配。
    - 从 *bus* 所在目录中读 *drivers_autoprobe* 文件可以获取此总线是否自动匹配驱动的信息，返回：0-不自动匹配、1-自动匹配；写此文件，写入 0 将关闭总线的自动匹配功能，写入其他则开启。
- `bus_add_groups()` 和 `bus_remove_groups()`
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    static int bus_add_groups(struct bus_type *bus, const struct attribute_group **groups) {
        return sysfs_create_groups(&bus->p->subsys.kobj, groups);
    }

    static void bus_remove_groups(struct bus_type *bus, const struct attribute_group **groups) {
        sysfs_remove_groups(&bus->p->subsys.kobj, groups);
    }
    ```

#### 总线注销
```c
// version: 4.1.15
// drivers/base/bus.c
/**
 * bus_unregister - remove a bus from the system 
 * @bus: bus.
 *
 * Unregister the child subsystems and the bus itself.
 * Finally, we call bus_put() to release the refcount.
 */
void bus_unregister(struct bus_type *bus) {
    pr_bebug("bus: '%s': unregistering\n", bus->name);
    if (bus->dev_root)
        device_unregister(bus->dev_root);
    bus_remove_groups(bus, bus->bus_groups);
    remove_probe_files(bus);
    kset_unregister(bus->p->drivers_kset);
    kset_unregister(bus->p->devices_kset);
    bus_remove_file(bus, &bus_attr_uevent);
    kset_unregister(&bus->p->subsys);
}
```
- 注销总线上的公用设备父对象（`bus->dev_root`）
- 移除总线的属性集（目录及）文件
- 移除名为 *probe* 的文件
- 注销总线上所有的驱动、设备
- 移除名为 *uevent* 的文件
- 注销总线子系统（`bus->p->subsys`）

#### add 机制
1. 设备：`bus_add_device(): /drivers/base/bus.c`
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    /** 
     * bus_add_device - add device to bus 
     * @dev: device being add
     *
     * - Add devcie's bus attributes.
     * - Create links to device's bus.
     * - Add the device to its bus's list of devices.
     */
    int bus_add_device(struct device *dev) {
        struct bus_type *bus = bus_get(dev->bus);
        int error = 0;
        if (bus) {
            pr_debug("bus: '%s': add device %s\n", bus->name, dev_name(dev));
            error = device_add_attrs(bus, dev);
            if (error)
                goto out_put;
            error = device_add_groups(dev, bus->dev_groups);
            if (error)
                goto out_id;
            error = sysfs_create_link(&bus->p->devices_kset->kobj,
                &dev->kobj, dev_name(dev));
            if (error)
                goto out_groups;
            error = sysfs_create_link(&dev->kobj, 
                &dev->bus->p->subsys.kobj, "subsystem");
            if (error)
                goto out_subsys;
            klist_add_tail(&dev->p->knode_bus, &bus->p->klist_devies);
        }
        return 0;
    out_subsys:
        sysfs_remove_link(&bus->p->devices_kset->kobj, dev_name(dev));
    out_groups:
        device_remove_groups(dev, bus->dev_groups);
    out_id:
        device_remove_attrs(bus, dev);
    out_put:
        bus_put(dev->bus);
        return error;
    }
    ```
    - 调用链 + 功能：
    ```c
    bus_add_device(device)
        1. 创建属性文件：
            1.1 device_add_attrs(bus, dev): 创建 bus->dev_attrs 中的属性文件
            1.2 device_add_groups(dev, bus->dev_groups): 创建 bus->dev_groups 提供的属性集文件
        2. 创建链接：
            2.1 在总线所在目录的 devices 子目录下创建与设备同名的链接文件，指向设备所在目录
            2.2 在设备所在目录创建名为 subsystem 的链接，指向总线所在目录
        3. 将设备挂到总线管理的设备链表上
    ```
2. 驱动：`bus_add_driver(): /drivers/base/bus.c`
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    /** 
     * bus_add_driver - Add a driver to the bus. 
     * @drv: driver
     */
    int bus_add_driver(struct device_driver *drv) {
        struct bus_type *bus;
        struct driver_private *priv;
        int error = 0;
        bus = bus_get(drv->bus);
        if (!bus)
            return -EINVAL;
        pr_debug("bus: '%s': add driver %s\n", bus->name, drv->name);
        priv = kzalloc(sizeof(*priv), GFP_KERNEL);
        if (!priv) {
            error = -ENOMEM;
            goto out_put_bus;
        }
        klist_init(&priv->klist_devices, NULL, NULL);
        priv->driver = drv;
        drv->p = priv;
        priv->kobj.kset = bus->p->drivers_kset;
        error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL, "%s", drv->name);
        if (error)
            goto out_unregister;
        klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
        if (drv->bus->p->drivers_autoprobe) {
            error = driver_attach(drv);
            if (error)
                goto out_unregister;
        }
        module_add_driver(drv->owner, drv);
        error = driver_create_file(drv, &driver_attr_uevent);
        if (error) {
            printk(KERN_ERR "%s: uevent attr (%s) failed\n", __func__, drv->name);
        }
        error = driver_add_groups(drv, &bus->drv_groups);
        if (error) {
            /* How the hell do we get out of this pickle? Give up */
            printk(KERN_ERR "%s: driver_create_groups(%s) failed\n", __func__, drv->name);
        }
        if (!drv->suppress_bind_attrs) {
            error = add_bind_files(drv);
            if (error) {
                /* Ditto */
                printk(KERN_ERR "%s: add_bind_files(%s) failed\n", __func__, drv->name);
            }
        }
        return 0;
    out_unregister:
        kobject_put(&priv->kobj);
        kfree(drv->p);
        drv->p = NULL;
    out_put_bus:
        bus_put(bus);
        return error;
    }

    static struct kobj_type driver_ktype = {
        .sysfs_ops = &driver_sysfs_ops,     // 对应 struct driver_attribute
        .release   = driver_release,
    };

    static void driver_release(struct kobject *kobj) {
        struct driver_private *drv_priv = to_driver(kobj);
        pr_debug("driver: '%s': '%s'\n", kobject_name(kobj), __func__);
        kfree(drv_priv);
    }

    // drivers/base/module.c
    void module_add_driver(struct module *mod, struct device_driver *drv) {
        char *driver_name;
        int no_warn;
        struct module_kobject *mk = NULL;
        if (!drv)
            return;
        if (mod)
            mk = &mod->mkobj;
        else if (drv->mod_name) {
            struct kobject *mkobj;
            /* Lookup built-in module entry in /sys/modules */
            mkobj = kset_find_obj(module_kset, drv->mod_name);
            if (mkobj) {
                mk = container_of(mkobj, struct module_kobject, kobj);
                /* remember out module structure */
                drv->p->mkobj = mk;
                /* kset_find_obj took a reference */
                kobject_put(mkobj);
            }
        }
        if (!mk)
            return;
        /* Don't check return codes; these calls are idempotent */
        no_warn = sysfs_create_link(&drv->p->kobj, &mk->kobj, "module");
        driver_name = make_driver_name(drv);
        if (driver_name) {
            module_create_drivers_dir(mk);
            no_warn = sysfs_create_link(mk->drivers_dir, &drv->p->kobj, driver_name);
            kfree(driver_name);
        }
    }

    static void module_create_drivers_dir(struct module_kobject *mk) {
        if (!mk || mk->drivers_dir)
            return;
        mk->drivers_dir = kobject_create_and_add("drivers", &mk->kobj);
    }
    ```
    - 调用链 + 功能：
    ```c
    bus_add_driver(driver)
        1. 分配 drv->p 的内存、初始化 drv->p->klist_devices、
           设置 drv->p->kobj.set = bus->p->drivers_kset（在
           /sys/bus/xxx/drivers 目录下能找到该 drv 的目录）
        2. 注册 kobject: drv->p->kobj，其对应的 kobj_type 为 driver_ktype
        3. 将 drv 挂到总线上
        4. 匹配与驱动兼容的设备，触发 match 机制：driver_attach(drv)，
           见 match 机制解析
        5. 将 drv 添加到 module 上：module_add_driver()：
            5.1 在驱动目录下，创建名为 module 的链接，指向驱动所属的模块
            5.2 在驱动所属模块的 drivers 目录下创建链接，指向此驱动
        6. 创建属性文件：创建于 drv->p->kobj 目录下，该目录又位于 drv->p->kobj.kset 目录下
            6.1 uevent 文件
            6.2 总线提供的驱动属性集：bus->drv_groups
            6.3 （如果）bind 和 unbind 文件
    ```

#### match 机制
1. 设备：`device_attach(): /drivers/base/dd.c`
    ```c
    // version: 4.1.15
    // drivers/base/dd.c
    /**
     * device_attach - try to attach device to a driver 
     * @dev: device
     *
     * Walk the list of drivers that the bus has and call
     * driver_probe_device() for each pair. If a compatible
     * pair is found, break out and return.
     *
     * Return 1 if the device was bound to a driver;
     * 0 if no matching driver was found;
     * -ENODEV if the device is not registered.
     *
     * When called for a USB interface, @dev->parent lock must be held.
     */
    int device_attach(struct device *dev) {
        int ret = 0;
        device_lock(dev);
        if (dev->driver) {
            if (klist_node_attached(&dev->p->knode_driver)) {
                ret = 1;
                goto out_unlock;
            }
            ret = device_bind_driver(dev);
            if (ret == 0)
                ret = 1;
            else {
                dev->driver = NULL;
                ret = 0;
            }
        } else {
            ret = bus_for_each_drv(dev->bus, NULL, dev, __device_attach);
            pm_request_idle(dev);
        }
    out_unlock:
        device_unlock(dev);
        return ret;
    }

    /** 
     * device_bind_driver - bind a driver to one device. 
     * @dev: device.
     * 
     * Allow manual attachment of a driver to a device.
     * Caller must have already set @dev->driver.
     *
     * Note that this does not modify the bus reference count
     * nor take the bus's rwsem. Please verify those are accounted
     * for before calling this. (It is ok to call with no other
     * effort from a driver's probe() method.)
     * 
     * This function must be called with the device lock held.
     */
    int device_bind_driver(struct device *dev) {
        int ret;
        ret = driver_sysfs_add(dev);
        if (!ret)
            driver_bound(dev);
        return ret;
    }
    EXPORT_SYMBOL_GPL(device_bind_driver);

    static int __device_attach(struct device_driver *drv, void *data) {
        struct device *dev = data;
        if (!driver_match_device(drv, dev))
            return 0;
        return driver_probe_device(drv, dev);
    }

    // drivers/base/bus.c
    /** 
     * bus_for_each_drv - driver iterator
     * @bus: bus we're dealing with
     * @start: driver to start iterating on
     * @data: data to pass to the callback.
     * @fn: function to call for each driver.
     * 
     * This is nearly identical to the device iterator above 
     * We iterate over each driver that belongs to @bus, and call
     * @fn for each. If @fn returns anything but 0, we break out
     * and return it. If @start is not NULL, we use it as the head
     * of the list.
     *
     * NOTE: we don't return the driver that returns a non-zero
     * value, nor do we leave the reference count incremented for
     * that driver. If the caller needs to know that info, it must
     * set it in the callback. It must also be sure to increment the
     * refcount so it doesn't disappear before returning to the caller.
     */
    int bus_for_each_drv(struct bus_type *bus, struct device_driver *start, 
        void *data, int (*fn)(struct device_driver *, void *)) {
        struct klist_iter i;
        struct device_driver *drv;
        int error = 0;
        if (!bus)
            return -EINVAL;
        klist_iter_init_node(&bus->p->klist_drivers, &i, start ?
            &start->p->knode_bus : NULL);
        while ((drv = next_driver(&i)) && !error)
            error = fn(drv, data);
        klist_iter_exit(&i);
        return error;
    }
    EXPORT_SYMBOL_GPL(bus_for_each_drv);
    ```
    - 调用链 + 功能：
    ```c
    device_attach(device)
        1.1 已有匹配的驱动：device->driver 不为空
            2.1 如果设备已经挂在驱动的设备链表上，则直接返回已 bind
            2.2 如果设备没有挂载驱动的设备链表上，调用 device_bind_driver(dev):
                3.1 调用 driver_sysfs_add(dev)
                    3.1.1 if (dev->bus) -> blocking_notifier_call_chain():
                    dev + BUS_NOTIFY_BIND_DRIVER，触发 notify 机制
                    3.1.2 创建文件：
                        a. 驱动目录下指向设备的链接文件（名字同设备名）
                        b. 设备目录下名为 driver 的链接文件，指向驱动
                3.2 driver_bound(dev)
                    3.2.1 将设备挂到当前驱动上：dev->p->knode_driver 和 drv->p->klist_devices
                    3.2.2 defer probe 的一些处理
                    3.2.3 if (dev->bus) -> blocking_notifier_call_chain():
                    dev + BUS_NOTIFY_BOUND_DRIVER，触发 notify 机制
            2.3 根据 device_bind_driver(dev) 的返回结果判断 bind 
                是否成功，并作相关设置，最后返回。
        1.2 没有匹配的驱动
            2. 遍历设备所在总线上的驱动，逐个与设备进行匹配比较 __device_attach(drv, dev)
                3.1 driver_match_device(device_driver, device)，
                    通过 bus->match() 进行判断是否匹配
                3.2 driver_probe_device(drv, dev)   // 匹配后，触发 probe 机制
    ```
2. 驱动：`driver_attach(): /drivers/base/dd.c`
    ```c
    // version: 4.1.15
    // drivers/base/dd.c
    /**
     * driver_attach - try to bind driver to devices 
     * @drv: driver.
     *
     * Walk the list of devices that the bus has on it and try to
     * match the driver with each one. If driver_probe_device()
     * returns 0 and the @dev->driver is set, we've found a compatible pair
     */
    int driver_attach(struct device_driver *drv) {
        return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
    }
    EXPORT_SYMBOL_GPL(driver_attach);

    static int __driver_attach(struct device *dev, void *data) {
        struct device_driver *drv = data;
        /* Lock device and try to bind to it. We drop the error
         * here and always return 0, because we need to keep trying
         * to bind to devices and some drivers will return an error
         * simply if it didn't support the device.
         *
         * driver_probe_device() will spit a warning if there is
         * an error.
         */
        if (!driver_match_device(drv, dev))
            return 0;
        if (dev->parent)    /* Needed for USB */
            device_lock(dev->parent);
        device_lock(dev);
        if (!dev->driver)
            driver_probe_device(drv, dev);
        device_unlock(dev);
        if (dev->parent)
            device_unlock(dev->parent);
        return 0;
    }

    // drivers/base/base.h
    static inline int driver_match_device(struct device_driver *drv, struct device *dev) {
        return drv->bus->match ? drv->bus->match(dev, drv) : 1;
    }

    // drivers/base/bus.c
    /**
     * bus_for_each_dev - device iterator 
     * @bus: bus type.
     * @start: device to start iterating from.
     * @data: data for the callback.
     * @fn: function to be called for each device.
     *
     * Iterate over @bus's list of devices, and call @fn for each,
     * passing it @data. If @start is not NULL, we use that device
     * to begin iterating from.
     *
     * We check the return of @fn each time. If it returns anything
     * other than 0, we break out and return that value.
     *
     * NOTE: The device that returns a non-zero value is not retained
     * in any way, nor is its refcount incremented. If the caller
     * needs to retain this data, it should do so, and increment the
     * reference count in the supplied callback.
     */
    int bus_for_each_dev(struct bus_type *bus, struct device *start,
        void *data, int (*fn)(struct device *, void *)) {
        struct klist_iter i;
        struct device *dev;
        int error = 0;
        if (!bus || !bus->p)
            return -EINVAL;
        klist_iter_init_node(&bus->p->klist_devices, &i, (start ? &start->p->klist_bus : NULL));
        while ((dev = next_device(&i)) && !error)
            error = fn(dev, data);
        klist_iter_exit(&i);
        return error;
    }
    EXPORT_SYMBOL_GPL(bus_for_each_dev);
    ```
    - 调用链 + 功能：
    ```c
    driver_attach(device_driver)
        bus_for_each_dev(device_driver->bus, NULL, drv, __driver_attach)
    
    // 对总线上的每个 dev 执行 __driver_attach
    __driver_attach(device, device_driver)
        driver_match_device(device_driver, device) // 匹配，则往下执行；反之，直接返回
            drv->bus->match(dev, drv)   // 匹配，返回非 0 值。
                                        // 一般情况下，bus->match() 会继续调用 drv->match()
        if (!dev->driver)               // 只有当 dev 匹配 drv 且 dev 没有 bind 驱动，才执行 probe 逻辑
            driver_probe_device(drv, dev)   // 触发 probe 机制
    ```

#### probe 机制
1. 设备：`bus_probe_device(): /drivers/base/bus.c` + `driver_probe_device(): /drivers/base/dd.c`（同下）
    ```c
    /**
     * bus_probe_device - probe drivers for a new device 
     * @dev: device to probe
     * 
     * Automatically probe for a driver if the bus allows it.
     */
    void bus_probe_device(struct device *dev) {
        struct bus_type *bus = dev->bus;
        struct subsys_interface *sif;
        int ret;
        if (!bus)
            return;
        if (bus->p->drivers_autoprobe) {
            ret = device_attach(dev);
            WARN_ON(ret < 0);
        }
        mutex_lock(&bus->p->mutex);
        list_for_each_entry(sif, &bus->p->interfaces, node)
            if (sif->add_dev)
                sif->add_dev(dev, sif);
        mutex_unlock(&bus->p->mutex);
    }
    ```
    - 说明：`bus_probe_device()` 调用了执行 *match* 的 `device_attach()`，这里未按执行顺序将 `bus_probe_device()` 放到 *probe* 之前的部分，是考虑到在通过 `device_attach()` 完成匹配并触发 `driver_probe_device()` 后的 `sif->add_dev()` 应当也是属于 *probe* 的一部分。
    - 关于 `driver_probe_device()` 的说明见驱动部分。
    - 总线/驱动对设备的 *probe* 操作结束以后，遍历总线上的 *interfaces* 链表，逐个执行其中的回调 `add_dev()`。
2. 驱动：`driver_probe_device(): /drivers/base/dd.c`
    ```c
    // version: 4.1.15
    // drivers/base/dd.c
    /* driver_probe_device - attempt to bind device & driver together 
     * @drv: driver to bind a device to
     * @dev: device to try to bind the driver
     *
     * This function returns -ENODEV if the device is not registered,
     * 1 if the device is bound successfully and 0 otherwise.
     * 
     * This function must be called with @dev lock held. When called
     * for USB interface, @dev->parent lock must be held as well.
     */
    int driver_probe_device(struct device_driver *drv, struct device *dev) {
        int ret = 0;
        if (!device_is_registered(dev))
            return -ENODEV;
        pr_debug("bus: '%s': %s: matched device %s with driver %s\n", 
            drv->bus->name, __func__, dev_name(dev), drv->name);
        pm_runtime_barrier(dev);
        ret = really_probe(dev, drv);
        pm_request_ilde(dev);
        return ret;
    }

    static int really_probe(struct device *dev, struct device_driver *drv) {
        int ret = 0;
        int local_trigger_count = atomic_read(&deferred_trigger_count);
        atomic_inc(&probe_count);
        pr_debug("bus: '%s': %s: probing driver %s with device %s\n",
            drv->bus->name, __func__, drv->name, dev_name(dev));
        WARN_ON(!list_empty(&dev->devres_head));
        dev->driver = drv;
        /* If using pinctrl, bind pins now before probing */
        ret = pinctrl_bind_pins(dev);
        if (ret)
            goto probe_failed;
        if (driver_sysfs_add(dev)) {
            printk(KERN_ERR "%s: driver_sysfs_add(%s) failed\n",
                __func__, dev_name(dev));
            goto probe_failed;
        }
        if (dev->pm_domain && dev->pm_domain->activate) {
            ret = dev->pm_domain->activate(dev);
            if (ret)
                goto probe_failed;
        }
        if (dev->bus->probe) {
            ret = dev->bus->probe(dev);
            if (ret)
                goto probe_failed;
        } else if (drv->probe) {
            ret = drv->probe(dev);
            if (ret)
                goto probe_failed;
        }
        if (dev->pm_domain && dev->pm_domain->sync)
            dev->pm_domain->sync(dev);
        driver_bound(dev);
        ret = 1;
        pr_debug("bus: '%s': %s: bound device %s to driver %s\n",
            drv->bus->name, __func__, dev_name(dev), drv->name);
        goto done;
    probe_failed:
        devres_release_all(dev);
        driver_sysfs_remove(dev);
        dev->driver = NULL;
        dev_set_drvdata(dev, NULL);
        if (dev->pm_domain && dev->pm_domain->dismiss)
            dev->pm_domain->dismiss(dev);
        switch (ret) {
        case -EPROBE_DEFER:
            /* Driver requested deferred probing */
            dev_dbg(dev, "Driver %s requests probe deferral\n", drv->name);
            driver_deferred_probe_add(dev);
            /* Did a trigger occur while probing? Need to re-trigger if yes */
            if (local_trigger_count != atomic_read(&deferred_trigger_count))
                driver_deferred_probe_trigger();
            break;
        case -ENODEV:
        case -ENXIO:
            pr_debug("%s: probe of %s rejects match %d\n", drv->name, 
                dev_name(dev), ret);
            break;
        default:
            /* driver matched but the probe failed */
            printk(KERN_WARNING, "%s: probe of %s failed with error %d\n",
                drv->name, dev_name(dev), ret);
        }
        /* Ignore errors returned by ->probe so that the next 
         * driver can try its luck. 
         */
        ret = 0;
    done:
        atomic_dec(&probe_count);
        wake_up(&probe_waitqueue);
        return ret;
    }
    ```
    - 调用链 + 功能：
    ```c
    driver_probe_device(device_driver, device)
        // 如果 device 没有注册，则直接返回
        really_probe(device, device_driver)
            1. 设置 dev->driver = drv
            2. 设备的 pinctrl: pinctrl_bind_pins(dev)
                2.1 pins 资源通过 devres 机制管理
                2.2 查找 pins 中的默认配置，（如果）以及 sleep 和 idle 的配置
                2.3 使能 pins 中的默认配置
            3. 调用 driver_sysfs_add(dev)
                3.1 if (dev->bus) -> blocking_notifier_call_chain():
                    dev + BUS_NOTIFY_BIND_DRIVER，触发 notify 机制
                3.2 创建文件：
                    a. 驱动目录下指向设备的链接文件（名字同设备名）
                    b. 设备目录下名为 driver 的链接文件，指向驱动
            4. （如果） dev->pm_domain->activate(dev)
            5. 检查设备所在总线是否设置 probe 回调，有则调用该 probe；
               否则检查当前驱动是否设置 probe 回调，有则调用该 probe；
            6. （如果） dev->pm_domain->sync(dev)
            7. driver_bound(dev)
                7.1 将设备挂到当前驱动上：dev->p->knode_driver 和 drv->p->klist_devices
                7.2 defer probe 的一些处理
                7.3 if (dev->bus) -> blocking_notifier_call_chain():
                    dev + BUS_NOTIFY_BOUND_DRIVER，触发 notify 机制
            8. wake_up(&probe_waitqueue);
    // 注：如果驱动的 probe 返回了 -EPROBE_DEFER，则会触发 defer probe 机制         
    ```

#### remove 机制
1. 设备：`bus_remove_device(): /drivers/base/bus.c`
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    /**
     * bus_remove_device - remove device from bus 
     * @dev: device to be removed
     *
     * - Remove device from all interfaces.
     * - Remove symlink from bus's directory
     * - Delete device from bus's list
     * - Detach from its driver
     * - Drop reference taken in bus_add_device()
     */
    void bus_remove_device(struct device *dev) {
        struct bus_type *bus = dev->bus;
        struct subsys_interface *sif;
        if (!bus)
            return;
        mutex_lock(&bus->p->mutex);
        list_for_each_entry(sif, &bus->p->interfaces, node)
            if (sif->remove_dev)
                sif->remove_dev(dev, sif);
        mutex_unlock(&bus->p->mutex);
        sysfs_remove_link(&dev->kobj, "subsystem");
        sysfs_remove_link(&dev->bus->p->devices_kset->kobj, dev_name(dev));
        device_remove_attrs(dev->bus, dev);
        device_remove_groups(dev, dev->bus->dev_groups);
        if (klist_node_attached(&dev->p->knode_bus))
            klist_del(&dev->p->knode_bus);
        pr_debug("bus: '%s': remove device %s\n", dev->bus->name, dev_name(dev));
        device_release_driver(dev);
        bus_put(dev->bus);
    }

    // drivers/base/dd.c
    /**
     * device_release_driver - manually detach device from driver. 
     * @dev: device.
     *
     * Manually detach device from driver.
     * When called for a USB interface, @dev->parent lock must be held.
     */
    void device_release_driver(struct device *dev) {
        /* If anyone calls device_release_driver() recursively from
         * within their ->remove callback for the same device, they
         * will deadlock right here.
         */
        device_lock(dev);
        __device_release_driver(dev);
        device_unlock(dev);
    }
    EXPORT_SYMBOL_GPL(device_release_driver);

    /* __device_release_driver() must be called with @dev lock held.
     * When called for a USB interface, @dev->parent lock must be held as well.
     */
    static void __device_release_driver(struct device *dev) {
        struct device_driver *drv;
        drv = dev->driver;
        if (drv) {
            pm_runtime_get_sync(dev);
            driver_sysfs_remove(dev);
            if (dev->bus)
                blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
                    BUS_NOTIFY_UNBIND_DRIVER, dev);
            pm_runtime_put_sync(dev);
            if (dev->bus && dev->bus->remove)
                dev->bus->remove(dev);
            else if (drv->remove)
                drv->remove(dev);
            devres_release_all(dev);
            dev->driver = NULL;
            dev_set_drvdata(dev, NULL);
            if (dev->pm_domain && dev->pm_domain->dismiss)
                dev->pm_doamin->dismiss(dev);
            klist_remove(&dev->p->knode_driver);
            if (dev->bus)
                blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
                    BUS_NOTIFY_UNBOUND_DRIVER, dev);
        }
    }

    static void driver_sys_remove(struct device *dev) {
        struct device_driver *drv = dev->driver;
        if (drv) {
            sysfs_remove_link(&drv->p->kobj, kobject_name(&dev->kobj));
            sysfs_remove_link(&dev->kobj, "driver");
        }
    }
    ```
    - 功能 + 调用链：
    ```c
    bus_remove_device(device)
        1. 总线上的每个 subsys_interface 调用 remove_dev() 回调
        2. 删除 sysfs 中的链接文件：
            2.1 设备目录下的 subsystem 文件
            2.2 总线 devices 目录下的设备文件
        3. 删除设备目录下总线提供的属性文件、属性文件集合
        4. 从总线上移除该设备
        5. 调用 device_release_driver(device)
        6. put 总线引用计数
    
    // device_release_driver:
    device_release_driver(device)
        __device_release_driver(device)
            1. 删除 sysfs 中的链接文件：驱动 devices 目录下的设备文件 + 设备目录下的 driver 文件
            2. 总线上的通知机制，通知内容：正要删除设备 + device
            3. 调用总线或者驱动的 remove 回调（前者优先）
            4. 调用 devres_release_all(device)，释放 attached to this device 的资源
            5. 调用（如有） dev->pm_domain->dismiss(device)
            6. 从驱动上移除该设备
            7. 总线上的通知机制，通知内容：设备已删除 + device
    ```
2. 驱动：`bus_remove_driver(): /drivers/base/bus.c`
    ```c
    // version: 4.1.15
    // drivers/base/bus.c
    void bus_remove_driver(struct device_driver *drv) {
        if (!drv->bus)
            return;
        if (!drv->suppress_bind_attrs)
            remove_bind_files(drv);
        driver_remove_groups(drv, drv->bus->drv_groups);
        driver_remove_file(drv, &driver_attr_uevent);
        klist_remove(&drv->p->knode_bus);
        pr_debug("bus: '%s': remove driver %s\n", drv->bus->name, drv->name);
        driver_detach(drv);
        module_remove_driver(drv);
        kobject_put(&drv->p->kobj);
        bus_put(drv->bus);
    }

    // drivers/base/dd.c
    /** 
     * driver_detach - detach driver from all devices it controls. 
     * @drv: driver
     */
    void driver_detach(struct device_driver *drv) {
        struct device_private *dev_prv;
        struct device *dev;
        for (;;) {
            spin_lock(&drv->p->klist_devices.k_lock);
            if (list_empty(&drv->p->klist_devices.k_list)) {
                spin_unlock(&drv->p->klist_devices.k_lock);
                break;
            }
            dev_prv = list_entry(drv->p->klist_devices.k_list.prev,
                struct device_private, knode_driver.n_node);
            dev = dev_prv->device;
            get_device(dev);
            spin_unlock(&drv->p->klist_devices.k_lock);
            if (dev->parent)    /* Needed for USB */
                device_lock(dev->parent);
            device_lock(dev);
            if (dev->driver == drv)
                __device_release_driver(dev);
            device_unlock(dev);
            if (dev->parent)
                device_unlock(dev->parent);
            put_device(dev);
        }
    }

    // drivers/base/module.c
    void module_remove_driver(struct device_driver *drv) {
        struct module_kobject *mk = NULL;
        char *driver_name;
        if (!drv)
            return;
        sysfs_remove_link(&drv->p->kobj, "module");
        if (drv->owner)
            mk = &drv->owner->mkobj;
        else if (drv->p->mkobj)
            mk = &drv->p->mkobj;
        if (mk && mk->drivers_dir) {
            driver_name = make_driver_name(drv);
            if (driver_name) {
                sysfs_remove_link(mk->drivers_dir, driver_name);
                kfree(driver_name);
            }
        }
    }
    ```
    - 调用链 + 功能
    ```c
    bus_remove_driver(driver)
        1. 删除属性文件：
            1.1 （如果）bind 和 unbind 文件
            1.2 总线提供的属性集文件，drv->bus->drv_groups
            1.3 uevent 文件
        2. 从总线上移除 drv
        3. driver_detach(drv)
            3.1 从 drv->p->klist_devices 上获取该 drv 驱动的设备
            3.2 对每个设备：__device_release_driver(dev)，见设备 remove 部分解析
        4. 将 drv 从 module 上移除
            4.1 删除驱动目录下的 module 文件
            4.2 从模块的驱动目录下删除该驱动对应的文件（文件名格式：“总线名:驱动名”）
        5. 释放驱动的引用计数：kobject_put(&drv->p->kobj)，可能触发进一步的 kobject_release()
        6. put 总线的引用计数
    
    // 当驱动的引用计数为 0 时，触发 kobject_release()
    kobject_release()
        ... (详情见 kobject 的 remove 机制)
        // drv->p->kobj 的 kobj_type 为 driver_ktype
        driver_ktype->release(kobj)
            kfree(drv_priv) // 释放对应的 driver_private
    ```

#### notify 机制
1. 触发函数：`blocking_notifier_call_chain(): /kernel/notifier.c`
    ```c
    // 调用示例：blocking_notifier_call_chain(&dev->bus->p->bus_notifier, BUS_NOTIFY_BOUND_DRIVER, dev);
    blocking_notifier_call_chain(struct blocking_notifier_head *nh, unsigned long val, void *v)
        __blocking_notifier_call_chain(nh, val, v, -1, NULL)
            notifier_call_chain(&nh->head, val, v, nr_to_call, nr_calls)
                struct notifier_block *nb, *next_nb;
                ...
                ret = nb->notifier_call(nb, val, v)
                ...
    ```
2. 触发类型：
    ```c
    // version: 4.1.15
    // include/linux
    /* All 4 notifiers below get called with the target struct 
     * device as an argument. Note that those functions are likely
     * to be called with the device lock held in the core, so
     * be careful.
     */
    #define BUS_NOTIFY_ADD_DEVICE   0x00000001      /* device added */
    #define BUS_NOTIFY_DEL_DEVICE   0x00000002      /* device to be removed */
    #define BUS_NOTIFY_REMOVED_DEVICE   0x00000003  /* device removed */
    #define BUS_NOTIFY_BIND_DRIVER  0x00000004      /* driver about to be bound */

    #define BUS_NOTIFY_BOUND_DRIVER     0x00000005  /* driver bound to device */
    #define BUS_NOTIFY_UNBIND_DRIVER    0x00000006  /* driver about to be unbound */
    #define BUS_NOTIFY_UNBOUND_DRIVER   0x00000007  /* driver is unbound from the deivce */
    ```
3. 触发时机：在设备状态发生变化时，触发 *notify* 机制。驱动的状态变化不会触发，除非驱动的状态变化引起了设备的状态变化。
   - *add* 设备时：
     - `device_add()` - `BUS_NOTIFY_ADD_DEVICE`，设备已经添加
   - *probe* 设备/驱动时：
     - `really_probe()` - `BUS_NOTIFY_BIND_DRIVER`，设备正要 *bind* 驱动（驱动 *probe* 时会调用）
     - `really_probe()` - `BUS_NOTIFY_BOUND_DRIVER`，设备已经 *bind* 驱动（驱动 *probe* 时会调用）
   - *remove* 设备/驱动时：
     - `device_del()` - `BUS_NOTIFY_DEL_DEVICE`，设备正要移除
     - `__device_release_driver()` - `BUS_NOTIFY_UNBIND_DRIVER`，设备正在 *unbind* 驱动（驱动 *remove* 时会调用）
     - `__device_release_driver()` - `BUS_NOTIFY_UNBOUND_DRIVER`，设备已经 *unbind* 驱动（驱动 *remove* 时会调用）
     - `device_del()` - `BUS_NOTIFY_REMOVED_DEVICE`，设备已经移除
5. 注册 *notifier*
    ```c
    // version: 4.1.15
    // include/linux/device.h

    /* Bus notifiers: Get notified of addition/removal of devices
     * and binding/unbinding of drivers to devices.
     * In the long run, it should be a replacement for the platform
     * notify hooks.
     */
    // 返回值：当前总是返回 0
    extern int bus_register_notifier(struct bus_type *bus,
        struct notifier_block *nb);
    // 返回值：0-成功，-ENOENT-失败
    extern int bus_unregister_notifier(struct bus_type *bus,
        struct notifier_block *nb);

    // include/linux/notifier.h
    typedef int (*notifier_fn_t)(struct notifier_block *nb, 
        unsigned long action, void *data);

    struct notifier_block {
        notifier_fn_t notifier_call;
        struct notifier_block __rcu *next;
        int priority;
    };
    ```
    - 示例：
    ```c
    // version: 4.1.15
    // drivers/i2c/i2c-dev.c
    static int __init i2c_dev_init(void) {
        ...
        res = bus_register_notifier(&i2c_bus_type, i2cdev_notifier);
        ...
    }

    static struct notifier_block i2cdev_notifier = {
        .notifier_call = i2cdev_notifier_call,
    };

    static int i2cdev_notifier_call(struct notifier_block *nb, 
        unsigned long action, void *data) {
        struct device *dev = data;
        switch (action) {
        case BUS_NOTIFY_ADD_DEVICE:
            return i2cdev_attach_adapter(dev, NULL);
        case BUS_NOTIFY_DEL_DEVICE:
            return i2cdev_detach_adapter(dev, NULL);
        }
        return 0;
    }
    ```

#### uevent 机制

### 设备
#### 概述
`struct device` 为设备基类，提供设备的基本属性。类似于 `struct kobject`，它被嵌入到更 *high level* 的设备描述结构体。

#### 数据结构描述
```c
// version: 4.1.15
// include/linux/device.h
/**
 * struct device - The basic device structure. 
 * @parent: The device's "parent" device, the device to which
 *  it is attached. In most cases, a parent device is some sort
 *  of bus or host controller. If parent is NULL, the device,
 *  is a top-level device, which is not usually what you want.
 * @p: Holds the private data of the driver core portions of 
 *  the device. See the comment of the struct device_private
 *  for detail.
 * @kobj: A top-level, abstract class from which other class
 *  are derived.
 * @init_name: Initial name of the device.
 * @type: The type of device. This identifies the device type
 *  and carries type-specific information.
 * @mutex: Mutex to synchronize calls to its driver.
 * @bus: Type of bus device is no.
 * @driver: which driver has allocated this.
 * @platform_data: Platform data specific to the device.
 *  Example: For devices on custom boards, as typical of
 *  embedded and SOC based hardware, Linux often used platform_
 *  data to point to board-specific structures describing 
 *  devices and how they are wired. That can include what ports
 *  are available, chip variants, which GPIO pins act in what 
 *  additional role, and so on. This shrinks the "Board 
 *  Support Packages" (BSPs) and minimizes board-specific
 *  #ifdefs in drivers.
 * @driver_data: Private pointer for driver specific info.
 * @power: For device power management. See Documentation/
 *  power/devices.txt for details.
 * @pm_domain: Provide callbacks that are executed during
 *  system suspend, hibernation, system resume and during
 *  runtime PM transitions along with subsystem-level and
 *  driver-level callbacks.
 * @pins: For device pin management. See Documentation/
 *  pinctrl.txt for details.
 * @numa_node: NUMA node this device is close to.
 * @dma_mask: Dma mask (if dma'ble device).
 * @coherent_dma_mask: Like dma_mask, buf for alloc_coherent
 *  mapping as not all hardware supports 64-bit addresses for
 *  consistent allocations such descriptors.
 * @dma_pfn_offset: offset of DMA memory range relatively of 
 *  RAM
 * @dma_parms: A low level driver may set these to teach IOMMU
 *  code about segment limitations.
 * @dma_pool: Dma pools (if dma'ble device).
 * @dma_mem: Internal for coherent mem override.
 * @cma_area: Contiguous memory area for dam allocations
 * @archdata: For arch-specific additions.
 * @of_node: Associated device tree node.
 * @fwnode: Associated device node supplied by platform
 *  firmware.
 * @devt: For creating the sysfs "dev".
 * @id: device instance.
 * @devres_lock: Spinlock to protect the resource of the device
 * @devres_head: The resource list of the device.
 * @knode_class: The node used to add the device to the class 
 *  list.
 * @class: The class of the device.
 * @groups: Optional attribute groups.
 * @release: Callback to free the device after all references
 *  have gone away. This should be set by the allocator of the
 *  device (i.e. the bus driver that discovered the device).
 * @iommu_group: IOMMU group the device belongs to.
 * @offline_disable: If set, the device is permanently online.
 * @offline: Set after successful invocation of bus type's
 * .offline().
 * At the lowest level, every device in a Linux system is
 * represented by an instance of struct device. The device
 * structure contains the information that the device model
 * core needs to model the system. Most subsystems, however,
 * track additional information about the devices they host.
 * As a result, it is rare for devices to be represented by
 * bare device structures; instead, that structure, like
 * kobject structures, is usually embedded within a high-
 * level representation of the device.
 */
struct device {
   struct device *parent;
   struct device_private *p;
   struct kobject kobj;
   const char *init_name;
   const struct devie_type *type;
   struct mutex mutex;
   struct bus_type *bus;
   struct device_driver *driver;
   void *platform_data;       /* platform specific data, device core doesn't touch it */
   void *driver_data;            /* driver data, set and get with dev_set/get_drvdata */
   struct dev_pm_info power;
   struct dev_pm_domain *pm_domian;
#ifdef CONFIG_PINCTRL
   struct dev_pin_info *pins;
#endif
#ifdef CONFIG_NUMA
   int numa_node;
#endif
   u64 *dma_mask;
   u64 coherent_dma_mask;
   unsigned long dma_pfn_offset;
   struct device_dma_parameters *dma_params;
   struct list_head dma_pools;
   struct dma_coherent_mem *dma_mem;
#ifdef CONFIG_DMA_CMA
   struct cma *cma_area;
#endif
   struct dev_archdata archdata;    /* arch specific additions */
   struct device_node *of_node;
   struct fwnode_handle *fwnode;
   dev_t devt;
   u32 id;
   spinlock_t devres_lock;
   struct list_head devres_head;
   struct klist_node knode_class;
   struct class *class;
   const struct attribute_group **groups;
   void (*release)(struct device *dev);
   struct iommu_group *iommu_group;
   bool offline_disable:1;
   bool offline:1;
};
```
在源文件的注释中已对各个字段进行了描述，下面的内容为对注释的补充：

##### platform_data
`platform_data` - `void *`（部分内容来自 *kimi* 问答）

- `struct device` 中的 `platform_data` 是一个历史遗留的指针字段，用于在设备树普及之前，向平台设备驱动传递板级特定的配置数据。
- 通常指向驱动自定义的结构体，由板级代码填充。
- 内核设备核心不管理、不解析、不释放。
- 通常在设备注册时设置，驱动 *probe* 时读取。

1. 在设备树普及之前（*ARM* 架构约 *2012* 年前），硬件信息硬编码在 *C* 文件中：
    ```c
     // arch/arm/mach-xxx/board-xxx.c
     // 1. 定义板级特定数据结构
     struct my_platform_data {
         int gpio_reset;
         int irq_number;
         u32 clock_freq;
         const char *firmware_name;
     };

     // 2. 实例化配置
     static struct my_platform_data mydev_pdata = {
         .gpio_reset = 42,
         .irq_number = 7,
         .clock_freq = 25000000,
         .firmware_name = "my_fw.bin",
     };

     // 3. 注册平台设备时传递
     static struct platform_device my_device = {
         .name = "my_driver",
         .id   = 0,
         .dev  = {
             .platform_data = &mydev_pdata,      // 传递指针
         },
     };

     // 4. 启动时注册
     static int __init board_init(void) {
         return platform_device_register(&my_device);
     }

     // 在驱动端读取
     // drivers/xxx/my_driver.c
     static int my_driver_probe(struct platform_device *pdev) {
         struct my_platform_data *pdata = dev_get_platdata(&pdev->dev);
         if (!pdata) {
             dev_err(&pdev->dev, "No platform data!\n");
             return -EINVAL;
         }
         // 使用配置
         gpio_request(pdata->gpio_reset, "reset");
         ...
     }
    ```

2. 设备树普及后，硬件描述从 *C* 代码迁移到 *DTS* 文件，`platform_data` 被 `struct device_node *of_node` 和属性解析 *API* 取代：
     ```dts
     // arch/arm/boot/dts/my-board.dts
     my_device@12340000 {
         compatible = "vendor,my-device";
         reg = <0x12340000 0x1000>
         interrupts = <7 IRQ_TYPE_EDGE_RISING>;
         
         // 这些属性替代了 platform_data 结构体
         reset-gpios = <&gpio1 10 GPIO_ACTIVE_LOW>;
         clock-frequency = <25000000>;
         firmware-name = "my_fw.bin";
     };
     ```
     ```c
     // 驱动端读取
     static int my_driver_probe(struct platform_device *pdev) {
         struct device *dev = &pdev->dev;
         struct my_driver_data *priv;
         int ret;

         // 不再使用 platform_data!
         // 该用设备树解析：
         priv->reset_gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
         if (IS_ERR(priv->reset_gpio))
             return PTR_ERR(priv->reset_gpio);
         
         ret = device_property_read_u32(dev, "clock-frequency", &priv->clk_freq);
         if (ret)
             return ret;
         ...
     }
     ```

3. 相关驱动源码：`drivers/leds/leds-gpio.c` 中，在 `gpio_led_probe()` 内会区分是否使用设备树来获取设备硬件信息：`if (pdata && pdata->num_leds)` 成立时就是用 `platform_data` 字段来解析硬件信息，否则通过设备树。

##### fwnode
`fwnode` - `struct fwnode_handle *`（部分内容来自 *kimi* 问答）

- `fwnode, Firmware Node` 是 *Linux* 内核设备模型中用于统一抽象不同固件描述接口的字段，它是设备树（*Device Tree*）、*ACPI* 等固件描述机制的通用封装。

1. 需要 *fwnode* 的原因：
    历史问题：多种固件描述方式。
    | 平台 | 固件接口 | 数据结构 |
    | ----- | ----- | ----- |
    | *ARM/ARM64/RISC-V* | *Device Tree* | `struct device_node` |
    | *x86* | *ACPI* | `struct acpi_device_node` |
    | 旧 *PowerPC* | *Open Firmware* | `struct device_node` |
    | 某些嵌入式 | 静态平台数据 | 无 |

    驱动痛点：
    - 同样的驱动需要在不同平台上支持 DT 或 ACPI；
    - 代码里写 `#ifdef CONFIG_OF` 或 `#ifdef CONFIG_ACPI` 很丑陋；
    - 无法统一处理属性读取、中断解析、DMA 配置等。

2. *fwnode* 的本质：统一抽象层
    ```c
    // include/linux/fwnode.h
    // 与内核 4.1.15 内容不一致，没有 ops 和 flags 字段，有 type 字段
    struct fwnode_handle {
        struct fwnode_handle *secondary;    // 备用 fwnode（如 DT + ACPI 并存）
        const struct fwnode_operations *ops;  // 操作函数表（多态）
        unsigned int flags;
    };

    // include/linux/device.h
    struct device {
        // ...
        struct fwnode_handle *fwnode;         // ← 指向固件节点
        // ...
    };
    ```
    - 核心思想：驱动不再直接访问 *device_node* 或 *acpi_device*，而是通过 *fwnode_handle* 接口操作。

3. *fwnode* 的操作函数表
    ```c
    // include/linux/fwnode.h
    // 内核 4.1.15 没有这些内容
    struct fwnode_operations {
        struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
        void (*put)(struct fwnode_handle *fwnode);
        bool (*device_is_available)(const struct fwnode_handle *fwnode);
        const void *(*device_get_match_data)(const struct fwnode_handle *fwnode,
                                            struct device *dev);
        bool (*property_present)(const struct fwnode_handle *fwnode,
                                const char *propname);
        int (*property_read_int_array)(const struct fwnode_handle *fwnode,
                                        const char *propname,
                                        unsigned int elem_size, void *val,
                                        size_t nval);
        int (*property_read_string_array)(const fwnode_handle_t *fwnode,
                                        const char *propname,
                                        const char **val, size_t nval);
        // ... 更多操作
    };
    ```
    不同固件类型的实现：
    | 固件类型 | *fwnode\_ops* 实现文件 | 具体结构体 |
    | ----- | ----- | ----- |
    | *Device Tree* | `drivers/of/property.c` | `struct device_node` 内嵌 `struct fwnode_handle` |
    | *ACPI* | `drivers/acpi/property.c` | `struct acpi_device` 内嵌 `struct fwnode_handle` |
    | 软件节点 | `drivers/base/swnode.c` | `struct software_node` |

4. 实际使用：驱动如何操作 *fwnode*
    ```c
    /* 属性读取（统一 API）*/
    // 旧方式（DT 专用）：
    u32 val;
    of_property_read_u32(pdev->dev.of_node, "clock-frequency", &val);

    // 新方式（fwnode 通用）
    u32 val;
    device_property_read_u32(&pdev->dev, "clock-frequency", &val);

    /* 中断解析 */
    // 旧方式
    int irq = of_irq_get(pdev->dev.of_node, 0);
    // 新方式
    int irq = fwnode_irq_get(dev_fwnode(&pdev->dev), 0);
    // 或者更简单的
    int irq = platform_get_irq(pdev, 0);    // 内部也调用 fwnode

    /* DMA 配置 */
    int dma_mask = device_property_read_u64(&pdev->dev, "dma-mask", &mask);
    ```

5. `fwnode` 的来源：
    ```c
    // Device Tree 平台
    // drivers/of/platform.c
    static int of_platform_device_create(struct device_node *np, const char *bus_id, struct device *parent) {
        struct platform_device *dev;

        dev = platform_device_alloc("", -1);
        // 关键：将 device_node 与 fwnode 关联
        dev->dev.of_node = of_node_get(np);
        dev->dev.fw_node = &np->fwnode;

        ...
        platform_device_add(dev);
    }
    // ACPI 平台
    // drivers/acpi/acpi_platform.c
    static int acpi_create_platfrom_device(struct acpi_device *adev, const struct acpi_device_id *id) {
        // 略
    }
    ```

6. *secondary fwnode*: *DT + ACPI* 并存
    ```c
    // 某些平台（如 ARM64 Windows 设备）同时存在 DT 和 ACPI
    struct fwnode_handle {
        struct fwnode_handle *secondary;    // 备用节点
        ...
    };

    // 设备匹配时，先用 primary，再 fallback 到 secondary
    const void *device_get_match_data(struct device *dev) {
        const void *data;
        if (dev->fwnode && dev->fwnode->ops->device_get_match_data)
                data = dev->fwnode->ops->device_get_match_data(...);
        
        if (!data && dev->fwnode->secondary)
                data = dev->fwnode->secondary->ops->device_get_match_data(...);
        
        return data;
    }
    ```

7. 与 *of_node* 的关系
    ```c
    // include/linux/device.h
    struct device {
        // ...
    #ifdef CONFIG_OF
        struct device_node *of_node;           // DT 节点（历史遗留）
    #endif
    #ifdef CONFIG_ACPI
        struct acpi_device_node *acpi_node;    // ACPI 节点（历史遗留）
    #endif
        struct fwnode_handle *fwnode;          // 统一接口（推荐新代码使用）
    };
    ```
    演进路线：
    - 旧代码：直接访问 `dev->of_node`；
    - 新代码：通过 `dev->fwnode` 或辅助函数（`device_property_read_*`）；
    - 长期目标：`of_node`/`acpi_node` 逐步被 `fwnode` 取代。

8. 辅助函数快捷访问
    ```c
    // include/linux/property.h
    // 获取设备的 fwnode
    static inline struct fwnode_handle *dev_fwnode(struct device *dev) {
        return dev->fwnode;
    }
    // 判断设备是否有某属性
    bool device_property_preset(struct device *dev, const char *propname);
    // 读取各种类型属性
    int device_property_read_u8(struct device *dev, const char *propname, u8 *val);
    ...
    ```

9. 总结：`struct device` 中的 `fwnode` 是 *Linux* 内核为统一不同固件描述接口（*Device Tree、ACPI* 等）而设计的抽象层。它通过函数指针表（`fwnode_operations`）实现多态，让驱动代码无需关心底层是 *DT* 还是 *ACPI*，统一使用 `device_property_read_*` 等 *API* 访问硬件属性。这是内核设备模型向平台无关化演进的关键设计。

- 补充：与设备树的关联
    ```c
    // include/linux/of.h
    static inline bool is_of_node(struct fwnode_handle *fwnode) {
        return fwnode && fwnode->type == FWNODE_OF;
    }

    static inline struct device_ndoe *of_node(struct fwnode_handle *fwnode) {
        return fwnode ? 
        container_of(fwnode, struct device_node, fwnode) : NULL;
    }
    ```

##### devt
`devt` - `dev_t`, *For creating the sysfs "dev"*.

1. 主设备号
    ```c
    // version: 4.1.15
    // include/upai/linux/major.h
    /* 
     * This file has definitions for major device numbers. 
     * For the device number assignments, see Documentation/devices.txt
     */
    #define UNNAMED_MAJOR       0
    #define MEM_MAJOR           1
    #define RAMDISK_MAJOR       1
    #define FLOPPY_MAJOR        2       // floppy - 软盘
    #define PTY_MASTER_MAJOR    2
    ...
    ```

2. 设备号的唯一性
   - 在同一类型的设备中（如在字符设备中、在块设备中）的设备号是唯一的，可以根据设备号在其中查找对应的设备描述
   - 在不同类型的设备间（如在字符设备和块设备间）设备号不是唯一的，不同类型的设备可以拥有相同的设备号

##### p
`p` - `struct device_private`

通过该字段构建设备与子系统间的联系

1. 数据结构描述
    ```c
    // version: 4.1.15
    // drivers/base/base.h
    struct device_private {
        struct klist klist_children;
        struct klist_node knode_parent;
        struct klist_node knode_driver;
        struct klist_node knode_bus;
        struct list_head deferred_probe;
        struct device *device;
    };
    ```
    
2. `deferred_probe` 字段（部分内容来自 *kimi* 问答）：
   - `struct device_private` 中的 `deferred_probe` 字段是 *Linux* 设备模型中 延迟探测（*Deferred Probe*）机制 的核心数据结构。
   - *Deferred Probe* 机制的背景：
     - 问题场景：在复杂的 *SoC* 系统中，设备之间存在依赖关系：
        ```
        ┌─────────────────────────────────────────────────────────────────────────────┐
        │                         设备依赖关系示例                                      │
        ├─────────────────────────────────────────────────────────────────────────────┤
        │                                                                             │
        │   摄像头驱动 probe()                                                      │
        │        ↓                                                                    │
        │   需要获取 regulator（电源）──→ regulator 驱动尚未加载！                      │
        │        ↓                                                                    │
        │   probe() 返回 -EPROBE_DEFER                                               │
        │        ↓                                                                    │
        │   内核：这个设备暂时无法 probe，等依赖的 regulator 准备好后再试                │
        │                                                                             │
        └─────────────────────────────────────────────────────────────────────────────┘
        ```
     - 典型依赖：
       - *GPIO* 控制器
       - *Regulator*（电源管理）
       - *Clock*（时钟）
       - *PinCtrl*（引脚控制）
       - *DMA* 控制器
       - 中断控制器
   - `deferred_probe` 字段的作用
        | 特性 | 说明 |
        |------|------|
        | 类型 | `struct list_head` — 链表节点 |
        | 所属链表 | 全局 `deferred_probe_pending_list` |
        | 作用 | 将因依赖未满足而 *deferred* 的设备链接起来，等待重新 *probe* |
   - *Deferred Probe* 的核心流程
        ```
        ┌─────────────────────────────────────────────────────────────────────────────┐
        │                    Deferred Probe 完整流程                                    │
        ├─────────────────────────────────────────────────────────────────────────────┤
        │                                                                             │
        │  1. 驱动 probe() 尝试获取资源                                                │
        │  ─────────────────────────────                                              │
        │     devm_regulator_get(dev, "vcc");                                         │
        │     ↓                                                                       │
        │     regulator 驱动尚未注册 → 返回 -EPROBE_DEFER                              │
        │                                                                             │
        │  2. 内核处理 -EPROBE_DEFER                                                  │
        │  ─────────────────────────                                                  │
        │     driver_probe_device()                                                   │
        │         ↓                                                                   │
        │     if (ret == -EPROBE_DEFER)                                               │
        │         driver_deferred_probe_add(dev);  ← 加入延迟探测队列                    │
        │                                                                             │
        │  3. 设备加入 deferred_probe_pending_list                                     │
        │  ─────────────────────────────────────                                      │
        │                                                                             │
        │     deferred_probe_pending_list                                             │
        │         ↓                                                                   │
        │     ┌─────────┐    ┌─────────┐    ┌─────────┐                            │
        │     │ device_A│───→│ device_B│───→│ device_C│───→ ...                      │
        │     │ (camera)│    │ (audio) │    │ (video) │                                │
        │     │ deferred│    │ deferred│    │ deferred│                                │
        │     │ _probe  │    │ _probe  │    │ _probe  │                                │
        │     └─────────┘    └─────────┘    └─────────┘                                │
        │                                                                             │
        │  4. 依赖驱动注册完成                                                          │
        │  ───────────────────                                                         │
        │     regulator 驱动终于加载了！                                               │
        │         ↓                                                                   │
        │     regulator_register()                                                    │
        │         ↓                                                                   │
        │     driver_deferred_probe_trigger();  ← 触发重新 probe                       │
        │                                                                             │
        │  5. 重新尝试 probe                                                            │
        │  ───────────────────                                                         │
        │     遍历 deferred_probe_pending_list                                         │
        │         ↓                                                                   │
        │     对每个设备：device_attach(dev)  ← 再次尝试匹配和 probe                     │
        │         ↓                                                                   │
        │     成功 → 从链表中移除                                                       │
        │     仍失败 → 保留在链表中，等待下次触发                                        │
        │                                                                             │
        └─────────────────────────────────────────────────────────────────────────────┘
        ```
   - 关键代码解析
        ```c
        /* 加入延迟队列 */
        // drivers/base/dd.c
        void driver_deferred_probe_add(struct device *dev)
        {
            // 将设备的 deferred_probe 节点加入全局链表
            list_add_tail(&dev->p->deferred_probe, &deferred_probe_pending_list);
            
            // 记录 deferred 原因（调试）
            dev_dbg(dev, "Added to deferred list\n");
        }

        /* 触发重新 probe */
        // drivers/base/dd.c
        void driver_deferred_probe_trigger(void)
        {
            struct device *dev, *tmp;
            
            // 遍历延迟探测链表
            list_for_each_entry_safe(dev, tmp, &deferred_probe_pending_list, 
                                    p->deferred_probe) 
            {
                // 从链表中移除
                list_del_init(&dev->p->deferred_probe);
                
                // 重新尝试 probe
                device_attach(dev);
            }
        }
        ```
   - 调试 *Deferred Probe*
        ```bash
        # 查看 deferred probe 的设备
        cat /sys/kernel/debug/devices_deferred

        # 开启调试日志
        echo 'file drivers/base/dd.c +p' > /sys/kernel/debug/dynamic_debug/control

        # 查看内核日志
        dmesg | grep -i "defer"
        ```
   - 总结
        | 问题 | 答案 |
        |------|------|
        | `deferred_probe` 是什么 | 将设备链接到全局延迟探测队列的链表节点 |
        | 为什么需要它 | 解决设备驱动依赖关系导致的 *probe* 时序问题 |
        | 什么时候加入链表 | `probe()` 返回 `-EPROBE_DEFER` 时 |
        | 什么时候移出链表 | 依赖满足后重新 *probe* 成功时 |
        | 谁触发重新 *probe* | 被依赖的驱动注册完成时调用 `driver_deferred_probe_trigger()` |

        这是 Linux 设备模型处理复杂依赖关系的关键机制，确保了驱动加载的正确时序。

#### 设备注册
```c
// include/linux/device.h
/* High level routines for use by the bus drivers */
extern int __must_check device_register(struct device *dev);
extern void device_unregister(struct device *dev);
extern void device_initialize(struct device *dev);
extern int __must_check device_add(struct device *dev);
extern void device_del(struct device *dev);
...

/* Easy functions for dynamically creating devices on the fly */
extern struct device *device_create_vargs(struct class *cls, 
    struct device *parent,
    dev_t devt,
    void *drvdata,
    const char *fmt,
    va_list vargs);
extern __printf(5, 6)
struct devcie *device_create(struct class *cls, struct device *parent,
    dev_t devt, void *drvdata, const char *fmt, ...);
extern __printf(6, 7)
struct device *device_create_with_groups(struct class *cls,
    struct device *parent, dev_t devt, void *drvdata,
    const struct attribute_group **groups,
    const char *fmt, ...);
extern void device_destroy(struct class *cls, dev_t devt);
```

- 设备注册的两种方式：
  1. `device_create()` - 注册到 *class* 上
    ```c
    device_create(class, parent, devt, drvdata, fmt, ...)
        device_create_vargs(class, parent, devt, drvdata, fmt, vargs)
            device_create_groups_vargs(class, parent, devt, drvdata, NULL, fmt, args)
                struct device *dev = NULL;
                dev = kzalloc(sizeof(*dev), GFP_KERNEL);
                device_initialize(dev);
                dev->devt = devt;
                dev->class = class;
                dev->parent = parent;
                dev->groups = NULL;
                dev->release = device_create_release;
                dev_set_drvdata(dev, drvdata);
                retval = kobject_set_name_vargs(&dev->kobj, fmt, args);
                device_add(dev);
    ```
    - 函数流程 + 功能可以总结为：
       1. 分配 *device* 内存；
       2. 初始化 *device*，`device_initialize(device)`
       3. 设置 *device* 的字段，包括：`devt`、`class`、`parent`、`release`、`driver_data`、`groups`
       4. 设置 *device* 对应 *kobject* 的名字
       5. 注册 *device*，`device_add()`
  2. `device_register()` - 注册到 *bus* 上 
    ```c
    device_register(device)
        device_initialize(device)
        device_add(dev)
    ```
  3. 对比两种注册方式可知，注册设备的核心分为两个步骤：初始化 + 注册。

- 函数源码：
  1. `device_create()`
    ```c
    // version: 4.1.15
    // drivers/base/core.c
    /**
     * device_create - creates a device and registers it with 
     * sysfs.
     * @class: pointer to the struct class that this device should
     *  be registered to
     * @parent: pointer to the parent struct device of this new device,
     *  if any
     * @devt: the dev_t for the char device to be added
     * @drvdata: the data to be added to the device for callbacks
     * @fmt: string for the device's name
     *
     * This function can be used by char device classes. A structure
     * device will be created in sysfs, registered to the specified
     * class.
     *
     * A "dev" file will be created, showing the dev_t for the device,
     * if the dev_t is not 0,0.
     * If a pointer to a parent struct device is passed in, the newly
     * created struct device will be a child of that device in sysfs.
     * The pointer to the struct device will be returned from the 
     * call. Any further sysfs files that might be required can be
     * created using this pointer.
     *
     * Returns &struct device pointer on success, or ERR_PTR() on error.
     * Note: the struct class passed to this function must have 
     * previously been created with a call to class_create().
     */
    struct device *device_create(struct class *class, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...) {
        va_list vargs;
        struct device *dev;
        va_start(vargs, fmt);
        dev = device_create_vargs(class, parent, devt, drvdata, fmt, vargs);
        va_end(vargs);
        return dev;
    }
    EXPORT_SYMBOL_GPL(device_create);

    /**
     * device_create_vargs - creates a device and registers it 
     * with sysfs
     * 参数说明：略
     * This function can be used by char device classes. A struct
     * device will be created in sysfs, registered to specified class.
     *
     * A "dev" file will be created, showing the dev_t for the device,
     * if the dev_t is not 0,0
     * If a pointer to parent struct device is passed in, the newly
     * created struct device will be child of that device in sysfs.
     * The pointer to the struct device will be returned from the call.
     * Any further sysfs files that might be required can be created
     * using this pointer.
     * ... (同 device_create)
     */
    struct device *device_create_vargs(struct class *class, struct device *parent, dev_t devt, void *drvdata, const char *fmt, va_list args) {
        return device_create_groups_vargs(class, parent, devt, drvdata, NULL, fmt, args);
    }
    EXPORT_SYMBOL_GPL(device_create_vargs);

    static struct device *device_create_groups_vargs(struct class *class, struct device *parent, dev_t devt, void *drvdata, const struct attribute_group **groups, const char *fmt, va_list args) {
        struct device *dev = NULL;
        int retval = -ENODEV;

        if (class == NULL || IS_ERR(class))
            goto error;
        
        dev = kzalloc(sizeof(*dev), GFP_KERNEL);
        if (!dev) {
            retval = -ENOMEM;
            goto error;
        }
        
        device_initialize(dev);
        dev->devt = devt;
        dev->class = class;
        dev->parent = parent;
        dev->groups = groups;
        dev->release = device_create_release;
        dev_set_drvdata(dev, drvdata);

        retval = kobject_set_name_vargs(&dev->kobj, fmt, args);
        if (retval)
            goto error;
        
        retval = device_add(dev);
        if (retval)
            goto error;
        
        return dev;
    error:
        put_device(dev);
        return ERR_PTR(retval);
    }

    static void device_create_release(struct device *dev) {
        pr_debug("device: '%s': %s\n", dev_name(dev), __func__);
        kfree(dev);
    }
    ```
  2. `device_register()`
    ```c
    // version: 4.1.15
    // drivers/base/core.c
    /**
     * device_register - register a device with the system.
     * @dev: pointer to the device structure
     * 
     * This happens in two clean steps - initialize the device and
     * add it to the system. The two steps can be called separately,
     * but this is the easiest and most common. I.e. you should only
     * call the two helpers separately if have a clearly defined need
     * to use and refcount the device before it is added to the hierarchy.
     *
     * For more information, see the kerneldoc for device_initialize()
     * and device_add().
     * NOTE: _Never_ directly free @dev after calling this function,
     * even if it returned an error! Always use put_device() to give
     * up the reference initialized in this function instead.
     */
    int device_register(struct device *dev) {
        device_initialize(dev);
        return device_add(dev);
    }
    ```

- `device_initialize()`
    ```c
    // version: 4.1.15
    // drivers/base/core.c
    /**
     * device_initialize - init device structure.
     * @dev: device.
     *
     * This prepares the device for use by other layers by initializing
     * its fields. It is the first half of device_register(), if called
     * by that function, though it can also be called separately, so
     * one may use @dev's fields. In particular, get_device()/put_device()
     * may be used for reference counting of @dev after calling this
     * function.
     *
     * All fields in @dev must be initialized by the caller to 0, except
     * for those explicitly set to some other value. The simplest
     * approach is to use kzalloc() to allocate the structure containing @dev.
     * NOTE: Use put_device() to give up your reference instead of
     * freeing @dev directly once you have called this function.
     */
    void device_initialize(struct device *dev) {
        dev->kobj.kset = devices_kset;
        kobject_init(&dev->kobj, &device_ktype);
        INIT_LIST_HEAD(&dev->dma_pools);
        mutex_init(&dev->mutex);
        lockdep_set_novalidate_class(&dev->mutex);
        spin_lock_init(&dev->devres_lock);
        INIT_LIST_HEAD(&dev->devres_head);
        device_pm_init(dev);
        set_dev_node(dev, -1);
    }
    ```
    - *device* 初始化涉及多个方面：*kobject*、*dma*、*mutex*、*devres*、*pm* 以及 *numa*。
    - *kobject*：
      - 设置 *device* 对应 *kobject* 的 *kset* 和 *kobj_type*。
      - 设置 *kset* 为 `devices_kset`（提供 *uevent* 管理）：
        ```c
        // version: 4.1.15
        // drivers/base/init.c
        /**
         * driver_init - initialize driver model. 
         * Call the driver model init functions to initialize their
         * subsystems. Called early from init/main.c
         */
        void __init driver_init(void) {
            /* These are the core pieces */
            devtmpfs_init();
            devices_init();
            buses_init();
            classes_init();
            firmware_init();
            hypervisor_init();

            /* These are also core pieces, but most come after the
             * core core pieces.
             */
            platform_bus_init();
            cpu_dev_init();
            memory_dev_init();
            container_dev_init();
            of_core_init();
        }

        // drivers/base/core.c
        int __init devices_init(void) {
            devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL);
            if (!device_kset)
                return -ENOMEM;
            dev_kobj = kobject_create_and_add("dev", NULL);
            if (!dev_kobj)
                goto dev_kobj_err;
            sysfs_dev_block_kboj = kobject_create_and_add("block", dev_kobj);
            if (!sysfs_dev_block_kobj)
                goto block_kobj_err;
            sysfs_dev_char_kobj = kobject_create_and_add("char", dev_kobj);
            if (!sysfs_dev_char_kobj)
                goto char_kobj_err;
            return 0;
        char_kobj_err:
            kobject_put(sysfs_dev_block_kobj);
        block_kobj_err:
            kobject_put(dev_kobj);
        dev_kobj_err:
            kset_unregister(devices_kset);
            return -ENOMEM;
        }

        // drivers/base/core.c
        static int dev_uevent_filter(struct kset *kset, struct kobject *kobj) {
            struct kobj_type *ktype = get_ktype(kobj);
            if (ktyep == &device_ktype) {
                struct device *dev = kobj_to_dev(kobj);
                if (dev->bus)
                    return 1;
                if (dev->class)
                    return 1;
            }
            return 0;
        }

        static const char *dev_uevent_name(struct kset *kset, struct kobject *kobj) {
            struct device *dev = kobj_to_dev(kobj);
            if (dev->bus)
                return dev->bus->name;
            if (dev->class)
                return dev->class->name;
            return NULL;
        }

        static int dev_uevent(struct kset *kset, struct kobject *kobj, struct kobj_uevent_env *env) {
            struct device *dev = kobj_to_dev(kobj);
            int retval = 0;

            /* add device node properties if present */
            if (MAJOR(dev->devt)) {
                // ...
            }
            /* have the bus specific function add its stuff */
            if (dev->bus && dev->bus->uevent) {
                retval = dev->bus->uevent(dev, env);
                if (retval) {
                    // error handling
                }
            }
            /* have the class specific function add its stuff */
            if (dev->class && dev->class->uevent) {
                retval = dev->class->uevent(dev, env);
                if (retval) {
                    // error handing
                }
            }
            /* have device type specific function add its stuff */
            if (dev->type && dev->type->uevent) {
                retval = dev->type->uevent(dev, env);
                if (retval) {
                    // error handling
                }
            }
            return retval;
        }

        static const struct kset_uevent_ops device_uevent_ops = {
            .filter = dev_uevent_filter,
            .name   = dev_uevent_name,
            .uevent = dev_uevent,
        };
        ```
        - Linux 启动时调用 `driver_init()`；
        - 在启动时通过 `devices_init()` 完成 `devices_kset` 的创建，根据提供给 `kset_create_and_add()` 函数的参数可知，该 *kset* 对应到的入口目录为 `/sys/devices`、*uevent_ops* 为 `device_uevent_ops`。
        - `devices_init()` 完成的其他工作：
          - 创建入口目录为 `/sys/dev` 的 kobject；
          - 进一步创建入口目录为 `/sys/dev/block` 的 kobject；
          - 进一步创建入口目录为 `/sys/dev/char` 的 kobject；
        - `filter()` 过滤掉 *ktype* 不是 `device_ktype` 的 *kobj*、或者 *kobj* 对应的 *device* 未指定 *bus* 和 *class*；
        - `name()` 优先返回 `dev->bus` 的名字，其次是 `dev->class` 的名字，如果两者都未指定，则返回空；
        - `uevent()` 添加环境变量及调用其他环节上的 *uevent* 操作：
          - 当 `dev` 指定了主设备号（主设备号不为 0）时，添加的环境变量有：*MAJOR*、*MINOR*、*DEVNAME*（获取时与 `dev->type->devnode()` 和 `dev->class->devnode()` 这两个回调有关）、根据实际情况还有添加 *DEVMODE*（`umode_t`）、*DEVUID*、*DEVGID* 环境变量。
          - 其他环境变量：*DEVTYPE*(`dev->type->name`)、*DRIVER*(`dev->driver->name`)。
          - *OF* 相关的环境变量：*OF_NAME*(`dev->of_node-name`)、*OF_FULLNAME*(`dev->of_node->full_name`)、*OF_TYPE*(`dev->of_node->type`)、*OF_COMPATIBLE*(*compatible* 属性值，因为 *compatible* 可能有多个，因此实际的环境变量形式为 `OF_COMPATIBLE_%d`)、*OF_COMPATIBLE_N*(*compatible* 个数)、*OF_ALIAS_%d*(设备树的 *alias* 信息)
          - 其他环节上的 *uevent* 操作：`dev->bus->uevent()`、`dev->class->uevent()`、`dev->type->uevent()`；
      - 设置 *kobj_type* 为 `device_ktype`（提供 *sysfs* 操作集和 `release()` 回调）：
        ```c
        // version: 4.1.15
        // drivers/base/core.c

        /** 
         * device_release - free device structure. 
         * @kobj: device's kobj
         *
         * This is called once the reference count for the object
         * reaches 0. We forward the call to the device's release
         * method, which should handle actually freeing the structure.
         */
        static void device_release(struct kobject *kobj) {
            struct device *dev = kobj_to_dev(kobj);
            struct device_private *p = dev->p;


            /* Some platform devices are driven without driver attached
             * and managed resources may have been acquired. Make sure
             * all resources are released.
             * Drivers still can add resources into device after device
             * is deleted but alive, so release devres here to avoid possible memory leak.
             */
            devres_release_all(dev);
            if (dev->release)
                dev->release(dev);
            else if (dev->type && dev->type->release)
                dev->type->release(dev);
            else if (dev->class && dev->class->dev_release)
                dev->class->dev_release(dev);
            else
                WARN(1, KERN_ERR "Device '%s' does not have a release() "
                "function, it is broken and must be fixed.\n", dev_name(dev));
            kfree(p);
        }

        static const void *device_namespace(struct kobject *kobj) {
            struct device *dev = kobj_to_dev(kobj);
            const void *ns = NULL;
            if (dev->class && dev->class_ns_type)
                ns = dev->class->namespace(dev);
            return ns;
        }

        static ssize_t dev_attr_show(struct kobject *kobj, struct attribute *attr, char *buf) {
            struct device_attribute *dev_attr = to_dev_attr(attr);
            struct device *dev = kobj_to_dev(kobj);
            ssize_t ret = -EIO;
            if (dev_attr->show)
                ret = dev_attr->show(dev, dev_attr, buf);
            if (ret >= (ssize_t)PAGE_SIZE) {
                print_symbol("dev_attr_show: %s returned bad count\n", (unsigned long)dev_attr->show);
            }
            return ret;
        }

        static ssize_t dev_attr_store(struct kobject *kobj, struct attribute *attr, const char *buf, size_t count) {
            struct device_attribute *dev_attr = to_dev_attr(attr);
            struct device *dev = kobj_to_dev(kobj);
            ssize_t ret = -EIO;
            if (dev_attr->store)
                ret = dev_attr->store(dev, dev_attr, buf, count);
            return ret;
        }

        static const struct sysfs_ops dev_sysfs_ops = {
            .show = dev_attr_show,
            .store = dev_attr_show,
        }

        static struct kobj_type device_ktype = {
            .release = device_release,
            .sysfs_ops = &dev_sysfs_ops,
            .namespace = device_namespace,
        };
        ```
        - [ ] namepsace 是什么？

- `device_add()`
    ```c
    // version: 4.1.15
    // drivers/base/core.c
    /** 
     * device_add - add device to device hierarchy. 
     * @dev: device.
     *
     * This is part 2 of device_register(), though may be called
     * separately __iff__ device_initialize() has been called separately.
     *
     * This adds @dev to the kobject hierarchy via kobject_add(), adds
     * it to the global and sibling lists for the device, then adds it
     * to the other relevant subsystems of the driver model.
     *
     * Do not call this routine or device_register() more than once for
     * any device structure. The driver model core is not designed to
     * work with devices that get unregistered and then spring back to life (恢复生机、复苏).
     * (Among other things (此外), it's very hard to guarantee that all references
     * to the previous incarnation (化身，体现) of @dev have been dropped.)
     * Allocate and register a fresh new struct device instead.
     *
     * NOTE: _Never_ directly free @dev after calling this function,
     * even if it returned an error! Always use put_device() to give up
     * your reference instead.
     */
    int device_add(struct device *dev) {
        struct device *parent = NULL;
        struct kobject *kobj;
        struct class_interface *class_intf;
        int error = -EINVAL;

        dev = get_device(dev);
        if (!dev)
            goto done;
        
        if (!dev->p) {
            error = device_private_init(dev);
            if (error)
                goto done;
        }

        /* for statically allocated devices, which should all be
         * converted some day, we need to initialize the name.
         * We prevent reading back the name, and force the use
         * of dev_name().
         */
        if (dev->init_name) {
            dev_set_name(dev, "%s", dev->init_name);
            dev->init_name = NULL;
        }
        /* subsystems can specify simple device enumeration */
        if (!dev_name(dev) && dev->bus && dev->bus->dev_name)
            dev_set_name(dev, "%s%u", dev->bus->dev_name, dev->id);
        if (!dev_name(dev)) {
            error = -EINVAL;
            goto name_error;
        }

        pr_debug("device: '%s': %s\n", dev_name(dev), __func__);

        parent = get_device(dev->parent);
        kobj = get_device_parent(dev, parent);
        if (kobj)
            dev->kobj.parent = kobj;

        /* use parent numa_node */
        if (parent)
            set_dev_node(dev, dev_to_node(parent));
        
        /* first, register with generic layer 
         * we require the name to be set before, and pass NULL
         */
        error = kobject_add(&dev->kobj, dev->kobj.parent, NULL);
        if (error)
            goto Error;
        
        /* notify platform of device entry */
        if (platform_notify)
            platform_notify(dev);
        
        error = device_create_file(dev, &dev_attr_uevent);
        if (error)
            goto attrError;
        error = device_add_class_symlinks(dev);
        if (error)
            goto SymlinkError;
        error = device_add_attrs(dev);
        if (error)
            goto AttrsError;
        error = bus_add_device(dev);
        if (error)
            goto BusError;
        error = dpm_sysfs_add(dev);
        if (error)
            goto DPMError;
        device_pm_add(dev);

        if (MAJOR(dev->devt)) {
            error = device_create_file(dev, &dev_attr_dev);
            if (error)
                goto DevAttrError;
            error = device_create_sys_dev_entry(dev);   // /sys/dev/char/ (默认) 下名为 主设备号:此设备号的链接，指向当前设备 kobj
            if (error)
                goto SysEntryError;
            devtmpfs_create_node(dev);
        }
        
        /* Notify clients of device addition. This call must come
         * after dpm_sysfs_add() and before kobject_uevent().
         */
        if (dev->bus)
            blocking_notifier_call_chain(&dev->bus->p->bus_notifier, BUS_NOTIFY_ADD_DEVICE, dev);
        
        kobject_uevent(&dev->kobj, KOBJ_ADD);
        bus_probe_deivce(dev);
        if (parent)
            klist_add_tail(&dev->p->knode_parent, &parent->p->klist_children);
        if (dev->class) {
            mutex_lock(&dev->class->p->mutex);
            /* tie the class to the device */
            klist_add_tail(&dev->knode_class, &dev->class->p->klist_devices);
            /* notify any interfaces that the device is here */
            list_for_each_entry(class_intf, &dev->class->p->interfaces, node)
                if (class_intf->add_dev)
                    class_intf->add_dev(dev, class_intf);
            mutex_unlock(&dev->class->p->mutex);
        }
    done:
        put_device(dev);
        return error;
    SysEntryError:
        if (MAJOR(dev->devt))
            device_remove_file(dev, &dev_attr_dev);
    DevAttrError:
        device_pm_remvoe(dev);
        dpm_sysfs_remvoe(dev);
    DPMError:
        bus_remove_device(dev);
    BusError:
        device_remvoe_attrs(dev);
    AttrsError:
        device_remove_class_symlinks(dev);
    SymlinkError:
        device_remove_file(dev, &dev_attr_uevent);
    attrError:
        kobject_uevent(&dev->kobj, KOBJ_REMOVE);
        kobject_del(&dev->kobj);
    Error:
        cleanup_device_parent(dev);
        put_device(parent);
    name_error:
        kfree(dev->p);
        dev->p = NULL;
        goto done;
    }
    EXPORT_SYMBOL_GPL(device_add);

    static struct kobject *get_device_parent(struct device *dev, struct device *parent) {
        if (dev->class) {
            struct kobject *kobj = NULL;
            struct kobject *parent_kobj;
            struct kobject *k;
    #define CONFIG_BLOCK
            /* block disks show up in /sys/block */
            if (sysfs_deprecated && dev->class == &block_class) {
                if (parent && parent->class == &block_class)
                    return &parent->kobj;
                return &block_class.p->subsys.kobj;
            }
    #endif
            /* If we have no parent, we live in "virtual". 
             * Class-devices with a non class-device as parent,
             * live in a "glue" directory to prevent namespace collisions
             */
            if (parent == NULL)
                parent_kobj = virtual_device_parent(dev);
            else if (parent->class && !dev->class->ns_type)
                return &parent->kobj;
            else
                parent_kobj = &parent->kobj;
            
            mutex_lock(&gdp_mutex);
            /* find our class-directory at the parent and reference it */
            spin_lock(&dev->class->p->glue_dirs.list_lock);
            list_for_each_entry(k, &dev->class->p->glue_dirs.list, entry)
                if (k->parent == parent_kobj) {
                    kobj = kobject_get(k);
                    break;
                }
            spin_unlock(&dev->class->p->glue_dirs.list_lock);
            if (kobj) {
                mutex_unlock(&gdp_mutex);
                return kobj;
            }
            /* or create a new class-directory at the parent device */
            k = class_dir_create_add_add(dev->class, parent_kobj);
            /* do not emit an uevent for this simple "glue" directory */
            mutex_unlock(&gdp_mutex);
            return k;
        }

        /* subsystems can specify a default root directory for their devices */
        if (!parent && dev->bus && dev->bus->dev_root)
            return &dev->bus->dev_root->kobj;
        if (parent)
            return &parent->kobj;
        return NULL;
    }

    struct kobject *virtual_device_parent(struct device *dev) {
        static struct kobject *virtual_dir = NULL;
        if (!virtual_dir)
            virtual_dir = kobject_create_and_add("virtual", &devices_kset->kobj);
        return virtual_dir;
    }
    ```
    - 调用链 + 功能：
    ```c
    device_add(dev)
        get_device(dev)
        if (!dev->p) device_private_init(dev)   // 分配、设置、初始化 device_private
        设置设备对应 kobject 名字
            1. 根据 dev->init_name（不为空）设置，设置后给该字段置空
            2. 否则，按照 "%s%u" - dev->bus->dev_name + dev->id 组合得到设备名字
            3. 如果最后仍然未获得设备名字，则返回错误
        get_device(dev->parent)
        设置 dev->kobject.parent
            1. 如果 get_devcie_parent(dev, dev->parent) 找到非空的 kobject，则将其设置为 dev->kobject.parent
            2. 否则，dev->kobject.parent 保持不变
        if (dev->parent) set_dev_node(dev, dev_to_node(dev->parent));    // use parent numa_node
        kobject_add(&dev->kobj, dev->kobj.parent, NULL) // 注册 dev->kobj
        if (platform_notify) platform_notify(dev)
        device_create_file(dev, &dev_attr_uevent)   // 设备的 uevent 文件；
                                                    // 读-获取 uevent 环境变量；写-产生 uevent 事件
        device_add_class_symlinks(dev)
            1. 在设备目录下，创建名为 of_node 的链接，指向此设备的 of_node 对象对应的目录
            2. 如果设备未被归类（dev->class 为空），返回
            3. 在设备目录下，创建名为 subsystem 的链接，指向此设备的种类的对应目录
            4. （一般还会）在设备目录下，创建名为 device 的链接，指向此设备的父设备（dev->parent->kobj）的对应目录
            5. 在此设备的种类的对应目录下，创建与设备同名的链接，指向该设备的目录
        device_add_attrs(dev)
            1. 创建属性集文件，属性集包括：
                a. dev->class->dev_groups
                b. dev->type->dev_groups
                c. dev->groups
            2. 如果设备支持且允许 offline/online，则在设备目录下创建 online 文件（可读写）
        bus_add_device(dev) // 触发总线的 add 机制，详情见对应部分
        dpm_sysfs_add(dev)
            1. 创建属性集，属性集名字为 power
            2. 在该属性集目录下，可能会创建如下属性文件：
                a. runtime_status, control, runtime_suspended_time, runtime_active_time, autosuspend_delay_ms
                b. wakeup, wakeup_count, wakeup_active_count, wakeup_abort_count,...
                c. pm_qos_latency_tolerance_us
        device_pm_add(dev)  // 将设备添加到 dpm_list 列表上
        if (MAJOR(dev->devt)):
            1. 在设备目录下，创建名为 dev 属性文件。读-获取设备号
            2. 如果 dev->class 被设置，则在该 class 对应的目录下，
               否则默认是字符设备，则在 /sys/dev/char 目录下：
               创建名为 “主设备号:此设备号” 的链接文件，指向设备目录
            3. 通过 devtmpfs_create_node(dev)，在 /dev 目录下创建设备结点文件
               文件名字的来源：dev->type->devnode()、dev->class->devnode() 或者 dev_name(dev)
        if (dev->bus)
            blocking_notifier_call_chain() // 触发总线的 notify 机制，dev + BUS_NOTIFY_ADD_DEVICE
        kobject_uevent(&dev->kobj, KOBJ_ADD)    // 向用户空间发送 uevent 事件，类型为 KOBJ_ADD
                                                // 会触发总线的 uevent 机制
        bus_probe_device(dev)   // 触发总线的 match + probe 机制
        if (dev->parent) // 将设备挂到父对象的 children 链表上
            klist_add_tail(&dev->p->knode_parent, &parent->p->klist_children)
        if (dev->class) // 将设备挂到其对应种类管理的设备链表上并且调用 interfaces 链表上的 add_dev 回调
            1. klist_add_tail(&dev->knode_class, &dev->class->p->klist_devices)
            2. class_intf->add_dev(dev, class_intf)
        put_device(dev)
    ```

- 设备注册结果：
  1. 初始化 `dev->kobject` 和 `dev` 相关字段。将 `dev->kobject` 的 *kset* 设置为 `devices_kset`，*kobj_type* 设置为 `device_ktype`。
  2. 注册 `dev->kobject`：设置名字、构建与其父 *kobject* 间的层级关系、创建 *sysfs* 入口目录。
  3. 在 `dev->kobject` 对应的目录下创建属性（集）文件，这些文件的来源：设备的总线、设备的种类、设备的类型、设备自身、默认属性等。
  4. 创建设备目录与设备所在子系统（总线或种类）、父对象、设备树结点等之间的链接文件（两个方向都创建）。
  5. 通过 *device_private* 建立与总线/种类、父设备对象、驱动间的联系。
  6. 根据主设备号的情况，可能创建 `/dev` 下的设备结点文件。

#### 设备注销
- 设备注销的两种方式
  1. `device_destroy()`
    ```c
    // version: 4.1.15
    // drivers/base/core.c 
    /**
     * device_destroy - removes a device that was created with device_create() 
     * @class: pointer to the struct class that this device was registered with
     * @devt: the dev_t of the device that was previously registered
     * 
     * This call unregisters and cleans up a device that was created with a
     * call to device_create()
     */
    void device_destroy(struct class *class, dev_t devt) {
        struct device *dev;

        dev = class_find_device(class, NULL, &devt, __match_devt);
        if (dev) {
            put_device(dev);
            device_unregister(dev);
        }
    }
    EXPORT_SYMBOL_GPL(device_destroy);
    ```
    - *class* 与 *bus* 类似，通过 *subsys_private* 管理属于它们的设备。这里通过 `class_find_device()` 根据传入的数据（设备号 `devt`）从第一个设备开始（`NULL`）基于匹配函数 `__match_devt()` 查找此 *class* (`class`) 上的目标设备
        ```c
        // version: 4.1.15
        // drivers/base/core.c
        static int __match_devt(struct device *dev, const void *data) {
            const dev_t *devt = data;
            return dev->devt == *devt;
        }
        ```
    - 通过 `put_device()` 减少 *device* 的引用计数（在 `class_find_device()` 中增加了对 *device* 的引用）
    - 通过 `device_unregister()` 从设备所属的子系统中移除设备，详情见下述内容
  2. `device_unregister()`
    ```c
    // version: 4.1.15
    // drivers/base/core.c
    /**
     * device_unregister - unregister device from system. 
     * @dev: device going away.
     *
     * We do this in two parts, like we do device_register().
     * First, we remove it from all the subsystems with device_del(),
     * then we decrement the reference count via put_device().
     * If that is the final reference count, the device will be
     * cleaned up via device_release() above. Otherwise, the structure
     * will stick around (逗留，停留) until the final reference to the device
     * is dropped.
     */
    void device_unregister(struct device *dev) {
        pr_debug("device: '%s': %s\n", dev_name(dev), __func__);
        device_del(dev);
        put_device(dev);
    }
    EXPORT_SYMBOL_GPL(device_unregister);

    /** 
     * device_del - delete device from system. 
     * @dev: device.
     * 
     * This is the first part of the device unregistration
     * sequence. This removes the device from the lists we
     * control from here, has it removed from the other driver
     * model subsystems it was added to in device_add, and removes
     * it from the kobject hierachy.
     *
     * NOTE: this should be called manually _iff_ device_add() was
     * also called manually.
     */
    void device_del(struct device *dev) {
        struct device *parent = dev->parent;
        struct class_interface *class_intf;

        /* Notify clients of device removal. Thi call must come
         * before dpm_sysfs_remove().
         */
        if (dev->bus)
            blocking_notifier_call_chain(&dev->bus->p->bus_notifier, BUS_NOTIFY_DEL_DEVICE, dev);
        dpm_sysfs_remove(dev);
        if (parent)
            klist_del(&dev->p->knode_parent);
        if (MAJOR(dev->devt)) {
            devtmpfs_delete_node(dev);
            device_remove_sys_dev_entry(dev);
            device_remove_file(dev, &dev_attr_dev);
        }
        if (dev->class) {
            device_remove_class_symlinks(dev);
            mutex_lock(&dev->class->p->mutex);
            /* notify any interfaces that device is now gone */
            list_for_each_entry(class_intf, &dev->class->p->interfaces, node)
                if (class_intf->remove_dev)
                    class_intf->remove_dev(dev, class_intf);
            /* remove the device from the class list */
            klist_del(&dev->knode_class);
            mutex_unlock(&dev->class->p->mutex);
        }
        device_remove_file(dev, &dev_attr_uevent);
        device_remove_attrs(dev);
        bus_remove_device(dev);
        device_pm_remove(dev);
        driver_deferred_probe_del(dev);

        /* Notify the platform of the removal, in case they 
         * need to do anything...
         */
        if (platform_notify_remove)
            platform_notify_remove(dev);
        if (dev->bus)
            blocking_notifier_call_chain(&dev->bus->p->bus_notifier, BUS_NOTIFY_REMOVED_DEVICE, dev);
        kobject_uevent(&dev->kojb, KOBJ_REMOVE);
        cleanup_device_parent(dev);
        kobject_dev(&dev->kobj);
        put_device(parent);
    }
    EXPORT_SYMBOL_GPL(device_del);
    ```

#### 相关 API
1. `device_get_devnode()`
    ```c
    // version: 4.1.15
    // drivers/base/core.c
    /**
     * device_get_devnode - path of device node file 
     * @dev: device
     * @mode: returned file access mode
     * @uid: returned file owner
     * @gid: returned file group
     * @tmp: possibly allocated string
     *
     * Return the relative path of a possible device node.
     * Non-default names may need to allocate a memory to compose
     * a name. This memory is returned in tmp and needs to
     * be freed by the caller.
     */
    const char *device_get_devnode(struct device *dev, umode_t *mode, kuid_t *uid, kgid_t *gid, const char **tmp) {
        char *s;
        *tmp = NULL;
        /* the device type may provide a specific name */
        if (dev->type && dev->type->devnode)
            *tmp = dev->type->devnode(dev, mode, uid, gid);
        if (*tmp)
            return *tmp;
        /* the class may provide a specific name */
        if (dev->class && dev->class->devnode)
            *tmp = dev->class->devnode(dev, mode);
        if (*tmp)
            return *tmp;
        /* return name without allocation, tmp == NULL */
        if (strchr(dev_name(dev), '!') == NULL)
            return dev_name(dev);
        /* replace '!' in the name with '/' */
        *tmp = kstrdup(dev_name(dev), GFP_KERNEL);
        if (!*tmp)
            return NULL;
        while ((s = strchr(*tmp, '!')))
            s[0] = '/';
        return *tmp;
    }
    ```

2. `dev_driver_string(), drivers/base/core.c`

### 驱动
#### 概述
`strut device_driver` 为驱动基类，为设备提供属性（*attr*）和功能（各个回调函数）。

#### 数据结构描述
```c
// version: 4.1.15
// include/linux/device.h
/** 
 * struct device_driver - The basic device driver structure 
 * @name: Name of the device driver.
 * @bus: The bus which the device of this driver belongs to.
 * @owner: The module owner.
 * @mod_name: Used for built-in modules.
 * @suppress_bind_attrs: Disables bind/unbind via sysfs.
 * @of_match_table: The open firmware table.
 * @acpi_match_table: The ACPI match table.
 * @probe: Called to query the existence of a specific device,
 *    whether this driver can work with it, and bind the driver
 *    to a specific device.
 * @remvoe: Called when the device is removed from the system to
 *    unbind a device from this driver.
 * @shutdown: Called at shut-down time to quiesce the device.
 * @suspend: Called to put the device to sleep mode. Usually to
 *    low power state.
 * @rusume: Called to bring a device from sleep mode.
 * @groups: Default attributes that get created by the driver core
 *    automatically.
 * @pm: Power management operations of the device which match this
 *    driver.
 * @p: Driver core's private data, no one other than the driver core
 *    can touch this.
 * 
 * The device driver-model tracks all of the drivers known to the
 * system. The main reason for this tracking is to enable the driver
 * core to match up drivers with new devices. Once drivers are known
 * objects within the system, however, a number of other things
 * become possiblee. Device drivers can export information and 
 * configuration variables that are independent of any specific device.
 */
struct device_driver {
   const char *name;
   struct bus_type *bus;

   struct module *owner;
   const char *mod_name;         // used for built-in modules

   bool suppress_bind_attrs;

   const struct of_device_id *of_match_table;
   const struct acpi_device_id *acpi_match_table;

   int (*probe)(struct device *dev);
   int (*remove)(struct device *dev);
   void (*shutdown)(struct device *dev);
   int (*suspend)(struct device *dev, pm_message_t state);
   int (*resume)(struct device *dev);
   const struct attribute_group **groups;

   const struct dev_pm_ops *pm;

   struct driver_private *p;
};
```

#### 驱动注册
```c
// version: 4.1.15
// include/linux/device.h
extern int __must_check driver_register(struct device_driver *drv);
extern void driver_unregister(struct device_driver *drv);
```

```c
// version: 4.1.15
// driver/base/driver.c
/**
 * driver_register - register driver with bus 
 * @drv: driver to register
 * 
 * We pass off most of the work to the bus_add_driver call,
 * since most of the things we have to to deal with the bus
 * structures.
 */
int driver_register(struct device_driver *drv) {
    int ret;
    struct device_driver *other;
    BUG_ON(!drv->bus->p);
    if ((drv->bus->probe && drv->probe) ||
        (drv->bus->remove && drv->remove) ||
        (drv->bus->shutdown && drv->shutdown))
        printk(KERN_WARNING "Driver '%s' needs updating - please
            use bus_type methods\n", drv->name);
    other = driver_find(drv->name, drv->bus);
    if (other) {
        printk(KERN_ERR "Error: Driver '%s' is already registered,"
         " aborting...\n", drv->name);
        return -EBUSY;
    }
    ret = bus_add_driver(drv);
    if (ret)
        return ret;
    ret = driver_add_groups(drv, drv->groups);
    if (ret) {
        bus_remove_driver(drv);
        return ret;
    }
    kobject_uevent(&drv->p->kobj, KOBJ_ADD);
    return ret;
}
EXPORT_SYMBOL_GPL(driver_register);

/**
 * driver_find - locate driver on a bus by its name 
 * @name: name of the driver
 * @bus: bus to scan for the driver
 *
 * Call kset_find_obj() to iterate over list of drivers on a bus
 * to find driver by name. Return driver if found.
 *
 * This routine provides no locking to prevent the driver it returns
 * from being unregistered or unloaded while the caller is using it.
 * The caller is responsible for preventing this.
 */
struct device_driver *driver_find(const char *name, struct bus_type *bus) {
    struct kobject *k = kset_find_obj(bus->p->drivers_kset, name);
    struct driver_private *priv;
    if (k) {
        /* Drop reference added by kset_find_obj() */
        kobject_put(k);
        priv = to_driver(k);
        return priv->driver;
    }
    return NULL;
}
EXPORT_SYMBOL_GPL(driver_find);
```
- 调用链 + 功能
```c
1. BUG_ON(...) // 要求驱动挂在有效的总线上
2. if ((drv->bus->probe && drv->probe) || ...) 
    // 检查总线和驱动在 probe, remove 和 shutdown 三个回调上是否有冲突
    // 优先使用总线的相关方法，因此当总线和驱动两者都提供上述方法是会输出 WARNING 日志
3. driver_find() // 在总线管理的驱动集合中查找当前驱动，检查其是否已注册
4. bus_add_driver() // bus 的 add 机制-驱动，见相关内容
5. driver_add_groups()  // 在驱动目录下添加自身提供的属性集文件/目录
6. kobject_uevent() // 发送 uevent 事件，类型为 KOBJ_ADD，对象为驱动对应 kobj
```

#### 驱动注销
```c
// version: 4.1.15
// drivers/base/driver.c
/** driver_unregister - remove driver from system. 
 * @drv: driver.
 *
 * Again, we pass off most of the work to the bus-level call.
 */
void driver_unregister(struct device_driver *drv) {
    if (!drv || !drv->p) {
        WARN(1, "Unexpected driver unregister!\n");
        return;
    }
    driver_remove_groups(drv, drv->groups);
    bus_remove_driver(drv);
}
EXPORT_SYMBOL_GPL(driver_unregister);
```

### 设备与类型
#### 概述
设备层面对设备进行分类。

在 `device_release()` 中，若 *device* 并未设置 *release* 回调，则 `device_type->release()` 将作为第二顺位考虑。

#### 数据结构描述
```c
// version: 4.1.15
// include/linux/device.h
/* 
 * The type of device, "struct device" is embedded in. A class or
 * bus can contain devices of different types
 * like "partitions" and "disks", "mouse" and "event".
 * This identifies the device type and carries type-specific
 * information, equivalent to the kobj_type of a kobject.
 * If "name" is specified, the uevent will contain it in the
 * DEVTYPE variable.
 */
struct device_type {
    const char *name;
    const struct attribute_group **groups;
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
    char *(*devnode)(struct device *dev, umode_t *mode, kuid_t *uid, 
        kgid_t *gid);
    void (*release)(struct device *dev);
    const struct dev_pm_ops *pm;
}
```

### 设备与种类
#### 概述
`struct class` 是从设备的功能等方面对其进行归类处理，例如 *input* 类设备。

内核层面对对象进行分类。

在 `device_release()` 中，若 *device* 并未设置 *release* 回调，则 `class->dev_release()` 将作为第三顺位考虑。

从总线上设备的注册源码猜测（设备目录下创建的 *subsystem* 文件要么对应到 *class*，要么对应到 *bus*）：设备应该在一般情况下只会通过一个子系统管理，要么 *bus*、要么 *class*。另外，从设备注册的两种方式也反映了这一点：`bus_create()` 将 *device* 交由 *class* 管理，而 `bus_register()` 将 *device* 交由 *bus* 管理。

#### 数据结构描述
```c
// version: 4.1.15
// include/linux/device.h
/**
 * struct class - device classes 
 * @name: Name of the class.
 * @owner: The module owner
 * @class_attrs: Default attributes of this class.
 * @dev_groups: Default attributes of the devices that belong
 *  to the class.
 * @dev_kobj: The kobject that represents this class and links
 *  it into hierarchy.
 * @dev_uevent: Called when a device is added, removed from
 *  this class, or a few other things that generate uevents to
 *  add the environment variables.
 * @devnode: Callback to provide the devtmpfs.
 * @class_release: Called to release this class.
 * @dev_release: Called to release the device.
 * @suspend: Used to put the device to sleep mode, usually to a
 *  low power state.
 * @resume: Used to bring the device from the sleep mode.
 * @ns_type: Callbacks so sysfs can determine namespaces.
 * @namespace: Namespace of the device belongs to this class.
 * @pm: The default device power management operations of this class.
 * @p: The private data of the driver core, no one other than the
 *  driver core can touch this.
 *
 * A class is a higher-level view of a device that abstracts out
 * low-level implementation details. Drivers may see a SCSI disk
 * or an ATA disk, but, at the class level, they ara all simply
 * disk. Classes allow user space to work with devices based on
 * what they do, rather than how they are connected or how they
 * work.
 */
struct class {
    const char *name;
    struct module *owner;
    struct class_attribute *class_attrs;
    const struct attribute_group **dev_groups;
    struct kobject *dev_kobj;
    int (*dev_uevent)(struct device *dev, struct kobj_uevent_env *env);
    char *(*devnode)(struct device *dev, umode_t *mode);
    void (*class_release)(struct class *class);
    void (*dev_release)(struct device *dev);
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);
    const struct kobj_ns_type_operations *ns_type;
    const void *(*namespace)(struct device *dev);
    const struct dev_pm_ops *pm;
    struct subsys_private *p;
}
```

#### 种类注册
```c
// version: 4.1.15
// include/linux/device.h
/* This is a #define to keep the compiler from merging 
    * different instances of the __key variable
    */
#define class_register(class) \
({ \
    static struct lock_class_key __key; \
    __class_register(class, &__key);    \
})

// drivers/base/class.c
int __class_register(struct class *cls, struct lock_class_key *key) {
    struct subsys_private *cp;
    int error;
    pr_debug("device class '%s': registering\n", cls->name);
    cp = kzalloc(sizeof(*cp), GFP_KERNEL);
    if (!cp)
        return -ENOMEN;
    klist_init(&cp->klist_devices, klist_class_dev_get, klist_class_dev_put);
    INIT_LIST_HEAD(&cp->interfaces);
    kset_init(&cp->glue_dirs);
    __mutex_init(&cp->mutex, "subsys mutex", key);
    error = kobject_set_name(&cp->subsys.kobj, "%s", cls->name);
    if (error) {
        kfree(cp);
        return error;
    }
    /* set the default /sys/dev directory for devices of this class */
    if (!cls->dev_kobj)
        cls->dev_kobj = sysfs_dev_char_kobj;
#if define(CONFIG_BLOCK)
    /* let the block class directory show up in the root of sysfs */
    if (!sysfs_deprecated || cls != &block_class)
        cp->subsys.kobj.kset = class_kset;
#else
    cp->subsys.kobj.kset = class_kset;
#endif
    cp->subsys.kobj.ktype = class_ktype;
    cp->class = cls;
    cls->p = cp;

    error = kset_register(&cp->subsys);
    if (error) {
        kfree(cp);
        return error;
    }
    error = add_class_attrs(class_get(cls));
    class_put(cls);
    return error;
}
EXPORT_SYMBOL_GPL(__class_register);

static void klist_class_dev_get(struct klist_node *n) {
    struct device *dev = container_of(n, struct device, knode_class);
    get_device(dev);
}

static void klist_class_dev_put(struct klist_node *n) {
    struct device *dev = container_of(n, struct device, knode_class);
    put_device(dev);
}
```
- 创建一个种类的目录入口（在 `/sys/class` 下，`cp->subsys.kobj` 未设置 `parent` 但设置 `kset` 为 `class_kset`），名为 `cls->name`（`kobject_set_name(&cp->subsys.kobj, "%s", cls->name)` + `kset_register(&cp->subsys)`）。
- 该种类的 *devices* 目录入口，默认在 `/sys/dev/char` 下（此时 `cls->dev_kobj` 为空，因而设置 `cls->dev_kobj = sysfs_dev_char_kobj`），否则在 `cls->dev_kobj` 对应的目录下。

1. `class_kset`、`class_ktype` 和 `block_class`
    ```c
    // version: 4.1.15
    // drivers/base/class.c
    /* Hotplug events for classes go to the class subsys */
    static struct kset *class_kset;

    int __init classes_init(void) {
        class_kset = kset_create_and_add("class", NULL, NULL);
        if (!class_kset)
            return -ENOMEM;
        return 0;
    }

    static struct kobj_type class_ktype = {
        .sysfs_ops = &class_sysfs_ops,
        .release   = class_release,
        .child_ns_type = class_child_ns_type,
    };

    static const struct sysfs_ops class_sysfs_ops = {
        .show = class_attr_show,
        .store = class_attr_store,
    };

    static ssize_t class_attr_show(struct kobject *kobj, struct attribute *attr, char *buf) {
        struct class_attribute *class_attr = to_class_attr(attr);
        struct subsys_private *cp = to_subsys_private(kobj);
        ssize_t ret = -EIO;

        if (class_attr->show)
            ret = class_attr->show(cp->class, class_attr, buf);
        return ret;
    }

    static ssize_t class_attr_store(struct kobject *kobj, struct attribute *attr, const char *buf, size_t count) {
        struct class_attribute *class_attr = to_class_attr(attr);
        struct subsys_private *cp = to_subsys_private(kobj);
        ssize_t ret = -EIO;

        if (class_attr->store)
            ret = class_attr->store(cp->class, class_attr, buf, count);
        return ret;
    }

    static void class_release(struct kobject *kobj) {
        struct subsys_private *cp = to_subsys_private(kobj);
        struct class *class = cp->class;
        pr_debug("class '%s': release.\n", class->name);
        if (class->class_release)
            class->class_release(class);
        else
            pr_debug("class '%s' does not have a release() function"
            ", be careful\n", class->name);
        kfree(cp);
    }

    static const struct kobj_ns_type_operations *class_child_ns_type(struct kobject *kobj) {
        struct subsys_private *cp = to_subsys_private(kobj);
        struct class *class = cp->class;
        return class->ns_type;
    }

    // block/genhd.c
    struct class block_class = {
        .name = "block",
    };
    ```
    - `class_kset` 对应到 *sysfs* 的目录入口为 `/sys/class`。
    - `classes_init()` 在 `drivers/base/init.c - driver_init()` 中被调用，因此 *Linux* 内核启动时就会创建 `/sys/class`。

2. *helper func*
   - `class_create()`
    ```c
    // version: 4.1.15
    // include/linux/device.h
    /* This is a #define to keep the compiler from merging 
     * different instances of the __key variable
     */
    #define class_create(owner, name) \
    ({ \
        static struct lock_class_key __key; \
        __class_create(owner, name, __key); \
    })

    // drivers/base/class.c
    /**
     * class_create - create a struct class structure 
     * @owner: pointer to the module that is to "own" this struct class
     * @name: pointer to a string for the name of this class.
     * @key: the lock_class_key for this class; used by mutex lock
     *  debugging
     * 
     * This is used to create a struct class pointer that can then 
     * be used in calls to device_create().
     *
     * Returns &struct class pointer on success, or ERR_PTR() on error.
     * Note, the pointer created here is to be destroyed when finished
     * by making a call to class_destroy().
     */
    struct class* __class_create(struct module *owner, const char *name, struct lock_class_key *key) {
        struct class *cls;
        int retval;
        cls = kzalloc(sizeof(*cls), GFP_KERNEL);
        if (!cls) {
            retval = -ENOMEM;
            goto error;
        }
        cls->name = name;
        cls->owner = owner;
        cls->class_create = class_create_release;
        retval = __class_register(cls, key);
        if (retval)
            goto error;
        return cls;
    error:
        kfree(cls);
        return ERR_PTR(retval);
    }

    static void class_create_release(struct class *cls) {
        pr_debug("%s called for %s\n", __func__, cls->name);
        kfree(cls);
    }
    ```
    - `class_create()` 只是在 `__class_register()` 基础上：帮助 `struct class` 所需内存的申请、*class* 实例的 `name` 和 `owner` 字段以及 `class_create()` 回调的设置。是对 `__class_register()` 简单地封装。

#### 种类注销
```c
// version: 4.1.15
// drivers/base/class.c
void class_unregister(struct class *cls) {
    pr_debug("device class '%s': unregistering\n", cls->name);
    remove_class_attrs(cls);
    kset_unregister(&cls->p->subsys);
}
```

- *helper func*
  1. `class_destroy()`
    ```c
    // version: 4.1.15
    // drivers/base/class.c
    /* Note: the pointer to be destroyed must have been 
     * created with a call to class_create().
     */
    void class_destroy(struct class *cls) {
        if ((cls == NULL) || IS_ERR(cls))
            return;
        class_unregister(cls);
    }
    ```

#### 相关 API
1. `class_find_device()`
    ```c
    // version: 4.1.15
    // drivers/base/class.c
    /**
     * class_find_device() - device iterator for locating a particular device 
     * @class: the class we're iterating
     * @start: Device to begin with
     * @data: data for the match function
     * @match: function to check device
     *
     * This is similar to the class_for_each_dev() function above, but it returns
     * a reference to a device that is 'found' for later use, as determined by the
     * @match callback.
     * 
     * The callback should return 0 if the device doesn't match and non-zero if
     * it does. If the callback returns non-zero, that function will return to
     * the caller and not iterate over any more devices.
     * 
     * Note, you will need to drop the reference with put_device() after use.
     *
     * @fn is allowed to do anything including calling back into class code.
     * There's no locking restriction.
     */
    struct device *class_find_device(struct class *class, struct device *start,
        const void *data, int (*match)(struct device *, const void *)) {
        struct class_dev_iter iter;
        struct device *dev;

        if (!class)
            return NULL;
        if (!class->p) {
            WARN(1, "%s called for class '%s' before it was initialized", 
                __func__, class->name);
            return NULL;
        }

        class_dev_iter_init(&iter, class, start, NULL);
        while ((dev = class_dev_iter_next(&iter))) {
            if (match(dev, data)) {
                get_device(dev);
                break;
            }
        }
        class_dev_iter_exit(&iter);
        return dev;
    }
    EXPORT_SYMBOL_GPL(class_find_device);
    ```

2. 属性文件操作：
    ```c
    // version: 4.1.15
    // include/linux/device.h
    struct class_attribute {
        struct attribute attr;
        ssize_t (*show)(struct class *class, struct class_attribute *attr, char *buf);
        ssize_t (*store)(struct class *class, struct class_attribute *attr, const char *buf, size_t count);
    };

    // drivers/base/class.c
    static int add_class_attrs(struct class *cls) {
        int i;
        int error = 0;
        if (cls->class_attrs) {
            for (i = 0; cls->class_attrs[i].attr.name; i++) {
                error = class_create_file(cls, &cls->class_attrs[i]);
                if (error)
                    goto error;
            }
        }
    done:
        return error;
    error:
        while (--i >= 0)
            class_remove_file(cls, &cls->class_atts[i]);
        goto done;
    }

    static void remove_class_attrs(struct class *cls) {
        int i;
        if (cls->class_attrs) {
            for (i = 0; cls->class_attrs[i].attr.name; i++)
                class_remvoe_file(cls, &cls->class_attrs[i]);
        }
    }

    // include/linux/device.h
    static inline int __must_check class_create_file(struct class *cls, const struct class_attribute *attr) {
        return class_create_file_ns(cls, attr, NULL);
    }

    static inline void class_revmoe_file(struct class *cls, const struct class_attribute *attr) {
        return class_remove_file_ns(cls, attr, NULL);
    }

    // drivers/base/class.c
    int class_create_file_ns(struct class *cls, const struct class_attribute *attr, const void *ns) {
        int error;
        if (cls)
            error = sysfs_create_file_ns(&cls->p->subsys.kobj, &attr->attr, ns);
        else
            error = -EINVAL;
        return error;
    }

    void class_remove_file_ns(...) {
        if (cls)
            sysfs_remove_file_ns(&cls->p->subsys.kobj, &attr->attr, ns);
    }
    ```
    - 属性文件的添加和删除都是在 `cls->p->subsys.kobj` 对应的目录下。

### 驱动与模块
#### 概述
见 《深入 Linux 内核架构-第七章，模块》

#### 数据结构描述
```c
// version: 4.1.15
// include/linux/module.h
struct module {
    enum module_state state;
    /* Member of list of modules */
    struct list_head list;
    /* Unique handle for this module */
    char name[MODULE_NAME_LEN];
    /* Sysfs stuff. */
    struct module_kobject mkobj;
    struct module_attribute *modinfo_attrs;
    const char *version;
    const char *srcversion;
    struct kobject *holders_dir;
    /* Exported symbols */
    const struct kernel_symbol *syms;
    const unsigned long *crcs;
    unsigned int num_syms;
    /* Kernel parameters */
    struct kernel_param *kp;
    unsigned int num_kp;
    /* GPL-only exported symbols */
    unsigned int num_gpl_syms;
    const struct kernel_symbol *gpl_syms;
    const unsigned long *gpl_crcs;
#ifdef CONFIG_UNUSED_SYMBOLS
    /* unused exported symbols */
    const struct kernel_symbol *unused_syms;
    const unsigned long *unused_crcs;
    unsigned int num_unused_syms;
    /* GPL-only, unused exported symbols */
    unsigned int num_unused_gpl_syms;
    const struct kernel_symbol *unused_gpl_syms;
    const unsigned long *unused_gpl_crcs;
#endif
#ifdef CONFIG_MODULE_SIG
    /* Signature was verified */
    bool sig_ok;
#endif
    /* symbols that will be GPL-only in the near future */
    const struct kernel_symbol *gpl_future_syms;
    const unsigned long *gpl_future_crcs;
    unsigned int num_gpl_future_syms;
    /* Exception talbe */
    unsigned int num_exentries;
    struct exception_table_entry *extable;
    /* Startup function */
    int (*init)(void);
    /* If this is non-NULL, vfree after init() returns */
    void *module_init;
    /* Here is the actual code + data, vfree'd on unload */
    void *module_core;
    /* Here are the sizes of the init and core sections */
    unsigned int init_size, core_size;
    /* The size of the executable code in each section */
    unsigned int init_text_size, core_text_size;
    /* Size of RO sections of the module (text+rodata) */
    unsigned int init_ro_size, core_ro_size;
    /* Arch-specific module values */
    struct mod_arch_specific arch;
    unsigned int taints;    // same bits as kernel:tainted (污染)
    ...
};
```

#### 相关 API
1. `module_platform_driver()`
   ```c
   // include/linux/platform_device.h
   #define module_platform_driver(__platform_driver) \
      module_driver(__platform_driver, platform_driver_register, \
         platform_driver_unregister)

   // include/linux/device.h
   #define module_driver(__driver, __register, __unregister, ...) \
   static int __init __driver##_init(void) \
   { \
      return __register(&(__driver), ##__VA_ARGS__); \
   } \
   module_init(__driver##_init); \
   static void __exit __driver##_exit(void) \
   { \
      __unregister(&(__driver), ##__VA_ARGS__); \
   } \
   module_exit(__driver##_exit);
   ```
   - 通过 `module_platform_driver(xxx_platform_driver)` 宏即可完成对应的 platform driver 的注册（__init）和注销（__exit）代码。

2. `MODULE_DEVICE_TABLE()`
```c
// drivers/i2c/buses/i2c-imx.c
static struct platform_device_id imx_i2c_devtype[] = {
   {
      .name = "imx1-i2c",
      .driver_data = (kernel_ulong_t)&imx1_i2c_hwdata,
   },
   {
      .name = "imx21-i2c",
      .driver_data = (kernel_ulong_t)&imx21_i2c_hwdata,
   },
   {
      /* sentinel */
   }
};
MODULE_DEVICE_TABLE(platform, imx_i2c_devtype);
```

## 补充
### 回调接口比较
#### probe
*bus* 和 *driver* 都有 `probe()` 回调：
- 在 `bus->probe()` 中一般进行一些前置检查和设置，例如 `i2c_bus_type->probe()`：
    ```c
    // version: 4.1.15
    // drivers/i2c/i2c-core.c
    static int i2c_device_probe(struct device *dev) {
        struct i2c_client *client = i2c_verify_client(dev);
        struct i2c_driver *driver;
        int status;

        // 设备是否为空
        if (!client)
            return 0;
        // 设备中断号是否有效
        if (!client->irq && dev->of_node) {
            int irq = of_irq_get(dev->of_node, 0);
            if (irq == -EPROBE_DEFER)
                return irq;
            if (irq < 0)
                irq = 0;
            client->irq = irq;
        }
        // 设备驱动是否有效
        driver = to_i2c_driver(dev->driver);
        if (!driver->probe || !driver->id_table)
            return -ENODEV;

        if (!device_can_wakeup(&client->dev))
            device_init_wakeup(&client->dev,
                        client->flags & I2C_CLIENT_WAKE);
        dev_dbg(dev, "probe\n");

        status = of_clk_set_defaults(dev->of_node, false);
        if (status < 0)
            return status;

        status = dev_pm_domain_attach(&client->dev, true);
        if (status != -EPROBE_DEFER) {
            status = driver->probe(client, i2c_match_id(driver->id_table,
                        client));
            if (status)
                dev_pm_domain_detach(&client->dev, true);
        }

        return status;
    }
    ```
- 在 `driver->probe()` 中一般进行：
  1. 验证当前驱动是否支持给定的设备；
  2. 将系统资源 *attach to dev*：一些默认资源 + 解析设备树、设备树提供的资源 + `driver->probe()` 函数的参数中提供的资源（如 *i2c_device_id* 中提供的 `data` 数据）；
  3. 进行设备初始化工作；
  4. 将驱动相关数据存储到给定设备上，如通过 `dev_set_drvdata()` 等方法；
  5. 成功时返回 0。失败时返回负数。（注：在设备移除之前需要先释放之前 *attach* 到设备上的资源）

#### remove
- `driver->remove()`:
  - 当设备上的驱动 *unbind* 时，`remove()` 被调用；

### klist
```c
// version: 4.1.15
// include/linux/klist.h

/* klist.h - Some generic list helpers, extending struct 
 * list_head a bit.
 * Implementations are found in lib/klist.c
 */
struct klist_node;
struct klist {
    spinlock_t k_lock;
    struct list_head k_list;
    void (*get)(struct klist_node *);
    void (*put)(struct klist_node *);
} __attribute__ ((aligned(sizeof(void*))));

struct klist_node {
    void *n_klist; /* never access directly */
    struct list_head n_node;
    struct kref n_ref;
};

// lib/klist.c
/** 
 * klist_init - Initialize a klist structure. 
 * @k: The klist we're initializing.
 * @get: The get function for the embedding object (NULL if none)
 * @put: The put function for the embedding object (NULL if none)
 *
 * Initialises the klist structure. If the klist_node structures 
 * are going to be embedded in refcounted objects (necessary for
 * safe delection) then the get/put arguments are used to initialise
 * functions that take and release reference on embedding objects.
 */
void klist_init(struct klist *k, void (*get)(struct klist_node *), void (*put)(struct klist_node *)) {
    INIT_LIST_HEAD(&k->k_list);
    spin_lock_init(&k->k_lock);
    k->get = get;
    k->put = put;
}
```
- 如果 *klist_node* 嵌入到一个被引用计数的对象，则通过 *get/put* 回调来增加/释放该对象的引用计数。

### mdev
- *mdev* 和 *sysfs* 的关系（如下内容待验证）
    *mdev* 在 *busybox* 的代码包中能找到，位于 `busybox/util-linux/mdev.c` 文件中，它通过 `uevent_helper()` 函数被调用。在 *mdev* 主要完成两件事情：
    - 执行 `mdev -s` 命令时，*mdev* 扫描 `/sys/block` 和 `/sys/class` 两个目录下的 *dev* 属性文件，从该 *dev* 属性文件中获取到设备编号，并以包含该 *dev* 属性文件的目录名称作为设备名，在 `/dev` 目录下创建相应的设备文件。例如，`cat /sys/class/tty/tty0/dev` 会得到 `4:0`，*subsystem* 为 *tty*，*device_name* 为 *tty0*。
      - 补充-1：块设备保存在 `/sys/block` 目录下，内核 2.6.25 版本以后，块设备也保存在 `/sys/class/block` 目录下。*mdev* 扫描 `/sys/block` 是为了实现向后兼容。
      - 补充-2：*dev* 属性文件以 `major:minor` 的形式保存设备编号。
      - 补充-3：包含 *dev* 属性文件的目录称为 *device_name*，而 `/sys/class/` 和 *devie_name* 之间的那部分目录称为 *subsystem*。也就是每个 *dev* 属性文件所在的路径都可以表示为 `/sys/class/subsystem/device_name/dev`。
    - 当 *mdev* 因 *uevent* 事件（以前叫 *hotplug* 事件）被调用时，*mdev* 通过由 *uevent* 事件传递给他的环境变量获取到：引起该 *uevent* 事件的设备 *action* 以及该设备所在的路径 *device path*。然后判断引起该 *uevent* 事件的 *action* 是什么，并做相应操作。
      - 若 *action* 是 *add*，即有新设备加入到系统中。不管该设备是虚拟设备还是实际物理设备，*mdev* 都会通过 *device path* 路径下的 *dev* 属性文件获取到设备编号，然后以 *device path* 路径最后一个目录作为设备名，在 `/dev` 目录下创建相应的设备文件。
      - 若 *action* 是 *remove*，即设备已从系统中移除。则删除 `/dev` 目录下以 *device path* 路径最后一个目录名称作为文件名的设备文件。
      - 若 *action* 既不是 *add*，也不是 *remove*，*mdev* 则什么都不做。

    由上可知，如果需要在设备加入到系统中或从系统中移除时，由 *mdev* 自动地创建和删除设备文件，则前提条件是：
    1. 在 `/sys/class` 的某一 *subsystem* 目录下，创建一个以设备名 *device_name* 作为名称的目录；
    2. 此目录下还必须包含一个 *dev* 属性文件，该 *dev* 属性文件以 `major:minor` 形式输出设备编号。

    由上可知，***sysfs* 为 *uevent* 机制做前期的准备工作，即创建相应的目录**，而 *mdev* 则是在 *sysfs* 的基础上通过调用 *sysfs* 创建的目录或文件来实现设备节点的创建。

## 参考
1. [CSDN-嵌入式 Linux--uevent 机制：uevent 原理分析](https://blog.csdn.net/W1107101310/article/details/80211885)
2. [蜗窝科技-统一设备模型](http://www.wowotech.net/sort/device_model/page/2)
3. `Documentation/kobject.txt`
4. `Documentation/zh_CN/filesystems/sysfs.txt`
5. `Documentation/driver-model/*`