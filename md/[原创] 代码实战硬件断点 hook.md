> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271216.htm)

> [原创] 代码实战硬件断点 hook

见字如面，我是东北码农。

 

上文我们介绍了硬件断点，大家可以回顾一下。本文将介绍使用硬件断点 + veh，实现硬件断点 hook。

 

关注后，聊天框回复 “硬件断点 hook”，可以获取本文源码。

[](#1、不修改原函数hook)1、不修改原函数 hook
------------------------------

硬件断点这个手法，我还是分析外挂时学到的。开始做反外挂时候，只会用 xuetr、pchunter 等工具查看游戏进程的 inlinehook，分析修改了游戏的哪些代码，然后再防御。然后有一天，我发现了一个外挂不按套路出牌，没有修改任何代码实现了透视功能，我就很纳闷这个外挂是怎么实现的呢，不改代码我怎么防啊？

 

后来经过不断学习，才知道 hook 的方式有很多种，硬件断点 hook 就是其中一种比较隐蔽的 hook 方式，可以在不修改源函数的前提实现 hook，这一点比 inlinehook 强。

2、VEH（Vectored Exception Handling）
----------------------------------

### 2.1、VEH 介绍

我们先来介绍一下 VEH，先放一段微软官方描述。

 

参考资料：https://docs.microsoft.com/en-us/windows/win32/debug/vectored-exception-handling

 

<font color=gray size=2>  
Vectored exception handlers are an extension to structured exception handling. An application can register a function to watch or handle all exceptions for the application. Vectored handlers are not frame-based, therefore, you can add a handler that will be called regardless of where you are in a call frame. Vectored handlers are called in the order that they were added, after the debugger gets a first chance notification, but before the system begins unwinding the stack.  
</font>

 

大概意思是，程序可以注册异常回调函数链表。当程序触发异常时，回调函数就会按照用户指定的顺序依次调用。

### 2.2、VEH 使用

向 VEH 链注册一个异常处理函数，可以使用 windows 提供的 API，

```
PVOID AddVectoredContinueHandler(
  ULONG                       First,
  PVECTORED_EXCEPTION_HANDLER Handler
);

```

*   First：是否插入 VEH 链头部。
*   Handler：异常处理函数。

接下来看一下异常处理函数，PVECTORED_EXCEPTION_HANDLER 的定义：

```
LONG PvectoredExceptionHandler(
  [in] _EXCEPTION_POINTERS *ExceptionInfo
)

```

唯一的参数_EXCEPTION_POINTERS 指向异常信息，定义如下：

 

定义如下：

```
typedef struct _EXCEPTION_POINTERS {
  PEXCEPTION_RECORD ExceptionRecord;// 异常信息记录
  PCONTEXT          ContextRecord;// 寄存器信息
} EXCEPTION_POINTERS, *PEXCEPTION_POINTERS;

```

异常处理函数的返回值很重要，控制着异常处理的后续行为：

 

<font color=gray size=2>  
To return control to the point at which the exception occurred, return EXCEPTION_CONTINUE_EXECUTION (0xffffffff). To continue the handler search, return EXCEPTION_CONTINUE_SEARCH (0x0).  
</font>

*   返回 - 1：异常已处理，继续执行；
    
*   返回 0：继续调用 VEH 链的其它处理函数。
    

3、硬件断点 hook 实战代码
----------------

介绍完硬件断点和 veh 后，我们就可以实现硬件断点 hook 了。我们先介绍一下思路：

1.  设置 VEH 链异常处理函数。
    
2.  设置硬件断点，监控原函数的执行事件。
    
3.  执行原函数时，会触发异常调用我们的异常处理函数。
    
4.  异常处理函数判断，如果是原函数地址，则修改 rip 寄存器，跳转到 hook 函数。
    

### 3.1、设置 VEH 链异常处理函数

异常处理函数，xx_hw_bp_veh 的实现一会再介绍。

```
static bool xx_init_hwbp_hook() {
  return xx_add_veh(CALL_FIRST, xx_hw_bp_veh);
}

```

### 3.2、设置硬件断点

```
class xx_hw_bp
{
public:
  struct bp_info {
    void* src_ = nullptr;
    void* dst_ = nullptr;
  };
public:
  void hook(void* src,void* dst,HANDLE thread,int idx)
{
    //记录原函数与hook函数关系。
    // TODO idx check
    info_[idx].src_ = src;
    info_[idx].dst_ = dst;
    // 设置硬件断点
    xx_set_hw_bp(thread, idx, src, RW_EXE);
  }
    // 获取原函数与hook函数映射
  void* get_dst(void* src) {
    for (int i = 0; i < 4; ++i) {
      if (src == info_[i].src_)
      {
        return info_[i].dst_;
      }
    }
    return nullptr;
  }
  bp_info info_[4];
};
 
xx_hw_bp g_hwbp_;// 全局类，方便异常处理函数访问

```

先记录一下原函数和 hook 函数的映射关系，再使用上文硬件断点封装好的 xx_set_hw_bp 设置硬件断点。

### 3.3、异常处理函数实现

```
LONG NTAPI xx_hw_bp_veh(
  struct _EXCEPTION_POINTERS* ExceptionInfo)
{
  // 获取异常地址
  void* src = ExceptionInfo->ExceptionRecord->ExceptionAddress;
  // 查找对应关系
  void* dst = g_hwbp_.get_dst(src);
  if (nullptr != dst) {
    // 修改rip 跳转
    memcpy(&ExceptionInfo->ContextRecord->Rip, &dst, sizeof(void*));
    return EXCEPTION_CONTINUE_EXECUTION;
  }
  // 其它异常，继续veh链处理
  return EXCEPTION_CONTINUE_SEARCH;
}

```

先通过异常地址，找到对应的 hook 函数地址。若找到，则修改 Rip（指令寄存器）实现跳转；找不到则不是我们关心的异常，交给 VEH 链继续处理吧。

### [](#4、demo验证)4、demo 验证

```
int WINAPI My_MessageBoxA(
  _In_opt_ HWND hWnd,
  _In_opt_ LPCSTR lpText,
  _In_opt_ LPCSTR lpCaption,
  _In_ UINT uType)
{
  printf("[call %s]!\n", __FUNCTION__);
 
  auto ori = xx_trampoline_to_func(&MessageBoxA, trampoline);
  return (*ori)(hWnd, lpText, lpCaption, uType);
}
 
void test_hook() {
  // 制作跳板，只需要复制1条指令，跳过硬件断点即可
  xx_mem_unprotect(trampoline, 1024);
  xx_make_trampoline(&MessageBoxA, trampoline, 4);
 
  xx_init_hwbp_hook();
  g_hwbp_.hook(&MessageBoxA, &My_MessageBoxA,GetCurrentThread(), 0);
  ::MessageBoxA(0, "aaa", "bbb", 0);
}

```

使用时先制作跳板，再 hook。制作跳板复用 inlinehook 的代码。由于不修改代码，所以跳板值需要复制一条指令即可。复制的代码少，所以也基本不用考虑重定位问题。

 

欢迎大家点赞、转发、再看、留言交流~

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#系统底层](forum-4-1-2.htm)