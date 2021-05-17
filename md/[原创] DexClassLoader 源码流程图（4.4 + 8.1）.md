> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267596.htm)

简介
==

由于在 Android 逆向中偶尔能用到 DexClassLoader, 而且一代壳的加壳与脱壳也有其影子  
一直对 DexClassLoader 的细节不太清楚，所以抽时间看看，画了个图

 

先简要介绍一下大概流程：

*   如何告诉系统我要加载一个文件，要告诉文件的路径
*   如何加载，要告诉各个文件的属性
    
*   这时启动加载
    
    *   加载一个文件，就要构建它的文件类型。
        *   如果已经被优化为 ODEX（Android 4.4）或 OAT（Android 8.1），则从该对应文件加载 Dex
        *   如果没有被优化，则直接加载对应的 Dex
    *   当文件打开后，获得的文件结构中的各种数据结构。
        *   Android 4.4 根据是否已经被优化选择
            1.  直接映射并解析文件
            2.  对 Dex 文件进行加载到内存中进行优化验证后解析
        *   Android 8.1 根据是否为 oat 文件选择
            1.  直接打开 Dex 文件并验证。
            2.  从 oat 中获得 dex 文件打开并验证

我们在使用 DexClassLoader 时，特别是脱壳时，应用基本都是第一次启动，一般都会跳转到优化方向  
如果要成功脱壳，则需要 Dex 存在于内存中，所以定位的脱壳点就在将 Dex 进行分配内存映射完成后，并能够成功获取相应参数，如地址、大小等帮助我们成功 dump 出原 DEX

 

这仅仅是我的理解，如有错误，请大佬们不吝赐教

 

如果不清晰，已放附件

![](https://bbs.pediy.com/upload/attach/202105/752555_56CF6CG4KBX6QVD.png)

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 2 天前 被 Emiya 侍郎编辑 ，原因：

上传的附件：

*   [DexClassLoader.png](javascript:void(0)) （3.20MB，17 次下载）