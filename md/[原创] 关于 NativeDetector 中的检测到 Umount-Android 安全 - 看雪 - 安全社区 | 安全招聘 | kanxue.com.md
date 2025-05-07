> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286664.htm)

> [原创] 关于 NativeDetector 中的检测到 Umount

检测
--

`Native Detector`中能检测到:'/debug_rakdisk/.magisk/modules/xx/riru/lib64'  
![](https://bbs.kanxue.com/upload/attach/202504/98159_44PZJG2AG8A5B8P.webp)

猜测
--

还是直接上 ce 搜索:  
![](https://bbs.kanxue.com/upload/attach/202504/98159_2Y465VU8DCWA5FV.webp)  
对一下区块, 最可疑的就是`[anon:stack_and_tls:main]`

```
redmi:/ # cat /proc/1097/maps | grep 7a077
7a076a0000-7a07768000 r--p 00000000 00:00 0                              [anon:linker_alloc]
7a07768000-7a07788000 r--s 00000000 00:11 13760                          /dev/__properties__/u:object_r:vndk_prop:s0
7a07788000-7a0778b000 rw-p 00000000 00:00 0                              [anon:bionic_alloc_small_objects]
7a0778b000-7a0778d000 r--p 00041000 07:78 53                             /apex/com.android.art/javalib/arm64/boot-core-icu4j.art
7a0778d000-7a07791000 rw-p 00000000 00:00 0                              [anon:bionic_alloc_small_objects]
7a07791000-7a07792000 r--p 0001d000 07:78 56                             /apex/com.android.art/javalib/arm64/boot-core-libart.art
7a07792000-7a07794000 rw-p 00000000 00:00 0                              [anon:bionic_alloc_small_objects]
7a07794000-7a07795000 rw-p 00000000 00:00 0                              [anon:bionic_alloc_lob]
7a07795000-7a0779a000 rw-p 00000000 00:00 0                              [anon:System property context nodes]
7a0779a000-7a0779b000 ---p 00000000 00:00 0
7a0779b000-7a0779f000 rw-p 00000000 00:00 0                              [anon:stack_and_tls:main]
7a0779f000-7a077a0000 ---p 00000000 00:00 0
7a077a0000-7a077a4000 rw-p 00000000 00:00 0                              [anon:bionic_alloc_small_objects]
7a077a4000-7a077a5000 rw-p 00000000 00:00 0                              [anon:ReadFileToBuffer]
7a077a5000-7a077ab000 rw-p 00000000 00:00 0                              [anon:bionic_alloc_small_objects]
7a077ab000-7a0780f000 r--p 00000000 00:00 0                              [anon:linker_alloc]
```

找到内存区块后, 难道是通过哪个指针泄露的?  
随便改点字符就没有了, 再下面随便复制一下是相同的字符串, 又能搜索到了, 证明只是通过`[anon:stack_and_tls:main]`的字符串`/debug_ramdisk/.magisk/`搜索

![](https://bbs.kanxue.com/upload/attach/202504/98159_US7P9UAMYD9EQW6.webp)  
![](https://bbs.kanxue.com/upload/attach/202504/98159_D59MWPX673DSXZU.webp)

探索
--

> 这个指针是从哪来的呢?

```
if (android_prop::GetApiLevel() >= __ANDROID_API_O__) {
        auto *dir = dirname(path);
        auto *ns = &__loader_android_create_namespace == nullptr ? nullptr :
                   __loader_android_create_namespace(path, dir, nullptr,
                                                     2/*ANDROID_NAMESPACE_TYPE_SHARED*/,
                                                     nullptr, nullptr,
                                                     reinterpret_cast(&DlopenExt));
        if (ns) {
            info.flags = ANDROID_DLEXT_USE_NAMESPACE;
            info.library_namespace = ns;
 
            LOGD("open %s with namespace %p, dir: %p", path, ns, dir);
        } else {
            LOGD("cannot create namespace for %s", path);
        }
    } 
```

就是`android`的`DlopenExt`方法里的, 当要`__loader_android_create_namespace`时会有一个`dirname`查询目录.

结论
--

可以在每次`open`之后直接清除这个字符串

```
// 直接清除
memset(dir, 0, strlen(dir));
```

也可以随便改一下关键字, 或者自己也搜索内存改下算了.

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

最后于 2025-4-29 09:55 被无味编辑 ，原因： 优化

[#漏洞相关](forum-161-1-123.htm) [#其他](forum-161-1-129.htm) [#工具脚本](forum-161-1-128.htm)