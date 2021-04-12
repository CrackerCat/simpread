> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266999.htm)

unidbg 多线程分析
============

目录

*   [unidbg 多线程分析](#unidbg多线程分析)
*            一. 概述
*            二. 准备
*            三. 开始分析
*                    3.1 unidbgMutil 的多线程创建分析
*                    3.2 Android 多线程分析
*            四. 问题
*                    4.1 测试
*                    4.2 增加功能
*                            4.2.1 Futex 概述
*                            4.2.2 unidbg futex 修改
*            五. 总结

一. 概述
-----

由于在工作中遇到了某翻译 so 中有多线程调用，因此使用 unidbg 分析（基于 [unidbgMutilThread](https://github.com/asmjmp0/unidbgMutilThread)）并增加阻塞唤醒机制（futex 系统调用），但仍未调用成功，因此本文概述对 unidbg 多线程的理解、android 多线程的创建流程、实现简单的阻塞唤醒、以及近段时间分析的总结，也希望大神网友能提出宝贵意见及分析方向，文末会有相关内容。

二. 准备
-----

android6.0(sdk23) ，kernel 源码  
[https://github.com/zhkl0228/unidbg](https://github.com/zhkl0228/unidbg)  
[https://github.com/asmjmp0/unidbgMutilThread](https://github.com/asmjmp0/unidbgMutilThread)

 

相关源码路径

```
/bionic/libc/bionic/pthread_create.cpp
/bionic/libc/bionic/pthread_mutex.cpp
/bionic/libc/bionic/pthread_cond.cpp
 
/bionic/libc/bionic/clone.cpp
/bionic/libc/arch-arm/bionic/__bionic_clone.S
 
/bionic/libc/private/bionic_futex.h
/kernel/kernel/futex.c

```

三. 开始分析
-------

### 3.1 unidbgMutil 的多线程创建分析

我们知道，在 C 中创建一个线程是要用到 pthread_create 这个函数的，这个函数简单来说，在用户空间通过 mmap 为子线程分配线程栈空间，在底层的是使用了 clone 这个系统调用创建线程。

 

因此 unidbgMutil 也选择在 clone 这个系统调用里面实现自己的线程创建。

```
//com.github.unidbg.linux.ARM32SyscallHandler 
private int pthread_clone(Backend backend, Emulator emulator) {
        . . . . . .
        Pointer child_stack = UnidbgPointer.register(emulator, ArmConst.UC_ARM_REG_R1);
 
        Pointer fn = child_stack.getPointer(0);
        child_stack = child_stack.share(4);
        Pointer arg = child_stack.getPointer(0);
        child_stack = child_stack.share(4);
 
        threadId = ++ThreadDispatcher.thread_count_index;
 
        emulator.getThreadDispatcher().threadMap.put(threadId, new LinuxThread(emulator,child_stack, fn, arg));
        . . . . . .
}

```

这里可以看到，在 clone 的系统调用里，我们取出了 R1 寄存器的值，然后又通过 R1 取得了 fn、arg，接着创建一个 LinuxThread 对象，并把当前线程 id 和这个对象绑定在一起，存入全局的 threadMap 中。然后在 LinuxThread 里保存当前 cpu 上下文，保存线程栈，通过 arg.getPointer(48) 获取子线程函数的地址。通过 this.arg.getPointer(52) 获取子线程参数的地址。

 

![](https://bbs.pediy.com/upload/attach/202104/804803_3UYQ4K55YKFARBY.png)

 

其实到这里，我们需要分析一下，child_stack 的连续取地址，arg 的 pointer 48,52 的偏移究竟是什么，不然我们后续增加功能，修改代码，就会一头雾水。

### 3.2 Android 多线程分析

前边简单概述了 pthread_create 的相关内容，但如果要了解 unidbg 的多线程实现，我们则要详细分析 Android 是如何创建多线程的。我们看代码  
![](https://bbs.pediy.com/upload/attach/202104/804803_JCFSUPMK7B7R7ZH.png)  
我们知道 pthread_create 一共有 4 个参数，这里要关注第三和第四个参数，也就是子线程函数的地址和参数。代码块 1 调用了__allocate_thread 函数，传入 thread 变量（pthread_internal_t 结构体，很重要），和 child_stack 指针。

 

![](https://bbs.pediy.com/upload/attach/202104/804803_FE5YZ9NZS25GX4X.png)

 

进入后我们发现，这个函数的作用其实就是为我们的子线程，开启一份栈空间，attr->guard_size 是线程栈的保护区域这里是 4k，__create_thread_mapped_space 函数内部通过 mmap 系统调用，分配出一份匿名、私有的空间供子线程使用。然后将分配的内存大小，栈顶地址，赋值给 threadp 即 pthread_internal_t。

 

![](https://bbs.pediy.com/upload/attach/202104/804803_85AUH793W26K3US.png)

 

到这里我们的栈空间已经分配完成，接下来就要进行子线程函数地址和参数的分配。也就是我们看到的在 pthread_create 代码块 2 那里，将 start_routine 和 arg 全都赋值给 thread 这个变量。然后就调用到 clone 这个函数。  
clone

```
int clone( int (*fn)(void *),
            void *child_stack,
            int flags,
            void *arg,
            .... /* pid_t *ptid, struct user_desc *tls, pid_t *ctid */ );

```

通过查阅资料，linux 中进程和线程的创建在内核中都是通过 clone 系统调用完成的，区别在于 flags 参数，因为线程是可以共享进程中的资源的，而进程和进程之间是隔离的，就是因为在 clone 系统调用中，flags 参数的作用，如 CLONE_VM，CLONE_FS，CLONE_SIGHAND 等。也就是说线程创建的本质是共享进程的虚拟内存、文件系统属性、打开的文件列表、信号处理，以及将生成的线程加入父进程所属的线程组中等等。这里 flags 参数在 pthread_create 内部已经写好，我们这里只需要关注 fn，child_stack 和 arg 就可以了。

 

fn 表示 clone 生成的子进程 / 线程会调用 fn 指定的函数，我们发现这里的 fn，并不是 pthread_create 中传进来的子线程函数（start_routine），而是 pthread_create 内部的函数__pthread_start，而这个函数的参数必然不可能是子线程函数的参数，我们看一下，他的参数是 thread 变量（pthrea_internel_t），在我们前面的分析中，我们知道子线程的函数地址和函数参数就在这个 thread 变量中！  
![](https://bbs.pediy.com/upload/attach/202104/804803_XK4655QA2URG7Z5.png)  
接着往下走，进入 clone 函数  
![](https://bbs.pediy.com/upload/attach/202104/804803_QJ7JHC59DRY3AJR.png)

 

到这里，我们进入了_bionic_clone 这个函数，这个函数在 libc 中是用汇编写的，这里我们要注意下，_bionic_clone 的参数和 clone 的参数位置，因为接下来我们要分析寄存器里的内容，如果参数搞混了就头疼了。这里我们记住，fn 虽然是 clone 要调用的子线程函数，但是我们真正的子线程函数在 arg（thread）里。即 fn -> __pthread_start，arg -> thread(子线程函数，参数)，child_stack 是 mmap 分配的，不用多说。  
![](https://bbs.pediy.com/upload/attach/202104/804803_627DNNUQBB7WB4Y.png)

 

进入__bionic_clone 这个汇编，他有 7 个参数，我们知道 arm 函数调用的参数传递，少于 4 个参数由 R0-R3 完成，多于 4 个参数用栈（sp）传递，并且入栈的方式是从右向左入栈。这个代码以及注释已经写得很清楚了，首先保存 sp 栈指针的值 mov ip, sp；然后将 R4-R7 入栈。linux 的栈是高地址向低地址压的，而且 arm 规定 sp 指向栈顶位置，因此下面两条指令的含义是存储原始的 R4-R7 寄存器的值，即将 R4-R7 入主线程的栈中，然后将 ip 中的值，也就是原始 sp 栈中的参数 tid，fn，arg，加载到 R4-R6 寄存器中。具体的 stmfd，ldmfd，stmdb 指令，可以查看相关资料，我画了一个图应该更容易理解这几条指令。

 

![](https://bbs.pediy.com/upload/attach/202104/804803_4NADC92JFJWMN8E.png)

 

接下来的指令 stmdb r1!, {r5, r6}，很重要，这条指令是理解 unidbg 中对 child_stack 的指令偏移的关键。stmdb 的含义是，地址先减然后完成操作，因此 r1 寄存器的地址先减 4（减 4 是因为 32 位）然后存入 r6，再减 4，存入 r5。根据上边的指令，r6 里边存的是 arg 参数，r5 里边存放的是 fn 指针。

 

![](https://bbs.pediy.com/upload/attach/202104/804803_8BCBVAR9HFV4EGM.png)

 

接下来的指令 ldr r7, =__NR_clone；swi #0；则是通过 R7 传递系统调用号，swi 软中断（现在是 svc 指令，功能相同）从用户空间（libc）真正进入到内核空间，之后的操作则是在内核态由 kernel 操作（位置在 / kernel/kernel/fork.c -> SYSCALL_DEFINE5 -> do_fork 完成，这里不是我们的重点），在 unidbg 里则是直接进入了 ARM32SyscallHandler 中的 hook 方法。

 

现在我们再来看一下 child_stack 的操作  
![](https://bbs.pediy.com/upload/attach/202104/804803_QNZ7FVJ49ZCBGTQ.png)

 

首先获取 R1 寄存器的值（记得我们已经在 "内核态" 了），通过上边的分析，我们已经非常清楚了，此时 R1 里的值就是 fn，这个 fn 就是__pthread_start，child_stack.share(4); 相当于 R1 地址加 4，getPointer(0) 就是获取当前地址里的值，即 arg，还记得这个 arg 实际上是一个 pthread_internel_t 的结构体，里面有我们子线程的函数地址和参数。

 

那么`this.fn = (UnidbgPointer) arg.getPointer(48);`和`UnidbgPointer this_arg=((UnidbgPointer) this.arg).getPointer(52);`猜想也能够知道，就是 pthread_internel_t 的结构体里的子线程函数和参数，我们这里验证一下 pthread_internel_t 所占的内存大小，由于类 class（结构体 struct）中定义的成员函数和构造和析构函数不占整体的空间 [[注](https://blog.csdn.net/Netown_Ethereal/article/details/38898003)]。因此可以计算，next，prev，cleanup_stack（指针类型占 4 字节），tid(int 类型占 4 字节)，join_state(枚举类型占 4 字节)，即 5 * 4 = 20 个字节  
![](https://bbs.pediy.com/upload/attach/202104/804803_JM4PH35A2F6FGEK.png)

 

其中 attr 为结构体，里面是 int 和指针类型，占 4 * 6=24 个字节，不过按照我这里的计算方式为 44 个字节偏移，少了 4 个字节，可能是计算 join_state 占用空间不对，或者在哪块有内存对齐，有大神知道的话可以指导一下。  
![](https://bbs.pediy.com/upload/attach/202104/804803_Y9SVFVZX845G53C.png)

 

不过最终，start_routine 所在的偏移是 48 个字节是没毛病的，start_routine_arg 所占的字节自然是 48+4=52 的位置。  
到此，我们已经完整的分析了 unidbgMutil 的多线程创建机制，接下来将实现阻塞唤醒功能，以及提出我遇到的问题。

四. 问题
-----

当我在调用这个翻译的 so 时，配置好环境后，用 unidbg 调用，在单线程的时候，有些是可以成功的。调用这个 so 分两步，1. 加载模型。2. 翻译  
![](https://bbs.pediy.com/upload/attach/202104/804803_9MYJ7AZVC9G9Q7N.png)  
但问题是大部分要传入翻译的字段，在 unidbg 里会陷入一个死循环，在系统调用号 240 的位置（futex），于是在大致看看 so 之后，发现这个 so 是使用多线程的，其中导入函数里面有很多关于线程同步的东西，锁，信号量，条件变量等。于是我准备在 unidbg 的基础上实现同步机制。

### 4.1 测试

首先写了一个 demo，例子很简单，就是创建 3 个线程，在子线程里进行加锁，并用条件变量控制。主线程里是一个死循环，只有子线程操作完毕后，主线程才会退出循环，输出完成的 log。（测试用例的位置在 unidbg-android/src/main/java/thread/Test）

 

![](https://bbs.pediy.com/upload/attach/202104/804803_YCHK824B27N378C.png)

 

![](https://bbs.pediy.com/upload/attach/202104/804803_7XNW6BUXMJNRFZT.png)

 

![](https://bbs.pediy.com/upload/attach/202104/804803_8R3FU36PDVEUQWC.png)

### 4.2 增加功能

在这个测试例子中，我们使用到了锁 (pthread_mutex_lock)，条件变量(pthread_cond_wait/signal) 对线程进行同步控制，而这些函数的底层机制都是使用到了 futex 这个系统调用，因此要了解一下 linux futex 机制。

#### 4.2.1 Futex 概述

关于 futex 系统调用，网上资料很多，简单来说，在 android 里可以实现进程 / 线程间阻塞唤醒功能。他的参数有很多，最主要的是前三个参数，第二个参数 futex_op 在 android 里只有两个选项，FUTEX_WAIT，FUTEX_WAKE 即阻塞和唤醒。

```
int futex ( int *uaddr,  int futex_op,  int val,         
    const struct timespec *timeout,   /* or: uint32_t val2 */         
    int *uaddr2, int val3);

```

第一个参数 uaddr 是一个地址，地址里边是一个 int 的值，一般被称为 futex 字，或者 futex 变量。这个值一般是由用户空间定义，比如 pthread_mutex_lock 函数在使用 futex 时，futex 字就是 & mutex->state 这个值。

 

他的作用是当 futex_op 的类型为 FUTEX_WAIT 时，会比较 futex 字和第三个参数 val 的大小，如果相同表示要进入阻塞（不相等则失败）。当 futex_op 的类型为 FUTEX_WAKE 时，第三个参数 val 的值，代表要唤醒阻塞着的进程 / 线程数，比如使用 pthread_cond_broadcast 时，val 为 INT_MAX，即唤醒所有线程。  
![](https://bbs.pediy.com/upload/attach/202104/804803_QFGBPMGYMB2BJKV.png)

#### 4.2.2 unidbg futex 修改

知道了 futex 的原理，我们自己实现阻塞唤醒也就有了思路，由于实现多线程的方式是基于指令的时间片  
![](https://bbs.pediy.com/upload/attach/202104/804803_4TPS3DZX66DVEA5.png)  
因此，阻塞对于我们来讲，也就是在一个线程被阻塞后，unidbg 切换线程时，不要切换到这个阻塞线程。唤醒就是可以重新切换到这个阻塞的线程。

 

因此我这里实现的方式比较简单，在 futex_wait 里，将 futex uaddr 和当前线程 id 关联起来，然后将当前线程 id 添加进阻塞线程。  
![](https://bbs.pediy.com/upload/attach/202104/804803_J2KMXBRZ8PYVCH2.png)  
唤醒的方式，同样简单粗暴，移除阻塞在 uaddr 上的任意一个线程即可。  
![](https://bbs.pediy.com/upload/attach/202104/804803_XQR9P5HC8FRWWQW.png)  
然后，每当调用到 futex 阻塞和唤醒后，切换线程。

 

之前我切换线程时，直接在 futex 里进行切换，后来导致 unicorn 数据错乱，一直报 Invalid memory read (UC_ERR_READ_UNMAPPED) 错误，这个错误是 unicorn 在 emu_start 里，如果某条指令出现问题，则会抛出异常，但是并不会告诉你是哪条指令。  
幸运的是 unidbg 提供了 tracecode 的功能，于是经过多次调试后最终发现，在切换完线程进行保存 / 恢复寄存器上下文后，R0 寄存器的值总是为 0，这个奇怪的现象联想到，这正是 futex 的返回值。系统调用返回后，会修改 R0 寄存器的值，进而导致了数据错乱。接着我们把切换线程的代码放到系统调用返回之后就 OK 了。  
![](https://bbs.pediy.com/upload/attach/202104/804803_4CDCH56Q6G85Q3K.png)  
然后，我们的阻塞唤醒已经基本完成了（pthread_exit 里有锁会调用 futex，会出现问题，不过线程已经退出了这个问题就没有再研究）。

五. 总结
-----

到这里，本文也快结束了，其实本文看似是个分析贴，实则是一个求助帖，因为最后我仍然没有把翻译 so 调用成功。所以回过头来，想了想近段时间一直在研究 unidbg 而减少了对翻译 so 本身的研究，而对翻译 so 的分析本身也充满了挑战。

 

所以请教各位网友，也想和大家交流一下，我们的目标是用 unidbg 成功调用 so，并不需要还原 so 的算法，如何更好的去分析多线程的 so，然后用 unidbg 模拟出来，目前我的思路可能就是看出错堆栈，然后 frida 去 hook 原始 so，比较跟 unicorn 调用的不同？

 

这个翻译 so 在加载模型阶段，会开启 4 个线程，如果只单线程模式调用（只运行主线程），模型的加载可以成功，但后续的翻译阶段有的会陷入死循环。使用多线程加载时，加载模型阶段失败。希望有厉害的网友可以帮忙看一看。

 

代码已经上传到 git 上，地址是 [https://github.com/SliverBullet5563/unidbg_test](https://github.com/SliverBullet5563/unidbg_test) 基于 unidbg0.9.4 以 [https://github.com/cxapython/unidbgok](https://github.com/cxapython/unidbgok) 的构建方式实现。

 

最后，虽然没有成功调用，但是对 unidbg 的理解又加深了一些，大致如下。  
unidbg 的内存布局:

```
[0xffffffffL-0xffff0000L] ： svc #0  0xffff0fa0: bx lr
 
[0xffff0000L-0xfffe0000L]: ARMSvcMemory jni引用
 
[0xc0000000L-0xbff00000L] :  栈空间
 
[xxx - 0x40000000L] :  so起始地址

```

*   打断点：emulator.attach().addBreakPoint(address);
*   任意位置调试: emulator.attach().debug();
*   任意位置打印调用栈：emulator.getUnwinder().unwind();
*   tracecode: emulator.traceCode(begin,end);
*   patchcode: emulator.getMemory().pointer(address).setInt(patchCode); // nop 0xbf00bf00;
*   获取 modules：emulator.getMemory().getLoadedModules()。
*   继承 IOResolver 接口，在 resolve 函数里可以监控 open 系统调用。
*   实现 VirtualModule 子类，注册 register 方法，可以实现 "虚拟"so 的加载。
*   使用 vm.setDvmClassFactory(new ProxyClassFactory());ProxyDvmObject.createObject(vm,value); 通过反射可以直接使用 java 里的类。

等等。

 

最后，谢谢！

 

参考：

 

[https://github.com/zhkl0228/unidbg](https://github.com/zhkl0228/unidbg)  
[https://github.com/asmjmp0/unidbgMutilThread](https://github.com/asmjmp0/unidbgMutilThread)  
[https://github.com/cxapython/unidbgok](https://github.com/cxapython/unidbgok)

 

[https://juejin.cn/post/6844904197335285774](https://juejin.cn/post/6844904197335285774)  
[https://blog.csdn.net/Netown_Ethereal/article/details/38898003](https://blog.csdn.net/Netown_Ethereal/article/details/38898003)  
[https://blog.csdn.net/baidu_37973494/article/details/86766342](https://blog.csdn.net/baidu_37973494/article/details/86766342)  
[https://www.twblogs.net/a/5b84705b2b71775d1cd0bec8/?lang=zh-cn](https://www.twblogs.net/a/5b84705b2b71775d1cd0bec8/?lang=zh-cn)  
[https://root1iu.github.io/2019/01/07/%E5%88%9D%E8%AF%86futex/](https://root1iu.github.io/2019/01/07/%E5%88%9D%E8%AF%86futex/)

[安卓应用层抓包通杀脚本发布！《高研班》2021 年 6 月班开始招生！](https://bbs.pediy.com/thread-264283.htm)