> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274901.htm#msg_header_h1_11)

> [原创] 60 秒学会用 eBPF-BCC hook 系统调用

(备注 1: 为了格式工整, 前面都是废话, 建议直接从 11 hello world 开始看)  
(备注 2: 60 秒指的是在 Linux 上, 如果是 Android 可能要在基础再上加点)

目录
==

> 整理自 2022/10 (bcc Release v0.25.0)
> 
> ##### (1) BPF 是什么?
> 
> ##### (2) eBPF 是什么?
> 
> ##### (3) BCC 是什么?
> 
> ##### (4) IO Visor 是什么?
> 
> ##### (5) BCC 在内核调试技术栈中的位置?
> 
> ##### (6) 不同 Linux 内核版本对 eBPF 的支持?
> 
> ##### (7) 官方文档
> 
> ##### (8) 其他参考
> 
> ##### (9) 安装 BCC 二进制包 (Ubuntu) (测试发现没法用)
> 
> ##### (10) 自行编译安装 (Ubuntu) (推荐)
> 
> ##### (11) hello world!
> 
> ##### (12) 如何用监控 open() 函数的执行?
> 
> ##### (13) 如何 hook 任意 system call?

 

.  
.

1 BPF 是什么?
==========

Linux 内核中运行的虚拟机,  
可以在外部向其注入代码执行.  
.  
.

2 eBPF 是什么?
===========

理解成 BFP PLUS++  
.  
.

3 BCC 是什么?
==========

BPF 虚拟机只运行 BPF 指令, 直接敲 BPF 指令比较恶心.  
BCC 可以理解成辅助写 BPF 指令的**工具包**,  
用 python 和 c 语言间接生成 EBPF 指令.  
.  
.

4 IO Visor 是什么?
===============

指的是开源项目 && 开发者社区,  
BCC 是 IOVisor 项目下的编译器工具集.  
.  
.

5 BCC 在内核调试技术栈中的位置?
===================

![](https://bbs.pediy.com/upload/attach/202210/760871_AC9NUEDDBU2KNJS.png)  
.  
.

6 不同 Linux 内核版本对 eBPF 的支持?
==========================

*   参考官方文档  
    https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md  
    .
    
*   查看自己 Linux 内核版本 (ubuntu)
    
    ```
    xxx@ubuntu:~/Desktop/bcc/build$ uname -a
    Linux ubuntu 5.15.0-52-generic #58~20.04.1-Ubuntu SMP Thu Oct 13 13:09:46 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
    
    ```
    
    .  
    .
    

7 官方文档
======

*   项目地址  
    https://github.com/iovisor/bcc  
    .
*   使用 bcc 快速排除性能 / 故障 / 网络问题的例子  
    https://github.com/iovisor/bcc/blob/master/docs/tutorial.md  
    .
*   API 指南  
    https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md  
    .
*   python 开发者教程  
    https://github.com/iovisor/bcc/blob/master/docs/tutorial_bcc_python_developer.md  
    .
*   examples  
    https://github.com/iovisor/bcc/tree/master/examples  
    .  
    .

8 其他参考
======

*   Brendan Gregg 出品教程  
    https://www.brendangregg.com/ebpf.html  
    .
    
*   linux 内核调试追踪技术 20 讲  
    https://space.bilibili.com/646178510/channel/collectiondetail?sid=468091  
    .
    
*   使用 ebpf 跟踪 rpcx 微服务  
    https://colobu.com/2022/05/22/use-ebpf-to-trace-rpcx-microservices/  
    .
    

#### eBPF 监控工具 bcc 系列

*   一 启航  
    https://developer.aliyun.com/article/590484?spm=a2c6h.13262185.profile.109.74541f13UmhQiC  
    .
    
*   二 性能问题定位  
    https://developer.aliyun.com/article/590865?spm=a2c6h.13262185.profile.108.74541f13UmhQiC  
    .
    
*   三 自定义工具 trace  
    https://developer.aliyun.com/article/590867?spm=a2c6h.13262185.profile.107.74541f13UmhQiC  
    .
    
*   四 工具 argdist  
    https://developer.aliyun.com/article/590869?spm=a2c6h.13262185.profile.106.74541f13UmhQiC  
    .
    
*   五 工具 funccount  
    https://developer.aliyun.com/article/590870?spm=a2c6h.13262185.profile.105.74541f13UmhQiC  
    .
    
*   六 工具查询列表  
    https://developer.aliyun.com/article/591411?spm=a2c6h.13262185.profile.104.74541f13UmhQiC  
    .
    
*   七 开发脚本  
    https://developer.aliyun.com/article/591412?spm=a2c6h.13262185.profile.103.74541f13UmhQiC  
    .
    
*   八 BPF C  
    https://developer.aliyun.com/article/591413?spm=a2c6h.13262185.profile.102.74541f13UmhQiC  
    .
    
*   九 bcc Python  
    https://developer.aliyun.com/article/591415?spm=a2c6h.13262185.profile.101.74541f13UmhQiC  
    .  
    .
    

9 安装 BCC 二进制包 (Ubuntu) (测试发现没法用)
================================

*   具体参考官方文档  
    https://github.com/iovisor/bcc/blob/master/INSTALL.md  
    .
    
*   iovisor 版 (官网说这个比较旧)
    
    ```
    sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD
    echo "deb https://repo.iovisor.org/apt/$(lsb_release -cs) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/iovisor.list
    sudo apt-get update
    sudo apt-get install bcc-tools libbcc-examples linux-headers-$(uname -r)
    
    ```
    
    .
    
*   Nightly 版
    
    ```
    echo "deb [trusted=yes] https://repo.iovisor.org/apt/xenial xenial-nightly main" | sudo tee /etc/apt/sources.list.d/iovisor.list
    sudo apt-get update
    sudo apt-get install bcc-tools libbcc-examples linux-headers-$(uname -r)
    
    ```
    
    .
    
*   安装后目录结构  
    bcc 路径为 / usr/share/bcc
    
    ![](https://bbs.pediy.com/upload/attach/202210/760871_HHYZ7KXK6874DUS.png)  
    .  
    .
    

10 自行编译安装 (Ubuntu) (推荐)
=======================

*   确定版本自己 ubuntu 版本代码
    
    ```
    lsb_release -a
    No LSB modules are available.
    Distributor ID:    Ubuntu
    Description:    Ubuntu 20.04.5 LTS
    Release:    20.04
    Codename:    focal
    
    ```
    
    .
    
*   官网找对应 ubuntu 版本的依赖
    
    ```
    # For Focal (20.04.1 LTS)
    sudo apt install -y bison build-essential cmake flex git libedit-dev \
    libllvm12 llvm-12-dev libclang-12-dev python zlib1g-dev libelf-dev libfl-dev python3-distutils
    
    ```
    
    .
    
*   下载编译
    
    ```
    git clone https://github.com/iovisor/bcc.git
    mkdir bcc/build; cd bcc/build
    cmake ..
    make
    sudo make install
    cmake -DPYTHON_CMD=python3 .. # build python3 binding
    pushd src/python/
    make
    sudo make install
    popd
    
    ```
    
    .  
    .
    

11 hello world!
===============

*   **运行 hello_world.py**  
    进入 bcc/examples 目录,  
    运行脚本 sudo python3 hello_world.py,  
    它的逻辑是, hook 了某个 syscall, 每当运行该 syscall, 就输出 helloworld.  
    你随便点点鼠标, 就能触发它显示日志了.  
    ![](https://bbs.pediy.com/upload/attach/202210/760871_7BQU6AVSXUAS5RY.png)  
    .
    
*   **运行 hello_fields.py**  
    这个脚本是一样的逻辑, 不过输出格式对齐了,  
    ![](https://bbs.pediy.com/upload/attach/202210/760871_J4ACGQRUDJ9M4DB.png)  
    .  
    .
    

12 如何用 BCC 监控 open() 函数的执行?
===========================

进入 bcc/tools / 目录, 运行 opensoop.py 脚本.  
然后自己开 clion 编一个 demo,  
调用 open 触发 eBFP 的 callback.  
![](https://bbs.pediy.com/upload/attach/202210/760871_85ZKP5A899877UU.png)  
.  
.

13 如何 hook 任意 system call?
==========================

*   **opensoop.py 的实现**  
    ok, 上面这样 eBPF 就算跑起来了,  
    然后我不想学那些罗里吧嗦的东西,  
    就直奔主题, 你就说上面那个脚本是怎么 hook 的 open?  
    .  
    我打开那个脚本看了一下, blabla 一大堆, 基本都在处理兼容和格式,  
    看的烦, 我把不关心的东西都删了, 留下核心的代码, 写好注释放这里了.  
    .  
    .
    
*   **如何任意的 hook syscall?**  
    只关心 4 点:  
    (1) 怎么写 before?  
    (2) 怎么写 after?  
    (3) 怎么注册 hook?  
    (4) 怎么输出日志?
    

```
#!/usr/bin/env python
from __future__ import print_function
from bcc import ArgString, BPF
from bcc.containers import filter_by_containers
from bcc.utils import printb
import argparse
from collections import defaultdict
from datetime import datetime, timedelta
import os
 
# 该代码在ubuntu 20环境里运行通过
bpf_text = '''
#include #include #include struct val_t {
  u64 id;
  char comm[TASK_COMM_LEN];
  const char *fname;
};
 
 
struct data_t {
  u64 id;
  int ret;
  char comm[TASK_COMM_LEN];
  char name[NAME_MAX];
};
 
 
// 定义perf输出事件
BPF_PERF_OUTPUT(events);
 
 
// 这个api是在创建一个map变量
BPF_HASH(infotmp, u64, struct val_t);
 
 
int after_openat(struct pt_regs *ctx) {
  u64 id = bpf_get_current_pid_tgid(); // 获取tid
  struct val_t *valp;
  struct data_t data = {};
  valp = infotmp.lookup(&id); // 在map中查询id
  if (valp == 0) {
    return 0;
  }
  // 从map中读取至局部变量
  bpf_probe_read_kernel(&data.comm, sizeof(data.comm), valp->comm);
  bpf_probe_read_user_str(&data.name, sizeof(data.name), (void *)valp->fname);
  data.id = valp->id;
  data.ret = PT_REGS_RC(ctx); // before里读取了参数,此时在after里补充返回值
  events.perf_submit(ctx, &data, sizeof(data)); // 提交perf poll事件来让perf输出
  infotmp.delete(&id); // 从map中删除id
  return 0;
}
 
 
int syscall__before_openat(struct pt_regs *ctx, int dfd,
                                const char __user *filename, int flags) {
  struct val_t val = {};
  u64 id = bpf_get_current_pid_tgid();
  u32 pid = id >> 32;
  // 获取当前进程名
  if (bpf_get_current_comm(&val.comm, sizeof(val.comm)) == 0) {
    val.id = id;
    val.fname = filename;
    infotmp.update(&id, &val); // id插入map
  }
  return 0;
};
'''
 
# 注册hook
b = BPF(text=bpf_text)
b.attach_kprobe(event="__x64_sys_openat", fn_)
b.attach_kretprobe(event="__x64_sys_openat", fn_)
 
# 回调函数
def my_callback(cpu, data, size):
    temp = b["events"].event(data)
    if temp.id is not None:
        print("[pid]",temp.id & 0xffffffff, end=" ")
    if temp.name is not None:
        print("[path]",temp.name, end=" ")
    if temp.ret is not None:
        print("[ret]",temp.ret, end=" ")
    if temp.comm is not None:
        print("[comm]",temp.comm, end=" ")
    print("")
 
b["events"].open_perf_buffer(my_callback, page_cnt=64)
while True:
    try:
        # 等待数据, 触发open_perf_buffer指定的回调函数
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
pass 
```

[[2022 冬季班]《安卓高级研修班 (网课)》月薪三万班招生中～](https://www.kanxue.com/book-section_list-84.htm)

[#HOOK 注入](forum-161-1-125.htm) [#工具脚本](forum-161-1-128.htm)