> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jumpnowtek.com](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part6.html)

One defense against memory exploits is to vary the locations of a program’s different memory segments on every execution.

Linux refers to this mitigation strategy as “address space layout randomization” or **ASLR** for short.

To make writing these examples easier and consistent when I come back to them, I will temporary disable this on the test system.

First a short check to see if **ASLR** is working.

```
#include <stdio.h>

int main(int argc, char **argv)
{
    char buff[64];

    printf("buff address: %p\n", buff);

    return 0;
}


```

Build and run a few times.

```
$ gcc main.c -o main

$ ./main
buff address: 0xbea3abc0

$ ./main
buff address: 0xbe9f8bc0

$ ./main
buff address: 0xbefd9bc0

$ ./main
buff address: 0xbede8bc0

$ ./main
buff address: 0xbeb9cbc0


```

A different stack address every run. (Note that only the lower 24-bits were randomized.)

To completely disable **ASLR**

```
# cat /proc/sys/kernel/randomize_va_space
2

# echo 0 > /proc/sys/kernel/randomize_va_space


```

Now running that same test program

```
$ ./main
buff address: 0xbefffbc0

$ ./main
buff address: 0xbefffbc0

$ ./main
buff address: 0xbefffbc0

$ ./main
buff address: 0xbefffbc0

$ ./main
buff address: 0xbefffbc0


```

Running a program under the debugger also disables **ASLR**.