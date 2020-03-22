## CGroup 实现原理

前面我们介绍了 `CGroup` 的使用与基本概念，接下来将通过分析源码（本文使用的 Linux2.6.25 版本）来介绍 `CGroup` 的实现原理。在分析源码前，我们先介绍几个重要的数据结构，因为 `CGroup` 就是通过这几个数据结构来控制进程组对各种资源的使用。

### `cgroup` 结构体

前面介绍过，`cgroup` 是用来控制进程组对各种资源的使用，而在内核中，`cgroup` 是通过 `cgroup` 结构体来描述的，我们来看看其定义：

```cpp
struct cgroup {
    unsigned long flags;        /* "unsigned long" so bitops work */
    atomic_t count;
    struct list_head sibling;   /* my parent's children */
    struct list_head children;  /* my children */
    struct cgroup *parent;      /* my parent */
    struct dentry *dentry;      /* cgroup fs entry */
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    struct cgroupfs_root *root;
    struct cgroup *top_cgroup;
    struct list_head css_sets;
    struct list_head release_list;
};
```

下面我们来介绍一下 `cgroup` 结构体各个字段的用途：
1. `flags`: 用于标识当前 `cgroup` 的状态。
2. `count`: 引用计数器，表示有多少个进程在使用这个 `cgroup`。
3. `sibling、children、parent`: 由于 `cgroup` 是通过 `层级` 来进行管理的，这三个字段就把同一个 `层级` 的所有 `cgroup` 连接成一棵树。`parent` 指向当前 `cgroup` 的父节点，`sibling` 连接着所有兄弟节点，而 `children` 连接着当前 `cgroup` 的所有子节点。
4. `dentry`: 由于 `cgroup` 是通过 `虚拟文件系统` 来进行管理的，在介绍 `cgroup` 使用时说过，可以把 `cgroup` 当成是 `层级` 中的一个目录，所以 `dentry` 字段就是用来描述这个目录的。
5. `subsys`: 前面说过，`子系统` 能够附加到 `层级`，而附加到 `层级` 的 `子系统` 都有其限制进程组使用资源的算法和统计数据。所以 `subsys` 字段就是提供给各个 `子系统` 存放其限制进程组使用资源的统计数据。我们可以看到 `subsys` 字段是一个数组，而数组中的每一个元素都代表了一个 `子系统` 相关的统计数据。从实现来看，`cgroup` 只是把多个进程组织成控制进程组，而真正限制资源使用的是各个 `子系统`。
6. `root`: 用于保存 `层级` 的一些数据，比如：`层级` 的根节点，附加到 `层级` 的 `子系统` 列表（因为一个 `层级` 可以附加多个 `子系统`），还有这个 `层级` 有多少个 `cgroup` 节点等。
7. `top_cgroup`: `层级` 的根节点（根cgroup）。

我们通过下面图片来描述 `层级` 中各个 `cgroup` 组成的树状关系：

![cgroup-links](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-links.jpg)

### `cgroup_subsys_state` 结构体

每个 `子系统` 都有属于自己的资源控制统计信息结构，而且每个 `cgroup` 都绑定一个这样的结构，这种资源控制统计信息结构就是通过 `cgroup_subsys_state` 结构体实现的，其定义如下：

```cpp
struct cgroup_subsys_state {
    struct cgroup *cgroup;
    atomic_t refcnt;
    unsigned long flags;
};
```

下面介绍一下 `cgroup_subsys_state` 结构各个字段的作用：
1. `cgroup`: 指向了这个资源控制统计信息所属的 `cgroup`。
2. `refcnt`: 引用计数器。
3. `flags`: 标志位，如果这个资源控制统计信息所属的 `cgroup` 是 `层级` 的根节点，那么就会将这个标志位设置为 `CSS_ROOT` 表示属于根节点。

从 `cgroup_subsys_state` 结构的定义看不到各个 `子系统` 相关的资源控制统计信息，这是因为 `cgroup_subsys_state` 结构并不是真实的资源控制统计信息结构，比如 `内存子系统` 真正的资源控制统计信息结构是 `mem_cgroup`，那么怎样通过这个 `cgroup_subsys_state` 结构去找到对应的 `mem_cgroup` 结构呢？我们来看看 `mem_cgroup` 结构的定义：

```cpp
struct mem_cgroup {
    struct cgroup_subsys_state css; // 注意这里
    struct res_counter res;
    struct mem_cgroup_lru_info info;
    int prev_priority;
    struct mem_cgroup_stat stat;
};
```

从 `mem_cgroup` 结构的定义可以发现，`mem_cgroup` 结构的第一个字段就是一个 `cgroup_subsys_state` 结构。下面的图片展示了他们之间的关系：

![cgroup-state-memory](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-state-memory.jpg)

从上图可以看出，`mem_cgroup` 结构包含了 `cgroup_subsys_state` 结构，`内存子系统` 对外暴露出 `mem_cgroup` 结构的 `cgroup_subsys_state` 部分（即返回 `cgroup_subsys_state` 结构的指针），而其余部分由 `内存子系统` 自己维护和使用。

由于 `cgroup_subsys_state` 部分在 `mem_cgroup` 结构的首部，所以要将 `cgroup_subsys_state` 结构转换成 `mem_cgroup` 结构，只需要通过指针类型转换即可。如下代码：

`cgroup` 结构与 `cgroup_subsys_state` 结构之间的关系如下图：

![cgroup-subsys-state](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-subsys-state.jpg)

### `css_set` 结构体

由于一个进程可以同时添加到不同的 `cgroup` 中（前提是这些 `cgroup` 属于不同的 `层级`）进行资源控制，而这些 `cgroup` 附加了不同的资源控制 `子系统`。所以需要使用一个结构把这些 `子系统` 的资源控制统计信息收集起来，方便进程通过 `子系统ID` 快速查找到对应的 `子系统` 资源控制统计信息，而 `css_set` 结构体就是用来做这件事情。`css_set` 结构体定义如下：

```cpp
struct css_set {
    struct kref ref;
    struct list_head list;
    struct list_head tasks;
    struct list_head cg_links;
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
};
```

下面介绍一下 `css_set` 结构体各个字段的作用：
1. `ref`: 引用计数器，用于计算有多少个进程在使用此 `css_set`。
2. `list`: 用于连接所有 `css_set`。
3. `tasks`: 由于可能存在多个进程同时受到相同的 `cgroup` 控制，所以用此字段把所有使用此 `css_set` 的进程连接起来。
4. `subsys`: 用于收集各种 `子系统` 的统计信息结构。

进程描述符 `task_struct` 有两个字段与此相关，如下：

```cpp
struct task_struct {
    ...
    struct css_set *cgroups;
    struct list_head cg_list;
    ...
}
```

可以看出，`task_struct` 结构的 `cgroups` 字段就是指向 `css_set` 结构的指针，而 `cg_list` 字段用于连接所有使用此 `css_set` 结构的进程列表。

`task_struct` 结构与 `css_set` 结构的关系如下图：

![cgroup-task-cssset](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-task-cssset.jpg)

### `cgroup_subsys` 结构

`CGroup` 通过 `cgroup_subsys` 结构操作各个 `子系统`，每个 `子系统` 都要实现一个这样的结构，其定义如下：

```cpp
struct cgroup_subsys {
    struct cgroup_subsys_state *(*create)(struct cgroup_subsys *ss,
                          struct cgroup *cgrp);
    void (*pre_destroy)(struct cgroup_subsys *ss, struct cgroup *cgrp);
    void (*destroy)(struct cgroup_subsys *ss, struct cgroup *cgrp);
    int (*can_attach)(struct cgroup_subsys *ss,
              struct cgroup *cgrp, struct task_struct *tsk);
    void (*attach)(struct cgroup_subsys *ss, struct cgroup *cgrp,
            struct cgroup *old_cgrp, struct task_struct *tsk);
    void (*fork)(struct cgroup_subsys *ss, struct task_struct *task);
    void (*exit)(struct cgroup_subsys *ss, struct task_struct *task);
    int (*populate)(struct cgroup_subsys *ss,
            struct cgroup *cgrp);
    void (*post_clone)(struct cgroup_subsys *ss, struct cgroup *cgrp);
    void (*bind)(struct cgroup_subsys *ss, struct cgroup *root);

    int subsys_id;
    int active;
    int disabled;
    int early_init;
    const char *name;
    struct cgroupfs_root *root;
    struct list_head sibling;
    void *private;
};
```

`cgroup_subsys` 结构包含了很多函数指针，通过这些函数指针，`CGroup` 可以对 `子系统` 进行一些操作。比如向 `CGroup` 的 `tasks` 文件添加要控制的进程PID时，就会调用 `cgroup_subsys` 结构的 `attach()` 函数。当在 `层级` 中创建新目录时，就会调用 `create()` 函数创建一些与 `子系统` 相关的文件。

除了函数指针外，`cgroup_subsys` 结构还包含了很多字段，下面说明一下各个字段的作用：
1. `subsys_id`: 表示了子系统的ID。
2. `active`: 表示子系统是否被激活。
3. `disabled`: 子系统是否被禁止。
4. `name`: 子系统名称。
5. `root`: 被附加到的层级挂载点。
6. `sibling`: 用于连接被附加到同一个层级的所有子系统。
7. `private`: 私有数据。

`内存子系统` 定义了一个名为 `mem_cgroup_subsys` 的 `cgroup_subsys` 结构，如下：

```cpp
struct cgroup_subsys mem_cgroup_subsys = {
    .name = "memory",
    .subsys_id = mem_cgroup_subsys_id,
    .create = mem_cgroup_create,
    .pre_destroy = mem_cgroup_pre_destroy,
    .destroy = mem_cgroup_destroy,
    .populate = mem_cgroup_populate,
    .attach = mem_cgroup_move_task,
    .early_init = 0,
};
```

另外 Linux 内核还定义了一个 `cgroup_subsys` 结构的数组 `subsys`，用于保存所有 `子系统` 的 `cgroup_subsys` 结构，如下：

```cpp
static struct cgroup_subsys *subsys[] = {
    cpuset_subsys,
    debug_subsys,
    ns_subsys,
    cpu_cgroup_subsys,
    cpuacct_subsys,
    mem_cgroup_subsys
};
```

### `CGroup` 的挂载

前面介绍了 `CGroup` 相关的几个结构体，接下来我们分析一下 `CGroup` 的实现。

要使用 `CGroup` 功能首先必须先进行挂载操作，比如使用下面命令挂载一个 `CGroup`：

```bash
$ mount -t cgroup -o memory memory /sys/fs/cgroup/memory
```

在上面的命令中，`-t` 参数指定了要挂载的文件系统类型为 `cgroup`，而 `-o` 参数表示要附加到此 `层级` 的子系统，上面表示附加了 `内存子系统`，当然可以附加多个 `子系统`。而紧随 `-o` 参数后的 `memory` 指定了此 `CGroup` 的名字，最后一个参数表示要挂载的目录路径。

挂载过程最终会调用内核函数 `cgroup_get_sb()` 完成，由于 `cgroup_get_sb()` 函数比较长，我们下面分段来分析其实现：

```cpp
static int cgroup_get_sb(struct file_system_type *fs_type,
     int flags, const char *unused_dev_name,
     void *data, struct vfsmount *mnt)
{
    ...

    /* First find the desired set of subsystems */
    ret = parse_cgroupfs_options(data, &opts);
    if (ret) {
        if (opts.release_agent)
            kfree(opts.release_agent);
        return ret;
    }

    root = kzalloc(sizeof(*root), GFP_KERNEL);
    if (!root) {
        if (opts.release_agent)
            kfree(opts.release_agent);
        return -ENOMEM;
    }
```

`cgroup_get_sb()` 函数首先会调用 `parse_cgroupfs_options()` 函数来解析挂载命令的参数，然后调用 `kzalloc()` 函数创建一个 `cgroupfs_root` 结构。`cgroupfs_root` 结构主要用于描述这个 `CGroup` 的挂载点，同时也代表 `层级` 的根节点，其定义如下：

```cpp
struct cgroupfs_root {
    struct super_block *sb;
    unsigned long subsys_bits;
    unsigned long actual_subsys_bits;
    struct list_head subsys_list;
    struct cgroup top_cgroup;
    int number_of_cgroups;
    struct list_head root_list;
    unsigned long flags;
    char release_agent_path[PATH_MAX];
};
```
