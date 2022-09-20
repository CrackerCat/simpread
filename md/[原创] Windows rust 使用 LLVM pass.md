> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274453.htm)

> [原创] Windows rust 使用 LLVM pass

前言
==

rust 的编译器 rustc 用 llvm 进行中间代码生成 (MIR-> LLVM IR [链接](https://rustc-dev-guide.rust-lang.org/overview.html))，所以我想尝试下在 rust 编译过程加个 pass 进行代码混淆，进而保护生产代码。

 

由于 rust 在 Windows 下有两种 toolchain，一种是 msvc，另外一种是用 mingw 的 _windows-gnu_。由于 LLVM 在 Windows 下的动态库编译只能使用 Mingw-w64 环境，具体来源：[LLVM 官方 CMake 参数](https://llvm.org/docs/CMake.html#llvm-related-variables)，并且 rust 自己编译的 LLVM 不支持动态链接。

 

![](https://bbs.pediy.com/upload/attach/202209/956011_PJGJHAXV63NRNXA.png)

 

即本文使用 MSYS2 下的 Mingw-w64 环境。

环境准备
====

机器要求
----

足够强劲的机器，大概 20G 的硬盘空间 (固态更好)，8G 以上的内存，以及_良好的网络链接_

 

使用 Ninja 代替 make 能大幅提升速度，但是增加内存消耗 (link 时可能达到 24G 内存占用), 若内存不足的可以在生产 cmake build 的时候使用`MinGW Makefiles`或者`Unix Makefiles`., 本文主要使用`Ninja`作为主要构建工具.

 

空间占用：  
![](https://bbs.pediy.com/upload/attach/202209/956011_T8VAP7XUBF2Q8XE.png)

 

内存占用:

 

![](https://bbs.pediy.com/upload/attach/202209/956011_USMWT4ZD9AJ9P4H.png)

msys2
-----

从清华源 [链接](https://mirrors.tuna.tsinghua.edu.cn/msys2/distrib/x86_64/) 下载最新的安装包，例如 **msys2-x86_64-20220904.exe**

 

然后直接安装

 

![](https://bbs.pediy.com/upload/attach/202209/956011_QNHXHYFZKRW5VPH.png)

 

中间会卡一会，等一下就好

 

完成的时候不要直接启动它默认的终端  
![](https://bbs.pediy.com/upload/attach/202209/956011_XQ968G9YV5FQTRE.png)

 

去开始菜单找 msys2 - mingw64  
![](https://bbs.pediy.com/upload/attach/202209/956011_6YHMDAHDYCUDFPZ.png)

 

安装编译依赖

 

`pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-ninja mingw-w64-x86_64-python3 mingw-w64-x86_64-cmake autoconf libtool`

 

这里安装了 gcc、ninja 构建工具，mingw-w64 版的 python3，mingw-w64 下的 cmake，以及一些编译用到的工具

git
---

可自己从 git [官网](https://git-scm.com/download/win)下，也可以在 msys2 安装，看个人喜好

 

我偏向于使用官网下载的 git，其他 ide 也能用上

 

![](https://bbs.pediy.com/upload/attach/202209/956011_9QFKRQNDYGZU3H2.png)

 

下载 64 位 Standalone Installer 即可，安装过程可以按照个人喜好配置，如果只是为了本次编译就直接无脑下一步即可。

cmake
-----

可从[官网](https://cmake.org/download/)下 windows 安装包，也可以在 msys2 中安装 **mingw-w64-x86_64-cmake**, 但千万不能使用 msys2 提供的 **cmake** 包.

 

即使用`pacman -S mingw-w64-x86_64-cmake`安装 cmake, 本文使用 msys2 中的 **mingw-w64-x86_64-cmake**.

 

若使用官网安装的 cmake, 需要在 msys2 mingw 中手动指定 cmake.exe 位置, 即

 

`/c/Program\ Files/CMake/bin/cmake -G "Unix Makefiles"`

rust 源码
-------

在官方 github 分支中找到你想要的版本，例如 **1.63.0**

 

然后在你自己的工作目录用命令

 

`git clone --single-branch --branch 1.63.0 https://github.com/rust-lang/rust`

 

克隆指定分支的 rust 源码  
![](https://bbs.pediy.com/upload/attach/202209/956011_DW5M62QANGXSPSP.png)

对应 rust 版本的 LLVM
----------------

这里注意不要去 llvm-org 直接下载源码，以免出现不兼容等 bug

 

在 rust 源码里有**.gitmodules** 这个文件，打开看对应 LLVM 的分支，并克隆。例如 rust 1.63.0 对着的 LLVM 版本 **rustc/14.0-2022-03-22**

 

`git clone --branch rustc/14.0-2022-03-22 https://github.com/rust-lang/llvm-project.git`  
![](https://bbs.pediy.com/upload/attach/202209/956011_MHV5RAVFFPYR5BJ.png)

编译 LLVM
=======

编译 LLVM 本体
----------

1.  生成 cmake 构建项目:
    
    在刚刚克隆的 LLVM 源码**同级目录**输入以下命令
    
    `cmake -G "Ninja" -S ./llvm-project/llvm -B ./build_dyn_x64 -DCMAKE_INSTALL_PREFIX=./llvm_x64 -DCMAKE_CXX_STANDARD=17 -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;lld;" -DLLVM_TARGETS_TO_BUILD="X86" -DBUILD_SHARED_LIBS=ON -DLLVM_INSTALL_UTILS=ON -DLLVM_INCLUDE_TESTS=OFF -DLLVM_BUILD_TESTS=OFF -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_BUILD_BENCHMARKS=OFF`
    
    命令解析:
    
    ```
    -G "Ninja" 让cmake生成ninja的编译项目
    -S ./llvm-project/llvm 指定源码目录
    -B ./build_dyn_x64 指定编译输出目录
    -DCMAKE_INSTALL_PREFIX=./llvm_x64 指定LLVM安装目录为当前目录下的llvm_x64
    -DCMAKE_CXX_STANDARD=17 指定C++ 17标准， LLVM需要至少C++17以上
    -DCMAKE_BUILD_TYPE=Release 指定构建类型为Release
    -DLLVM_ENABLE_PROJECTS="clang;lld;" 指定LLVM启用项目
    -DLLVM_TARGETS_TO_BUILD="X86" 指定编译LLVM的目标,这里只启用了x86
    -DBUILD_SHARED_LIBS=ON 构建LLVM动态库
    -DLLVM_INSTALL_UTILS=ON 安装LLVM的其他工具，这个是rust编译时需要的FileCheck等工具
    -DLLVM_INCLUDE_TESTS=OFF -DLLVM_BUILD_TESTS=OFF 不编译测试
    -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_BUILD_BENCHMARKS=OFF 不编译基准测试
    
    ```
    
    具体参数可以查看 LLVM 的[官方文档](https://llvm.org/docs/CMake.html)
    
    ![](https://bbs.pediy.com/upload/attach/202209/956011_8RPQ9R7QXKPQMN9.png)
    
2.  开始编译:
    
    `cmake --build ./build_dyn_x64 -j 22`
    
    命令解析:
    
    ```
    --build ./build_dyn_x64 告诉cmake从哪里开始编译
    -j 22 指定多少线程同时编译
    
    ```
    
    然后就是等待，这个时间可以刷会视频看看小说, 大概 30-40 分钟，具体时长取决于机器配置。
    
3.  安装 (生成)LLVM
    
    `cmake --install ./build_dyn_x64`
    
    安装编译好的 LLVM 到 **llvm_x64** 目录
    
4.  添加运行 LLVM 的依赖
    
    由于是使用 Mingw-w64 的环境，编译出来的 LLVM 无法直接使用，需要去 MSYS2 的 **mingw64 的 bin 目录**复制下列 dll 到 LLVM 的 bin 目录。
    
    需要复制的 dll 列表
    
    ```
    libstdc++-6.dll
    libgcc_s_seh-1.dll
    libwinpthread-1.dll
    zlib1.dll
    
    ```
    
    ![](https://bbs.pediy.com/upload/attach/202209/956011_SP3P4KAC7T93YTP.png)
    

编译完成后的目录结构：  
![](https://bbs.pediy.com/upload/attach/202209/956011_44C5KY8W34PD4N2.png)

 

成果：  
![](https://bbs.pediy.com/upload/attach/202209/956011_VTRWSG9J26VA57K.png)

给 LLVM 添加混淆插件
-------------

`cmake -DLLVM_DIR="C:/Users/Administrator/Desktop/tutorial/llvm_x64/lib/cmake/llvm" -G "Ninja" -S obfuscator-llvm/ -B ollvm -DCMAKE_CXX_STANDARD=17`

 

`cmake --build ollvm/ -j 20`

 

cp

编译 rustc
========

配置 rust 编译选项
------------

1.  进入 rust 源码目录，复制一份`config.toml.example`到`config.toml`
    
    如图所示
    
    ![](https://bbs.pediy.com/upload/attach/202209/956011_JYQRNW2JQUAFNFW.png)
    
2.  修改`config.toml`
    
    *   在 [rust] 子项，debug = false
        
        这里把 debug 关闭，提升速度
        
        ![](https://bbs.pediy.com/upload/attach/202209/956011_TDE7GWDA6FW6U9Z.png)
        
    *   在 [rust] 子项，往下的 channel 设置成 nightly
        
        使用 nightly channel 才能设置 LLVM pass 的参数
        
        ![](https://bbs.pediy.com/upload/attach/202209/956011_V2Z3SZ345W877ED.png)
        
    *   在 [target.x86_64-unknown-linux-gnu] 子项，`llvm-config`设置到刚刚我们编译的 llvm 目录下的 llvm-config
        
        修改 **x86_64-unknown-linux-gnu** 为 **x86_64-pc-windows-gnu**
        
        这里设置 llvm-config 选项后，就能跳过 rustc 编译的过程中编译非动态版本的 llvm
        
        ![](https://bbs.pediy.com/upload/attach/202209/956011_PBV3Z8WDUM7FHCP.png)
        

修改 rust 的源码
-----------

因为 rustc(rust 的编译器) 是有几个不同的编译阶段

*   阶段 0：启动，下载当前 beta 版本的 rustc 等工具来编译 src/bootstrap, std, rustc
*   阶段 1：由阶段 0 编译出当前源码的编译器 (rustc)，使用阶段 0 的 abi
*   阶段 2：由阶段 1 编译出当前源码的编译器，使用阶段 1 的 abi

这里我只是大概了解了一下，可能会有些偏差，具体详情可以看[官方文档](https://rustc-dev-guide.rust-lang.org/building/bootstrapping.html)以及[源码的说明](https://github.com/rust-lang/rust/tree/master/src/bootstrap)

 

其中，我们真正使用的是阶段 2 的文件作为我们自己的 toolchain。由于我们的 llvm 是动态编译的，所以在阶段 1-> 阶段 2 的时候编译出的 rustc 无法执行，需要跟 LLVM 一样需要手动补充依赖文件。

 

修改的文件：src/bootstrap/compile.rs

```
if target_compiler.stage == 1 {
    let llvm_dir = r"C:\\Users\\Administrator\\Desktop\\tutorial\\llvm_x64\\bin";
    for entry in fs::read_dir(llvm_dir).expect("Dir not found!") {
        let entry = entry.expect("this was dir");
        let path = entry.path();
        if let Some(extension) = path.extension() {
            if extension == "dll" {
                let dst_path = bindir.join(entry.file_name());
                // println!("{}", dst_path.as_path().display().to_string());
                fs::copy(path, dst_path).expect("error copy");
            }
        }
    }
}

```

位置：

 

![](https://bbs.pediy.com/upload/attach/202209/956011_KN7P4HTZUN72SVZ.png)

 

这里我们在创建完每个阶段的 **bin** 文件夹后，判断当前阶段，如果是阶段 1 就复制我们自己编译的 LLVM 的 dll 文件到当前阶段的 **bin** 文件夹，这样就能保证 rustc 的编译过程能完成。

编译 rustc
--------

这里有个小坑，如果没有在 MSYS2 装 Git，但是在 Windows 下装了 Git 的，需要手动设置下环境变量，否则 rust 的编译脚本无法找到 git

 

`export PATH=$PATH:/c/Program\ Files/Git/bin`

 

![](https://bbs.pediy.com/upload/attach/202209/956011_67EN8F7BTUD27ZJ.png)

 

在 mingw-w64 窗口，进到 rust 的源码目录，执行

 

`python x.py build`

 

然后等待大概 30-40 分钟，具体时间取决于编译机器的配置以及网络情况。

 

编译完成：

 

![](https://bbs.pediy.com/upload/attach/202209/956011_88GW9R2P6Q4J8BE.png)

 

成品：

 

![](https://bbs.pediy.com/upload/attach/202209/956011_DXGRVF2XRHKX52Z.png)

 

![](https://bbs.pediy.com/upload/attach/202209/956011_E88VQBD8WG47CRN.png)

编译 cargo
========

编译 cargo 本体
-----------

在 rust 源码目录执行

 

`python x.py build tools/cargo`

 

等待 20-30 分钟，具体时长取决于编译机器的配置。

 

成品：

 

![](https://bbs.pediy.com/upload/attach/202209/956011_3AFUZARQH4TDVET.png)

 

![](https://bbs.pediy.com/upload/attach/202209/956011_GZS3YJV9MK9JASB.png)

复制依赖
----

编译出来的 cargo 还是跟之前一样，缺少 dll，从 Mingw-w64 里面复制 **zlib1.dll** 到 **rust/build/x86_64-pc-windows-gnu/stage1-tools-bin**

 

**![](https://bbs.pediy.com/upload/attach/202209/956011_UM692A7GZVVBCHR.png)**

使用自己的 toolchain
===============

安装 rustup
---------

配置自己编译的 toolchain
-----------------

使用
--

总结
==

[恭喜 ID[飞翔的猫咪] 获看雪安卓应用安全能力认证高级安全工程师！！](https://mp.weixin.qq.com/s?src=11&timestamp=1659838130&ver=3967&signature=s6siC7hKil1wiaYVM8OGNqi79zQUdeCyW1TxoUpoK84v1ad8jvFOToeTvldyCoA8TKIlBSG1v2tginEMXvpjApaL4PVAL08iUQRuDhMz-aox2jpUAiSkr9s-EHsw88wi&new=1)

[#基础知识](forum-41-1-130.htm) [#开发技巧](forum-41-1-135.htm)