> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.v4ler1an.com](https://www.v4ler1an.com/fuzzing101-2/)

> Fuzzing 101 系列 note 2

本文是 Fuzzing101 系列第二篇，fuzz 的对象为 libexif 库。

<table><thead><tr><th>Target</th><th>CVES to find</th><th>Time estimated</th><th>Main topics</th></tr></thead><tbody><tr><td>libexif</td><td>CVE-2009-3895,<br>CVE-2012-2836</td><td>3hous</td><td>aft-clang-lto, fuzz libraries, Eclipse IDE</td></tr><tr><td></td><td></td><td></td><td></td></tr></tbody></table>

*   CVE-2009-3895: heap-based buffer overflow vulnerability.
*   CVE-2012-2836: out-of-bounds read vulnerability.

1.  如何对使用了外部库的应用进行 fuzz
2.  使用 `afl-clang-lto` 进行 fuzz，它比 `afl-clang-fast` 的速度更快
3.  使用 Eclipse IDE 进行动态调试

1.  寻找使用了 `libexif` 库的应用接口
2.  创建 exif 样例的种子语料库
3.  使用 afl-clang-lto 编译 libexif 和选择的应用程序
4.  对 libexif 进行 fuzz
5.  对 crash 进行分类过滤，确认每个漏洞的 PoC
6.  修复漏洞

#### [](#1-download-and-build-target)1. Download and build target

首先创建待 fuzz 的 libexif 环境，进行编译待用：

```
# download
wget https://github.com/libexif/libexif/archive/refs/tags/libexif-0_6_14-release.tar.gz
tar -xzvf libexif-0_6_15-release.tar.gz

# build and install libexif
cd libexif-libexif-0_6_15-release/
sudo apt install autopoint libtool gettext libpopt-dev
autoreconf -fvi
./configure --enable-shared=no --prefix="$HOME/Desktop/Fuzz/training/fuzzing_libexif/install/"
make
make install

# choosing an interface application
wget https://github.com/libexif/exif/archive/refs/tags/exif-0_6_15-release.tar.gz
tar -xzvf exif-0_6_15-release.tar.gz
# build and install exif command-line utility
cd ..
cd exif-exif-0_6_15-release/
autoreconf -fvi
./configure --enable-shared=no --prefix="$HOME/Desktop/Fuzz/traning/fuzzing_libexif/install/" PKG_CONFIG_PATH=$HOME/Desktop/Fuzz/traning/fuzzing_libexif/install/lib/pkgconfig
make
make install
```

备注：这里的 libexif 的版本最好选用 0_6_15 版本，14 的版本 make install 会一直报错，而且没有出现过官方 issue。为节省时间，更换了版本。

创建种子语料库，这里选用的是 github 上公开的一个 exif 的样例库：https://github.com/ianare/exif-samples。

```
# download and unzip
cd $HOME/Desktop/Fuzz/training/fuzzing_libexif
wget https://github.com/ianare/exif-samples/archive/refs/heads/master.zip
unzip master.zip
```

安装完成后，使用 `exif` 检测一下样本，可以成功识别即可。

使用 `afl-clang-lto` 重新对 libexif 和 exif 进行编译：

```
# recompile libexif with afl-clang-lto
rm -r $HOME/Desktop/Fuzz/training/fuzzing_libexif/install
cd $HOME/Desktop/Fuzz/training/fuzzing_libexif/libexif-libexif-0_6_15-release/
make clean
export LLVM_CONFIG="llvm-config-12" # llvm-config-version at least is 11
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/Desktop/Fuzz/training/fuzzing_libexif/install/"
make
make install

# recompile exif with afl-clang-lto
cd $HOME/fuzzing_libexif/exif-exif-0_6_15-release
make clean
export LLVM_CONFIG="llvm-config-12"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/" PKG_CONFIG_PATH=$HOME/fuzzing_libexif/install/lib/pkgconfig
make
make install
```

编译完成后，可以使用 afl++ 在 `afl-clang-lto` 模式下开始进行 fuzz：

```
afl-fuzz -i $HOME/Desktop/Fuzz/training/fuzzing_libexif/exif-samples-master/jpg/ -o $HOME/Desktop/Fuzz/training/fuzzing_libexif/out/ -s 123 -- $HOME/Desktop/Fuzz/training/fuzzing_libexif/install/bin/exif @@
```

最终跑得的结果如下（因为自动跑的，所以 cycle 超了）：

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210816192235.png)

```
# install java
sudo apt install default-jdk

# download and run Eclipse
wget https://download.eclipse.org/technology/epp/downloads/release/2021-06/R/eclipse-cpp-2021-06-R-linux-gtk-x86_64.tar.gz
tar -zxvf eclipse-cpp-2021-06-R-linux-gtk-x86_64.tar.gz
```

解压完成后，进入文件夹，运行 `eclipse` 即可。

导入项目：选择 `File -> Import ` ， 然后选择 `C/C++` 里的 `Existing code as makefile project` 。然后选择 `Linux GCC` ，并选择代码路径。

调试：选择 `run -> Debug Configurations`，然后选择 exif 项目并且选定 exif 可执行程序，然后设置 `Arguments` 中为 crash 的绝对路径名，最后点击 `Debug` 即可。调试过程中，直接 `F8` 或者 `run -> Resume` 可以直接来到 crash 现场。

最后就是使用 Eclipse 进行 crash 的 debug 了，这个就不做记录了，需要花时间调试每个 crash 文件。

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210816194955.png)