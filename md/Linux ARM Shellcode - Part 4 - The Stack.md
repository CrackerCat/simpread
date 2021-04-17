> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jumpnowtek.com](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part4.html)

Understanding the behavior of the arm stack is necessary if we are going to exploit it with our shellcode.

Here is an example to show the basic stack operations **push** and **pop**

```
.text
.global _start

_start:
    mov r0, #0
    mov r1, #1
    mov r2, #2
    mov r3, #3
    mov r4, #4
_push:
    push {r0}
    push {r1}
    push {r2, r3, r4}
_pop:
    pop {r}
    pop {r1}
    pop {r2, r3, r4}
_exit:
    mov r7, #1
    svc #1


```

Compile and dump the object file.

```
$ as stack.s -o stack.o
$ ld stack.o -o stack
$ objdump -d stack.o

stack.o:     file format elf32-littlearm


Disassembly of section .text:

00000000 <_start>:
   0:   e3a00000        mov     r0, #0
   4:   e3a01001        mov     r1, #1
   8:   e3a02002        mov     r2, #2
   c:   e3a03003        mov     r3, #3
  10:   e3a04004        mov     r4, #4

00000014 <_push>:
  14:   e52d0004        push    {r0}            ; (str r0, [sp, #-4]!)
  18:   e52d1004        push    {r1}            ; (str r1, [sp, #-4]!)
  1c:   e92d001c        push    {r2, r3, r4}

00000020 <_pop>:
  20:   e49d0004        pop     {r0}            ; (ldr r0, [sp], #4)
  24:   e49d1004        pop     {r1}            ; (ldr r1, [sp], #4)
  28:   e8bd001c        pop     {r2, r3, r4}

0000002c <_exit>:
  2c:   e3a07001        mov     r7, #1
  30:   ef000001        svc     0x00000001


```

In order to see the stack in action, we need to be running the program.

We will use the standard Linux debugger [gdb](https://www.gnu.org/software/gdb/documentation/).

Start gdb passing as an argument our program.

```
$ gdb -q stack
Reading symbols from stack...(no debugging symbols found)...done.

(gdb) break _start
Breakpoint 1 at 0x10054

(gdb) run
Starting program: /opt/asm/stack/stack

Breakpoint 1, 0x00010054 in _start ()

(gdb) disassemble _start
Dump of assembler code for function _start:
=> 0x00010054 <+0>:     mov     r0, #0
   0x00010058 <+4>:     mov     r1, #1
   0x0001005c <+8>:     mov     r2, #2
   0x00010060 <+12>:    mov     r3, #3
   0x00010064 <+16>:    mov     r4, #4
End of assembler dump.


```

The **-q** switch is just to avoid some boilerplate messages.

You can use abbreviations for commands in gdb as long as they are not ambiguous. After first use I will use abbreviations.

Here is where the stack pointer **sp** is at the start of the run.

```
(gdb) info registers sp r0 r1 r2 r3 r4
sp             0xbefffd40          0xbefffd40
r0             0x0                 0
r1             0x0                 0
r2             0x0                 0
r3             0x0                 0
r4             0x0                 0


```

And here is a memory dump ‘x’ of 8 32-bit values in hex ‘8x’ starting with where the address **sp** points to.

```
(gdb) x/8x $sp
0xbefffd40:     0x00000001      0xbefffe57      0x00000000      0xbefffe6c
0xbefffd50:     0xbefffea0      0xbefffeaf      0xbefffec1      0xbefffecb


```

Let’s jump down to the _push section of code.

```
(gdb) b _push
Breakpoint 2 at 0x10074

(gdb) step
Single stepping until exit from function _start,
which has no line number information.
0x00010068 in _push ()

(gdb) disas _push
Dump of assembler code for function _push:
=> 0x00010068 <+0>:     push    {r0}            ; (str r0, [sp, #-4]!)
   0x0001006c <+4>:     push    {r1}            ; (str r1, [sp, #-4]!)
   0x00010070 <+8>:     push    {r2, r3, r4}
End of assembler dump.


```

Take a look at the current register states

```
(gdb) i r sp r0 r1 r2 r3 r4
sp             0xbefffd40          0xbefffd40
r0             0x0                 0
r1             0x1                 1
r2             0x2                 2
r3             0x3                 3
r4             0x4                 4

(gdb) x/8x $sp
0xbefffd40:     0x00000001      0xbefffe57      0x00000000      0xbefffe6c
0xbefffd50:     0xbefffea0      0xbefffeaf      0xbefffec1      0xbefffecb


```

The stack pointer **sp** has not moved and the registers **r0 - r4** have the values we set.

Now execute the first push instruction.

```
(gdb) si
0x0001006c in _push ()

(gdb) i r sp
sp             0xbefffd3c          0xbefffd3c

(gdb) x/8x $sp
0xbefffd3c:     0x00000000      0x00000001      0xbefffe57      0x00000000
0xbefffd4c:     0xbefffe6c      0xbefffea0      0xbefffeaf      0xbefffec1


```

The stack was at **0xbefffd40** before the **push {r0}** instruction.

After the push the stack is at **0xbefffd3c** and the memory at **0xbefffd3c** got the value of **r0**.

```
0xbefffd40 - 4 = 0xbefffd3c


```

From this we can see that our stack moves down in memory (toward zero) and that the **sp** points to the last used location on the stack.

When we push something onto the stack, the **sp** is decremented first and then it is used.

In C psuedo code a push operation is

and this is what the **str r0, [sp, #-4]!** is telling us. Arm calls this **preindex with writeback** indexing.

Stepping through the remaining push instructions.

```
(gdb) si
0x00010070 in _push ()

(gdb) i r sp
sp             0xbefffd38          0xbefffd38

(gdb) x/8x $sp
0xbefffd38:     0x00000001      0x00000000      0x00000001      0xbefffe57
0xbefffd48:     0x00000000      0xbefffe6c      0xbefffea0      0xbefffeaf


```

The stack moved to

```
0xbefffd3c - 4 = 0xbeffd38


```

The final **push**

```
(gdb) si

Breakpoint 2, 0x00010074 in _pop ()

(gdb) i r sp r0 r1 r2 r3 r4
sp             0xbefffd2c          0xbefffd2c
r0             0x0                 0
r1             0x1                 1
r2             0x2                 2
r3             0x3                 3
r4             0x4                 4

(gdb) x/8x $sp
0xbefffd2c:     0x00000002      0x00000003      0x00000004      0x00000001
0xbefffd3c:     0x00000000      0x00000001      0xbefffe57      0x00000000


```

Note that when we did a multiple register push, the registers were pushed from right to left.

This instruction

is equivalent to the sequence

```
push {r4}
push {r3}
push {r2}


```

Here is another way of looking at the stack

```
ADDRESS      VALUE
0xffffffff
...
0xbefffd44   ...
0xbefffd40   ...         <= stack pointer started here
0xbefffd3c   0x00000000
0xbefffd38   0x00000001
0xbefffd34   0x00000004
0xbefffd30   0x00000003
0xbefffd2c   0x00000002  <= current stack pointer
0xbefffd28   ...
...
0x00000000


```

Now the pop instructions.

This is where we are in the code.

```
(gdb) disas _pop
Dump of assembler code for function _pop:
=> 0x00010074 <+0>:     pop     {r0}            ; (ldr r0, [sp], #4)
   0x00010078 <+4>:     pop     {r1}            ; (ldr r1, [sp], #4)
   0x0001007c <+8>:     pop     {r2, r3, r4}
End of assembler dump.


```

Run the first **pop**

```
(gdb) si
0x00010078 in _pop ()

(gdb) i r sp r0 r1 r2 r3 r4
sp             0xbefffd30          0xbefffd30
r0             0x2                 2
r1             0x1                 1
r2             0x2                 2
r3             0x3                 3
r4             0x4                 4

(gdb) x/8x $sp
0xbefffd30:     0x00000003      0x00000004      0x00000001      0x00000000
0xbefffd40:     0x00000001      0xbefffe57      0x00000000      0xbefffe6c


```

From this we can see that the **sp** is used first and then incremented.

```
0xbefffd2c + 4 = 0xbefffd30


```

In C psuedo code a **pop** is

The next **pop**

```
(gdb) si
0x0001007c in _pop ()

(gdb) i r sp r0 r1 r2 r3 r4
sp             0xbefffd34          0xbefffd34
r0             0x2                 2
r1             0x3                 3
r2             0x2                 2
r3             0x3                 3
r4             0x4                 4

(gdb) x/8x $sp
0xbefffd34:     0x00000004      0x00000001      0x00000000      0x00000001
0xbefffd44:     0xbefffe57      0x00000000      0xbefffe6c      0xbefffea0


```

And the final multi-register **pop**

```
(gdb) si
0x00010080 in _exit ()

(gdb) i r sp r0 r1 r2 r3 r4
sp             0xbefffd40          0xbefffd40
r0             0x2                 2
r1             0x3                 3
r2             0x4                 4
r3             0x1                 1
r4             0x0                 0

(gdb) x/8x $sp
0xbefffd40:     0x00000001      0xbefffe57      0x00000000      0xbefffe6c
0xbefffd50:     0xbefffea0      0xbefffeaf      0xbefffec1      0xbefffecb


```

Note that with the multiple register pop, the registers were popped from left to right.

Or in other words

is equivalent to the sequence

```
pop {r2}
pop {r3}
pop {r4}


```