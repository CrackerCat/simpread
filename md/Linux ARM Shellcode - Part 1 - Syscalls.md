> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jumpnowtek.com](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part1.html)

Jumping right into a standard **Hello world** program.

```
.text
.global _start

_start:
    mov r0, #1              @ stdout
    add r1, pc, #16         @ address of the string
    mov r2, #12             @ string length
    mov r7, #4              @ syscall for 'write'
    swi #0                  @ software interrupt

_exit:
    mov r7, #1              @ syscall for 'exit'
    swi #0                  @ software interrupt

_string:
.asciz "Hello world\n"          @ our string, NULL terminated


```

Use the GNU [as(1)](http://man7.org/linux/man-pages/man1/as.1.html) assembler and [ld(1)](http://man7.org/linux/man-pages/man1/ld.1.html) linker to generate a Linux executable

```
$ as hello.s -o hello.o
$ ld hello.o -o hello


```

We can then run the program

Now a breakdown.

Linux ARM/EABI syscalls are invoked using a software interrupt.

The function arguments go in registers **R0-R6**, the syscall number in register **R7**.

This information can be found in [syscall(2)](http://man7.org/linux/man-pages/man2/syscall.2.html).

A few tables from that man page

```
The first table lists the instruction used to transition to kernel mode,...

arch/ABI   instruction          syscall #   retval Notes
   ───────────────────────────────────────────────────────────────────
   arm/OABI   swi NR               -           a1     NR is syscall #
   arm/EABI   swi 0x0              r7          r0
   arm64      svc #0               x8          x0
   blackfin   excpt 0x0            P0          R0
   i386       int $0x80            eax         eax
   ia64       break 0x100000       r15         r8     See below
   mips       syscall              v0          v0     See below
   parisc     ble 0x100(%sr2, %r0) r20         r28
   s390       svc 0                r1          r2     See below
   s390x      svc 0                r1          r2     See below
   sparc/32   t 0x10               g1          o0
   sparc/64   t 0x6d               g1          o0
   x86_64     syscall              rax         rax    See below
   x32        syscall              rax         rax    See below

The second table shows the registers used to pass the system call arguments.

   arch/ABI      arg1  arg2  arg3  arg4  arg5  arg6  arg7  Notes
   ──────────────────────────────────────────────────────────────────
   arm/OABI      a1    a2    a3    a4    v1    v2    v3
   arm/EABI      r0    r1    r2    r3    r4    r5    r6
   arm64         x0    x1    x2    x3    x4    x5    -
   blackfin      R0    R1    R2    R3    R4    R5    -
   i386          ebx   ecx   edx   esi   edi   ebp   -
   ia64          out0  out1  out2  out3  out4  out5  -
   mips/o32      a0    a1    a2    a3    -     -     -     See below
   mips/n32,64   a0    a1    a2    a3    a4    a5    -
   parisc        r26   r25   r24   r23   r22   r21   -
   s390          r2    r3    r4    r5    r6    r7    -
   s390x         r2    r3    r4    r5    r6    r7    -
   sparc/32      o0    o1    o2    o3    o4    o5    -
   sparc/64      o0    o1    o2    o3    o4    o5    -
   x86_64        rdi   rsi   rdx   r10   r8    r9    -
   x32           rdi   rsi   rdx   r10   r8    r9    -


```

Unless noted, all of the examples will assume **arm/EABI** systems.

There are a couple of exceptions to that first table from the [syscall(2)](http://man7.org/linux/man-pages/man2/syscall.2.html) man page.

The arm instruction **swi** is deprecated and gets converted to [svc](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0489c/Cihidabi.html) during assembly. I’ll use **svc** from now on.

The argument to **svc** does not have to be zero. The cpu does not use this value. Software has the option to use the argument in the interrupt handler, but Linux does not. Linux syscalls rely only on the arguments passed in registers. That means we can use any number as the **svc** argument (subject to some max ranges) and for reasons explained later we will use a non-zero value like **#1** from now on.

Syscall numbers for the system can be found in the **unistd-common.h** header file.

```
# cat /usr/include/asm/unistd-common.h
#ifndef _ASM_ARM_UNISTD_COMMON_H
#define _ASM_ARM_UNISTD_COMMON_H 1

#define __NR_restart_syscall (__NR_SYSCALL_BASE + 0)
#define __NR_exit (__NR_SYSCALL_BASE + 1)
#define __NR_fork (__NR_SYSCALL_BASE + 2)
#define __NR_read (__NR_SYSCALL_BASE + 3)
#define __NR_write (__NR_SYSCALL_BASE + 4)
#define __NR_open (__NR_SYSCALL_BASE + 5)
#define __NR_close (__NR_SYSCALL_BASE + 6)
...


```

That is where the #4 and #1 constant values for these instructions came from

```
mov r7, #4              @ syscall for 'write'
mov r7, #1              @ syscall for 'exit'


```

The write system call takes 3 arguments: [write(2)](http://man7.org/linux/man-pages/man2/write.2.html).

```
ssize_t write(int fd, const void *buf, size_t count);


```

So when we invoke the write syscall, the kernel expects to find arguments in registers **r0**, **r1** and **r2**.

Standard POSIX, **stdout** is file descriptor 1.

The calculation of the buffer address is a little confusing at first

```
add r1, pc, #16         @ address of the string


```

This instruction adds 16 to the current program counter **pc** into register **r1**.

If we take a look at the object code for the program using the [objdump(1)](http://man7.org/linux/man-pages/man1/objdump.1.html) utility

```
# objdump -d hello.o

hello.o:     file format elf32-littlearm


Disassembly of section .text:

00000000 <_start>:
   0:   e3a00001        mov     r0, #1
   4:   e28f1010        add     r1, pc, #16
   8:   e3a0200c        mov     r2, #12
   c:   e3a07004        mov     r7, #4
  10:   ef000000        svc     0x00000000

00000014 <_exit>:
  14:   e3a07001        mov     r7, #1
  18:   ef000000        svc     0x00000000

0000001c <_string>:
  1c:   6c6c6548        .word   0x6c6c6548
  20:   6f77206f        .word   0x6f77206f
  24:   0a646c72        .word   0x0a646c72
  28:   00              .byte   0x00
  29:   00              .byte   0x00
        ...


```

We note that the memory values at the start of each line differ by 4 bytes.

That’s because each arm 32 instruction is 4 bytes.

There are arm instruction sets like **thumb** where the instruction size is different, but we are just using the standard arm instructions for now.

Looking at these two instructions

```
   4:   e28f1010        add     r1, pc, #16
  1c:   6c6c6548        .word   0x6c6c6548


```

We can see that the start of our string is at memory location **0x1c** (think little-endian).

```
1c:   6c6c6548 => 'lleH'
20:   6f77206f => 'ow o'
24:   0a646c72 => '\ndlr'


```

Then we would expect the offset calculation from the current program counter **pc** to be

```
0x1c - 0x04 = 0x18 = 24 bytes


```

So why isn’t the instruction

The reason has to do with the way the arm instruction pipeline works.

The **pc** register points to the next instruction to **Fetch**.

The arm instruction pipeline is either 3, 5 or 6 stages depending on the arm family, but for the boards I am looking at the pipelines share this start sequence

```
Fetch -> Decode -> Execute -> ...


```

The side effect of this is that by the time we use the **pc** register in the **Execute** stage it has actually moved two instructions further along or 8 bytes.

So if we want the address of the start of our string in **hello.s** we need to add **16** not **24** to the current **pc**.

Note that regular programs would put data like our string into the **.data** or **.rodata** section of the program and the compiler would place this address into the instruction for us.

But we are interested in writing shell code where all of our code and data is going to be inserted into a program at runtime to an unknown memory location.

So we need to use relative addressing when we reference our data. The **pc** register makes a good base for offset addressing like this.

There is an instruction **adr** that will simplify this address calculation as long as the calculated address is a short distance away. This could have been used instead and the assembler will substitute an appropriate **add** or **sub** instruction as we did manually before.

The final syscall argument is self-explanatory.

```
mov r2, #12             @ string length


```

Next a program a little more interesting.

### Launching a shell

We want an assembly program equivalent to this C code to launch a shell.

```
#include <unistd.h>

void main()
{
    execve("/bin/sh", NULL, NULL);
}


```

We first note that the syscall number for [execve(2)](http://man7.org/linux/man-pages/man2/execve.2.html) is 11.

```
# grep execve /usr/include/asm/unistd-common.h
#define __NR_execve (__NR_SYSCALL_BASE + 11)
#define __NR_execveat (__NR_SYSCALL_BASE + 387)


```

The arguments for [execve](http://man7.org/linux/man-pages/man2/execve.2.html) are

```
int execve(const char *filename, char *const argv[], char *const envp[]);


```

Here is the code

```
.text
.global _start

_start:
    adr r0, _string
    mov r1, #0
    mov r2, #0
    mov r7, #11
    svc #1

_string:
.asciz  "/bin/sh"


```

The code is very similar to the hello world example.

The syscall to [exit(3)](http://man7.org/linux/man-pages/man3/exit.3.html) has been removed as unnecessary since **execve** does not return if successful.

Build and run the program

```
$ as shell.s -o shell.o
$ ld shell.o -o shell
$ ./shell
bash-4.4$ echo $$
859
exit


```

Here is the objdump

```
# objdump -d shell.o

shell.o:     file format elf32-littlearm


Disassembly of section .text:

00000000 <_start>:
   0:   e28f000c        add     r0, pc, #12
   4:   e0211001        eor     r1, r1, r1
   8:   e0222002        eor     r2, r2, r2
   c:   e3a0700b        mov     r7, #11
  10:   ef000001        svc     0x00000001

00000014 <_string>:
  14:   6e69622f        .word   0x6e69622f
  18:   0068732f        .word   0x0068732f


```