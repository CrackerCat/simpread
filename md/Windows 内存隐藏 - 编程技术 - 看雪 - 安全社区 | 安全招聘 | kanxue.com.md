> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282107.htm)

> Windows 内存隐藏

Windows 内存隐藏

16 小时前 804

### Windows 内存隐藏

 [![](http://passport.kanxue.com/upload/avatar/317/925317.png?1647249241)](user-home-925317.htm) [Qfrost](user-home-925317.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif) 16 小时前  804

整理笔记发现之前整理的一些内存隐藏方式，考虑到现在用这些方法的外挂已经很少了（全是 VT、UEFI 起手），所以小发一手可以一起学习一下

**VAD 节点合并**
------------

VAD（Virtual Address Descriptor）虚拟地址描述符，是用来管理 Windows 进程地址空间的，其可以描述一段连续的地址范围。Windows 用一颗平衡二叉树（AVL 树）管理 VAD 对象，每个进程拥有一颗 VAD 树，其树的根节点指针（VadRoot）保存在进程的 EPROCESS 中，而进程的每一个内存块（包含 VirtualAlloc 申请的私有的和 Mapping 共享的）都对应一个 VAD 树结点（_MMVAD）。可以通过对进程的 VAD 树结构进行调整以达到隐藏某块内存的目的。

可以将待隐藏的 VAD 节点融合到前一个节点上，其属性将显示为前一个节点的属性。常规手段是向前找一个 NoAccess 属性的父节点，并将待隐藏节点及其到目标父节点之间的所有节点均融合进去，这样就可以将待隐藏内存块显示为 No_Access 属性。

```
nStatus = BBFindVAD(pProcess, parent_address, &pVadShort_parent);
nStatus |= BBFindVAD(pProcess, address, &pVadShort_target);
if (!NT_SUCCESS(nStatus))
    return nStatus;
 
 
 
if (pVadShort_parent && pVadShort_target && pVadShort_parent != pVadShort_target)
{
    KIRQL Irql;
    HIDE_VAD vad = { 0 };
    vad.pid = pid;
    vad.address = address;
    vad.pVadShort = pVadShort_parent;
    vad.StartingVpn = pVadShort_parent->StartingVpn;
 
    {
        combined_vad_list.emplace_back(vad);
        pVadShort_parent->StartingVpn = pVadShort_target->EndingVpn;
    }
 
    nStatus = STATUS_SUCCESS;
}

```

![](https://bbs.kanxue.com/upload/attach/202406/925317_A4JFYM6J2FTV58E.webp)

这种方式需要在进程退出时，将融合的节点还原（不然会出现内存管理错误而 BSOD）

**VAD 节点挂靠不存在的地址范围**
--------------------

VAD 节点中，最关键的两个字段是 StartingVpn 和 EndingVpn，分别指示该 VAD 节点所指示的地址空间范围。在 VAD 节点合并中，是将父节点的 StartingVpn 修改成子节点（待隐藏节点）的 EndingVpn 以实现合并，而挂靠不存在地址范围的意思是将待隐藏节点的 VAD.StartingVpn 和 VAD.EndingVpn 修改成一个不存在的虚拟地址，比如修改成 0。这样的话，本来这个 VAD 节点是管理我们待隐藏的那块虚拟地址内存的，修改后就变成管理一个不存在的地址了，而我们那块待隐藏的节点因为没有 VAD 结构指向它了，也就无法被常规的 API 内存枚举遍历到了，从而实现了内存隐藏。这种方式有点类似于 VAD Unlink，都是目标内存区域不再有 VAD 节点管理，但是又不太一样，因为我们并没有对 VAD 树上的节点做摘除，只是修改了其字段值导致其管理的目标区域被改变。

```
// 将内存页对应的VAD节点挂靠到0内存上
// 在18362上设置会崩溃
if (KernelOffsets::BuildNumber != 18362 && KernelOffsets::BuildNumber != 18363) {
 
    KIRQL Irql;
    HIDE_VAD vad = { 0 };
    vad.pid = pid;
    vad.address = address;
    vad.pVadShort = pVadShort;
    vad.StartingVpn = pVadShort->StartingVpn;
    vad.EndingVpn = pVadShort->EndingVpn;
    vad.Protection = pVadShort->u.VadFlags.Protection;
 
    KeAcquireSpinLock(&unlink_vad_list_spin_lock, &Irql);
    {
        unlink_vad_list.emplace_back(vad);
        pVadShort->StartingVpn = 0;
        pVadShort->EndingVpn = 0;
        pVadShort->u.VadFlags.Protection = MM_ZERO_ACCESS;
        kprintf("[QSafe] : Unlink VAD : 0x%llx  Success!\n", address);
    }
    KeReleaseSpinLock(&unlink_vad_list_spin_lock, Irql);
 
}

```

![](https://bbs.kanxue.com/upload/attach/202406/925317_Y8Z8VDERXFP3CFE.webp)

同样这种方式需要在进程退出时还原被修改的 VAD 节点，否则会 BSOD

**PTE.USER（高位注入）**
------------------

之前非常流行的注入方式。其核心原理是：驱动申请内核内存，设置 PTE/PDE 对某进程（游戏进程和外挂进程）可见（supervisor 位），然后设置 X 位使其具有执行权限，再过掉各版本 Windows 对它的亿点点检测就好了。这里只介绍方法原理，具体的代码实现 Github 上有很多开源的仓库可以参考。

### **实现步骤**

#### Alloc Kernel MDL

高位注入，shellcode/Dll 是跑在内核空间的，所以需要先用驱动申请一块内核的内存然后 MmMapLockedPagesSpecifyCache 指定 PreviousMode 为 KernelMode 映射到内核空间。

关于内存申请的方法有两种，即申请分页内存和非分页内存，申请这两种内存也有各自的优缺点

1.  申请非分页内存，过大可能因为内存不足而申请失败；隐匿性较差（扫系统具有 us 位的非分页内存就可以把这类内存抓出来）
    
2.  申请分页内存，在换页时，us 位和 X 位会被操作系统恢复。即当换页发生时，操作系统会根据 VAD 的属性为内存页还原正常的属性位（可以通过锁页和 Hook 换页相关的 API 解决
    

```
MDL_INFORMATION memory;
 
PHYSICAL_ADDRESS lower, higher;
lower.QuadPart = 0;
higher.QuadPart = 0xffff'ffff'ffff'ffffULL;
 
const auto pages = (size / PAGE_SIZE) + 1;
 
const auto mdl = MmAllocatePagesForMdl(lower, higher, lower, pages * (uintptr_t)0x1000);
 
if (!mdl)
    return { 0, 0 };
 
const auto mapping_start_address = MmMapLockedPagesSpecifyCache(mdl, KernelMode, MmCached, NULL, FALSE, NormalPagePriority);
 
if (!mapping_start_address)
    return { 0, 0 };
 
memory.mdl = mdl;
memory.va = reinterpret_cast (mapping_start_address);
 
return memory; 
```

#### Attach Process & Set Supervisor

申请到内核内存后，需要将该内存页挂靠到用户态进程上，为了使用户态进程对其可见，需要对内存页的 PTE/PDE 设置 supervisor(user) 位

```
for (uintptr_t address = kernel_address; address <= kernel_address + size; address += 0x1000)
{
    const PAGE_INFORMATION page_information = get_page_information((void*)address, cr3);
 
    page_information.PDE->Supervisor = 1;
    page_information.PDPTE->Supervisor = 1;
    page_information.PML4E->Supervisor = 1;
 
    if (!page_information.PDE || (page_information.PTE && !page_information.PTE->Present))
    {
 
    }
    else
    {
        page_information.PTE->Supervisor = 1;
    }
 
}

```

#### ByPass NX

这样申请到的内存不具有可执行权限，需要为其设置可执行权限。可以在申请内存的时候调用 _MmProtectMdlSystemAddress(mdl, PAGE_EXECUTE_READWRITE);_ , 但似乎这个 API 经常不成功，比较稳妥的是直接为 PTE/PDE 的 ExecuteDisable 位置 0

```
page_information.PDE->ExecuteDisable = 0;
page_information.PTE->ExecuteDisable = 0;

```

并且保证 CR3 的 63 位为 0

```
CR3 cr3{ };
cr3.Flags = __readcr3();
cr3.Flags &= 0x7FFFFFFFFFFFFFFF;
__writecr3(cr3.Flags);

```

#### Flush TLB

TLB 是一块高速缓存，其缓存虚拟地址和其映射的物理地址。硬件存在 TLB 后，虚拟地址到物理地址的转换过程发生了变化。虚拟地址首先发往 TLB 确认是否命中 cache，如果 cache hit 直接可以得到物理地址。否则，一级一级查找页表获取物理地址。并将虚拟地址和物理地址的映射关系缓存到 TLB 中。 而我们对页表结构进行了更改，需要刷新 TLB 避免出现和缓存的不同步问题，刷新 TLB 实际就是使 TLB 无效化，下一次再进行虚拟地址到物理地址转化时需要从页表一级一级向下找并将结果重新填回缓存。

有一条特权级指令为我们实现了这个功能，即 invlpg。再每次修改 PDE/PTE 属性后，调用一下该指令刷新 TLB 即可

```
__invlpg((void*)address);

```

#### Lock MDL

如前面所说，申请分页内存，在换页时，us 位和 X 位会被操作系统恢复。即当换页发生时，操作系统会根据 VAD 的属性为内存页还原正常的属性位。

需要对内存页进行锁页，并在进程结束前恢复。

#### Disable SMAP

通过前面的步骤，可以在 Windows10 1809（17163）及之前的版本注入成功，但是从 Windows10 1903（18362）开始多了很多保护

SMAP(Supervisor Mode Access Prevention，管理模式访问保护) : 禁止内核 CPU 访问用户空间的数据  
SMEP(Supervisor Mode Execution Prevention，管理模式执行保护) : 禁止内核 CPU 执行用户空间的代码

设计这两种保护是为了防止恶意程序提权到 0 环后去执行布置在三环的恶意 Shellcode。而实际上，在高位注入中，是从三环跳过来运行 0 环的代码，所以实际上只需要关闭 SMAP 而不需要关闭 SMEP，因为我们没有在内核执行 Ring3 的代码，但是我们需要在内核访问 Ring3 的代码。

关 SMAP 有两种方法，一种直接暴力改 CR4 的 SMAP_ENABLE_FLAG 位

```
CR4 cr4{};
cr4.Flags = __readcr4();
cr4.Flags &= ~CR4_SMAP_ENABLE_FLAG;
__writecr4(cr4.Flags);

```

还有一种标准关闭方法，即先用 stac 指令标准关闭，后续再通过 clac 标准打开

![](https://bbs.kanxue.com/upload/attach/202406/925317_RFU3DJUKSDZ4F6S.webp)

#### ByPass MiCheckProcessShadow

在 Windows 10 1903（18362）+ 上运行注入，会马上出现这样的蓝屏

![](https://bbs.kanxue.com/upload/attach/202406/925317_28KZ48X6JTRY95M.webp)

BSOD 提示在 MiCheckProcessShadow 函数中发生了 _KeBugCheckEx 0x1A, 0x3604_ 的错误，分析其报错检测，该函数会检测 PXE，即，当前确实一定是在 Kernel CR3 而非用户 CR3，目的是防止恶意代码对内核内存自己加 us 位。

对于这种检测有两种处理方式

1.  Hook MiCheckProcessShadow ：其实只需要 _eb MiCheckProcessShadow C3_ 将其 return 就好了
    
2.  管理员身份运行程序 ： 在 Windows10 上以管理员身份运行程序，API 便不会进行校验
    

#### ByPass KVAS

对 Windows KVAS（Kernel Virtual Address Shadow，内核虚拟地址影子）机制的分析，参考资料  
[Windows KVAS](http://www.qfrost.com/posts/windowskernel/kvas/)

#### Disable VBS

基于虚拟化的安全性 (VBS) 使用 Windows 管理程序将主内存的一部分与操作系统的其余部分虚拟隔离。 Windows 使用这个隔离的、安全的内存区域来存储重要的安全解决方案，例如登录凭据和负责 Windows 安全的代码等。

这一安全机制的本意是防止内核级的恶意程序， 因为内核级的恶意程序可以访问操作系统的所有资源而使所有程序和操作系统本身均无感。所以这一安全机制通过 Hypervisor 实现更高的一层权限来阻止这一类操作。

高位注入无法在开启 _基于虚拟化的安全性（VBS）_ 的计算机中使用。（在目前版本的 Windows 下，这个选项还是默认关闭的

![](https://bbs.kanxue.com/upload/attach/202406/925317_XS8HFBU7Z9PPGM4.webp)

#### Handle PF

高位注入后，在使用很多 API 时会出现 PF。原因是很多 API 内会做一些检测，比如对参数地址的检测（是否在 MmUserProbeAddress 范围内），所以在 release 后需要进行单元测试，对可能的 PF 进行处理修复。

PS：还有个办法就是完全不用 API，所有操作均由自己直接与内核通信实现

### **局限性**

1.  无法在 32 位机器上运行
2.  被注入进程必须是特权级进程（以管理员身份运行的）
3.  不支持大页
4.  被注入 Dll 不可直接使用 API：因为注入在高位，使外部依赖库的加载和导入表重定位的修正变的非常困难。难以像正常注入一样对外部依赖库调用 LoadLibrary 并修正 IAT。可以使用 PEB->ldr 链并 getProcAddrByHash 直接获得 API 函数地址使用
5.  资源、异常、TLS 无法使用（内存加载 Dll 的通病）
6.  全局变量不可使用 ： 内存在内核地址里，其全局变量无法通过任何的 NTAPI 检查
7.  注册异常处理函数处理可能发生的 PF

**Session Space**
-----------------

22 年的时候发现一些样本申请的高位内存，对内核不可见（驱动直接访问该内存无法访问），需要挂靠到指定进程下，才可以访问该内存。分析发现一些样本利用了 Session Space 原理，实现了内存的隐藏。

Windows 下虽然只使用了 Ring0 和 Ring3，但是实际上也有任务会话层的概念在其中，并有会话隔离的概念。不同会话之间的会话内存不可共享，就像三环应用程序之间不可以共享私有内存一样。通常情况下，在单用户单桌面时，Windows 下只有两个 Session Id，也就是 0 和 1。Session Id 为 0 的进程是 System 和非 GUI 进程，而 Session Id 为 1 的进程则是 lsass、csrss、svchost 和 GUI 进程，这类进程又被称为 Session 进程。由于游戏进程通常是 GUI 进程，此时 Session Id 是 1；而在内核扫描内存时是挂靠在 System 进程下的，此时 Session Id 为 0，就无法扫描到这类挂靠在 Session Id = 1 下的内存，从而实现隐藏。

而同 SessionId 下的 SessionSpace 是共享的，故可以创建一个单独的 SessionId，将外挂进程和游戏进程挂靠在该 Session 下，创建 SessionSpace 使得外挂于游戏共享该片区域，实现注入。同时申请的这个区段可以是高位的内核内存，并与 PTE.USER 注入相结合，实现出对内核（System 进程）不可见的高位注入

```
nStatus = Utils::AttachProcess(pid);
if (!NT_SUCCESS(nStatus))
    return nStatus;
 
 
LARGE_INTEGER maximum_size;
maximum_size.QuadPart = size;
nStatus = MmCreateSection(§ion, SECTION_ALL_ACCESS, NULL, &maximum_size, PAGE_EXECUTE_READWRITE, NULL, NULL, NULL);
if (!NT_SUCCESS(nStatus))
    return nStatus;
 
 
nStatus = MmMapViewInSessionSpace(section, reinterpret_cast(mapped_base), &size);
if (!NT_SUCCESS(nStatus))
    return nStatus;
 
memset((PVOID)*mapped_base, 0, size);
 
Utils::DetachProcess(); 
```

**参考资料**
--------

1.  [VAD（Virtual Address Descriptor）虚拟地址描述符 学习笔记](https://www.cnblogs.com/revercc/p/16055714.html)
    
2.  [x64 下隐藏可执行内存](https://tttang.com/archive/1589/)
    
3.  [看雪 - 如何在驱动层完美隐藏内存](https://bbs.pediy.com/thread-265189.htm)
    
4.  [PTE.USER PresentInjector](https://github.com/Nou4r/PresentInjector)
    
5.  [invlpg windows 如何使用](https://www.csdn.net/tags/MtTaAg3sMTExNDUxLWJsb2cO0O0O.html)
    
6.  [X64 内核 SMAP,SMEP](https://bbs.pediy.com/thread-261744.htm)
    
7.  [随手写：X86 PCID 和 TLB flush](https://zhuanlan.zhihu.com/p/163524487)
    
8.  [PCID & 与 PTI 的结合](http://happyseeker.github.io/kernel/2018/05/04/pti-and-pcid.html)
    
9.  [Windows 中基于虚拟化的安全性](https://rishivoice.com/post/6367.html)
    

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌 握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

[#系统内核](forum-41-1-131.htm) [#驱动开发](forum-41-1-132.htm)