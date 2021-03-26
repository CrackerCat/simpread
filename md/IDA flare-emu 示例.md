> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=MzUzMjQyMDE3Ng==&mid=2247484538&idx=1&sn=11d4e3a6c840e2bc1033ea3bb5ad7782&chksm=fab2c745cdc54e53b874c3d61e7e827a04cff0a799002abcedb59a03f658c4fe3004851c7564&scene=178&cur_album_id=1642597492376518662#rd)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPN4JBr5jf0dnKdS2fCr4BvOgYCviaQf8d27E1cbZHWZc9iajwfMZOvsiceibX6QZyR5xUj5xMmmkZ9S1g/640?wx_fmt=png)

标题: IDA flare-emu 示例

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPN4JBr5jf0dnKdS2fCr4BvOwUvibGgQbYPOBZGmpwBUqRrVfFicUaeb759icdibd594nDPEb9KO6kK6YA/640?wx_fmt=png)

  
☆ 简介

IDA 静态分析时想调用目标 binary 中一段代码，或者说模拟 / 仿真执行目标 binary 中一段代码，这个需求比较常见。最简单的一种场景是，想检查目标 binary 中某种算法函数的 in/out，想自己提供 in，让其过一遍算法函数，检查 out。稍微复杂点的场景，目标 binary 中有私有字符串反混淆函数，想调用之，自动反混淆目标 binary 中的一段数据区。

过去为了应对这种需求，可以将汇编代码片段扒出来重编译，可以用 Python 模拟实现，是否可行视该段代码复杂度而不同。还有更野蛮的方案，2006 年 hume 和我逆向 Skype 时需要调用若干 Skype.exe 中的代码片段，hume 把 Skype.exe 简单改成 Skype.dll 的效果。后来搞一些 ELF 时我也用过类似的技术思路，好处是不用扒代码出来，懒人超爱。

不久前 bluerust 向我推荐使用 flare-emu 应对这种需求，原话大致如下:

》这小半年断断续续接触到之前主观上不太愿意碰的一些东西，包括 llvm、docker、unicorn 等等。除了开眼界外，对业务水平、生产力提高也是帮助巨大。像模拟执行，以前我就恃着自己写过几年汇编，经常硬生生把汇编代码从 IDA 抠出，当库函数用。现在不干了，几句 python 让 unicorn 跑去，就地解决。还是脑袋似木瓜，很多年轻的娃本科毕业就把这些玩意玩得烂熟了。

》flare-emu 是个 IDA 插件，封装了 unicorn。我用了几轮，感觉十分友好，封装的文件就三个，十分方便二次开发。逆向时，有时需要调用原来的一段代码，改写当然可以，扒出来重编译也是条路，但模拟执行就地解决在大多数情况下可能是最优解。

bluerust 是我前同事，后来曾入职过 FireEye 几年，现下在北美逍遥自在。若 scz 曾经是前浪的话，bluerust 就是将前者拍死在沙滩上的后浪。现在这位后浪亦将步入中年，迟早会被后后浪拍死在沙滩上。他将来怎么死的我不知道，反正现如今我对他的各种技术推荐甚为重视，毕竟老年程序员眼界缩窄，再不虚心好学的话，只会加快自身被遗弃于技术垃圾堆的进程。不是所有的前浪都像 hume 那样，浪奔浪流，万里涛涛江水永不休。

本文不从上帝视角展开，没有直接给精简演示方案，会介绍完整学习过程，稍显冗长，诸君可依据自身技术背景进行跳跃式阅读。

主体技术方案由 bluerust 提供。

☆ 寻找练手对象

决定用 x64/RedHat 上的 md5sum 测试 flare-emu 的效果。md5sum 必然包含 MD5 算法相关函数，较容易定位它们，然后用 flare-emu 模拟执行，检验 MD5 算法的 in/out。

1) 获取 md5sum 源码

这与 flare-emu 无关，获取 md5sum 源码只是便于在本文演示中减少一些解释性文字。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPN4JBr5jf0dnKdS2fCr4BvOrjt6BDPvBgnP98icZ6lmStZaqeYNFmhX0FKk6YxkcoUDFkgEFh0Lf0g/640?wx_fmt=png)

  
放狗搜之，比如:  

```
http://ftp.scientificlinux.org/linux/scientific/7.2/SRPMS/vendor/coreutils-8.22-23.el7.src.rpm


```

md5sum 源码位置:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPN4JBr5jf0dnKdS2fCr4BvOj4vVBJprOlYQCaw8uEx4PLSOyqBQNry4n5NhGstAx1Iiaa3GTibRYg7Q/640?wx_fmt=png)

  
2) DIGEST_STREAM()/md5_stream()

看过 md5sum.c 才知道，从 md5sum 到 sha384sum 用一套模具，调的都是 DIGEST_STREAM()，区别只是:

```
#if HASH_ALGO_MD5
# define DIGEST_STREAM md5_stream
#elif HASH_ALGO_SHA384
# define DIGEST_STREAM sha384_stream
#endif


```

```
int md5_stream ( FILE *stream, void *resblock )
{
    struct md5_ctx  ctx;
    size_t          sum;
    char           *buffer  = malloc( BLOCKSIZE + 72 );

    if ( !buffer )
        return 1;
    md5_init_ctx( &ctx );
    while ( 1 )
    {
        size_t  n;

        sum = 0;
        while ( 1 )
        {
            n       = fread( buffer + sum, 1, BLOCKSIZE - sum, stream );
            sum    += n;
            if ( sum == BLOCKSIZE )
                break;
            if ( n == 0 )
            {
                /*
                 * Check for the error flag IFF N == 0, so that we don't
                 * exit the loop after a partial read due to e.g., EAGAIN
                 * or EWOULDBLOCK.
                 */
                if ( ferror( stream ) )
                {
                    free( buffer );
                    return 1;
                }
                goto process_partial_block;
            }
            /*
             * We've read at least one byte, so ignore errors. But always
             * check for EOF, since feof may be true even though N > 0.
             * Otherwise, we could end up calling fread after EOF.
             */
            if ( feof( stream ) )
                goto process_partial_block;
        }
        md5_process_block( buffer, BLOCKSIZE, &ctx );
    }

process_partial_block:

    if ( sum > 0 )
        md5_process_bytes( buffer, sum, &ctx );
    md5_finish_ctx( &ctx, resblock );
    free( buffer );
    return 0;
}


```

☆ IDA 反汇编时识别 MD5 算法

假设不知道目标 binary 包含哪些知名算法，IDA 7.1 及之前版本用 findcrypt.plw，之后的 IDA 用 "IDA Signsrch" 或 findcrypt-yara。

1) IDA Signsrch/signsrch.exe

```
https://github.com/nihilus/IDA_Signsrch
http://aluigi.altervista.org/mytoolz/signsrch.zip
http://aluigi.altervista.org/mytoolz/signsrch.sig.zip (特征数据库)


```

"IDA Signsrch" 有 BUG，扫 md5sum 未能找到 MD5 算法特征常量，bluerust 也曾跟我吐槽说它不灵。但我在下文用它找到过 ZIP 算法特征常量:

```
《唤醒沉睡的木马》
http://scz.617.cn:8/windows/202011231525.txt


```

"IDA Signsrch" 是可执行版本的 IDA 插件移植版，原版无 BUG:

```
$ signsrch.exe -e md5sum

  offset   num  description [bits.endian.size]
  --------------------------------------------
  00402c32 1018 MD5 digest [32.le.272&]
  00402c47 2053 RIPEMD-128 InitState [32.le.16&]
  00406b40 1038 padding used in hashing algorithms (0x80 0 ... 0) [..64]


```

signsrch.exe 找到 MD5 算法特征常量。指定 "-e" 时，把目标 binary 当成 PE/ELF，给出的 offset 是 RVA 而不是文件偏移，0x402c32 对应 md5_init_ctx()。

2) findcrypt-yara

```
https://github.com/polymorf/findcrypt-yara


```

这也是一个 IDA 插件。若你的 IDA、Python 都是安装版，跳过本小节内容。假设你是 "Portable IDA+IDAPython" 爱好者，参看:

```
《Portable Python》
http://scz.617.cn:8/python/202011191444.txt


```

findcrypt-yara 依赖 yara-python 模块，在有 Visual Studio 2019 社区版的环境中执行:

```
$ python.exe -m pip install yara-python


```

不要求手动设置编译环境，主要是得到

```
<Python39>\Lib\site-packages\yara.cp39-win_amd64.pyd


```

复制 yara.cp39-win_amd64.pyd 到

```
<IDA>\Lib\site-packages\yara.cp39-win_amd64.pyd


```

复制 findcrypt3.py、findcrypt3.rules 到

```
<IDA>\plugins\


```

Edit  
 Plugins  
   Findcrypt (Ctrl+Alt+F)

findcrypt-yara 没有明显 BUG，在 "Findcrypt results" 窗口显示找到的 MD5 算法特征值，双击跳过去。

☆ flare-emu 示例

关于 flare-emu 的安装，参 [1]

1) Hex-Rays 下的 md5_stream()

```
int __fastcall md5_stream(FILE *stream, void *resblock)
{
  int *buffer; // r12
  int ret; // eax
  size_t sum; // rbx
  size_t n; // rax
  char ctx[168]; // [rsp+0h] [rbp-E8h]

  buffer = (int *)malloc(0x8048uLL);
  ret = 1;
  if ( buffer )
  {
    sum = 0LL;
    md5_init_ctx(ctx);
    while ( 1 )
    {
      while ( 1 )
      {
        n = fread_unlocked((char *)buffer + sum, 1uLL, 0x8000 - sum, stream);
        sum += n;
        if ( sum != 0x8000 )
          break;
        md5_process_block(buffer, 0x8000LL, ctx);
        sum = 0LL;
      }
      if ( !n )
        break;
      /*
       * feof()
       */
      if ( stream->_flags & 0x10 )
        goto process_partial_block;
    }
    /*
     * ferror()
     */
    if ( stream->_flags & 0x20 )
    {
      free(buffer);
      return 1;
    }
process_partial_block:
    if ( sum )
      md5_process_bytes(buffer, sum, ctx);
    md5_finish_ctx(ctx, (__int64)resblock);
    free(buffer);
    ret = 0;
  }
  return ret;
}


```

为了聚焦演示 flare-emu，上面的 F5 结果已重命名过，真实世界没有这么理想的 F5 结果。注意到 ferror()、feof() 在汇编代码中已 inline 展开。

最初想得挺简单，假设 md5sum 的实现是读文件到 buf，然后对 buf 求 MD5，此时求 MD5 的代码将只涉及算法，不涉及文件 I/O 或其他什么系统调用、库函数调用。若真是如此实现，非常适合演示 flare-emu，事实上在逆向工程中很多验证 in/out 的需求就是这类情形。对于前述理想情形验证 in/out，还可以在调试器中直接修改 PC 寄存器指向算法函数入口，临时组织函数形参，当然这已超出静态分析范畴。

起初我没有去找 md5_stream() 的源码，只在 F5 中看到上述代码，发现有 I/O，就问 bluerust，是不是没法用 flare-emu 模拟执行 md5_stream()；他说可以，然后给我秀了一番。

2) emu_md5_stream.py

```
#!/usr/bin/env python3
# -*- encoding: cp936 -*-

#
# Author: bluerust, scz
#

#
# IDA 7.5.1+Python 3.9
#
# 对"c:\windows\system.ini"求MD5，在IDA中看到类似输出
#
# Opening 0x7fff1e33fa90
# ret = 0
# 00000000: 28 6A 9E DB 37 9D C3 42  3A 52 8B 08 64 A0 F1 11  (j..7..B:R..d...
# 00000010: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................
# Closing 0x7fff1e33fa90
#
# 对比如下命令的求值结果
#
# $ wsl md5sum /mnt/c/windows/system.ini
# 286a9edb379dc3423a528b0864a0f111  /mnt/c/windows/system.ini
#

import sys, ctypes, inspect, traceback, functools
import hexdump
import flare_emu

#
# dir(ctypes)
# dir(ctypes.cdll)
#
cso                 = ctypes.cdll.msvcrt

#
# https://docs.python.org/3/library/ctypes.html
#
# 参看"Fundamental data types"，这样可以避免NoneType。
#
# >>> ctypes.sizeof( VOIDP )
# 8
#
class VOIDP ( ctypes.c_void_p ) :

    #
    # 有这个才可以对VOIDP类型求int()
    #
    def __int__ ( self ) :
        return self.value

#
# end of class VOIDP
#

#
# 为被调函数指定函数原型，即指定restype(返回值类型)、argtypes(参数类形)，
# 否则极易触发"段错误"。
#
# FILE *fopen(const char *pathname, const char *mode);
# size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
# int fclose(FILE *stream);
# void *calloc(size_t nmemb, size_t size);
# void free(void *ptr);
# void *memmove(void *dest, const void *src, size_t n);
#
# >>> ctypes.sizeof( ctypes.c_size_t )
# 8
#
cso.fopen.argtypes      = ( ctypes.c_char_p, ctypes.c_char_p )
cso.fopen.restype       = VOIDP

cso.fread.argtypes      = ( VOIDP, ctypes.c_ulong, ctypes.c_ulong, VOIDP )
cso.fread.restype       = ctypes.c_size_t

#
# 该行只能用[]，不能用()
#
cso.fclose.argtypes     = [ VOIDP ]
cso.fclose.restype      = ctypes.c_int

cso.calloc.argtypes     = ( ctypes.c_size_t, ctypes.c_size_t )
cso.calloc.restype      = VOIDP

cso.free.argtypes       = [ VOIDP ]
cso.free.restype        = None

#
# 其实有ctypes.memmove()、ctypes.memset()，不需要自己准备这两个函数
#
cso.memmove.argtypes    = ( VOIDP, VOIDP, ctypes.c_size_t )
cso.memmove.restype     = VOIDP

#
# Be called before each instruction is emulated.
#
def PrivateInstructionHook ( unicornObject, address, instructionSize, userData ) :
    #
    # 如果不需要跟踪到指令级别，注释掉这条语句
    #
    print( "=> %#x [%s]" % ( address, ida_lines.tag_remove( ida_lines.generate_disasm_line( address ) ) ) )
    pass
#
# end of PrivateInstructionHook
#

#
# How can I find the number of arguments of a Python function
# https://stackoverflow.com/questions/847936/how-can-i-find-the-number-of-arguments-of-a-python-function
#
def GetFuncParamNum ( func ) :
    sig = inspect.signature( func )
    return len( sig.parameters )
#
# end of GetFuncParamNum
#

#
# 关于装饰器参看
#
# https://www.programiz.com/python-programming/decorator (入门推荐)
# https://www.python-course.eu/python3_decorators.php
# https://www.runoob.com/w3cnote/python-func-decorators.html
#
# 这是一个通用装饰器函数
#
def PrivateHook ( func ) :

    #
    # args will be the tuple of positional arguments and kwargs will be
    # the dictionary of keyword arguments.
    #
    # 下面这句隐式包含
    #
    # func_wrapper.__name__   = func.__name__
    # func_wrapper.__doc__    = func.__doc__
    # func_wrapper.__module__ = func.__module__
    #
    @functools.wraps(func)
    def func_wrapper ( *args, **kwargs ) :
        #
        # print( func.__name__ )
        # print( args )
        # print( kwargs )
        #
        # 参看flare_emu_hooks.py，本例所涉及的函数原型是
        #
        # (eh, address, argv, funcName, userData)
        #
        eh          = args[0]
        address     = args[1]
        #
        # flare-emu无法获知形参类型，它只是按一般ABI固定提取8个实参，遇上
        # 浮点传参或是Delphi那种调用约定，得自己提取实参
        #
        argv        = args[2]
        funcName    = args[3]
        userData    = args[4]
        try :
            #
            # 减1是减去eh所占形参
            #
            n   = GetFuncParamNum( func ) - 1
            #
            # 传递不定长形参
            #
            ret = func( eh, *argv[:n] )
            #
            # hook_free()会返回None
            #
            if ret is not None :
                eh.uc.reg_write( eh.regs["ret"], ret )
            return ret
        except :
            traceback.print_exc()
            sys.exit()
    #
    # end of func_wrapper
    #

    return func_wrapper
#
# end of PrivateHook
#

#
# 原始意图是让FILE结构保持同步，但实际上有巨坑等着我们。考虑在Windows上模
# 拟执行ELF的情形，两种OS的FILE结构定义并不相同。
#
def update_stream ( eh, stream ) :
    f   = eh.ftable[stream]
    buf = ctypes.create_string_buffer( 256 )
    assert( buf )
    cso.memmove( buf, f, 256 )
    #
    # 若想检查FILE结构，让下述语句生效
    #
    # hexdump.hexdump( buf )
    #
    # 写入Guest进程空间
    #
    eh.writeEmuMem( stream, buf.raw[:256] )
#
# end of update_stream
#

#
# 这是bluerust的版本
#
@PrivateHook
def hook_fread ( eh, ptr, size, nmemb, stream ) :
    f   = eh.ftable[stream]
    p   = ctypes.create_string_buffer( size * nmemb )
    n   = cso.fread( p, size, nmemb, f )
    update_stream( eh, stream )
    #
    # p.raw是bytes类型，ptr指向Guest进程空间
    #
    eh.writeEmuMem( ptr, p.raw[:size * n] )
    return n
#
# end of hook_fread
#

#
# 这是scz的版本，二者都可以
#
# @PrivateHook
# def hook_fread ( eh, ptr, size, nmemb, stream ) :
#     f   = eh.ftable[stream]
#     p   = cso.calloc( nmemb, size )
#     n   = cso.fread( p, size, nmemb, f )
#     update_stream( eh, stream )
#     eh.writeEmuMem( ptr, ctypes.string_at( p, size * n ) )
#     cso.free( p )
#     return n
# #
# # end of hook_fread
# #

@PrivateHook
def hook_fclose( eh, stream ):
    f   = eh.ftable[stream]
    ret = cso.fclose( f )
    update_stream( eh, stream )
    return ret
#
# end of hook_fclose
#

@PrivateHook
def hook_free ( eh, ptr ) :
    #
    # 这种返回None
    #
    return

def main () :

    #
    # 如果前面没有"@functools.wraps(func)"，此处将输出"func_wrapper"，反之
    # 输出"hook_fread"
    #
    # print( hook_fread.__name__ )
    #

    eh              = flare_emu.EmuHelper()

    #
    # 参看flare_emu.py，已经hook malloc()，不必自己干这事
    #
    # 目标binary中feof()、ferror()已inline展开，无法用eh.addApiHook()
    #

    #
    # 此处的hook_fread()函数原型是
    #
    # (eh, address, argv, funcName, userData)
    #
    # 此处写"_fread_unlocked"、"_free"也可以
    #
    eh.addApiHook( "fread_unlocked", hook_fread )
    eh.addApiHook( "free", hook_free )

    f               = cso.fopen( b"c:\\windows\\system.ini", b"rb" )
    if not f :
        print( "Unable to open file" )
        return
    print( "Opening", hex( int( f ) ) )

    #
    # 获取被模拟函数的起始地址
    #
    startAddr       = eh.analysisHelper.getNameAddr( "md5_stream" )
    assert( startAddr and startAddr != 0 )
    #
    # 准备被模拟函数的形参，在Guest进程空间分配内存
    #
    FILE            = eh.allocEmuMem( 256 )
    resblock        = eh.allocEmuMem( 0x20 )

    #
    # 这是自己临时增加的属性，用于存放文件句柄(FILE*)
    #
    eh.ftable       = dict()
    eh.ftable[FILE] = f

    update_stream( eh, FILE )

    #
    # 参看flare_emu.py
    #
    # 在x64/Win10上用寄存器传递两个实参，模拟/仿真执行md5_stream()
    #
    # eh.emulateRange( startAddr, skipCalls=False, instructionHook=PrivateInstructionHook, registers={'arg1':FILE, 'arg2':resblock} )
    #
    # 如果不需要跟踪到指令级别，注释掉上面这条语句，换用下面这条语句
    #
    eh.emulateRange( startAddr, skipCalls=False, registers={'arg1':FILE, 'arg2':resblock} )
    #
    # 获取模拟/仿真执行md5_stream()返回值
    #
    ret             = eh.getRegVal( "rax" )
    print( "ret =", ret )
    #
    # 获取MD5结果
    #
    md5             = eh.getEmuBytes( resblock, 0x20 )
    hexdump.hexdump( md5 )
    #
    # 本例只有一个句柄需要关闭
    #
    for k,v in eh.ftable.items() :
        if v :
            cso.fclose( v )
            print( "Closing", hex( int( v ) ) )
#
# end of main
#

if __name__ == '__main__' :
    main()


```

该脚本事实上有重大 BUG，但阴差阳错间 BUG 并未影响最终模拟执行结果，这事后面再细说。

Alt-F7 加载 emu_md5_stream.py

若在 update_stream() 中输出 FILE 结构的内容，将看到三次

3) 用 windbg 调试 IDA 对 emu_md5_stream.py 的加载执行

```
tasklist | findstr ida64
"X:\Green\Windows Kits\10\x64\Debuggers\x64\cdb.exe" -noinh -snul -hd -o -p <ida64 pid>

.prompt_allow +reg +ea +dis;rm 0xa
.load jsprovider.dll;.scriptload dbghelper_20201205.js
bp msvcrt!fopen "dx @$scriptContents.BreakEndWithAEx(@rcx,\"system.ini\",true,false);.if(@$t19==0x9c85130d){gc}"


```

当 ida64 进程试图打开 system.ini 时断下来。不要照搬调试命令，按原始意图换成自己环境中的等价命令。

为什么拦截 msvcrt!fopen() 呢？首先 emu_md5_stream.py 中通过 ctypes 调过该函数，其次 Process Monitor 调用栈回溯中看到它。为什么用 windbg？因为 Process Monitor 看不到更多细节，比如 fopen() 返回值、FILE 结构内容等等。

```
> da @rcx
0000025e`4883bd10  "c:\windows\system.ini"

> kpn
 # Child-SP          RetAddr           Call Site
00 000000b9`6f3f9578 00007fff`1b204461 msvcrt!fopen
01 000000b9`6f3f9580 00007fff`1b20418d libffi_7!ffi_prep_go_closure+0x71
02 000000b9`6f3f95b0 00007fff`1b204042 libffi_7!ffi_call_go+0x13d
03 000000b9`6f3f9600 00007fff`16472bd2 libffi_7!ffi_call+0x12
04 000000b9`6f3f9640 00007fff`164728c8 _ctypes+0x2bd2
05 000000b9`6f3f97a0 00007fff`164725ab _ctypes+0x28c8
06 000000b9`6f3f98d0 00007fff`1015410c _ctypes+0x25ab
07 000000b9`6f3f9980 00007fff`101d3047 python39!PyObject_MakeTpCall+0x14c
...
20 000000b9`6f3face0 00000000`762255c9 python39!PyObject_CallFunctionObjArgs+0x2d
21 000000b9`6f3fad10 00000000`762215ba idapython3_64+0x55c9
22 000000b9`6f3fb1c0 00007ff6`2af9c086 idapython3_64+0x15ba
23 000000b9`6f3fb240 00007ff6`2af9ef4a ida64_exe+0x18c086
...
29 000000b9`6f3fb5d0 00000000`767c81b2 Qt5Core!QT::QMetaObject::activate+0x591
...
41 000000b9`6f3ffe40 00007fff`1eb47034 ida64_exe+0x229002
42 000000b9`6f3ffe80 00007fff`1fa9d0d1 KERNEL32!BaseThreadInitThunk+0x14
43 000000b9`6f3ffeb0 00000000`00000000 ntdll!RtlUserThreadStart+0x21


```

让 fopen() 完成，查看其返回值及 FILE 结构:

```
> g poi(@rsp)
> r rax
rax=00007fff1e33fa90

> db @rax l 0n256
00007fff`1e33fa90  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007fff`1e33faa0  00 00 00 00 00 00 00 00-01 00 00 00 03 00 00 00  ................
...
00007fff`1e33fb80  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................


```

拦截 fclose()，查看关闭前的 FILE 结构:

```
> bp msvcrt!fclose ".if(@rcx!=0x7fff1e33fa90){gc}.else{db @rcx l 0x100}"
> g
00007fff`1e33fa90  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007fff`1e33faa0  00 00 00 00 00 00 00 00-11 00 00 00 03 00 00 00  ................
...
00007fff`1e33fb80  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................


```

fopen() 返回值、FILE 结构与 IDA 中输出相符。

☆ FILE 结构

1) FILEStructureTest.c

```
#if 0

x64/RedHat gcc 4.8.5

gcc -Wall -pipe -O3 -s -o FILEStructureTest_linux FILEStructureTest.c
gcc -Wall -pipe -O0 -g -o FILEStructureTest_linux FILEStructureTest.c

$ ./FILEStructureTest_linux FILEStructureTest.c
sizeof( FILE ) = 216
OFFSETOF( FILE*, _flags ) = 0

Visual Studio 2019 社区版

cl.exe FILEStructureTest.c /FeFILEStructureTest_windows.exe /Zi /FdFILEStructureTest_windows.pdb /nologo /Os /Gs65536 /W4 /WX /D "WIN32" /D "NDEBUG" /D "_CONSOLE" /link /machine:x64 /pdbaltpath:FILEStructureTest_windows.pdb /RELEASE /opt:ref
editbin.exe /dynamicbase:no FILEStructureTest_windows.exe

"/pdbaltpath"是link.exe的参数，使得将来.exe中的.pdb只有指定名字，而不是缺
省的绝对路径，减少信息泄露。

editbin.exe禁止对FILEStructureTest_windows.exe启用ASLR，便于调试。

$ FILEStructureTest_windows.exe FILEStructureTest.c
sizeof( FILE ) = 8
OFFSETOF( FILE*, _Placeholder ) = 0

#endif

/*
 * 抑制VS 2019关于fopen()的安全警告
 */
#define _CRT_SECURE_NO_WARNINGS

#include <stdio.h>
#include <stdlib.h>

#define OFFSETOF(TYPE, MEMBER)  ((size_t)&((TYPE)0)->MEMBER)
#define BLOCKSIZE               32768

static void hexdump
(
    FILE           *out,
    unsigned char  *in,
    size_t          insize,
    size_t          count,
    size_t          offset
)
{
    size_t          k, j, i;

    if ( insize <= 0 || count <= 0 || NULL == in || NULL == out )
    {
        return;
    }
    i       = 0;
    for ( k = insize / count; k > 0; k--, offset += count )
    {
        fprintf( out, "%016zx:", offset );
        for ( j = 0; j < count; j++, i++ )
        {
            fprintf( out, " %02x", in[i] );
        }
        fprintf( out, "  " );
        i  -= count;
        for ( j = 0; j < count; j++, i++ )
        {
            if ( ( in[i] >= ' ' ) && ( in[i] < 0x7f ) )
            {
                fprintf( out, "%c", in[i] );
            }
            else
            {
                fprintf( out, "." );
            }
        }
        fprintf( out, "\n" );
    }  /* end of for */
    k       = insize - i;
    if ( k <= 0 )
    {
        return;
    }
    fprintf( out, "%016zx:", offset );
    for ( j = 0 ; j < k; j++, i++ )
    {
        fprintf( out, " %02x", in[i] );
    }
    i      -= k;
    for ( j = count - k; j > 0; j-- )
    {
        fprintf( out, "   " );
    }
    fprintf( out, "  " );
    for ( j = 0; j < k; j++, i++ )
    {
        if ( ( in[i] >= ' ' ) && ( in[i] < 0x7f ) )
        {
            fprintf( out, "%c", in[i] );
        }
        else
        {
            fprintf( out, "." );
        }
    }
    fprintf( out, "\n" );
    return;
}  /* end of hexdump */

#ifdef WIN32
#pragma warning( push )
#pragma warning( disable : 4100 )
#endif

/*
 * VS 2019如下告警被抑制
 *
 * FILEStructureTest.c(num): warning C4100: 'argc': unreferenced formal parameter
 *
 * 聚焦测试，未做各种安全检查
 */
int main ( int argc, char * argv[] )
{
    char           *filename    = argv[1];
    FILE           *stream      = fopen( filename, "rb" );
    size_t          sum;
    unsigned char  *buffer      = malloc( BLOCKSIZE + 72 );
    size_t          offset      = 0;

    while ( 1 )
    {
        size_t  n;

        sum     = 0;
        while ( 1 )
        {
            n       = fread( buffer + sum, 1, BLOCKSIZE - sum, stream );
            sum    += n;
            if ( sum == BLOCKSIZE )
                break;
            if ( n == 0 )
            {
                if ( ferror( stream ) )
                {
                    free( buffer );
                    return 1;
                }
                goto process_partial_block;
            }
            if ( feof( stream ) )
                goto process_partial_block;
        }
        hexdump( stdout, buffer, BLOCKSIZE, 16, offset );
        offset += BLOCKSIZE;
    }

process_partial_block:

    if ( sum > 0 )
        hexdump( stdout, buffer, sum, 16, offset );
    free( buffer );
    fclose( stream );

#ifdef WIN32
    /*
     * C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\ucrt\corecrt_wstdio.h
     * C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\ucrt\mbstring.h
     */
    printf( "sizeof( FILE ) = %zu\n", sizeof( FILE ) );
    printf( "OFFSETOF( FILE*, _Placeholder ) = %zu\n", OFFSETOF( FILE*, _Placeholder ) );
#else
    /*
     * /usr/include/libio.h
     */
    printf( "sizeof( FILE ) = %zu\n", sizeof( FILE ) );
    printf( "OFFSETOF( FILE*, _flags ) = %zu\n", OFFSETOF( FILE*, _flags ) );
#endif

    return 0;
}

#ifdef WIN32
#pragma warning( pop )
#endif


```

2) Linux 的 FILE 结构

vi /usr/include/libio.h

```
struct _IO_FILE {
  int _flags;           /* High-order word is _IO_MAGIC; rest is flags. */
#define _IO_file_flags _flags
...
};


```

ferror()、feof() 这些运行时库函数屏蔽了 FILE 结构细节，正经编程时不需要了解 FILE 结构细节，更不应该直接操作 FILE 结构。但是，对于 emu_md5_stream.py 这种场景，逆向工程时不得不直面 FILE 结构细节。

反汇编 md5sum 时注意到 ferror()、feof() 已 inline 展开:

```
/*
 * ferror()
 */
0000000000403840 F6 45 00 20                 test    byte ptr [rbp+0], 20h


```

```
/*
 * feof()
 */
00000000004037D0 F6 45 00 10                 test    byte ptr [rbp+0], 10h


```

此时无法通过 eh.addApiHook() 安装 Hook 模拟 ferror()、feof()。bluerust 直接复制 Host 进程空间的 FILE 结构到 Guest 进程空间，就是应对前述 inline 展开。就 ferror()、feof() 而言，它们只操作偏移 0 处的_flags 成员的最低字节，可以只向 Guest 空间复制 1 字节，而不是复制整个 FILE 结构。

反汇编 FILEStructureTest_linux，注意到 ferror()、feof() 以动态链接的运行时库函数方式出现，并未 inline 展开，估计静态编译时有可能 inline 展开。

3) Windows 的 FILE 结构

查看 Visual Studio 2019 社区版中这两个头文件:

```
C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\ucrt\corecrt_wstdio.h
C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\ucrt\mbstring.h


```

```
#ifndef _FILE_DEFINED
    #define _FILE_DEFINED
    typedef struct _iobuf
    {
        void* _Placeholder;
    } FILE;
#endif


```

而在 VC 6 时代，是这样定义的:

```
#ifndef _FILE_DEFINED
struct _iobuf {
        char *_ptr;
        int   _cnt;
        char *_base;
        int   _flag;
        int   _file;
        int   _charbuf;
        int   _bufsiz;
        char *_tmpfname;
        };
typedef struct _iobuf FILE;
#define _FILE_DEFINED
#endif


```

VS 2019 的定义非常具有迷惑性，起初我以为_Placeholder 指向一个更不透明的内部数据结构。但反汇编 FILEStructureTest_windows.exe 后发现我想多了，事实上 FILE 结构仍同 VC 6 时代。_Placeholder 真地就如其名所言，只是个占位成员。微软认为 FILE 结构是未文档化的，不应该直接操作它，包括 sizeof(FILE) 这种都是不应有的操作，干脆在头文件中彻底屏蔽了 FILE 结构细节，只能通过反汇编观察其内部细节。

x64 的_flag 成员位于偏移 0x14，占 4 字节。ferror()、feof() 会访问_flag 成员:

```
/*
 * ferror()
 */
0000000140003241 8B 41 14        mov     eax, [rcx+14h]
0000000140003244 C1 E8 04        shr     eax, 4
0000000140003247 83 E0 01        and     eax, 1


```

```
/*
 * feof()
 */
0000000140003215 8B 41 14        mov     eax, [rcx+14h]
0000000140003218 C1 E8 03        shr     eax, 3
000000014000321B 83 E0 01        and     eax, 1


```

Windows 的_flag 成员偏移不同于 Linux 的_flags 成员偏移，并且 ferror()、feof() 具体访问的二进制位也不一样，前者整体左移了一位。

4) emu_md5_stream.py 中 update_stream()

考虑这样一种场景，md5_stream() 本来是在 x64/Linux 中运行的，现在在 x64/Win10 中用 flare-emu 模拟执行它。

Linux、Windows 的 FILE 结构完全不一样，像 emu_md5_stream.py 中 update_stream() 那样简单复制 FILE 结构，没有意义，更有可能导致不可预期的混乱。

单就 md5_stream() 这一场景而言，应该在 update_stream() 中完成 Windows _flag 到 Linux _flags 的映射，这才是完备有效的模拟。此次并未涉及 FILE 结构其他成员，很容易重新实现 update_stream()，留给读者自己完成。

5) 为什么 emu_md5_stream.py 有 BUG 仍然得到正确结果

flare-emu 可以在汇编指令级别跟踪，下列代码会显示每一条被模拟执行的汇编指令:

```
def PrivateInstructionHook ( unicornObject, address, instructionSize, userData ) :
    print( "=> %#x [%s]" % ( address, ida_lines.tag_remove( ida_lines.generate_disasm_line( address ) ) ) )

eh.emulateRange( startAddr, skipCalls=False, instructionHook=PrivateInstructionHook, registers={'arg1':FILE, 'arg2':resblock} )


```

输出类似这样:

```
=> 0x4037ba [call    _fread_unlocked]
=> 0x4037bf [add     rbx, rax]
=> 0x4037c2 [cmp     rbx, 8000h]
=> 0x4037c9 [jz      short loc_403818]
=> 0x4037cb [test    rax, rax]
=> 0x4037ce [jz      short loc_403840]
=> 0x403840 [test    byte ptr [rbp+0], 20h]
=> 0x403844 [jz      short loc_4037D6]


```

通过这招，可以看到发生了什么:

```
int __fastcall md5_stream(FILE *stream, void *resblock)
{
...
  if ( buffer )
  {
...
    while ( 1 )
    {
      while ( 1 )
      {
        n = fread_unlocked((char *)buffer + sum, 1uLL, 0x8000 - sum, stream);
        sum += n;
        if ( sum != 0x8000 )
          break;
        md5_process_block(buffer, 0x8000LL, ctx);
        sum = 0LL;
      }
      /*
       * 第二次到达此处时，n为0，break，跳去检查ferror()。
       */
      if ( !n )
        break;
      /*
       * 缓冲区够大，一次就读完了整个文件。但模拟执行时feof()判断不为真，
       * 因为偏移0处的字节值为0。继续while循环。
       */
      if ( stream->_flags & 0x10 )
        goto process_partial_block;
    }
    /*
     * 模拟执行时ferror()判断不为真，因为偏移0处的字节值为0。幸运地离开。
     */
    if ( stream->_flags & 0x20 )
    {
      free(buffer);
      return 1;
    }
process_partial_block:
    if ( sum )
      md5_process_bytes(buffer, sum, ctx);
    md5_finish_ctx(ctx, (__int64)resblock);
    free(buffer);
    ret = 0;
  }
  return ret;
}


```

模拟执行时 feof()、ferror() 依次访问了错误的_flags 成员，由于其固定为 0，对于 md5_stream() 具体实现来说，未影响最终 MD5 结果。

事实上 bluerust 最早实现的 emu_md5_stream.py 连 update_stream() 都没有，也求得正确 MD5 值。然后他在静态审计中意识到应该有 update_stream()，就是前面演示的版本。即使这样，仍然在特定场景中存在 BUG。emu_md5_stream.py 误打误撞逃过一劫。

后来 bluerust 解释，FILE 结构的平台差异我是有想到的，我对 FILE 结构非常熟悉，但我当时以为那个. i64 对应一个 PE 文件，第二觉得要映射结构好烦啊，作为 DEMO，就别折腾了。

☆ 后记

被模拟代码如果有 I/O、操作系统相关的动作或其他更复杂的什么，是模拟执行还是动态调试，需要具体情况具体分析，选用较优解。函数调用可以 Hook，inline 展开则涉及内部数据结构，如果非要模拟执行，务必深刻理解上下文后谨慎处理。

flare-emu 小巧精悍，相比之下另一些模拟框架显得重型，是否适用于你的目标场景需要另行评估。不要相信它们宣称的 NB，在逆向工程的世界里，永远有一些意想不到的坑等着你。

flare-emu 的 iterate() 功能也很实用。举个例子，目标 binary 中有个私有解码函数对 binary 中混淆存放的字符串进行解码，iterate() 通过分析交叉引用信息自动定位该解码函数的主调位置，根据 ABI 自动分析、抽取形参，自动调用我们提供的回调函数；回调函数中可以调用解码函数获取反混淆后的明文字符串，在 IDA 中自动增加注释。这种功能都有重大假设，ABI 就是其中之一，当目标 binary 不满足这些先验假设时，就需要其他 Hacking。本文未就 iterate() 进行示例，相比之下 emu_md5_stream.py 已把最困难的部分示例清楚了。

相关测试用例:

```
http://scz.617.cn:8/python/202012021733.7z


```

如果以上帝视角展开，本文将精简许多，但学习过程中林林总总的坑比精简的结论更有价值。

☆ 参考资源

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VbJOzZqovPN4JBr5jf0dnKdS2fCr4BvO2pkibaQAEkC8vo3IyibKgnOibtj9MddjukZWnfwRuibCviblVWHOabcEWdQ/640?wx_fmt=png)