> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [panda0s.top](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/)

> plugins\hexrays_sdk\plugins 目录下有很多 demo 值得学习

plugins\hexrays_sdk\plugins 目录下有很多 demo 值得学习

20 个 demo 程序，断断续续分析了大概一个月的样子，最后还是发出来吧，毕竟这方面的资料太少了。

[](#一些基本的概念 "一些基本的概念")一些基本的概念[](#一些基本的概念)
-----------------------------------------

Microcode 是 IDA 内部的一种介于机器语言与伪代码的之间的中间语言，用于抽象机器代码，便于后面的优化。

Microcode 展示插件: [https://github.com/gaasedelen/lucid](https://github.com/gaasedelen/lucid)

Microcode 是分阶段生成的，类似与 llvm 中的 pass ，每个阶段 Microcode 都可以得到一定的优化，最终得到最优结果。

IDA 的 API 支持在 Microcode 生成之前调用用户的 Filter 重写机器指令到 Microcode 的翻译 (lift), 也支持用户自定义优化规则。用户自定义的优化规则包括指令间优化与基本块间的优化。

IDA 提供直接生成 Microcode 的 API 函数，并提供数据流信息，使得我们可以很方便地编写静态代码分析插件。

Microcode 生成完成后，IDA 在 Microcode 的基础上生成 CTree。 CTree 是 IDA 内部用于表示 C 语言伪代码的抽象语法树，IDA 也提供了大量 API 操作 CTree，可以实现一下伪代码展示方面的优化，例如删除某些节点等等。

Microcode 可视化插件

[https://github.com/gaasedelen/lucid](https://github.com/gaasedelen/lucid)

Ctree 可视化插件  
[https://github.com/patois/HRDevHelper](https://github.com/patois/HRDevHelper)

Microcode 指令格式

opcode left, right, destination  
一般来说有三个操作数，有一些指令可能缺少某个操作数，destination 也不一定会被修改（Store 指令）

每一个操作数都带有一个宽度描述符，描述的是该操作数的字节大小，例如 `eax.4` 表示 eax 是一个 4 字节的操作数。

Microcode 中的寄存器

Microcode 中寄存器都是虚拟寄存器，寄存器的数量没有限制。一般情况，物理寄存器会直接映射到虚拟寄存器。

Microcode 中常见的数据结构

函数是 IDA 中最大的汇编结果表示单位

函数 → 基本块 → 指令 → 操作数

每一级都有对应的数据结构来表示。

`mbl_array_t` 可以说是最顶层的数据结构，每个函数都拥有该类型的字段，存储了基本块信息。

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled.png "Untitled")

基本块是一个双向链表，其对应的类型是 `mblock_t` 。注意基本块本身逻辑前驱与后继由 `mblock_t` 内部的 `predset` 和 `succset` 维护而不是由双向链表维护。在上图中，natural 以数组形式存放基本块。

`mblock_t` 的结构如下

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled1.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled1.png "Untitled")

基本块内的指令也是以双向链表的形式组织，指令的类型是 `minsn_t`

`minsn_t` 的结构图如下

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled2.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled2.png "Untitled")

你没有看错，这其实是一条指令，只不过该指令由多条指令嵌套而来。IDA 允许指令的操作数是另一条指令的 dest, 这使得在做一些优化的时候很方便，我们在后面会多次遇到这种情况。最顶层的指令叫做 top-level 指令，在 `mblock_t` 链表中存放的指令就只有 top-level 指令。指令的下一级是操作数，对应的数据结构是 `mop_t`

`mop_t` 结构的定义如下

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled3.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled3.png "Untitled")

在`mop_t` 结构中我们可以很容易的找到 `mop_t` 对嵌套指令的支持。

[](#环境配置 "环境配置")环境配置[](#环境配置)
-----------------------------

我调试的环境用的 VS2019 。经过我测试，VS2019 可以编译 IDA 7.6 插件并顺利运行和调试。

贴一下 VS 的配置项

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled4.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled4.png "Untitled")

调试配置

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled5.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled5.png "Untitled")

包含目录 (Include 目录) 添加 sdk 目录与 plugin 的 include 目录

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled6.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled6.png "Untitled")

库目录添加上 ida sdk 的预编译库目录

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled7.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled7.png "Untitled")

添加链接器参数

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled8.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled8.png "Untitled")

/EXPORT:PLUGIN

[](#vds1 "vds1")vds1[](#vds1)
-----------------------------

在 output-windows 输出指定函数的伪代码。

```
func_t* pfn = get_func(get_screen_ea());
hexrays_failure_t hf;
cfuncptr_t cfunc = decompile(pfn, &hf, DECOMP_WARNINGS);
const strvec_t& sv = cfunc->get_pseudocode();
for (int i = 0; i < sv.size(); i++)
{
    qstring buf;
    tag_remove(&buf, sv[i].line);
    msg("%s\\n", buf.c_str());
}
```

cfuncptr_t 的定义如下:

```
struct cfunc_t
{
  ea_t entry_ea;             
  mba_t *mba;                
  cinsn_t body;              
  intvec_t &argidx;          
  ctree_maturity_t maturity; 
  
  
  user_labels_t *user_labels;
  user_cmts_t *user_cmts;    
  user_numforms_t *numforms; 
  user_iflags_t *user_iflags;
  user_unions_t *user_unions;


#define CIT_COLLAPSED 0x0001 

  int refcnt;                
  int statebits;             

#define CFS_BOUNDS       0x0001 
#define CFS_TEXT         0x0002 
#define CFS_LVARS_HIDDEN 0x0004 
#define CFS_LOCKED       0x0008 
  eamap_t *eamap;            
  boundaries_t *boundaries;  
  strvec_t sv;               
  int hdrlines;              
  mutable ctree_items_t treeitems; 

  
  char reserved[];

public:
  cfunc_t(mba_t *mba);          
  ~cfunc_t(void) { cleanup(); }
  void release(void) { delete this; }
  HEXRAYS_MEMORY_ALLOCATION_FUNCS()

  
  
  void hexapi build_c_tree(void);

  
  
  
  
  
  void hexapi verify(allow_unused_labels_t aul, bool even_without_debugger) const;

  
  
  void hexapi print_dcl(qstring *vout) const;

  
  
  void hexapi print_func(vc_printer_t &vp) const;

  
  
  
  bool hexapi get_func_type(tinfo_t *type) const;

  
  
  
  
  
  
  lvars_t *hexapi get_lvars(void);

  
  
  
  
  
  sval_t hexapi get_stkoff_delta(void);

  
  
  citem_t *hexapi find_label(int label);

  
  
  
  void hexapi remove_unused_labels(void);

  
  
  
  
  const char *hexapi get_user_cmt(const treeloc_t &loc, cmt_retrieval_type_t rt) const;

  
  
  
  
  
  void hexapi set_user_cmt(const treeloc_t &loc, const char *cmt);

  
  
  
  int32 hexapi get_user_iflags(const citem_locator_t &loc) const;

  
  
  
  void hexapi set_user_iflags(const citem_locator_t &loc, int32 iflags);

  
  bool hexapi has_orphan_cmts(void) const;

  
  
  int hexapi del_orphan_cmts(void);

  
  
  
  
  bool hexapi get_user_union_selection(ea_t ea, intvec_t *path);

  
  
  
  
  void hexapi set_user_union_selection(ea_t ea, const intvec_t &path);

  
  void hexapi save_user_labels(void) const;
  
  void hexapi save_user_cmts(void) const;
  
  void hexapi save_user_numforms(void) const;
  
  void hexapi save_user_iflags(void) const;
  
  void hexapi save_user_unions(void) const;

  
  
  
  
  
  
  
  
  
  bool hexapi get_line_item(
        const char *line,
        int x,
        bool is_ctree_line,
        ctree_item_t *phead,
        ctree_item_t *pitem,
        ctree_item_t *ptail);

  
  
  hexwarns_t &hexapi get_warnings(void);

  
  
  eamap_t &hexapi get_eamap(void);

  
  
  boundaries_t &hexapi get_boundaries(void);

  
  
  const strvec_t &hexapi get_pseudocode(void);

  
  
  
  
  void hexapi refresh_func_ctext(void);

  bool hexapi gather_derefs(const ctree_item_t &ci, udt_type_data_t *udm=NULL) const;
  bool hexapi find_item_coords(const citem_t *item, int *px, int *py);
  bool locked(void) const { return (statebits & CFS_LOCKED) != 0; }
private:
  
  
  void hexapi cleanup(void);
  DECLARE_UNCOPYABLE(cfunc_t)
};
typedef qrefcnt_t<cfunc_t> cfuncptr_t;
```

[](#vds2 "vds2")vds2[](#vds2)
-----------------------------

这个插件将指针中的 0 替换成 Null。  
IDA 的反编译结果经常是动态变化的，为了实现这个任务，这个插件通过注册 IDA 的 hexrays 事件回调，当 ctree 生成完成后立即对 ctree 中的 0 进行替换。

注册 hexrays 回调

```
install_hexrays_callback(hr_callback, nullptr);
remove_hexrays_callback(hr_callback, nullptr);
```

回调函数中判断回调的类型

```
ssize_t idaapi plugin_ctx_t::hr_callback(
    void*,
    hexrays_event_t event,
    va_list va)
{
    if (event == hxe_maturity)
    {
        cfunc_t* cfunc = va_arg(va, cfunc_t*);
        ctree_maturity_t mat = va_argi(va, ctree_maturity_t);
        if (mat == CMAT_FINAL) 
            convert_zeroes(cfunc);
    }
    return 0;
}
```

event 参数由 IDA 传入回调类型，全部的回调类型与说明可以查看 hexrays.hpp 中 `enum hexrays_event_t` 定义代码与注释。  
`hxe_maturity` 回调会在 Ctree maturity 改变时调用，并且变参列表 va 的参数是:

```
cfunc_t * cfunc
ctree_maturity_t new_maturity
```

Ctree 与 MicroCode 一样，也是分阶段生成的，最后一个阶段叫做 `CMAT_FINAL`.  
该插件还会进一步判断 maturity 阶段，当完成最后的阶段后才调用 `convert_zeroes` 函数对目标函数进行进一步处理。

convert_zeroes 函数代码如下

```
static void convert_zeroes(cfunc_t* cfunc)
{
    
    
    if (!get_named_type(NULL, null_type, NTF_TYPE))
    {
        msg("%s type is missing, cannot convert zeroes to NULLs\\n", null_type);
        return;
    }

    
    
    
    
    
    
    struct ida_local zero_converter_t : public ctree_visitor_t
    {
        zero_converter_t(void) : ctree_visitor_t(CV_FAST) {}
        int idaapi visit_expr(cexpr_t* e) override
        {
            
            
            
            
            
            switch (e->op)
            {
            case cot_asg:   
                if (e->x->type.is_ptr()) 
                    make_null_if_zero(e->y);
                break;

            case cot_call:  
            {
                carglist_t& args = *e->a;
                for (int i = 0; i < args.size(); i++) 
                {
                    carg_t& a = args[i];
                    if (a.formal_type.is_ptr_or_array())
                        make_null_if_zero(&a);
                }
            }
            break;

            case cot_eq:    
            case cot_ne:
            case cot_sge:
            case cot_uge:
            case cot_sle:
            case cot_ule:
            case cot_sgt:
            case cot_ugt:
            case cot_slt:
            case cot_ult:
                
                if (e->y->type.is_ptr()) 
                    make_null_if_zero(e->x);
                if (e->x->type.is_ptr())
                    make_null_if_zero(e->y);
                break;

            default:
                break;

            }
            return 0; 
        }
    };
    zero_converter_t zc;
    
    zc.apply_to(&cfunc->body, NULL);
}
```

这段代码定义一个类 `zero_converter_t` 继承于 `ctree_visitor_t`，用观察者模式来访问函数中的所有 ctree 节点。并针对三种语法节点中的`0`进行处理。

1.  ptr = 0;
2.  func(0); where argument is a pointer
3.  ptr op 0 where op is a comparison

识别这三种语句的方法依赖于 Ctree 节点的 op 属性，匹配成功后调用 `make_null_if_zero` 进行进一步处理。  
make_null_if_zero 代码如下

```
static void make_null_if_zero(cexpr_t* e)
{
    if (e->is_zero_const() && !e->type.is_ptr())
    { 
        number_format_t& nf = e->n->nf;
        nf.flags = enum_flag(); 
        nf.serial = 0;         
        nf.props |= NF_VALID;
        nf.type_name = null_type;
        e->type.get_named_type(nullptr, null_type, BTF_ENUM);
    }
}
```

make_null_if_zero 代码中首先判断传入的表达式是否为 0 常量且类型是指针，如果是的话，将其类型改为 enum 并指定类型名为 MACRO_NULL

### [](#python-实现版本 "python 实现版本")python 实现版本[](#python-实现版本)

python 版本仅供参考，目前我实现的这份代码在替换 0 为 Null 之后会导致 IDA Internal Error.. 不清楚是什么原因

```
def make_null_if_zero(expr: idaapi.cexpr_t):
    if expr.is_zero_const():
        print("modify at:", hex(expr.ea))
        nf = expr.n.nf
        nf.flags = idaapi.enum_flag()
        nf.serial = 0
        nf.props = idaapi.NF_VALID
        nf.type_name = "MACRO_NULL"
        expr.type.get_named_type(None, "MACRO_NULL", idaapi.BTF_ENUM)

def convert_zeroes(cfunc: idaapi.cfunc_t):
    class zero_converter_t(idaapi.ctree_visitor_t):
        def __init__(self, *args):
            super().__init__(idaapi.CV_FAST)

        def visit_expr(self, *args) -> "int":
            e = args[0]
            if e.op == idaapi.cot_asg: # A
                if e.x.type.is_ptr():
                    make_null_if_zero(e.y)
            if e.op == idaapi.cot_call: # B
                for arg in e.a:
                    if arg.formal_type.is_ptr_or_array():
                        make_null_if_zero(arg)
            if e.op in [idaapi.cot_eq, idaapi.cot_ne, idaapi.cot_sge, idaapi.cot_uge, idaapi.cot_sle,idaapi.cot_ule, idaapi.cot_sgt, idaapi.cot_ugt, idaapi.cot_slt, idaapi.cot_ult]:
                if e.y.type.is_ptr():
                    make_null_if_zero(e.x)
                elif e.x.type.is_ptr():
                    make_null_if_zero(e.y)
            return 0

    zc = zero_converter_t()
    zc.apply_to(cfunc.body, None)

class vds2_hook_t(idaapi.Hexrays_Hooks):
    def __init__(self, *args):
        super().__init__(*args)

    def maturity(self, *args) -> "int": # alias for hxe_maturity
        mat = args[1]
        cfunc = args[0]
        if mat == idaapi.CMAT_FINAL:
            convert_zeroes(cfunc)
        return super().maturity(*args)
vds2_hooks = vds2_hook_t()
vds2_hooks.hook()
# vds2_hooks.unhook()
```

[](#vds3 "vds3")vds3[](#vds3)
-----------------------------

这个插件能够将反转 if 语句的条件

ctree 被修改后是一次性的，如果 ctree 重新创建，则之前的修改就会失效. 为了持久化, 该插件将修改信息存储在了 database，并在每一次 ctree 创建完成后重新对记录生效, 即使用户退出并重启 IDA 也是生效的。

### [](#handler-分析 "handler 分析")handler 分析[](#handler-分析)

handler 代码如下

```
-----------------------------

static ssize_t idaapi callback(void* ud, hexrays_event_t event, va_list va)
{
    vds3_t* plugmod = (vds3_t*)ud;
    switch (event)
    {
    case hxe_populating_popup:
    { 
        TWidget* widget = va_arg(va, TWidget*);
        TPopupMenu* popup = va_arg(va, TPopupMenu*);
        vdui_t& vu = *va_arg(va, vdui_t*);
        if (plugmod->find_if_statement(vu) != NULL)
            attach_action_to_popup(widget, popup, ACTION_NAME);
    }
    break;

    case hxe_maturity:
        if (!plugmod->inverted_ifs.empty())
        { 
            cfunc_t* cfunc = va_arg(va, cfunc_t*);
            ctree_maturity_t new_maturity = va_argi(va, ctree_maturity_t);
            if (new_maturity == CMAT_FINAL) 
                plugmod->convert_marked_ifs(cfunc);
        }
        break;

    default:
        break;
    }
    return 0;
}
```

handler 代码分别处理了 `hxe_populating_popup` 与 `hxe_maturity` 事件。

对于 `hxe_populating_popup` 事件，先判断用户选中的代码是否为 if-statement ，若是的话在菜单中插入 “”sample3:invertif” 选项。  
该事件的第二个参数的类型是 `vdui_t` ，该参数用来描述用户在 pseudocode 窗口中选中项目的一些属性。

vds3_t::find_if_statement 的代码如下

```
cinsn_t* vds3_t::find_if_statement(const vdui_t& vu)
{
    
    if (vu.item.is_citem())
    {
        cinsn_t* i = vu.item.i;
        
        
        if (i->op == cit_if && i->cif->ielse != NULL)
            return i;
    }
    
    
    
    
    
    
    if (vu.tail.citype == VDI_TAIL && vu.tail.loc.itp == ITP_ELSE)
    {
        
        
        
        struct ida_local if_finder_t : public ctree_visitor_t
        {
            ea_t ea;
            cinsn_t* found;
            if_finder_t(ea_t e)
                : ctree_visitor_t(CV_FAST | CV_INSNS), ea(e), found(NULL) {}
            int idaapi visit_insn(cinsn_t* i) override
            {
                if (i->op == cit_if && i->ea == ea)
                {
                    found = i;
                    return 1; 
                }
                return 0;
            }
        };
        if_finder_t iff(vu.tail.loc.ea);
        if (iff.apply_to(&vu.cfunc->body, NULL))
            return iff.found;
    }
    return NULL;
}
```

如果用户选中的是 if 且该 if 存在 else 就直接返回其对应的 ctree 指令对象指针。若用户选中的是 else，就有点复杂了。  
else 在 ctree 中没有任何对应的项，只能通过 vdui_t 中提供的信息来判断，判断代码如下

```
vu.tail.citype == VDI_TAIL && vu.tail.loc.itp == ITP_ELSE
```

识别出用户选中的 else 之后，需要在整个 ctree 中搜索该 else 对应的 if-statement, 并返回 if-statement 对应的指令。

### [](#hxe-maturity "hxe_maturity")hxe_maturity[](#hxe-maturity)

ctree 完成最后阶段生成后调用

```
plugmod->convert_marked_ifs(cfunc);
```

### [](#convert-marked-ifs-分析 "convert_marked_ifs 分析")convert_marked_ifs 分析[](#convert-marked-ifs-分析)

```
void vds3_t::convert_marked_ifs(cfunc_t* cfunc)
{
    
    struct ida_local if_inverter_t : public ctree_visitor_t
    {
        vds3_t* self;
        if_inverter_t(vds3_t* _self)
            : ctree_visitor_t(CV_FAST | CV_INSNS),
            self(_self) {}
        int idaapi visit_insn(cinsn_t* i) override
        {
            if (i->op == cit_if && self->inverted_ifs.has(i->ea))
                self->do_invert_if(i);
            return 0; 
        }
    };
    if_inverter_t ifi(this);
    ifi.apply_to(&cfunc->body, NULL); 
}
```

这段代码非常简单，遍历 cree 寻找 if-statement，并判断其是否需要反转。反转调用函数 `do_invert_if`

### [](#do-invert-if-分析 "do_invert_if 分析")do_invert_if 分析[](#do-invert-if-分析)

这是最核心的函数，即将 if 反转的核心函数，实际上非常简单，代码如下

```
void vds3_t::do_invert_if(cinsn_t* i) 
{
    QASSERT(30198, i->op == cit_if);
    cif_t& cif = *i->cif;
    
    cexpr_t* notcond = lnot(new cexpr_t(cif.expr));
    notcond->swap(cif.expr);
    delete notcond;
    
    qswap(cif.ielse, cif.ithen);
}
```

该函数将输入的 if-statement 中的 expr 替换为 not expr，并交换 then 与 else 指令。

[](#vds4 "vds4")vds4[](#vds4)
-----------------------------

略，与 ctree & microcode 关系不大

[](#vds5 "vds5")vds5[](#vds5)
-----------------------------

略，Ctree 图形化显示插件

[](#vds6 "vds6")vds6[](#vds6)
-----------------------------

略，删除输出代码的空格

[](#vds7 "vds7")vds7[](#vds7)
-----------------------------

这个插件演示了如何使用 `cblock_t::iterator`  
我们只关注核心代码，核心代码如下

```
struct ida_local cblock_visitor_t : public ctree_visitor_t
{
    cblock_visitor_t(void) : ctree_visitor_t(CV_FAST) {}
    int idaapi visit_insn(cinsn_t* ins) override
    {
        if (ins->op == cit_block)
            dump_block(ins->ea, ins->cblock);
        return 0;
    }
    void dump_block(ea_t ea, cblock_t* b)
    {
        
        msg("dumping block %a\\n", ea);
        for (cblock_t::iterator p = b->begin(); p != b->end(); ++p)
        {
            cinsn_t& i = *p;
            msg("  %a: insn %s\\n", i.ea, get_ctype_name(i.op));
        }
    }
};
cblock_visitor_t cbv;
cbv.apply_to(&func->body, NULL);
```

`cblock_t::iterator` 的使用方式与 STL 的迭代器很相似，调用很方便。

[](#vds8 "vds8")vds8[](#vds8)
-----------------------------

这个插件将 svc 0x900001 与 svc 0x9000F8 指令反编译成一条 call 指令, 这个功能与 IDA 菜单: Edit->Other->Decomile as call 类似。  
实现方法是通过 `install_microcode_filter` 注册 microcode filter.

```
class udc_exit_t : public udc_filter_t
{
    int code;
    bool installed;

public:
    udc_exit_t() : code(0), installed(false) {}
    bool prepare(int svc_code, const char* name)
    {
        char decl[MAXSTR];
        qsnprintf(decl, sizeof(decl), "int __usercall %s@<R0>(int status@<R1>);", name);
        bool ok = init(decl);
        if (!ok)
            msg("Could not initialize UDC plugin '%s'\\n", name);
        code = svc_code;
        return ok;
    }
    void install()
    {
        install_microcode_filter(this, true);
        installed = true;
    }
    void uninstall()
    {
        install_microcode_filter(this, false);
        installed = false;
    }
    void toggle_install()
    {
        if (installed)
            uninstall();
        else
            install();
    }
    virtual bool match(codegen_t& cdg) override
    {
        return cdg.insn.itype == ARM_svc && cdg.insn.Op1.value == code;
    }
    virtual ~udc_exit_t() {} 
};
```

`udc_exit_t` 类继承于 `udc_filter_t` 其定义如下

```
class udc_filter_t : public microcode_filter_t
{
  udcall_t udc;

public:
  
  virtual bool match(codegen_t &cdg) override = 0;

  bool hexapi init(const char *decl);
  virtual merror_t hexapi apply(codegen_t &cdg) override;
};
```

我们进一步来学习一下 `microcode_filter_t` 的定义

```
struct microcode_filter_t
{
  
  
  virtual bool match(codegen_t &cdg) = 0;

  
  
  
  
  
  virtual merror_t apply(codegen_t &cdg) = 0;
};
```

microcode_filter_t 的派生类可以用于非标准的 microcode 生成. 在 microcode 生成指令之前，所有注册的 filter 都将以如下形式调用

```
if ( filter->match(cdg) )
    code = filter->apply(cdg);
if ( code == MERR_OK )
    continue;
```

回到 `udc_filter_t`该类要求子类实现 match 函数，若 match 函数返回 true，则 match 匹配的指令将由 `udc_filter_t::apply` 替换。

install_microcode_filter 的定义如下

```
bool hexapi install_microcode_filter(microcode_filter_t *filter, bool install=true);
```

最后回到用户自定义的 `udc_exit_t` 该类继承于 `udc_filter_t` 因此实现了父类中的 match 方法  
当指令为 svc 且操作数是对应的系统调用号时返回 True，修改其指令的定义。

```
virtual bool match(codegen_t& cdg) override {
    return cdg.insn.itype == ARM_svc && cdg.insn.Op1.value == code;
}
```

初始化阶段调用 `init` 构造 call 指令对应目标函数的定义

```
char decl[MAXSTR];
qsnprintf(decl, sizeof(decl), "int __usercall %s@<R0>(int status@<R1>);", name);
bool ok = init(decl);
```

udc_filter_t 只需要指定函数描述以及自定义 match 函数就可实现将任意指令替换成一条 call 指令

### [](#python-实现版本-1 "python 实现版本")python 实现版本[](#python-实现版本-1)

```
class udc_exit_t(ida_hexrays.udc_filter_t):
    def __init__(self, code, name):
        ida_hexrays.udc_filter_t.__init__(self)
        if not self.init("int __usercall %s@<R0>(int status@<R1>);" % name):
            raise Exception("Couldn't initialize udc_exit_t instance")
        self.code = code
        self.installed = False

    def match(self, cdg):
        return cdg.insn.itype == ida_allins.ARM_svc and cdg.insn.Op1.value == self.code

    def install(self):
        ida_hexrays.install_microcode_filter(self, True);
        self.installed = True

    def uninstall(self):
        ida_hexrays.install_microcode_filter(self, False);
        self.installed = False

    def toggle_install(self):
        if self.installed:
            self.uninstall()
        else:
            self.install()
udc_exit = udc_exit_t(0x900001, "svc_exit")
udc_exit.toggle_install()
```

看起来 python 操作 microcode 似乎也挺方便的。

最后指令常量可以在 `allins.hpp` 文件中找到。

[](#vsd9 "vsd9")vsd9[](#vsd9)
-----------------------------

生成当前函数的 microcode 并将生成结果输出在 output window

核心代码如下

```
hexrays_failure_t hf;
func_t *pfn = get_func(get_screen_ea());
mba_t *mba = gen_microcode(pfn, &hf, NULL, DECOMP_WARNINGS);

vd_printer_t vp;
mba->print(vp);
delete mba;
```

gen_microcode 函数定义如下

```
mba_t *hexapi gen_microcode(
        const mba_ranges_t &mbr,
        hexrays_failure_t *hf,
        const mlist_t *retlist=NULL,
        int decomp_flags=0,
        mba_maturity_t reqmat=MMAT_GLBOPT3);
```

gen_microcode 的最后一个参数可以指定 mirocode 生成的阶段，microcode 是分阶段生成的。

microcode maturity 枚举值如下

```
enum mba_maturity_t
{
  MMAT_ZERO,         
  MMAT_GENERATED,    
  MMAT_PREOPTIMIZED, 
  MMAT_LOCOPT,       
                     
  MMAT_CALLS,        
  MMAT_GLBOPT1,      
  MMAT_GLBOPT2,      
  MMAT_GLBOPT3,      
  MMAT_LVARS,        
};
```

关于 vd_printer_t 的一些注释

[](#vds10 "vds10")vds10[](#vds10)
---------------------------------

该插件演示了如何添加一条 microcode 优化规则

```
*        call   !DbgRaiseAssertionFailure <fast:>.0
*      =>
*        call   !DbgRaiseAssertionFailure <fast:"char *" "assertion text">.0
```

安装注册优化规则处理类

```
install_optinsn_handler(&nt_assert_optimizer);
```

优化规则类

```
struct nt_assert_optimizer_t : public optinsn_t
{
    virtual int idaapi func(mblock_t *, minsn_t *ins, int ) override
    {
        if ( handle_nt_assert(ins) )
            return 1;
        return 0;
    }
    
    bool handle_nt_assert(minsn_t *ins) const
    {
        
        if ( !ins->is_helper("DbgRaiseAssertionFailure") )
            return false;

        
        mcallinfo_t &fi = *ins->d.f;
        if ( !fi.args.empty() )
            return false;

        
        qstring cmt;
        if ( !get_cmt(&cmt, ins->ea, false) )
            return false;

        
        if ( strneq(cmt.begin(), "NT_ASSERT(\\"", 11) )
            cmt.remove(0, 11);
        if ( cmt.length() > 2 && streq(cmt.begin()+cmt.length()-2, "\\")") )
            cmt.remove_last(2);

        
        mcallarg_t &fa = fi.args.push_back();
        fa.t    = mop_str;
        fa.cstr = cmt.extract();
        fa.type = tinfo_t::get_stock(STI_PCCHAR); 
        fa.size = fa.type.get_size();
        return true;
    }
};
```

代码有一点多，我们分开来看，该类继承于 `optinsn_t` 其定义如下

```
struct optinsn_t
{
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  virtual int idaapi func(mblock_t *blk, minsn_t *ins, int optflags) = 0;
};
```

使用 `install_optinsn_handler` 注册的回调都是 `optinsn_t` 的派生类。

`optinsn_t` 的派生类应该实现 `func` 方法， ida 在优化 microcode 的时候自动调用该方法。

`func`的 `ins` 参数传入的指令总是 top-level 指令，即最顶层的指令，callback 不允许删除该指令，但是可以替换成 nop （`mblock_t::make_nop`）。为了优化 sub-instructions , 使用 `minsn_visitor_t` 遍历子指令，子指令不能替换成 nop，可以用 mov x,x 代替。

返回值，返回改变指令的数量，若改变了 use/def 集合，必须将 block 标记为 dirty (见 mark_lists_dirty)

看注释觉得有点迷人，回到插件代码类: `nt_assert_optimizer_t`

`func` 函数中直接调用了 `handle_nt_assert` 函数，代码如下

```
bool handle_nt_assert(minsn_t *ins) const
{
    
    if ( !ins->is_helper("DbgRaiseAssertionFailure") )
        return false;

    
    mcallinfo_t &fi = *ins->d.f;
    if ( !fi.args.empty() )
        return false;

    
    qstring cmt;
    if ( !get_cmt(&cmt, ins->ea, false) )
        return false;

    
    if ( strneq(cmt.begin(), "NT_ASSERT(\\"", 11) )
        cmt.remove(0, 11);
    if ( cmt.length() > 2 && streq(cmt.begin()+cmt.length()-2, "\\")") )
        cmt.remove_last(2);

    
    mcallarg_t &fa = fi.args.push_back();
    fa.t    = mop_str;
    fa.cstr = cmt.extract();
    fa.type = tinfo_t::get_stock(STI_PCCHAR); 
    fa.size = fa.type.get_size();
    return true;
}
```

TODO 注释什么鬼东西….

[](#vds11 "vds11")vds11[](#vds11)
---------------------------------

演示添加一条跳转优化规则

```
*         goto L1     =>        goto L2
*         ...
*      L1:
*         goto L2
```

不同于 vds10， 这次继承的类是 `optblock_t` 而不是 vds10 的 `optinsn_t`

注册安装函数是

```
install_optblock_handler(&goto_optimizer);
```

`optblock_t` 的定义如下

```
struct optblock_t
{
  
  
  
  
  
  
  
  
  
  
  
  
  virtual int idaapi func(mblock_t *blk) = 0;
};
```

goto_optimizer_t 的代码如下

```
struct goto_optimizer_t : public optblock_t
{
    virtual int idaapi func(mblock_t *blk) override
    {
        if ( handle_goto_chain(blk) )
            return 1;
        return 0;
    }
    
    bool handle_goto_chain(mblock_t *blk) const
    {
        minsn_t *mgoto = blk->tail;
        if ( mgoto == NULL || mgoto->opcode != m_goto )
            return false;

        intvec_t visited;
        int t0 = mgoto->l.b;
        int i = t0;
        mba_t *mba = blk->mba;

        
        while ( true )
        {
            if ( !visited.add_unique(i) )
                return false; 
            mblock_t *b = mba->get_mblock(i);
            
            minsn_t *m2 = getf_reginsn(b->head);
            if ( m2 == NULL || m2->opcode != m_goto )
                break; 
            i = m2->l.b;
        }
        if ( i == t0 )
            return false; 

        
        mgoto->l.b = i; 

        
        blk->succset[0] = i;
        mba->get_mblock(i)->predset.add(blk->serial);
        mba->get_mblock(t0)->predset.del(blk->serial);

        
        
        
        mba->mark_chains_dirty();

        
        
        mba->verify(true);
        return true;
    }
};
```

逻辑比较简单，判断基本块结尾的指令是否为 goto 指令，并使用循环判断其目标基本块的第一条指令是否为 goto 指令。  
这段代码展示的 goto 指令的修改，不仅要修改 goto 指令，还需修改 successor/predecessor 。

[](#vds12 "vds12")vds12[](#vds12)
---------------------------------

这个插件可以显示对指定寄存器的所有直接引用。  
这个看似简单的插件实际上非常复杂，演示了如何使用 microcode 的 ud / du 链相关的 API

寄存器交叉引用整体思路:

1.  get_screen_ea & get_func 获取 EA 与当前函数
2.  get_flags & is_code 判断当前选中的是否为指令
3.  get_current_operand 获取用户选中的操作数寄存器
4.  gen_microcode, MMAT_PREOPTIMIZED 阶段，物理寄存器信息保留比较完整
5.  mba_t::build_graph(), MMAT_PREOPTIMIZED 阶段要求手动调用分析 CFG
6.  mba_t::analyze_calls(), MMAT_PREOPTIMIZED 阶段要求手动调用分析函数调用
7.  gco_info_t::append_to_list 把用户选中的寄存器转成 mlist_t （因为后面的 API 要求 mlist_t）
8.  mba_t::find_mop 找到 mlist_t 中寄存器对应的 mop_t 以及上下文信息 (ctx)
9.  获取 ud 与 du 链
10.  根据选中寄存器 use/def 情况调用 collect_xrefs

collect_xrefs 流程大概是:

1.  调用 collect_block_xrefs 收集当前基本块的 xref
2.  遍历 ud / du 分析其它基本块的 xref

将用户选中的寄存器有两种情况:

1.  引用 use
2.  定值 def

为了实现寄存器的直接交叉引用,  
若选中寄存器是引用， 则可以通过 ud 链获取所有可以到达该引用的定值；  
若选中寄存器是定值， 则可以通过 du 链获取该定值能到达的所有引用指令。

汇编语言中，有许多指令的第一个操作数，既是定值，也是引用，我们可以依次进行交叉引用分析。

为什么生成 MMAT_PREOPTIMIZED 阶段的 microcode 呢？ 因为这个阶段的寄存器信息保留比较完整！  
其后分析寄存器交叉引用需要用到 ud / du 链，而它们需要 microcode 生成 CFG 并完成函数调用的分析才能够使用，但是在 MMAT_PREOPTIMIZED 阶段并没有 CFG 与 Call 分析，所以要手动调用相关函数进行分析。

### [](#1-get-screen-ea-amp-get-func-获取-EA-与当前函数 "1. get_screen_ea & get_func 获取 EA 与当前函数")1. get_screen_ea & get_func 获取 EA 与当前函数[](#1-get-screen-ea-amp-get-func-获取-EA-与当前函数)

```
ea_t ea = get_screen_ea();
func_t* pfn = get_func(ea);
```

### [](#2-get-flags-amp-is-code-判断当前选中的是否为指令 "2. get_flags & is_code 判断当前选中的是否为指令")2. get_flags & is_code 判断当前选中的是否为指令[](#2-get-flags-amp-is-code-判断当前选中的是否为指令)

```
flags_t F = get_flags(ea);
if (!is_code(F)) {.....}
```

### [](#3-get-current-operand-获取用户选中的操作数寄存器 "3. get_current_operand 获取用户选中的操作数寄存器")3. get_current_operand 获取用户选中的操作数寄存器[](#3-get-current-operand-获取用户选中的操作数寄存器)

```
gco_info_t gco;
get_current_operand(&gco);
```

### [](#4-gen-microcode-MMAT-PREOPTIMIZED-阶段，物理寄存器信息保留比较完整 "4. gen_microcode, MMAT_PREOPTIMIZED 阶段，物理寄存器信息保留比较完整")4. gen_microcode, MMAT_PREOPTIMIZED 阶段，物理寄存器信息保留比较完整[](#4-gen-microcode-MMAT-PREOPTIMIZED-阶段，物理寄存器信息保留比较完整)

```
hexrays_failure_t hf;
mba_ranges_t mbr(pfn);
mba_t* mba = gen_microcode(mbr, &hf, NULL, DECOMP_WARNINGS, MMAT_PREOPTIMIZED);
```

### [](#5-mba-t-build-graph-MMAT-PREOPTIMIZED-阶段要求手动调用分析-CFG "5. mba_t::build_graph(), MMAT_PREOPTIMIZED 阶段要求手动调用分析 CFG")5. mba_t::build_graph(), MMAT_PREOPTIMIZED 阶段要求手动调用分析 CFG[](#5-mba-t-build-graph-MMAT-PREOPTIMIZED-阶段要求手动调用分析-CFG)

```
merror_t merr = mba->build_graph();
```

### [](#7-gco-info-t-append-to-list-把用户选中的寄存器转成-mlist-t-（因为后面的-API-要求-mlist-t） "7. gco_info_t::append_to_list 把用户选中的寄存器转成 mlist_t （因为后面的 API 要求 mlist_t）")7. gco_info_t::append_to_list 把用户选中的寄存器转成 mlist_t （因为后面的 API 要求 mlist_t）[](#7-gco-info-t-append-to-list-把用户选中的寄存器转成-mlist-t-（因为后面的-API-要求-mlist-t）)

这一步是非常迷惑的操作, 先看代码

```
mlist_t list;
if (!gco.append_to_list(&list, mba))
{
    warning("Failed to represent %s as microcode list", gco.name.c_str());
    delete mba;
    return false;
}
```

注释写得很清楚，就是说后面查找 mop 的时候用的是 mlist_t。另外 mlist_t::print 函数，可以将 mlist 中的寄存器 / 内存信息 dump 成文本方便调试。

gco 的类型是 `gco_info_t` 是一个与用户界面 UI 相关的类, 用来表示用户选中的寄存器, 用户选中的寄存器经过三步转换才能得到详细准确的寄存器与对应的指令等信息。  
从数据结构来看大概是这样的: gco_info_t -> mlist_t -> mop_t, op_parent_info_t

### [](#8-mba-t-find-mop-找到-mlist-t-中寄存器对应的-mop-t-以及上下文信息-ctx "8. mba_t::find_mop 找到 mlist_t 中寄存器对应的 mop_t 以及上下文信息(ctx)")8. mba_t::find_mop 找到 mlist_t 中寄存器对应的 mop_t 以及上下文信息 (ctx)[](#8-mba-t-find-mop-找到-mlist-t-中寄存器对应的-mop-t-以及上下文信息-ctx)

```
op_parent_info_t ctx;
mop_t* mop = mba->find_mop(&ctx, ea, gco.is_def(), list);
```

`op_parent_info_t` 包含了操作数对应的上下文信息，例如对应的 Top-level 指令。

### [](#9-获取-ud-与-du-链 "9. 获取 ud 与 du 链")9. 获取 ud 与 du 链[](#9-获取-ud-与-du-链)

```
{
    
    
    mbl_graph_t* graph = mba->get_graph();
    chain_keeper_t ud = graph->get_ud(GC_REGS_AND_STKVARS);
    chain_keeper_t du = graph->get_du(GC_REGS_AND_STKVARS);
}
```

`mbl_graph_t *` 会在基本块退出时自动释放。

### [](#10-根据选中寄存器-use-def-情况调用-collect-xrefs "10. 根据选中寄存器 use/def 情况调用 collect_xrefs")10. 根据选中寄存器 use/def 情况调用 collect_xrefs[](#10-根据选中寄存器-use-def-情况调用-collect-xrefs)

```
if (gco.is_use())
{
    
    collect_xrefs(&xrefs, ctx, mop, list, ud, false);
    ndefs = xrefs.size();
    
    xrefs.add_unique(ea);
}

if (gco.is_def())
{
    
    if (xrefs.add_unique(ea))
        ndefs = xrefs.size();
    
    collect_xrefs(&xrefs, ctx, mop, list, du, true);
}
```

如果选中的寄存器是 use 则使用 ud 去调用 `collect_xrefs` 且最后一个参数 find_uses = true  
如果选中的寄存器是 def 则使用 du 去调用 `collect_xrefs` 且最后一个参数 find_uses = false  
find_uses 决定的是基本块内搜索的方向。

### [](#collect-xrefs-分析 "collect_xrefs 分析")collect_xrefs 分析[](#collect-xrefs-分析)

collect_xrefs 主要的作用是调用 `collect_block_xrefs` 函数收集基本块内的交叉引用信息, 先收集当前基本块的交叉引用，再通过 du 链查找其它 use 或 def 基本块，并收集其信息。  
ud/du 链中只存储了 use 或 def 的基本块信息，没有具体到某一条指令。

```
static void collect_xrefs(
    eavec_t* out,
    const op_parent_info_t& ctx,
    const mop_t* mop,
    mlist_t list,
    const graph_chains_t& du,
    bool find_uses)
{
    
    minsn_t* start = find_uses ? ctx.topins->next : ctx.topins->prev;
    collect_block_xrefs(out, &list, ctx.blk, start, find_uses);

    
    int serial = ctx.blk->serial;                 
    const block_chains_t& bc = du[serial];        
    const chain_t* ch = bc.get_chain(*mop);       
    if (ch == NULL)
        return; 
    for (int i = 0; i < ch->size(); i++)
    {
        int bn = ch->at(i);
        mblock_t* b = ctx.mba->get_mblock(bn);      
        minsn_t* ins = find_uses ? b->head : b->tail;
        mlist_t tmp = list; 
        collect_block_xrefs(out, &tmp, b, ins, find_uses);
    }
}
```

从这段代码我们可以窥探 IDA 的 du/ud 链条的设计， IDA 用 `graph_chains_t` 数据类型来表示整个函数的 du/ud 链，它其实是一个 vector 存储的是 `block_chains_t` 其下标与基本块的 serial 相对应。  
`block_chains_t` 是一个 map 结构，将基本块内的所有 mop 操作数与对应的 ud/du 链关联。  
`chain_keeper_t` 其实是一个三级结构，第一级由基本块下标索引，第二级由操作数索引，第三级才是操作数对应的 du 链。

总结一下，我们如果想获取某个基本块中某个操作数的 du 链条:

```
mbl_graph_t* graph = mba->get_graph();
chain_keeper_t du = graph->get_du(GC_REGS_AND_STKVARS); 
const block_chains_t& bc = du[serial]; 
const chain_t* ch = bc.get_chain(*mop);
```

操作数对应的 du 链条中存储的是 use 该操作数的其它基本块的下标， 而不是具体的指令表示。`collect_block_xrefs` 函数的功能就是收集基本块内的 use/def 信息

### [](#collect-block-xrefs-分析 "collect_block_xrefs 分析")collect_block_xrefs 分析[](#collect-block-xrefs-分析)

```
static void collect_block_xrefs(
    eavec_t* out,
    mlist_t* list,
    const mblock_t* blk,
    const minsn_t* ins,
    bool find_uses)
{
    for (const minsn_t* p = ins;
        p != NULL && !list->empty();
        p = find_uses ? p->next : p->prev)
    {
        mlist_t use = blk->build_use_list(*p, MUST_ACCESS); 
        mlist_t def = blk->build_def_list(*p, MUST_ACCESS); 
        mlist_t& plst = find_uses ? use : def;
        if (list->has_common(plst))
            out->add_unique(p->ea); 
        list->sub(def);
    }
}
```

首先要讲的就是 `find_uses` 的作用，字面意思很好理解，就是是否搜索引用还是搜索定值，其实区别主要还是在于搜索方向的不同。

搜索 use 的方向是从第一条指令开始，向后搜索，遇到对这条指令的 def 则退出搜索。  
搜索 def 的方向是从最后一条指令开始，向前搜索，遇到对这条指令的 def 则退出搜索。

根据数据流分析理论，def 可以 kill 掉变量的活跃性。因此，我们可以说，搜索 use 的时候正向遍历，遇到 def 的时候退出。 搜索 def 的时候反向遍历，遇到 def 的时候退出。

`mlist_t` 设计真的非常好，寄存器用的是 BitVector 存储，支持快速位运算，做数据流算法的时候非常高效。

我们主要来看一下循环，循环根据 find_uses 方向遍历指令（向前 / 向后），并且 list 不为空才会继续遍历。  
循环内部先获取了当前指令的 use/def，并通过 has_common 判断是否与当前搜索的操作数是否有交集，如果有交集则直接将指令的地址加入到结果列表。

从 list 中删除 def 的寄存器。

```
list->sub(def)
```

list 中其实就只有待搜索的一个寄存器，若该寄存器被 def 就会从 list 删除，循环也会因此退出。

### [](#其它交叉引用-UI-部分 "其它交叉引用 UI 部分")其它交叉引用 UI 部分[](#其它交叉引用-UI-部分)

UI 部分的代码分析我就不写了，比较简单，可以自行查看。

[](#vds13 "vds13")vds13[](#vds13)
---------------------------------

这个插件生成选中的代码并输出 microcode

核心代码非常简单

```
hexrays_failure_t hf;
mba_ranges_t mbr;
mbr.ranges.push_back(range_t(ea1, ea2));
mba_t *mba = gen_microcode(mbr, &hf, NULL, DECOMP_WARNINGS);
if ( mba == NULL )
{
    warning("%a: %s", hf.errea, hf.desc().c_str());
    return true;
}

msg("Successfully generated microcode for %a..%a\\n", ea1, ea2);
vd_printer_t vp;
mba->print(vp);


delete mba;
```

主要演示了 `mba_ranges_t` 的构造。

[](#vds14 "vds14")vds14[](#vds14)
---------------------------------

略

[](#vds15 "vds15")vds15[](#vds15)
---------------------------------

演示使用 `get_valranges()` 获取寄存器的值域.

这个插件也受到诸多的限制

```
*      Unfortunately this plugin is of limited use because:
*        - simple cases where a single value is assigned to a register
*          are automatically handled by the decompiler and the register
*          is replaced by the value
*        - too complex cases where the register gets its value from untrackable
*          sources, it fails
*        - only value ranges at the basic block start are shown
```

主要核心代码

```
bool idaapi plugin_ctx_t::run(size_t)
{
    ea_t ea = get_screen_ea();
    func_t* pfn = get_func(ea);
    if (pfn == NULL)
    {
        msg("Please position the cursor within a function\\n");
        return true;
    }

    flags_t F = get_flags(ea);
    if (!is_code(F))
    {
        msg("Please position the cursor on an instruction\\n\\n");
        return true;
    }

    gco_info_t gco;
    if (!get_current_operand(&gco))
    {
        msg("Could not find a register or stkvar in the current operand\\n");
        return true;
    }

    
    hexrays_failure_t hf;
    mba_ranges_t mbr(pfn);
    mba_t* mba = gen_microcode(mbr, &hf, NULL, DECOMP_WARNINGS);
    if (mba == NULL)
    {
        msg("%a: %s\\n", hf.errea, hf.desc().c_str());
        return true;
    }

    
    mlist_t list;
    if (!gco.append_to_list(&list, mba))
    {
        msg("Failed to represent %s as microcode list\\n", gco.name.c_str());
        delete mba;
        return false;
    }

    
    const mblock_t* b;
    const minsn_t* ins;
    if (!find_insn_with_list(&b, &ins, mba, ea, list, gco.is_def()))
    {
        msg("Could not find %s after %a in the microcode, sorry\\n"
            "Probably it has been optimized away\\n",
            gco.name.c_str(), ea);
        delete mba;
        return false;
    }

    valrng_t vr;
    int vrflags = VR_AT_START | VR_EXACT;
    if (b->get_valranges(&vr, gco.cvt_to_ivl(), ins, vrflags))
    {
        qstring vrstr;
        vr.print(&vrstr);
        msg("Value ranges of %s at %a: %s\\n",
            gco.name.c_str(),
            ins->ea,
            vrstr.c_str());
    }
    else
    {
        msg("Cannot find value ranges of %s\\n", gco.name.c_str());
    }

    
    delete mba;
    return true;
}
```

`find_insn_with_list` 函数的功能是寻找指定含有选中 opcode 最接近 EA 的顶级指令，待会儿我们来分析这个函数。  
我们来看看 `get_valranges` 的调用  
函数原型: `bool get_valranges(valrng_t valrng, vivl_t vivl, minsn_t minsn, int VRFLAGS)`  
第一个参数 `valrng`: 输出结果, 用于表示值域  
第二个参数 `vivl` : 输入, 用于表示搜索的对象 `gco.cvt_to_ivl()` 转换  
第三个参数 `minsn` : 输入, 搜索的目标指令  
第四个参数 `VRFLAGS`: 标志信息, 宏以 `VR_` 开头

`VR_` 开头的宏主要有三个

```
#define VR_AT_START 0x0000    
                              
#define VR_AT_END   0x0001    
                              
                              
#define VR_EXACT    0x0002
```

具体调用代码如下:

```
valrng_t vr;
int vrflags = VR_AT_START | VR_EXACT;
if (b->get_valranges(&vr, gco.cvt_to_ivl(), ins, vrflags))
{
    qstring vrstr;
    vr.print(&vrstr);
    msg("Value ranges of %s at %a: %s\\n",
        gco.name.c_str(),
        ins->ea,
        vrstr.c_str());
}
```

### [](#find-insn-with-list-分析 "find_insn_with_list 分析")find_insn_with_list 分析[](#find-insn-with-list-分析)

```
static bool find_insn_with_list(
    const mblock_t** blk,
    const minsn_t** ins,
    mba_t* mba,
    ea_t _ea,
    const mlist_t& _list,
    bool _is_dest)
{
    struct ida_local top_visitor_t : public minsn_visitor_t
    {
        const mblock_t* b = nullptr;
        const minsn_t* ins = nullptr;
        ea_t ea;
        const mlist_t& list;
        bool is_dest;
        top_visitor_t(ea_t e, const mlist_t& l, bool d) : ea(e), list(l), is_dest(d) {}
        int idaapi visit_minsn(void) override
        {
            if (topins->ea == ea) 
            {
                
                b = blk; 
                ins = topins;
                return true;
            }
            if (blk->start <= ea && topins->ea > ea) 
            {
                mlist_t defuse = is_dest
                    ? blk->build_def_list(*topins, MUST_ACCESS)
                    : blk->build_use_list(*topins, MUST_ACCESS);
                if (defuse.has_common(list)
                    && (ins == nullptr || topins->ea < ins->ea))
                {
                    
                    b = blk;
                    ins = topins;
                }
            }
            return false;
        }
    };
    top_visitor_t tv(_ea, _list, _is_dest);

    mba->for_all_topinsns(tv); 
    if (tv.ins != nullptr)
    {
        *blk = tv.b;
        *ins = tv.ins;
        return true;
    }
    return false;
}
```

这段代码很简单就不过多解释了，唯一需要注意的是 `minsn_visitor_t` 继承于 `op_parent_info_t`  
所以在实现的观察者类中可以直接用 `op_parent_info_t` 中的 `blk`， 而非参数中的 `blk`.

[](#vds16 "vds16")vds16[](#vds16)
---------------------------------

添加一个常规的指令优化规则插件

```
*        mov #N, var.4                  mov #N, var.4
*        xor var@1.1, #M, var@1.1    => mov #NM, var@1.1
*                                         where NM == (N>>8)^M
*
*      We need this rule because the decompiler cannot propagate the second
*      byte of VAR into the xor instruction.
*
*      The XOR opcode can be replaced by any other, we do not rely on it.
*      Also operand sizes can vary.
```

说实话，没有看懂这个优化具体是干什么的, 部分分析看注释吧。  
优化的主要代码如下: func

```
struct glbprop_t : public optinsn_t
{
    virtual int idaapi func(mblock_t* blk, minsn_t* ins, int ) override
    {
        if (ins->r.t != mop_n) 
            return 0; 

        if (ins->r.size > 2) 
            return 0; 

          
        mlist_t use = blk->build_use_list(*ins, MAY_ACCESS);

        
        const minsn_t* di = find_prev_def(blk, use, ins); 
        if (di == NULL)
            return 0; 

        if (di->opcode != m_mov || di->l.t != mop_n)
            return 0; 

          
        mop_t v1 = ins->l;
        const mop_t& v2 = di->d;
        if (v1.t != v2.t)
            return 0; 

          
          
          
          
        if (v1.size >= v2.size)
            return 0;

        
        int off = 0;
        while (!v1.equal_mops(v2, EQ_IGNSIZE)) 
        {
            if (++off >= v2.size)
                return 0;
            if (!v1.shift_mop(-1))
                return 0;
        }

        
        
        uint64 N = di->l.value(false);
        N >>= (off * 8);

        
        ins->l.make_number(N, ins->l.size, di->l.nnn->ea, di->l.nnn->opnum);

        
        
        ins->optimize_solo();

        return 1; 
    }
};
```

向前（低地址）寻找引用的定值指令

```
static const minsn_t* find_prev_def(
    const mblock_t* blk,
    const mlist_t& lst,
    const minsn_t* ins)
{
    const minsn_t* p = ins;
    while ((p = p->prev) != NULL)
    {
        mlist_t def = blk->build_def_list(*p, MAY_ACCESS | FULL_XDSU);
        if (def.has_common(lst))
            break;
    }
    return p;
}
```

[](#vds18 "vds18")vds18[](#vds18)
---------------------------------

这个插件展示了如何给指定位置的指定寄存器指定一个值，这个功能在一些混淆场景中非常有用。

插件入口函数代码如下

```
bool idaapi plugin_ctx_t::run(size_t)
{
    
    
    
    static const char form[] =
        "Specify known register value\\n"
        "<~A~ddress :$::16::>\\n"
        "<~R~egister:q::16::>\\n"
        "<~V~alue   :L::16::>\\n"
        "\\n";
    static qstring regname;
    static fixed_regval_info_t fri;
    CASSERT(sizeof(fri.ea) == sizeof(ea_t));
    CASSERT(sizeof(fri.value) == sizeof(uint64));
    while (ask_form(form, &fri.ea, ®name, &fri.value))
    {
        reg_info_t ri;
        if (!parse_reg_name(&ri, regname.c_str()))
        {
            warning("Sorry, bad register name: %s", regname.c_str());
            continue;
        }
        fri.nbytes = ri.size;
        fri.reg = reg2mreg(ri.reg);
        if (fri.reg == mr_none)
        {
            warning("Failed to convert to microregister: %s", regname.c_str());
            continue; 
        }
        bool found = false;
        for (auto& rv : user_regvals)
        {
            if (rv.ea == fri.ea && rv.reg == fri.reg)
            {
                rv.nbytes = fri.nbytes;
                rv.value = fri.value;
                found = true;
                break;
            }
        }
        if (!found)
            user_regvals.push_back(fri);
        static const char fmt[] = "Register %s at %a is considered to be equal to 0x%" FMT_64 "X\\n";
        info(fmt, regname.c_str(), fri.ea, fri.value);
        msg(fmt, regname.c_str(), fri.ea, fri.value);
        return true;
    }
    return false;
}
```

调用 `ask_form` 询问用户固定的寄存器信息，并将该值存入 `user_regvals`.  
`user_regvals` 的值通过 idb 持久化存储。

`hr_callback` 处理 `hxe_microcode` 回调，该回调在 microcode 生成完成的时候调用。

```
ssize_t idaapi plugin_ctx_t::hr_callback(
    void* ud,
    hexrays_event_t event,
    va_list va)
{
    plugin_ctx_t& ctx = *(plugin_ctx_t*)ud;
    if (event == hxe_microcode)
    {
        mba_t* mba = va_arg(va, mba_t*);
        ctx.insert_assertions(mba);
    }
    return 0;
}
```

主要逻辑还是要看 `insert_assertions`

```
void plugin_ctx_t::insert_assertions(mba_t* mba) const
{
    func_t* pfn = mba->get_curfunc();
    if (pfn == NULL)
        return; 

      
    fixed_regvals_t regvals;
    for (const auto& rv : user_regvals) 
    {
        if (func_contains(pfn, rv.ea)) 
            regvals.push_back(rv);
    }
    if (regvals.empty())
        return; 

    struct ida_local assertion_inserter_t : public minsn_visitor_t
    {
        fixed_regvals_t& regvals;
        virtual int idaapi visit_minsn(void) override
        {
            for (size_t i = 0; i < regvals.size(); i++)
            {
                fixed_regval_info_t& fri = regvals[i];
                if (curins->ea == fri.ea) 
                {
                    
                    minsn_t* m = create_mov(fri);
                    
                    blk->insert_into_block(m, curins->prev);
                    
                    regvals.erase(regvals.begin() + i);
                    --i;
                }
            }
            return regvals.empty(); 
        }
        assertion_inserter_t(fixed_regvals_t& fr) : regvals(fr) {}
    };
    assertion_inserter_t ai(regvals);

    
    
    
    
    
    
    mba->for_all_topinsns(ai);

    
    mba->dump();

    
    
    mba->verify(true);
}
```

上面这段代码中最核心的代码就是插入指令的代码

```
minsn_t* m = create_mov(fri);

blk->insert_into_block(m, curins->prev);

regvals.erase(regvals.begin() + i);
--i;
```

`create_mov` 的代码如下

```
static minsn_t* create_mov(const fixed_regval_info_t& fri)
{
    minsn_t* m = new minsn_t(fri.ea);
    m->opcode = m_mov;
    m->l.make_number(fri.value, fri.nbytes, fri.ea);
    m->d.make_reg(fri.reg, fri.nbytes);
    
    
    
    m->iprops |= IPROP_ASSERT;
    
    msg("Created insn: %s\\n", m->dstr());
    return m;
}
```

插入的指令被修饰为 `IPROP_ASSERT` ，即断言，不会出现在 Ctree 中，因此不会影响反编译的结果。

效果如下:  
在 0x12F8 处设置 rax = 8

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled9.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled9.png "Untitled")

判断条件成立

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled10.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled10.png "Untitled")

microcode 代码如下

[![](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled11.png)

Untitled

](https://panda0s.top/2021/08/19/Microcode-%E6%8F%92%E4%BB%B6%E5%AD%A6%E4%B9%A0/Untitled11.png "Untitled")

可以看出添加了一条指令（注意该指令为 assertion 不会在 ctree 中呈现）

[](#vds19 "vds19")vds19[](#vds19)
---------------------------------

这个插件添加了一条优化规则， 规则如下

```
x | ~x => -1
```

这个优化器的难点是实现 `x | ~x` 表达式的识别。

注册指令优化器代码

```
install_optinsn_handler(&tiny_optimizer);
```

IDA 会依次为函数的所有 top-level 指令调用优化器类的 func 函数

```
struct sample_optimizer_t : public optinsn_t
{
  virtual int idaapi func(
        mblock_t *blk,
        minsn_t *ins,
        int ) override
  {
    subinsn_optimizer_t so;
    ins->for_all_insns(so);
    if ( so.cnt != 0 && blk != nullptr ) 
      blk->mba->verify(true);            
    return so.cnt;                       
  }
};
```

在 top 指令级并不能完成什么有实际性的优化工作，为了识别子表达式，继承 `minsn_visitor_t` 类实现了一个子指令遍历类，实现了具体的优化任务。

```
struct subinsn_optimizer_t : public minsn_visitor_t
{
  int cnt = 0;
  int idaapi visit_minsn() override 
  {
    
    
    if ( curins->opcode == m_or
      && curins->r.is_insn(m_bnot)
      && curins->l == curins->r.d->l )
    {
      if ( !curins->l.has_side_effects() ) 
      {
        
        curins->opcode = m_mov;
        curins->l.make_number(-1, curins->r.size);
        curins->r.erase();
        cnt = cnt + 1; 
      }
    }
    return 0; 
  }
};
```

逻辑比较简单，遍历指令过程中若遇到 `x | ~x` 表达式，则将其替换成 `mov -1` 指令。

通过代码可以看出，这种优化只能处理 not 是 or 的子指令的情况，IDA 设计的指令嵌套确实很方便这种优化，可以不要考虑 删除掉 not 或 or 对于其它指令的影响。

[](#vds20 "vds20")vds20[](#vds20)
---------------------------------

好像与 mircocode 无关就不写了。