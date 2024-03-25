> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281032.htm)

> 2022 的腾讯游戏安全竞赛复盘

2022 的腾讯游戏安全竞赛复盘

2 天前 1284

### 2022 的腾讯游戏安全竞赛复盘

 [![](http://passport.kanxue.com/upload/avatar/002/919002.png?1638441945)](user-home-919002.htm) [xi@0ji233](user-home-919002.htm) ![](https://bbs.kanxue.com/view/img/rank/9.png) 8  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 2 天前  1284

复盘一下 2022 的腾讯游戏安全比赛。

初赛
--

### 题目说明

这里有一个画了 flag 的小程序，可好像出了点问题，flag 丢失了，需要把它找回来。

题目

![](https://bbs.kanxue.com/upload/attach/202403/919002_SUENS2ZBRKGKVUY.jpg)

找回 flag 样例：

![](https://bbs.kanxue.com/upload/attach/202403/919002_W4F4RQ7NRP5P26F.jpg)

**要求：**

1.  不得直接 patch 系统组件实现绘制（如：直接编写 D3D 代码绘制 flag），只能对题目自身代码进行修改或调用。
2.  找回的 flag 需要和预期图案（包括颜色）一致，如果绘制结果存在偏差会扣除一定分数。
3.  赛后需要提交找回 flag 的**截图**和**解题代码或文档**进行评分。

**评分标准：**

根据提交截图和代码文档的时间作为评分依据。

### 解题过程

先打开所给的程序，发现输出 ACE 的 LOGO 过一会后会消失。

IDA，打开找到 WinMain 函数，找到消息循环函数，分析主体逻辑

![](https://bbs.kanxue.com/upload/attach/202403/919002_4HB3S7CFNVRE3FX.jpg)

看到主体的函数

![](https://bbs.kanxue.com/upload/attach/202403/919002_ACQDGA3DMRGKZPY.jpg)

有一个初始化的操作，会分配一片可读可写可执行（`#define PAGE_EXECUTE_READWRITE 0x40`）内存并将代码拷贝过去，主要有 `140005040` 和 `140006350` 两个地址的代码。

![](https://bbs.kanxue.com/upload/attach/202403/919002_JRG7QQWB5YN5FGK.jpg)

第一段拷贝完成之后可以发现它 PATCH 了函数开头的几个字节，应该是防静态分析的，而且看字节大概是 `PUSH,POP` 指令。

下面可以看到记录了当前的时刻，如果发现起始时刻与当前时刻超过了 4000（4000MS）那么执行下面的指令，这里根据开始的运行大概能猜测出来应该就是停止绘制的代码了。

![](https://bbs.kanxue.com/upload/attach/202403/919002_YFWRZ7CB5G3ATCB.jpg)

最后调用 `shellcode+0x650` 的代码作为入口，下面可以尝试跟一下这个函数，这里可以采用静态修改代码为真实代码，也可以动调执行到这里的时候反编译。

直接跟到入口，可以很明显地发现函数入口往后有一段对自身的调用

![](https://bbs.kanxue.com/upload/attach/202403/919002_JJPTXHFJXP39BCX.jpg)

看地址是 `shellcode+0x420`，跟过去，重建函数，是一个很标准的虚拟机流程

![](https://bbs.kanxue.com/upload/attach/202403/919002_B8NUV4U7WYH5R4M.jpg)

虚拟机的代码存在于 `shellcode+0x1301` 的地址，也就是第二次拷贝得到的代码。

根据自己的理解还原了一下虚拟机的流程：

![](https://bbs.kanxue.com/upload/attach/202403/919002_ASNP5FRQXGK9G77.jpg)

*   op0：Stack[0]+=Stack[1]
*   op1：Stack[0]-=Stack[1]
*   op2 num1,num2：Stack[num2]=Stack[num1]
*   op3 num1,num2：Stack[num2]=num1
*   op4 num：这里的操作很神奇，会把栈中第一个值赋值为 `num^'ACE'`，第二个值赋值为一个很复杂的运算。
*   op5：调用 shellcode 头部的函数。
*   op6：调用 shellcode 头部的函数，与上一个唯一的区别是第五个参数。
*   op7：退出

而这里的 v16-v18 大概率也是 op2 和 op3 会操作到的，也算栈中的值。

这里很容易猜测 op5 和 op6 应该是绘制函数的代码。

FFFF00 刚好是黄色的代码，将取色器放置在程序上也能发现蓝色的代码是 2DDBE7，和这里的颜色代码差了一点，但是可以尝试修改一下。

CE 找到这个位置，将代码修改一下

![](https://bbs.kanxue.com/upload/attach/202403/919002_E33CD2ZD9ZU4XZA.jpg)

这里我直接把它改成 000000 也就是黑色，直接跳出来。

然后执行完下面的代码，发现输出变成黑色了

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

那么主要肯定是要分析 `shellcode+0` 处的函数代码了（看不懂直接放弃）。

虽然看不懂，但是已经知道里面传的一个值是颜色了，去分析分析其他参数的含义就可以了。这里最好的一个办法应该是 hook，去打印它的参数，为了方便可以把它限时输出这点 PATCH 了。

这个直接去找到它的跳转让它永远跳转或者永远不跳转就行了，这里是改成永远跳转，90 加前面，偏移可以不用动。

![](https://bbs.kanxue.com/upload/attach/202403/919002_5ZS9QQBGWTBH4KE.jpg)

写一个 DLL 去做 HOOK，主要去 HOOK shellcode，这个地址通过全局变量可以获得（2022 游戏安全技术竞赛初赛. exe+0x8308）。

64 位的程序 hook 一般直接用 `inline` 或者 `hotfix` 或者无痕，个人感觉 `hotfix` 实现起来简单，但是个人更喜欢 `inline hook`，因为它 `windows` 消息的机制，会不停地打印数据，因此加全局变量限制输出前 100 次调用的结果。

注入器（基本通用的）：

```
#include #include #include #include #include #define EXEFILEW L"2022游戏安全技术竞赛初赛.exe"
#define EXEFILE "2022游戏安全技术竞赛初赛.exe"
DWORD old;
SIZE_T written;
DWORD FindProcess() {
    HANDLE hSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    PROCESSENTRY32 pe32;
    pe32 = { sizeof(pe32) };
    BOOL ret = Process32First(hSnap, &pe32);
    while (ret)
    {
        if (!wcsncmp(pe32.szExeFile,  EXEFILEW, lstrlen(EXEFILEW))) {
            printf("找到程序 %s ,PID=%d\n", EXEFILE, pe32.th32ProcessID);
            return pe32.th32ProcessID;
        }
        ret = Process32Next(hSnap, &pe32);
    }
    return 0;
}
void InjectModule(DWORD ProcessId, const char* szPath)
{
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, ProcessId);
    printf("进程句柄:%p\n", hProcess);
    LPVOID lpAddress = VirtualAllocEx(hProcess, NULL, 0x100, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    SIZE_T dwWriteLength = 0;
    WriteProcessMemory(hProcess, lpAddress, szPath, strlen(szPath), &dwWriteLength);
    HANDLE hThread = CreateRemoteThread(hProcess, NULL, NULL, (LPTHREAD_START_ROUTINE)LoadLibraryA, lpAddress, NULL, NULL);
    WaitForSingleObject(hThread, -1);
    VirtualFreeEx(hProcess, lpAddress, 0, MEM_RELEASE);
    CloseHandle(hProcess);
    CloseHandle(hThread);
}
int main() {
    DWORD ProcessId = FindProcess();
    while (!ProcessId) {
        printf("未找到%s程序，等待两秒中再试\n",EXEFILE);
        Sleep(2000);
        ProcessId = FindProcess();
    }
    InjectModule(ProcessId, "C:\\Users\\xia0ji233\\source\\repos\\T2022Pre\\x64\\Debug\\hack.dll");
} 
```

用于 HOOK 的 DLL：

```
// dllmain.cpp : 定义 DLL 应用程序的入口点。
#include "pch.h"
#include #include #include typedef __int64 (*Func)(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, __int64 a8, __int64 a9, __int64 a10);
__int64 GetBaseAddr() {
    HMODULE hMode = GetModuleHandle(nullptr);
    return (__int64)hMode;
}
void* shellcode = 0;
BYTE HookCode[] = {
    0x48,0xB8,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,  //mov rax,xxx
    0xFF,0xE0                                           //jmp rax
};
BYTE OriginCode[0x50];
size_t HookLen = 12;
__int64 times = 100;
__int64 HackShellcode(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, __int64 a8, __int64 a9, __int64 a10) {
    memcpy(shellcode, OriginCode,HookLen);              //unhook
    //
    int x = a1, y = a2;
    __int64 ret=(*(Func)shellcode)(x, y, a3, a4, 0xFFFF0000, a6, a7, a8, a9, a10);
    times--;
    if (times>0) {
        printf("call shellcode(%d,%d,%d,%d,%d,%p,%p,%p,%p,%p)\n",x, y, a3, a4, a5, a6, a7, a8, a9, a10);         
 
    }
    memcpy(shellcode, HookCode, HookLen);               //rehook
    return ret;
}
 
 
void HookShellcode() {
    __int64 base = GetBaseAddr();
    __int64 Ptr = base + 0x8308;
 
    shellcode = (void*)(*(__int64*)Ptr);
    while (!shellcode) {
        shellcode = (void*)(*(__int64*)Ptr);
        printf("Find shellcode Fail\n");
        Sleep(200);
    }
    printf("shellcode addr=%p\n", shellcode);
    memcpy(OriginCode, shellcode,HookLen);              //saved
    Func FuncPtr = HackShellcode;
    *(__int64*)(HookCode + 2) = (__int64)FuncPtr;       //construct
    memcpy(shellcode, HookCode, HookLen);               //hook
 
}
 
 
BOOL APIENTRY DllMain( HMODULE hModule,
                      DWORD  ul_reason_for_call,
                      LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
        case DLL_PROCESS_ATTACH:
            AllocConsole();
            freopen("CONOUT$", "w", stdout);
            HookShellcode();
        case DLL_THREAD_ATTACH:
        case DLL_THREAD_DETACH:
        case DLL_PROCESS_DETACH:
            break;
    }
    return TRUE;
} 
```

首先是为了输出方便创建一个终端定向标准输出，然后 IO 函数就能往终端打印了。

hook 其实就是覆盖函数头，劫持到自己的函数里面，打印出参数之后把钩子去掉，恢复回原来的样子，再去正常调用，调用结束之后重新挂回钩子，要实现输出前 100 条的话最后不要重新挂钩子就行。

结果（PS：输出是正常的，但是我强制都改成黄色绘制了）：

![](https://bbs.kanxue.com/upload/attach/202403/919002_XB25R4MKBFPJQA3.jpg)

因为前几个参数是 `__int32` 类型的，所以直接换 `%d` 打印一下，这里放一下部分的数据：

```
shellcode addr=000001C5654B0000
call shellcode(-950,50,-14703700,1248208,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(50,-390,1822057,-1524539,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(-950,170,-7425437,14227863,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(50,230,1897472,15215384,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(-950,-210,3743658,7267794,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(50,350,966258,14122466,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(-890,-270,463184,7666472,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(170,-270,-2474473,12856971,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(-770,230,3460907,-13492529,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(110,-390,-5989351,14280177,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(-830,170,-3514649,12856715,ffffff00,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(650,50,15384343,11002795,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(590,110,12856715,13307703,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(530,170,14096303,2054931,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(470,230,12869865,15314261,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(410,290,13085065,12188981,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(470,290,11235584,6581496,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(530,290,12847529,3263901,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(710,50,14122635,3090291,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(770,50,14265601,10052917,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(830,50,3793238,14305830,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(710,110,1253500,3434568,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)
call shellcode(770,170,13177867,745383,ff2ddbe7,000001C56640AC28,000001C5701C78B0,000001C5665CA4D8,000001C5704025C8,000001C570402D88)

```

可以发现，前两个参数应该是坐标（我猜的），但是出现了负值，也就是打印到了屏幕外面，而且都是黄色的点会出现这种情况，导致了 `FLAG` 打印不出来，因此尝试在 hook 层面修复这个 bug，把所有的负值翻转，但是发现并没什么用，说明 bug 应该不止那么简单，还得再分析分析。

首先就是想看看它一轮有多少个点，直接建个 `set` 去输出就行，把所有点保存下来。

DLL 代码：

```
// dllmain.cpp : 定义 DLL 应用程序的入口点。
#include "pch.h"
#include #include #include #include std::set>s;
 
typedef __int64 (*Func)(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, __int64 a8, __int64 a9, __int64 a10);
__int64 GetBaseAddr() {
    HMODULE hMode = GetModuleHandle(nullptr);
    return (__int64)hMode;
}
void* shellcode = 0;
BYTE HookCode[] = {
    0x48,0xB8,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,  //mov rax,xxx
    0xFF,0xE0                                           //jmp rax
};
BYTE OriginCode[0x50];
size_t HookLen = 12;
__int64 times = 100;
void printset() {
    for (auto k : s) {
        printf("(%d,%d)\n", k.first, k.second);
    }
}
__int64 HackShellcode(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, __int64 a8, __int64 a9, __int64 a10) {
    static int flag = 1;
    memcpy(shellcode, OriginCode,HookLen);              //unhook
    //
    int x = a1, y = a2;
    __int64 ret=(*(Func)shellcode)(x, y, a3, a4, a5, a6, a7, a8, a9, a10);
    times--;
    if (times>0) {
        printf("call shellcode(%d,%d,%d,%d,%x,%p,%p,%p,%p,%p)\n",x, y, a3, a4, a5, a6, a7, a8, a9, a10);         
    }
    int presize = s.size();
    s.insert({ x,y });
    if (s.size() == presize) {
        if (flag) {
            printset();
            flag = 0;
        }
    }
    memcpy(shellcode, HookCode, HookLen);               //rehook
    return ret;
}
 
 
void HookShellcode() {
    __int64 base = GetBaseAddr();
    __int64 Ptr = base + 0x8308;
     
    shellcode = (void*)(*(__int64*)Ptr);
    while (!shellcode) {
        shellcode = (void*)(*(__int64*)Ptr);
        printf("Find shellcode Fail\n");
        Sleep(200);
    }
    printf("shellcode addr=%p\n", shellcode);
    memcpy(OriginCode, shellcode,HookLen);              //saved
    Func FuncPtr = HackShellcode;
    *(__int64*)(HookCode + 2) = (__int64)FuncPtr;       //construct
    memcpy(shellcode, HookCode, HookLen);               //hook
 
}
 
 
BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        AllocConsole();
        freopen("CONOUT$", "w", stdout);
        HookShellcode();
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
} 
```

输出：

```
(-950,-210)
(-950,50)
(-950,170)
(-890,-270)
(-830,170)
(-770,230)
(50,-390)
(50,230)
(50,350)
(110,-390)
(170,-270)
(410,290)
(470,230)
(470,290)
(530,170)
(530,290)
(590,110)
(650,50)
(710,50)
(710,110)
(770,50)
(770,170)
(830,50)
(830,230)
(890,290)
(950,50)
(950,290)
(1010,50)
(1010,110)
(1010,290)
(1070,50)
(1070,170)
(1070,290)
(1130,50)
(1130,170)
(1130,230)
(1190,170)
(1190,290)
(1250,170)
(1250,290)
(1310,290)
(1370,290)

```

一共是 42 个点，而数了一下它题目给的正确的点数刚好也是 42 个，黄点是有 11 个， 蓝点有 31 个。

但是稍微改了一下第一个第二个参数发现点会直接消失，看样子是跟第三个第四个参数会有关系。

![](https://bbs.kanxue.com/upload/attach/202403/919002_RQVSY74ATYMGD5K.jpg)

通过左右参数的比对观察一下，正确调用时的这些局部变量分别是多少，看看能不能找到点关系（放弃）。

还是选择自己写一个虚拟机去跑。

代码太长了不放了，可以自己去 dump，我写一下我自己的虚拟机调试流程：

```
#include #include #include int code[1855];     //...
int main(){
    int RIP=0;
    int reg;
    int Stack[50];
    int v13,v14;
    printf("start\n");
    memset(Stack,0,sizeof(Stack));
    Stack[8]=Stack[9]=50;//截图中忘了这一句，不要忘了加上
    while(RIP<=0x1301){
        //printf("execute opcode=%d RIP=%d\n",code[RIP],RIP);
        int opcode=code[RIP];
        RIP++;
        switch (opcode) {
 
            case 0:
                Stack[0]+=Stack[1];
                printf("stack[0]=%d+%d\n",Stack[0],Stack[1]);
                break;
            case 1:
                Stack[0]-=Stack[1];
                printf("stack[0]=%d-%d\n",Stack[0],Stack[1]);
                break;
            case 2:
                Stack[code[RIP+1]]=Stack[code[RIP]];
                printf("Stack[%d]=Stack[%d]=%d\n",code[RIP+1],code[RIP],Stack[code[RIP]]);
                RIP+=2;
                break;
            case 3:
                Stack[code[RIP+1]]=code[RIP];
                printf("Stack[%d]=%d\n",code[RIP+1],code[RIP]);
                RIP+=2;
                break;
            case 4:
                v13=Stack[0];
                v14=Stack[0]*(Stack[1]+1);
                printf("Origin: Stack[0]=%d Stack[1]=%d ",Stack[0],Stack[1]);
                Stack[0]=code[RIP]^0x414345;
                Stack[1]=((Stack[0] ^ (Stack[1] + v13)) % 256+ (((Stack[0] ^ (v13 * Stack[1])) % 256 + (((Stack[0] ^ (Stack[1] + v14)) % 256) << 8)) << 8));
                printf("Target: Stack[0]=%d Stack[1]=%d\n",Stack[0],Stack[1]);
                RIP+=1;
                break;
            case 5:
                printf("paint(%d,%d,%d,%d,0xFFFFFF00);\n",Stack[4],Stack[5],Stack[6],Stack[7]);
                break;
            case 6:
                printf("paint(%d,%d,%d,%d,0xFF2DDBE7);\n",Stack[4],Stack[5],Stack[6],Stack[7]);
                break;
            case 7:
                printf("exit\n");
                exit(0);
            default:
                exit(0);
                break;
 
        }
    }
} 
```

运行之后发现了一点：

![](https://bbs.kanxue.com/upload/attach/202403/919002_4D64MU5PG43TU42.jpg)

第三个和第四个参数分别为用 `opcode==4` 时候的 x 和 y 的坐标运算得到的值。

试试看利用 opcode=4 的流程能否让程序任意位置输出色块。

```
v13=x;
v14=x*(y+1);
a3=operand^0x414345;
a4=((x ^ (y + v13)) % 256+ (((x ^ (v13 * y)) % 256 + (((x ^ (y + v14)) % 256) << 8)) << 8));

```

因为每次的这个操作数都不同，这里选取第一个错误坐标 `-950 50` 来绘制，利用这里的逻辑去做。

DLL 代码：

```
// dllmain.cpp : 定义 DLL 应用程序的入口点。
#include "pch.h"
#include #include #include #include std::set>s;
 
typedef __int64 (*Func)(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, __int64 a8, __int64 a9, __int64 a10);
__int64 GetBaseAddr() {
    HMODULE hMode = GetModuleHandle(nullptr);
    return (__int64)hMode;
}
void* shellcode = 0;
BYTE HookCode[] = {
    0x48,0xB8,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,  //mov rax,xxx
    0xFF,0xE0                                           //jmp rax
};
BYTE OriginCode[0x50];
size_t HookLen = 12;
__int64 times = 100;
void printset() {
    for (auto k : s) {
        printf("(%d,%d)\n", k.first, k.second);
    }
}
__int64 HackShellcode(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, __int64 a8, __int64 a9, __int64 a10) {
    static int flag = 1;
    memcpy(shellcode, OriginCode,HookLen);              //unhook
    //
    int x = a1, y = a2;
    int v13, v14;
    if (x == -950 && y == 50) {
        x = 50;
        y = 50;
        v13=x;
        v14=x*(y+1);
        //printf("Origin: Stack[0]=%d Stack[1]=%d ",Stack[0],Stack[1]);
        //printf("num=0x%x ",code[RIP]);
        a3=0x524895^0x414345;
        a4=(unsigned int)((a3 ^ (y + v13)) % 256
                        + (((a3 ^ (v13 * y)) % 256 + (((a3 ^ (y + v14)) % 256) << 8)) << 8));
    }
    __int64 ret=(*(Func)shellcode)(x, y, a3, a4, a5, a6, a7, a8, a9, a10);
    times--;
    if (times>0) {
        printf("call shellcode(%d,%d,%d,%d,%x) retval=%d\n",x, y, a3, a4, a5, ret);         
    }
    int presize = s.size();
    s.insert({ x,y });
    if (s.size() == presize) {
        if (flag) {
            printset();
            flag = 0;
        }
    }
    memcpy(shellcode, HookCode, HookLen);               //rehook
    return ret;
}
 
 
void HookShellcode() {
    __int64 base = GetBaseAddr();
    __int64 Ptr = base + 0x8308;
     
    shellcode = (void*)(*(__int64*)Ptr);
    while (!shellcode) {
        shellcode = (void*)(*(__int64*)Ptr);
        printf("Find shellcode Fail\n");
        Sleep(200);
    }
    printf("shellcode addr=%p\n", shellcode);
    memcpy(OriginCode, shellcode,HookLen);              //saved
    Func FuncPtr = HackShellcode;
    *(__int64*)(HookCode + 2) = (__int64)FuncPtr;       //construct
    memcpy(shellcode, HookCode, HookLen);               //hook
 
}
 
 
BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        AllocConsole();
        freopen("CONOUT$", "w", stdout);
        HookShellcode();
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
} 
```

注入之后，在指定的位置输出了黄色方块

![](https://bbs.kanxue.com/upload/attach/202403/919002_B64GVZ3VVWKQPC4.jpg)

说明修复思路是没有问题的，接下来可以用虚拟机流程把错误的坐标和对应的该操作数 dump 出来，hook 的时候进行替换。

后面用截图工具比了一下，发现它们水平距离都一样的，所以可以用已有的正确坐标参考，从上到下坐标分别为 `50,110,170,230...` 就是每隔一个查了 `60` 的距离，水平距离也同样是 60，那么最靠左的正确的方块是 `(410,290)`，肉眼分析下来，最左边的坐标是 50，y 坐标因为对对齐的也是 50，所以第一个色块是完美还原的。

那么最左边 6 个就是

```
50,50
50,110
50,170
50,230
50,290
50,350

```

对角线延伸出去三个就是

```
110,110
170,170
230,230

```

最后两个补齐 FLAG 是

```
110,230
170,230

```

最后的 DLL：

```
// dllmain.cpp : 定义 DLL 应用程序的入口点。
#include "pch.h"
#include #include #include #include std::set>s;
 
typedef __int64 (*Func)(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, __int64 a8, __int64 a9, __int64 a10);
__int64 GetBaseAddr() {
    HMODULE hMode = GetModuleHandle(nullptr);
    return (__int64)hMode;
}
void* shellcode = 0;
BYTE HookCode[] = {
    0x48,0xB8,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,  //mov rax,xxx
    0xFF,0xE0                                           //jmp rax
};
BYTE OriginCode[0x50];
size_t HookLen = 12;
__int64 times = 100;
void printset() {
    for (auto k : s) {
        printf("(%d,%d)\n", k.first, k.second);
    }
}
int val[] = {5392533,5934636,9984722,11102301,7888111,9846439,4608533,8744398,7703662,10004148,8744654};
std::pairWrongPos[] = {
    {-950,50},
    {50,-390},
    {-950,170},
    {50,230},
    {-950,-210},
    {50,350},
    {-890,-270},
    {170,-270},
    {-770,230},
    {110,-390},
    {-830,170},
};
std::pairTargetPos[] = {
    {50,50},
    {50,110},
    {50,170},
    {50,230},
    {50,290},
    {50,350},
    {110,110},
    {170,170},
    {230,230},
    {110,230},
    {170,230},
};
int CountOfWrongPos=11;
__int64 HackShellcode(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, __int64 a8, __int64 a9, __int64 a10) {
    static int flag = 1;
    memcpy(shellcode, OriginCode,HookLen);              //unhook
    //
    int x = a1, y = a2;
    int v13, v14;
    for (int i = 0; i < CountOfWrongPos; i++) {
        if (std::pair{ x,y } == WrongPos[i]) {
            x = TargetPos[i].first;
            y = TargetPos[i].second;
            v13=x;
            v14=x*(y+1);
            a3=val[i]^0x414345;
            a4=(unsigned int)((a3 ^ (y + v13)) % 256
                        + (((a3 ^ (v13 * y)) % 256 + (((a3 ^ (y + v14)) % 256) << 8)) << 8));
            printf("Hook Value Success from position (%d,%d) to (%d,%d)\n",a1,a2,x,y);
            break;
        }
    }
    __int64 ret=(*(Func)shellcode)(x, y, a3, a4, a5, a6, a7, a8, a9, a10);
    times--;
    if (times>0) {
        printf("call shellcode(%d,%d,%d,%d,%x) retval=%d\n",x, y, a3, a4, a5, ret);         
    }
    int presize = s.size();
    s.insert({ x,y });
    if (s.size() == presize) {
        if (flag) {
            printset();
            flag = 0;
        }
    }
    memcpy(shellcode, HookCode, HookLen);               //rehook
    return ret;
}
 
 
void HookShellcode() {
    __int64 base = GetBaseAddr();
    __int64 Ptr = base + 0x8308;
     
    shellcode = (void*)(*(__int64*)Ptr);
    while (!shellcode) {
        shellcode = (void*)(*(__int64*)Ptr);
        printf("Find shellcode Fail\n");
        Sleep(200);
    }
    printf("shellcode addr=%p\n", shellcode);
    memcpy(OriginCode, shellcode,HookLen);              //saved
    Func FuncPtr = HackShellcode;
    *(__int64*)(HookCode + 2) = (__int64)FuncPtr;       //construct
    memcpy(shellcode, HookCode, HookLen);               //hook
 
}
 
 
BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        AllocConsole();
        freopen("CONOUT$", "w", stdout);
        HookShellcode();
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
} 
```

最终结果也是完美实现了

![](https://bbs.kanxue.com/upload/attach/202403/919002_JGTRP7CZRPAWXHU.jpg)

* * *

今天试试复盘这个决赛

<!--more-->

题目
--

### 介绍

这里有一个在屏幕上画 flag 的小程序，可好像出了点问题，flag 丢失了，需要把它找回来，并尝试截图留念。

![](https://bbs.kanxue.com/upload/attach/202403/919002_99A9AYQMUPEY5RJ.jpg)

找回 flag 样例：

![](https://bbs.kanxue.com/upload/attach/202403/919002_Z5YRTW54CUVYBKC.jpg)

### 要求

*   自行寻找办法加载驱动文件，再执行题目 exe 文件。
*   不得直接 patch 系统组件实现绘制（如：直接编写 D3D 代码绘制 flag），只能对题目自身代码进行修改或调用。
*   找回的 flag 需要和预期图案（包括颜色）一致，如果绘制结果存在偏差会扣除一定分数。
*   修复后的 flag 截图操作必须在题目同一系统环境中进行（如：虚拟机运行题目则在虚拟机中截图，本机运行题目则在本机截图；不得拍照）。
*   赛后需要提交找回 flag 的**截图**、**解题代码或文档**和**截图代码或文档**进行评分，方法越多得分越高。
*   建议使用系统版本：Win10 1809、Win10 1903、Win10 1909、Win10 2004、Win10 20h1、Win10 20h2、Win10 21h1、Win10 21h2，在虚拟机中可能无法正常显示图形。
*   提交结果打包为 XXX_writeup_A.zip，XXX 为名称，A 为提交序号，从 1 开始。

分析
--

P.S.，在做复现的时候发现虚拟机无法正常绘制，且自己 Win11 的物理机运行会蓝屏，因此本次复现不含动态调试部分，一切只停留于静态分析和理论阶段，刷它指定的系统成本过高了接受不了。

### 驱动分析

IDA 打开，

`sub_140001150` 函数很像是注册的驱动卸载函数。

`sub_140001188` 函数应该就是获取了一下系统信息，没什么东西。

`sub_140001414` 函数往下跟到 `sub_1400014A0` 函数有大东西，不过这个函数不是直接调用的，像是注册了某种回调，三环程序应该是处罚这个回调的。

开头通过调用 `sub_140001318` 函数获得了 `dwm.exe` 的 `EPROCESS` 结构。

![](https://bbs.kanxue.com/upload/attach/202403/919002_RYYA5V529GSZWH4.jpg)

对于接下来调用的函数

![](https://bbs.kanxue.com/upload/attach/202403/919002_QT9SPC43T57T4CD.jpg)

`sub_140001000` 比较像是获取指定进程的某个 `DLL`，具体也跟进来看看

![](https://bbs.kanxue.com/upload/attach/202403/919002_G38ZCMJBTHKKHAW.jpg)

对于这些 API，网上找到了一些说法：

**GetUserModuleBaseAddress():** 实现取进程中模块基址，该功能在`《驱动开发：内核取应用层模块基地址》`中详细介绍过原理，这段代码核心原理如下所示，此处最需要注意的是如果是`32位进程`则我们需要得到`PPEB32 Peb32`结构体，该结构体通常可以直接使用`PsGetProcessWow64Process()`这个内核函数获取到，而如果是`64位进程`则需要将寻找 PEB 的函数替换为`PsGetProcessPeb()`

这个地方也不难判断，就是获取 PEB 结构体，只不过多了一个 32 位和 64 位的判断，以 32 位的为例，中间有类似遍历链表的写法，如果找到了那么把某个结果保存到第二个参数指向的位置然后返回。

这里且当 `sub_140001264(v24, "D3DCompile");` 函数是获取了某个函数的地址作为返回值出去的，随后是比较关键的点

![](https://bbs.kanxue.com/upload/attach/202403/919002_GJRMM7J3QE59RCR.jpg)

调用了两次 `ZwAllocateVirtualMemory` 函数给进程申请内存，然后拷贝 shellcode 并进行了一定的异或混淆，最关键它把 `D3DCompile` 的地址和第二次申请内存的地址保存在第一次申请的内存后方，应该是方便 `shellcode` 找到虚拟代码，剩下的大概没有什么了，虽然没有运行成功大概也能猜测这个 shellcode 应该就是直接在屏幕绘制的代码了。

### exe 分析

三环程序比较大，先用火绒剑分析一下行为，主要是排除 exe 有跟内核做直接数据交互。

![](https://bbs.kanxue.com/upload/attach/202403/919002_PWMDSDNHSNJTFPP.jpg)

然而并没有，但是发现它也打开了 `D3DCompiler_47.dll`，于是从这里开始交叉引用，通过 DLL 路径交叉是一个比较好的思路，不管动态加载或者是运行时直接导入，都是可以大概分析到主逻辑的。

![](https://bbs.kanxue.com/upload/attach/202403/919002_JJFZV2FA69D26PS.jpg)

里面就进行了一个 `NtQuerySystemInformation`，外面是创建线程调用的这个函数，这里应该是触发回调的一个函数，为了验证也是准备去调试，但是它根本不触发这个回调，如图所见。

![](https://bbs.kanxue.com/upload/attach/202403/919002_KA5HDPSG8GCFVYF.jpg)

之前配置环境的时候一直以为是虚拟机没有 `dwm.exe` 这个进程，结果没想到是回调没有办法调用，于是我选择自己运行一个 dwm.exe 进程（我直接拿初赛的三环程序去改名然后运行，可以在第一个函数成功被获取），然后自己写一个驱动手动调用那个回调写 shellcode。

```
#include"ntddk.h" 
#define kprintf(format, ...) DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, format, ##__VA_ARGS__)
typedef void (* func)();
PDRIVER_OBJECT g_Object = NULL;
typedef struct _LDR_DATA_TABLE_ENTRY {
    LIST_ENTRY InLoadOrderLinks;
    LIST_ENTRY InMemoryOrderLinks;
    LIST_ENTRY InInitializationOrderLinks;
    PVOID DllBase;
    PVOID EntryPoint;//驱动的进入点 DriverEntry 
    ULONG SizeOfImage;
    UNICODE_STRING FullDllName;//驱动的满路径 
    UNICODE_STRING BaseDllName;//不带路径的驱动名字 
    ULONG Flags;
    USHORT LoadCount;
    USHORT TlsIndex;
    union {
        LIST_ENTRY HashLinks;
        struct {
            PVOID SectionPointer;
            ULONG CheckSum;
        };
    };
    union {
        struct {
            ULONG TimeDateStamp;
        };
        struct {
            PVOID LoadedImports;
        };
    };
} LDR_DATA_TABLE_ENTRY, *PLDR_DATA_TABLE_ENTRY;
 
VOID bianliqudongmokuai(PUNICODE_STRING name, UINT64* pBaseAddr,UINT64* pSize)
{
    LDR_DATA_TABLE_ENTRY*TE, *Tmp;
    TE = (LDR_DATA_TABLE_ENTRY*)g_Object->DriverSection;
    PLIST_ENTRY LinkList;
    ;
    int i = 0;
    LinkList = TE->InLoadOrderLinks.Flink; 
    while (LinkList != &TE->InLoadOrderLinks)
    {
        Tmp = (LDR_DATA_TABLE_ENTRY*)LinkList;
        if (RtlCompareUnicodeString(&Tmp->BaseDllName, name, FALSE))
        {
        }
        else
        {
            kprintf(("Found Module!\n"));
            *pBaseAddr = (UINT64)(Tmp->DllBase);
            *pSize = (UINT64)(Tmp->SizeOfImage);
 
        }
        LinkList = LinkList->Flink;
        i++;
    }
 
 
}
VOID Unload(PDRIVER_OBJECT DriverObject)
{
    kprintf(("BYE xia0ji233\n"));
}
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    kprintf(("Hello xia0ji233\n"));
    g_Object = DriverObject;
    DriverObject->DriverUnload = Unload;
    UNICODE_STRING name;
    UINT64 Base = 0;
    UINT64 Size=0;
    RtlInitUnicodeString(&name, L"2022GameSafeRace.sys");
    //
    bianliqudongmokuai(&name,&Base,&Size);
    if (Base) {
        kprintf(("Base:%p Size:%x\n"), Base, Size);
        func funcptr = (func)(Base + 0x1490);
        DbgBreakPoint();
        funcptr();
    }
 
    return STATUS_SUCCESS;
}

```

运行结果如下，用初赛的 exe 改名为 dwm 成功被这个函数获取到 EProcess。

![](https://bbs.kanxue.com/upload/attach/202403/919002_XYBS24NWPQEYUP7.jpg)

随后使用静态分析去解一下 `shellcode`，用下面的 IDC 脚本即可

```
#include static main(){
    auto start_ea = 0x000000140005A00;
    auto end_ea =  0x000000140005A00+0x16E6;
    auto len = end_ea - start_ea;
    auto ea=0;
    for (ea = start_ea; ea < end_ea; ea++) {
        PatchByte(ea, Byte(ea)^0xC3);   
    }
} 
```

解密后的 `shellcode` 可以被直接反编译

![](https://bbs.kanxue.com/upload/attach/202403/919002_59NB9PSSEWEWWZ4.jpg)

看起来跟初赛是差不多的，相同的配方，相同的味道。

再往下看

![](https://bbs.kanxue.com/upload/attach/202403/919002_DXBDFC6A2C2QZBS.jpg)

就连这个 ACE 都是一样的，这里大概是一个全新的虚拟机了。

然后本来是打算搜字节码去看看 shellcode 有没有写成功的，但是发现还是搜不到，突然想到好像这个回调最后会 free 这片内存，所以决定直接改 sys 去把原来的 free 给 `jmp` 掉（还是失败，想复现太难了 qwq）。

还是老老实实分析虚拟机代码吧，看到 `unk_140004030`，它被放到了 `BaseAddress + 0x16E6` 的位置上，这里的代码在我们看来是在 `0x140005A00`，而直接分析可得，代码实际在 `&qword_140009600[136]=0x140009600+136*8=0x140009a40` 的位置上。

然而这里没找到对应的数据，确实也不太会分析了，按理来说如果能直接调试运行到这的话是肯定可以定位 shellcode 找到位置 dump 出来的。

这里还原一下虚拟机的流程吧

```
#include #include #include unsigned int code[] ={0};
 
int main(){
    int Stack[0x50];
    unsigned __int64 RIP_S=0; // rsi
    unsigned __int64 v10; // r8
    unsigned int opcode; // ecx
    unsigned __int64 v12; // rdx
    __int64 v13; // rcx
    __int32 v14; // r9d
    __int64 v15; // r8
    __int64 v16; // r9
    __int32 v17; // edx
    unsigned __int64 v18; // r10
    unsigned __int64 v19; // rcx
    __int32 v20; // r9d
    __int32 v21; // r9d
    __int32 v22; // eax
    __int64 result; // rax
    memset(Stack,0,sizeof(Stack));
    Stack[8] = 50;
    Stack[9] = 50;
    do
    {
        v10 = RIP_S;
        opcode = code[RIP_S + 272];
        if ( (int)opcode > (int)0x9A8ECD52 )
        {
            switch ( opcode )
            {
                case 0xEE2362FC:
                    ++RIP_S;
                    v21 = Stack[0];
                    v22 = Stack[0] * (Stack[1] + 1);
                    Stack[0] = code[RIP_S + 272] ^ 0x414345;
                    Stack[1] = (Stack[0] ^ (Stack[1] + v21)) % 256
                        + (((Stack[0] ^ (v21 * Stack[1])) % 256 + (((Stack[0] ^ (Stack[1] + v22)) % 256) << 8)) << 8);
                    break;
                case 0xEE69524A:
                    v19 = 0;
                    v20 = code[v10 + 273];
                    code[RIP_S + 272] = -1;
                    code[v10 + 273] = -1;
                    if ( RIP_S != 1 )
                    {
                        do
                        {
                            code[v19 + 272] ^= v20;
                            ++v19;
                        }
                        while ( v19 < RIP_S - 1 );
                    }
                    ++RIP_S;
                    break;
                case 0xFF4578AE:
                    RIP_S += 2;
                    v16 = code[v10 + 273];
                    v17 = code[RIP_S + 272];
                    if ( v16 )
                    {
                        v18 = RIP_S;
                        do
                        {
                            code[++v18 + 272] ^= v17;
                            v17 = code[v18 + 271] + 305419896 * v17;
                            --v16;
                        }
                        while ( v16 );
                    }
                    code[v10 + 272] = -1;
                    code[v10 + 273] = -1;
                    code[RIP_S + 272] = -1;
                    break;
                case 0x1132EADF:
                    RIP_S += 2;
                    Stack[code[RIP_S + 272]] = code[v10 + 273];
                    break;
                default:
                    if ( opcode == 2018683631 && code[272] == -295083446 && code[273] == 1755241482 && code[274] == -1729111095 )
                        printf("call Paint(%d, %d, %d, %d, NAN, a3, a4, a5, a6, a7)",Stack[4], Stack[5], Stack[6], Stack[7]);
                    break;
            }
        }
        else
        {
            switch ( opcode )
            {
                case 0x9A8ECD52:
                    Stack[0] -= Stack[1];
                    break;
                case 0x88659264:
                    RIP_S += 2;
                    v12 = RIP_S;
                    v13 = code[v10 + 273];
                    v14 = code[RIP_S + 272];
                    code[v10 + 272] = -1;
                    code[v10 + 273] = -1;
                    v15 = v13;
                    code[RIP_S + 272] = -1;
                    if ( v13 )
                    {
                        do
                        {
                            code[++v12 + 272] ^= v14;
                            --v15;
                        }
                        while ( v15 );
                    }
                    break;
                case 0x89657EAD:
                    Stack[0] += Stack[1];
                    break;
                case 0x8E7CADF2:
                    RIP_S += 2;
                    Stack[code[RIP_S + 272]] = Stack[code[v10 + 273]];
                    break;
                case 0x9645AAED:
                    if ( code[272] == 0xEE69624A && code[273] == 0x689EDC0A && code[274] == 0x98EFDBC9 )
                        printf("call Paint(%d, %d, %d, %d, NAN, a3, a4, a5, a6, a7)",Stack[4], Stack[5], Stack[6], Stack[7]);
                    break;
                case 0x9645AEDC:
                    RIP_S = 0x671;
                    break;
            }
        }
        result = 0x671;
        ++RIP_S;
    }
    while ( RIP_S < 0x671 );
} 
```

对比起来这个虚拟机的流程也是更大更难去分析了，但是根据已有的资料看来，似乎出的问题与初赛一致，最好的办法就是做 hook 然后替换坐标。据说决赛是卷方法数，当然其他的方法也可以有，这里可以说一些理论可行的方案：

*   自己生成正确的指令流，直接 PATCH SYS 文件。
*   等代码注入完成之后，搜索指令的特征码找到三环程序中代码的位置，替换（感觉和上面算一种）。
*   hook 绘制的代码，写入正确坐标。
*   不用虚拟机，自己接管流程，然后自己计算正确的坐标和加密的参数调用绘制函数。
*   不知道它代码坐标计算出错的原因，如果是逻辑错误可以直接修虚拟机，也能算一种。

脑子有限，只能想那么多了，希望有时间那个旧电脑退役了刷个系统再去实现这些操作把。

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

[#调试逆向](forum-4-1-1.htm) [#软件保护](forum-4-1-3.htm)