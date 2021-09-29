> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.v4ler1an.com](https://www.v4ler1an.com/fuzzing101-3/)

> Fuzzing 101 系列 note 3

本文是 Fuzzing101 系列第三篇，fuzz 的对象为 tcpdump。

<table><thead><tr><th>Target</th><th>CVES to find</th><th>Time estimated</th><th>Main topics</th></tr></thead><tbody><tr><td>TCPdump</td><td>CVE-2017-13028</td><td>4hous</td><td>ASAN</td></tr></tbody></table>

*   CVE-2017-13028: Out-of-bounds Read vulneratibily.

1.  什么是 ASAN(Address Sanitizer)，一个运行时内存错误检测工具
2.  如何使用 ASAN 进行 fuzz
3.  使用 ASAN 对 crash 进行分类

1.  确定如何进行 TCpdump 的 fuzz 工作
2.  在 fuzz 时开启 ASAN 功能
3.  实际的 fuzz 过程
4.  追踪 crash，找到对应的漏洞的 poc
5.  修复漏洞

#### [](#1-download-and-build-target)1. Download and build target

首先创建待 fuzz 的 TCPdump 环境，进行编译待用：

```
cd $HOME/Desktop/Fuzz/training/fuzzing_tcpdump

# download and uncompress tcpdump-4.9.2.tar.gz
wget https://github.com/the-tcpdump-group/tcpdump/archive/refs/tags/tcpdump-4.9.2.tar.gz
tar -xzvf tcpdump-4.9.2.tar.gz

# download and uncompress libpcap-1.8.1.tar.gz
wget https://github.com/the-tcpdump-group/libpcap/archive/refs/tags/libpcap-1.8.1.tar.gz
tar -xzvf libpcap-1.8.1.tar.gz

# build and install
cd libpcap-libpcap-1.8.1/
./configure --enable-shared=no --prefix="$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/"
make
make install

# build and install tcpdump
cd ..
cd tcpdump-tcpdump-4.9.2/
CPPFLAGS=-I$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/include/ LDFLAGS=-L$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/lib/ ./configure --prefix="$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/"
make
make install

# test
$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/sbin/tcpdump -h
```

以上安装不报错的话，可以正常启动 tcpdump ：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210823194623.png)

在 `tests` 文件夹中有很多的测试样例：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210823194813.png)

运行样例的命令如下：

```
$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r [.pcap file]
# -vvvv 输出极为详细的信息
# -XX 把协议头和包内容都原原本本的显示出来（tcpdump会以16进制和ASCII的形式显示）
# -ee 在输出行打印出数据链路层的头部信息，包括源mac和目的mac，以及网络层的协议
# -nn 指定将每个监听到的数据包中的域名转换成IP、端口从应用名称转换成端口号后显示
# -r 指定文件
```

运行结果大概如下：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210823195058.png)

`AddressSanitizer(ASAN)` 是一个 C 和 C++ 的内存错误检测工具，2011 年由 Google 的研究员开发。

它包括一个编译器检测模块和一个运行时库，该工具可以发现对堆、栈和全局对象的越界访问、释放后重利用、双重释放和内存泄漏错误。

AddressSanitizer 是开源的，并且从 3.1 版开始与 LLVM 编译器工具链集成。虽然它最初是作为 LLVM 的项目开发的，但它已被移植到 GCC 并包含在 GCC >= 4.8 的版本中 。

更多内容请参考 [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)。

本文我们主要是为了在 fuzz 时开启 ASAN，所以先删除上面已经编译好的对象文件和可执行文件：

```
rm -r $HOME/fuzzing_tcpdump/install
cd $HOME/fuzzing_tcpdump/libpcap-libpcap-1.8.1/
make clean

cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
make clean
```

clean 完成后，在进行 make 前附加 `AFL_USE_ASAN=1` 的编译选项：

```
cd $HOME/Desktop/Fuzz/training/fuzzing_tcpdump/libpcap-libpcap-1.8.1/
export LLVM_CONFIG="llvm-config-12"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make
AFL_USE_ASAN=1 make install

cd $HOME/Desktop/Fuzz/training/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
CC=afl-clang-lto CPPFLAGS=-I$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/include/ LDFLAGS=-L$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/lib/ ./configure --prefix="$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/"

CC=afl-clang-lto CPPFLAGS=-I$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/include/ LDFLAGS=-L$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/lib/ ./configure --prefix="$HOME/Desktop/Fuzz/training/fuzzing_tcpdump/install/"


AFL_USE_ASAN=1 make
AFL_USE_ASAN=1 make install
```

执行 `afl-fuzz` :

```
afl-fuzz -m none -i $HOME/Desktop/Fuzz/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/tests/ -o $HOME/Desktop/Fuzz/fuzzing_tcpdump/out/ -s 123 -- $HOME/Desktop/Fuzz/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r @@
```

备注：这里指定了 `-m none` 选项是取消了 AFL 的内存使用限制，因为在 64-bit 系统下，ASAN 会占用较多的内存。

最终跑得的结果如下：

![](https://github.com/antonio-morales/Fuzzing101/raw/main/Exercise%203/Images/Image3.png)

对使用了 ASAN 进行 build 的程序进行 debug 是一件十分容易的事情，只要直接将 crash 文件喂给程序运行即可，然后就可以得到 crash 的相关信息，包括函数的执行追踪：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210824081203.png)

官方的修复地址：

*   [https://github.com/the-tcpdump-group/tcpdump/commit/29e5470e6ab84badbc31f4532bb7554a796d9d52](https://github.com/the-tcpdump-group/tcpdump/commit/29e5470e6ab84badbc31f4532bb7554a796d9d52)

后续将对该漏洞进行深入分析和补丁分析，待完善。