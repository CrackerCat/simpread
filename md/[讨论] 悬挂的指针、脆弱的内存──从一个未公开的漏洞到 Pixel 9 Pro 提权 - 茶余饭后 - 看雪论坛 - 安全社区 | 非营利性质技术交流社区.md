> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-289333.htm)

> [讨论] 悬挂的指针、脆弱的内存──从一个未公开的漏洞到 Pixel 9 Pro 提权

[讨论] 悬挂的指针、脆弱的内存──从一个未公开的漏洞到 Pixel 9 Pro 提权

发表于: 25 分钟前 9

[举报](javascript:void(0);)

### [讨论] 悬挂的指针、脆弱的内存──从一个未公开的漏洞到 Pixel 9 Pro 提权

 [![](http://passport.kanxue.com/upload/avatar/446/919446.png?1614670954)](user-home-919446.htm) [京东安全](user-home-919446.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 25 分钟前  9

初步分析
====

GPU 驱动由于其与内存管理的紧密联系，已经成为近年来 Android Kernel 中一个比较有价值的攻击面，与 GPU 相关的 CVE 不算少，但是只有很少数漏洞被公开分析，安全公告中也不会谈及漏洞细节，因此每个版本的 Patch 就成了分析漏洞的重要线索。  
在使用 LLM 分析 Mali GPU 驱动新版本 Patch (r54p0-r54p1) 的时候，我们在 csf/mali_kbase_csf_cpu_queue.c 文件中发现了以下变更

```
int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_printer *kbpr)
 {
-       bool timed_out = false;
-
        mutex_lock(&kctx->csf.lock);
        if (atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_COMPLETE) {
                kbasep_print(kbpr, "Dump request already started! (try again)\\n");
@@ -110,14 +108,10 @@ int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_pr
        kbasep_print(kbpr, "CPU Queues table (version:v" __stringify(
                                   MALI_CSF_CPU_QUEUE_DUMP_VERSION) "):\\n");
  
-       if (WARN_ON(!wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp,
-                                                msecs_to_jiffies(3000)))) {
-               kbasep_print(kbpr, "Failed to wait for completion of dump request\\n");
-               timed_out = true;
-       }
+       wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp, msecs_to_jiffies(3000));
  
        mutex_lock(&kctx->csf.lock);
-       if (!timed_out && kctx->csf.cpu_queue.buffer) {
+       if (kctx->csf.cpu_queue.buffer) {
                WARN_ON(atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_PENDING);
  
                /* The CPU queue dump is returned as a single formatted string */
@@ -128,7 +122,7 @@ int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_pr
                kctx->csf.cpu_queue.buffer = NULL;
                kctx->csf.cpu_queue.buffer_size = 0;
        } else
-               kbasep_print(kbpr, "Dump error! (timed_out = %d)\\n", timed_out);
+               kbasep_print(kbpr, "Dump error! (time out)\\n");
  
        atomic_set(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_COMPLETE);

```

看起来只是移除了一个 timeout 的条件判断，但是在查看周围代码的时候，我们发现了一个朴素的问题。还是在 csf/mali_kbase_csf_cpu_queue.c 文件，kbasep_csf_cpu_queue_dump_print 函数的上方，kbase_csf_cpu_queue_dump_buffer 中，对 kctx->csf.cpu_queue.buffer 调用 kfree 之后，没有立即将其置 NULL，甚至在之后的 else 分支中直接留下了这个悬挂的指针。

```
int kbase_csf_cpu_queue_dump_buffer(struct kbase_context *kctx, u64 buffer, size_t buf_size)
{
    size_t alloc_size = buf_size;
    char *dump_buffer;
 
    if (!buffer || !buf_size)
        return 0;
 
    if (alloc_size > KBASE_MEM_ALLOC_MAX_SIZE)
        return -EINVAL;
 
    alloc_size = (alloc_size + PAGE_SIZE) & ~(PAGE_SIZE - 1);
    dump_buffer = kzalloc(alloc_size, GFP_KERNEL);
    if (!dump_buffer)
        return -ENOMEM;
 
    WARN_ON(kctx->csf.cpu_queue.buffer != NULL);
 
    if (copy_from_user(dump_buffer, u64_to_user_ptr(buffer), buf_size)) {
        kfree(dump_buffer);
        return -EFAULT;
    }
 
    mutex_lock(&kctx->csf.lock);
 
    kfree(kctx->csf.cpu_queue.buffer);
 
    if (atomic_read(&kctx->csf.cpu_queue.dump_req_status) == BASE_CSF_CPU_QUEUE_DUMP_PENDING) {
        kctx->csf.cpu_queue.buffer = dump_buffer;
        kctx->csf.cpu_queue.buffer_size = buf_size;
        complete_all(&kctx->csf.cpu_queue.dump_cmp);
    } else
        kfree(dump_buffer);
 
    mutex_unlock(&kctx->csf.lock);
 
    return 0;
}

```

而且在 kfree 调用前，对 cpu_queue.buffer 的检查仅限于当其不为 NULL 时抛出一个 warning，这直接暗示了这里有 Double Free 的可能性。  
仔细检查 Double Free 的条件

首先需要 cpu_queue.dump_req_status 为 BASE_CSF_CPU_QUEUE_DUMP_PENDING 时调用一次函数，给 cpu_queue.buffer 挂上一个指针（符合预设逻辑）  
然后需要 cpu_queue.dump_req_status 不为 BASE_CSF_CPU_QUEUE_DUMP_PENDING 时连续调用函数，使 cpu_queue.buffer 在被 kfree 之后不被修改（不符合预设逻辑）

检查所有设置 cpu_queue.dump_req_status 的地方，一共有 3 处  
kbase_csf_cpu_queue_init 可以设置 BASE_CSF_CPU_QUEUE_DUMP_COMPLETE，但 cpu_queue.buffer 也会被清空  
kbase_csf_cpu_queue_read_dump_req 可以设置 BASE_CSF_CPU_QUEUE_DUMP_PENDING  
kbasep_csf_cpu_queue_dump_print 开头可以设置 BASE_CSF_CPU_QUEUE_DUMP_ISSUED，末尾可以设置 BASE_CSF_CPU_QUEUE_DUMP_COMPLETE

再考虑 r54p1 的 patch 中所移除的 timeout 逻辑，如果在 kbasep_csf_cpu_queue_dump_print 中发生了 timeout，就可以在不重置 cpu_queue.buffer 指针的情况下设置 cpu_queue.dump_req_status 为 BASE_CSF_CPU_QUEUE_DUMP_PENDING 以外的值

因此可行的调用顺序为  
1.kbasep_csf_cpu_queue_dump_print 开头设置 dump_req_status 为 ISSUED  
2. 调用 kbase_csf_cpu_queue_read_dump_req 设置 dump_req_status 为 PENDING  
3. 调用 kbase_csf_cpu_queue_dump_buffer，且 kbasep_csf_cpu_queue_dump_print 中 timeout  
4.kbasep_csf_cpu_queue_dump_print 末尾设置 dump_req_status 为 COMPLETE

很明显这需要一个 race，检查 kbasep_csf_cpu_queue_dump_print 函数的实现可以发现两处设置 dump_req_status 的地方分别用了两次锁，而中间则是 wait_for_completion_timeout，显然在这里 race 是有希望的。

```
int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_printer *kbpr)
{
    bool timed_out = false;
 
    mutex_lock(&kctx->csf.lock);
    if (atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_COMPLETE) {
        kbasep_print(kbpr, "Dump request already started! (try again)\\n");
        mutex_unlock(&kctx->csf.lock);
        return -EBUSY;
    }
 
    atomic_set(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_ISSUED);
    init_completion(&kctx->csf.cpu_queue.dump_cmp);
    kbase_event_wakeup(kctx);
    mutex_unlock(&kctx->csf.lock);
 
    kbasep_print(kbpr, "CPU Queues table (version:v" __stringify(MALI_CSF_CPU_QUEUE_DUMP_VERSION) "):\\n");
 
    if (WARN_ON(!wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp, msecs_to_jiffies(3000)))) {
        kbasep_print(kbpr, "Failed to wait for completion of dump request\\n");
        timed_out = true;
    }
 
    mutex_lock(&kctx->csf.lock);
    if (!timed_out && kctx->csf.cpu_queue.buffer) {
        WARN_ON(atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_PENDING);
 
        /* The CPU queue dump is returned as a single formatted string */
        kbasep_puts(kbpr, kctx->csf.cpu_queue.buffer);
        kbasep_puts(kbpr, "\\n");
 
        kfree(kctx->csf.cpu_queue.buffer);
        kctx->csf.cpu_queue.buffer = NULL;
        kctx->csf.cpu_queue.buffer_size = 0;
    } else
        kbasep_print(kbpr, "Dump error! (timed_out = %d)\\n", timed_out);
 
    atomic_set(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_COMPLETE);
 
    mutex_unlock(&kctx->csf.lock);
    return 0;
}
```int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_printer *kbpr)
{
    bool timed_out = false;
 
    mutex_lock(&kctx->csf.lock);
    if (atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_COMPLETE) {
        kbasep_print(kbpr, "Dump request already started! (try again)\\n");
        mutex_unlock(&kctx->csf.lock);
        return -EBUSY;
    }
 
    atomic_set(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_ISSUED);
    init_completion(&kctx->csf.cpu_queue.dump_cmp);
    kbase_event_wakeup(kctx);
    mutex_unlock(&kctx->csf.lock);
 
    kbasep_print(kbpr, "CPU Queues table (version:v" __stringify(MALI_CSF_CPU_QUEUE_DUMP_VERSION) "):\\n");
 
    if (WARN_ON(!wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp, msecs_to_jiffies(3000)))) {
        kbasep_print(kbpr, "Failed to wait for completion of dump request\\n");
        timed_out = true;
    }
 
    mutex_lock(&kctx->csf.lock);
    if (!timed_out && kctx->csf.cpu_queue.buffer) {
        WARN_ON(atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_PENDING);
 
        /* The CPU queue dump is returned as a single formatted string */
        kbasep_puts(kbpr, kctx->csf.cpu_queue.buffer);
        kbasep_puts(kbpr, "\\n");
 
        kfree(kctx->csf.cpu_queue.buffer);
        kctx->csf.cpu_queue.buffer = NULL;
        kctx->csf.cpu_queue.buffer_size = 0;
    } else
        kbasep_print(kbpr, "Dump error! (timed_out = %d)\\n", timed_out);
 
    atomic_set(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_COMPLETE);
 
    mutex_unlock(&kctx->csf.lock);
    return 0;
}

```

kbase_csf_cpu_queue_dump_buffer 函数本身并不会改变 cpu_queue.dump_req_status，因此只需在 kbasep_csf_cpu_queue_dump_print 函数中 race 成功，就可以随意 free cpu_queue.buffer 指针。

利用过程首先考虑理想中的执行顺序：  
1.kbasep_csf_cpu_queue_dump_print 被调用  
2. 第一次设置 dump_req_status 为 ISSUED 后，调用 kbase_csf_cpu_queue_read_dump_req 设置 dump_req_status 为 PENDING  
3. 等 3s 再调用 kbase_csf_cpu_queue_dump_buffer  
4.kbasep_csf_cpu_queue_dump_print 中 timeout  
5.kbasep_csf_cpu_queue_dump_print 第二次设置 dump_req_status  
6. 随意调用 kbase_csf_cpu_queue_dump_buffer

需要解决的问题：  
如何调用 kbasep_csf_cpu_queue_dump_print  
如何判断 dump_req_status 被第一次设置了  
如何触发 kbasep_csf_cpu_queue_dump_print 中的 timeout

调用 kbasep_csf_cpu_queue_dump_print
----------------------------------

首先搜索 kbasep_csf_cpu_queue_dump_print 的调用点：

kbasep_csf_cpu_queue_debugfs_show (debugfs 不可用)  
kcpu_queue_timeout_worker → kcpu_fence_timeout_dump  
kcpu_queue_timeout_worker 是 queue->timeout_work 的具体实现，在

fence_signal_timeout_cb 中被执行，fence_signal_timeout_cb 又是 queue->fence_signal_timeout 这个 timer 的 callback，kcpu fence signal 时 timer 会启动，因此有以下调用链

```
ioctl(KBASE_IOCTL_KCPU_QUEUE_ENQUEUE)
-> kbasep_kcpu_queue_enqueue()
-> kbase_csf_kcpu_queue_enqueue()
-> kbase_kcpu_fence_signal_prepare()
-> fence_signal_timeout_start() -> mod_timer(&kcpu_queue->fence_signal_timeout, ...)
 
fence_signal timeout
-> fence_signal_timeout_cb()
-> queue_work(..., &kcpu_queue->timeout_work);
-> kcpu_queue_timeout_worker()
-> kcpu_fence_timeout_dump()
-> kbasep_csf_cpu_queue_dump_print()

```

也就是说，申请一个 kcpu queue，enqueue 一个 BASE_KCPU_COMMAND_TYPE_FENCE_SIGNAL，fence signal 超时，就可以触发 kbasep_csf_cpu_queue_dump_print  
fence signal 在 kcpu_queue 中的处理分为两个阶段

1.prepare 阶段，会调用 kbase_kcpu_fence_signal_prepare，调用 mod_timer 启动 timer  
2.process 阶段，会调用 kbasep_kcpu_fence_signal_process，调用 mod_timer/del_timer_sync 刷新 / 结束 timer

在调用 kcpu_queue enqueue 的时候，Mali 驱动会先执行所有 command 的 prepare 阶段，再执行所有 command 的 process 阶段。虽然 kbasep_kcpu_fence_signal_process 中并没有明显的阻塞点，但是 kbase_csf_kcpu_queue_process 函数处理 command 的循环中会有一个判断，如果队列中某些 command（比如 CQS_WAIT）出错，就会终止后续 command 的处理。

```
void kbase_csf_kcpu_queue_process(struct kbase_kcpu_command_queue *queue, bool drain_queue)
    ...
    bool process_next = true;
    ...
     
    for (i = 0; i != queue->num_pending_cmds; ++i) {
        struct kbase_kcpu_command *cmd = &queue->commands[(u8)(queue->start_offset + i)];
        int status;
         
        switch (cmd->type) {
        ...
        case BASE_KCPU_COMMAND_TYPE_FENCE_SIGNAL:
            status = kbasep_kcpu_fence_signal_process(queue, &cmd->info.fence);
            KBASE_TLSTREAM_TL_KBASE_KCPUQUEUE_EXECUTE_FENCE_SIGNAL_END(kbdev, queue, status);
            break;
        case BASE_KCPU_COMMAND_TYPE_CQS_WAIT:
            status = kbase_kcpu_cqs_wait_process(kbdev, queue, &cmd->info.cqs_wait);
 
            if (!status && !drain_queue) {
                process_next = false;
            } else {
                cleanup_cqs_wait(queue, &cmd->info.cqs_wait);
            }
 
            break;
        ...
        }
        if (!process_next)
            break;
    }
    ...
}

```

因此只要在 fence_signal 前加一个会出错的 CQS_WAIT command，就可以让其超时，从而触发 kbasep_csf_cpu_queue_dump_print

判断 race 时机
----------

再来看第二个问题：判断 dump_req_status 被设置的时间点。  
kbase_csf_cpu_queue_read_dump_req 是在 kbase_read 中被调用的，如果 dump_req_status 的旧值是 ISSUED，就会将返回给用户态 read 的 event_data 设置为 BASE_CSF_NOTIFICATION_CPU_QUEUE_DUMP

```
bool kbase_csf_cpu_queue_read_dump_req(struct kbase_context *kctx, struct base_csf_notification *req)
{
    if (atomic_cmpxchg(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_ISSUED,
               BASE_CSF_CPU_QUEUE_DUMP_PENDING) != BASE_CSF_CPU_QUEUE_DUMP_ISSUED) {
        return false;
    }
 
    req->type = BASE_CSF_NOTIFICATION_CPU_QUEUE_DUMP;
    return true;
}
```bool kbase_csf_cpu_queue_read_dump_req(struct kbase_context *kctx, struct base_csf_notification *req)
{
    if (atomic_cmpxchg(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_ISSUED,
               BASE_CSF_CPU_QUEUE_DUMP_PENDING) != BASE_CSF_CPU_QUEUE_DUMP_ISSUED) {
        return false;
    }
 
    req->type = BASE_CSF_NOTIFICATION_CPU_QUEUE_DUMP;
    return true;
}

```

因此可以在用户态持续 read，直到读到 BASE_CSF_NOTIFICATION_CPU_QUEUE_DUMP。

Timeout race
------------

最后，kbasep_csf_cpu_queue_dump_print 中的 timeout 也非常容易触发，其等待的是 cpu_queue.dump_cmp 这个 completion，只有 kbase_csf_cpu_queue_dump_buffer 中会 complete。

```
if (WARN_ON(!wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp, msecs_to_jiffies(3000)))) {
        kbasep_print(kbpr, "Failed to wait for completion of dump request\\n");
        timed_out = true;
}

```

```
```if (atomic_read(&kctx->csf.cpu_queue.dump_req_status) == BASE_CSF_CPU_QUEUE_DUMP_PENDING) {
    kctx->csf.cpu_queue.buffer = dump_buffer;
    kctx->csf.cpu_queue.buffer_size = buf_size;
    complete_all(&kctx->csf.cpu_queue.dump_cmp);
} else
    kfree(dump_buffer);

```

因此，只需要 kbasep_csf_cpu_queue_dump_print 调用后 3s 内不调用 kbase_csf_cpu_queue_dump_buffer 就可以触发 timeout，但是要在超时后立马调用 kbase_csf_cpu_queue_dump_buffer，否则 dump_req_status 被设置为 COMPLETE 后就无法及时给 cpu_queue.buffer 挂上 kmalloc 指针了。

**Page UAF → Get root**

最终可以构造以下调用  
1.ioctl(KBASE_IOCTL_KCPU_QUEUE_ENQUEUE) BASE_KCPU_COMMAND_TYPE_CQS_WAIT  
2.ioctl(KBASE_IOCTL_KCPU_QUEUE_ENQUEUE) BASE_KCPU_COMMAND_TYPE_FENCE_SIGNAL  
3.wait for fence_signal timeout  
4.read until BASE_CSF_NOTIFICATION_CPU_QUEUE_DUMP  
5.wait for queue_dump_print timeout  
6.ioctl(KBASE_IOCTL_CS_CPU_QUEUE_DUMP)

如果 race 成功，接下来调用 ioctl(KBASE_IOCTL_CS_CPU_QUEUE_DUMP) 就可以重复 kfree 同一个 kmalloc 指针了。  
但是 Mali 驱动中并没有提供直接操作 cpu_queue.buffer 的接口，单纯地多次 free 除了把系统搞崩并没有其他影响，因此考虑寻找其他可控的 gadget，在第一次 free 后把 page 再分配走，从而将 Double Free 转化为可利用的 UAF。  
现在可以检查一下这个 kmalloc 指针的品相：

```
int kbase_csf_cpu_queue_dump_buffer(struct kbase_context *kctx, u64 buffer, size_t buf_size)
{
    size_t alloc_size = buf_size;
    char *dump_buffer;
     
    if (alloc_size > KBASE_MEM_ALLOC_MAX_SIZE) // 0x200000
        return -EINVAL;
 
    alloc_size = (alloc_size + PAGE_SIZE) & ~(PAGE_SIZE - 1);
    dump_buffer = kzalloc(alloc_size, GFP_KERNEL);
    if (!dump_buffer)
        return -ENOMEM;
    ...
}

```

从 kbase_csf_cpu_queue_dump_buffer 函数中可以看出，alloc size 是页对齐的，所以能用的 gadget 还是比较有限的，小一些的 slab gadget 诸如 signalfd、seq_operations 等都比较难用（需要让大 slab 的 page 被回收再被小 slab 拿走，很不稳定）。  
所幸 dump_buffer 中对 alloc_size 的要求并不算严格，最大可以达到 0x200000。翻一下 kmalloc 的源码可以发现，当请求的 size 大于最大 slab 的 size 时，就会通过 kmalloc_large 直接用 alloc_pages 从 buddy allocator 拿 pages。对应 kfree 的时候，会识别到指针所指向的 page 不属于 slab 而直接用 __free_pages 将 page 归还到 buddy allocator。  
此时可以自然地想到 Mali 作为 GPU 驱动所提供的直接处理内存的能力，而且从 Mali 拿到的内存可以被映射到用户态从而直接在用户态读写，非常好用。Mali 的 mem pool allocator 会在 mem pool 中的 page 不足时直接从 buddy allocator 拿 page，只要一次性申请大量的 GPU 内存，就可以拿到刚 kfree 到 buddy allocator 的 page。  
或许会想到，用户态 mmap 匿名页也可以从 buddy allocator 拿 page，为什么要再去从 GPU 拿呢？实际上 buddy allocator 内的 page 有一定程度的隔离，除了最基础的按照不同的 order 分组（较易流通），还会按照不同的 migrate type 分组（很难流通）。通过 mmap 匿名页拿到的 page 的 migrate type 是 Movable，通过 kmalloc_large 拿到的 page 是 Unmovable 的，通过 mmap 匿名页很难拿到被 kfree 释放的 page，而从 GPU 拿到的 page 也是 Unmovable 的，可以很容易地实现与 kmalloc 之间的 page 互通。当然还有许多其他方法，这里就不深入讨论了。  
此时再触发一次 kfree，就可以拿到可读写的已回收 page，接下来要做的就简单了，mmap 大量内存，使 UAF 的 page 被分配为页表，然后通过改写页表就可以做到任意物理地址读写（可以通过 GPU 内存 mmap 到用户态的 buffer 直接读写页表而无需其他介质），之后的利用就如履平地了。  
在实际利用的过程中，我们尝试将 mmap 的地址按 2M 对齐，发现在申请的 GPU 内存中，只有一个 page 被重新分配为一张 2 级页表，其他的既没有变成 3 级页表，也没有变成普通的数据页。这是因为 Mali 的 mem pool allocator 在向 buddy allocator 拿 page 的时候，是一张一张申请的，从而把原本 buddy allocator 中的复合 page 打碎了。此时 kfree 会将指针对应的 page 当作单一 page 处理，最终只有一个 page 被再次回收。对于 2 级页表，只需将其页表项低位的描述符改为 block，就可以当作 huge page 的末级页表来使用。  
最终我们在一台 Pixel 9 Pro（安全更新版本为 25 年 11 月）上拿到了 root（在其他相关机型上也理论可行）。在开启 kernel MTE 的情况下，仍然可以利用成功拿到 root 权限，但是在 kernel 的日志中可以看到 kasan 的 UAF warning。

漏洞历史
====

这个漏洞在 r53p0 引入，为 kbasep_csf_cpu_queue_dump_print 函数添加了 timeout，从而让 kbase_csf_cpu_queue_dump_buffer 中 cpu_queue.buffer 指针的悬挂成为可能。

```
插入代码
```diff --git a/driver-r52p0/drivers/gpu/arm/midgard/csf/mali_kbase_csf_cpu_queue.c b/driver-r53p0/drivers/gpu/arm/midgard/csf/mali_kbase_csf_cpu_queue.c
index 087cdb4..2a1bdaa 100644
--- a/driver-r52p0/drivers/gpu/arm/midgard/csf/mali_kbase_csf_cpu_queue.c
+++ b/driver-r53p0/drivers/gpu/arm/midgard/csf/mali_kbase_csf_cpu_queue.c
@@ -1,7 +1,7 @@
 // SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note
 /*
  *
- * (C) COPYRIGHT 2023 ARM Limited. All rights reserved.
+ * (C) COPYRIGHT 2023-2024 ARM Limited. All rights reserved.
  *
  * This program is free software and is provided to you under the terms of the
  * GNU General Public License version 2 as published by the Free Software
@@ -93,6 +93,8 @@ int kbase_csf_cpu_queue_dump_buffer(struct kbase_context *kctx, u64 buffer, size
  
 int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_printer *kbpr)
 {
+       bool timed_out = false;
+
        mutex_lock(&kctx->csf.lock);
        if (atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_COMPLETE) {
                kbasep_print(kbpr, "Dump request already started! (try again)\\n");
@@ -108,10 +110,14 @@ int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_pr
        kbasep_print(kbpr, "CPU Queues table (version:v" __stringify(
                                   MALI_CSF_CPU_QUEUE_DUMP_VERSION) "):\\n");
  
-       wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp, msecs_to_jiffies(3000));
+       if (WARN_ON(!wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp,
+                                                msecs_to_jiffies(3000)))) {
+               kbasep_print(kbpr, "Failed to wait for completion of dump request\\n");
+               timed_out = true;
+       }
  
        mutex_lock(&kctx->csf.lock);
-       if (kctx->csf.cpu_queue.buffer) {
+       if (!timed_out && kctx->csf.cpu_queue.buffer) {
                WARN_ON(atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_PENDING);
  
                /* The CPU queue dump is returned as a single formatted string */
@@ -122,7 +128,7 @@ int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_pr
                kctx->csf.cpu_queue.buffer = NULL;
                kctx->csf.cpu_queue.buffer_size = 0;
        } else
-               kbasep_print(kbpr, "Dump error! (time out)\\n");
+               kbasep_print(kbpr, "Dump error! (timed_out = %d)\\n", timed_out);
  
        atomic_set(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_COMPLETE);

```

不过在 r54p1 的 “修复” 中，并没有解决 kbase_csf_cpu_queue_dump_buffer 中的指针悬挂，而是简单的回退了 kbasep_csf_cpu_queue_dump_print 中的 timeout，也许这个漏洞在之后的版本还会“死灰复燃”。

```
插入代码
```diff --git a/driver-r54p0/drivers/gpu/arm/midgard/csf/mali_kbase_csf_cpu_queue.c b/driver-r54p1/drivers/gpu/arm/midgard/csf/mali_kbase_csf_cpu_queue.c
index 2a1bdaa..087cdb4 100644
--- a/driver-r54p0/drivers/gpu/arm/midgard/csf/mali_kbase_csf_cpu_queue.c
+++ b/driver-r54p1/drivers/gpu/arm/midgard/csf/mali_kbase_csf_cpu_queue.c
@@ -1,7 +1,7 @@
 // SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note
 /*
  *
- * (C) COPYRIGHT 2023-2024 ARM Limited. All rights reserved.
+ * (C) COPYRIGHT 2023 ARM Limited. All rights reserved.
  *
  * This program is free software and is provided to you under the terms of the
  * GNU General Public License version 2 as published by the Free Software
@@ -93,8 +93,6 @@ int kbase_csf_cpu_queue_dump_buffer(struct kbase_context *kctx, u64 buffer, size
  
 int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_printer *kbpr)
 {
-       bool timed_out = false;
-
        mutex_lock(&kctx->csf.lock);
        if (atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_COMPLETE) {
                kbasep_print(kbpr, "Dump request already started! (try again)\\n");
@@ -110,14 +108,10 @@ int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_pr
        kbasep_print(kbpr, "CPU Queues table (version:v" __stringify(
                                   MALI_CSF_CPU_QUEUE_DUMP_VERSION) "):\\n");
  
-       if (WARN_ON(!wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp,
-                                                msecs_to_jiffies(3000)))) {
-               kbasep_print(kbpr, "Failed to wait for completion of dump request\\n");
-               timed_out = true;
-       }
+       wait_for_completion_timeout(&kctx->csf.cpu_queue.dump_cmp, msecs_to_jiffies(3000));
  
        mutex_lock(&kctx->csf.lock);
-       if (!timed_out && kctx->csf.cpu_queue.buffer) {
+       if (kctx->csf.cpu_queue.buffer) {
                WARN_ON(atomic_read(&kctx->csf.cpu_queue.dump_req_status) != BASE_CSF_CPU_QUEUE_DUMP_PENDING);
  
                /* The CPU queue dump is returned as a single formatted string */
@@ -128,7 +122,7 @@ int kbasep_csf_cpu_queue_dump_print(struct kbase_context *kctx, struct kbasep_pr
                kctx->csf.cpu_queue.buffer = NULL;
                kctx->csf.cpu_queue.buffer_size = 0;
        } else
-               kbasep_print(kbpr, "Dump error! (timed_out = %d)\\n", timed_out);
+               kbasep_print(kbpr, "Dump error! (time out)\\n");
  
        atomic_set(&kctx->csf.cpu_queue.dump_req_status, BASE_CSF_CPU_QUEUE_DUMP_COMPLETE);

```

ARM 在 12 月的安全公告中披露了三个 CVE：  
A local non-privileged user process can perform improper GPU processing operations to expose sensitive data. This issue has been assigned the identifier CVE-2025-2879.  
A local non-privileged user process can perform improper GPU memory processing operations to gain access to already freed memory. This issue has been assigned the identifier CVE-2025-6349.  
A local non-privileged user process can perform improper GPU processing operations to gain access to already freed memory. This issue has been assigned the identifier CVE-2025-8045.

影响范围为：  
CVE-2025-2879: All versions from r29p0-r49p4, r50p0-r54p0  
CVE-2025-6349: All versions from r53p0-r54p1  
CVE-2025-8045: All versions from r53p0-r54p1  
虽然没有漏洞细节，但从受影响的版本来看，本文所分析的漏洞可能已被认定为 CVE-2025-6349 或 CVE-2025-8045。

总结
==

本文介绍了一次从 patch 分析未公开漏洞并最终完成提权利用的过程。尽管漏洞细节未被公开，但从 patch 文件中仍可以发现一些蛛丝马迹，这其中很有可能暗藏通向提权等利用的路径，而且供应链上下游之间安全补丁传递的延迟也为漏洞的在野利用提供了不小的风险窗口。  
本文所分析的漏洞仅表现为一处悬挂指针，在正常流程下会被直接覆盖掉而不会有任何影响，但是在攻击者的视角下，任何 “不完美” 的代码都有可能被利用。通过进行构造输入，这个悬挂的指针被引申到内存页的 UAF 中，并最终导致了权限提示。或许开发人员意识到了悬挂的指针可能会被滥用，但是一句简单的 WARN_ON 并不能在生产环境中阻止恶意程序的攻击，甚至无法让用户感知到风险的存在。  
从移动端 GPU 安全的视角出发，GPU 与 kernel 内存管理的紧密联系暴露了一个非常大的攻击面，不仅体现在其漏洞会直接影响内存，也有为漏洞利用提供优良 Gadget 的风险。用户态程序通过 GPU 驱动可以非常方便的直接操作内存页，包括内存页的申请 / 回收 / 读写，这也可能会为源自其他地方（GPU 以外）的漏洞的利用提供便利，“短板效应” 在安全领域尤为明显。未来 GPU 安全的发展如何，也是值得持续关注的。

References
==========

https://developer.arm.com/documentation/110697/1-0/?lang=en  
https://source.android.com/docs/security/bulletin/2025-12-01?hl=zh-cn#Arm-components

  

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)