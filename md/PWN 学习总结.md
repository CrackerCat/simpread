> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/aWxvdmVseXc0/p/15734510.html)

一、栈溢出原理
=======

    **什么是栈溢出？栈溢出就是缓冲区溢出的一种。 由于缓冲区溢出而使得有用的存储单元被改写, 往往会引发不可预料的后果。程序在运行过程中，为了临时存取数据的需要，一般都要分配一些内存空间，通常称这些空间为缓冲区。如果向缓冲区中写入超过其本身长度的数据，以致于缓冲区无法容纳，就会造成缓冲区以外的存储单元被改写，这种现象就称为缓冲区溢出。缓冲区长度一般与用户自己定义的缓冲变量的类型有关。(PS: 摘自百度百科)**

    **简单来说，就是程序没有检查用户输入的数据长度，导致攻击者覆盖栈上不是程序希望写入的地方，比如说返回地址。(PS: 本人是这么理解的，如有问题，还请斧正)**

    **在 x86 中，对于调用一个函数，栈的变化如下：首先，将被调用的函数的参数从右到左依次压入栈中，然后将被调用的函数的返回地址压入栈中，然后跳转到被调用函数的地址去，在被调用的函数中，首先将`ebp`(这时的`ebp`是调用者的`ebp`) 压入栈中，最后，将此时的栈顶`esp`赋值给`ebp`寄存器 (此时的`ebp`便是被调用函数的栈底了)**

    **以一个实际程序为例，源码如下所示：**

[![](https://pic.liesio.com/2021/12/26/e3b5eb1257391.png)](https://pic.liesio.com/2021/12/26/e3b5eb1257391.png)

    **对于进入`test`函数后，栈的变化情况如下图所示：**

[![](https://pic.liesio.com/2021/12/26/025298ebc552e.jpg)](https://pic.liesio.com/2021/12/26/025298ebc552e.jpg)

    **接着利用 gdb 调试验证栈的情况如上图所示，首先在`test`函数处打下断点，然后`start`命令将程序运行到`main`函数开头处，查看一下此时的`ebp`寄存器的值和`call test`指令后下一条指令的地址，这里可以看到此时`ebp`寄存器的值为`0xffffd1d8`，`call test`指令后的下一条指令地址为`0x565561f2`，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/96de181db999a.png)](https://pic.liesio.com/2021/12/26/96de181db999a.png)

[![](https://pic.liesio.com/2021/12/26/1b04237e12b3e.png)](https://pic.liesio.com/2021/12/26/1b04237e12b3e.png)

[![](https://pic.liesio.com/2021/12/26/366c6e32bb560.png)](https://pic.liesio.com/2021/12/26/366c6e32bb560.png)

    **然后运行`r`命令，进入`test`函数内部，使用`x`命令查看此时的栈情况，可以发现栈的情况如上图示意图一样，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/b01072be2506b.png)](https://pic.liesio.com/2021/12/26/b01072be2506b.png)

[![](https://pic.liesio.com/2021/12/26/d6dbe8741860f.png)](https://pic.liesio.com/2021/12/26/d6dbe8741860f.png)

    **在`test`函数内部，有个`ver`字符数组，在上面的源码中，我们对其进行了手动赋值，假如该数组通过`strcpy`等函数完成赋值，并且赋值的字符串由用户输入，那么在用户输入的字符串超过`12`个字节大小之后，`ver`数组就会接着往高地址增长，在这其中，可以覆盖掉`test`函数的返回地址、`main`函数的`ebp`等等，这就是栈溢出漏洞。下面来看一个具体的程序实例，有漏洞的程序源码如下所示：**

[![](https://pic.liesio.com/2021/12/26/aeeaab197f469.png)](https://pic.liesio.com/2021/12/26/aeeaab197f469.png)

    **从源码中可以看到，程序对用户的输入无限制，并且`strcpy`函数拷贝也无限制，就造成了栈溢出漏洞，下面使用 gdb 确定偏移地址和`getshell`函数的首地址，首先断点打在`call strcpy`的前一行，查看此时的`buf`数组的地址，也就是`eax`寄存器的值，再看`ebp`寄存器的值，发现其两者相距`0x10`个字节，加上`ebp` 4 个字节，也就是`0x14`个字节即可到达返回地址，再利用命令`disassemble getshell`查看`getshell`函数的首地址，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/6476c9259d6e5.png)](https://pic.liesio.com/2021/12/26/6476c9259d6e5.png)

[![](https://pic.liesio.com/2021/12/26/1e531cc1be97e.png)](https://pic.liesio.com/2021/12/26/1e531cc1be97e.png)

    **有了偏移量和`getshell`函数地址就可以写`exp`了，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/8f6ff153cabdd.png)](https://pic.liesio.com/2021/12/26/8f6ff153cabdd.png)

二、ret2shellcode
===============

    **`ret2shellcode`就是直接部署一段`shellcode`到栈上或者内存其他位置，当然，要使用的前提是，部署的`shellcode`所在的位置要具有执行权限，要关闭地址随机化，下面以一道 ctf 题为例，题目来自`ctfhub`，首先下载好题目，丢到 ida 里面反编译一下，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/6c9e5cd8a9857.png)](https://pic.liesio.com/2021/12/26/6c9e5cd8a9857.png)

    **可以看到，`read`函数允许输入的大小为`0x400`，远大于`buf`数组的长度，这就造成了栈溢出，并且距离`ebp`偏移距离为`0x10`。利用`checksec`检查一下程序，啥保护都没开，利用`file`查看一下，是`64位`的，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/d8e7d1e80145f.png)](https://pic.liesio.com/2021/12/26/d8e7d1e80145f.png)

    **有了偏移量，并且题目输出给出了`buf`数组的地址，那么就可以写`exp`了，首先外面用`socat`将程序发布到某个端口上去，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/f48f39795dfec.png)](https://pic.liesio.com/2021/12/26/f48f39795dfec.png)

    **最后执行`exp`即可，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/9636ddff47f88.png)](https://pic.liesio.com/2021/12/26/9636ddff47f88.png)

三、Rop
=====

    **`ROP`的全称是`Return Oriented Programming`，简单来说就是修改返回地址，指向内存中的指令片段，也就是`gadget`，通过`ret`指令将程序的控制器拿在手里。例如一个存在栈溢出的程序，再将返回地址覆盖为`pop eax;ret`指令地址后，会将返回地址后的 4 个字节弹到`eax`寄存器中，然后`esp+4`，之后又将栈上 4 个字节弹到`eip`寄存器中去，而攻击者要做的就是给栈上返回地址后面覆盖要赋给`eax`的值，下一条`gadget`的地址，通过这种方式就组合成了一条`rop`攻击链，栈情况如下图所示 (PS：下面图片来自 [https://zhuanlan.zhihu.com/p/25892385](https://zhuanlan.zhihu.com/p/25892385) 文章中)：**

[![](https://pic.liesio.com/2021/12/26/36043e980cbc4.png)](https://pic.liesio.com/2021/12/26/36043e980cbc4.png)

    **下面以一道 ctf 题为例 (PS: 题目来自`ctfwiki`)，首先下载好题目，然后查看一下常规信息，可以发现为`x86`架构，开启了`NX`，也就是说无法在栈上执行`shellcode`，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/0263f47cdda0c.png)](https://pic.liesio.com/2021/12/26/0263f47cdda0c.png)

    **丢到`IDA`里面去反编译一下，可以发现是`gets`函数引起的栈溢出漏洞，并且没有直接可以`getshell`的函数，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/4aa29674807bf.png)](https://pic.liesio.com/2021/12/26/4aa29674807bf.png)

    **那么可以考虑利用`rop`来`getshell`，这是一个 32 为的程序，我们可以通过系统调用来获取`shell`，常规的执行命令函数的系统调用汇编代码如下图所示：**

```
mov eax, 0xb  
mov ebx, "/bin/sh"  
mov ecx, 0  
mov edx, 0  
int 80  


``` 

    **要给`eax`赋值，那么我们寻找`pop eax;ret`之类的代码片段，之所以要有`ret`指令，是为了要把程序的控制权拿在手中，给其他寄存器赋值也类似，接下来利用`ROPgadget`工具来寻找相应的代码片段，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/d899406d00e22.png)](https://pic.liesio.com/2021/12/26/d899406d00e22.png)

[![](https://pic.liesio.com/2021/12/26/7c84c4245fd08.png)](https://pic.liesio.com/2021/12/26/7c84c4245fd08.png)

    **在这里，我们在寻找控制`ecx`寄存器的代码片段中，发现可以同时控制`ebx`、`ecx`以及`edx`的一条指令，使用它即可 (在图中地址为`0x806eb90`)，接下来，我们需要`/bin/sh`字符串的地址，打开`IDA`，键入`shift+F12`，发现了`/bin/sh`字符串的地址，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/0d649f11e70d2.png)](https://pic.liesio.com/2021/12/26/0d649f11e70d2.png)

    **当然，我们还需要最重要的偏移量数据，使用`gdb`调试程序，将断点打在调用`gets`函数处，然后`r`，查看当前`eax`与`ebp`的距离为`0x68`(PS: 此时`eax`存放着`v4`数组的首地址)，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/119d8734c4a5e.png)](https://pic.liesio.com/2021/12/26/119d8734c4a5e.png)

    **所有我们编写`exp`的数据都有了，接下来使用`socat`将程序发布到某个端口上去，然后执行`exp`即可，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/c099f95e309bb.png)](https://pic.liesio.com/2021/12/26/c099f95e309bb.png)

[![](https://pic.liesio.com/2021/12/26/5d095e727e014.png)](https://pic.liesio.com/2021/12/26/5d095e727e014.png)

四、ret2libc
==========

    **`ret2libc`从名字上来看，就是通过覆盖返回地址为`libc`库中的函数来`getshell`的一种技术，通常来说，我们会选择`system`等函数来`getshell`，但是一般无法获取到这些函数在内存中的绝对地址的，这就需要通过`got`和`plt`表泄露以经加载的函数在内存中的地址然后减去其偏移地址，从而拿到`libc`库的基地址，然后加上`system`等函数的偏移地址，从而得到`system`等函数在内存中的地址。**

    **`plt`表和`got`表是保存程序动态链接的函数地址，程序通过查询`plt`表获取函数在`got`表中保存的位置，`plt`表就相当于一个索引数组，指向`got`表，程序获取到`plt`表中相关位置之后，然后查询`got`表获取到函数地址，之后跳转到该地址去。下面以一道 ctf 题为例，题目来自`ctfwiki`，如下所示：**

    **首先还是老一套，查看一下程序信息，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/518ec82439f2d.png)](https://pic.liesio.com/2021/12/26/518ec82439f2d.png)

    **可以发现，程序是 32 位的，并且开启了堆栈不可执行保护，也就是说不可以将`shellcode`写入栈上执行。再将程序拖进 IDA 里面看一下，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/adaa920fa8c2b.png)](https://pic.liesio.com/2021/12/26/adaa920fa8c2b.png)

    **可以看到，溢出点在`gets`函数，下面用`gdb`调试一下寻找到偏移量，首先断点下载`gets`函数，然后查看此时`eax`与`ebp`的距离，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/f385b29f198bc.png)](https://pic.liesio.com/2021/12/26/f385b29f198bc.png)

    **可以看到，此时`eax`距离`ebp`一共为`108`个字节，再加上这是 32 位程序，`ebp`本身占 4 个字节，也就是说，填充`108 + 4 = 112`字节后，便是返回地址，找到返回地址后，便是找到`system`函数地址**

    **这里通过访问`got`表来拿到一个已经运行过的函数在内存中的地址，在 linux 中，如果一个函数被调用运行过，那么它的真实地址就会被写进`got`表中，我们可以通过打印函数将其打印出来，获取其地址，之后通过该地址的后三位 (PS: 之所以找后三位，是因为即使开启了`aslr`也不会影响低 12 位的地址) 来确定`libc`的版本 (PS: 可以通过在线网站 [https://libc.blukat.me/](https://libc.blukat.me/) 来查询)，从而获取该版本的`libc`库中函数的偏移地址，最后`泄露地址 - 偏移地址`即可得到`libc`的基地址。然后`libc基地址 + 该版本libc库中system偏移地址`即可得到`system`函数在内存中的地址，同理，`/bin/sh`字符串也是一样的，此处选用的泄露函数地址的函数为`puts`函数，将程序利用`socat`发布后，最后`exp`结果如下图所示 (PS: 程序最好运行在 ubuntu 上，经过实际测试，kali 上失败，下同)：**

[![](https://pic.liesio.com/2021/12/26/5007f21294093.png)](https://pic.liesio.com/2021/12/26/5007f21294093.png)

[![](https://pic.liesio.com/2021/12/26/02a2f78cf4241.png)](https://pic.liesio.com/2021/12/26/02a2f78cf4241.png)

五、格式化字符串
========

    **格式化字符串漏洞，个人理解就是格式字符串参数与其余参数的个数不匹配造成的，网上将原理的文章一大堆，这里就不在重复了。**

    **对于格式化字符串漏洞，可以做到读取任意地址的值，也可以往任意地址写入任意值。**

    **对于利用格式化字符串漏洞读取任意地址的值，首先需要确定偏移量，此处的偏移量不是值上面栈溢出的偏移量，而是格式化字符串函数参数的地址相对于格式化字符串参数的偏移量，确定偏移量可以利用形如`AAAA%n$x`(PS: 里面的`n`就是偏移量) 的格式化字符串参数来确定，或者利用`AAAA%x%x%x%x%x%x%x%x...`(PS: 这里也可以使用`%p`来，但为了防止读到不可读的地址导致程序崩溃，还是推荐使用`%x`来读取) 这种形式来确定，确定的偏移量之后，即可通过`addr%n$x`来读取任意地址的值 (PS: 这里的`addr`指要读取的地址，`n`为偏移量，当然`addr`也可以写在后面，把`n`加`1`即可，因为`%n$x`是第一个参数，`addr`自然是第二个参数，所以`n`要加上`1`)**

    **对于利用格式化字符串漏洞往任意地址写入值，也是需要先确定偏移量，方法和上面一样，写主要利用`%n`，`%n`作用为将前面所写字节数写入指定地址，我们可以利用形如`addr%kc%n$n`这种形式写入，其中`addr`为要写入的地址，`k`为要写入的大小 (PS: 这里需要减去`addr`所占用的字节数)，`n`为偏移量，`$n`表示写入四个字节，当然，也可以使用`$hn`写入双字节，可以使用`$hhn`写入一个字节，当然，确定好偏移量之后，最简单的方法是使用`pwntools`提供的函数即可**

    **下面以一道 ctf 题举例，题目来自`ctfwik`，首先下载好题目，解压后，丢到`kali`里面去，看一下常规信息, 可以发现，该程序为`32位`，开了`NX`等，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/f7a50b2103fe7.png)](https://pic.liesio.com/2021/12/26/f7a50b2103fe7.png)

    **丢到`IDA`中反编译一下，可以发现程序实现了类似`ftp`的功能，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/e1e18c623ce5a.png)](https://pic.liesio.com/2021/12/26/e1e18c623ce5a.png)

    **首先程序调用了`ask_usename`和`ask_password`两个函数获取一个密码，密码就是将`sysbdmin`字符串加一，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/eec2f46523dcd.png)](https://pic.liesio.com/2021/12/26/eec2f46523dcd.png)

[![](https://pic.liesio.com/2021/12/26/b2fe95ee37656.png)](https://pic.liesio.com/2021/12/26/b2fe95ee37656.png)

    **然后程序获取命令，命令有`get`、`put`、`dir`三个命令，获取命令之后，便执行相应的功能，首先来看一下`put`对应的功能函数，该函数首先要求用户输入一个字符串作为文件名，然后要求用户再输入一个字符串作为文件内容，该函数没什么漏洞，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/4a8cc7a6d4d9f.png)](https://pic.liesio.com/2021/12/26/4a8cc7a6d4d9f.png)

    **接下来再看一下`dir`对应的功能函数，该函数作用就是将所有文件名打印出来，也没有什么漏洞，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/131c3536c98b8.png)](https://pic.liesio.com/2021/12/26/131c3536c98b8.png)

    **最后来看一下`get`对应的功能函数，该函数首先要求用户输入文件名，然后将文件名对应的内容拷贝到一个数组中去，最后直接将该数组作为参数传入到`printf`函数中去，典型的格式化字符串漏洞，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/b5274560dee85.png)](https://pic.liesio.com/2021/12/26/b5274560dee85.png)

    **通过以上的代码分析，解决该 ctf 的思路已经很明显了，我们可以首先利用`put`建立一个文件，将`payload`写入到该文件中去，之后调用`get`指令读取该文件，触发格式化字符串漏洞。首先我们先确定偏移量，这里使用`BBBB%x....`来确定，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/594ea3cab12ff.png)](https://pic.liesio.com/2021/12/26/594ea3cab12ff.png)

    **我们可以通过上图发现，偏移量为`7`，那么怎么`getshell`喃，我们可以通过修改`got`表来实现，将`dir`指令对应的功能函数中的`puts`函数修改为`system`函数的地址，这样调用`dir`指令后，里面的`puts`函数实际上会指向`system`函数，我们通过提前新建一个名为`/bon/sh;`的文件，即可解决参数的问题，这里怎么写前面`ret2libc`已经讲了，不在重复，最后使用`socat`发布程序，指向`exp`即可，如下图所示：**

[![](https://pic.liesio.com/2021/12/26/89a921b2f9a1b.png)](https://pic.liesio.com/2021/12/26/89a921b2f9a1b.png)

[![](https://pic.liesio.com/2021/12/26/84985cadb43c7.png)](https://pic.liesio.com/2021/12/26/84985cadb43c7.png)

六、参考链接
======

  **`exp`脚本和题目`github`链接:[https://github.com/windy-purple/pwn_study_summary](https://github.com/windy-purple/pwn_study_summary)**

  **参考链接：**

    [https://www.cnblogs.com/ichunqiu/p/11122229.html](https://www.cnblogs.com/ichunqiu/p/11122229.html)

    [https://www.cnblogs.com/ichunqiu/p/11156155.html](https://www.cnblogs.com/ichunqiu/p/11156155.html)

    [https://www.cnblogs.com/ichunqiu/p/11162515.html](https://www.cnblogs.com/ichunqiu/p/11162515.html)

    [http://drops.xmd5.com/static/drops/tips-4225.html](http://drops.xmd5.com/static/drops/tips-4225.html)

    [https://blog.csdn.net/xiaoi123/article/details/80899155](https://blog.csdn.net/xiaoi123/article/details/80899155)

    [https://bbs.pediy.com/thread-230148.htm](https://bbs.pediy.com/thread-230148.htm)

    [https://sploitfun.wordpress.com/2015/](https://sploitfun.wordpress.com/2015/)

    [https://www.jianshu.com/p/187b810e78d2](https://www.jianshu.com/p/187b810e78d2)

    [https://zhuanlan.zhihu.com/p/25816426](https://zhuanlan.zhihu.com/p/25816426)

    [https://www.cnblogs.com/Donoy/p/5690402.html](https://www.cnblogs.com/Donoy/p/5690402.html)

    [http://shell-storm.org/shellcode/](http://shell-storm.org/shellcode/)

    [https://bbs.pediy.com/thread-259723.htm](https://bbs.pediy.com/thread-259723.htm)

    [https://zhuanlan.zhihu.com/p/25892385](https://zhuanlan.zhihu.com/p/25892385)

    [http://events.jianshu.io/p/9214e84139eb](http://events.jianshu.io/p/9214e84139eb)

    [https://www.cnblogs.com/wulitaotao/p/13909451.html](https://www.cnblogs.com/wulitaotao/p/13909451.html)

    [https://www.cnblogs.com/hktk1643/p/15218090.html](https://www.cnblogs.com/hktk1643/p/15218090.html)

    [https://zhuanlan.zhihu.com/p/367387964](https://zhuanlan.zhihu.com/p/367387964)

    [https://blog.csdn.net/xiaoi123/article/details/80985646](https://blog.csdn.net/xiaoi123/article/details/80985646)

    [https://ctf-wiki.org/pwn/windows/readme/](https://ctf-wiki.org/pwn/windows/readme/)

    [https://libc.blukat.me/](https://libc.blukat.me/)

    [https://blog.csdn.net/qq_41918771/article/details/90665950](https://blog.csdn.net/qq_41918771/article/details/90665950)

    [https://bbs.pediy.com/thread-253638.htm](https://bbs.pediy.com/thread-253638.htm)

    [https://www.anquanke.com/post/id/83835](https://www.anquanke.com/post/id/83835)

    [https://bbs.pediy.com/thread-254869.htm](https://bbs.pediy.com/thread-254869.htm)

    [https://bbs.pediy.com/thread-262816.htm](https://bbs.pediy.com/thread-262816.htm)

*   [一、栈溢出原理](#一栈溢出原理)
*   [二、ret2shellcode](#二ret2shellcode)
*   [三、Rop](#三rop)
*   [四、ret2libc](#四ret2libc)
*   [五、格式化字符串](#五格式化字符串)
*   [六、参考链接](#六参考链接)

  

__EOF__

[![](https://images.cnblogs.com/cnblogs_com/aWxvdmVseXc0/1933883/t_210220024934%E7%99%BD%E8%80%81%E5%B8%88.jpg?t=1637652630521)](https://images.cnblogs.com/cnblogs_com/aWxvdmVseXc0/1933883/t_210220024934%E7%99%BD%E8%80%81%E5%B8%88.jpg?t=1637652630521) *   **本文作者：** [windy_ll](https://www.cnblogs.com/aWxvdmVseXc0)
*   **本文链接：** [https://www.cnblogs.com/aWxvdmVseXc0/p/15734510.html](https://www.cnblogs.com/aWxvdmVseXc0/p/15734510.html)
*   **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://msg.cnblogs.com/msg/send/aWxvdmVseXc0)我。
*   **版权声明：** 本博客所有文章除特别声明外，均采用 [BY-NC-SA](https://creativecommons.org/licenses/by-nc-nd/4.0/ "BY-NC-SA") 许可协议。转载请注明出处！
*   **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。