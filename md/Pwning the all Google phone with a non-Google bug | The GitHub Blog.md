> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [github.blog](https://github.blog/2023-01-23-pwning-the-all-google-phone-with-a-non-google-bug/)

> It turns out that the first “all Google” phone includes a non-Google bug. Learn about the details of ......

The “not-Google” bug in the “all-Google” phone[](#the-not-google-bug-in-the-all-google-phone)
---------------------------------------------------------------------------------------------

The year is 2021 A.D. The first “all Google” phone, the Pixel 6 series, made entirely by Google, is launched.

Well not entirely…

One small GPU chip still holds out. And life is not easy for security researchers who audit the fortified camps of [Midgard](https://developer.arm.com/Architectures/Midgard), [Bifrost](https://developer.arm.com/Architectures/Bifrost), and [Valhall](https://developer.arm.com/Architectures/Valhall).[1](#fn-69576-1 "Read footnote.")

An unfortunate security researcher was about to learn this the hard way as he wandered into the Arm Mali regime:  

CVE-2022-38181[](#cve-2022-38181)
---------------------------------

In this post I’ll cover the details of [CVE-2022-38181](https://developer.arm.com/Arm%20Security%20Center/Mali%20GPU%20Driver%20Vulnerabilities), a vulnerability in the Arm Mali GPU that I reported to the Android security team on 2022-07-12 along with a proof-of-concept exploit that used this vulnerability to gain arbitrary kernel code execution and root privileges on a Pixel 6 from an Android app. The bug was assigned bug ID 238770628. After initially rating it as a High-severity vulnerability, the Android security team later decided to reclassify it as a “Won’t fix” and they passed my report to Arm’s security team. I was eventually able to get in touch with Arm’s security team to independently follow up on the issue. The Arm security team were very helpful throughout and released a public patch in version [r40p0 of the driver on 2022-10-07](https://developer.arm.com/downloads/-/mali-drivers/valhall-kernel) to address the issue, which was considerably quicker than similar disclosures that I had in the past on Android. A coordinated disclosure date of around mid-November was also agreed to allow time for users to apply the patch. However, I was unable to connect with the Android security team and the bug was quietly fixed in the January update on the Pixel devices as bug [259695958](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13-qpr1%5E%21/#F0). Neither the CVE ID, nor the bug ID (the original 238770628 and the new 259695958) were mentioned in the security bulletin. Our advisory, including the disclosure timeline, can be found [here](https://securitylab.github.com/advisories/GHSL-2022-054_Arm_Mali/).

The Arm Mali GPU[](#the-arm-mali-gpu)
-------------------------------------

The Arm Mali GPU is a “device-specific” hardware component which can be integrated into various devices, ranging from Android phones to smart TV boxes. For example, all of the international versions of the Samsung S series phones, up to S21 use the Mali GPU, as well as the Pixel 6 series. For additional examples, see “implementations” in [Mali(GPU) Wikipedia entry](https://en.wikipedia.org/wiki/Mali_(GPU)) for some specific devices that use the Mali GPU.

As explained in my [other post](https://github.blog/2022-06-16-the-android-kernel-mitigations-obstacle-race/), GPU drivers on Android are a very attractive target for an attacker, as they can be reached directly from the untrusted app domain and most Android devices use either Qualcomm’s Adreno GPU, or the Arm Mali GPU, meaning that relatively few bugs can cover a large number of devices.

In fact, of the seven Android 0-days that were detected as exploited in the wild in 2021– five targeted GPU drivers. Another more recent bug that was exploited in the wild – [CVE-2021-39793](https://source.android.com/security/bulletin/pixel/2022-03-01), disclosed in March 2022 – also targeted a GPU driver. Together, of these six bugs that were exploited in the wild that targeted Android GPU drivers, three bugs targeted the Qualcomm GPU, while the other three targeted the Arm Mali GPU.

Due to the complexity involved in managing memory sharing between user space applications and the GPU, many of the vulnerabilities in the Arm Mali GPU involve the memory management code. The current vulnerability is another example of this, and involves a special type of GPU memory: the JIT memory.

Contrary to the name, JIT memory does not seem to be related to JIT compiled code, as it is created as non-executable memory. Instead, it seems to be used for memory caches, managed by the GPU kernel driver, that can readily be shared with user applications and returned to the kernel when memory pressure arises.

Many other types of GPU memory are created directly using ioctl calls like `[KBASE_IOCTL_MEM_IMPORT](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/tags/android-12.0.0_r0.42/mali_kbase/mali_kbase_mem_linux.c#292%3EKBASE_IOCTL_MEM_ALLOC%3C/a%3E%3C/code%3E%20or%20%3Ccode%3E%3Ca%20href=)`. (See, for example, the section [“Memory management in the Mali kernel driver”](https://github.blog/2022-07-27-corrupting-memory-without-memory-corruption/#memory-management-in-the-mali-kernel-driver) in my previous post.) This, however, is not the case for JIT memory regions, which are created by submitting a special GPU instruction using the `[KBASE_IOCTL_JOB_SUBMIT](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_core_linux.c#822)` `ioctl` call.

The `KBASE_IOCTL_JOB_SUBMIT` `ioctl` can be used to submit a “job chain” to the GPU for processing. Each job chain is basically a list of jobs, which are opaque data structures that contain job headers, followed by payloads that contain the specific instructions. For an example, see the [Writing to GPU memory](https://github.blog/2022-07-27-corrupting-memory-without-memory-corruption/#writing-to-gpu-memory) section in my previous post. While the `KBASE_IOCTL_JOB_SUBMIT` is normally used for sending instructions to the GPU itself, there are also some jobs that are implemented in the kernel and run on the host (CPU) instead. These are the software jobs (“softjobs”) and among them are jobs that instruct the kernel to allocate and free JIT memory (`BASE_JD_REQ_SOFT_JIT_ALLOC` and `BASE_JD_REQ_SOFT_JIT_FREE`).

### The life cycle of JIT memory[](#the-life-cycle-of-jit-memory)

While `KBASE_IOCTL_JOB_SUBMIT` is a general purpose `ioctl` call and contains code paths that are responsible for handling different types of GPU jobs, the `BASE_JD_REQ_SOFT_JIT_ALLOC` job essentially calls `kbase_jit_allocate_process`, which then calls `kbase_jit_allocate` to create a JIT memory region. To understand the lifetime and usage of JIT memory, let me first introduce a few different concepts.

When using the Mali GPU driver, a user app first needs to create and initialize a `[kbase_context](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/tags/android-12.0.0_r0.42/mali_kbase/mali_kbase_defs.h#1747)` kernel object. This involves the user app opening the driver file and using the resulting file descriptor to make a series of `ioctl` calls. A `kbase_context` object is responsible for managing resources for each driver file that is opened and is unique for each file handle. In particular, it has three `list_head` fields that are responsible for managing JIT memory: the `[jit_active_head](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_defs.h#1915)`, the `[jit_pool_head](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_defs.h#1916))`, and the `[jit_destroy_head](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_defs.h#1917)`. As their names suggest, `jit_active_head` contains memory that is still in use by the user application, `jit_pool_head` contains memory regions that are not in use, and `jit_destroy_head` contains memory regions that are pending to be freed and returned to the kernel. Although both `jit_pool_head` and `jit_destroy_head` are used to manage JIT regions that are free, `jit_pool_head` acts like a memory pool and contains JIT regions that are intended to be reused when new JIT regions are allocated, while `jit_destroy_head` contains regions that are going to be returned to the kernel.

When `kbase_jit_allocate` is called, it’ll first try to find a suitable region in the `jit_pool_head`:

```
    if (info->usage_id != 0)
        
        reg = find_reasonable_region(info, &kctx->jit_pool_head, false);
        ...
    if (reg) {
        ...
        list_move(®->jit_node, &kctx->jit_active_head);


```

If a suitable region is found, then it’ll be moved to `jit_active_head`, indicating that it is now in use in userland. Otherwise, a memory region will be created and added to the `jit_active_head` instead. The region allocated by `kbase_jit_allocate`, whether it is newly created or reused from `jit_pool_head`, is then stored in the `jit_alloc` array of the `kbase_context` by `[kbase_jit_allocate_process](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_softjobs.c#1201)`.

When the user no longer needs the JIT memory, it can send a `BASE_JD_REQ_SOFT_JIT_FREE` job to the GPU. This then uses `[kbase_jit_free](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem.c#4525)` to free the memory. However, rather than returning the backing pages of the memory region back to the kernel immediately, `kbase_jit_free` first reduces the backing region to a minimal size and removes any CPU side mapping, so the pages in the region are no longer reachable from the address space of the user process:

```
void kbase_jit_free(struct kbase_context *kctx, struct kbase_va_region *reg)
{
    ...
    
    old_pages = kbase_reg_current_backed_size(reg);
    if (reg->initial_commit < old_pages) {
        u64 new_size = MAX(reg->initial_commit,
            div_u64(old_pages * (100 - kctx->trim_level), 100));
        u64 delta = old_pages - new_size;
        
        if (delta) {
            mutex_lock(&kctx->reg_lock);
            kbase_mem_shrink(kctx, reg, old_pages - delta);
            mutex_unlock(&kctx->reg_lock);
        }
    }
    ...
    
    kbase_mem_shrink_cpu_mapping(kctx, reg, 0, reg->gpu_alloc->nents);    


```

Note that the backing pages of the region (`reg`) are not completely removed at this stage, and `reg` is also not going to be freed here. Instead, `reg` is moved back into `[jit_pool_head](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem.c#4579)`. However, perhaps more interestingly, `reg` is also moved to the `evict_list` of the `[kbase_context](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem.c#4579)`:

```
    kbase_mem_shrink_cpu_mapping(kctx, reg, 0, reg->gpu_alloc->nents);
    ...
    mutex_lock(&kctx->jit_evict_lock);
    
    WARN_ON(!list_empty(®->gpu_alloc->evict_node));
    
    list_add(®->gpu_alloc->evict_node, &kctx->evict_list);
    atomic_add(reg->gpu_alloc->nents, &kctx->evict_nents);
    
    list_move(®->jit_node, &kctx->jit_pool_head);


```

After `kbase_jit_free` completed, its caller, `kbase_jit_free_finish`, will also clean up the reference stored in `jit_alloc` when the region was allocated, even though `reg` is still valid at this stage:

```
static void kbase_jit_free_finish(struct kbase_jd_atom *katom)
{
    ...
    for (j = 0; j != katom->nr_extres; ++j) {
        if ((ids[j] != 0) && (kctx->jit_alloc[ids[j]] != NULL)) {
            ...
            if (kctx->jit_alloc[ids[j]] !=
                    KBASE_RESERVED_REG_JIT_ALLOC) {
                ...
                kbase_jit_free(kctx, kctx->jit_alloc[ids[j]]);
            }
            kctx->jit_alloc[ids[j]] = NULL;    
        }
    }
    ...
}


```

As we’ve seen before, the memory region in the `jit_pool_head` list may now be reused when the user allocates another JIT region. So this explains `jit_pool_head` and `jit_active_head`. What about `jit_destroy_head`? When JIT memory is freed by calling `kbase_jit_free`, it is also put on the `evict_list`. Memory regions in the `evict_list` are regions that can be freed when memory pressure arises. By putting a JIT region that is no longer in use in the `evict_list`, the Mali driver can hold onto unused JIT memory for quick reallocation, while returning them to the kernel when the resources are needed.

The Linux kernel provides a mechanism to reclaim unused cached memory, called [shrinkers](https://lwn.net/Articles/550463/). Kernel components, such as drivers, can define a `[shrinker](https://elixir.bootlin.com/linux/v5.19.9/source/include/linux/shrinker.h#L60)` object, which, amongst other things, involves defining the `count_objects` and `scan_objects` methods:

```
struct shrinker {
    unsigned long (*count_objects)(struct shrinker *,
                       struct shrink_control *sc);
    unsigned long (*scan_objects)(struct shrinker *,
                      struct shrink_control *sc);
    ...
};


```

The custom `shrinker` can then be registered via the `[register_shrinker](https://elixir.bootlin.com/linux/v5.19.9/source/mm/vmscan.c#L656)` method. When the kernel is under memory pressure, it’ll go through the list of registered `shrinkers` and use their `count_objects` method to determine potential amount of memory that can be freed, and then use `scan_objects` to free the memory. In the case of the Mali GPU driver, the `shrinker` is defined and registered in the `[kbase_mem_evictable_init](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#757)` method:

```
int kbase_mem_evictable_init(struct kbase_context *kctx)
{
    ...
    
    kctx->reclaim.count_objects = kbase_mem_evictable_reclaim_count_objects;
    kctx->reclaim.scan_objects = kbase_mem_evictable_reclaim_scan_objects;
    ...
    register_shrinker(&kctx->reclaim);
    return 0;
}


```

The more interesting part of these methods is the `[kbase_mem_evictable_reclaim_scan_objects](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#702)`, which is responsible for freeing the memory needed by the kernel.

```
static
unsigned long kbase_mem_evictable_reclaim_scan_objects(struct shrinker *s,
        struct shrink_control *sc)
{
    ...
    list_for_each_entry_safe(alloc, tmp, &kctx->evict_list, evict_node) {
        int err;

        err = kbase_mem_shrink_gpu_mapping(kctx, alloc->reg,
                0, alloc->nents);
        ...
        kbase_free_phy_pages_helper(alloc, alloc->evicted);
        ...
        list_del_init(&alloc->evict_node);
        ...
        kbase_jit_backing_lost(alloc->reg);   
    }
    ...
}


```

This is called to remove cached memory in `jit_pool_head` and return it to the kernel. The function `kbase_mem_evictable_reclaim_scan_objects` goes through the `evict_list`, unmaps the backing pages from the GPU (recall that the CPU mapping is already removed in `kbase_jit_free`) and then frees the backing pages. It then calls `kbase_jit_backing_lost` to move `reg` from `jit_pool_head` to `jit_destroy_head`:

```
void kbase_jit_backing_lost(struct kbase_va_region *reg)
{
    ...
    list_move(®->jit_node, &kctx->jit_destroy_head);

    schedule_work(&kctx->jit_work);
}


```

The memory region in `jit_destroy_head` is then picked up by the `[kbase_jit_destroy_worker](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem.c#3750)`, which then frees the `kbase_va_region` in `jit_destroy_head` and removes references to the `kbase_va_region` entirely.

Well not entirely…one small pointer still holds out against the clean up logic. And lifetime management is not easy for the pointers in the fortified camps of the Arm Mali regime.

The clean up logic in `kbase_mem_evictable_reclaim_scan_objects` is not responsible for removing the reference in `jit_alloc` from when the JIT memory is allocated, but this is not a problem, because as we’ve seen before, this reference was cleared when `kbase_jit_free_finish` was called to put the region in the `evict_list` and, normally, a JIT region is only moved to the `evict_list` when the user frees it via a `BASE_JD_REQ_SOFT_JIT_FREE` job, which removes the reference stored in `jit_alloc`.

But we don’t do normal things here, nor do the people who seek to compromise devices.

### The vulnerability[](#the-vulnerability)

While the semantics of memory eviction is closely tied to JIT memory with most eviction functionality referencing “JIT” (for example, the use of `kbase_jit_backing_lost` in `kbase_mem_evictable_reclaim_objects`), evictable memory is more general and other types of GPU memory can also be added to the `evict_list` and be made evictable. This can be achieved by calling `[kbase_mem_evictable_make](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#829)` to add memory regions to the `evict_list` and `[kbase_mem_evictable_unmake](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#855)` to remove memory regions from it. From userspace, these can be called via the `[KBASE_IOCTL_MEM_FLAGS_CHANGE](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#909)` `ioctl`. Depending on whether the `KBASE_REG_DONT_NEED` flag is passed, a memory region can be added or removed from the `evict_list`:

```
int kbase_mem_flags_change(struct kbase_context *kctx, u64 gpu_addr, unsigned int flags, unsigned int mask)
{
    ...
    prev_needed = (KBASE_REG_DONT_NEED & reg->flags) == KBASE_REG_DONT_NEED;
    new_needed = (BASE_MEM_DONT_NEED & flags) == BASE_MEM_DONT_NEED;
    if (prev_needed != new_needed) {
        ...
        if (new_needed) {
            ...
            ret = kbase_mem_evictable_make(reg->gpu_alloc);  
            if (ret)
                goto out_unlock;
        } else {
            kbase_mem_evictable_unmake(reg->gpu_alloc);     
        }
    }


```

By putting a JIT memory region directly in the `evict_list` and then creating memory pressure to trigger `kbase_mem_evictable_reclaim_scan_objects`, the JIT region will be freed with a pointer to it still stored in `jit_alloc`. After that, a `BASE_JD_REQ_SOFT_JIT_FREE` job can be submitted to trigger `kbase_jit_free_finish` to use the freed object pointed to in `jit_alloc`:

```
static void kbase_jit_free_finish(struct kbase_jd_atom *katom)
{
    ...
    for (j = 0; j != katom->nr_extres; ++j) {
        if ((ids[j] != 0) && (kctx->jit_alloc[ids[j]] != NULL)) {
            ...
            if (kctx->jit_alloc[ids[j]] !=
                    KBASE_RESERVED_REG_JIT_ALLOC) {
                ...
                kbase_jit_free(kctx, kctx->jit_alloc[ids[j]]);  
            }
            kctx->jit_alloc[ids[j]] = NULL;
        }
    }


```

Amongst other things, `kbase_jit_free` will first free some of the backing pages in the now freed `kctx->jit_alloc[ids[j]]`:

```
void kbase_jit_free(struct kbase_context *kctx, struct kbase_va_region *reg)
{
    ...
    old_pages = kbase_reg_current_backed_size(reg);
    if (reg->initial_commit < old_pages) {
        ...
        u64 delta = old_pages - new_size;
        if (delta) {
            mutex_lock(&kctx->reg_lock);
            kbase_mem_shrink(kctx, reg, old_pages - delta);  
            mutex_unlock(&kctx->reg_lock);
        }
    }


```

So by replacing the freed JIT region with a fake object, I can potentially free arbitrary pages, which is a very powerful primitive.

Exploiting the bug[](#exploiting-the-bug)
-----------------------------------------

As explained before, this bug is triggered when the kernel is under memory pressure and calls `kbase_mem_evictable_reclaim_scan_objects` via the shrinker mechanism. From a user process, the required memory pressure can be created as simply as mapping a large amount of memory using the `mmap` system call. However, the exact amount of memory required to trigger the shrinker scanning is uncertain, meaning that there is no guarantee that a shrinker scan will be triggered after such an allocation. While I can try to allocate an excessive amount of memory to ensure that the shrinker scanning is triggered, doing so risks causing an out-of-memory crash and may also cause the object replacement to be unreliable. This causes problems in triggering and exploiting the bug reliably.

It would be good if I could allocate memory incrementally and check whether the JIT region is freed by `kbase_mem_evictable_reclaim_scan_objects` after each allocation step and only proceed with the exploit when I’m sure that the bug has been triggered.

The Mali driver provides an `ioctl`, `[KBASE_IOCTL_MEM_QUERY](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#541)` for querying properties of memory regions at a specific GPU address. If the address is invalid, the `ioctl` will fail and return an error. This allows me to check whether the JIT region is freed, because when `kbase_mem_evictable_reclaim_scan_objects` is called to free the JIT region, it’ll first remove its GPU mappings, making its GPU address invalid. By using the `KBASE_IOCTL_MEM_QUERY` `ioctl` to query the GPU address of the JIT region after each allocation, I can therefore check whether the region has been freed by `kbase_mem_evictable_reclaim_scan_objects` or not, and only start spraying the heap to replace the JIT region when it is actually freed. Moreover, the `KBASE_IOCTL_MEM_QUERY` `ioctl` doesn’t involve memory allocation, so it won’t interfere with the object replacement. This makes it perfect for testing whether the bug has been triggered.

Although shrinker is a kernel mechanism for freeing up evictable memory, the scanning and removal of evictable objects via shrinkers is actually performed by the process that is requesting the memory. So for example, if my process is mapping some memory to its address space (via `mmap` and then faulting the pages), and the amount of memory that I am mapping creates sufficient memory pressure that a shrinker scan is triggered, then the shrinker scan and the removal of the evictable objects will be done in the context of my process. This, in particular, means that if I pin my process to a CPU while the shrinker scan is triggered, the JIT region that is removed during the scan will be freed on the same CPU. (Strictly speaking, this is not a hundred percent correct, because the JIT region is actually scheduled to be freed on a [worker](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem.c#4607), but most of the time, the worker is indeed executed immediately on the same CPU.) This helps me to replace the freed JIT region reliably, because when objects are freed in the kernel, they are placed within a per CPU cache, and subsequent object allocations on the same CPU will first be allocated from the CPU cache. This means that, by allocating another object of similar size on the same CPU, I’m likely to be able to replace the freed JIT region. Moreover, the JIT region, which is a `kbase_va_region`, is actually a rather large object that is allocated in the `kmalloc-256` cache, (which is used to allocate objects of size between `256-512` bytes when `kmalloc` is called) instead of the `kmalloc-128` cache, (which allocates objects of size less than `128` bytes), and the `kmalloc-256` cache is a less used cache. This, together with the relative certainty of the CPU that frees the JIT region, allows me to reliably replace the JIT region after it is freed.

### Replacing the freed object[](#replacing-the-freed-object)

Now that I can reliably replace the freed JIT region, I can look at how to exploit the bug. As explained before, the freed JIT memory can be used as the `reg` argument in the `kbase_jit_free` function to potentially be used for freeing arbitrary pages:

```
void kbase_jit_free(struct kbase_context *kctx, struct kbase_va_region *reg)
{
    ...
    old_pages = kbase_reg_current_backed_size(reg);
    if (reg->initial_commit < old_pages) {
        ...
        u64 delta = old_pages - new_size;
        if (delta) {
            mutex_lock(&kctx->reg_lock);
            kbase_mem_shrink(kctx, reg, old_pages - delta);  
            mutex_unlock(&kctx->reg_lock);
        }
    }


```

One possibility is to use the well-known [heap spraying technique](https://blog.lexfo.fr/cve-2017-11176-linux-kernel-exploitation-part3.html) to replace the freed JIT region with arbitrary data using `sendmsg`. This would enable me to create a fake `kbase_va_region` with a fake `[gpu_alloc](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem.h#529)` and fake `[pages](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem.h#138)` that could be used to free arbitrary pages in `[kbase_mem_shrink](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#2334)`:

```
int kbase_mem_shrink(struct kbase_context *const kctx,
        struct kbase_va_region *const reg, u64 new_pages)
{
    ...
    err = kbase_mem_shrink_gpu_mapping(kctx, reg,
            new_pages, old_pages);
    if (err >= 0) {
        
        kbase_mem_shrink_cpu_mapping(kctx, reg,
                new_pages, old_pages);
        kbase_free_phy_pages_helper(reg->cpu_alloc, delta);   
        if (reg->cpu_alloc != reg->gpu_alloc)
            kbase_free_phy_pages_helper(reg->gpu_alloc, delta);   


```

In order to do so, I’d need to know the addresses of some data that I can control, so I could create a fake `gpu_alloc` and its `pages` field at those addresses. This could be done either by finding a way to leak addresses of kernel objects, or use techniques like the one I wrote about in the Section “[The Ultimate fake object store](https://github.blog/2022-06-16-the-android-kernel-mitigations-obstacle-race/#the-ultimate-fake-object-store)” in my other post.

But why use a fake object when you can use a real one?

The JIT region that is involved in the use-after-free bug here is a `kbase_va_region`, which is a complex object that has multiple states. Many operations can only be performed on memory objects with a correct state. In particular, `kbase_mem_shrink` can only be used on a `kbase_va_region` that has not been mapped multiple times.

The Mali driver provides the `[KBASE_IOCTL_MEM_ALIAS](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#1765)` `ioctl` that allows multiple memory regions to share the same backing pages. I’ve written about how `KBASE_IOCTL_MEM_ALIAS` works in more details in [my previous post](https://github.blog/2022-07-27-corrupting-memory-without-memory-corruption/#memory-alias), but for the purpose of this exploit, the crucial point is that `KBASE_IOCTL_MEM_ALIAS` can be used to create memory regions in the GPU and user address spaces that are aliased to a `kbase_va_region`, meaning that they are backed by the same physical pages. If a `kbase_va_region` `reg` is mapped multiple times by using `KBASE_IOCTL_MEM_ALIAS` and then has its backing pages freed by `kbase_mem_shrink`, then only the memory mappings in `reg` are removed, so the alias regions created by `KBASE_IOCTL_MEM_ALIAS` can still be used to access the freed backing pages.

To prevent `kbase_mem_shrink` from being called on aliased JIT memory, `kbase_mem_alias` checks for the `KBASE_REG_NO_USER_FREE`, so that JIT memory cannot be aliased:

```
u64 kbase_mem_alias(struct kbase_context *kctx, u64 *flags, u64 stride,
            u64 nents, struct base_mem_aliasing_info *ai,
            u64 *num_pages)
{
    ...
    for (i = 0; i < nents; i++) {
        if (ai[i].handle.basep.handle < BASE_MEM_FIRST_FREE_ADDRESS) {
            if (ai[i].handle.basep.handle !=
                BASEP_MEM_WRITE_ALLOC_PAGES_HANDLE)
            ...
        } else {
            ...
            if (aliasing_reg->flags & KBASE_REG_NO_USER_FREE)  
                goto bad_handle; 

```

Now suppose I trigger the bug and replace the freed JIT region with a normal memory region allocated via the `[KBASE_IOCTL_MEM_ALLOC](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/heads/android-gs-raviole-5.10-android13/mali_kbase/mali_kbase_mem_linux.c#292)` `ioctl`, which is an object of the exact same type, but without the `KBASE_REG_NO_USER_FREE` flag that is associated with a JIT region. I then use `KBASE_IOCTL_MEM_ALIAS` to create an extra mapping for the backing store of this new region. All these are valid as I’m just aliasing a normal memory region that does not have the `KBASE_REG_NO_USER_FREE` flag. However, because of the bug, a dangling pointer in `jit_alloc` also points to this new region, which has now been aliased.

If I now submit a `BASE_JD_REQ_SOFT_JIT_FREE` job to call `kbase_jit_free` on this memory, then `kbase_mem_shrink` will be called, and part of the backing store in this new region will be freed, but the extra mappings created in the aliased region will not be removed, meaning that I can still access the freed backing pages from the alias region. By using a real object of the same type, not only do I save the effort needed to craft a fake object, but it also reduces the risk of having side effects that could result in a crash.

The situation is now very similar to what I had in [my previous post](https://github.blog/2022-07-27-corrupting-memory-without-memory-corruption/) and the exploit flow from this point on is also very similar. For completeness, I’ll give an overview of how the exploit works here, but readers who are interested can take a look at more details from the Section “[Breaking out of the context](https://github.blog/2022-07-27-corrupting-memory-without-memory-corruption/#breaking-out-of-the-context)” onwards in that post.

To recap, I now have access to the backing pages in a `kbase_va_region` object that is already freed and I’d like to reuse these freed backing pages so I can gain read and write access to arbitrary memory. To understand how this can be done, we need to know how backing pages to a `kbase_va_region` are allocated.

When allocating pages for the backing store of a `kbase_va_region`, the `[kbase_mem_pool_alloc_pages](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/tags/android-12.0.0_r0.42/mali_kbase/mali_kbase_mem_pool.c#529)` function is used:

```
int kbase_mem_pool_alloc_pages(struct kbase_mem_pool *pool, size_t nr_4k_pages,
        struct tagged_addr *pages, bool partial_allowed)
{
    ...
    
    while (nr_from_pool--) {
        p = kbase_mem_pool_remove_locked(pool);     
        ...
    }
    ...
    if (i != nr_4k_pages && pool->next_pool) {
        
        err = kbase_mem_pool_alloc_pages(pool->next_pool,      
                nr_4k_pages - i, pages + i, partial_allowed);
        ...
    } else {
        
        while (i != nr_4k_pages) {
            p = kbase_mem_alloc_page(pool);     
            ...
        }
        ...
    }
    ...
}


```

The input argument `kbase_mem_pool` is a memory pool managed by the `kbase_context` object associated with the driver file that is used to allocate the GPU memory. As the comments suggest, the allocation is actually done in tiers. First the pages are allocated from the current `kbase_mem_pool` using `[kbase_mem_pool_remove_locked](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/tags/android-12.0.0_r0.42/mali_kbase/mali_kbase_mem_pool.c#96)` (1 in the above). If there is not enough capacity in the current `kbase_mem_pool` to meet the request, then `pool->next_pool`, is used to allocate the pages (2 in the above). If even `pool->next_pool` does not have the capacity, then `[kbase_mem_alloc_page](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/tags/android-12.0.0_r0.42/mali_kbase/mali_kbase_mem_pool.c#153)` is used to allocate pages directly from the kernel via the buddy allocator (the page allocator in the kernel).

When freeing a page, the opposite happens: `[kbase_mem_pool_free_pages](https://android.googlesource.com/kernel/google-modules/gpu/+/refs/tags/android-12.0.0_r0.42/mali_kbase/mali_kbase_mem_pool.c#738)` first tries to return the pages to the `kbase_mem_pool` of the current `kbase_context`, if the memory pool is full, it’ll try to return the remaining pages to `pool->next_pool`. If the next pool is also full, then the remaining pages are returned to the kernel by freeing them via the buddy allocator.

As noted in [my previous post](https://github.blog/2022-07-27-corrupting-memory-without-memory-corruption/#breaking-out-of-the-context), `pool->next_pool` is a memory pool managed by the Mali driver and shared by all the `kbase_context`. It is also used for allocating [page table global directories (PGD)](https://www.kernel.org/doc/gorman/html/understand/understand006.html) used by GPU contexts. In particular, this means that by carefully arranging the memory pools, it is possible to cause a freed backing page in a `kbase_va_region` to be reused as a PGD of a GPU context. (The details of how to achieve this can be found in [my previous post](https://github.blog/2022-07-27-corrupting-memory-without-memory-corruption/#breaking-out-of-the-context).) As the bottom level PGD stores the physical addresses of the backing pages to GPU virtual memory addresses, being able to write to PGD will allow me to map arbitrary physical pages to the GPU memory, which I can then read from and write to by issuing GPU commands. This gives me access to arbitrary physical memory. As physical addresses for kernel code and static data are not randomized and depend only on the kernel image, I can use this primitive to overwrite arbitrary kernel code and gain arbitrary kernel code execution.

In the following figure, the green block indicates the same page being reused as the PGD.

To summarize, the exploit involves the following steps:

1.  Create JIT memory.
2.  Mark the JIT memory as evictable.
3.  Increase memory pressure by mapping memory to the user space via normal `mmap` system calls.
4.  Use the `KBASE_IOCTL_MEM_QUERY` `ioctl` to check if the JIT memory is freed. Carry on applying memory pressure until the JIT region is freed.
5.  Allocate new GPU memory regions using the `KBASE_IOCTL_MEM_ALLOC` `ioctl` to replace the freed JIT memory.
6.  Create an alias region to the new GPU memory region that replaced the JIT memory so that the backing pages of the new GPU memory are shared with the alias region.
7.  Submit a `BASE_JD_REQ_SOFT_JIT_FREE` job to free the JIT region. As the JIT region is now replaced by the new memory region, this will cause `kbase_jit_free` to remove the backing pages of the new memory region, but the GPU mappings created in the alias region in step 6. will not be removed. The alias region can now be used to access the freed backing pages.
8.  Reuse the freed backing pages as PGD of the `kbase_context`. The alias region can now be used to rewrite the PGD. I can then map arbitrary physical pages to the GPU address space.
9.  Map kernel code to the GPU address space to gain arbitrary kernel code execution, which can then be used to rewrite the credentials of our process to gain root, and to disable SELinux.

The exploit for Pixel 6 can be found [here](https://github.com/github/securitylab/tree/main/SecurityExploits/Android/Mali/CVE_2022_38181) with some setup notes.

Disclosure and patch gapping[](#disclosure-and-patch-gapping)
-------------------------------------------------------------

At the start of the post, I mentioned that I initially reported this bug to the Android Security team, but it was later dismissed as a “Won’t fix” bug. While it is unclear to me why such a decision was made, it is perhaps worth taking a look at the wider picture instead of treating this as an isolated incident.

There has been a long history of N-day vulnerabilities being exploited in the Android kernel, many of which were fixed in the upstream kernel but didn’t get ported to Android. Perhaps the most infamous of these was [CVE-2019-2215 (Bad Binder)](https://googleprojectzero.blogspot.com/2019/11/bad-binder-android-in-wild-exploit.html), which was initially discovered by the [syzkaller fuzzer](https://groups.google.com/forum/#!msg/syzkaller-bugs/QyXdgUhAF50/g-FXVo1OAwAJ) in November 2017 and patched in February 2018. However, this fix was never included in an Android monthly security bulletin until it was rediscovered as an exploited in-the-wild bug in September 2019. Another exploited in-the-wild bug, [CVE-2021-1048](https://googleprojectzero.github.io/0days-in-the-wild//0day-RCAs/2021/CVE-2021-1048.html), was introduced in the upstream kernel in December 2020, and was fixed upstream a few weeks later. The patch, however, was not included in the Android Security Bulletin until November 2021, when it was discovered to be exploited in-the-wild. Yet another exploited in-the-wild vulnerability, [CVE-2021-0920](https://googleprojectzero.github.io/0days-in-the-wild//0day-RCAs/2021/CVE-2021-0920.html), was found in 2016 with details visible in a Linux kernel email [thread](https://patchwork.ozlabs.org/project/netdev/patch/CAOssrKcfncAYsQWkfLGFgoOxAQJVT2hYVWdBA6Cw7hhO8RJ_wQ@mail.gmail.com/). The report, however, was dismissed by kernel developers at the time, until it was rediscovered to be exploited in-the-wild and patched in November 2021.

To be fair, these cases were patched or ignored upstream without being identified as security issues (for example, CVE-2021-0920 was ignored), making it difficult for any vendor to identify such issues before it’s too late.

This again shows the importance of properly addressing security issues and recording them by assigning a CVE-ID, so that downstream users can apply the relevant security patches. Unfortunately, vendors sometimes see having security vulnerabilities in their products as a damage to their reputation and try to silently patch or downplay security issues instead. The above examples show just how serious the consequences of such a mentality can be.

While Android has made improvements to keep the kernel branches more unified and up-to-date to avoid problems such as CVE-2019-2215, where the vulnerability was patched in some branches but not others, some recent disclosures highlight a rather worrying trend.

On March 7th, 2022, [CVE-2022-0847 (Dirty pipe)](https://dirtypipe.cm4all.com/) was disclosed publicly, with full details and a proof-of-concept exploit to overwrite read-only files. While the bug was patched upstream on February 23rd, 2022, with the patch merged into the Android kernel on [February, 24th, 2022](https://android-review.googlesource.com/c/kernel/common/+/1998671), the patch was not included in the Android Security Bulletin until May 2022 and the public proof-of-concept exploit still ran successfully on a Pixel 6 with the April patch. While this may look like another incident where a security bug was patched silently upstream, this case was very different. According to the disclosure timeline, the bug report was shared with the Android Security Team on February 21st, 2022, a day after it was reported to the Linux kernel.

Another vulnerability, [CVE-2021-39793](https://googleprojectzero.github.io/0days-in-the-wild//0day-RCAs/2021/CVE-2021-39793.html) in the Mali driver, was patched by Arm in version r36p0 of the driver (as [CVE-2022-22706](https://developer.arm.com/Arm%20Security%20Center/Mali%20GPU%20Driver%20Vulnerabilities)), which was released on [February 11th, 2022](https://developer.arm.com/downloads/-/mali-drivers/valhall-kernel). The patch was only included in the Android Security Bulletin in March as an exploited in-the-wild bug.

Yet another vulnerability, [CVE-2022-20186](https://github.blog/2022-07-27-corrupting-memory-without-memory-corruption/) that I reported to the Android Security Team on January 15th, 2022, was patched by Arm in version r37p0 of the Mali driver, which was released on April 21st, 2022. The patch was only included in the Android Security Bulletin in June and a Pixel 6 running the May patch was still affected.

Taking a look at security issues reported by Google’s Project Zero team, between June and July 2022, Jann Horn of Project Zero reported five security issues ([2325](https://bugs.chromium.org/p/project-zero/issues/detail?id=2325), [2327](https://bugs.chromium.org/p/project-zero/issues/detail?id=2327), [2331](https://bugs.chromium.org/p/project-zero/issues/detail?id=2331), [2333](https://bugs.chromium.org/p/project-zero/issues/detail?id=2333), [2334](https://bugs.chromium.org/p/project-zero/issues/detail?id=2334)) in the Arm Mali GPU that affected the Pixel phones. These issues were promptly fixed as [CVE-2022-33917](https://developer.arm.com/Arm%20Security%20Center/Mali%20GPU%20Driver%20Vulnerabilities) and [CVE-2022-36449](https://developer.arm.com/Arm%20Security%20Center/Mali%20GPU%20Driver%20Vulnerabilities) by the Arm security team on July 25th, 2022 (CVE-2022-33917 in [r39p0](https://developer.arm.com/downloads/-/mali-drivers/valhall-kernel)) and on August 18th, 2022 (CVE-2022-36449 in [r38p1](https://developer.arm.com/downloads/-/mali-drivers/valhall-kernel)). The details of these bugs, including proof of concepts, were disclosed on September 18th, 2022. However, at least some of the issues remained unfixed in December 2022, (for example, issue [2327](https://bugs.chromium.org/p/project-zero/issues/detail?id=2327) was only fixed silently in the January 2023 patch, without mentioning the CVE ID) after Project zero published a [blog post](https://googleprojectzero.blogspot.com/2022/11/mind-the-gap.html) highlighting the patching problem with these particular issues on November 22nd, 2022.

In all of these instances, the bugs were only patched in Android a couple months after a patch was publicly released upstream. In light of this, the response to this current bug is perhaps not too surprising.

The year is 2023 A.D., and it’s still easy to pwn Android with N-days entirely. Well, yes, entirely.

### Notes[](#notes)