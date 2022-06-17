> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [github.blog](https://github.blog/2022-06-16-the-android-kernel-mitigations-obstacle-race/)

> In this post I’ll exploit CVE-2022-22057, a use-after-free in the Qualcomm gpu kernel driver, to gain......

In this post, I’ll exploit a use-after-free (UAF) bug, CVE-2022-22057 in the Qualcomm GPU driver, which affected the kernel branch 5.4 or above, and is mostly used by flagship models running the Snapdragon 888 chipset or above (for example, the Snapdragon version of S21—used in the U.S. and a number of Asian countries, such as China and Korea— and all versions of the Galaxy Z Flip3, and many others). The device tested here is the Samsung Galaxy Z Flip3 and I was able to use this bug alone to gain arbitrary kernel memory read and write, and from there, disable SELinux and run arbitrary commands as root. The bug itself was publicly disclosed in the [Qualcomm security bulletin in May 2022](https://docs.qualcomm.com/product/publicresources/securitybulletin/may-2022-bulletin.html) and the fix was applied to devices in the May 2022 Android security patch.

Why Android GPU drivers
-----------------------

While the bug itself is a fairly standard use-after-free bug that involves a tight race condition in the GPU driver, and this post focuses mostly on bypassing the many mitigations that are in place on the device rather on the GPU, it is, nevertheless, worth giving some motivations as to why the Android GPU makes an attractive target for attackers.

As was mentioned in the article “[The More You Know, The More You Know You Don’t Know](https://googleprojectzero.blogspot.com/2022/04/the-more-you-know-more-you-know-you.html)” by Maddie Stone, out of seven Android 0-days that were detected as exploited in the wild in 2021, five of them targeted GPU drivers. As of the date of writing, another bug that was exploited in the wild, [CVE-2021-39793](https://source.android.com/security/bulletin/pixel/2022-03-01) disclosed in March 2022 also targeted the GPU driver.

Apart from the fact that most Android devices use either the Qualcomm Adreno or the ARM Mali GPU, making it possible to obtain universal coverage with relatively few bugs (this was mentioned in Maddie Stone’s article), the GPU drivers are also reachable from the untrusted app sandbox in all Android devices, further reducing the number of bugs that are required in a full chain. Another reason GPU drivers are attractive is that most GPU drivers also handle rather complex memory sharing logic between the GPU device and the CPU. These often involve fairly elaborate memory management code that is prone to bugs that can be abused to achieve arbitrary read and write of physical memory or to bypass memory protection. As these bugs enable an attacker to abuse the functionality of the GPU memory management code, many of them are also undetectable as memory corruptions and immune to existing mitigations, which mostly aim at preventing control flow hijacking. Some examples are the work of [Guang Gong](https://github.com/secmob/TiYunZong-An-Exploit-Chain-to-Remotely-Root-Modern-Android-Devices/blob/master/us-20-Gong-TiYunZong-An-Exploit-Chain-to-Remotely-Root-Modern-Android-Devices-wp.pdf) and [Ben Hawkes](https://googleprojectzero.blogspot.com/2020/09/attacking-qualcomm-adreno-gpu.html), who exploited logic errors in the handling of GPU opcode to gain arbitrary memory read and write.

The vulnerability
-----------------

The vulnerability was introduced in the 5.4 branch of the Qualcomm [msm 5.4 kernel](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/tree/msm-5.4.r2) when the new [kgsl timeline](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/e0952b5cce5bcccbe04a18a9673869e6924ccac0/drivers/gpu/msm/kgsl_timeline.c) feature, together with some new ioctl associated with it, was introduced. The msm 5.4 kernel carried out some rather major refactoring of the kernel graphics support layer (kgsl) driver (under [drivers/gpu/msm](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/tree/msm-5.4.r2/drivers/gpu/msm), which is Qualcomm’s GPU driver) and introduced some new features. Both these new features and refactoring resulted in a number of regressions and new security issues, most of which were found and fixed internally and then disclosed publicly as security issues in the bulletins (kudos to Qualcomm for not silently patching security issues), including some that look [fairly exploitable](https://source.codeaurora.org/quic/la/kernel/msm-5.4/commit/?id=6c58829f4428f2564833743de7135f4451077e75).

The `kgsl_timeline` object can be created and destroyed via the ioctl `[IOCTL_KGSL_TIMELINE_CREATE](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/e0952b5cce5bcccbe04a18a9673869e6924ccac0/drivers/gpu/msm/kgsl_timeline.c#L323)` and `[IOCTL_KGSL_TIMELINE_DESTROY](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/e0952b5cce5bcccbe04a18a9673869e6924ccac0/drivers/gpu/msm/kgsl_timeline.c#L96)`. The `kgsl_timeline` object stores a list of `[dma_fence](https://www.kernel.org/doc/html/v5.9/driver-api/dma-buf.html)` objects in the field `[fences](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/e0952b5cce5bcccbe04a18a9673869e6924ccac0/drivers/gpu/msm/kgsl_timeline.h#L26)`. The ioctl `[IOCTL_KGSL_TIMELINE_FENCE_GET](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/e0952b5cce5bcccbe04a18a9673869e6924ccac0/drivers/gpu/msm/kgsl_timeline.c#L427)` and `[IOCTL_KGSL_TIMELINE_WAIT](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/e0952b5cce5bcccbe04a18a9673869e6924ccac0/drivers/gpu/msm/kgsl_timeline.c#L358)` can be used to add `dma_fence` objects to this list. The `dma_fence` objects added are refcounted objects and their refcounts are decreased using the standard `dma_fence_put` method.

What is interesting about `timeline->fences` is that it does not actually hold an extra refcount to the fences. Instead, to avoid a `dma_fence` in `timeline->fences` from being freed, a customized `[release](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/drivers/gpu/msm/kgsl_timeline.c#L231)` function, `[timeline_fence_release](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/drivers/gpu/msm/kgsl_timeline.c#L165)` is used to remove the `dma_fence` from `timeline->fences` before it gets freed.

When the refcount of a `dma_fence` stored in `kgsl_timeline::fences` is decreased to zero, the method `[timeline_fence_release](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/drivers/gpu/msm/kgsl_timeline.c#L165)` will be called to remove the `dma_fence` from `kgsl_timeline::fences` so that it can no longer be referenced from the `kgsl_timeline`, and then `dma_fence_free` is called to free the object itself:

```
static void timeline_fence_release(struct dma_fence *fence)
{
    ...
    spin_lock_irqsave(&timeline->fence_lock, flags);

    /* If the fence is still on the active list, remove it */
    list_for_each_entry_safe(cur, temp, &timeline->fences, node) {
        if (f != cur)
            continue;

        list_del_init(&f->node);    //<----- 1. Remove fence
        break;
    }
    spin_unlock_irqrestore(&timeline->fence_lock, flags);
    ...
    kgsl_timeline_put(f->timeline);
    dma_fence_free(fence);     //<-------    2.  frees the fence
}
```

Although the removal of `fence` from `timeline->fences` is correctly protected by the `timeline->fence_lock`, `[IOCTL_KGSL_TIMELINE_DESTROY](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/drivers/gpu/msm/kgsl_timeline.c#L515)` makes it possible to acquire a reference to a `dma_fence` in `fences` after its refcount has reached zero but before it gets removed from `fences` in `timeline_fence_release`:

```
long kgsl_ioctl_timeline_destroy(struct kgsl_device_private *dev_priv,
        unsigned int cmd, void *data)
{
    ...
    spin_lock(&timeline->fence_lock);  //<------------- a.
    list_for_each_entry_safe(fence, tmp, &timeline->fences, node)
        dma_fence_get(&fence->base);
    list_replace_init(&timeline->fences, &temp);
    spin_unlock(&timeline->fence_lock);


    spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) { //<----- b.
        dma_fence_set_error(&fence->base, -ENOENT);
        dma_fence_signal_locked(&fence->base);
        dma_fence_put(&fence->base);
    }
    spin_unlock_irq(&timeline->lock);
    ...
}
```

In `kgsl_ioctl_timeline_destroy`, when destroying the timeline, the fences in `timeline->fences` are first copied to another list, `temp` and then removed from `timeline->fences` (point a.). As `timeline->fences` does not hold an extra reference of the fence, refcount is increased to stop them from being free’d in `temp`. Again, the manipulation of `timeline->fences` is protected by `timeline->fence_lock` here. However, if the refcount of a fence is already zero when `a` in the above is reached, but `timeline_fence_release` has not yet been able to remove it from `timeline->fences`, (it has not reached point 1. In the snippet included in `timeline_fence_release`), then the `dma_fence` would be moved to `temp`, and although its reference is increased, it is already too late, because `timeline_fence_release` will free the `dma_fence` when it reaches point 2., regardless of the refcount. So if the events happens in the following order, then a use-after-free could be triggered at point b.:

![](https://github.blog/wp-content/uploads/2022/06/Kernel-1.png?resize=960%2C540)

In the above, the red blocks indicate code that are holding the same lock, meaning that the execution of these blocks are mutually exclusive. While the order of events may look rather contrived (as it always is when you try to illustrate a race condition), the actual timing is not too hard to achieve. As the code in `timeline_fence_release` that removes a `dma_fence` from `timeline->fences` cannot run while the code in `kgsl_ioctl_timeline_destroy` is accessing `timeline->fence` (both are holding `timeline->fence_lock`), by adding a large number of `dma_fence` to `timeline->fence`, I can increase the time required to run the red code block in `kgsl_ioctl_timeline_destroy`. If I decrease the refcount of the last `dma_fence` in `timeline->fences` in thread two to zero while the red code block in thread one is running, I can trigger `timeline_fence_release` before `dma_fence_get` increases the refcount of this `dma_fence` in thread one. As the red code block in thread two also needs to acquire the `timeline->fence_lock`, it can not remove the `dma_fence` from `timeline->fences` until after the red code block in thread one finished. By that time, all the `dma_fence` in `timeline->fences` have been moved to the list `temp`. This also means that by the time the red code block in thread two runs, `timeline->fences` is an empty list and the loop finishes quickly and proceeds to `dma_fence_free`. In short, as long as I add a large enough number of `dma_fences` to `timeline->fences`, I can create a large race window when `kgsl_ioctl_timeline_destroy` is moving the `dma_fences` in `timeline->fences` to `temp`. As long as I reduce the last refcount of the last `dma_fence` in `timeline->fences` within this window, I’m able to trigger the UAF bug.

Mitigations
-----------

While triggering the bug is not too difficult, exploiting it, on the other hand, is a completely different matter. The device that I used for testing this bug and for developing the exploit is a Samsung Galaxy Z Flip3. The latest Samsung devices running kernel version 5.x probably have the most mitigations in place, even more so than the Google Pixels. While older devices running kernel 4.x often have mitigations such as the [kCFI](https://source.android.com/devices/tech/debug/kcfi) (Kernel Control Flow Integrity) and [variable initialization](https://android-developers.googleblog.com/2020/06/system-hardening-in-android-11.html) switched off, all those features are switched on in the 5.x kernel branch, and on top of that, there is also the [Samsung RKP (Realtime Kernel Protection)](https://www.samsungknox.com/en) that protects various memory area, such as kernel code and process credentials, making it difficult to execute arbitrary code even when arbitrary memory read and write is achieved. In this section, I’ll briefly explain how those mitigations affect the exploit.

### kCFI

The kCFI is arguably the mitigation that takes the most effort to bypass, especially when used in conjunction with the Samsung hypervisor which protects many important memory areas in the kernel. The kCFI prevents hijacking of control flow by limiting the locations where a dynamic callsite can jump to using function signatures. For example, in the current vulnerability, after the `dma_fence` is freed, the function `dma_fence_signal_locked` is called:

```
long kgsl_ioctl_timeline_destroy(struct kgsl_device_private *dev_priv,
        unsigned int cmd, void *data)
{
    ...
    spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) {
        dma_fence_set_error(&fence->base, -ENOENT);
        dma_fence_signal_locked(&fence->base);       //<---- free'd fence is used
        dma_fence_put(&fence->base);
    }
    spin_unlock_irq(&timeline->lock);
    ...
}
```

The function `dma_fence_signal_locked` then invokes a function `cur->func` that is an element inside the `fence->cb_list` list.

```
int dma_fence_signal_locked(struct dma_fence *fence)
{
    ...
    list_for_each_entry_safe(cur, tmp, &cb_list, node) {
        INIT_LIST_HEAD(&cur->node);
        cur->func(fence, cur);
    }
    ...
}
```

Without kCFI, the now free’d `fence` object can be replaced with a fake object, meaning that `cb_list` and its elements, hence `func`, can all be faked, giving a ready to use primitive to call an arbitrary function with both its first and second arguments pointing to controlled data (`fence` and `cur` can both be faked). The exploit would have been very easy once KASLR was defeated (for example, with a separate bug to leak kernel addresses like in [this exploit](https://securitylab.github.com/research/qualcomm_npu/)). However, because of kCFI, `func` can now only be replaced by functions that have the type `[dma_fence_func_t](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/include/linux/dma-fence.h#L118)`, which greatly limits the use of this primitive.

While in the past, I’ve written about how easy it is to [bypass](https://securitylab.github.com/research/qualcomm_npu/) Samsung’s control flow integrity checks (JOPP, jump-oriented programming prevention), there is no easy way round kCFI. One common way to bypass kCFI is to use a double free to hijack the freelist and then apply the [Kernel Space Mirroring Attack (KSMA)](https://i.blackhat.com/briefings/asia/2018/asia-18-WANG-KSMA-Breaking-Android-kernel-isolation-and-Rooting-with-ARM-MMU-features.pdf). This was used a number of times, for example, in [Three dark clouds over the Android kernel](https://github.com/2freeman/Slides/blob/main/PoC-2020-Three%20Dark%20clouds%20over%20the%20Android%20kernel.pdf) of Jun Yao, [Typhoon Mangkhut: One-click remote universal root formed with two vulnerabilities](https://i.blackhat.com/USA21/Wednesday-Handouts/us-21-Typhoon-Mangkhut-One-Click-Remote-Universal-Root-Formed-With-Two-Vulnerabilities.pdf) of Hongli Han, Rong Jian, Xiaodong Wang and Peng Zhou.

While the current bug also gives me a double free primitive when `dma_fence_put` is called after `fence` is freed:

```
long kgsl_ioctl_timeline_destroy(struct kgsl_device_private *dev_priv,
        unsigned int cmd, void *data)
{
    ...
    spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) {
        dma_fence_set_error(&fence->base, -ENOENT);
        dma_fence_signal_locked(&fence->base); 
        dma_fence_put(&fence->base);       //<----- free'd fence can be freed again
    }
    spin_unlock_irq(&timeline->lock);
    ...
}
```

The above decreases the refcount of the fake `fence` object, which I can control to make it one, so that the fake fence gets freed again. This, however, does not allow me to apply KSMA as it would require overwriting of the `[swapper_pg_dir](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/arch/arm64/include/asm/pgtable.h#L465)` data structure, which is protected by the Samsung hypervisor.

### Variable initialization

From Android 11 onwards, the kernel can enable automatic variable initialization by enabling various kernel build flags. The following, for example, is taken from the build configuration of the Z Flip3:

```
# Memory initialization
#
CONFIG_CC_HAS_AUTO_VAR_INIT_PATTERN=y
CONFIG_CC_HAS_AUTO_VAR_INIT_ZERO=y
# CONFIG_INIT_STACK_NONE is not set
# CONFIG_INIT_STACK_ALL_PATTERN is not set
CONFIG_INIT_STACK_ALL_ZERO=y
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
# CONFIG_INIT_ON_FREE_DEFAULT_ON is not set
# end of Memory initialization
```

While the feature is available since Android 11, many devices running kernel branch 4.x do not have these enabled. On the other hand, devices running kernel 5.x seem to have these enabled. Apart from the obvious uninitialized variables vulnerabilities that this feature prevents, it also makes it harder for object replacement. In particular, it is no longer possible to perform partial object replacement, in which only the first bytes of the object are replaced, while the rest of the object remains valid. So for example, the type of heap spray technique under the section, “Spraying the heap” in “[Mitigations are attack surface, too](https://googleprojectzero.blogspot.com/2020/02/mitigations-are-attack-surface-too.html)” by Jann Horn is no longer possible with automatic variable initialization. In the context of the current bug, this mitigation limits the heap spray options that I have, and I’ll explain more as we go through the exploit.

### kfree_rcu

This isn’t a security mitigation at all, but it is nevertheless interesting to mention it here, because it has a similar effect to some UAF mitigations that had been proposed. The UAF `fence` object in this bug is freed when `[dma_fence_free](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/drivers/dma-buf/dma-fence.c#L270)` is called, which, instead of the normal `kfree`, uses `kfree_rcu`. In short, `kfree_rcu` does not free an object immediately, but rather schedules it to be freed when certain criteria are met. This acts somewhat like a delayed free that introduces an uncertainty in the time when the object is freed. Interestingly, this effect is quite similar to the UAF mitigation that is used in the [Scudo](https://source.android.com/devices/tech/debug/scudo) allocator (default allocator of Android user space processes), which quarantines the free’d objects before actually freeing them to introduce uncertainty. A similar proposal has been suggested for the [linux kernel](https://www.openwall.com/lists/kernel-hardening/2020/08/13/7) (but was rejected later). Apart from introducing uncertainty in the object replacement, a delayed free may also cause problems for UAF with tight race windows. So, on the face of it, the use of `kfree_rcu` would be rather problematic for exploiting the current bug. However, with many primitives to manipulate the size of a race window, such as the ones detailed in [Racing against the clock—hitting a tiny kernel race window](https://googleprojectzero.blogspot.com/2022/03/racing-against-clock-hitting-tiny.html) and an older technique in [Exploiting race conditions on [ancient] Linux](https://static.sched.com/hosted_files/lsseu2019/04/LSSEU2019%20-%20Exploiting%20race%20conditions%20on%20Linux.pdf), (both by Jann Horn, the older technique is used for exploiting the current bug) any tight race window can be made large enough to allow for the delay caused by `kfree_rcu`, and the subsequent object replacement. As for the uncertainty, that does not seem to cause a very big problem either. In exploiting this bug, I actually had to perform object replacement with `kfree_rcu` twice, the second time without even knowing on which CPU core the free is going to happen, and yet even with this and all the other moving parts, a rather unoptimized exploit still runs at a reasonable reliability (~70%) on the device tested. While I believe that the second object replacement with `kfree_rcu` (where the CPU that frees the object is uncertain) is probably the main source of unreliability, I’d attribute that reliability loss more to the lack of CPU knowledge rather than to the delayed free. In my opinion, a delayed free may not be a very effective UAF mitigation when there are primitives that allow the scheduler to be manipulated.

### Samsung RKP (Realtime Kernel Protection)

The Samsung RKP protects various parts of the memory from being written to. This prevents processes from overwriting their own credentials to become root, as well as protecting SELinux settings from being overwritten. It also prevents kernel code regions and other important objects, such as kernel page tables, from being overwritten. In practice, though, once arbitrary kernel memory read and write (subject to RKP restrictions) is achieved, there are ways to bypass these restrictions. For example, SELinux rules can be modified by overwriting the avc cache (see, for example, this [exploit](https://github.com/chompie1337/s8_2019_2215_poc/blob/master/poc/selinux_bypass.c) by Valentina Palmiotti), while gaining root can be done by [hijacking other processes that run as root](https://securitylab.github.com/research/qualcomm_npu/). In the context of the current bug, the Samsung RKP mostly works with kCFI to prevent arbitrary functions from being called.

In this post, I’ll exploit the bug with all these mitigations enabled.

Exploiting the bug
------------------

I’ll now start going through the exploit of the bug. It is a fairly typical use-after-free bug that involves a race condition and perhaps reasonably strong primitives with both the possibility of arbitrary function call and double free, which is not that uncommon. Apart from that, this is a typical bug, just like many other UAF found in the kernel. So, it seems fitting to use this bug to gauge how these mitigations affect the development of a standard UAF exploit.

### Adding `dma_fence` to `timeline->fences`

In the section, “The vulnerability,” I explained that the bug relies on having `dma_fence` objects added to the `fences` list in a `kgsl_timeline` object, which then have their refcount decreased to zero while the `kgsl_timeline` is being destroyed. There are two options to add `dma_fence` objects to a `kgsl_timeline`, the first is to use `IOCTL_KGSL_TIMELINE_FENCE_GET`:

```
long kgsl_ioctl_timeline_fence_get(struct kgsl_device_private *dev_priv,
        unsigned int cmd, void *data)
{
    ...
    timeline = kgsl_timeline_by_id(device, param->timeline);
    ...
    fence = kgsl_timeline_fence_alloc(timeline, param->seqno); //<----- dma_fence created and added to timeline
    ...
    sync_file = sync_file_create(fence);
    if (sync_file) {
        fd_install(fd, sync_file->file);
        param->handle = fd;
    }
    ...
}
```

This will create a `dma_fence` with `kgsl_timeline_fence_alloc` and add it to the `timeline`. The caller then gets a file descriptor for a `sync_file` that corresponds to the `dma_fence`. When the `sync_file` is closed, the refcount of `dma_fence` is decreased to zero.

The second option is to use `IOCTL_KGSL_TIMELINE_WAIT`:

```
long kgsl_ioctl_timeline_wait(struct kgsl_device_private *dev_priv,
        unsigned int cmd, void *data)
{
    ...
    fence = kgsl_timelines_to_fence_array(device, param->timelines,
        param->count, param->timelines_size,
        (param->flags == KGSL_TIMELINE_WAIT_ANY));     //<------ dma_fence created and added to timeline
    ...
    if (!timeout)
        ret = dma_fence_is_signaled(fence) ? 0 : -EBUSY;
    else {
        ret = dma_fence_wait_timeout(fence, true, timeout);   //<----- 1.
        ...
    }

    dma_fence_put(fence);
    ...
}
```

This will create `dma_fence` objects using `kgsl_timelines_to_fence_array` and add them to the `timeline`. If a `timeout` value is specified, then the call will enter `dma_fence_wait_timeout` (path labeled 1), which will wait until either the timeout expires or when the thread receives an interrupt. After `dma_fence_wait_timeout` finishes, `dma_fence_put` is called to reduce the refcount of the `dma_fence` to zero. So, by specifying a large timeout, `dma_fence_wait_timeout` will block until it receives an interrupt, which will then free the `dma_fence` that was added to the `timeline`.

While `IOCTL_KGSL_TIMELINE_FENCE_GET` may seem easier to use and control at first glance, in practice, the overhead incurred by closing the `sync_file` makes the timing for destruction of the `dma_fence` less reliable. So, for the exploit, I use `IOCTL_KGSL_TIMELINE_FENCE_GET` to create and add persistent `dma_fence` objects to fill the `timeline->fences` list to enlarge the race window, while the last `dma_fence` object that is used for the UAF bug is added using `IOCTL_KGSL_TIMELINE_WAIT` and which gets freed when I send an interrupt signal to the thread that calls `IOCTL_KGSL_TIMELINE_WAIT`.

### Widening the tiny race window

To recap, in order to exploit the vulnerability, I need to remove the refcount of a `dma_fence` in the `fences` list of a `kgsl_timeline` within the first race window labeled in the following code block:

```
long kgsl_ioctl_timeline_destroy(struct kgsl_device_private *dev_priv,
        unsigned int cmd, void *data)
{
    //BEGIN OF FIRST RACE WINDOW
    spin_lock(&timeline->fence_lock);
    list_for_each_entry_safe(fence, tmp, &timeline->fences, node)
        dma_fence_get(&fence->base);
    list_replace_init(&timeline->fences, &temp);
    spin_unlock(&timeline->fence_lock);
    //END OF FIRST RACE WINDOW
    //BEGIN OF SECOND RACE WINDOW
    spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) {
        dma_fence_set_error(&fence->base, -ENOENT);
        dma_fence_signal_locked(&fence->base);
        dma_fence_put(&fence->base);
    }
    spin_unlock_irq(&timeline->lock);
    //END OF SECOND RACE WINDOW
    ...
}
```

As explained before, the first race window can be enlarged by adding a large number of `dma_fence` objects to `timeline->fences`, which makes it easy to trigger the decrease of refcount within this window. However, to exploit the bug, the following [code](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/drivers/gpu/msm/kgsl_timeline.c#L172), as well as the object replacement, must be completed before the end of the second race window:

```
spin_lock_irqsave(&timeline->fence_lock, flags);
    list_for_each_entry_safe(cur, temp, &timeline->fences, node) {
        if (f != cur)
            continue;
        list_del_init(&f->node);
        break;
    }
    spin_unlock_irqrestore(&timeline->fence_lock, flags);
    trace_kgsl_timeline_fence_release(f->timeline->id, fence->seqno);
    kgsl_timeline_put(f->timeline);
    dma_fence_free(fence);
```

As explained before, because of the `spin_lock`, the above cannot start until the first race window ends, but by the time this code is run, `timeline->fences` has been emptied, so the loop will be quick to run. However, since `dma_fence_free` uses `kfree_rcu`, the actual freeing of `fence` is delayed. This makes it impossible to replace the free’d fence before the second race window finishes, unless we manipulate the scheduler. To do so, I’ll use a technique in “[Exploiting race conditions on [ancient] Linux](https://static.sched.com/hosted_files/lsseu2019/04/LSSEU2019%20-%20Exploiting%20race%20conditions%20on%20Linux.pdf)” that I also used in another [Android exploit](https://securitylab.github.com/research/one_day_short_of_a_fullchain_android/) to widen this race window.

I’ll recap the essence of the technique here for readers who are not familiar with it.

To ensure that each task (thread or process) has a fair share of the CPU time, the linux kernel scheduler can interrupt a running task and put it on hold, so that another task can be run. This kind of interruption and stopping of a task is called preemption (where the interrupted task is preempted). A task can also put itself on hold to allow another task to run, such as when it is waiting for some I/O input, or when it calls `sched_yield()`. In this case, we say that the task is voluntarily preempted. Preemption can happen inside syscalls such as ioctl calls as well, and on Android, tasks can be preempted except in some critical regions (e.g. holding a spinlock). This behavior can be manipulated by using CPU affinity and task priorities.

By default, a task is run with the priority `SCHED_NORMAL`, but a lower priority `SCHED_IDLE` can also be set using the `sched_setscheduler` call (or `pthread_setschedparam` for threads). Furthermore, it can also be pinned to a CPU with `sched_setaffinity`, which would only allow it to run on a specific CPU. By pinning two tasks, one with `SCHED_NORMAL` priority and the other with `SCHED_IDLE` priority to the same CPU, it is possible to control the timing of the preemption as follows.

1.  First have the `SCHED_NORMAL` task perform a syscall that would cause it to pause and wait. For example, it can read from a pipe with no data coming in from the other end, then it would wait for more data and voluntarily preempt itself, so that the `SCHED_IDLE` task can run.
2.  As the `SCHED_IDLE` task is running, send some data to the pipe that the `SCHED_NORMAL` task had been waiting on. This will wake up the `SCHED_NORMAL` task and cause it to preempt the `SCHED_IDLE` task, and because of the task priority, the `SCHED_IDLE` task will be preempted and put on hold.
3.  The `SCHED_NORMAL` task can then run a busy loop to keep the `SCHED_IDLE` task from waking up.

In our case, the object replacement sequence goes as follows:

1.  Run `IOCTL_KGSL_TIMELINE_WAIT` on a thread to add `dma_fence` objects to a `kgsl_timeline`. Set the timeout to a large value and use `sched_setaffinity` to pin this task to a CPU, call it `SPRAY_CPU`. Once the `dma_fence` object is added, the task will then become idle until it receives an interrupt.
2.  Set up a `SCHED_NORMAL` task and pin it to another CPU (`DESTROY_CPU`) that listens to an empty pipe. This will cause this task to become idle initially and allow `DESTROY_CPU` to run a lower priority task. Once the empty pipe receives some data, this task then will run a busy loop.
3.  Set up a `SCHED_IDLE` task on `DESTROY_CPU` which will run `IOCTL_KGSL_TIMELINE_DESTROY` to destroy the timeline where the `dma_fence` is added in step one. As the task set up in step two is waiting for a response to an empty pipe, `DESTROY_CPU` will run this task first.
4.  Send an interrupt to the task running `IOCTL_KGSL_TIMELINE_WAIT`. The task will then unblock and free the `dma_fence` while `IOCTL_KGSL_TIMELINE_DESTROY` is running within the first race window.
5.  Write to the empty pipe that the `SCHED_NORMAL` task is listening to. This will cause the `SCHED_NORMAL` task to preempt the `SCHED_IDLE` task. Once it has successfully preempted the task, `DESTROY_CPU` will run the busy loop, causing the `SCHED_IDLE` task to be put on hold.
6.  As the `SCHED_IDLE` task running `IOCTL_KGSL_TIMELINE_DESTROY` is put on hold, there is now enough time to overcome the delay introduced by `kfree_rcu` and allow the `dma_fence` in step four to be freed and replaced. After that, I can resume `IOCTL_KGSL_TIMELINE_DESTROY` so that the subsequent operations will be performed on the now free’d and replaced `dma_fence` object.

One caveat here is that, because preemption cannot happen while a thread is holding a `spinlock`, so `IOCTL_KGSL_TIMELINE_DESTROY` can only be preempted during the window between spinlocks (marked by the comment below):

```
long kgsl_ioctl_timeline_destroy(struct kgsl_device_private *dev_priv,
        unsigned int cmd, void *data)
{
    spin_lock(&timeline->fence_lock);
    list_for_each_entry_safe(fence, tmp, &timeline->fences, node)
      ...
    spin_unlock(&timeline->fence_lock);
    //Preemption window
    spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) {
    ...
    }
    spin_unlock_irq(&timeline->lock);
    ...
}
```

Although the preemption window in the above appears to be very small, in practice, as long as the `SCHED_NORMAL` task tries to preempt the `SCHED_IDLE` task running `IOCTL_KGSL_TIMELINE_DESTROY` while the first `spinlock` is held, preemption will happen as soon as the `spinlock` is released, making it much easier to succeed in preempting `IOCTL_KGSL_TIMELINE_DESTROY` at the right time.

The following figure illustrates what happens in an ideal world, with red blocks indicating regions that hold a `spinlock` and are therefore not possible to preempt, and dotted lines indicating tasks that are idle.

![](https://github.blog/wp-content/uploads/2022/06/Kernel-2.png?resize=875%2C536)

The following figure illustrates what happens in the real world:

![](https://github.blog/wp-content/uploads/2022/06/Kernel-3.png?resize=1024%2C775)

For object replacement, I’ll use `[sendmsg](https://blog.lexfo.fr/cve-2017-11176-linux-kernel-exploitation-part3.html)`, which is a standard way to replace free’d objects in the linux kernel with controlled data. As the method is fairly standard, I won’t give the details here, but refer readers to the link above. From now on, I’ll assume that the free’d `dma_fence` object is replaced by arbitrary data. (There are some restrictions in the first 12 bytes using this method, but that does not affect our exploit.)

Assuming that the free’d `dma_fence` object can be replaced with arbitrary data, let’s take a look at how this fake object is used. After the `dma_fence` is replaced, it is then used in `kgsl_ioctl_timeline_destroy` as follows:

```
spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) {
        dma_fence_set_error(&fence->base, -ENOENT);
        dma_fence_signal_locked(&fence->base);
        dma_fence_put(&fence->base);
    }
    spin_unlock_irq(&timeline->lock);
```

Three different functions, `dma_fence_set_error`, `dma_fence_signal_locked` and `dma_fence_put` will be called with the argument `fence`. The function `dma_fence_set_error` will write an error code to the `fence` object, which may be useful with a suitable object replacement, but not for `sendmsg` object replacements and I’ll not be investigating this possibility here. The function `dma_fence_signal_locked` does the following:

```
int dma_fence_signal_locked(struct dma_fence *fence)
{
    ...
    if (unlikely(test_and_set_bit(DMA_FENCE_FLAG_SIGNALED_BIT,   //<-- 1.
                      &fence->flags)))
        return -EINVAL;

    /* Stash the cb_list before replacing it with the timestamp */
    list_replace(&fence->cb_list, &cb_list);                    //<-- 2.
    ...
    list_for_each_entry_safe(cur, tmp, &cb_list, node) {        //<-- 3.
        INIT_LIST_HEAD(&cur->node);
        cur->func(fence, cur);
    }

    return 0;
}
```

It first checks `fence->flags` (1. in the above): if the `DMA_FENCE_FLAG_SIGNALED_BIT` flag is set, then the `fence` has been signaled, and the function exits early. If the `fence` has not been signaled, then `list_replace` is called to remove objects in `fence->cb_list` and place them in a temporary `cb_list` (2. in the above). After that, functions stored in `cb_list` are called (3. above). As explained in the section, “kCFI” because of the CFI mitigation, this will only allow me to call functions of a certain type; besides, at this stage I have no knowledge of function addresses, so I’m most likely just going to crash the kernel if I reach this path. So, at this stage, I have little choice but to set the `DMA_FENCE_FLAG_SIGNALED_BIT`flag in my fake object so that `dma_fence_signal_locked` exits early.

This leaves me the `dma_fence_put` function, which decreases the refcount of `fence` and calls `dma_fence_release` if the refcount reaches zero:

```
void dma_fence_release(struct kref *kref)
{
    ...
    if (fence->ops->release)
        fence->ops->release(fence);
    else
        dma_fence_free(fence);
}
```

If `dma_fence_release` is called, then eventually it’ll check the `fence->ops` and call `fence->ops->release`. This gives me two problems: First, `fence->ops` needs to point to valid memory, otherwise the dereference will fail, and even if the dereference succeeds, `fence->ops->release` either needs to be zero, or it has to be the address of a function of an appropriate type.

All these present me with two choices. I can either follow the standard path: try to replace the `fence` object with another object or try to make use of the limited write primitives that `dma_fence_put` and `dma_fence_set_error` offer me, while hoping that I can still control the `flags` and `refcount` fields to avoid `dma_fence_signal_locked` or `dma_fence_release` crashing the kernel.

Or, I can try something else.

The ultimate fake object store
------------------------------

While exploiting [another bug](https://securitylab.github.com/research/one_day_short_of_a_fullchain_android/), I came across the Software Input Output Translation Lookaside Buffer (SWIOTLB), which is a memory region that is allocated at a very early stage during boot time. As such, the physical address of the SWIOTLB is very much fixed and depends only on the hardware configuration. Moreover, as this memory is in the “low memory” region (Android devices do not seem to have a “high memory” region) and not in the kernel image, the virtual address is simply the physical address with a fixed offset (readers who are interested in the details can, for example, follow the implementation of the `kmap` function):

```
#define __virt_to_phys_nodebug(x) ({                   \
    phys_addr_t __x = (phys_addr_t)(__tag_reset(x));        \
    __is_lm_address(__x) ? __lm_to_phys(__x) : __kimg_to_phys(__x); \
})
#define __is_lm_address(addr)  (!(((u64)addr) & BIT(vabits_actual - 1)))

#define __lm_to_phys(addr) (((addr) + physvirt_offset))
```

The above definitions are from `[arch/arm64/include/asm/memory.h](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/arch/arm64/include/asm/memory.h)`, which is the relevant implementation for Android. The variable `physvirt_offset` used for translating the address is a fixed constant set in `[arm64_memblock_init](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/arch/arm64/mm/init.c#L517)`:

```
void __init arm64_memblock_init(void)
{...
    memstart_addr = round_down(memblock_start_of_DRAM(),
                   ARM64_MEMSTART_ALIGN);
    physvirt_offset = PHYS_OFFSET - PAGE_OFFSET;

 ...
}
```

On top of that, the memory in the SWIOTLB can be accessed via the `adsp` driver that is reachable from an untrusted app, so this seems to be a good place to store fake objects and redirect fake pointers to. However, in the 5.x version of the kernel, the SWIOTLB is only allocated when the kernel is compiled with the `CONFIG_DMA_ZONE32` flag, which is not the case for our device.

There is, however, something better. The fact that the early allocation of SWIOTLB gives it a predictable address prompted me to inspect the boot log to see if there are other regions of memory that are allocated early during the boot, and it turns out that there are indeed other memory regions that are allocated very early during the boot.

```
<6>[    0.000000] [0:        swapper:    0]  Reserved memory: created CMA memory pool at 0x00000000f2800000, size 212 MiB
<6>[    0.000000] [0:        swapper:    0]  OF: reserved mem: initialized node secure_display_region, compatible id shared-dma-pool
...
<6>[    0.000000] [0:        swapper:    0]  OF: reserved mem: initialized node user_contig_region, compatible id shared-dma-pool
<6>[    0.000000] [0:        swapper:    0]  Reserved memory: created CMA memory pool at 0x00000000f0c00000, size 12 MiB

<6>[    0.578613] [7:      swapper/0:    1]  platform soc:qcom,ion:qcom,ion-heap@22: assigned reserved memory node sdsp_region
...
<6>[    0.578829] [7:      swapper/0:    1]  platform soc:qcom,ion:qcom,ion-heap@26: assigned reserved memory node user_contig_region
...
```

The `Reserved memory` regions in the above seem to be the memory pools that are used for allocating ion buffers.

On Android, the `[ion_allocator](https://lwn.net/Articles/480055/)` is used to allocate memory regions used for DMA (direct memory access) that allows kernel drivers and userspace processes to share the same underlying memory. The ion allocator is accessible by an untrusted app via the `/dev/ion` file, and the `ION_IOC_ALLOC` ioctl can be used to allocate an ion buffer. The ioctl returns a new file descriptor to the user, which can then be used in the `mmap` syscall to map the backing store of the ion buffer to userspace.

One particular reason for using the ion buffers is that the user can request memory that has contiguous physical addresses. This is particularly important as some devices (as in devices on the hardware, not the phone itself) access physical memory directly and having contiguous memory addresses can greatly improve the performance of such memory accesses, while some devices cannot handle non contiguous physical memory.

Similar to SWIOTLB, in order to ensure a region of contiguous physical memory with the requested size is available, the ion driver allocates these memory regions very early in the boot and uses them as memory pools (“carved out regions”), which are then used to allocate ion buffers later on when requested. Not all memory pools in the ion device are contiguous memory (for example, the general purpose “system heap” may not be a physically contiguous region), but the user can specify the `heap_id_mask` when using `ION_IOC_ALLOC` to specify the ion heap with specific properties (for example, contiguous physical memory).

The fact that these memory pools are allocated at such an early stage means that their addresses are predictable and depend only on the configuration of the hardware (device tree, available memory, start of memory address, various boot parameters, etc.). This, in particular, means that if I allocate an ion buffer from a rarely used memory pool using `ION_IOC_ALLOC`, the buffer will most likely be allocated at a predictable address. If I then use `mmap` to map the buffer to userspace, I’ll be able to access the memory at this predictable address at any time!

After some experimentation, it seems that the `user_contig_region` is almost never used and I was able to map the entire region to userspace everytime. So in the exploit, I used this memory pool and assumed that I can allocate the entire region to keep it simple. (It would be easy to modify the exploit to accommodate the case where part of the region is not available without compromising reliability.)

Now that I am able to put controlled data at a predictable address, I can resolve the problem I encountered previously in the exploit. Recall that, when `dma_fence_release` is called on my fake `fence` object:

```
void dma_fence_release(struct kref *kref)
{
    ...
    if (fence->ops->release)
        fence->ops->release(fence);
    else
        dma_fence_free(fence);
}
```

I had a problem where I needed `fence->ops` to point to a valid address that contains all zeros, so that `fence->ops->release` will not be called (as I do not have a valid function address that matches the signature of `fence->ops->release` at this stage and taking this path would crash the kernel)

With the ion buffer at a predictable address, I can simply fill it with zero and have `fence->ops` point there. This will ensure the path `dma_fence_free` is taken, which will then free my fake object, giving me a double free primitive while preventing a kernel crash. Before proceeding to exploit this double free primitive, there is, however, another issue that needs resolving first.

Escaping an infinite loop
-------------------------

Recall that, in the `kgsl_ioctl_timeline_destroy` function, after the `fence` object is destroyed and replaced, the following loop is executed:

```
spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) {
        dma_fence_set_error(&fence->base, -ENOENT);
        dma_fence_signal_locked(&fence->base);
        dma_fence_put(&fence->base);
    }
    spin_unlock_irq(&timeline->lock);
```

The `list_for_each_entry_safe` will first take the `next` pointer from the `list_head` `temp` to find the first `fence` entry in the list, and then iterate by following the `next` pointer in `fence->node` until the `next` entry points back to `temp` again. If the `next` entry does not point back to `temp`, then the loop will just carry on following the `next` pointer indefinitely. This is a place where variable initialization makes life more difficult. Look at the layout of `kgsl_timeline_fence`, which embeds a `dma_fence` object that is added to the `kgsl_timeline`:

```
struct kgsl_timeline_fence {
    struct dma_fence base;
    struct kgsl_timeline *timeline;
    struct list_head node;
};
```

I can see that the `node` field is the last field in `kgsl_timeline_fence`, while to construct the exploit, I only need to replace `base` with controlled data. The above problem would have been solved easily with partial object replacement. Without automatic variable initialization, if I only replace the free’d `kgsl_timeline_fence` with an object that is of the size of a `dma_fence`, then the fields `timeline` and `node` would remain intact and contain valid data. This would both cause the `next` pointer in `node` to be valid and allow the loop in `kgsl_ioctl_timeline_destroy` to exit normally. However, with automatic variable initialization, even if I replace the free’d `kgsl_timeline_fence` object with a smaller object, the entire memory chunk would be set to zero first, erasing both `kgsl_timeline` and `node`, meaning that I now have to fake the `node` field so that:

1.  The `next` pointer points to a valid address to avoid an immediate crash, in fact, more than that, it needs to point to an object that is another fake `kgsl_timeline_fence` that can be operated by the functions in the loop (`dma_fence_set_error`, `dma_fence_signal_locked` and `dma_fence_put`) without crashing. That means more fake objects need to be crafted.
2.  One of the `next` pointers in these fake `kgsl_timeline_fence` objects points back to the `temp` list to exit the loop, which is a stack allocated variable.

The first requirement is not too hard, as I can now use the ion buffer to create these fake `kgsl_timeline_fence` objects. The second requirement, however, is much harder.

On the face of it, this obstacle may seem more like an aesthetic issue rather than a real problem. After all, I can create the fake objects so that the list becomes circular within the fake `kgsl_timeline_fence` objects:

![](https://github.blog/wp-content/uploads/2022/06/Kernel-4.png?resize=623%2C525)

This would cause an infinite loop and hold up a CPU. While it is ugly, the fake objects should take care of the dereferencing issues and avoid crashes, so it may not be a fatal issue after all. Unfortunately, as the loop runs inside a `spinlock`, after running for a short while, it seems that the watchdog will flag it as a CPU hogging issue and trigger a kernel panic. So, I do need to find a way to exit the loop, and exit it quickly.

Let’s take a step back and take a look at the function `dma_fence_signal_locked`:

```
int dma_fence_signal_locked(struct dma_fence *fence)
{
    ...
    struct list_head cb_list;
    ...
    /* Stash the cb_list before replacing it with the timestamp */
    list_replace(&fence->cb_list, &cb_list);             //<-- 1.
    ...
    list_for_each_entry_safe(cur, tmp, &cb_list, node) { //<-- 2.
        INIT_LIST_HEAD(&cur->node);
        cur->func(fence, cur);
    }

    return 0;
}
```

This function will be run for each of the fake `dma_fence` in the list `temp` (the original free’d and replaced `dma_fence`, plus the ones that it links to in the ion buffer). As mentioned before, if the code at 2. in the above is run, then the kernel will probably crash because I cannot provide a valid `func`, so I still would like to avoid running that path.

In order to be able to run this code but not the loop code in 2. above, I need to initialize `fence.cb_list` to be an empty list, so that its `next` and `prev` both point to itself. This is not possible with the initial fake `dma_fence` that was free’d by the vulnerability, because the address of `fence` and hence `fence.cb_list` is unknown, so I had to avoid the `list_replace` code altogether for this first fake object. However, because the subsequent fake `dma_fence` objects that are linked to it are in an ion buffer with a known address, I can now create an empty `cb_list` for these objects, setting both the `next` and `prev` pointers to the addresses of the `fence.cb_list` field. The function `list_replace` will then do the following:

```
static inline void list_replace(struct list_head *old,
                struct list_head *new)
{
    //old->next = &(fence->cb_list)
    new->next = old->next;
    //new->next = &(fence->cb_list) => fence->cb_list.prev = &cb_list
    new->next->prev = new;
    //new->prev = fence->cb_list.prev => &cb_list
    new->prev = old->prev;
    //&cb_list->next = &cb_list
    new->prev->next = new;
}
```

As we can see, after `list_replace`, the address of the stack variable `cb_list` has been written to `fence->cb_list.prev`, which is somewhere in the ion buffer. As the ion buffer is mapped to user space, I can simply read this address by polling the ion buffer. As `dma_fence_signal_locked` is run inside `kgsl_ioctl_timeline_destroy` after the stack variable `temp` is allocated:

```
long kgsl_ioctl_timeline_destroy(struct kgsl_device_private *dev_priv,
        unsigned int cmd, void *data)
{
    ...
    struct list_head temp;
    ...


    spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) {
        dma_fence_set_error(&fence->base, -ENOENT);
        //cb_list, is a stack variable allocated inside `dma_fence_signal_locked`
        dma_fence_signal_locked(&fence->base);
        dma_fence_put(&fence->base);
    }
    spin_unlock_irq(&timeline->lock);
    ...
}
```

Having the address of `cb_list` allows me to compute the address of `temp`, (which will be at a fixed offset from the address of `cb_list`), so by polling for the address of `cb_list` and then using this to compute the address of `temp` and write it back into the `next` pointer of one of the fake `kgsl_timeline_fence` objects in the ion buffer, I can exit the loop before the watch dog bites.

![](https://github.blog/wp-content/uploads/2022/06/Kernel-5.png?resize=960%2C540)

Hijacking the freelist
----------------------

Now that I am able to avoid kernel crashes, I can continue to exploit the double free primitive mentioned earlier. To recap, once the initial use-after-free vulnerability is triggered and the free’d object is successfully replaced with controlled data using `sendmsg`, the replaced object will be used in the following loop as `fence`:

```
spin_lock_irq(&timeline->lock);
    list_for_each_entry_safe(fence, tmp, &temp, node) {
        dma_fence_set_error(&fence->base, -ENOENT);
        dma_fence_signal_locked(&fence->base);
        dma_fence_put(&fence->base);
    }
    spin_unlock_irq(&timeline->lock);
```

In particular, `dma_fence_put` will reduce the refcount of the fake object and if the refcount reaches zero, it’ll call `dma_fence_free`, which will then free the object with `kfree_rcu`. Since the fake object is in complete control and I was able to resolve various issues that may lead to kernel crashes, I will now assume that this is the code path that is taken and that the fake object will be freed by `kfree_rcu`. By replacing the fake object again with another object, I can then obtain two references to the same object, which I will be able to free at any time using either of these object handles. The general idea is that, when a memory chunk is freed, the freelist pointer, which points to the next free chunk, will be written to the first 8 bytes of the memory chunk. If I free the object from one handle, and then modify the first 8 bytes of this free’d object using another handle, then I can hijack the freelist pointer and have it point to an address of my choice, which is where the next allocation will happen. (This is an overly simplified version of what happens as this is only true when the free and allocation are from the same slab, pages used by the memory allocator—SLUB allocator in this case—to allocate memory chunks, with allocation done via the fast path, but this scenario is not difficult to achieve.)

In order to be able to modify the first 8 bytes of the object after it was allocated, I’ll use the `signalfd` object used in “[Mitigations are attack surface too](https://googleprojectzero.blogspot.com/2020/02/mitigations-are-attack-surface-too.html)”. The `signalfd` syscall allocates an 8 byte object to store a mask for the `signalfd` file, which can be specified by the user with some minor restrictions. The lifetime of the allocated object is tied to the `signalfd` file that is returned to the user and can be controlled easily by closing the file. Moreover, the first 8 bytes in this object can be changed by calling `signalfd` again with a different mask. This makes `signalfd` ideal for my purpose.

To hijack the freelist pointer, I have to do the following:

1.  Trigger the UAF bug and replaced the free’d `dma_fence` object with a fake `dma_fence` object allocated via `sendmsg` such that `dma_fence_free` will be called to free this fake object with `kfree_rcu`.
2.  Spray the heap with `signalfd` to allocate another object at the same address as the `sendmsg` object after it was freed.
3.  Free the `sendmsg` object so that the freelist pointer is written to the mask of the `signalfd` object in step two.
4.  Modify the mask of the `signalfd` object so that the freelist pointer now points to an address of my choice, then spray the heap again to allocate objects at that address.

If I set the address of the freelist pointer to the address of an ion buffer that I control, then subsequent allocations will place objects in the ion buffer, which I can then access and modify at any time. This gives me a very strong primitive in that I can read and modify any object that I allocate. Essentially, I can fake my own kernel heap in a region where I have both read and write access.

![](https://github.blog/wp-content/uploads/2022/06/Kernel-6.png?resize=960%2C540)

The main hurdle to this plan comes from the combination of `kfree_rcu` and the fact that the CPU running `dma_fence_put` will be temporarily trapped in a busy loop after `kfree_rcu` is called. Recall from the previous section that, until I am able to exit the loop by writing the address of the `temp` list to the `next` pointer of one of the fake `kgsl_timeline_fence::node` objects, the loop will be running. This, in particular, means that once `kfree_rcu` is called and `dma_fence_put` is exited, the loop will continue to process the other fake `dma_fence` objects on the CPU that is running `kfree_rcu`. As explained earlier, `kfree_rcu` does not immediately free an object, but rather schedules its removal. Most of the time, the free will actually happen on the same CPU that calls `kfree_rcu`. However, in this case, because the CPU running `kfree_rcu` is kept busy inside a `spinlock` by running the loop, the object will almost certainly not be free’d on that same CPU. Instead, a different CPU will be used to free the object. This causes a problem because the reliability of object replacement depends on the CPU that is used for freeing the object. When an object is freed on a CPU, the memory allocator will place it in a per CPU cache. An allocation that follows immediately on the same CPU will first look for free space in the CPU cache and is most likely going to replace that newly freed object. However, if the allocation happens on a different CPU, then it’ll most likely replace an object in the cache of a different CPU, rather than the newly freed object. Not knowing which CPU is responsible for freeing the object, together with the uncertainty of when the object is freed (because of the delay introduced by `kfree_rcu`) means that it may be difficult to replace the object. In practice, however, I was able to achieve reasonable results on the testing device (>70% success rate) with a rather simple scheme: Simply run a loop that spray objects on each CPU and repeat the spraying in intervals to account for the uncertainty in the timing. There is probably room for improvement here to make the exploit more reliable.

Another slight modification used in the exploit was to also replace the `sendmsg` objects after they are freed with another round of `signalfd` heap spray. This is to ensure that those `sendmsg` objects don’t accidentally get replaced by objects that I don’t control which may interfere with the exploit, as well as to make it easier to identify the actual corrupted object.

Now that I can hijack the freelist and redirect new object allocations to the ion buffer that I can freely access at any time, I need to turn this into an arbitrary memory read and write primitive.

The Device Memory Mirroring Attack
----------------------------------

Kernel drivers often need to map memory to the user space, and as such, there are often structures that contain pointers to the `page` struct or the `sg_table` struct. These structures often hold pointers to pages that would be mapped to user space when, for example, `mmap` is called. This makes them very good corruption targets. For example, the `ion_buffer` object that I have already used is available on all Android devices. It has a `sg_table` struct that contains information about the pages that will get mapped to user space when `mmap` is used.

Apart from being widely available and accessible from untrusted apps, `ion_buffer` objects also solve a few other problems, so in what follows, I’ll use the freelist hijacking primitive above to allocate an `ion_buffer` struct in an ion buffer backing store that I have arbitrary read and write access to. By doing so, I can freely corrupt the data in all of the `ion_buffer` structs that are allocated. To avoid confusion, from now on, I’ll use the term “fake kernel heap” to indicate the ion buffer backing store that I use as the fake kernel heap and `ion_buffer` struct as the structures that I allocate in the fake heap for use as corruption targets.

The general idea here is that, by allocating `ion_buffer` structs in the fake kernel heap, I’ll be able to modify the `ion_buffer` struct and replace its `sg_table` with controlled data. The `sg_table` structure contains a `scatterlist` structure that represents a collection of pages that back the `ion_buffer` structure:

```
struct sg_table {
    struct scatterlist *sgl;    /* the list */
    unsigned int nents;     /* number of mapped entries */
    unsigned int orig_nents;    /* original size of list */
};

struct scatterlist {
    unsigned long   page_link;
    unsigned int    offset;
    unsigned int    length;
    dma_addr_t  dma_address;
#ifdef CONFIG_NEED_SG_DMA_LENGTH
    unsigned int    dma_length;
#endif
};
```

The `page_link` field in the `scatterlist` is an encoded form of a `page` pointer, indicating the actual `page` where the backing store of the `ion_buffer` structure is:

```
static inline struct page *sg_page(struct scatterlist *sg)
{
#ifdef CONFIG_DEBUG_SG
    BUG_ON(sg_is_chain(sg));
#endif
    return (struct page *)((sg)->page_link & ~(SG_CHAIN | SG_END));
}
```

When `mmap` is called, the page encoded by `page_link` will be mapped to user space:

```
int ion_heap_map_user(struct ion_heap *heap, struct ion_buffer *buffer,
              struct vm_area_struct *vma)
{
    struct sg_table *table = buffer->sg_table;
    ...
    for_each_sg(table->sgl, sg, table->nents, i) {
        struct page *page = sg_page(sg);
        ...
        //Maps pages to user space
        ret = remap_pfn_range(vma, addr, page_to_pfn(page), len,
                      vma->vm_page_prot);
        ...
    }

    return 0;
}
```

As the `page` pointer is simply a logical shift of the physical address of the page followed by a constant linear offset (see the definition of [phys_to_page](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/arch/arm64/include/asm/memory.h#L285)), being able to control `page_link` allows me to map an arbitrary page to user space. For many devices, this would be sufficient to achieve arbitrary kernel memory read and write because the kernel image is mapped at a fixed physical address (KASLR randomizes the _virtual_ address offset from this fixed physical address), so there is no need to worry about KASLR when working with physical addresses.

![](https://github.blog/wp-content/uploads/2022/06/Kernel-7.png?resize=960%2C540)

Samsung devices, however, do KASLR differently. Instead of mapping the kernel image to a fixed physical address, the physical address of the kernel image is randomized (strictly speaking, the intermediate physical address as perceived by the kernel, which is not the real physical address but rather a virtual address given by the hypervisor) instead. So in our case, I still need to leak an address to defeat KASLR. With the fake kernel heap, however, this is fairly easy to achieve. An `ion_buffer` object contains a pointer to an `ion_heap`, which is responsible for allocating the backing stores for the `ion_buffer`:

```
struct ion_buffer {
    struct list_head list;
    struct ion_heap *heap;
    ...
};
```

While the `ion_heap` is not an global object in the kernel image, each `ion_heap` contains an `ion_heap_ops` field, which points to the corresponding “vtable” of the specific `ion_heap` object:

```
struct ion_heap {
    struct plist_node node;
    enum ion_heap_type type;
    struct ion_heap_ops *ops;
    ...
}
```

The `ops` field in the above is a global object in the kernel image. If I can read `ion_buffer->heap->ops`, then I’m also able to get an address to defeat KASLR and translate addresses in the kernel image to physical addresses. This can be done as follows:

1) First locate the `ion_buffer` struct in the fake kernel heap. This can be done using the `flags` field in the `ion_buffer`:

```
struct ion_buffer {
    struct list_head list;
    struct ion_heap *heap;
    unsigned long flags;
    ...
```

which is a 4 byte value passed from the parameters of the `ION_IOC_ALLOC` ioctl when the `ion_buffer` is created. I can set these to specific “magic” values and search for them in the fake kernel heap.

2) Once the `ion_buffer` struct is located, read its `heap` pointer. This will be a virtual address in the low memory area outside of the kernel image, and as such, its physical address can be obtained by applying a constant offset.

3) Once the physical address of the corresponding `ion_heap` object is obtained, modify the `sg_table` of the `ion_buffer` so that its backing store points to the page containing the `ion_heap`. 4. Call `mmap` on the `ion_buffer` file descriptor, this will map the page containing the `ion_heap` to user space. This page can then be read directly from user space to obtain the `ops` pointer, which will give the KASLR offset.

The use of the `ion_buffer` struct also solves another problem. While the fake kernel heap is convenient, it is not perfect. Whenever an object in the fake kernel heap is freed, `kfree` will check whether the page containing the object is a single page slab from the SLUB allocator by using the `PageSlab` check. If the check fails, then the `PageCompound` check will be performed to check whether the page is part of a bigger slab.

```
void kfree(const void *x)
{
    struct page *page;
    void *object = (void *)x;

    trace_kfree(_RET_IP_, x);

    if (unlikely(ZERO_OR_NULL_PTR(x)))
        return;

    page = virt_to_head_page(x);
    if (unlikely(!PageSlab(page))) {     //<-------- check if the page allocated is a single page slab
        unsigned int order = compound_order(page);

        BUG_ON(!PageCompound(page));     //<-------- check if the page is allocated as part of a multipage slab
        ...
    }
    ...
}
```

As these checks are performed on the `page` struct itself, which contains metadata to the page, they will fail and cause the kernel to crash whenever an object is freed. This can be fixed by using the arbitrary read and write primitives that I now have to overwrite the respective metadata in the `page` struct (the address of a `page` struct that corresponds to a physical address is simply a logical shift of the physical address followed by a translation of a fixed offset, so I can map the `page` containing the `page` struct to user space and modify its content). It would, however, be simpler if I can ensure that the objects that occupy the fake kernel heap never get freed. Before an `ion_buffer` struct is freed, `ion_buffer_destroy` is called:

```
int ion_buffer_destroy(struct ion_device *dev, struct ion_buffer *buffer)
{
    ...
    heap = buffer->heap;
    ...
    if (heap->flags & ION_HEAP_FLAG_DEFER_FREE)
        ion_heap_freelist_add(heap, buffer);    //<--------- does not free immediately
    else
        ion_buffer_release(buffer);

    return 0;
}
```

If the `ion_heap` contains the flag `ION_HEAP_FLAG_DEFER_FREE`, then the `ion_buffer` will not be freed immediately, but instead gets added to the `free_list` of the `ion_heap` using `ion_heap_freelist_add`. The `ion_buffer` objects added to this list will only be freed at a later stage when needed and only if the `ION_HEAP_FLAG_DEFER_FREE` flag is set. Normally, of course, the `ION_HEAP_FLAG_DEFER_FREE` does not change over the lifetime of the `ion_heap`, but with our arbitrary memory write primitive, I can simply add `ION_HEAP_FLAG_DEFER_FREE` to the `ion_heap->flags`, free the `ion_buffer`, and then remove `ION_HEAP_FLAG_DEFER_FREE` again and the `ion_buffer` will just get stuck in the freelist of the `ion_heap` and never get freed. Moreover, the page containing the `ion_heap` object is already mapped for the purpose of defeating KASLR, so toggling the flag is fairly trivial. By spraying the fake kernel heap so that it is filled with `ion_buffer` objects and their dependents, I can ensure that those objects are never freed and avoid the kernel crash.

Bypassing SELinux
-----------------

When SELinux is enabled, it can be run in either the `permissive` mode or the `enforcing` mode. When in `permissive` mode, it will only audit and log unauthorized accesses but will not block them. The mode in which SELinux is run is controlled by the `selinux_enforcing` variable. If this variable is zero, then SELinux is run in `permissive` mode. Normally, variables that are critical to the security of the system are protected by Samsung’s Kernel Data Protection (KDP), by marking them as read-only using the `__kdp_ro` or the `__rkp_ro` attribute. This attribute indicates that the variable is in a read-only page and its modification is guarded by hypervisor calls. However, to my surprise, it seems that Samsung has forgotten to protect this variable ([again?!](https://securitylab.github.com/research/qualcomm_npu/)) in the 5.x branch Qualcomm kernel:

```
//In security/selinux/hooks.c
#ifdef CONFIG_SECURITY_SELINUX_DEVELOP
static int selinux_enforcing_boot;
int selinux_enforcing;
```

So, I can just overwrite `selinux_enforcing` to zero and set SELinux to the `permissive` mode. While there are other means to bypass SELinux (such as the one used in this [exploit](https://github.com/chompie1337/s8_2019_2215_poc/blob/master/poc/selinux_bypass.c) by Valentina Palmiotti) that work more universally, a shortcut at this point is more than welcome, so I’ll just set the `selinux_enforcing` variable.

Running arbitrary root commands using ret2kworker(TM)
-----------------------------------------------------

A well-known problem with getting root on Samsung devices is the protection imposed by the Samsung’s RKP (Realtime Kernel Protection). A common way to gain root on Android devices is to overwrite the credentials of our own process with the root credentials. However, Samsung’s RKP write protects the credentials of each process, so that is not possible here. In my [last exploit](https://securitylab.github.com/research/qualcomm_npu/), I was able to execute arbitrary code as root because the particular UAF exploited led to a controlled function pointer being executed in code run by a `kworker`, which is run as root. In that exploit, I was able to corrupt objects that are then added to a work queue, which was then consumed by a `kworker` and executed by running a function supplied as a function pointer. This made it relatively easy to run arbitrary functions as root.

Of course, with arbitrary memory read and write primitives, it is possible to simply add objects to one of these work queues (which are basically linked lists containing `work` structs) and wait for a `kworker` to pick up the work. As it turns out, many of these work queues are indeed static global objects, with fixed addresses in the kernel image:

```
ffffffc012c8f7e0 D system_wq
ffffffc012c8f7e8 D system_highpri_wq
ffffffc012c8f7f0 D system_long_wq
ffffffc012c8f7f8 D system_unbound_wq
ffffffc012c8f800 D system_freezable_wq
ffffffc012c8f808 D system_power_efficient_wq
ffffffc012c8f810 D system_freezable_power_efficient_wq
```

So, it is relatively straightforward to add entries to these work queues and have a `kworker` pick up the work. However, because of `kCFI`, I would only be able to call functions with the following signatures:

```
void (func*)(struct work_struct *work)
```

The problem is whether I can find a powerful enough function to run. It turns out to be fairly simple. The function `[call_usermodehelper_exec_work](https://git.codelinaro.org/clo/la/kernel/msm-5.4/-/blob/5e3cf80f1b6a12fcf54b007f3c9f235f35b9b7f1/kernel/umh.c#L190)`, which is commonly used in kernel exploits to run shell commands, fits the bill and will run a shell command supplied by me. So by modifying, say, the `system_unbound_wq` and adding an entry to it that holds a pointer to `call_usermodehelper_exec_work`, I can bypass both Samsung’s RKP and kCFI to run arbitrary commands as root.

The exploit can be found [here](https://github.com/github/securitylab/tree/main/SecurityExploits/Android/Qualcomm/CVE-2022-22057) with some setup notes.

Conclusions
-----------

In this post, I exploited a UAF with fairly typical primitives and examined how various mitigations affected the exploit. While in the end, I was able to bypass all the mitigations and develop an exploit that is no less reliable than another one that I did [last year](https://securitylab.github.com/research/one_day_short_of_a_fullchain_android/), the mitigations did force the exploit to take a very different, and longer path.

The biggest hurdle was the kCFI, which turned a relatively straightforward exploit into a rather complex one. As explained in the post, the UAF bug offers many primitives to execute an arbitrary function pointer. Combined with a separate information leak (which I happen to have, and is pending disclosure) the bug would have been trivial to exploit as in the case of the [NPU bugs](https://securitylab.github.com/research/qualcomm_npu/) I wrote about last year. Instead, the combination of Samsung’s RKP and kCFI made this impossible and forced me to look into an alternative path, which is far less straightforward.

On the other hand, many of the techniques introduced here, such as the ones in “The ultimate fake object store” and “The device memory mirroring attack” can be readily standardized to turn common primitives into arbitrary memory read and write. As we’ve seen, having arbitrary memory read and write, even with the restrictions imposed by Samsung’s RKP, is already powerful enough for many purposes. In this regard, it seems to me that the effect of kCFI may be to shift exploitation techniques in favor of certain primitives over others rather than rendering many bugs unexploitable. It is, after all, as many have said, a mitigation that happens fairly late in the process.

A rather underrated mitigation, perhaps, is automatic variable initialization. While this mitigation mainly targets vulnerabilities that exploit uninitialized variables, we’ve seen that it also prevents partial object replacement, which is a common exploitation technique. In fact, it nearly broke the exploit if it hadn’t been for a piece of luck where I was able to leak the address of a stack variable (see “Escaping an infinite loop”). Not only does this mitigation kill a whole class of bugs, but it also breaks some useful exploit primitives.

We’ve also seen how primitives in manipulating the kernel scheduler enabled us to widen many race windows to allow time for object replacements. This allowed me to overcome the problem posed by the delayed free caused by `kfree_rcu` (not a mitigation), without significantly compromising the reliability. It seems that, in the context of the kernel, mitigating a UAF with a quarantine and delayed free of objects (an approach adopted by the Scudo allocator for Android user processes) may not be that effective after all.

Disclosure practices, patching time, and patch gapping
------------------------------------------------------

I reported this vulnerability to Qualcomm on November 16 2021 and they publicly disclosed it in early May 2022. It took about six months, which seems to be the standard time Qualcomm takes between receiving a report and publicly disclosing it. In the past, the long disclosure times have led to some researchers disclosing full exploits before vulnerabilities were patched in public (see, for example, [this](https://googleprojectzero.blogspot.com/2020/09/attacking-qualcomm-adreno-gpu.html)). Qualcomm employs a rather “special” disclosure process, in which it normally discloses the patch of a vulnerability privately to its customers within 90 days (this is indicated by the “Customer Notified Date” in [Qualcomm’s advisories](https://docs.qualcomm.com/product/publicresources/securitybulletin/may-2022-bulletin.html)), but the patch may only be fully integrated and publicly disclosed several months after that.

While this practice gives customers (OEM) time to patch the vulnerabilities before they become public, it creates an opportunity for what is known as “patch gapping.” As patches are first disclosed privately to customers, it means that some customers may decide to apply the patch before it is publicly disclosed, while others may wait until the patch is fully integrated and disclosed publicly and then apply the monthly patch. For example, [this patch](https://source.codeaurora.org/quic/la/kernel/msm-5.4/commit/?id=6c58829f4428f2564833743de7135f4451077e75) was applied to the Samsung S21 in July 2021 (by inspecting the [Samsung open source](https://opensource.samsung.com/) code), but only publicly disclosed as CVE-2021-30305 in October 2021. By comparing firmware between different vendors, it may be possible to discover vulnerabilities that are privately patched by one vendor, but not the other. As the private disclosure time and public disclosure time are often a few months apart, this leaves ample time to develop an exploit. Moreover, the [patch](https://source.codeaurora.org/quic/la/kernel/msm-5.4/commit/?id=339824df70a0e9e08f2a7151b776d72421050f04) for the vulnerability described in this post was publicly visible sometime in December 2021 with a very clear commit message, leaving a five‐month (or two‐month for private disclosure) gap in between. While the exploit is complex, a skilled hacking team could easily have developed and deployed it within that time window. I hope that Qualcomm will improve their patching time or reduce the gap between their private and public disclosure time.