> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xtuly.cn](https://xtuly.cn/article/ndk-load-llvm-pass-plugin)

> 免编译 LLVM，使用 NDK 加载 pass 插件（Linux 和 macOS）

本文基于 ndk r25c （25.2.9519653）

[](#95f19cabf5934adeac1b13e0c54c405e "获取ndk r25c")获取 ndk r25c
-------------------------------------------------------------

### [](#d44f5b3ac1ab451ebd40962d38df3aec "直接下载")直接下载

### [](#3fa1173045b645f1bcfc020911d475e3 "使用sdkmanager安装")使用 sdkmanager 安装

> 坑：archlinux 需要 `sudo archlinux-java set java-8-openjdk`

[](#58f096894b524782af6d932703635e71 "查看clang信息")查看 clang 信息
------------------------------------------------------------

内容：

[](#7044e68d03df4fb0a874ae4dbfbea149 "下载未精简的clang")下载未精简的 clang
---------------------------------------------------------------

![](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F342a29a7-6a23-4674-9b83-23015fa45bb7%2FUntitled.png?table=block&id=4f88692c-88b5-477b-a9ad-81eb962c8410)

然后去 Google 的 prebuilt clang 仓库找到 r450784d1 相关的分支并打开：

![](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fa2476bc5-1300-4e79-afca-b7672adc9acd%2FUntitled.png?table=block&id=341a9b55-2d02-4308-bebb-8444ee91e777)

点开最新的一个 commit，进入

然后点击 tgz 下载这一份 clang

![](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F914c728f-0335-4ee0-8b57-4660fb5dd387%2FUntitled.png?table=block&id=feb4c365-2581-4959-972d-97acd41c8f72)

然后基于这一份 clang 直接编译 llvm 动态库插件就可以直接用 ndk 加载了

解压

[](#dfc73c86206c43f5b45fbe469a50acbe "编译使用pass实例")编译使用 pass 实例
--------------------------------------------------------------

1.  下载代码

1.  修改 CMakeLists.txt，在`project()`后加上

这个时候`cmake ..`会报错

这个时候去注释掉`clang/lib64/cmake/llvm/LLVMExports.cmake` 下面这一段代码

![](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fb6dd05d5-4d4b-4b1c-85fa-2749fff1403c%2FUntitled.png?table=block&id=05f250c5-1dd4-402a-8e70-311e81201e4b)

然后

![](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F5ad76a3d-a2d8-4ce2-8b1f-24f96f8c5554%2FUntitled.png?table=block&id=bce8b430-67ff-44f0-9add-60cda7f8d67d)

build.sh:

编译后：

![](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F62d9e7fc-a5fe-4053-8c58-d7d7b5b4c4b7%2FUntitled.png?table=block&id=98e288fc-9a90-4065-8f49-3878d65ab8d2)

差不多鸟～

[](#1cd4a9d4be1347a1955cb3c5ed012a40 "当我们来到macOS上")当我们来到 macOS 上
----------------------------------------------------------------

由于 Google 编译 macOS 工具链的系统版本太低，加载 so 的时候会报错

解决办法：

使用下载的 clang 替换掉 ndk 里面的 clang

找不到头文件是 macOS 的问题，修改 `build.sh` 为

后重新编译，成功混淆

![](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F59f5fac3-78e8-4d08-bf73-527a93b0fa25%2FUntitled.png?table=block&id=97bceb32-7742-4387-919a-5657736ea78f)

[](#d883eeb2410d4aff9b945291083704ac "参考")参考
--------------------------------------------