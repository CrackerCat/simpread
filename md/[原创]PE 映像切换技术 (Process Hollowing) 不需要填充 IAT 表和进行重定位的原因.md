> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268005.htm)

> [原创]PE 映像切换技术 (Process Hollowing) 不需要填充 IAT 表和进行重定位的原因

记下笔记以及一点小思考，让诸位见笑啦

### 问题阐述

在学习了 PE 加载器的实现原理后，我发现这个加载 PE 的过程与 PE 映像切换技术 (Process Hollowing) 有点类似，区别在于后者并没有去填充内存中 PE 文件的 IAT 表与进行重定位。那同样是把 PE 文件手动从硬盘里加载到内存中运行，为什么 PE 映像切换技术就没有去对内存中 PE 文件的 IAT 表进行操作呢？

 

为了叙述的完整，先介绍一下 PE 映像切换技术与 PE 加载器的具体原理。

### PE 映像切换技术 (Process Hollowing)

![](https://bbs.pediy.com/upload/attach/202106/926584_J399BQP6RJZ8XY7.png)

 

效果就是套一个傀儡进程的壳来执行我们希望执行的其他 PE 文件，《逆向工程核心原理》中给出的代码实现如下：

 

1、创建挂起的傀儡进程

```
CreateProcess(NULL, FakeProcesssPath, NULL, NULL, FALSE, CREATE_SUSPENDED, NULL, NULL, &si, &pi)

```

2、卸载掉原来的模块

```
// 通过PEB获取傀儡进程的映像基址
GetThreadContext(pi->hThread, &ctx)
ReadProcessMemory(
            pi->hProcess,
            (LPCVOID)(ctx.Ebx + 8),     // ctx.Ebx = PEB, ctx.Ebx + 8 = PEB.ImageBase
            &dwFakeProcImageBase,
            sizeof(DWORD),
            NULL) )
...
// 卸载原傀儡进程映像
pFunc = GetProcAddress(GetModuleHandle(L"ntdll.dll"), "ZwUnmapViewOfSection");
(PFZWUNMAPVIEWOFSECTION)pFunc)(pi->hProcess, (PVOID)dwFakeProcImageBase);

```

3、写入新的文件

```
// 从硬盘上读取目标PE文件
hFile = CreateFile(RealProcessPath, GENERIC_READ, FILE_SHARE_READ, NULL,  OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
dwFileSize = GetFileSize(hFile, NULL);
ReadFile(hFile, pRealFileBuf, dwFileSize, &dwBytesRead, NULL);
CloseHandle(hFile);
...
// 根据PE文件的偏移量得到各个PE头
// DOS头在PE文件的最开始
PIMAGE_DOS_HEADER       pIDH = (PIMAGE_DOS_HEADER)pRealFileBuf;
// NT头的偏移量 = pIDH->e_lfanew, 可选头相对于NT头的偏移量 = 0x18 
PIMAGE_OPTIONAL_HEADER  pIOH = (PIMAGE_OPTIONAL_HEADER)(pRealFileBuf + pIDH->e_lfanew + 0x18);
// 节区头的偏移量 = NT头的偏移量 + NT头的大小, 因为节区头位于NT头的后面
PIMAGE_SECTION_HEADER   pISH = (PIMAGE_SECTION_HEADER)(pRealFileBuf + pIDH->e_lfanew + sizeof(IMAGE_NT_HEADERS));
// 在傀儡进程中，目标PE文件基址的地址处，分配目标PE文件大小的内存
pRealProcImage = (LPBYTE)VirtualAllocEx(
    pi->hProcess,
    (LPVOID)pIOH->ImageBase,
    pIOH->SizeOfImage,
    MEM_RESERVE | MEM_COMMIT,
    PAGE_EXECUTE_READWRITE)
// 写入PE头
WriteProcessMemory(
        pi->hProcess,
        pRealProcImage,
        pRealFileBuf,
        pIOH->SizeOfHeaders,
        NULL);
// 写入各节区
for( int i = 0; i < pIFH->NumberOfSections; i++, pISH++ ){
    if( pISH->SizeOfRawData != 0 ){
        // 这里注意要将各节区写到对应的 映像基址+RVA 处的内存中
        if( !WriteProcessMemory(
                ppi->hProcess,
                pRealProcImage + pISH->VirtualAddress,
                pRealFileBuf + pISH->PointerToRawData,
                pISH->SizeOfRawData,
                NULL) ){
            printf("WriteProcessMemory(%.8X) failed!!! [%d]\n",
            pRealProcImage + pISH->VirtualAddress, GetLastError());
            return FALSE;
        }
    }
}

```

4、恢复现场

```
// 把傀儡进程的控制流修改到目标PE的入口处
GetThreadContext(pi->hThread, &ctx);
// Eax寄存器保存的值为程序的入口点地址
ctx.Eax = pIOH->AddressOfEntryPoint + pIOH->ImageBase;
SetThreadContext(pi->hThread, &ctx);
ResumeThread(pi.hThread);

```

### PE 加载器

在看雪知识库的`Windows安全-系统篇-PE格式-PE文件的加载`章节收录了许多 PE 加载器的实现文章。实现上只要模拟操作系统加载 PE 文件的方式来做就可以，简单来说分为以下几个步骤：

 

1. 申请一块内存，将 PE 文件由硬盘加载到内存中，这部分与上文基本相同

 

2. 修复重定位

 

根据重定位表的内容，把 PE 文件中的对应位置的硬编码地址进行重定位

 

例如，假设该 PE 文件的默认基址为`0x00400000`，加载到内存后的基址为`0x00100000`

 

![](https://bbs.pediy.com/upload/attach/202106/926584_XP8VZVZE5W9HKGB.png)

 

就是把 RVA 为`10F5`处的硬编码地址`0x00467C28`重定位为`0x00167C28`，即减去原基址再加上实际的基址

 

![](https://bbs.pediy.com/upload/attach/202106/926584_A76G5WHBYXJMQN6.png)

 

3. 加载导入表

 

先根据 IDT 里的 dll 名称用`LoadLibrary()`加载对应的 dll

 

![](https://bbs.pediy.com/upload/attach/202106/926584_FV5QUZZWRYARKR2.png)

 

然后到对应 dll 的导入表项 (理论上应该去 INT 中，但实际上这俩表内容是一样的) 中用`GetProcAddress()`获取导入的函数地址，并写到导入表的相应位置

 

![](https://bbs.pediy.com/upload/attach/202106/926584_2ZKYB7TPMCTQMF6.png)

 

4. 跳转到 PE 的入口点处执行

### 问题解答

问题的关键就是因为 PE 映像切换技术使用了`CreateProcess()`来挂起创建一个傀儡进程，当恢复线程执行后还会进行一些进程初始化的工作，所以不用我们手工的去填 IAT 表和进行重定位。

 

![](https://bbs.pediy.com/upload/attach/202106/926584_DY5GPNT9SS4HTPT.png)

 

进程初始化有一部分实在新的进程中进行的，就比如 IAT 表的填充和重定位。当初始线程启动时，首先会执行`KiThreadStartup`，把目标线程的 IRQL 从 DPC 级降低到 APC 级。然后调用`PspUserThreadStartup`，将用户空间`ntdll.dll`中的函数`LdrInitializeThunk`作为 APC 函数挂入 APC 队列，再企图返回到用户空间，执行`LdrInitializeThunk`，正是在这个函数中，进行了 IAT 表的填充以及重定位。

### 参考

《逆向工程核心原理》

 

[Process Hollowing and Portable Executable Relocations](https://www.ired.team/offensive-security/code-injection-process-injection/process-hollowing-and-pe-image-relocations)

 

常见进程注入的实现及内存 dump 分析——Process Hollowing（冷注入）

 

[PE 加载器的简单实现](https://bbs.pediy.com/thread-249133.htm)

 

[一种保护应用程序的方法 模拟 Windows PE 加载器，从内存资源中加载 DLL](https://bbs.pediy.com/thread-149326.htm)

 

《深入解析 Windows 操作系统》

 

《漫谈兼容内核》

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#问题讨论](forum-4-1-197.htm)