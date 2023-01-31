> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.iret.xyz](https://blog.iret.xyz/article.aspx/thread_pointer_aarch64)

> Investigate more, whatever it is.

Last updated:Nov.27, 2017 CST 22:33:51

Problem
-------

Sometimes when disassembling some AArch64 programs, we can find some interestring pseudocode:

```
v1 = _ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2));

```

And we can see those code in the disassemble window:

```
MRS             X19, #3, c13, c0, #2

```

What is `MRS`?
--------------

You can certainly skip this section if you really know AArch64.

Let's search it in the ARM v8 instruction set manual. The first thing we need to check is [6.5.2. system register access](http://infocenter.arm.com/help/topic/com.arm.doc.den0024a/ch06s05s02.html). It told us `MRS` can copy the value of a _system register_, which is "socalled SYSREG", into a GPR.

And in [chapter 4.3. System registers](http://infocenter.arm.com/help/topic/com.arm.doc.den0024a/BABGBFBF.html), we can find out here the `ARM64_SYSREG` is used to represent the name of a system register. In table 4.5, you can find a lot of those registers. However, only names of the register is listed, so we need to know how to translate arguments into the register name.

The manual told us that we need to read [Appendix J of the _ARM Architecture Reference Manual - ARMv8, for ARMv8-A architecture profile_](https://developer.arm.com/docs/ddi0487/latest/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile), which is a large file and you may want to save it just in case. However in _Table K12-5 Alphabetical index of AArch64 Registers_ at page K12-6427, those registers are still listed in name.

So let's go to the _C6.2.162_ in page _C6-781_, which described how the `MRS` is formatted:

```
System variant
  MRS <Xt>, (<systemreg>|S<op0>_<op1>_<Cn>_<Cm>_<op2>)

```

It also said the operand shoud have 6 insns, not 5 in IDA. So let's take this instruciton out and it seems like that:

```
|   D5   |   3B      |     D0  |    53   |
|11010101|00 1 11 011|1101 0000|010 10011|
|  OPCode   |L|o0|op1|CRn |CRm |op2| Rt  |
==>
op0 = 3
op1 = 3
op2 = 2
CRn = 13
CRm = 0
Rt  = 19

```

Which seems pretty related to the pseudocode. So now we can go to _Chapter D9 AArch64 System Register Encoding_ to decode it. After you have thoroughly read this section, you can know this instruction actually means "accessing non-debug system register `TPIDR_EL0` with `RW` access and save it to `X19`. The full name of `TPIDR_EL0` is "EL0 Read/Write Software Thread ID Register". According to the manual, this register is providing a location where EL0 software can store thread identifying information for management proposes. Sounds like a TLS register.

So how it worked
----------------

You may know something about system register now. Actually there is a tool on [Github](https://github.com/gdelugre/ida-arm-system-highlight) which is really helpful to get those info. However, it is still a problem if you want to know what excatly it represents in a running system.

TLS, stands for "Thread-Local Storage", is a way to share data between different threads in one program. We all know global vars can be shared in the whole program, however in a multi-thread program, each thread have its own private address space, thus unable to access other thread's memory. TLS is introduced to make some thing, like object ref counter, can be shared between different threads. This is done by allocate a memory space accessible by all threads, and give its pointer to each threads. Windows under x86 implemented it by using `PEB`, and Linux used `TPIDR_EL0`.

The initialization of TLS relies on both compiler and runtime library, which can be found in [this paper by G.O.Costa and A.Oliva](https://www.fsfla.org/~lxoliva/writeups/TLS/paper-lk2006.pdf). Those things can also be found in the source code. In `glibc/sysdeps/aarch64/nptl/tls.h:83` we can find a macro:

```
# define TLS_INIT_TP(tcbp) \
  ({ __asm __volatile ("msr tpidr_el0, %0" : : "r" (tcbp)); NULL; })

```

And in `glibc/csu/libc-tls.c:__libc_setup_tls(void):185`, which is called pretty early at `libc_start_main`, you can find the following initialization process:

```
  
#if TLS_TCB_AT_TP
  INSTALL_DTV ((char *) tlsblock + tcb_offset, _dl_static_dtv);

  const char *lossage = TLS_INIT_TP ((char *) tlsblock + tcb_offset);
#elif TLS_DTV_AT_TP
  INSTALL_DTV (tlsblock, _dl_static_dtv);
  const char *lossage = TLS_INIT_TP (tlsblock);
#else
# error "Either TLS_TCB_AT_TP or TLS_DTV_AT_TP must be defined"
#endif
  if (__builtin_expect (lossage != NULL, 0))
    _startup_fatal (lossage);

```

However, in real world, this job is finished by `ld.so`, the loader. You can find it in `glibc/elf/rtld.c`:

```
  bool was_tls_init_tp_called = tls_init_tp_called;
  if (tcbp == NULL)
    tcbp = init_tls ();
 
       for the TLS blocks has its final values and we can copy them
     into the main thread's TLS area, which we allocated above.
     Note: thread-local variables must only be accessed after completing
     the next step.  */
  _dl_allocate_tls_init (tcbp);

  
  if (! tls_init_tp_called)
    {
      const char *lossage = TLS_INIT_TP (tcbp);
      if (__glibc_unlikely (lossage != NULL))
    _dl_fatal_printf ("cannot set up thread-local storage: %s\n",
              lossage);
    }


```

The load procedure is nearly same, and pretty simple, so I think you might want to read it yourself. Generally, it will:

*   Check if `.tls` segment exists;
*   Set up the TCB block using `sbrk` (why not malloc? read code and you will know)
*   Initialize the `dtv` data and install its pointer
*   Write the pointer using `TLS_INIT_TP`
*   Update the linkmap
*   Load other TLS vars (actually, static TLS vars)

You can also read [this](http://codemacro.com/2014/10/07/pthread-tls-bug/) article to know more about TLS.

And what it is accessing?
-------------------------

So far we have know what is what is thread register and TLS. However, how they are associated with current programs?

Let's see some code related with them. Following pseudocode also came from the same program as above:

```
  v1 = _ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2));
  
  v3 = *(_QWORD *)(v1 + 40);
  v4 = *(_QWORD *)(*(_QWORD *)(v1 + 72) + 1568LL);
  *(_QWORD *)v3 = *(_QWORD *)(*(_QWORD *)(v1 + 72) + 1560LL);
  

```

Here although we know `v1` is related with TLS, we still don't know what is `v3` and `v4`. Let's check `glibc/csu/libc-tls.c:__libc_setup_tls(void):168` [1](#fn:1) :

```
  
#if TLS_TCB_AT_TP
  _dl_static_dtv[2].pointer.val = ((char *) tlsblock + tcb_offset
                   - roundup (memsz, align ?: 1));
  main_map->l_tls_offset = roundup (memsz, align ?: 1);
#elif TLS_DTV_AT_TP
  _dl_static_dtv[2].pointer.val = (char *) tlsblock + tcb_offset;
  main_map->l_tls_offset = tcb_offset;
#else
# error "Either TLS_TCB_AT_TP or TLS_DTV_AT_TP must be defined"
#endif
  _dl_static_dtv[2].pointer.to_free = NULL;
  
  memcpy (_dl_static_dtv[2].pointer.val, initimage, filesz);

```

So you can notice something called `initimage` is copied into the new memory space we have allocated for TLS. And in line `117`:

```
if (_dl_phdr != NULL)
  for (phdr = _dl_phdr; phdr < &_dl_phdr[_dl_phnum]; ++phdr)
    if (phdr->p_type == PT_TLS)
  {
    
    memsz = phdr->p_memsz;
    filesz = phdr->p_filesz;
    initimage = (void *) phdr->p_vaddr;
    
    break;
  }

```

So it seems the `memcpy` will simply copy all data in `.tls` segment to our memory area. And let's find out what is `v1+40` in IDA. First we need to know what is in the thread pointer register, `DTV` or `TCB`. Check `glibc/sysdeps/aarch64/nptl/tls.h:40`:

```
# define TLS_DTV_AT_TP 1
# define TLS_TCB_AT_TP 0

```

So it is `DTV`. According to the code above, we can know after initalized, the value of `tpidr_el0+16` is equal to `tlsblock`, and the content of it is _nearly_ identical[2](#fn:2) from `.tls` segment in ELF image. So we can go directly to that address using IDAPython:

```
print( "Type of v3 is %s"%(ida_name.demangle_name(ida_name.get_ea_name(ida_segment.get_segm_by_name(".tls").start_ea+40-16), 0)))


```

Now everything seems fine.