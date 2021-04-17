> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jumpnowtek.com](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part5.html)

Consider this C program

```
#include <unistd.h>

void hello()
{
    write(1, "Hello ", 6);
}

void world()
{
    write(1, "world\n", 6);
}

int main()
{
    hello();
    world();
}


```

Build and run and it does what you expect

```
$ gcc stack.c -o stack
$ ./stack
Hello world


```

Take an object dump on the executable so we can look at the stack operations. I am showing only the sections of interest.

```
# objdump -d stack

...

00000578 <hello>:
 578:   e92d4800        push    {fp, lr}
 57c:   e28db004        add     fp, sp, #4
 580:   e3a02006        mov     r2, #6
 584:   e59f3014        ldr     r3, [pc, #20]   ; 5a0 <hello+0x28>
 588:   e08f3003        add     r3, pc, r3
 58c:   e1a01003        mov     r1, r3
 590:   e3a00001        mov     r0, #1
 594:   ebffff92        bl      3e4 <write@plt>
 598:   e320f000        nop     {0}
 59c:   e8bd8800        pop     {fp, pc}
 5a0:   000000cc        .word   0x000000cc

000005a4 <world>:
 5a4:   e92d4800        push    {fp, lr}
 5a8:   e28db004        add     fp, sp, #4
 5ac:   e3a02006        mov     r2, #6
 5b0:   e59f3014        ldr     r3, [pc, #20]   ; 5cc <world+0x28>
 5b4:   e08f3003        add     r3, pc, r3
 5b8:   e1a01003        mov     r1, r3
 5bc:   e3a00001        mov     r0, #1
 5c0:   ebffff87        bl      3e4 <write@plt>
 5c4:   e320f000        nop     {0}
 5c8:   e8bd8800        pop     {fp, pc}
 5cc:   000000a8        .word   0x000000a8

000005d0 <main>:
 5d0:   e92d4800        push    {fp, lr}
 5d4:   e28db004        add     fp, sp, #4
 5d8:   ebffffe6        bl      578 <hello>
 5dc:   ebfffff0        bl      5a4 <world>
 5e0:   e3a03000        mov     r3, #0
 5e4:   e1a00003        mov     r0, r3
 5e8:   e8bd8800        pop     {fp, pc}

...


```

Break on **main** and dump the registers at startup

```
$ gdb -q stack
Reading symbols from stack...done.

(gdb) b main
Breakpoint 1 at 0x5d8

(gdb) run
Starting program: /opt/code/stack/stack

Breakpoint 1, 0x004005d8 in main ()

(gdb) disas main
Dump of assembler code for function main:
   0x004005d0 <+0>:     push    {r11, lr}
   0x004005d4 <+4>:     add     r11, sp, #4
=> 0x004005d8 <+8>:     bl      0x400578 <hello>
   0x004005dc <+12>:    bl      0x4005a4 <world>
   0x004005e0 <+16>:    mov     r3, #0
   0x004005e4 <+20>:    mov     r0, r3
   0x004005e8 <+24>:    pop     {r11, pc}

End of assembler dump.

(gdb) i r
r0             0x1                 1
r1             0xbefffd44          3204447556
r2             0xbefffd4c          3204447564
r3             0x4005d0            4195792
r4             0x4005ec            4195820
r5             0x0                 0
r6             0x4003fc            4195324
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x411000            4263936
r11            0xbefffbf4          3204447220
r12            0x0                 0
sp             0xbefffbf0          0xbefffbf0
lr             0xb6ea3700          -1226164480
pc             0x4005d8            0x4005d8 <main+8>
cpsr           0x60080010          1611137040
fpscr          0x0                 0


```

There is a new instruction and a couple of new registers to introduce here.

The **pc** register always holds the next instruction to execute.

The ‘branch with link’ **bl** instruction is used to make function calls. Before jumping the **bl** instruction saves the address of the next instruction in the link register **lr**. This will be the next instruction to execute once the function returns.

So in normal arm mode, the **bl** instruction does this

```
lr = pc + 4
pc = <some branch address>


```

in thumb mode

```
lr = pc + 2
pc = <some branch address>


```

A more concrete example, this instruction

```
0x004005d8 <+8>:     bl      0x400578 <hello>   


```

will put this in the **lr** register before branching

```
lr = pc + 4 = 0x004005d8 + 4 = 0x004005dc


```

This instruction

```
0x004005dc <+12>:    bl      0x4005a4 <world>


```

will put this in **lr**

```
lr = pc + 4 = 0x004005dc + 4 = 0x004005e0


```

Register **r11** by convention is used as the frame pointer **fp**. The frame pointer is a reference to the stack address when a function is entered. We will see how it is used in later examples. It is important that the **fp** not change throughout the duration of the function.

Note: [gdb](https://www.gnu.org/software/gdb/documentation/) disassembly shows **fp** as **r11**, but you can substitute **fp** when referencing.

So looking at just the registers we are interested in

```
(gdb) i r fp sp lr pc
fp             0xbefffbf4          3204447220
sp             0xbefffbf0          0xbefffbf0
lr             0xb6ea3700          -1226164480
pc             0x4005d8            0x4005d8 <main+8>


```

Now step into the **hello** function

```
(gdb) si
0x00400578 in hello ()

(gdb) disas hello
Dump of assembler code for function hello:
=> 0x00400578 <+0>:     push    {r11, lr}
   0x0040057c <+4>:     add     r11, sp, #4
   0x00400580 <+8>:     mov     r2, #6
   0x00400584 <+12>:    ldr     r3, [pc, #20]   ; 0x4005a0 <hello+40>
   0x00400588 <+16>:    add     r3, pc, r3
   0x0040058c <+20>:    mov     r1, r3
   0x00400590 <+24>:    mov     r0, #1
   0x00400594 <+28>:    bl      0x4003e4 <write@plt>
   0x00400598 <+32>:    nop     {0}
   0x0040059c <+36>:    pop     {r11, pc}
   0x004005a0 <+40>:    andeq   r0, r0, r8, asr #1
End of assembler dump.

(gdb) i r fp sp lr pc
fp            0xbefffbf4          3204447220
sp             0xbefffbf0          0xbefffbf0
lr             0x4005dc            4195804
pc             0x400578            0x400578 <hello>


```

Note that **lr** now has the address of the instruction to execute after **hello** finishes **0x4005dc**.

And **pc** has the address of the next instruction to execute **0x400578**

```
(gdb) si
0x0040057c in hello ()

(gdb) i r fp sp lr pc
fp             0xbefffbf4          3204447220
sp             0xbefffbe8          0xbefffbe8
lr             0x4005dc            4195804
pc             0x40057c            0x40057c <hello+4>

(gdb) x/4x $sp
0xbefffbe8:     0xbefffbf4      0x004005dc      0x00000000      0xb6ea3700


```

The **lr** and **fp** registers got pushed onto the stack in that order.

```
(gdb) si
0x00400580 in hello ()

(gdb) i r fp sp lr pc
fp            0xbefffbec          3204447212
sp             0xbefffbe8          0xbefffbe8
lr             0x4005dc            4195804
pc             0x400580            0x400580 <hello+8>


```

The point of this is to save the return address and the frame pointer of the calling function so we can properly return when this function is over.

The frame pointer is set to

The frame pointer is used as a reference for local stack variables. We don’t have any local variables in this function, so the **fp** isn’t used. The next example will show its use.

Here is another view of the current stack

```
      ADDRESS      VALUE
      0xffffffff
      ...
      0xbefffd44   ...
      0xbefffd40   ...         
fp => 0xbefffbe4   0x004005dc
sp => 0xbefffbe8   0xbefffbf4
      0xbefffbec   ...
      0xbefffbf0   ...
      ...
      0x00000000


```

Skipping to the end of the function

```
(gdb) b *0x0040059c
Breakpoint 2 at 0x40059c

(gdb) s
Single stepping until exit from function hello,
which has no line number information.
Hello
Breakpoint 2, 0x0040059c in hello ()

(gdb) disas
Dump of assembler code for function hello:
   0x00400578 <+0>:     push    {r11, lr}
   0x0040057c <+4>:     add     r11, sp, #4
   0x00400580 <+8>:     mov     r2, #6
   0x00400584 <+12>:    ldr     r3, [pc, #20]   ; 0x4005a0 <hello+40>
   0x00400588 <+16>:    add     r3, pc, r3
   0x0040058c <+20>:    mov     r1, r3
   0x00400590 <+24>:    mov     r0, #1
   0x00400594 <+28>:    bl      0x4003e4 <write@plt>
   0x00400598 <+32>:    nop     {0}
=> 0x0040059c <+36>:    pop     {r11, pc}
   0x004005a0 <+40>:    andeq   r0, r0, r8, asr #1
End of assembler dump.


```

Look at the registers before the **pop {r11, pc}**

```
(gdb) i r fp sp lr pc
fp             0xbefffbec          3204447212
sp             0xbefffbe8          0xbefffbe8
lr             0x400598            4195736
pc             0x40059c            0x40059c <hello+36>

(gdb) si
0x004005dc in main ()


```

Executing that **pop** instruction loaded **pc** with **0x004005dc** and the debugger immediately jumped there to show us the next instruction.

```
(gdb) i r fp sp lr pc
fp             0xbefffbf4          3204447220
sp             0xbefffbf0          0xbefffbf0
lr             0x400598            4195736
pc             0x4005dc            0x4005dc <main+12>

(gdb) disas
Dump of assembler code for function main:
   0x004005d0 <+0>:     push    {r11, lr}
   0x004005d4 <+4>:     add     r11, sp, #4
   0x004005d8 <+8>:     bl      0x400578 <hello>
=> 0x004005dc <+12>:    bl      0x4005a4 <world>
   0x004005e0 <+16>:    mov     r3, #0
   0x004005e4 <+20>:    mov     r0, r3
   0x004005e8 <+24>:    pop     {r11, pc}
End of assembler dump.


```

### Summarizing

1.  The program counter register **pc** holds the next instruction to Execute.
    
2.  The frame pointer register **fp** holds a reference to the stack location at the start of a function.
    
3.  The link register **lr** holds the address to execute after a function returns. The function prolog saves this on the stack.
    
4.  The function prolog saves the **fp** for the caller and sets up a new **fp** for the this function.
    
5.  The function epilog restores the **fp** for the caller and sets the **pc** register to **lr**.
    

Standard function prolog

```
push {fp, lr}
add fp, sp, #4


```

Standard function epilog

In a C program, **main()** is just another function and has the same prolog and epilog code to setup the stack and return address.