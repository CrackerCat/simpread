> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-264287.htm)

> [原创]python exe 文件解包方法总结

_注：_  
_除特别说明外，下面使用的 python 版本均为 3.8.6_  
_文章中提到的相关工具均已包含在附件中。_

[](#一、步骤)一、步骤
-------------

### 1. exe → pyc

#### 方法 1：pyinstxtractor.py

执行`python pyinstxtractor.py <待解包文件名>` ，如果成功，即可获得`<待解包文件名>_extracted` 文件夹。

 

_注：执行时会提示 python 版本问题，想要正常解包必须使用正确的 python 版本_。

#### 方法 2：archive_viewer.py

执行`python archive_viewer.py <待解包文件名>` ，会打印 EXE 文件中包含的所有文件信息

 

使用`x <文件名>`命令将想要提取出的文件提取出来，`q` 命令退出。

#### 两者的区别

方法 1 可以一次性提取出所有文件，方法 2 只能逐个提取文件。但是在个人使用时，方法 2 的成功率相对较高。可以先尝试用方法 1，失败后用方法 2。

### 2. pyc → py

步骤 1 获得的文件是 pyc 文件，还需要进一步反编译获得 py 文件。

 

_注：我遇到过直接获得 py 文件的情况，所以在反编译之前可以先查看一下是不是已经成功了。_

#### 2.1 pyc 文件恢复

_注：最新版 pyinstxtractor.py 支持自动恢复 pyc，但是经实验不能保证准确性。或者说需要使用准确的 python 版本才行。_

 

在将 python 文件打包成 exe 文件的过程中，会抹去 pyc 文件前面的部分信息，所以在反编译之前需要检查并添加上这部分信息。

 

抹去的信息内容可以从`struct`文件中获取：

 

struct 文件：  
![](https://bbs.pediy.com/upload/attach/202012/600394_RVR8EMZYXH2BVCW.png)

 

pyc 文件：  
![](https://bbs.pediy.com/upload/attach/202012/600394_EZEJWJ4J4PC4JN8.png)

 

**Q1：需要添加多少字节？**

 

多个参考文章中提到的添加字节数都不一致，这应该与使用的 python 版本有关。但是在已知的几个例子中，可以看出`pyc`文件开头的几个字节与`struct`文件中的字节是一样的。比如说上图中`pyc`文件是以`E3 00 00 00`开头，而这部分字节就和`struct`文件的第二行起始字节相同。

 

因此在此例中，需要复制添加的字节就是`struct`文件中第一行的 16 个字节

 

**Q2：添加的方法是什么？**

 

在 010editor 中，选择`Edit→Insert/Overwrite→InsertBytes`，`Start Address`填 0，`Size`填 16。

 

然后将字节复制进去即可。

#### 2.2 反编译

**反编译工具**

1.  Easy Python Decompiler
    
    这是一个 GUI 界面的可执行文件，下载下来直接用就可以，但是并不是每次都能成功，有些 magic value 不能识别。
    
2.  uncompyle6
    
    该工具需要使用`pip`安装，使用脚本执行。成功率较高。
    

**反编译提示 magic value 有问题怎么办？**

 

在上面我们添加的 16 个字节中，前四个字节表示的就是 magic value，其中前两个字节表示的是 python 的版本号，一般来说 magic value 的问题就是版本号有问题，编译工具没有识别出来该版本号表示的 python 版本。

 

如果使用的是 Easy Python Decompiler，那么就可以直接转用`uncompyle6`了。

 

如果用的已经是`uncompyle6`，那么需要先看一下它可以识别的版本号都有哪些，这个信息可以在`xdis`包的`magics.py`文件中找到，具体方法如下：

1.  命令行输入`pip install xdis`，查看其安装位置
    
    ![](https://bbs.pediy.com/upload/attach/202012/600394_BJEDVBYG9HPR6VK.png)
    
2.  到该目录下，打开 xdis 文件夹下的 magics.py 文件
    
3.  确定需要识别的版本号
    
    就是之前添加的 16 个字节的前两个字节，此例中为`0D42`，需要转换为十进制，就是`3394`。
    
4.  查看 magics.py 中是否有该版本号。
    
    我这里一开始没有发现这个版本号
    
    ![](https://bbs.pediy.com/upload/attach/202012/600394_YKH89PB6D8FX66S.png)
    

**我的解决方法**

 

我在谷歌上搜索了`int2magic(3394)`，找到了[这个 Github 项目](https://github.com/rocky/python-xdis)。

 

下载下来，按照介绍进行安装。

 

在执行`pip install -e .`的时候，提示我`xdis`和`uncompyle6`的版本不匹配，于是卸载了`uncompyle6`，又重新安装了一次。

 

目前的版本：`xdis 5.0.5` `uncompyle6 3.7.4`

 

查看该版本`xdis`的`magics.py`文件：

 

![](https://bbs.pediy.com/upload/attach/202012/600394_7Y95XYYTSBGEQCN.png)

 

可以看到已经有 3394 了。

 

反编译：

 

![](https://bbs.pediy.com/upload/attach/202012/600394_MTMA25FQKGAUHPW.png)

 

成功！

 

_注：pyc 文件一定要有后缀名 pyc，不然会报错_

[](#二、pyz文件的加密问题)二、PYZ 文件的加密问题
------------------------------

有些时候在**步骤 1 exe→pyc** 的过程中，会出现 PYZ 中的文件无法正常提取（archive_viewer.py），或者提取出来后显示 encrypted（pyinstxtractor.py）的问题。

 

这个问题可以使用**参考文章 2 和 3** 中的方法解决：

 

PYZ 文件加密的密钥保存在`pyimod00_crypto_key`文件中，该文件也是一个 pyc 文件，可以使用上面介绍的方法进行反编译，然后就可以获得密钥：

 

![](https://bbs.pediy.com/upload/attach/202012/600394_8E82CZ6YCU6BT8P.png)

 

之后的解密脚本有三个可供选择，均在**参考文章 3** 中，根据 pyinstaller 的版本不同选择不同的脚本，使用时需要替换其中的 key、header 以及待解密文件名和目标文件名，执行后即可获得解密后的 pyc 文件，再使用 uncompyle6 反编译即可。

 

_注：这三个脚本中，第一个脚本适用于 PyInstaller<4.0，使用 python2 执行；第二个和第三个脚本适用于 PyInstaller≥4.0，使用 python3 执行。_

 

![](https://bbs.pediy.com/upload/attach/202012/600394_M8F4F4MBCMHKS2V.png)

参考文章
----

1.  [反编译 python 打包的 exe 文件](https://www.cnblogs.com/QKSword/p/10540431.html)
    
2.  [Extracting encrypted pyinstaller executables](https://0xec.blogspot.com/2017/02/extracting-encrypted-pyinstaller.html)
    
3.  [pyinstxtractor wiki](https://github.com/extremecoders-re/pyinstxtractor/wiki/Frequently-Asked-Questions)
    
4.  [https://fishc.com.cn/forum.php?mod=viewthread&tid=131172&page=1#pid3772102](https://fishc.com.cn/forum.php?mod=viewthread&tid=131172&page=1#pid3772102)
    

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 2020-12-15 15:01 被 LarryS 编辑 ，原因： 增加原创标志以及调试逆向分类

上传的附件：

*   [python_exe 解包工具. zip](javascript:void(0)) （7.98MB，85 次下载）