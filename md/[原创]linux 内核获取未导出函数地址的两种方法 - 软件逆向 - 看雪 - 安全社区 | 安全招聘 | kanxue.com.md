> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282094.htm)

> [原创]linux 内核获取未导出函数地址的两种方法

第一种
===

第一种是借助于 kprobe 机制，通过 kprobe 机制中会调用 kallsyms_lookup_name 函数并设置到 kprobe 结构体中返回的原理找到我们需要的函数地址

内核中调用逻辑简化代码如下：

```
int register_kprobe(struct kprobe *p)
{
    int ret;
    struct kprobe *old_p;
    struct module *probed_mod;
    kprobe_opcode_t *addr;
 
    /* Adjust probe address from symbol */
    addr = kprobe_addr(p);
    if (IS_ERR(addr))
        return PTR_ERR(addr);
    p->addr = addr;
    。。。。。。
}
static kprobe_opcode_t *kprobe_addr(struct kprobe *p)
{
    return _kprobe_addr(p->addr, p->symbol_name, p->offset);
}
static kprobe_opcode_t *_kprobe_addr(kprobe_opcode_t *addr,
            const char *symbol_name, unsigned int offset)
{
    if ((symbol_name && addr) || (!symbol_name && !addr))
        goto invalid;
 
    if (symbol_name) {
        addr = kprobe_lookup_name(symbol_name, offset);
        if (!addr)
            return ERR_PTR(-ENOENT);
    }
    。。。。。
}
kprobe_opcode_t * __weak kprobe_lookup_name(const char *name,
                    unsigned int __unused)
{
    return ((kprobe_opcode_t *)(kallsyms_lookup_name(name)));
}

```

如上逻辑可以看到其实就是调用的 kallsyms_lookup_name 函数来获取地址存到我们的 addr 中

第二种
===

第二种方式是借助于内核调试器提供的能力间接调用 kallsyms_lookup_name 函数获取地址

在内核中有这样一个函数并且这个函数是导出的

```
extern int kdbgetsymval(const char *, kdb_symtab_t *);
 
int kdbgetsymval(const char *symname, kdb_symtab_t *symtab)
{
    if (KDB_DEBUG(AR))
        kdb_printf("kdbgetsymval: symname=%s, symtab=%px\n", symname,
               symtab);
    memset(symtab, 0, sizeof(*symtab));
    symtab->sym_start = kallsyms_lookup_name(symname);
    if (symtab->sym_start) {
        if (KDB_DEBUG(AR))
            kdb_printf("kdbgetsymval: returns 1, "
                   "symtab->sym_start=0x%lx\n",
                   symtab->sym_start);
        return 1;
    }
    if (KDB_DEBUG(AR))
        kdb_printf("kdbgetsymval: returns 0\n");
    return 0;
}
EXPORT_SYMBOL(kdbgetsymval);

```

代码示例
====

```
/*************************************************************************
    > File Name: test.c
    > Author:
    > Mail:
    > Created Time: 2024年06月09日 星期日 11时34分43秒
    > 两种获取内核未导出函数地址方法
 ************************************************************************/
 
#include #include #include #include #include MODULE_LICENSE("GPL");
MODULE_AUTHOR("ch");
typedef struct __ksymtab {
        unsigned long value;    /* Address of symbol */
        const char *mod_name;   /* Module containing symbol or
                     * "kernel" */
        unsigned long mod_start;
        unsigned long mod_end;
        const char *sec_name;   /* Section containing symbol */
        unsigned long sec_start;
        unsigned long sec_end;
        const char *sym_name;   /* Full symbol name, including
                     * any version */
        unsigned long sym_start;
        unsigned long sym_end;
        } kdb_symtab_t;
extern int kdbgetsymval(const char *, kdb_symtab_t *);
struct char_device_struct {
    struct char_device_struct *next;
    unsigned int major;
    unsigned int baseminor;
    int minorct;
    char name[64];
    struct cdev *cdev;      /* will die */
};
 
int noop_pre(struct kprobe* p, struct pt_regs* regs) {return 0;}
static struct kprobe kp = {
    .symbol_name = "kallsyms_lookup_name",
};
unsigned long (*kallsyms_lookup_name_func)(const char* name) = NULL;
static int __init m_init(void) {
    kp.pre_handler = noop_pre;
    int ret = register_kprobe(&kp);
    if (ret != 0) {
        printk("find failed\n");
        return ret;
    }
    kallsyms_lookup_name_func = (void*)kp.addr;
    printk("lookup func addr is %p\n", kp.addr);
    unregister_kprobe(&kp);
    unsigned long chrdevs_addr = kallsyms_lookup_name_func("chrdevs");
    printk("chardevs addr is 0x%p\n", chrdevs_addr);
    kdb_symtab_t val;
    kdbgetsymval("chrdevs", &val);
    printk("second method get chrdevs addr is 0x%p\n", val.sym_start);
    return 0;
}
 
static void __exit m_exit(void) {
 
}
 
module_init(m_init);
module_exit(m_exit); 
```

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工 作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#系统底层](forum-4-1-2.htm)