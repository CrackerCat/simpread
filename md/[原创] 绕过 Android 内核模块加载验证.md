> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271179.htm)

> [原创] 绕过 Android 内核模块加载验证

绕过 Android 内核模块加载验证
===================

> 以小米 10 为例，系统版本：MIUI 12.5.10.0

一、boot.img 的提取
==============

获取 boot.img 有两个途径：

1.  官方的镜像包里面提取
2.  手机里面提取（需要 root 权限）

[](#方法一、官方镜像包提取)方法一、官方镜像包提取
---------------------------

1.  前往 [XiaomiRom](https://xiaomirom.com/)，寻找对应的线刷包
    
    本文示例 ROM 下载地址：[https://xiaomirom.com/download/mi-10-umi-stable-V12.5.10.0.RJBCNXM/#china-fastboot](https://xiaomirom.com/download/mi-10-umi-stable-V12.5.10.0.RJBCNXM/#china-fastboot)
    
2.  解压，在 images 文件夹里面找到 boot.img
    

[](#方法二、手机里面提取)方法二、手机里面提取
-------------------------

adb shell 执行下面的命令（需要 root 权限）

```
dd if=$(readlink /dev/block/by-name/boot) of=/sdcard/boot.img

```

把 boot.img 拉到电脑上来

```
adb pull /sdcard/boot.img boot.img

```

二、解包 boot.img 获取 kernel
=======================

推荐使用 magiskboot

 

自行编译：https://github.com/topjohnwu/Magisk 获取 magiskboot 可执行文件

```
./magiskboot unpack boot.img

```

![](https://bbs.pediy.com/upload/tmp/892096_9UZZXT2S848MGRV.png)

 

kernel 文件就是我们需要的内核二进制文件

三、patch kernel 去除内核模块加载验证
=========================

内核模块加载验证分析
----------

insmod 使用 sys_init_module 系统调用，去源码中查看

```
SYSCALL_DEFINE3(init_module, void __user *, umod,
        unsigned long, len, const char __user *, uargs)
{
    int err;
    struct load_info info = { };
 
    err = may_init_module();
    if (err)
        return err;
 
    pr_debug("init_module: umod=%p, len=%lu, uargs=%p\n",
           umod, len, uargs);
 
    err = copy_module_from_user(umod, len, &info);
    if (err)
        return err;
 
    return load_module(&info, uargs, 0);
}

```

查看 load_module 主函数

```
/* Allocate and load the module: note that size of section 0 is always
   zero, and we rely on this for optional sections. */
static int load_module(struct load_info *info, const char __user *uargs,
               int flags)
{
    struct module *mod;
    long err = 0;
    char *after_dashes;
 
    /*
     * Do the signature check (if any) first. All that
     * the signature check needs is info->len, it does
     * not need any of the section info. That can be
     * set up later. This will minimize the chances
     * of a corrupt module causing problems before
     * we even get to the signature check.
     *
     * The check will also adjust info->len by stripping
     * off the sig length at the end of the module, making
     * checks against info->len more correct.
     */
    err = module_sig_check(info, flags); // 第一处签名校验
    if (err)
        goto free_copy;
 
    /*
     * Do basic sanity checks against the ELF header and
     * sections.
     */
    err = elf_validity_check(info);
    if (err) {
        pr_err("Module has invalid ELF structures\n");
        goto free_copy;
    }
 
    /*
     * Everything checks out, so set up the section info
     * in the info structure.
     */
    err = setup_load_info(info, flags);
    if (err)
        goto free_copy;
 
    /*
     * Now that we know we have the correct module name, check
     * if it's blacklisted.
     */
    if (blacklisted(info->name)) {
        err = -EPERM;
        goto free_copy;
    }
 
    err = rewrite_section_headers(info, flags);
    if (err)
        goto free_copy;
 
    // modstruct 版本校验
    /* Check module struct version now, before we try to use module. */
    if (!check_modstruct_version(info, info->mod)) {
        err = -ENOEXEC;
        goto free_copy;
    }
 
    /* Figure out module layout, and allocate all the memory. */
    mod = layout_and_allocate(info, flags);
    if (IS_ERR(mod)) {
        err = PTR_ERR(mod);
        goto free_copy;
    }
 
    audit_log_kern_module(mod->name);
 
    /* Reserve our place in the list. */
    err = add_unformed_module(mod);
    if (err)
        goto free_module;
 
/*第二处签名校验*/
#ifdef CONFIG_MODULE_SIG
    mod->sig_ok = info->sig_ok;
    if (!mod->sig_ok) {
        pr_notice_once("%s: module verification failed: signature "
                   "and/or required key missing - tainting "
                   "kernel\n", mod->name);
        add_taint_module(mod, TAINT_UNSIGNED_MODULE, LOCKDEP_STILL_OK);
    }
#endif
 
    /* To avoid stressing percpu allocator, do this once we're unique. */
    err = percpu_modalloc(mod, info);
    if (err)
        goto unlink_mod;
 
    /* Now module is in final location, initialize linked lists, etc. */
    err = module_unload_init(mod);
    if (err)
        goto unlink_mod;
 
    init_param_lock(mod);
 
    /* Now we've got everything in the final locations, we can
     * find optional sections. */
    err = find_module_sections(mod, info);
    if (err)
        goto free_unload;
 
    err = check_module_license_and_versions(mod);
    if (err)
        goto free_unload;
 
    /* Set up MODINFO_ATTR fields */
    setup_modinfo(mod, info);
 
    /* Fix up syms, so that st_value is a pointer to location. */
    err = simplify_symbols(mod, info);
    if (err < 0)
        goto free_modinfo;
 
    err = apply_relocations(mod, info);
    if (err < 0)
        goto free_modinfo;
 
    err = post_relocation(mod, info);
    if (err < 0)
        goto free_modinfo;
 
    flush_module_icache(mod);
 
    /* Now copy in args */
    mod->args = strndup_user(uargs, ~0UL >> 1);
    if (IS_ERR(mod->args)) {
        err = PTR_ERR(mod->args);
        goto free_arch_cleanup;
    }
 
    dynamic_debug_setup(mod, info->debug, info->num_debug);
 
    /* Ftrace init must be called in the MODULE_STATE_UNFORMED state */
    ftrace_module_init(mod);
 
    /* Finally it's fully formed, ready to start executing. */
    err = complete_formation(mod, info);
    if (err)
        goto ddebug_cleanup;
 
    err = prepare_coming_module(mod);
    if (err)
        goto bug_cleanup;
 
    /* Module is ready to execute: parsing args may do that. */
    after_dashes = parse_args(mod->name, mod->args, mod->kp, mod->num_kp,
                  -32768, 32767, mod,
                  unknown_module_param_cb);
    if (IS_ERR(after_dashes)) {
        err = PTR_ERR(after_dashes);
        goto coming_cleanup;
    } else if (after_dashes) {
        pr_warn("%s: parameters '%s' after `--' ignored\n",
               mod->name, after_dashes);
    }
 
    /* Link in to sysfs. */
    err = mod_sysfs_setup(mod, info, mod->kp, mod->num_kp);
    if (err < 0)
        goto coming_cleanup;
 
    if (is_livepatch_module(mod)) {
        err = copy_module_elf(mod, info);
        if (err < 0)
            goto sysfs_cleanup;
    }
 
    /* Get rid of temporary copy. */
    free_copy(info);
 
    /* Done! */
    trace_module_load(mod);
 
    return do_init_module(mod);
 
 sysfs_cleanup:
    mod_sysfs_teardown(mod);
 coming_cleanup:
    mod->state = MODULE_STATE_GOING;
    destroy_params(mod->kp, mod->num_kp);
    blocking_notifier_call_chain(&module_notify_list,
                     MODULE_STATE_GOING, mod);
    klp_module_going(mod);
 bug_cleanup:
    mod->state = MODULE_STATE_GOING;
    /* module_bug_cleanup needs module_mutex protection */
    mutex_lock(&module_mutex);
    module_bug_cleanup(mod);
    mutex_unlock(&module_mutex);
 
 ddebug_cleanup:
    ftrace_release_mod(mod);
    dynamic_debug_remove(mod, info->debug);
    synchronize_rcu();
    kfree(mod->args);
 free_arch_cleanup:
    module_arch_cleanup(mod);
 free_modinfo:
    free_modinfo(mod);
 free_unload:
    module_unload_free(mod);
 unlink_mod:
    mutex_lock(&module_mutex);
    /* Unlink carefully: kallsyms could be walking list. */
    list_del_rcu(&mod->list);
    mod_tree_remove(mod);
    wake_up_all(&module_wq);
    /* Wait for RCU-sched synchronizing before releasing mod->list. */
    synchronize_rcu();
    mutex_unlock(&module_mutex);
 free_module:
    /* Free lock-classes; relies on the preceding sync_rcu() */
    lockdep_free_key_range(mod->core_layout.base, mod->core_layout.size);
 
    module_deallocate(mod, info);
 free_copy:
    free_copy(info);
    return err;
}

```

影响加载验证的主要有两个：

1.  module_sig_check 模块签名校验
2.  check_modstruct_version 版本校验。

check_modstruct_version 函数源码：

```
static inline int check_modstruct_version(const struct load_info *info,
                      struct module *mod)
{
    const s32 *crc;
 
    /*
     * Since this should be found in kernel (which can't be removed), no
     * locking is necessary -- use preempt_disable() to placate lockdep.
     */
    preempt_disable();
    if (!find_symbol("module_layout", NULL, &crc, NULL, true, false)) {
        preempt_enable();
        BUG();
    }
    preempt_enable();
    return check_version(info, "module_layout", mod, crc);
}

```

check_version 函数源码：

```
static int check_version(const struct load_info *info,
             const char *symname,
             struct module *mod,
             const s32 *crc)
{
    Elf_Shdr *sechdrs = info->sechdrs;
    unsigned int versindex = info->index.vers;
    unsigned int i, num_versions;
    struct modversion_info *versions;
 
    /* Exporting module didn't supply crcs?  OK, we're already tainted. */
    if (!crc)
        return 1;
 
    /* No versions at all?  modprobe --force does this. */
    if (versindex == 0)
        return try_to_force_load(mod, symname) == 0;
 
    versions = (void *) sechdrs[versindex].sh_addr;
    num_versions = sechdrs[versindex].sh_size
        / sizeof(struct modversion_info);
 
    for (i = 0; i < num_versions; i++) {
        u32 crcval;
 
        if (strcmp(versions[i].name, symname) != 0)
            continue;
 
        if (IS_ENABLED(CONFIG_MODULE_REL_CRCS))
            crcval = resolve_rel_crc(crc);
        else
            crcval = *crc;
        if (versions[i].crc == crcval)
            return 1;
        pr_debug("Found checksum %X vs module %lX\n",
             crcval, versions[i].crc);
        goto bad_version;
    }
 
    /* Broken toolchain. Warn once, then let it go.. */
    pr_warn_once("%s: no symbol version for %s\n", info->name, symname);
    return 1;
 
bad_version:
    pr_warn("%s: disagrees about version of symbol %s\n",
           info->name, symname);
    return 0;
}

```

module_sig_check 函数源码：

```
static int module_sig_check(struct load_info *info, int flags)
{
    int err = -ENODATA;
    const unsigned long markerlen = sizeof(MODULE_SIG_STRING) - 1;
    const char *reason;
    const void *mod = info->hdr;
 
    /*
     * Require flags == 0, as a module with version information
     * removed is no longer the module that was signed
     */
    if (flags == 0 &&
        info->len > markerlen &&
        memcmp(mod + info->len - markerlen, MODULE_SIG_STRING, markerlen) == 0) {
        /* We truncate the module to discard the signature */
        info->len -= markerlen;
        err = mod_verify_sig(mod, info);
    }
 
    switch (err) {
    case 0:
        info->sig_ok = true; // 关键点！
        return 0;
 
        /* We don't permit modules to be loaded into trusted kernels
         * without a valid signature on them, but if we're not
         * enforcing, certain errors are non-fatal.
         */
    case -ENODATA:
        reason = "unsigned module";
        break;
    case -ENOPKG:
        reason = "module with unsupported crypto";
        break;
    case -ENOKEY:
        reason = "module with unavailable key";
        break;
 
        /* All other errors are fatal, including nomem, unparseable
         * signatures and signature check failures - even if signatures
         * aren't required.
         */
    default:
        return err;
    }
 
    if (is_module_sig_enforced()) {
        pr_notice("Loading of %s is rejected\n", reason);
        return -EKEYREJECTED;
    }
 
    return security_locked_down(LOCKDOWN_MODULE_SIGNATURE);
}

```

内核模块加载验证去除
----------

### patch 思路

1.  强制 check_version 返回 1
2.  module_sig_check 返回 0 并且让 info->sig_ok = true;

小米 10 的内核并没有开启签名校验，所以只需要让 check_version 返回 1 即可，这个是很好修改的，在实践中，module_sig_check 比较难改，通常它会被内联进 load_module 函数，此时需要分析 load_module 函数来进行定位修改。

### 操作流程

check_version 是非常好定位的，可以用字符串 “%s: disagrees about version of symbol %s\n” 来很方便地定位它。

 

使用 ida 打开后，它没有自动开始分析，手动

 

![](https://bbs.pediy.com/upload/tmp/892096_23GTSJ6GRJ2MTAW.png)

 

使用 ida python，手动 MakeCode 让它开始分析

```
import idc
for i in range(0, 0x100000):
    idc.MakeCode(i)

```

分析完毕后，在 strings 中搜索上述字符串，查找交叉引用即可找到 check_version 函数

 

当然也有取巧的办法：

1.  找到字符串的位置
2.  用 keystone 获取 “add x0, x0, #(位置 %0x1000)” 的汇编代码
3.  搜索上述汇编代码
4.  check_version 函数就在附近

```
from capstone import *
from keystone import *
 
def bin_finder(kernel_image: bytes, bin_data: bytes):
    return kernel_image.find(bin_data)
 
def string_finder(kernel_image: bytes, string: str):
    return bin_finder(kernel_image, string.encode("utf-8"))
 
if __name__ == '__main__':
    fp = open("kernel/kernel_10_12_5_10", "rb")
    kernel_image_data = fp.read()
    fp.close()
    offset = bin_finder(kernel_image_data, b"\x014%s: no symbol version for %s")
    print("string at:" + str(hex(offset)))
    ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
    code, stat_count = ks.asm("add    x0, x0, #" + str(hex(offset % 0x1000)), as_bytes=True)
    print(code)
    offset = bin_finder(kernel_image_data, code)
    print("xref to:" + str(hex(offset)))
    begin = offset - 0x100
    end = offset + 0x100
    function_bin = kernel_image_data[begin:end]
    cs = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
    for c in cs.disasm(function_bin, begin):
        print("0x%x: %s %s" % (c.address, c.mnemonic, c.op_str))

```

运行脚本可以获得输出 “xref to:0x121520”

 

可得函数 sub_121498 即为 check_version

 

![](https://bbs.pediy.com/upload/tmp/892096_JK6GXDH82RPYE7U.png)

 

把 CBZ X2, loc_121544 改为 B loc_121544 即可

 

![](https://bbs.pediy.com/upload/tmp/892096_XVAJCQFQCPYGNG4.png)

 

![](https://bbs.pediy.com/upload/tmp/892096_HWVUM3TZF6T4SS3.png)

 

应用修改到文件即可。

四、重新打包 boot.img
===============

还是使用 magiskboot，替换掉 kernel 后，执行：

```
./magiskboot repack boot.img

```

可以获得 new-boot.img

五、刷入新的 boot.img
===============

```
adb reboot fastboot
fastboot flash boot boot.img

```

[](#六、最终)六、最终
=============

现在的内核已经可以加载模块了，但是编译出来的模块还要经过 “后处理”

1.  自行编译模块
2.  修改模块的 vermagic
3.  修改模块的 module_layout

使用工具 vermagic：https://github.com/yaxinsn/vermagic

> vermagic 和 module_layout 可以在手机里面随便找个系统 ko 获得

 

执行：

```
./vermagic -c "{module_layout, 0X783EB3B8}" hello.ko
./vermagic -v "4.19.113-perf-g545a49d08f11 SMP preempt mod_unload modversions aarch64" hello.ko

```

测试效果：

 

![](https://bbs.pediy.com/upload/tmp/892096_B8X2QD5KNK4PJ9U.png)

[【公告】看雪团队招聘安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

最后于 20 小时前 被 Ylarod 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#系统相关](forum-161-1-126.htm)