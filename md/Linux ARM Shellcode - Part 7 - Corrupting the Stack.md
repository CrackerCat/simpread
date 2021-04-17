> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jumpnowtek.com](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part7.html)

Function **main()** is like other functions in that the stack is setup in the prolog and restored for the caller in the epilog.

Consider this program

```
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
    int a, b;
    char buff[8];

    a = 3;
    b = 5;

    if (argc > 1)
        strcpy(buff, argv[1]);

	printf("%08X %08X\n", a, b);

    return 0;
}


```

Build and verify it behaves as expected when provided the following input.

```
$ gcc main.c -o main
$ ./main
00000003 00000005
$ ./main AAAA
00000003 00000005
$ ./main AAAABBBB
00000003 00000000
$ ./main AAAABBBBCCCC
00000000 43434343
$ ./main AAAABBBBCCCCDDDD
44444444 43434343
$ ./main AAAABBBBCCCCDDDDEEEE
44444444 43434343
$ ./main AAAABBBBCCCCDDDDEEEEFFFF
44444444 43434343
Segmentation fault


```

There was an obvious buffer overflow situation, but let’s explain exactly why the program failed when it did.

```
# gdb -q main
Reading symbols from main...done.

(gdb) b main
Breakpoint 1 at 0x5bc

(gdb) run AAAABBBBCCCCDDDDEEEEFFFF
Starting program: /opt/code/mainstack/main AAAABBBBCCCCDDDDEEEEFFFF

Breakpoint 1, 0x004005bc in main ()

(gdb) disas
Dump of assembler code for function main:
   0x004005a8 <+0>:     push    {r11, lr}
   0x004005ac <+4>:     add     r11, sp, #4
   0x004005b0 <+8>:     sub     sp, sp, #24
   0x004005b4 <+12>:    str     r0, [r11, #-24] ; 0xffffffe8
   0x004005b8 <+16>:    str     r1, [r11, #-28] ; 0xffffffe4
=> 0x004005bc <+20>:    mov     r3, #3
   0x004005c0 <+24>:    str     r3, [r11, #-8]
   0x004005c4 <+28>:    mov     r3, #5
   0x004005c8 <+32>:    str     r3, [r11, #-12]
   0x004005cc <+36>:    ldr     r3, [r11, #-24] ; 0xffffffe8
   0x004005d0 <+40>:    cmp     r3, #1
   0x004005d4 <+44>:    ble     0x4005f4 <main+76>
   0x004005d8 <+48>:    ldr     r3, [r11, #-28] ; 0xffffffe4
   0x004005dc <+52>:    add     r3, r3, #4
   0x004005e0 <+56>:    ldr     r2, [r3]
   0x004005e4 <+60>:    sub     r3, r11, #20
   0x004005e8 <+64>:    mov     r1, r2
   0x004005ec <+68>:    mov     r0, r3
   0x004005f0 <+72>:    bl      0x4003fc <strcpy@plt>
   0x004005f4 <+76>:    ldr     r2, [r11, #-12]
   0x004005f8 <+80>:    ldr     r1, [r11, #-8]
   0x004005fc <+84>:    ldr     r3, [pc, #24]   ; 0x40061c <main+116>
   0x00400600 <+88>:    add     r3, pc, r3
   0x00400604 <+92>:    mov     r0, r3
   0x00400608 <+96>:    bl      0x4003f0 <printf@plt>
   0x0040060c <+100>:   mov     r3, #0
   0x00400610 <+104>:   mov     r0, r3
   0x00400614 <+108>:   sub     sp, r11, #4
   0x00400618 <+112>:   pop     {r11, pc}
   0x0040061c <+116>:   andeq   r0, r0, r8, lsl #1
End of assembler dump.

(gdb) i r
r0             0x2                 2
r1             0xbefffd24          3204447524
r2             0xbefffd30          3204447536
r3             0x4005a8            4195752
r4             0x400620            4195872
r5             0x0                 0
r6             0x40042c            4195372
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x411000            4263936
r11            0xbefffbd4          3204447188
r12            0x0                 0
sp             0xbefffbb8          0xbefffbb8
lr             0xb6ea3700          -1226164480
pc             0x4005bc            0x4005bc <main+20>
cpsr           0x60080010          1611137040
fpscr          0x0                 0


```

The command line args came into **main()** in registers **r0** and **r1** and were placed onto the stack after the local variables.

```
(gdb) i r r0
r0             0x2                 2

(gdb) x/s *($r1)
0xbefffe32:     "/opt/code/mainstack/main"

(gdb) x/s *($r1+4)
0xbefffe4b:     "AAAABBBBCCCCDDDDEEEEFFFF"

(gdb) x/s *(0xbefffd24)
0xbefffe31:     "/opt/code/mainstack/main"

(gdb) x/s *(0xbefffd28)
0xbefffe4a:     "AAAABBBBCCCCDDDDEEEEFFFF"


```

From this we can see our stack looks like this

```
      ADDRESS      VALUE
      0xffffffff
      ...
      0xbefffbe8   ...
fp => 0xbefffbd4   ...  [fp]       lr, return location
      0xbefffbd0   ...  [fp, #-4]  caller's fp
      0xbefffbcc   ...  [fp, #-8]  int a
      0xbefffbc8   ...  [fp, #-12] int b
      0xbefffbc4   ...  [fp, #-16] char buff[4-7]
      0xbefffbc0   ...  [fp, #-20] char buff[0-3]
      0xbefffbbc   ...  [fp, #-24] argc
sp => 0xbefffbb8   ...  [fp, #-28] argv, *0xbefffd24 = argv[0], *0xbefffd28 = argv[1]
      0xbefffbb4
      ...   
      0x00000000


```

Jump to the instruction where we call **strcpy(buff, argv[1])**

```
(gdb) b *0x4005f0
Breakpoint 2 at 0x4005f0

(gdb) s
Single stepping until exit from function main,
which has no line number information.

Breakpoint 2, 0x004005f0 in main ()


```

Dump the stack before the call

```
(gdb) x/8x $sp
0xbefffbb8:     0xbefffd24      0x00000002      0x0040042c      0x00000000
0xbefffbc8:     0x00000005      0x00000003      0x00000000      0xb6ea3700


```

Another view of the stack

```
      ADDRESS      VALUE
      0xffffffff
      ...
      0xbefffbe8   ...
fp => 0xbefffbd4   0xb6ea3700  [fp]       lr, return location
      0xbefffbd0   0x00000000  [fp, #-4]  caller's fp
      0xbefffbcc   0x00000003  [fp, #-8]  int a
      0xbefffbc8   0x00000005  [fp, #-12] int b
      0xbefffbc4   0x00000000  [fp, #-16] char buff[4-7]
      0xbefffbc0   0x0040042c  [fp, #-20] char buff[0-3]
      0xbefffbbc   0x00000002  [fp, #-24] argc
sp => 0xbefffbb8   0xbefffd24  [fp, #-28] argv, *0xbefffd24 = argv[0], *0xbefffd28 = argv[1]
      0xbefffbc4
      ...   
      0x00000000	  


```

Now break on the instruction after the strcpy (we don’t need to walk into strcpy)

```
(gdb) b *0x4005f4
Breakpoint 3 at 0x4005f4

(gdb) s
Single stepping until exit from function main,
which has no line number information.

Breakpoint 3, 0x004005f4 in main ()


```

And look at the stack

```
(gdb) x/8x $sp
0xbefffbb8:     0xbefffd24      0x00000002      0x41414141      0x42424242
0xbefffbc8:     0x43434343      0x44444444      0x45454545      0x46464646


```

Another view

```
      ADDRESS      VALUE
      0xffffffff
      ...
      0xbefffbe8   ...
fp => 0xbefffbd4   0x46464646  [fp]       lr, return location
      0xbefffbd0   0x45454545  [fp, #-4]  caller's fp
      0xbefffbcc   0x44444444  [fp, #-8]  int a
      0xbefffbc8   0x43434343  [fp, #-12] int b
      0xbefffbc4   0x42424242  [fp, #-16] char buff[4-7]
      0xbefffbc0   0x41414141  [fp, #-20] char buff[0-3]
      0xbefffbbc   0x00000002  [fp, #-24] argc
sp => 0xbefffbb8   0xbefffd24  [fp, #-28] argv, *0xbefffd24 = argv[0], *0xbefffd28 = argv[1]
      0xbefffbc4
      ...   
      0x00000000	  


```

We overwrote **lr** with 0x46464646 or ‘FFFF’.

When we reach the end of **main()** and attempt to restore the caller’s stack and return

```
0x00400614 <+108>:   sub     sp, r11, #4
0x00400618 <+112>:   pop     {r11, pc}


```

We end up popping **0x46464646** into **pc** which causes the **segfault** when the cpu tries to execute an instruction at that address.