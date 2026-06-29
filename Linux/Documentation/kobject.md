# kobjects, ksets, and ktypes
## 概述
### kobject
- A kobject is an object of type struct kobject；
- 包含的信息：
  - a name;
  - a reference count;
  - a parent pointer (allowing objects to be arranged into hierarchies);
  - a specific type;
  - **a representation in the sysfs virtual filesystem**;
- 特点：
  - kobjects are generally not interesting on their own; **they usually embedded within some other structure which contains the stuff the code is really interested in**.
  - No structure should EVER have more than one kobject embedded within it.

### ktype
**每个嵌入 kobject 的数据结构都需要包含一个对应的 ktype，ktype 包含对应 kobject 创建和销毁时所要执行的回调**。

- A ktype is the type of object that embeds a kobject.
- Every structure that embeds a kobject needs a corresponding ktype.
- The ktype controls what happens to the kobject when it is created and destroyed.

### kset
- 表示 kobject 的一个集合，是 kobject 的基础容器类型；
- 同一个集合中的 kobject 可以有相同的 ktype，也可以是不同的 ktype；
- **ksets 也有自己对应的 kobject**，但是可以安全地忽略它的实现细节，因为 kset 核心代码自动处理这个 kobject；
- **在 sysfs 中同一目录下的子目录会有 kobject 与之对应，而这些 kobject 属于同一个 kset**；

## kobject
### embedding kobjects
内核几乎不会单独创建 kobject，而是将其嵌入到其他数据结构中。类似的，list_head 的用法也是如此：
```c
// drivers/uio/uio.c
struct uio_map {
    struct kobject kojb;
    struct uio_mem *mem;
}
```
通常遇到的情况是：已有指向 kobject 的指针，需要通过它来找到包含它的数据结构，这可以通过 `container_of()` 宏来解决：
```c
// linux/kernel.h
container_of(pointer, type, member)
```
它会返回一个包含以该指针（`pointer`）为成员（`member`）的 `type` 类型的指针。
为方便考虑，程序员通常定义一个简要的宏以实现这样的 `back-casting`：
```c
// driver/uio/uio.c
struct uio_map {
    struct kobject kobj;
    struct uio_mem *mem;
}

#define to_map(map)     container_of(map, struct uio_map, kobj)

// 使用
struct uio_map *map = to_map(kobj);
```

### Initialization of kobjects
```c
// 初始化
void kobject_init(struct kobject *kobj, struct kobj_type *ktype);
// 初始化后，将 kobject 注册到 sysfs
int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...);
```
  - 如果 kobject 存在与之关联的 kset，那么在调用 `kobject_add()` 之前需要提前指定好 `kobj->kset`。
  - 如果一个 kset 和一个 kobject 相对应，那么在调用 `kobject_add()` 时可以给 parent 参数传递 NULL，这样使得 kobject 的 parent 将会是 kset 它自己。

当 kobject 被添加到内核，它的名字也会被设置。此时，可以通过调用 `kobject_rename()` 来改名，而绝对不可以直接操作 kobject 的名字：
```c
int kobject_rename(struct kobject *kobj, const char *new_name);
```
该函数不会执行锁和名字检查，这些都应由调用者来提供。另外，函数 `kobject_set_name()` 已被弃用。

正确地获取 kobject 的名字：
```c
const char *kobject_name(const struct kobject *kobj);
```
将初始化和加入内核这两个操作合并的函数：
```c
int kobject_init_and_add(struct kobject *kobj, struct kobj_type *ktype, struct kobject *parent, const char *fmt, ...);
```

### uevents
在 kobject 完成注册后，通过调用 `kobject_uevent()` 来 `announce to the world that it has been created`：
```c
int kobject_uevent(struct kobject *kobj, enum kobject_action action);
```
  - 在 kobject 第一次加入到内核时，传入 `KOBJ_ADD` 给 `action`。
  - This should be done only after any attributes or children of the kobject have been initialized properly, as userspace will instantly start to look for them when this call happens.
  - 当 kobject 被移出内核，`KOBJ_REMOVE` 的 `uevent` 将会自动地被 kobject 内核创建，因此调用者不必手动执行此项任务。

### Reference counts
kobject 的一个关键作用就是作为它自身被嵌入的数据结构所对应的引用计数器，只要对象的引用还存在，那么该对象（and the code which supports it）必须也存在。如下是用来操作 kobject 的引用计数的 low-level 函数：
```c
struct kobject *kobject_get(struct kobject *kobj);
void kobject_put(struct kobject *kobj);
```
  - 成功调用 `kobject_get()` 后会增加 kobject 的引用计数，并且返回指向 kobject 的指针；
  - 调用 `kobject_put()` 将会减小引用计数，同时也有可能会直接释放（free）该对象。
  - 注意，`kobject_init()` 会将引用计数设置为 1，因此设置 kobject 的代码也需要相应的在最终阶段调用 `kobject_put()` 来释放引用计数。

**因为 kobject 是动态的，所以它们被禁止通过静态声明或者在栈上定义，相应的，kobject 始终应该通过动态的方式声明**。未来版本的内核将会包含对 kobject 的运行时检查，如果它是以静态的方式被创建，那么将会警告开发者这一错误的使用方式。

如果你只是想通过 kobject 来给你的数据结构提供引用计数，那么请使用 `struct kref`，使用 kobject 只是显得“杀鸡用牛刀”（overkill）。关于使用 `struct kref` 的更多信息，请查看位于 Linux 内核源码中的 `Documentation/kref.txt` 文件。

### Create "simple" kobjects
有些时候，开发者想要的**仅仅是在 sysfs 文件目录结构中创建一个简单的目录**，而不需要考虑 ksets 的复杂性、`show()` 和 `store()` 函数以及一些其他的细节。这种是一种例外情况：**kobject 应被单独创建**。使用如下函数来创建一个入口：
```c
struct kobject *kobject_create_and_add(char *name, struct kobject *parent);
```
  - 该函数会在创建一个 kobject 并将其放置到 sysfs 中的指定父 kobject 位置下。

使用如下函数来创建对应 kobject 简单的属性：
```c
int sysfs_create_file(struct kobject *kobj, struct attribute *attr);
int sysfs_create_group(struct kobject *kobj, struct attribute_group *grp);
```
  - 上述使用的两种属性类型，以及通过 `kobject_create_and_add()` 创建的 kobject，都可以是 kobj_attribute 类型，因此不需要创建特殊的自定义属性（Both types of attributes used here, with a kobject that has been created with the kobject_create_and_add(), can be of type kobj_attribute, so no special custom attribute is needed to be created）。

查看示例模块，`samples/kobject/kobject-example.c` 来获取更多简单 kobject 及属性的应用信息。

## ktypes and release methods
还有一个没有讨论的重要事情就是当 kobject 的引用计数变为 0 时会触发什么。那些创建 kobject 的代码通常不清楚什么时候会触发，如果它们知道，there would be little point in using a kobject in the first place。sysfs 的引入使得对象可预见的生命周期变得更加复杂，因为内核的其他部分能够对任何一个已经注册到系统中的 kobject 来 get a reference。

这样造成的结果是，由 kobject 保护的数据结构在它的引用计数变为 0 之前是不能够被释放的，而引用计数又不是被创建 kobject 的代码来直接控制，因此这部分代码必须在最后一个引用计数消失（goes away）时被异步通知（notified asynchronously）。

**一旦你通过 `kobject_add()` 注册完 kobject 后，就应该决不使用 `kfree()` 来直接释放它，唯一安全的方式是调用 `kobject_put()`**。在 `kobject_init()` 后总是使用 `kobject_put()` 来避免错误的发生是一个好的实践。

kobject 的 `release()` 方法来完成（引用计数变为 0 的）事件通知，通常而言，`release()` 有如下的形式：
```c
void my_object_release(struct kobject *kobj) {
    struct my_object *mine = container_of(kobj, struct my_object, kobj);
    // 对该 kobject 执行其他的清除工作
    kfree(mine);
}
```

一个再强调也不为过的重要事情：**每一个 kobject 必须有一个 `release()` 方法，kobject 也必须一直持续存在（以一种一致的状态）直到该方法被调用**。如果这些限制并没有被满足，那么（这部分）代码是有问题的。注意，如果你忘记提供 `release()` 函数，内核会报出警告。不要试图通过提供一个 `release()` 的空实现来摆脱掉这个警告，如果你这样做了，那么你将会被内核维护者无情地嘲笑。

注意，在 `release()` 函数中可以获取 kobject 的 name，但是在该回调中改变 name 是被禁止的，否则就会造成在 kobject core 中的内存泄漏，这会让人很不爽。

有趣的是，`release()` 函数并不是存储在 kobject 中的，而是与 ktype 挂钩的。所以，让我们来介绍一下 kobj_type:
```c
struct ktype {
    void (*release)(struct kobject *kobj);
    const struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;
    const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
    const void *(*namespace)(struct kobject *kobj);
};
```
  - 该数据结构用以描述 kobject（更正确的是包含该 kobject 的对象）的特定类型，每一个 kobject 都需要有一个对应的 kobj_type 数据结构指针，**当你调用 `kobject_init()` 或 `kobject_init_and_add()` 时，该指针指向的数据结构必须被指定好**。
  - 数据结构 kobj_type 的 release 字段，是一个（函数）指针，指向这种类型的 kobject 的 `release()` 函数。另外的两个字段，sysfs_ops 和 default_attrs，控制这种类型的 kobject 如何在 sysfs 中呈现，它们超出了这个文档所讨论的范围。
  - **default_attrs 指针指向一个默认属性的链表，任何注册为该 ktype 的 kobject 会自动创建这个默认属性链表**。

## kset
kset 只不过是一个 kobject 集合，用于将 kobject 联系起来。（在同一 kset 中的 kobject）没有限制它们（必须是）相同的 ktype，但是必须小心它们的 ktype 不一样。

kset 提供如下功能：
  - 作为一个包（bag），包含 a group of objects。（例如，）内核可以通过 kset 来跟踪“所有的块设备”或者“所有的 PCI 设备驱动”；
  - **作为在 sysfs 中一个子目录，所有通过 kset 联系起来的 kobjects 会出现在该目录中。每一个 kset 包含一个 kobject，该 kobject 可以作为其他 kobjects 的父对象，sysfs 层级结构中的顶层目录通过这种方式来构建**。
  - kset 支持 kobjects 的热插拔（hotplugging），影响 uevent 事件如何被传播到用户空间。

用面向对象的术语，kset 是顶层容器对象。它包含自身的 kobject，该 kobject 被 kset 代码管理，而其他用户不应该来操作此 kobject。

kset 使用标准的内核 linked list（连接链表）来持有它的子 kobject 对象，而 kobject 对象又通过自身的 kset 字段指向包含它们的 kset。**在绝大多数情况下，kobjects 通过 parent 字段指向它们自己所属于的 kset（或者，严格来说，是嵌入 kobject 的对象）**。

kset 在自身内部包含一个 kobject，该 kobject 应当始终被动态地创建、决不应该静态声明或者在栈上定义。使用如下函数来创建 kset：
```c
struct kset *kset_create_and_add(const char *name, struct kset_uevent_ops *u, struct kobject *parent);
```

当 kset 使用结束时，调用如下函数来销毁它：
```c
void kset_unregister(struct kset *kset);
```
  - 该函数从 sysfs 中移除 kset 并且减少它的引用计数。当引用计数变为 0 时，kset 会被释放（release）；
  - 由于 kset 的其他引用可能仍然存在，`release()` 可能发生在 `kset_unregister()` 返回之后。

一个使用 kset 的示例可以从位于内核源码中 `samples/kobject/kset-example.c` 文件获取。

如果 kset 希望去控制它所联系的 kobjects 的 uevent 操作，可以通过 `kset_uevent_ops` 来处理：
```c
struct kset_uevnt_ops {
    int (*filter)(struct kset *kset, struct kobject *kobj);
    const char *(*name)(struct kset *kset, struct kobject *kobj);
    int (*uevent)(struct kset *kset, struct kobject *kobj, struct kobj_uevent_env *env);
};
```
  - `filter()` 函数允许 kset 拦截特定 kobject 的 uevent 被发送（emitted）到用户空间，如果该函数返回 0，则 uevent 将不会被发送；
  - `name()` 将会覆写掉 kset 的默认 name，该 name 会被 uevent 发送给用户空间。默认情况下，（发送到用户空间的）name 与 kset 自己的 name 相同，但是如果提供了此函数，则名字可以被覆写；
  - 当 uevent 将要被发送到用户空间时，`uevent()` 函数将会被调用，它允许更多的环境变量被添加到 uevent 上。

的确，也许有人会问，因为并没有提供将 kobject 添加到 kset 中的函数，那么 kobject 又是怎样被添加到 kset 中的呢？答案就是这部分任务被 `kobject_add()` 处理了。当 kobject 被传递给 `kobject_add` 时，它的 kset 成员就应当已经指定好了对应的 kobject 属于的 kset 对象，然后 `kobject_add` 将会处理剩余的其他工作。

If the kobject belonging to a kset has no parent kobject set，那么该 kobject 会被添加到 kset 的目录中。并不是 kset 的所有成员都有必要出现在 kset 的目录中。如果 kobject 在被添加之前已经显式地指定了父 kobject，那么该 kobject 被注册到该 kset 中，但是被添加到父 kobject 下。

## kobject removal
在 kobject 成员注册到 kobject core 之后，当使用结束时它必须被清理，这通过 `kobject_put()` 函数来完成。通过执行此操作，kobject core 将会自动地清理所有该 kobject 分配的内存。如果对该对象发送了一个 `KOBJ_ADD` 的 uevent，则与之对应的 `KOBJ_REMOVE` uevent 将会被发送，并且任何其他 sysfs housekeeping（管理）将被正确处理给调用方。

如果需要对 kobject 进行两阶段的的删除（比如当需要销毁对象时不允许休眠），则调用 `kobject_del()`，它将会从 sysfs 中注销 kobject。这会使得 kobject 不可见，但是它还未被清理掉，**它的引用计数仍会保持原样**。在这之后，调用 `kobject_put()` 来完成与之对应内存的清除工作。

如果存在循环引用的化，`kobject_del()` 函数可以被用于减少父对象的引用计数，这在父对象引用了一个子对象的情况下是有用的。循环引用必须被打破，通过显式地调用 `kobject_del()` 函数，从而使得 `release()` 函数被调用，并且前一个循环中的对象会互相释放。

## Example code to copy from
一个正确使用 ksets 和 kobjects 的更加复杂的示例，可以查看示例程序：`samples/kobject/{kobject-exampale.c,kset-example.c}`，当你选择了 `CONFIG_SAMPLE_KOBJECT` 时，该示例程序将会被作为可加载模块编译。
