> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483847&idx=1&sn=786c050dbf588658423e6c026aed44dc&chksm=ce075082f970d99468aeba4f97f36e044c4ca6ab11ec715c875124dfb62f8f5f8020632e24dc&scene=21#wechat_redirect)

**一、安卓 10 中 ptrace 介绍**

   **1. 源码位置追踪**

      在 Android10 中, ptrace 函数定义文件路径为:

```
 bionic/libc/include/sys/ptrace.h

```

    函数原型定义如下:

```
long ptrace(int __request, ...);

```

        ptrace 函数实现文件路径为:

```
 bionic/libc/bionic/ptrace.cpp

```

    函数实现代码如下:

```
long ptrace(int req, ...) {
  bool is_peek = (req == PTRACE_PEEKUSR || req == PTRACE_PEEKTEXT || req == PTRACE_PEEKDATA);
  long peek_result;
  va_list args;
  va_start(args, req);
  pid_t pid = va_arg(args, pid_t);
  void* addr = va_arg(args, void*);
  void* data;
  if (is_peek) {
    data = &peek_result;
  } else {
    data = va_arg(args, void*);
  }
  va_end(args);
  //最终是调用__ptrace函数
  long result = __ptrace(req, pid, addr, data);
  if (is_peek && result == 0) {
    return peek_result;
  }
  return result;
}

```

    从 ptrace 实现代码可以看到最终调用的是__ptrace 函数。经过代码追踪__ptrace 函数是通过不同平台的汇编实现，最终调用__NR_ptrace 系统调用。

   Android 中 arm 平台__ptrace 实现代码文件路径:

```
bionic/libc/arch-arm/syscalls/__ptrace.S

```

实现代码如下:

```
#include <private/bionic_asm.h>
ENTRY(__ptrace)
    mov     ip, r7
    .cfi_register r7, ip
    ldr     r7, =__NR_ptrace
    swi     #0
    mov     r7, ip
    .cfi_restore r7
    cmn     r0, #(MAX_ERRNO + 1)
    bxls    lr
    neg     r0, r0
    b       __set_errno_internal
END(__ptrace)

```

      Android 中 arm64 平台__ptrace 实现代码文件路径:

```
 bionic/libc/arch-arm64/syscalls/__ptrace.S

```

      实现代码如下:

```
#include <private/bionic_asm.h>
ENTRY(__ptrace)
    mov     x8, __NR_ptrace
    svc     #0
    cmn     x0, #(MAX_ERRNO + 1)
    cneg    x0, x0, hi
    b.hi    __set_errno_internal
    ret
END(__ptrace)
.hidden __ptrace

```

_**2. 功能作用**_ 

     ptrace 可以让一个进程监视和控制另一个进程的执行, 并且修改被监视进程的内存、寄存器等, 主要应用于调试器的断点调试、系统调用跟踪等。比如安卓系统提供的 strace 工具，可以跟踪 app 所有执行的系统调用，如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433ovwHM5wLN5lEsExr9hgibn5Z6mXo9fKevCCXQHLAUxhmwrsUt5Pn4M6xk9tlZq6QJTexwGKRd8hQ/640?wx_fmt=png)

   在 Android app 保护中, ptrace 被广泛用于反调试。一个进程只能被 ptrace 一次, 如果先调用了 ptrace 方法, 那就可以防止别人调试我们的程序。这就是传说中的先占坑。比如在 JNI_onLoad 中调用以下代码来先占坑:

```
    if (ptrace(PTRACE_TRACEME, 0, 1, 0) < 0) {
        LOGD("I am ptraced");
        exit(0);
    }else{
        LOGD("ptrace success");
    }

```

   也可以使用多进程，ptrace 父进程来检测是否被调试，网上参考的代码，如下:  

```
static void anti_ptrace()
{
    pid_t child;
    child = fork();
    if (child)
    {
        //等待子进程退出
        wait(NULL);
    }
    else
    {
        // 获取父进程的pid
        pid_t parent = getppid();
        LOGD("ptrace attach start");
        // ptrace附加父进程
        if (ptrace(PTRACE_ATTACH, parent, 0, 0) < 0)
        {
            LOGD("ptrace attach failure");
            //attach父进程失败，说明被调试了,进入死循环,也可以考虑用杀进程的方式杀掉父进程
            //kill(parent,SIGKILL);
            while(1);
        }
        LOGD("ptrace attach success");
        //说明没有被调试，附加成功
        // 释放附加的进程
        ptrace(PTRACE_DETACH, parent, 0, 0);
        // 结束当前子进程
        exit(0);
    }
}

```

**二、源码修改**  

     针对以上的反调试手段，我们可以将源码中的 ptrace 逻辑改为如下:

```
long ptrace(int req, ...) {
  bool is_peek = (req == PTRACE_PEEKUSR || req == PTRACE_PEEKTEXT || req == PTRACE_PEEKDATA);
  long peek_result;
  va_list args;
  va_start(args, req);
  pid_t pid = va_arg(args, pid_t);
  void* addr = va_arg(args, void*);
  void* data;
  if (is_peek) {
    data = &peek_result;
  } else {
    data = va_arg(args, void*);
  }
  va_end(args);
  ///ADD START
  int caller_uid=getuid();
  //int caller_pid=getpid();
  //caller_uid>10000说明是普通App调用
  if(caller_uid>10000)
  {
     if(req ==PTRACE_TRACEME)
     {
        //自己ptrace自己,直接返回成功
        return 0;
     }
   if(req==PTRACE_ATTACH)
   {
       int caller_ppid=getppid();
     if(caller_ppid==pid)
     {
        //如果是子进程ptrace父进程，直接返回成功
        return 0;
     }
         //TODO 也可以通过读取/proc/$pid/status获取进程的uid，如果uid和当前调用ptrace的uid一致，也直接返回成功
   }
  }
  ///ADD END
  long result = __ptrace(req, pid, addr, data);
  if (is_peek && result == 0) {
    return peek_result;
  }
  return result;
}

```

   完成修改之后，执行如下命令编译刷机测试:

```
source build/envsetup.sh
breakfast oneplus3
brunch  oneplus3

```

**三、扩展**  

    由于 ptrace 函数容易被劫持, 很多 app 内部使用 ptrace 功能都不会直接调用 ptrace。而是内部使用 syscall 调用或者使用内联汇编实现调用 ptrace 的功能。因此，可以通过开发内核 hook ptrace 模块, 兼顾各种 ptrace 的调用情况。关于内核系统调用拦截的，可以参考该文章:https://www.anquanke.com/post/id/85375。

关注公众号，获取更多相关文章:

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRnyKCFcMFoicc7APewgUGMuS7BRMiaiaWFrFvjTuUFd4TG2oD2taRVaUBQ/640?wx_fmt=jpeg)