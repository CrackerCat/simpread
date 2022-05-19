> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [vul.360.net](https://vul.360.net/archives/391?continueFlag=2065c4d6bed3a8e7a80c495d7066e013)

> 众所周知，ROP 是一种主流的 Linux 内核利用方式，它需要攻击者基于漏洞来寻找可用的 gadgets，然而这是一件十分耗费时间和精力的事情，并且有时候很有可能找不到合适的 gadget。

概述
--

众所周知，ROP 是一种主流的 Linux 内核利用方式，它需要攻击者基于漏洞来寻找可用的 gadgets，然而这是一件十分耗费时间和精力的事情，并且有时候很有可能找不到合适的 gadget。此外由于 CFI（控制流完整性校验）利用缓解措施已经被合并到了 Linux 内核主线中了，所以随着后续主流发行版的跟进，ROP 会变得不再可用。

这篇博客主要介绍一种叫做 USMA（User-Space-Mapping-Attack），跨平台通用的利用方法。它允许普通用户进程可以映射内核态内存并且修改内核代码段，通过这个方法，我们可以绕过 Linux 内核中的 CFI 缓解措施，在内核态中执行任意代码。下面此文会介绍一个漏洞，然后分别使用 ROP 和 USMA 两种方法完成对这个漏洞的利用，最后总结一下 USMA 的优势。

漏洞
--

漏洞出现在 Linux 内核中的 packet socket 模块，这个模块可以让用户在设备驱动层接受和发送 raw packets，并且为了加速数据报文的拷贝，它允许用户创建一块与内核态共享的环形缓冲区，具体的创建操作是在 packet_set_ring() 这个函数中实现的。

```
/net/packet/af_packet.c

4292 static int packet_set_ring(sk, req_u, closing, tx_ring)
4294 {
4317    if (req->tp_block_nr) {
4362        order = get_order(req->tp_block_size);
4363         pg_vec = alloc_pg_vec(req, order);
4366        switch (po->tp_version) {
4367        case TPACKET_V3:
4369            if (!tx_ring) {
4370                init_prb_bdqc(po, rb, pg_vec, req_u);
4371            }
4390        }
4391    }
4414        if (closing || atomic_read(&po->mapped) == 0) {
4417            swap(rb->pg_vec, pg_vec);
4418            if (po->tp_version <= TPACKET_V2)
4419                swap(rb->rx_owner_map, rx_owner_map);
4435        }
4450 out_free_pg_vec:
4451    bitmap_free(rx_owner_map);
4452    if (pg_vec)
4453        free_pg_vec(pg_vec, order, req->tp_block_nr);
4456 }

```

packet_set_ring() 通过用户传递的 tp_block_nr（行 4317）和 tp_block_size（行 4362）来决定分配的环形缓冲区的大小，如果 packet socket 的版本为 TPACKET_V3，那么在 init_prb_bdqc() 的调用中（行 4370），packet_ring_buffer.prb_bdqc.pkbdq 就会持有一份 pg_vec 的引用（行 584）。

```
/net/packet/af_packet.c

573 static void init_prb_bdqc(po, rb, pg_vec, req_u)
577 {
578     struct tpacket_kbdq_core *p1 = GET_PBDQC_FROM_RB(rb);
579     struct tpacket_block_desc *pbd;
583     p1->knxt_seq_num = 1;
584     p1->pkbdq = pg_vec;
603     prb_init_ft_ops(p1, req_u);
604     prb_setup_retire_blk_timer(po);
605     prb_open_block(p1, pbd);
606 }

```

如果用户传递的 tpacket_req.tp_block_nr 等于 0，那么就没有新的 pg_vec 会被分配，并且旧的 pg_vec 会被释放（行 4453），但是 packet_ring_buffer.prb_bdqc.pkbdq 仍然保留着被释放的 pg_vec 的引用。如果我们此时将 packet socket 的版本切换为 TPACKET_V2 并且再次设置缓冲区，那么保存在 pkbdq，被释放的 pg_vec 会被当做 rx_owner_map 再次被释放（行 4451），因为 packet_ring_buffer 是一个联合体，pkbdq（行 18）和 rx_owner_map（行 74）的内存偏移是一样的。

```
/net/packet/internal.h

59 struct packet_ring_buffer {
60    struct pgv *pg_vec;
73    union {
74        unsigned long *rx_owner_map;
75        struct tpacket_kbdq_core prb_bdqc;
76    };
77 };

17 struct tpacket_kbdq_core {
18     struct pgv *pkbdq;
19     unsigned int feature_req_word;
20     unsigned int hdrlen;
21     unsigned char reset_pending_on_curr_blk;
22     unsigned char delete_blk_timer;
52     struct timer_list retire_blk_timer;
53 };

```

ROP
---

ROP 的利用分为两个步骤：

1.  泄露内核地址，绕过 KASLR。
2.  劫持 PC，通过 gadget 修改进程的 cred。

这两个步骤要各自触发一次漏洞，通过选择不同的目标结构体，分别达到上述的目的。

### 信息泄露

```
/include/linux/msg.h 

9 struct msg_msg {          
10     struct list_head m_list;                                                         
11     long m_type;                                                                     
12     size_t m_ts;                                             
13     struct msg_msgseg *next;        
14     void *security;                                                                   
15                                          
16 };                                               

```

这里选择 msg_msg 结构体作为目标结构体，原因有以下两点：

1.  它含有 m_ts 成员 (行 12)，这个成员用来描述结构体下面跟着的缓冲区长度。
2.  普通用户可以读取缓冲区的内容。

通过 pg_vec double free 的漏洞，在第一次释放 pg_vec 之后，使用 msg_msg 进行堆喷，之后再次释放 pg_vec，使用 msg_msgseg 进行堆喷来修改 msg_msg 的 m_ts 成员，这样在 copy_msg 函数中就可以有一次越界读的机会（行 128）。

```
/ipc/msgutil.c
118 struct msg_msg *copy_msg(src, dst)
119 {
121         size_t len = src->m_ts;
127         alen = min(len, DATALEN_MSG);
128         memcpy(dst + 1, src + 1, alen);
129 
130         for (dst_pseg = dst->next, src_pseg = src->next;
131              src_pseg != NULL;
132              dst_pseg = dst_pseg->next, src_pseg = src_pseg->next) {
133 
134                 len -= alen;
135                 alen = min(len, DATALEN_SEG);
136                 memcpy(dst_pseg + 1, src_pseg + 1, alen);
137         }
142         return dst;
143 }

```

如果将 timerfd_ctx 结构体通过堆风水布局在 double free 的 pg_vec 后面，如下图所示，那么就可以将 timerfd_ctx 结构体的内容读取到用户态中。

![](https://vul.360.net/wp-content/uploads/2022/05/%E5%9B%BE%E7%89%871-1.png)

通过泄露 timerfd_ctx 结构体中的 function 函数指针（行 121）以及 wqh 等待队列头（行 38），就可以得到内核代码段的地址以及 timerfd_ctx 的堆地址。

```
/fs/timerfd.c
 31 struct timerfd_ctx {         
 32     union {          
 33         struct hrtimer tmr;         
 34         struct alarm alarm;          
 35     } t;                  
 38     wait_queue_head_t wqh;           
 47 };  
/include/linux/hrtimer.h
118 struct hrtimer {         
119     struct timerqueue_node      node;        
120     ktime_t             _softexpires;   
121     enum hrtimer_restart        (*function)(struct hrtimer *);     
127 };

```

### 劫持 PC

整个劫持 PC 进行 rop 的步骤如下：

![](https://vul.360.net/wp-content/uploads/2022/05/%E5%9B%BE%E7%89%872-3-1024x647.png)

1.  再次触发一次 double free，第一次释放 pg_vec 后，选择 pipe_buffer 进行占位。
2.  再次释放 pg_vec，使用 msg_msgseg 进行堆喷，修改 pipe_buffer 的 ops 成员指向刚刚泄露地址的 timerfd_ctx。
3.  释放 timerfd_ctx，使用 msg_msgseg 进行堆喷，伪造出一个 pipe_buf_operations。
4.  选择通过 ops 中的 release 函数指针劫持 PC，当 pipe 被 close 时，release 函数指针就会被调用。

```
/include/linux/pipe_fs_i.h 

 95 struct pipe_buf_operations {    
103     int (*confirm)(struct pipe_inode_info *, struct pipe_buffer *);        
109     void (*release)(struct pipe_inode_info *, struct pipe_buffer *);    
119     bool (*try_steal)(struct pipe_inode_info *, struct pipe_buffer *);     
124     bool (*get)(struct pipe_inode_info *, struct pipe_buffer *);   
125 };     

```

通过 release 函数指针的定义可以看到，pipe_buffer 作为函数的第二个参数且 pipe_buffer 内存内容可以被控制，那么通过以下的 gadget 来将栈迁移到 pipe_buffer 上。

```
push rsi; jmp qword ptr [rsi + 0x39];
pop rsp; pop r15; ret;
add rsp, 0xd0; ret;
pop rdi; ret; // 0
prepare_kernel_cred;
pop rcx; ret; // 0
test ecx, ecx; jne 0xd8ab5b; ret;
mov rdi, rax; jne 0x798d21; xor eax, eax; ret;
commit_creds;
mov rsp, rbp; pop rbp; ret;

```

可以看到上述的 gadgets 十分复杂，要是在不同的内核版本中编写通用的 exploit 的话，工作量会非常大。

USMA
----

USMA 这个利用方法的原理，其实来自于这个漏洞本身。如之前所说的，为了加速数据在用户态和内核态的传输，packet socket 可以创建一个共享环形缓冲区，这个环形缓冲区通过 alloc_pg_vec() 创建。

```
/net/packet/af_packet.c

4291 static struct pgv *alloc_pg_vec(struct tpacket_req *req, int order)         
4292 {         
4293     unsigned int block_nr = req->tp_block_nr;          
4294     struct pgv *pg_vec;     
4295     int i;
4296         
4297     pg_vec = kcalloc(block_nr, sizeof(struct pgv), GFP_KERNEL | __GFP_NOWARN);       
4301     for (i = 0; i < block_nr; i++) {     
4302         pg_vec[i].buffer = alloc_one_pg_vec_page(order);      
4305     }        
4308     return pg_vec;    
4314 } 

```

可以看到 pg_vec 实际上是一个保存着连续物理页的虚拟地址的数组，而这些虚拟地址会被 packet_mmap() 函数所使用，packet_mmap() 将这些内核虚拟地址代表的物理页映射进用户态（行 4502），这样普通用户就能在用户态对这些物理页直接进行读写。

```
/net/packet/af_packet.c

4458 static int packet_mmap(file, sock, vma)
4460 {
4491    for (rb = &po->rx_ring; rb <= &po->tx_ring; rb++) {
4495        for (i = 0; i < rb->pg_vec_len; i++) {
4496            struct page *page;
4497            void *kaddr = rb->pg_vec[i].buffer;
4500            for (pg_num = 0; pg_num < rb->pg_vec_pages; pg_num++) {
4501                page = pgv_to_page(kaddr);
4502                err = vm_insert_page(vma, start, page);
4503                if (unlikely(err))           
4504                    goto out;     
4505                start += PAGE_SIZE;
4506                kaddr += PAGE_SIZE;
4507            }
4508        }
4509      }
4517    return err;
4518 }

```

如果通过漏洞将存储在 pg_vec 的虚拟地址进行覆写，更改为内核代码段的虚拟地址，那么 vm_insert_page() 就能将内核代码段的内存页插入到用户态的虚拟地址空间中。值得一提的是，vm_insert_page 函数实际上调用 validate_page_before_insert() 函数对传入的 page 做了校验。

```
/mm/memory.c

1753 static int validate_page_before_insert(struct page *page)           
1754 {   
1755     if (PageAnon(page) || PageSlab(page) || page_has_type(page))
1756         return -EINVAL;      
1757     flush_dcache_page(page);        
1758     return 0;      
1759 }    

```

检查 page 是否为匿名页，是否为 Slab 子系统分配的页，以及 page 是否含有 type，而内存页的 type 总共有以下四种。

```
/include/linux/page-flags.h

718 #define PG_buddy      0x00000080
719 #define PG_offline    0x00000100
720 #define PG_table      0x00000200
721 #define PG_guard      0x00000400

```

PG_buddy 为伙伴系统中的页，PG_offline 为内存交换出去的页，PG_table 为用作页表的页，PG_guard 为用作内存屏障的页。可以看到如果传入的 page 为内核代码段的页，以上的检查全都可以绕过。

为了避免 vm_insert_page() 返回 err（行 4503），必须得控制 pg_vec 中所有的虚拟地址为合法的可插入的内核态虚拟地址，我们可以使用 fuse+setxattr 或者 ret2dir 来控制 pg_vec 中的所有内存。

在这个漏洞利用中，我们选择将 pg_vec 中保存的虚拟地址通过漏洞篡改为__sys_setresuid 函数所在的内核代码段页的虚拟地址，从而在用户态中对权限校验逻辑进行更改（行 659），使得普通用户也能设置自己的 uid，从而达到提权的目的。

```
/kernel/sys.c

631 long __sys_setresuid(uid_t ruid, uid_t euid, uid_t suid)
632 {
659     if (!ns_capable_setid(old->user_ns, CAP_SETUID)) {
660         if (ruid != (uid_t) -1 && !uid_eq(kruid, old->uid) &&
661             !uid_eq(kruid, old->euid) && !uid_eq(kruid, old->suid))
662             goto error;
663         if (euid != (uid_t) -1 && !uid_eq(keuid, old->uid) &&
664             !uid_eq(keuid, old->euid) && !uid_eq(keuid, old->suid))
665             goto error;
666         if (suid != (uid_t) -1 && !uid_eq(ksuid, old->uid) &&
667             !uid_eq(ksuid, old->euid) && !uid_eq(ksuid, old->suid))
668             goto error;
669     }
694 }

```

最后，可以在 alloc_pg_vec() 中看到，block_nr 是用户传入的，那么 pg_vec 的大小也是用户可控的（行 4297），这就意味着 pg_vec 可以占据不同大小的 slab，从而将各种堆上的问题转化为对内核代码段进行覆写。

总结
--

通过 USMA 这种方式，我们可以大幅提高利用编写的效率，对漏洞要求大大降低，克服了 gadget 可获得性限制，并且绕过现有的最新的 CFI 缓解措施。