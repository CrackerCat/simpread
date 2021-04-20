> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267084.htm)

1、QAsan 简介  

=============

        论文地址  [https://andreafioraldi.github.io/assets/qasan-secdev20.pdf](https://andreafioraldi.github.io/assets/qasan-secdev20.pdf)

        目前找到的唯一能在闭源的 arm 架构下进行内存漏洞检测的工具，要是有更好的工具希望能交流一下。

        QAsan 算是 ASan+QEMU 两个工具的结合，现在已经集成到 AFL++。Asan 只能对有源码的代码进行插桩检测，QAsan 能对闭源的代码进行内存检测，并且支持 arm 架构（包括 arm32 和 arm64）。不过缺点是会拉低 fuzz 的执行效率，而且只能检测出堆溢出的漏洞，如果对闭源的 x86_64 进行检测，可以去使用 [retrowrite](https://github.com/HexHive/retrowrite)，这一点在文档里有提到。

        参考文档：[https://github.com/AFLplusplus/AFLplusplus/tree/dev/qemu_mode/libqasan](https://github.com/AFLplusplus/AFLplusplus/tree/dev/qemu_mode/libqasan)

        **这里尝试的是 ARM64 的执行效果，原因是使用的 Ubuntu 系统是 X86_64，是 64 位的。如果想在 64 位的系统下对 32 位的 arm 文件使用 QASan，需要安装其他的依赖项（ia32-libs）。如果不想安装依赖项并直接对 arm32 位的文件进行检测的话，可以尝试 i386 的系统。**  

2、QASan 安装  

=============

![](https://bbs.pediy.com/upload/attach/202104/922479_XF65V9GM42JW4WB.jpg)

        上一篇（ [https://bbs.pediy.com/thread-267074.htm](https://bbs.pediy.com/thread-267074.htm)  ）提到怎么使用 AFL++ 进行 fuzz，在编译的时候会出现 qasan 编译不了的情况，但是这不影响 fuzz 的过程，不过跑好久都跑不出一个 crash 来，不如单独编译 qasan，等有了 crash 在去用 qasan，还能加快 fuzz 速度。  

        参考 [https://github.com/andreafioraldi/qasan](https://github.com/andreafioraldi/qasan)

                [https://github.com/andreafioraldi/qasan/issues/2](https://github.com/andreafioraldi/qasan/issues/2) 

1、先 git 下来，用下面的命令。要是太慢的话导入到 gitee 里面，要用 gitee 的话记得要把需要 recursive 的 [asan-giovese @ b844043](https://github.com/andreafioraldi/asan-giovese/tree/b8440434bf21830fff1401134558cabeb06f2977) 放到相应的位置。

```
git clone --recursive https://github.com/andreafioraldi/qasan.git

```

2、安装 arm64 的 gcc  

安装完成后，aarch64-linux-gnu 会默认安装在 / usr/aarch64-linux-gnu 路径下，设置 QEMU_LD_PREFIX 为此路径。

```
sudo apt-get install gcc-aarch64-linux-gnu
export QEMU_LD_PREFIX=/usr/aarch64-linux-gnu/

```

3、编译 qasan

```
./build.py --arch arm64 --cross aarch64-linux-gnu-gcc

```

![](https://bbs.pediy.com/upload/attach/202104/922479_HZCJUAP3VMTVP98.jpg)

按它的提示执行  

```
./qasan /bin/ls

```

![](https://bbs.pediy.com/upload/attach/202104/922479_6H47NJ9G9QJ3SH7.jpg)

会报这个错误，是正常的，因为 ls 是 x86 架构的，而编译出来的是 aarch64 的。

4、交叉编译 arm64 的文件

```
cd tests/
aarch64-linux-gnu-gcc --static invalid_free.c -o invalid_free
cd ..

```

![](https://bbs.pediy.com/upload/attach/202104/922479_FU7NJ4TR6UXUW3U.jpg)

5、直接使用 qasan 测试

tests 文件下全是有内存错误的文件，都可以直接报出错误。

```
./qasan ./tests/invalid_free

```

![](https://bbs.pediy.com/upload/attach/202104/922479_JGJWBHXU7M8MAVC.jpg)

6、使用 afl++ 和 QAsan 进行测试

默认的 tests 目录下的文件一执行就是错的，不能直接使用 afl，所以新建了一个. c 文件。

```
#include  int main(){
    int a;
    scanf("%d", &a);
    if(a==123456789)
        abort();
    printf("Retry!!\n");
    return 0;
} 
```

![](https://bbs.pediy.com/upload/attach/202104/922479_Y96NH6SURMV59NW.jpg)

用 arm64 编译后，回到 qasan 目录下

-U 是使用的 Unicorn，是 qemu 上的一个仿真环境。

```
afl-fuzz -U -i inputs/ -o outputs -m none -- python3 qasan ./tests/mytest

```

![](https://bbs.pediy.com/upload/attach/202104/922479_BZA4GRAGSHHMCD3.jpg)

[[公告] 2021 KCTF 春季赛 防守方征题火热进行中！](https://bbs.pediy.com/thread-266222.htm)