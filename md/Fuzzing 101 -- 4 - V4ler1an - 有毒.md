> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.v4ler1an.com](https://www.v4ler1an.com/fuzzing101-4/)

> Fuzzing 101 系列 note 4

本文是 Fuzzing101 系列第四篇，fuzz 的对象为 LibTIFF 。

<table><thead><tr><th>Target</th><th>CVES to find</th><th>Time estimated</th><th>Main topics</th></tr></thead><tbody><tr><td>LibTIFF</td><td>CVE-2016-9297</td><td>3hous</td><td>measure the code coverage data</td></tr></tbody></table>

> CVE-2017-13028: Out-of-bounds Read vulneratibily.

1.  什么是 Code Coverage，代码覆盖率
2.  使用 LCOV 对代码覆盖率进行测量
3.  如何通过代码覆盖率的优化提升 Fuzzing 性能

1.  开启 ASan 功能，对 LibTiff 库进行 fuzz
2.  分析 crash ，找到对应漏洞的 PoC
3.  测量该 PoC 的代码覆盖率情况
4.  修复漏洞

#### [](#1-download-and-build-target)1. Download and build target

首先创建待 fuzz 的 LibTiff 环境，进行编译待用：

```
cd $HOME/Desktop/Fuzz/training
mkdir fuzzing_tiff && cd fuzzing_tiff/

# download and uncompress the target
wget https://download.osgeo.org/libtiff/tiff-4.0.4.tar.gz
tar -xzvf tiff-4.0.4.tar.gz

# make and install libtiff
cd tiff-4.0.4/
./configure --prefix="$HOME/Desktop/Fuzz/training/fuzzing_tiff/install/" --disable-shared
make
make install


# test the target program
$HOME/Desktop/Fuzz/training/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/Desktop/Fuzz/training/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff
```

以上安装不报错的话，可以正常调用 LibTiff 库 ：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210909173303.png)

在上面的启动命令中，基本开启了软件的所有参数，这样有利于在进行 fuzz 时执行更多的代码路径，从而获得更高的代码覆盖率。

直接使用 `test/images` 文件夹下的测试用例作为本次 fuzz 的语料。

代码覆盖率是一种软件指标，表达了每行代码被触发的次数。通过使用代码覆盖率，我们可以了解 fuzzer 已经到达了代码的哪些部分，并可视化了 fuzzing 过程。

首先，需要安装 `lcov`：

然后，我们使用 `--coverage`选项来重建 libTIFF 库：

```
rm -r $HOME/Desktop/Fuzz/training/fuzzing_tiff/install
cd $HOME/Desktop/Fuzz/training/fuzzing_tiff/tiff-4.0.4/
make clean
  
CFLAGS="--coverage" LDFLAGS="--coverage" ./configure --prefix="$HOME/Desktop/Fuzz/training/fuzzing_tiff/install/" --disable-shared
make
make install
```

然后使用下面的指令来进行代码覆盖率收集：

```
cd $HOME/Desktop/Fuzz/training/fuzzing_tiff/tiff-4.0.4/
lcov --zerocounters --directory ./   # 重置计数器
lcov --capture --initial --directory ./ --output-file app.info
$HOME/Desktop/Fuzz/training/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/Desktop/Fuzz/training/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff
lcov --no-checksum --directory ./ --capture --output-file app2.info # 返回“基线”覆盖数据文件，其中包含每个检测行的零覆盖
```

最后，生成 HTML 输出：

```
genhtml --highlight --legend -output-directory ./html-coverage/ ./app2.info
```

一切顺利的话，会生成以下文件：

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

打开生成的 `index.html` 会看到如下结果：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210909173345.png)

重新编译：

```
rm -r $HOME/Desktop/Fuzz/training/fuzzing_tiff/install
cd $HOME/Desktop/Fuzz/training/fuzzing_tiff/tiff-4.0.4/
make clean

export LLVM_CONFIG="llvm-config-12"
CC=afl-clang-lto ./configure --prefix="$HOME/Desktop/Fuzz/training/fuzzing_tiff/install/" --disable-shared
# 开启AFL_USE_ASAN
AFL_USE_ASAN=1 make -j4
AFL_USE_ASAN=1 make install
```

执行 `afl-fuzz` :

```
afl-fuzz -m none -i $HOME/Desktop/Fuzz/training/fuzzing_tiff/tiff-4.0.4/test/images/ -o $HOME/Desktop/Fuzz/training/fuzzing_tiff/out/ -s 123 -- $HOME/Desktop/Fuzz/training/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w @@
```

最终跑得的结果如下：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210909173359.png)

ASan 追踪结果如下：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210909173412.png)

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210909173424.png)

官方的修复地址：

*   [https://github.com/the-tcpdump-group/tcpdump/commit/29e5470e6ab84badbc31f4532bb7554a796d9d52](https://github.com/the-tcpdump-group/tcpdump/commit/29e5470e6ab84badbc31f4532bb7554a796d9d52)

后续将对该漏洞进行深入分析和补丁分析，待完善。

官方的修复地址：

*   [https://github.com/the-tcpdump-group/tcpdump/commit/29e5470e6ab84badbc31f4532bb7554a796d9d52](https://github.com/the-tcpdump-group/tcpdump/commit/29e5470e6ab84badbc31f4532bb7554a796d9d52)

后续将对该漏洞进行深入分析和补丁分析，待完善。