> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271253.htm)

> [原创]Python 逆向——Pyinstaller 逆向

Pyinstaller
===========

最常见的打包库，打包后可以通过 Extractor 工具或者 pyinstaller 内置的 achieve_viewer.py 直接解包，使用—key 参数可以使其具有一定程度的反逆向能力，但由于其本身是个开源的库，也能找到其加密的原理，所以不是很难逆向

无 - key 参数的逆向
=============

得到可执行文件，使用 Extractor 工具进行解包

```
python pyinstxtractor.py xx.exe

```

将会得到一个文件夹，里面包含主程序所引用的所有库以及代码的 pyc 文件

 

![](https://bbs.pediy.com/upload/attach/202201/927366_U4MV6DK9CAMRMEJ.png)

 

pyinstaller 有一个奇怪的地方，它会把主函数 pyc 文件的文件头进行更改，这里我们就可以用到上面那个文件夹里面一定会有的 struct.pyc 文件，这个文件的文件头一般是不会更改的，将这个文件的文件头进行复制然后更改主函数 pyc 文件头就可以了。

有 - key 参数的逆向
=============

逆向前的源码分析
--------

由于 Pyinstaller 是个开源的包，这也给我们逆向提供了便利

 

在官方给出的用法中，有给出 - key 这个参数，说是可以将文件 pyc 进行一定的压缩加密，以防止被逆向

 

Pyinstaller 这个库本身的打包原理大概就是先将 py 编译成 pyc，然后部分压缩成 pyz，程序再通过对 pyc 和 pyz 的调用

 

这个是 pyinstaller 的打包过程语句

```
➜ pyinstaller.exe -F test.py
150 INFO: PyInstaller: 4.3
150 INFO: Python: 3.8.9
150 INFO: Platform: Windows-10-10.0.22000-SP0
152 INFO: wrote H:\My_CTF_Tools\Python_Reverse\pyinstaller\test.spec
158 INFO: UPX is not available.
205 INFO: Extending PYTHONPATH with paths
['H:\\My_CTF_Tools\\Python_Reverse\\pyinstaller',
 'H:\\My_CTF_Tools\\Python_Reverse\\pyinstaller']
300 INFO: checking Analysis
300 INFO: Building Analysis because Analysis-00.toc is non existent
300 INFO: Initializing module dependency graph...
304 INFO: Caching module graph hooks...
314 WARNING: Several hooks defined for module 'win32ctypes.core'. Please take care they do not conflict.
335 INFO: Analyzing base_library.zip ...
2867 INFO: Processing pre-find module path hook distutils from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks\\pre_find_module_path\\hook-distutils.py'.
2869 INFO: distutils: retargeting to non-venv dir 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib'
6284 INFO: Caching module dependency graph...
6460 INFO: running Analysis Analysis-00.toc
6464 INFO: Adding Microsoft.Windows.Common-Controls to dependent assemblies of final executable
  required by c:\users\lenovo\appdata\local\programs\python\python38\python.exe
7015 INFO: Analyzing H:\My_CTF_Tools\Python_Reverse\pyinstaller\test.py
7022 INFO: Processing module hooks...
7022 INFO: Loading module hook 'hook-difflib.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7025 INFO: Loading module hook 'hook-distutils.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7027 INFO: Loading module hook 'hook-distutils.util.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7028 INFO: Loading module hook 'hook-encodings.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7113 INFO: Loading module hook 'hook-heapq.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7115 INFO: Loading module hook 'hook-lib2to3.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7189 INFO: Loading module hook 'hook-multiprocessing.util.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7191 INFO: Loading module hook 'hook-pickle.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7193 INFO: Loading module hook 'hook-sysconfig.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7195 INFO: Loading module hook 'hook-xml.etree.cElementTree.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7197 INFO: Loading module hook 'hook-xml.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7275 INFO: Loading module hook 'hook-_tkinter.py' from 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks'...
7564 INFO: checking Tree
7574 INFO: Building Tree because Tree-00.toc is non existent
7576 INFO: Building Tree Tree-00.toc
7747 INFO: checking Tree
7747 INFO: Building Tree because Tree-01.toc is non existent
7747 INFO: Building Tree Tree-01.toc
7857 INFO: checking Tree
7857 INFO: Building Tree because Tree-02.toc is non existent
7857 INFO: Building Tree Tree-02.toc
7875 INFO: Looking for ctypes DLLs
7895 INFO: Analyzing run-time hooks ...
7896 INFO: Including run-time hook 'c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\hooks\\rthooks\\pyi_rth_multiprocessing.py'
7902 INFO: Looking for dynamic libraries
8061 INFO: Looking for eggs
8062 INFO: Using Python library c:\users\lenovo\appdata\local\programs\python\python38\python38.dll
8062 INFO: Found binding redirects:
[]
8066 INFO: Warnings written to H:\My_CTF_Tools\Python_Reverse\pyinstaller\build\test\warn-test.txt
8099 INFO: Graph cross-reference written to H:\My_CTF_Tools\Python_Reverse\pyinstaller\build\test\xref-test.html
8114 INFO: checking PYZ
8115 INFO: Building PYZ because PYZ-00.toc is non existent
8115 INFO: Building PYZ (ZlibArchive) H:\My_CTF_Tools\Python_Reverse\pyinstaller\build\test\PYZ-00.pyz
8658 INFO: Building PYZ (ZlibArchive) H:\My_CTF_Tools\Python_Reverse\pyinstaller\build\test\PYZ-00.pyz completed successfully.
8673 INFO: checking PKG
8673 INFO: Building PKG because PKG-00.toc is non existent
8673 INFO: Building PKG (CArchive) PKG-00.pkg
10402 INFO: Building PKG (CArchive) PKG-00.pkg completed successfully.
10406 INFO: Bootloader c:\users\lenovo\appdata\local\programs\python\python38\lib\site-packages\PyInstaller\bootloader\Windows-64bit\run.exe
10407 INFO: checking EXE
10407 INFO: Building EXE because EXE-00.toc is non existent
10407 INFO: Building EXE from EXE-00.toc
10473 INFO: Copying icons from ['c:\\users\\lenovo\\appdata\\local\\programs\\python\\python38\\lib\\site-packages\\PyInstaller\\bootloader\\images\\icon-console.ico']
10522 INFO: Writing RT_GROUP_ICON 0 resource with 104 bytes
10522 INFO: Writing RT_ICON 1 resource with 3752 bytes
10522 INFO: Writing RT_ICON 2 resource with 2216 bytes
10523 INFO: Writing RT_ICON 3 resource with 1384 bytes
10523 INFO: Writing RT_ICON 4 resource with 37019 bytes
10524 INFO: Writing RT_ICON 5 resource with 9640 bytes
10524 INFO: Writing RT_ICON 6 resource with 4264 bytes
10524 INFO: Writing RT_ICON 7 resource with 1128 bytes
10531 INFO: Updating manifest in H:\My_CTF_Tools\Python_Reverse\pyinstaller\build\test\run.exe.7_01cl0t
10592 INFO: Updating resource type 24 name 1 language 0
10599 INFO: Appending archive to EXE H:\My_CTF_Tools\Python_Reverse\pyinstaller\dist\test.exe
12711 INFO: Building EXE from EXE-00.toc completed successfully.

```

可以看到对 ctypes 的调用，pyz 的生成等等

 

生成之后的 build 文件夹如下

 

![](https://bbs.pediy.com/upload/attach/202201/927366_V63V8M2QRFQ23FV.png)

 

那个 Base_Library.zip 里面都是库文件的 pyc

 

然后我们在 pyinstaller 库中找到 archieve 这么一个文件夹，里面有一个 pyz_crypto.py 文件，是对 pyz 文件的加密代码

```
#-----------------------------------------------------------------------------
# Copyright (c) 2005-2021, PyInstaller Development Team.
#
# Distributed under the terms of the GNU General Public License (version 2
# or later) with exception for distributing the bootloader.
#
# The full license is in the file COPYING.txt, distributed with this software.
#
# SPDX-License-Identifier: (GPL-2.0-or-later WITH Bootloader-exception)
#-----------------------------------------------------------------------------
 
import os
 
BLOCK_SIZE = 16
 
class PyiBlockCipher(object):
    """
    This class is used only to encrypt Python modules.
    """
    def __init__(self, key=None):
        assert type(key) is str
        if len(key) > BLOCK_SIZE:
            self.key = key[0:BLOCK_SIZE]
        else:
            self.key = key.zfill(BLOCK_SIZE)
        assert len(self.key) == BLOCK_SIZE
 
        import tinyaes
        self._aesmod = tinyaes
 
    def encrypt(self, data):
        iv = os.urandom(BLOCK_SIZE)
        return iv + self.__create_cipher(iv).CTR_xcrypt_buffer(data)
 
    def __create_cipher(self, iv):
        # The 'AES' class is stateful, this factory method is used to
        # re-initialize the block cipher class with each call to xcrypt().
        return self._aesmod.AES(self.key.encode(), iv)

```

可以看出，他是使用的 tinyAES 库对 pyc 文件进行块加密，块大小为 16byte

[](#方法一：利用源码的脚本法)方法一：利用源码的脚本法
-----------------------------

这个方法的灵感来源于 github 上大佬 [extremecoders-re](https://github.com/extremecoders-re)

 

链接：[https://github.com/extremecoders-re/pyinstxtractor/wiki/Frequently-Asked-Questions](https://github.com/extremecoders-re/pyinstxtractor/wiki/Frequently-Asked-Questions)

 

因为我们已经根据源码知道了 pyz 加密方式和加密算法，所以根据解包后 pyc 文件提供的一系列参数，很容易就能编写出对应的解密脚本。

 

首先用 pyinstxtractor 工具对文件进行解包 (需软件对应相同的 python 大版本，否则无法得到 pyz 文件)，得到未被加密的部分 pyc 文件和加密的 pyz 文件，其中之一就有 archive.pyc, 我们可以通过 archive.pyc 文件得到加密过程, crypto_key 文件得到具体 key 参数

 

![](https://bbs.pediy.com/upload/attach/202201/927366_YGDQ4978AHKHCM4.png)

 

然后根据主函数 pyc 反编译内容我们又找到了需要找的包

 

![](https://bbs.pediy.com/upload/attach/202201/927366_BMDNBKSNKX94Q7F.png)

 

这个时候就只需要按照加密的方式进行解密就可以了，加密部分为 Cipher(脚本来源于 archive.pyc 的反编译)

```
import marshal, struct, sys, zlib, _thread as thread
CRYPT_BLOCK_SIZE = 16
PYZ_TYPE_MODULE = 0
PYZ_TYPE_PKG = 1
PYZ_TYPE_DATA = 2
PYZ_TYPE_NSPKG = 3
 
......
 
#关键加密代码
class Cipher(object):
    __doc__ = '\n    This class is used only to decrypt Python modules.\n    '
 
    def __init__(self):
                #引入密钥
        import pyimod00_crypto_key
        key = pyimod00_crypto_key.key
        if not type(key) is str:
            raise AssertionError
        elif len(key) > CRYPT_BLOCK_SIZE:
            self.key = key[0:CRYPT_BLOCK_SIZE]
        else:
            self.key = key.zfill(CRYPT_BLOCK_SIZE)
        assert len(self.key) == CRYPT_BLOCK_SIZE
        import tinyaes
        self._aesmod = tinyaes
        del sys.modules['tinyaes']
 
        #利用TinyAES进行加密
    def __create_cipher(self, iv):
        return self._aesmod.AES(self.key.encode(), iv)
 
        #提供的解密算法，原义是想让pyz文件正常解压运行，在这里可以被我们利用
    def decrypt(self, data):
        cipher = self._Cipher__create_cipher(data[:CRYPT_BLOCK_SIZE])
        return cipher.CTR_xcrypt_buffer(data[CRYPT_BLOCK_SIZE:])
 
......

```

解密脚本

```
#!/usr/bin/env python3
import tinyaes
import zlib
 
CRYPT_BLOCK_SIZE = 16
 
# 从crypt_key.pyc获取key，也可自行反编译获取
key = bytes('MySup3rS3cr3tK3y', 'utf-8')
 
inf = open('baby_core.pyc.encrypted', 'rb') # 打开加密文件
outf = open('baby_core.pyc', 'wb') # 输出文件
 
# 按加密块大小进行读取
iv = inf.read(CRYPT_BLOCK_SIZE)
 
cipher = tinyaes.AES(key, iv)
 
# 解密
plaintext = zlib.decompress(cipher.CTR_xcrypt_buffer(inf.read()))
 
# 补pyc头(最后自己补也行)
outf.write(b'\x55\x0d\x00\x00\0\0\0\0\0\0\0\0\0\0\0\0')
 
# 写入解密数据
outf.write(plaintext)
 
inf.close()
outf.close()

```

[](#方法二：改源码法)方法二：改源码法
---------------------

这个方法是跟某大师傅学的，先是之前就在翻找源码的时候看到了 pyinstaller 自己提供了内置的 archieve_view.py 这个东东，而且能用来解包，也曾经尝试过用一下，具体如图：

 

![](https://bbs.pediy.com/upload/attach/202201/927366_C7Y689BHDBA3NZ3.png)

 

我们将 baby.exe 放进这里然后用内置文件解包试试

 

![](https://bbs.pediy.com/upload/attach/202201/927366_XP86EXYYS74RDYP.png)

 

然后我们就能看到神奇的一幕了，exe 文件内容被展示出来了

 

看一下 archieve_view.py 的源码，其实就能理解了

 

这里只放一下操作相关的代码

```
......
 
while 1:
        try:
            toks = stdin_input('? ').split(None, 1)
        except EOFError:
            # Ctrl-D
            print(file=sys.stderr)  # Clear line.
            break
        if not toks:
            usage()
            continue
        if len(toks) == 1:
            cmd = toks[0]
            arg = ''
        else:
            cmd, arg = toks
        cmd = cmd.upper()
        if cmd == 'U':
            if len(stack) > 1:
                arch = stack[-1][1]
                del stack[-1]
            name, arch = stack[-1]
            show(name, arch)
        elif cmd == 'O':
            if not arg:
                arg = stdin_input('open name? ')
            arg = arg.strip()
            try:
                arch = get_archive(arg)
            except NotAnArchiveError as e:
                print(e, file=sys.stderr)
                continue
            if arch is None:
                print(arg, "not found", file=sys.stderr)
                continue
            stack.append((arg, arch))
            show(arg, arch)
        elif cmd == 'X':
            if not arg:
                arg = stdin_input('extract name? ')
            arg = arg.strip()
            data = get_data(arg, arch)
            if data is None:
                print("Not found", file=sys.stderr)
                continue
            filename = stdin_input('to filename? ')
            if not filename:
                print(repr(data))
            else:
                with open(filename, 'wb') as fp:
                    fp.write(data)
        elif cmd == 'Q':
            break
        else:
            usage()
 
......

```

总共三个操作

*   U 展示列表
*   O 打开文件
*   X 解包文件

那我们找到系统主函数的位置（从上文得知在 pyz 中）

 

![](https://bbs.pediy.com/upload/attach/202201/927366_9VF92SXC2PN3CEB.png)

 

找到了，直接解压是行不通的，有密码

 

![](https://bbs.pediy.com/upload/attach/202201/927366_YEBXTSQ87RYBV5W.png)

 

这里直接解压 python 报错提示头文件检查错误

 

当然了，因为文件被 AES 加密了

 

所以得从解 AES 入手，恰好我们有加密函数，甚至还有其解密函数

 

那就好办了

 

回头再看一下 archieve_view.py 的源码，找到 get_data 函数

```
def get_data(name, arch):
    if isinstance(arch.toc, dict):
        (ispkg, pos, length) = arch.toc.get(name, (0, None, 0))
        if pos is None:
            return None
        with arch.lib:
            arch.lib.seek(arch.start + pos)
            return zlib.decompress(arch.lib.read(length))
    ndx = arch.toc.find(name)
    dpos, dlen, ulen, flag, typcd, name = arch.toc[ndx]
    x, data = arch.extract(ndx)
    return data

```

很明显，里面调用的 decompass 就是解包的过程，里面调用的是 read 方法，如果我们 read 之后再来个 decrypt，再用 archieve_view.py 不就跑出来了吗

 

将上面那个 Cipher 类加入 archieve_view.py 文件中

 

然后将 get_data 函数下的 zlib.decompress(arch.lib.read(length)) 改为 zlib.decompress(cipher.decrypt(arch.lib.read(length)))，使其读取完直接解密再 decompass, 不就不会头文件错误了嘛

 

（别忘了将 key 导入，还有 blocksize 的宏定义）

 

运行，正常导出

 

![](https://bbs.pediy.com/upload/attach/202201/927366_77GN5NJRDUANUB8.png)

 

最后，插入 16 字节的头文件，反编译就出来源码了

 

![](https://bbs.pediy.com/upload/attach/202201/927366_M8V39NFGDWNER9D.png)

[【公告】看雪团队招聘安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

[#软件保护](forum-4-1-3.htm) [#加密算法](forum-4-1-5.htm)