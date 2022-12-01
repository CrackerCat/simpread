> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-275365.htm)

> [原创] Android Kernel 源码环境

> 工欲善其事 必先利其器

前言
==

> 本篇将会搭建一个完美的 Linux Kernel 及其内核模块的源码阅读开发环境

 

当我们开始研究 Android Kernel，想要优雅的阅读源码好像是一件费劲的事情。

 

因为 Kernel 源码实在是太庞大了，打开一个 c 文件，想要详细的研究研究，甚至上手写两句代码，即没有高亮提示，也没有代码跳转。在这种情况下，想要理清楚内核源码，相当不易。

*   用 CLion？ 貌似不太行。  
    Kernel 的构建体系是 make 而不是类似于 LLVM 的 CMake。Clion 直接打开 Kernel 源码是无法被 CLion 解析的。
    
*   用 Source Insight？貌似不是非常完美。  
    对于 Kernel 源码来说，很多函数 symbols 一样，只是适用于不同架构罢了。Source Insight 在跳转的时候，全源引索，并不会帮我们加以区分。诸如此类的问题，Source Insight 还有很多... 强迫症患者表示很难受~
    
*   用 VsCode？貌似更不行？  
    VsCode 对 C/C++ 代码的高亮提示，依托于`C/C++ Extension Pack`这个插件，我不清楚别人体验如何，至少对我来说，这个插件有多烂，我都不想多做评价....
    

浅谈代码索引
======

1.  为什么 IDE 集成开发环境可以完美的索引项目代码？  
    参考于[微软的一篇文档](https://learn.microsoft.com/zh-cn/visualstudio/extensibility/language-server-protocol?view=vs-2022#what-is-the-language-server-protocol)，我了解到了语言服务器这个东西。
    
2.  什么是语言服务器，有什么作用？
    
    *   简单来说，VsCode，VIM 等等，都是文本编辑器的性质，可以让我们愉快的编辑代码。而 clang，gcc 等等，都是代码编译套件，可以将我们的源码文件编译出来。最后 clangd，rust-analyzer 等等，就属于语言服务器，就是他们的工作，才使我们的源码具备了高亮提示和代码跳转方面的功能。
    *   其实，各大 IDE 可以完美的对代码做解析服务，多半也是内部集成了语言服务器。

> CLion 对代码索引的功能 貌似就是通过 clangd 实现的。

1.  如何让语言服务器运作起来？  
    我们现在知道了，想要对 c 语言的代码做解析，就需要 clangd 这个语言服务器。那么，如何让 clangd 运作起来呢？clangd 对代码做解析，需要`compile_commands.json`文件。
    
2.  该文件是什么作用？  
    这个文件记录着我们对源码编译时候的每个命令，用的什么编译器，提供的什么编译参数，链接了哪些库，设置了什么宏，编译的什么源码文件等等。  
    如此，clangd 就可以通过编译命令，详细的了解到整个编译流程。
    
3.  如何生成该文件？
    
    *   对于 make 项目来说，常规来讲，可以使用 [Bear](https://github.com/rizsotto/Bear) 来对源码生成`compile_commands.json`文件。
        
        ```
        make ## 原编译命令 --> 不生成compile_commands.json
        bear make ## 使用bear --> 生成compile_commands.json
        
        ```
        
    *   对于 CMake 项目来说，这个相当简单。  
        参考于 [CMake 官方文档](https://cmake.org/cmake/help/latest/variable/CMAKE_EXPORT_COMPILE_COMMANDS.html)，只需要设置`CMAKE_EXPORT_COMPILE_COMMANDS`为 True 即可在编译后生成`compile_commands.json`文件。
        
        ```
        set(CMAKE_EXPORT_COMPILE_COMMANDS True)
        
        ```
        

> 这里特别提一嘴，对于 mk 体系编译的的项目  
> mk 脚本，其本质上是 make，但是对于安卓的来说，生成编译描述文件非常简单  
> 官方有[文档](https://developer.android.com/ndk/guides/ndk-build#json)提供命令

```
ndk-build GEN_COMPILE_COMMANDS_DB=true ## 构建的时候顺便生成compile_commands.json

```

所以，理论上来说，搭建的这套代码解析方案，不仅仅适用于 Kernel 源码阅读。也适用于各种交叉编译

敲定方案
====

> 毫无疑问 Kernel 的源码大多是 c 文件，以及少部分的汇编文件，设备树文件......

1.  明确一点，我们需要 clangd 支持。  
    首先看看 VsCode 有没有 clangd 插件，搜了一下，果然有 [clangd 插件](https://github.com/clangd/vscode-clangd)。
2.  我们需要对内核编译的流程，生成描述文件。
    *   这里我找到一个项目，可以把已经完整编译好了的内核解析出编译描述文件出来:[vscode-linux-kernel](https://github.com/amezin/vscode-linux-kernel)。
    *   如何使用将在下面细说。
        
        > Kernel 正好是 make 体系，理论上是可以用 Bear 的。  
        > 由于我做教程的时候，已经把内核编译结束了，实在不想重新编译内核，理论上选哪个都一样，我们只是要个编译描述文件罢。  
        

*   使用套件最终定型:  
    VsCode + clangd + 编译描述文件 + C/C++ Extension Pack
    
    > 虽然 C/C++ Extension Pack 相当的烂，但是部分 C/C++ 方面的东西，还需他提供支持。  
    > 所以这里依然需要安装该插件，后续在设置中禁用掉他的代码提示服务，转交给 clangd 插件即可。
    

> 关于 clangd 插件

*   这个插件会网络下载 clangd 的 bin 文件，很多人可能网络不佳，会下载失败。
*   这里不需要担心这个，我们只管安装插件即可，ndk 里面自带的 clangd 就可以提供语言解析服务了，VsCode 设置里面指定 clangd 路径即可。
*   而且，貌似高版本 clangd 有什么大病？

开始动手
====

1.  完整编译内核
    
    *   这里以 ACK(Android Common Kernel) 为例.  
        因为 ACK 的编译最简单，不会出乱七八糟的错误，Google 基本上完善好了一键编译体系。
    *   参考 [Google 官方文档](https://source.android.google.cn/docs/setup/build/building-kernels)
        
        > 这里默认读者会换源或者其他手法保持网络通畅
        
    *   步骤简述:
        1.  初始化 repo 库
            
            ```
            ## 这里选择common-android12-5.10分支
            repo init -u https://android.googlesource.com/kernel/manifest -b common-android12-5.10
            
            ```
            
        2.  同步 repo 库代码
            
            ```
            repo sync
            
            ```
            
            漫长的等待...
        3.  选择自己需要的 config 文件，开启`build.sh`脚本  
            这里以`build.config.gki.aarch64`为例
            
            ```
            BUILD_CONFIG=common/build.config.gki.aarch64 build/build.sh
            
            ```
            
        4.  等待一段时间后，内核即可编译完成:  
            ![](https://bbs.pediy.com/upload/attach/202211/942658_PRAW35QDG3Q5T59.png)
2.  生成编译描述文件
    
    *   仔细观察我们编译的 out 目录，会发现有很多后缀为 cmd 的文件，这些其实就是编译过程中的临时文件，包含了编译命令等。
    *   [vscode-linux-kernel](https://github.com/amezin/vscode-linux-kernel) 其实就是利用编译后的 cmd 等文件做解析，生成编译描述文件`compile_commands.json`。
    *   步骤简述:
        1.  进入内核源码的根目录，将 [vscode-linux-kernel](https://github.com/amezin/vscode-linux-kernel) 项目拉下来:
            
            ```
            cd common # 进入内核源码根目录
            # clone vscode-linux-kernel项目到.vscode文件夹
            # 该文件夹为vscode配置文件夹 类似于.idea
            git clone --depth=1 https://github.com/amezin/vscode-linux-kernel.git .vscode
            
            ```
            
        2.  运行 python 脚本，并指定`-O`参数到编译产出目录:
            
            ```
            python .vscode/generate_compdb.py -O ../out/android12-5.10/common/
            ls -al | grep compile_commands.json
            # -rw-r--r--   1 kali kali 8888862 Nov 30 01:07 compile_commands.json
            
            ```
            
            ![](https://bbs.pediy.com/upload/attach/202211/942658_RC9SZCBHJEBNV29.png)  
            可以看到，源码目录下的`compile_commands.json`文件已经生成。
            
            > 对于 Out-of-tree 编译的内核模块  
            > 详细见原项目地址的 [Out-of-tree module development](https://github.com/amezin/vscode-linux-kernel#out-of-tree-module-development)  
            > 这一套代码解析方案也完美的适用于 Out-of-tree 的内核模块
            
3.  配置 VsCode  
    对于 VsCode 来说`.vscode`目录下，是一些配置文件。  
    我们修改原项目的`setting.json`文件，内容修改成如下:
    
    ```
    {
        "files.associations": {
            "iostream": "cpp",
            "intrinsics.h": "c",
            "ostream": "cpp",
            "vector": "cpp"
        },
        "editor.formatOnPaste": true,
        "editor.formatOnSave": true,
        "editor.formatOnType": true,
        // 关闭 C/C++ Extension Pack 插件的提示 防止其与clangd冲突
        "C_Cpp.errorSquiggles": "Disabled",
        "C_Cpp.intelliSenseEngineFallback": "Disabled",
        "C_Cpp.intelliSenseEngine": "Disabled",
        "C_Cpp.autocomplete": "Disabled", // So you don't get autocomplete from both extensions.
        // 指向clangd路径
        "clangd.path": "/tmp/NDK/ndk-r25b/toolchains/llvm/prebuilt/linux-x86_64/bin/clangd",
        "clangd.arguments": [
            // compelie_commands.json 文件的目录位置
            "--compile-commands-dir=${workspaceFolder}/",
            // 让 Clangd 生成更详细的日志
            "--log=verbose",
            // 输出的 JSON 文件更美观
            "--pretty",
            // 全局补全
            "--all-scopes-completion",
            // 建议风格：打包(重载函数只会给出一个建议）相反可以设置为detailed
            "--completion-style=bundled",
            // 跨文件重命名变量
            "--cross-file-rename",
            // 允许补充头文件
            "--header-insertion=iwyu",
            // 输入建议中，已包含头文件的项与还未包含头文件的项会以圆点加以区分
            "--header-insertion-decorators",
            // 在后台自动分析文件 基于 complie_commands
            "--background-index",
            // 启用 Clang-Tidy 以提供「静态检查」
            "--clang-tidy",
            // Clang-Tidy 静态检查的参数，指出按照哪些规则进行静态检查
            // 参数后部分的*表示通配符
            // 在参数前加入-，如-modernize-use-trailing-return-type，将会禁用某一规则
            "--clang-tidy-checks=cppcoreguidelines-*,performance-*,bugprone-*,portability-*,modernize-*,google-*",
            // 默认格式化风格: 谷歌开源项目代码指南
            "--fallback-style=file",
            // 同时开启的任务数量
            "-j=2",
            // pch优化的位置(memory 或 disk，选择memory会增加内存开销，但会提升性能)
            "--pch-storage=disk",
            // 启用这项时，补全函数时，将会给参数提供占位符
            // 我选择禁用
            "--function-arg-placeholders=false"
        ],
    }
    
    ```
    
    注释已经写到相当清楚，这里不再赘述。
    
4.  打开 Kernel 根目录，等待索引结束
    
    *   我这里是 ssh 远程连接的 Linux 虚拟机。
    *   打开源码根目录后，随便打开一个 c 文件，触发 VsCode 的插件后，他会提示你，你的默认配置会被`.vscode`目录下的 setting 文件覆盖，是否确认:  
        ![](https://bbs.pediy.com/upload/attach/202211/942658_XG2QX5AZQHTXRDZ.png)
    *   这里当然选择 YES，使用我们配置的 setting。
    *   停留在 c 文件上面，等待一段时间，则会出现如下画面:  
        ![](https://bbs.pediy.com/upload/attach/202211/942658_Y8RUJW3CT4NEECE.png)
    *   左边会出现`.cache`缓存目录，里面不断缓存着整个内核源码的 index 索引。
    *   左下角有一个 indexing，意味着 clangd 解析整个源码的进度。
    *   当 indexing 解析结束后，打开内核源码的任意 c 文件，都可以自由的跳转头文件，跳转到函数定义和实现，智能提示函数和结构体等等。就和用 IDE 写代码一样，非常丝滑舒服。
    *   所有的索引都在一秒不到内可以完成，而且也不会高额占用机器性能。
    *   甚至他还会结合编译的 config，对未开启的 config 代码块不高亮显示:  
        ![](https://bbs.pediy.com/upload/attach/202211/942658_MDH4Z5VW7HXRKW6.png)

做一些润滑
=====

*   你发现会报很多无关痛痒的警告:  
    ![](https://bbs.pediy.com/upload/attach/202211/942658_U8VXXRF68Z9FZV2.png)
    
    1.  内核源码根目录下，新建一个文件`.clangd`，内容如下:
        
        ```
        CompileFlags:
            Add: [-Wno-declaration-after-, -Wno-int-conversion, -Wno-all]
         
        Diagnostics:
            ClangTidy:
                Remove: bugprone-sizeof-expression
        
        ```
        
    2.  Reload Windows 后，Warning 消失不见。
*   小技巧:
    1.  cmd + T 全内核源码搜索 Symbols
    2.  cmd + Shift + o 当前文件搜索 Symbols  
        ![](https://bbs.pediy.com/upload/attach/202211/942658_R942ND9KGYA9VKR.png)

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

最后于 11 小时前 被 Ssage 泓清编辑 ，原因： 修正

[#基础理论](forum-161-1-117.htm)