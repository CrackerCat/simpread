> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/2024/06/driving-forward-in-android-drivers.html)

> Posted by Seth Jenkins, Google Project Zero Introduction Android's open-source ecosystem has led to......

Posted by Seth Jenkins, Google Project Zero

Introduction
------------

Android's open-source ecosystem has led to an incredible diversity of manufacturers and vendors developing software that runs on a broad variety of hardware. This hardware requires supporting drivers, meaning that many different codebases carry the potential to compromise a significant segment of Android phones. [There are recent public examples](https://googleprojectzero.blogspot.com/2023/09/analyzing-modern-in-wild-android-exploit.html) of third-party drivers containing serious vulnerabilities that are exploited on Android. While there exists a well-established body of public ([and In-the-Wild](https://storage.googleapis.com/gweb-uniblog-publish-prod/documents/Year_in_Review_of_ZeroDays.pdf)) security research on Android GPU drivers, other chipset components may not be as frequently audited so this research sought to explore those drivers in greater detail.

Driver Enumeration: Not as Easy as it Looks
-------------------------------------------

This research focused on three Android devices (chipset manufacturers in parentheses):

- Google Pixel 7 (Tensor)

- Xiaomi 11T (MediaTek)

- Asus ROG 6D (MediaTek)

In order to perform driver research on these devices I first had to find all of the kernel drivers that were accessible from an unprivileged context on each device; a task complicated by the non-uniformity of kernel drivers (and their permissions structures) across different devices even within the same chipset manufacturer. There are several different methodologies for discovering these drivers. The most straightforward technique is to search the associated filesystems looking for exposed driver device files. These files serve as the primary method by which userland can interact with the driver. Normally the “file” is open’d by a userland process, which then uses a combination of read, write, ioctl, or even mmap to interact with the driver. The driver then “translates” those interactions into manipulations of the underlying hardware device sending the output of that device back to userland as warranted. Effectively all drivers expose their interfaces through the ProcFS or DevFS filesystems, so I focused on the /proc and /dev directories while searching for viable attack surfaces. Theoretically, evaluating all the userland accessible drivers should be as simple as calling find /dev or find /proc, attempting to open every file discovered, and logging which open attempts were successful.

However, there is one major roadblock that prevents this approach from being comprehensive - permissions! SELinux and traditional Linux Discretionary Access Control policies can prevent simple filesystem enumeration from discovering all of the accessible device drivers associated on the filesystem. For example, the untrusted_app SELinux context has search permissions on the device SELinux context for directories, but is not allowed to directly open the directory itself:

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEioT3h0f4HwgQDI_0LTWJU6KJU3MwVqZ8zuQYgPhF0SXHNCeaw1ut_-WkpVwwqcqLytAsgahAjKshk8ugcbaCZyk8-Us3i8ZfRihE-EA3n8tRFIi0oPUntubw1dJ7y9l1LjTe4Zot7iFd-WJRQ0I1DPdsFA_sULQ5vLIQ8xgQerrz1hyphenhyphenwLj_2J8oC2VD0M/s790/image2.png)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEioT3h0f4HwgQDI_0LTWJU6KJU3MwVqZ8zuQYgPhF0SXHNCeaw1ut_-WkpVwwqcqLytAsgahAjKshk8ugcbaCZyk8-Us3i8ZfRihE-EA3n8tRFIi0oPUntubw1dJ7y9l1LjTe4Zot7iFd-WJRQ0I1DPdsFA_sULQ5vLIQ8xgQerrz1hyphenhyphenwLj_2J8oC2VD0M/s790/image2.png)

This search permission (rather counterintuitively) does not allow the source context to list the contents of directories that have this SELinux target context. Instead, it simply allows such a directory to be in the ancestor directories of the file path that the source context attempts to open. In practice, this means that untrusted_app is allowed to open e.g. /dev/mali0 but is usually not allowed to open /dev itself:

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjbJAsOOwnE0RJ9O7yET8vhuVE-UmrzcyoUVoRP4z4kL7wYGXHHJob-ir9s2pgEDQFaxPmAT14n0hu81XpDrl6BaO7ltXE2jZJ-zIBoPa4hH9PYSSIc2KLk0ulZ_0xLjjaO4_VvhG69zIuqmtoGu1a7s_xn3Ru7_QZnjjLZS2sc8oL-yrwFR6LWUBm1EqA/s624/image6.png)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjbJAsOOwnE0RJ9O7yET8vhuVE-UmrzcyoUVoRP4z4kL7wYGXHHJob-ir9s2pgEDQFaxPmAT14n0hu81XpDrl6BaO7ltXE2jZJ-zIBoPa4hH9PYSSIc2KLk0ulZ_0xLjjaO4_VvhG69zIuqmtoGu1a7s_xn3Ru7_QZnjjLZS2sc8oL-yrwFR6LWUBm1EqA/s624/image6.png)

In the case of /dev, the shell context is allowed to open and list the contents of /dev. That means that by first enumerating the /dev directory from the shell context, then attempting to open all discovered files from the untrusted_app context, a security researcher can understand what drivers are and are not accessible from an app context in the /dev directory. However, there are cases where certain directories are simply not listable from a debugging-accessible non-root context, particularly in /proc. One option to enumerate all these directories would be to root the phone, however, this is not always easily achievable.

A strategy I found helpful in this regard was to examine publicly released kernel source code for the phone model or for similar phone models. The location of this source code varies significantly from manufacturer to manufacturer, but the source code is usually either hosted on Github or via the manufacturer website. Device drivers create files in /proc primarily via the proc_create() and proc_mkdir() function calls. A real-world example of this would be:

        parent = proc_mkdir("perfmgr", NULL);

        perfmgr_root = parent;

        pe = proc_create("perf_ioctl", 0664, parent, &Fops);

        ...

        pe = proc_create("eara_ioctl", 0664, parent, &eara_Fops);

        ...

        pe = proc_create("eas_ioctl", 0664, parent, &eas_Fops);

        ...

        pe = proc_create("xgff_ioctl", 0664, parent, &xgff_Fops);

        ...

Although these files cannot be directly enumerated, they do exist and are accessible from an untrusted context.

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgyj_BhXcb7qJCBlmypX0tZzSn75nUpANd16viDRspDEgRhdbE_l6t8p2XK1bHVC9NI6vPYIrV4o_sylWf4WaqrOc4sLVFKVTQRCKKz-BxvagFmwO0GBziHvTJUXBYbfb4c1YmgmcpM5Rt-bT2Ld8z15J2s8sxoi_Ed830F3wZnA3dL5uy-3yf4HNBrz4c/s497/image5.png)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgyj_BhXcb7qJCBlmypX0tZzSn75nUpANd16viDRspDEgRhdbE_l6t8p2XK1bHVC9NI6vPYIrV4o_sylWf4WaqrOc4sLVFKVTQRCKKz-BxvagFmwO0GBziHvTJUXBYbfb4c1YmgmcpM5Rt-bT2Ld8z15J2s8sxoi_Ed830F3wZnA3dL5uy-3yf4HNBrz4c/s497/image5.png)

It would have otherwise required rooting the phone to discover this driver without analyzing the kernel source code.

Another useful resource is the SELinux policy itself. Userland interacts with the drivers via a fairly typical set of VFS operations. This means that the SELinux policy must encapsulate the necessary permissions to perform those operations. This means that the SELinux policy generally reflects what the developers intend to be accessible from an untrusted context. Analysis of the policy can lead to the discovery of certain oddities and idiosyncrasies in the accessibility of certain drivers. For example, occasionally a file may not be directly openable via the filesystem, but there may be some alternative method by which an app can ask another more privileged process to open the file on its behalf and hand the associated fd back, after which the app is allowed to read/write/ioctl to the fd itself. One example of this behavior would be the EdgeTPU device on the Pixel 7:  

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjD4s4ms0da-AJFNNkpiEVi9yew1XcUIY852C3-WuI8Tqidiyl7Zy2rNdNDvyD0MI8RGPsuF6Y9DGo4js9o4r4iI9JHP25Ygrv2cYIiRjkhH8DI9rg0oe-nnxrXCtn23ROy3f3k8Sqp22Nz21TcfjQhyphenhyphenbg1Hj4FU4W7WnMCOPOFYMmksu5uYi-VZeFD8I8/s740/image9.png)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjD4s4ms0da-AJFNNkpiEVi9yew1XcUIY852C3-WuI8Tqidiyl7Zy2rNdNDvyD0MI8RGPsuF6Y9DGo4js9o4r4iI9JHP25Ygrv2cYIiRjkhH8DI9rg0oe-nnxrXCtn23ROy3f3k8Sqp22Nz21TcfjQhyphenhyphenbg1Hj4FU4W7WnMCOPOFYMmksu5uYi-VZeFD8I8/s740/image9.png)

Additional research suggests that untrusted_app can ask a privileged process for access to the EdgeTPU driver fd itself if it lands on an allowlist of certain applications.

The performed surveys strongly imply that the GPU driver is the most consistently accessible driver from an untrusted application, which is expected. On the Google Pixel 7, I did not find much else that was accessible from an entirely unprivileged context. Nevertheless, inspired by [previous similar efforts on hardware like Samsung’s NPU](https://googleprojectzero.github.io/0days-in-the-wild//0day-RCAs/2022/CVE-2022-22265.html), I performed research on the EdgeTPU driver - Google’s tensor processing unit for doing ML related tasks on the Pixel series of devices. This resulted in the discovery of one significant [issue](https://bugs.chromium.org/p/project-zero/issues/detail?id=2455) - a race condition when registering memory with the EdgeTPU memory while vma’s are concurrently getting modified.

Unlike the Pixel 7, the MediaTek chipset phones (Asus ROG 6D and Xiaomi 11T) contained several different drivers that could be accessed from unprivileged userland:

*   /proc/ged
*   /proc/mtk_jpeg
*   /proc/perfmgr/[eara_ioctl,eas_ioctl,perf_ioctl,xgff_ioctl]

These drivers represent significantly more interesting and complex attack surfaces than what was available on the Pixel 7 device. The ged driver contained numerous interesting and valuable exploitation primitives that we’ll discuss in detail a bit later. While the perfmgr driver presented several attack surfaces, I wasn’t able to find any security-relevant bugs. The mtk_jpeg driver however, yielded significant fruit that deserves a closer look.

MediaTek JPEG Decoding Accelerator
----------------------------------

The mtk_jpeg driver manages specialized hardware on MediaTek devices to perform jpeg decoding acceleration. [Linux kernel documentation](https://github.com/torvalds/linux/blob/8f2c057754b25075aa3da132cd4fd4478cdab854/Documentation/devicetree/bindings/media/mediatek-jpeg-decoder.yaml#L12) notes that “Mediatek JPEG Decoder is the JPEG decode hardware present in Mediatek SoCs”. More relevantly, from an attacker's point of view, this driver can be accessed (at least on the phones assessed) from the untrusted_app context (although curiously, it cannot be accessed from an unprivileged adb debugging context). This JPEG decoding accelerator and its associated driver is present on both the Xiaomi 11T and the Asus ROG 6D. However, based on open-source codebases for these different devices' kernels, it appears MediaTek is actively maintaining several different trees for this driver, likely based on the associated kernel version, and these two devices use separate trees.

I found two vulnerabilities in this driver. [CVE-2023-32837](https://bugs.chromium.org/p/project-zero/issues/detail?id=2471) was a textbook OOB read/write in an array of structs. Various different members of the struct were accessed and modified, creating several different possibilities for exploitation, but also making them significantly more challenging. Interestingly, MediaTek [partially fixed](https://github.com/OnePlusOSS/android_kernel_5.10_oneplus_mt6983/commit/725e99c941caf985b25ef5595b2b4f632320d235) this bug in July 2021, although the exact date this patch went out to OEMs is unclear. From the commit message, it’s clear that MediaTek detected this issue with the Coverity static analysis tool, but it appears unlikely that the security impact was identified. Regardless, while the issue was fixed in some of the MediaTek kernel trees, it went unpatched in other versions of that same driver. This meant that while the Asus ROG 6D (running kernel 5.10) had received the patch for this vulnerability, the (otherwise fully patched and security supported) Xiaomi 11T (running 4.14) had not.

Some background knowledge on how the jpeg driver works eases discussion of the other issue, [CVE-2023-32832](https://bugs.chromium.org/p/project-zero/issues/detail?id=2470). The accelerator hardware has two separate “cores” that can perform JPEG decoding. When a process requests JPEG decoding work to be performed, it calls the ioctl JPEG_DEC_IOCTL_HYBRID_START, and the kernel decides which decoding core will perform that work inside of jpeg_drv_hybrid_dec_lock()(output has been colorized to ease following along):

static int jpeg_drv_hybrid_dec_lock(int *hwid)

{

        int retValue = 0;

        int id = 0;

        ...

        mutex_lock(&jpeg_hybrid_dec_lock);

        for (id = 0; id < HW_CORE_NUMBER; id++) {

                if (dec_hwlocked[id]) {

                        JPEG_LOG(1, "jpeg dec HW core %d is busy", id);

                        continue;

                } else {

                        *hwid = id;

                        dec_hwlocked[id] = true;

                        JPEG_LOG(1, "jpeg dec get %d HW core", id);

                        _jpeg_hybrid_dec_int_status[id] = 0;

                        jpeg_drv_hybrid_dec_power_on(id);

                        enable_irq(gJpegqDev.hybriddecIrqId[id]);

                        break;

                }

        }

        mutex_unlock(&jpeg_hybrid_dec_lock);

        if (id == HW_CORE_NUMBER) {

                JPEG_LOG(1, "jpeg dec HW core all busy");

                *hwid = -1;

                retValue = -EBUSY;

        }

        return retValue;

}

The array dec_hwlocked contains a boolean element for each core, with that element being set to true for locked cores, and false for unlocked cores. This array is also protected with a mutex to try and prevent concurrent calls to jpeg_drv_hybrid_dec_lock, or jpeg_drv_hybrid_dec_unlock from racing with each other. After locking the core, jpeg_drv_hybrid_dec_start sets up the data-structures to be utilized for the decoding operation:

switch (cmd) {

        case JPEG_DEC_IOCTL_HYBRID_START:

                if (copy_from_user(

                        &taskParams, (void *)arg,

                        sizeof(struct JPEG_DEC_DRV_HYBRID_TASK))) {

                        return -EFAULT;

                }

                ...

                if (jpeg_drv_hybrid_dec_lock(&hwid) == 0) {

                        *pStatus = JPEG_DEC_PROCESS;

                } else {

                        JPEG_LOG(1, "jpeg_drv_hybrid_dec_lock failed (hw busy)");

                        return -EBUSY;

                }

                if (jpeg_drv_hybrid_dec_start(taskParams.data, hwid, &index_buf_fd) == 0) {

                        ...

                } else {

                        JPEG_LOG(0, "jpeg_drv_dec_hybrid_start failed");

                        jpeg_drv_hybrid_dec_unlock(hwid);

                        return -EFAULT;

                }

                break;

...

}

static int jpeg_drv_hybrid_dec_start(unsigned int data[],unsigned int id,int *index_buf_fd)

{

        u64 ibuf_iova, obuf_iova;

        int ret;

        void *ptr;

        unsigned int node_id;

        JPEG_LOG(1, "+ id:%d", id);

        ret = 0;

        ibuf_iova = 0;

        obuf_iova = 0;

        node_id = id / 2;

        bufInfo[id].o_dbuf = jpg_dmabuf_alloc(data[20], 128, 0);

        bufInfo[id].o_attach = NULL;

        bufInfo[id].o_sgt = NULL;

        bufInfo[id].i_dbuf = jpg_dmabuf_get(data[7]);

        bufInfo[id].i_attach = NULL;

        bufInfo[id].i_sgt = NULL;

        if (!bufInfo[id].o_dbuf) {

            JPEG_LOG(0, "o_dbuf alloc failed");

                return -1;

        }

        if (!bufInfo[id].i_dbuf) {

            JPEG_LOG(0, "i_dbuf null error");

                return -1;

        }

        ret = jpg_dmabuf_get_iova(bufInfo[id].o_dbuf, &obuf_iova, gJpegqDev.pDev[node_id], &bufInfo[id].o_attach, &bufInfo[id].o_sgt);

        JPEG_LOG(1, "obuf_iova:0x%llx lsb:0x%lx msb:0x%lx", obuf_iova,

                (unsigned long)(unsigned char*)obuf_iova,

                (unsigned long)(unsigned char*)(obuf_iova>>32));

        ptr = jpg_dmabuf_vmap(bufInfo[id].o_dbuf);

        if (ptr != NULL && data[20] > 0)

                memset(ptr, 0, data[20]);

        jpg_dmabuf_vunmap(bufInfo[id].o_dbuf, ptr);

        jpg_get_dmabuf(bufInfo[id].o_dbuf);

        // get obuf for adding reference count, avoid early release in userspace.

        *index_buf_fd = jpg_dmabuf_fd(bufInfo[id].o_dbuf);

        ret = jpg_dmabuf_get_iova(bufInfo[id].i_dbuf, &ibuf_iova, gJpegqDev.pDev[node_id], &bufInfo[id].i_attach, &bufInfo[id].i_sgt);

        JPEG_LOG(1, "ibuf_iova 0x%llx lsb:0x%lx msb:0x%lx", ibuf_iova,

                (unsigned long)(unsigned char*)ibuf_iova,

                (unsigned long)(unsigned char*)(ibuf_iova>>32));

        if (ret != 0) {

                JPEG_LOG(0, "get iova fail i:0x%llx o:0x%llx", ibuf_iova, obuf_iova);

                return ret;

        }

        ...

        return ret;

}

Finally, utilizing an ioctl call to JPEG_DEC_IOCTL_HYBRID_WAIT (which calls jpeg_drv_hybrid_dec_unlock), resources associated with the core are freed, and the core is released back to be used in future operations.

case JPEG_DEC_IOCTL_HYBRID_WAIT:

                ...

                if (copy_from_user(

                        &pnsParmas, (void *)arg,

                        sizeof(struct JPEG_DEC_DRV_HYBRID_P_N_S))) {

                        JPEG_LOG(0, "Copy from user error");

                        return -EFAULT;

                }

                /* set timeout */

                timeout_jiff = msecs_to_jiffies(3000);

                JPEG_LOG(1, "JPEG Hybrid Decoder Wait Resume Time: %ld",

                                timeout_jiff);

                hwid = pnsParmas.hwid;

                if (hwid < 0 || hwid >= HW_CORE_NUMBER) { //In other versions of the driver, this >= check was omitted, which led to several different OOB accesses later aka CVE-2023-32837

                        JPEG_LOG(0, "get hybrid dec id failed");

                        return -EFAULT;

                }

                if (!dec_hwlocked[hwid]) {

                        JPEG_LOG(0, "wait on unlock core %d\n", hwid);

                        return -EFAULT;

                }

                if (jpeg_isr_hybrid_dec_lisr(hwid) < 0) {

                        long ret = 0;

                        int waitfailcnt = 0;

                        do {

                                ret = wait_event_interruptible_timeout(

                                        hybrid_dec_wait_queue[hwid],

                                        _jpeg_hybrid_dec_int_status[hwid],

                                        timeout_jiff);

                                ...

                                if (ret < 0) {

                                        waitfailcnt++;

                                        usleep_range(10000, 20000);

                                }

                        } while (ret < 0 && waitfailcnt < 500);

                }

                ...

                if (copy_to_user(pnsParmas.progress_n_status, &progress_n_status,                                sizeof(int))) {

                        return -EFAULT;

                }

                ...

                jpeg_drv_hybrid_dec_unlock(hwid);

                break;

                ...

}

...

static void jpeg_drv_hybrid_dec_unlock(unsigned int hwid)

{

        mutex_lock(&jpeg_hybrid_dec_lock);

        if (!dec_hwlocked[hwid]) {

                JPEG_LOG(0, "try to unlock a free core %d", hwid);

        } else {

                dec_hwlocked[hwid] = false;

                JPEG_LOG(1, "jpeg dec HW core %d is unlocked", hwid);

                jpeg_drv_hybrid_dec_power_off(hwid);

                disable_irq(gJpegqDev.hybriddecIrqId[hwid]);

                jpg_dmabuf_free_iova(bufInfo[hwid].i_dbuf,

                        bufInfo[hwid].i_attach,

                        bufInfo[hwid].i_sgt);

                jpg_dmabuf_free_iova(bufInfo[hwid].o_dbuf,

                        bufInfo[hwid].o_attach,

                        bufInfo[hwid].o_sgt);

                jpg_dmabuf_put(bufInfo[hwid].i_dbuf);

                jpg_dmabuf_put(bufInfo[hwid].o_dbuf);

                // we manually add 1 ref count, need to put it.

        }

        mutex_unlock(&jpeg_hybrid_dec_lock);

}

jpeg_drv_hybrid_dec_unlock is also called in the event that jpeg_drv_hybrid_dec_start fails.

While the jpeg_hybrid_dec_lock mutex protects the direct core locking and unlocking, it does not protect the body of the jpeg_drv_hybrid_dec_start function. This means that while there cannot be concurrent calls to both jpeg_drv_hybrid_dec_lock and jpeg_drv_hybrid_dec_unlock, there can be concurrent calls to jpeg_drv_hybrid_dec_start and jpeg_drv_hybrid_dec_unlock which in practice is just as bad, as these two functions racily access the same global data structure bufInfo.

One small added complication for this bug is that in order to reach the jpeg_drv_hybrid_dec_unlock in the JPEG_DEC_IOCTL_HYBRID_WAIT call, the core must be locked before the timeout due to a check to ensure that the core is locked before attempting to wait on the core.

An example of this race in practice with two processes A and B would be (colorized respective to the above code):

Process A:

Calls ioctl JPEG_DEC_IOCTL_HYBRID_START, which locks core 0 with jpeg_drv_hybrid_dec_lock and enters jpeg_drv_hybrid_dec_start

Process B:

Calls ioctl JPEG_DEC_IOCTL_HYBRID_WAIT, which confirms that core 0 is locked then begins a 3 second wait for the core to send an interrupt denoting completion of the decoding request.

Process A:

Fails jpeg_drv_hybrid_dec_start, (after initializing some of the data structures), calls jpeg_drv_hybrid_dec_unlock on core 0 freeing any allocated resources, and returns to userland.

[wait ~3 seconds]

Process A:

Calls ioctl JPEG_DEC_IOCTL_HYBRID_START, which locks core 0 with jpeg_drv_hybrid_dec_lock and enters jpeg_drv_hybrid_dec_start

Process B:  
3 second wait times out, and the JPEG_DEC_IOCTL_HYBRID_WAIT ioctl call unlocks core 0 with jpeg_drv_hybrid_dec_unlock.

[Process A and B are now concurrently initializing and freeing the same data-structures]

This can lead to a variety of use-after-free or double free conditions, depending on how process A and B race.

The Journey to root
-------------------

The next step was to try to exploit these issues. My first attempt targeted the OOB write issue, CVE-2023-32837. I was able to develop the primitive from an uncontrolled OOB read/write in the kernel .data region into a racy write of null bytes at a predetermined offset in a kernel task stack used by an attacker-controlled process. At this point it was possible to overwrite a kernel stack entry with nulls during any syscall which felt to me like enough flexibility to create a full exploit. However, despite my best efforts (including the creation of a tool to find where in an arbitrary backtrace the write would occur), I was unable to discover a technique to create a better primitive from this write.

Failing that effort, I decided to take a look at the other issue in the same driver CVE-2023-32832. During the freeing step that races with jpeg_drv_hybrid_dec_start, jpeg_drv_hybrid_dec_unlock drops access to four separate resources:

jpg_dmabuf_free_iova(bufInfo[hwid].i_dbuf, bufInfo[hwid].i_attach, bufInfo[hwid].i_sgt); //The input buffer virtual address mapping in the core

jpg_dmabuf_free_iova(bufInfo[hwid].o_dbuf, bufInfo[hwid].o_attach, bufInfo[hwid].o_sgt); //The output buffer virtual address mapping in the core

jpg_dmabuf_put(bufInfo[hwid].i_dbuf); //The input buffer file's refcount is decremented. This buffer was previously allocated by the attacker and is associated with a file descriptor.

jpg_dmabuf_put(bufInfo[hwid].o_dbuf); //The output buffer file's refcount is decremented. This buffer was previously allocated during jpeg_drv_hybrid_dec_start.

One critical behavior of the driver that enhanced exploitability was that although jpeg_drv_hybrid_dec_unlock properly drops the refcounts of i_dbuf and o_dbuf, it does not reinitialize those entries in the bufInfo global array to NULL. As it relates to the race, this means that if Process B’s racy jpeg_drv_hybrid_dec_unlock occurs before Process A’s second jpeg_drv_hybrid_dec_start reinitializes i_dbuf and o_dbuf, an extra refcount of i_dbuf and o_dbuf will be released. Since i_dbuf and o_dbuf are struct file*’s, this can lead directly to a struct file UAF. As the i_dbuf struct file comes directly from a dmabuf file descriptor passed into jpeg_drv_hybrid_dec_start, this leads to a dangling file descriptor with the struct file freed from underneath it. This is undoubtedly an exploitable bug.

There are several different techniques for exploiting a dangling file descriptor. One widely used strategy is causing the backing slab page of the struct file to be freed and returned back to the page allocator, then reallocating that page with pipe buffer data pages in order to gain attacker control over the memory used for the struct file. Another well-known strategy would be to utilize the cross-cache technique to reallocate the memory as a different kind of kmalloc slab/object. However, in the future both of these techniques may be remediated if the [SLAB_VIRTUAL mitigation](https://www.google.com/url?q=https://lore.kernel.org/all/20230915105933.495735-15-matteorizzo@google.com/&sa=D&source=docs&ust=1712947244908914&usg=AOvVaw0WRxCpYz3NaM3wsC8DKyOp) [](https://www.google.com/url?q=https://lore.kernel.org/all/20230915105933.495735-15-matteorizzo@google.com/&sa=D&source=docs&ust=1712947244908914&usg=AOvVaw0WRxCpYz3NaM3wsC8DKyOp) comes into effect in the mainline Linux kernel. In the interest of exploring the future of Android kernel hacking, I sought a novel exploitation technique which did not involve cross-cache or slab-cache->page allocator heap shaping techniques.

### Some Other Novel Exploitation Technique

One of the most common UAF exploit techniques involves reallocating the first-order freed object with a new object of a different type, creating a type-confusion condition which leads to an improved memory corruption primitive. However the only type of object that can be allocated in a page designated for the struct file cache is the struct file type, so the options for creating a type confusion memory corruption condition using first-order object reclamation are limited. However, just because an object is freed does not preclude it from being usable under limited circumstances. When the kernel inserts an object onto a freelist, this clobbers [the middle](https://lore.kernel.org/linux-mm/202003051624.AAAC9AECC@keescook/t/) of the object. The rest of the object however, remains in whatever state it was in at the moment it was freed, including pointers and any other member variables. Those stale pointers can (and in practice often do) point to other freed objects which may be allocated from a different slab cache entirely, potentially including the generic kmalloc slab-caches. Note that under the [C-ism](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=87152148) [where pointers are set to](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=87152148) [NULL](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=87152148) [after freeing](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=87152148), these stale pointers wouldn’t exist. However as the memory containing the pointer is getting freed anyway, setting these pointers to NULL is often seen as unnecessarily conservative (and in fact, [C compilers often throw away writes to objects that are about to be freed](https://www.cryptologie.net/article/419/zeroing-memory-compiler-optimizations-and-memset_s/)).

By continuing to use this freed first-order object, we can implicitly access freed second-order child objects. By reclaiming those objects, we can recreate a type-confusion memory corruption primitive, taking advantage of how the methods called on the first-order freed object implicitly access the second-order child object. Let’s see how we can apply this to our specific scenario.

Linux kernel struct files may represent many different types of files depending on the kind of opened file such as ext4 files, procfs files, or even MediaTek JPEG decoding driver files. In order to represent all of these different types while also maintaining some commonality of structure for the universally needed members of an opened file, struct file contains a private_data member which references any type-specific data needed.

As mentioned previously, the UAF’d struct file in this case is a dmabuf file. This means the private_data pointer points to a struct dma_buf object. The dma_buf object lifetime is implicitly tied to the lifetime of the associated dmabuf file struct. When the dmabuf struct file is freed, the dma_buf object is freed too. However unlike the struct file, dma_buf objects are allocated from the generic kmalloc slab caches. This means that the dma_buf can be reclaimed with a different type object that comes from the same generic kmalloc slab cache.

After a hypothetical reclamation by a new object, this new object can still be UAF referenced as a dma_buf through the freed but still very usable dmabuf struct file that itself is referenced via the dangling file descriptor! Thus we arrive at the following strategy:

1.  Free the file by using our race condition bug to drop an extra reference on i_dbuf (which also frees the dma_buf), leaving a dangling fd pointing to a freed struct file which still has a stale pointer to a freed dma_buf
2.  Reclaim the dma_buf WITHOUT reclaiming the struct file
3.  Call dma_buf operations on the dangling fd

You may have astutely noted by this point that this strategy relies on a freed object (that is, the struct file) not being reclaimed as another object by the heap allocator. This is absolutely correct, and one would expect that in an exploit where exceptional reliability is a priority, it may be necessary to perform some heap shaping in order to bury this freed struct file deeply in the allocator freelists. In practice (and in my exploit), the freed struct file will rarely be on the percpu active slab so it’s unlikely to get reclaimed immediately, and my exploit generally runs fast enough that it doesn’t matter.

At this point we now need to determine what object to use in order to reclaim the freed dma_buf, as well as what operation to call on the freed dma_buf file/object to develop a stronger primitive. I ended up finding the solutions to both of these problems in the GED driver.

### The GED driver

The GED (GPU Extension Device) driver is a MediaTek-specific interface that provides userland with several supplementary GPU features, primarily for tuning purposes. Two of its “features” appeared particularly valuable. Feature number one, GED GE Buffers, presented a truly remarkable heap spray and reclamation primitive. This feature provides several requisite characteristics of a suitable heap spray primitive:

*   Allocates buffers of a controlled size without causing undue noise for the rest of the heap.
*   Buffer data is fully attacker controlled, with no uncontrolled header at the beginning
*   Buffers can be freed at any time.

One standout characteristic that elevates this heap spray primitive above many of its peers however, is that even once allocated, the attacker can read and write to these buffers at will while keeping the buffer allocated. This is about as powerful a heap spray primitive as one could imagine. By reclaiming the UAF’d dma_buf struct with a GED GE buffer, we gain fully deterministic read/write over the dma_buf struct, including any pointers contained therein.

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjc09kF7Yn0WI8PNuSGP9V1mXoszwnxS_FSdSUgDBVeFuZJ4l_CNTFrqrmCBrCnrrQ43x_jrQEloAj5hHz8mIPLUW7KNWZC4TP39vBNgZHDNAT1gMjxYsCNXzGb0qU6aO8J4g4QrsvYBgnClQlm-I__4IWSCgZbWULq-zMZvBQPXlAU7qS83gj4fQPNrAk/s721/image7.png)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjc09kF7Yn0WI8PNuSGP9V1mXoszwnxS_FSdSUgDBVeFuZJ4l_CNTFrqrmCBrCnrrQ43x_jrQEloAj5hHz8mIPLUW7KNWZC4TP39vBNgZHDNAT1gMjxYsCNXzGb0qU6aO8J4g4QrsvYBgnClQlm-I__4IWSCgZbWULq-zMZvBQPXlAU7qS83gj4fQPNrAk/s721/image7.png)

Feature number two is an alternate codepath to the same functionality as the DMA_BUF_SET_NAME ioctl, which is (very sensibly) used to set the name of a dma_buf. The biggest difference between these paths is the GED codepath’s lack of SELinux inode checks on the underlying dma_buf fd. These inode checks would normally crash the kernel when they run on a freed struct file - however because of the GED codepath, we can skip this inode check, and change the dma_buf’s name despite running on a freed dma_buf file! Normally, this code would free the pointer to the previous name within the dma_buf struct and allocate a new buffer for the name string. However, because of GED GE buffers we are able to control the entirety of the dma_buf struct. By combining these primitives, we can kfree an arbitrary pointer before setting a new name string.

long mtk_dma_buf_set_name(struct dma_buf *dmabuf, const char *buf)

{

        char *name = kstrndup(buf, DMA_BUF_NAME_LEN, GFP_KERNEL);

        ...

        kfree(dmabuf->name); //dmabuf is attacker controlled

        dmabuf->name = name; //the name pointer is written to attacker controlled memory

        ...

}

### Achieving arbitrary read

Innocuous dmabuf file operations become potent primitives now that we have precise control of the dma_buf struct. For example, this is the code backing /proc/pid/fdinfo/n for dmabuf files:

static void dma_buf_show_fdinfo(struct seq_file *m, struct file *file)

{

        struct dma_buf *dmabuf = file->private_data;

        seq_printf(m, "size:\t%zu\n", dmabuf->size);

        /* Don't count the temporary reference taken inside procfs seq_show */

        seq_printf(m, "count:\t%ld\n", file_count(dmabuf->file) - 1);

        seq_printf(m, "exp_name:\t%s\n", dmabuf->exp_name);

        spin_lock(&dmabuf->name_lock);

        if (dmabuf->name)

                seq_printf(m, "name:\t%s\n", dmabuf->name);

        spin_unlock(&dmabuf->name_lock);

}

There are several opportunities in this function for achieving arbitrary read, but the cleanest one is the file_count() call, which will dereference the passed pointer + a hardcoded offset, and print the read 8 byte value as a signed long. Normally in the context of this C function, file == ((struct dma_buf*) file->private_data)->file , but since we control the dma_buf struct that isn’t necessarily the case.

### Achieving arbitrary write

At this point we have three powerful primitives:

*   Read/write UAF’d dma_buf struct memory (via GED GE buffers)
*   Arbitrary read (via dma_buf_show_fdinfo)
*   Arbitrary free (via ged_dmabuf_set_name)

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEilzJmXjC5MwoGnxuUe6-rqIyG5EXRaE_B0V8VCgMLkIqXExr7_70KBjmSunxcHwktChFADLZAzByhKhUZVFaps-ZvoyWrMCUCK2eEmbQdVLhQrtiJbSB4kW_WkPxjv_9xNWfj5VgBFOPxfIixfDAzf4OXZS58tY7GgSSWGZ8pf11uf1Z0KThjgxutlz_o/s721/image8.png)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEilzJmXjC5MwoGnxuUe6-rqIyG5EXRaE_B0V8VCgMLkIqXExr7_70KBjmSunxcHwktChFADLZAzByhKhUZVFaps-ZvoyWrMCUCK2eEmbQdVLhQrtiJbSB4kW_WkPxjv_9xNWfj5VgBFOPxfIixfDAzf4OXZS58tY7GgSSWGZ8pf11uf1Z0KThjgxutlz_o/s721/image8.png)

There are many potential strategies that use these primitives to achieve an arbitrary write primitive. The technique I chose was to type-confuse a GE buffer with a GE buffer array. GED GE Buffers are tracked through a hierarchy of structs and arrays. A GE file’s private_data member points to an array of GE buffer pointers like so:

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgSqvGA83iM6fq-D7sAnG4OCM6T_OdDGD5LpYKJBvE6xFxCciy51nIE4MBa3jERVONt_0tArpFf2YMFuxHE-LmT49UdX6fMgGlZ2kEPCc0sTKKeFE0F5BEphWMyZ9jWUF07vlEomRoxBgP9at8zPODEIPqTfPFlYzBTgShwo7TrlrNlEkieeDjxlWlRvWk/s611/image4.png)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgSqvGA83iM6fq-D7sAnG4OCM6T_OdDGD5LpYKJBvE6xFxCciy51nIE4MBa3jERVONt_0tArpFf2YMFuxHE-LmT49UdX6fMgGlZ2kEPCc0sTKKeFE0F5BEphWMyZ9jWUF07vlEomRoxBgP9at8zPODEIPqTfPFlYzBTgShwo7TrlrNlEkieeDjxlWlRvWk/s611/image4.png)

I achieve this type-confusion by using the arbitrary-free primitive developed previously to free a GE buffer array, then reclaiming that array with a GE buffer from a second GE file. Since the GE buffer array (and GE buffers as well) come from generic kmalloc caches, the only requirement for this reclamation is allocating GE buffers that are the same size as a GE buffer array. If two GE files are referencing the same memory, one (GE file A) as a GE buffer, and one (GE file B) as a GE buffer array, I can modify the contents of a GE buffer array at will.

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEioaW3vdBverVpVB95IW7_vsrH8JdiB52ly8zRn5A0QYG9LF5XCNdmhK07LRTlACDxxvOII82zFoHHGnA15VjkDJiKVZXqyykk2KI3pzr1-LXJaYTF2igmf4l7zfs__LrnJ0BcTpRjMXOjS6cJNAR3bJAZ6O4nRG1kfHCQs1QySE689IW-iha9dxoOz28o/s731/image1.png)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEioaW3vdBverVpVB95IW7_vsrH8JdiB52ly8zRn5A0QYG9LF5XCNdmhK07LRTlACDxxvOII82zFoHHGnA15VjkDJiKVZXqyykk2KI3pzr1-LXJaYTF2igmf4l7zfs__LrnJ0BcTpRjMXOjS6cJNAR3bJAZ6O4nRG1kfHCQs1QySE689IW-iha9dxoOz28o/s731/image1.png)

Then performing an arbitrary write to a virtual address X will be as simple as using GE file A’s GE buffer to change the contents of the array to point to the virtual address X, then writing to that address using GE file B which now thinks virtual address X is a GE buffer!

This technique hinges on being able to use the arbitrary free primitive to free a GE buffer array. To do that, it’s necessary to find the virtual address of a GE buffer array first. Since we have an arbitrary read primitive already, any parent struct/object/array of a GE buffer array will be enough to find the virtual address of a GE buffer array itself. The hierarchy is as follows:

1.  GE arrays are referenced by a GE file
2.  GE files are referenced by an fdtable as a file descriptor
3.  An fdtable is referenced by a task struct
4.  Task structs are referenced as part of the task list with the root node being the init task in the kernel image

The fdtable represents an attractive object to find, as it comes out of the same generic kmalloc cache as dma_buf name strings. We can find a dma_buf name string’s virtual address by using dma_buf_set_name (which we also use as our arbitrary free primitive) to insert a pointer to a dma_buf name string into the reclaimed UAF’d dma_buf object that is now a GE buffer. We then simply read it out of our GE buffer, free that dma_buf name string (again using dma_buf_set_name), and reclaim it with an fdtable. Creating fdtables is fairly easy - we simply fork many processes ahead of time that share an fdtable, then unshare(2) the fdtable at the appropriate time to allocate new fdtables. The full exploit strategy is as follows:

1.  Trigger a dangling fd of a dmabuf file using our mtk-jpeg race condition bug
2.  Reclaim the underlying dma_buf leaving the parent dmabuf file free (but still referenced by a dangling file descriptor)
3.  Use ged_dmabuf_set_name on our dangling file to place a new name pointer in the fake dma_buf struct
4.  Read the fake dma_buf struct (which is really a GE buffer) to get the name pointer
5.  Free the name pointer by calling ged_dmabuf_set_name again
6.  Reclaim the name pointer as an fdtable with references to a GE fd with an array of GE buffers
7.  Use the arbitrary read to find the GE buffer array
8.  Use the arbitrary free to free the GE buffer array
9.  Reclaim the GE buffer array with another GE buffer

At the end of this process we’ll have a reliable arbitrary read/write!

### Getting a root shell

As a fun exercise, I decided to see how easy it was to disable SELinux and get root after achieving arbitrary read/write. Various manufacturers may implement certain tripping hazards to slow down exploit development efforts, but in my case (Asus ROG 6D), there were no hoops I needed to jump through at all. It was enough to simply write 0 to the uid/gid of my process’s cred struct to achieve root, and write 0 to the selinux_enforcing bit to turn off SELinux. After this I just execlp(“/system/bin/sh”,...) and out pops a root shell!

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgAy_jgDPyEiAmxQxmtSQXC1Q4eQ2alAYsJFiGXLUAFRi-OrljxWON-M1NBxe8LfwO9PWGBRdq-feWA8J3mKl5TTfhlHpAPVuFaH5WSSjCXVUKoj5840vfXf-iGNe1Q9QlFmOENGbYHZHwGqIPbLvAhbpATKcJ6VxhrXaAH3zsh7k_t3pH2so7Zs-QU8ro/s1200/image3.gif)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgAy_jgDPyEiAmxQxmtSQXC1Q4eQ2alAYsJFiGXLUAFRi-OrljxWON-M1NBxe8LfwO9PWGBRdq-feWA8J3mKl5TTfhlHpAPVuFaH5WSSjCXVUKoj5840vfXf-iGNe1Q9QlFmOENGbYHZHwGqIPbLvAhbpATKcJ6VxhrXaAH3zsh7k_t3pH2so7Zs-QU8ro/s1422/image3.gif)

Conclusion
----------

I discovered significant security vulnerabilities across all 3 of the evaluated devices. It is highly likely that reviewing more devices comprising a greater spread of chipset manufacturers would lead to the discovery of additional vulnerabilities. Android regularly uses higher-privileged processes to liaise between applications and kernel drivers, meaning that most kernel drivers cannot be seen from an unprivileged app context (the GPU being the most obvious exception to this rule). Nevertheless, a determined attacker could use vulnerabilities in other more privileged processes to pivot into contexts from which the attack surface of these kernel drivers become reachable. This pivot strategy could widen the attack surface beyond the scope of this research.

As it becomes more difficult to find 0-days in core Android, third-party Linux kernel drivers continue to become a more and more attractive target for attackers. While the bulk of present-day detected ITW Android exploitation targets GPU drivers, it’s equally important that other third-party drivers are encouraged towards the same security standards.

There is room for improvement in the patching process across all 3 of the bugs discovered. None of the patches for these bugs met Project Zero's 90-day deadline for patches reaching end-users. This appears to largely be a result of the propagation delay from when third-party driver developers issue patches to when downstream manufacturers can incorporate those patches into Android security updates. Shortening this propagation delay (e.g. using [Android APEX](https://source.android.com/docs/core/ota/apex) to ship updated kernel drivers) would go a long way to minimizing the Android driver patch gap. In addition, one of these bugs was only partially patched and remained exploitable on some devices for an additional 2 years before the security impact of the bug was assessed and publicly reported. Developers should regularly consider the security impacts of bugs, especially those reported by static analysis tools designed to detect security-relevant issues.

Finally, while cross-cache heap shaping mitigations significantly impede exploit development strategies, they don’t entirely prevent a determined attacker from exploiting kernel UAF vulnerabilities, even if the UAF’d object comes out of a dedicated slab cache. In this particular case, second-order allocations in a UAF’d object lead to powerful and exploitable primitives. Developers may be able to mitigate this technique by setting pointers to NULL, even if the parent object is about to get freed anyway. However, this exploit technique demonstrates that even well-designed mitigations (such as SLAB_VIRTUAL) come with limitations in an era where an attacker can achieve undetected memory corruption. It will take more fundamental mitigations that address the root issue of memory corruption, like MTE, in order to dramatically raise the bar for attackers.

Resources
---------

This research was presented at ShmooCon, a video is available here [https://archive.org/details/shmoocon2024/Shmoocon2024-SethJenkins-Driving_Forward_in_Android_Drivers.mp4](https://archive.org/details/shmoocon2024/Shmoocon2024-SethJenkins-Driving_Forward_in_Android_Drivers.mp4)

The proof of concept exploit code developed and presented in this research is available at:

[https://bugs.chromium.org/p/project-zero/issues/detail?id=2470#c4](https://bugs.chromium.org/p/project-zero/issues/detail?id=2470#c4)