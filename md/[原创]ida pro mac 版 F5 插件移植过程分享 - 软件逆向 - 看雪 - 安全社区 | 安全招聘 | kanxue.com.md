> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-279736.htm)

> [原创]ida pro mac 版 F5 插件移植过程分享

[原创]ida pro mac 版 F5 插件移植过程分享

1 小时前 226

### [原创]ida pro mac 版 F5 插件移植过程分享

* * *

因为今天看到了隔壁论坛发布的 8.3 全插件版，因此把之前迁移 mac 版 F5 插件的流程发出来，抛砖引玉，希望对有需要人有点帮助吧。

*   环境：
    *   ida pro 7.6 macos arm64 版 带 x86/64，arm32/64 插件
    *   ida pro 8.3 macos arm64 版 带 arm32/64 插件
*   目标:
    *   把 ida pro 7.6 的 x86/64 插件移植到 ida pro 8.3
    *   不需要重算 key 来增加对 x86/64 插件的支持

首先，把 ida7.6 的 hexx64.dylib 直接复制到 8.3 的 plugins 文件夹。此时打开 ida，直接 crash  
![](https://bbs.kanxue.com/upload/attach/202312/749597_8X4R8HQ2S67UZXM.webp)

使用 lldb 调试一下，触发错误：  
![](https://bbs.kanxue.com/upload/attach/202312/749597_3F9PQT63W6QUC9U.webp)  
看一下崩溃位置：  
![](https://bbs.kanxue.com/upload/attach/202312/749597_SHZP8NPK6KHRDD5.webp)  
此函数就是 dlsym 的一个 wrapper，是从 libida64.dylib 加载函数。按理说应该不会出问题，现在出了问题，大概率是动态导入函数的时候，出了问题。交叉引用，向上看一下

![](https://bbs.kanxue.com/upload/attach/202312/749597_ZDGPSSC3TW7V3P8.webp)

发现一个要加载函数的列表  
![](https://bbs.kanxue.com/upload/attach/202312/749597_MC9J4F8VSAHBSVM.webp)

通过 `nm libida64.dylib`之后，发现 8.3 的 libida64 并没有导出红框部分的函数。但是下面 3 个函数导出了，patch 吧，感觉有点难受，不知道下面的 3 个函数用不用了。  
看一下保存位置的数组吧：  
![](https://bbs.kanxue.com/upload/attach/202312/749597_5E7UUT4SRT5XG34.webp)  
正好，看了一下保存的数据，交叉引用发现，红框部分的函数，都没有使用，非常 nice，但是不知道 ida 这是啥意思

此时再次回到崩溃位置：  
![](https://bbs.kanxue.com/upload/attach/202312/749597_4N3ZX9D8CJ8U2AR.webp)  
只有这里判断了返回值的正确，直接 patch 掉就行。

![](https://bbs.kanxue.com/upload/attach/202312/749597_9ZVEPY487JYM3N9.webp)

此时打开 ida8.3，出现了下图的提示：  
![](https://bbs.kanxue.com/upload/attach/202312/749597_5RDV4HJUFY5JD9T.webp)

NICE！！！这是一个号兆头，至少插件正常加载了。  
这个部分的提示，应该在 libida64 里面，进去看一下有没有搞头

没有字符串我可怎么活啊！！！  
字符串启动！

![](https://bbs.kanxue.com/upload/attach/202312/749597_4RXBBUJXMD4XMU9.webp)

![](https://bbs.kanxue.com/upload/attach/202312/749597_6C9UHC2PRUQ8474.webp)

这个引用在`get_license_info`函数内。这个函数太长了，看了一眼长度就够够的了。  
因此我决定看一下这个函数的返回值的作用，交叉引用看几个就行了：

![](https://bbs.kanxue.com/upload/attach/202312/749597_TTT283BH7P2RAN4.webp)  
![](https://bbs.kanxue.com/upload/attach/202312/749597_4D6TT8URA7ZRC3F.webp)  
意思就是这个函数的返回值!=0，说明加载的 license 就会报错，因此直接把返回值改成 0 试试看。  
![](https://bbs.kanxue.com/upload/attach/202312/749597_7FZTXSBG5SQ7PS8.webp)  
![](https://bbs.kanxue.com/upload/attach/202312/749597_SYGGJTQQJYX2Q5S.webp)

> 这是不是就是之前说的 ida 1 字节 patch

GOGOGO！此时打开 ida8.3，F5 成功了！  
![](https://bbs.kanxue.com/upload/attach/202312/749597_HAX3NTGV22D82YX.webp)  
此时：  
![](https://bbs.kanxue.com/upload/attach/202312/749597_FUPEYNR5SAE6JGG.webp)

32 位同理。

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

最后于 1 小时前 被八岛编辑 ，原因： 我尼玛看雪你这个编辑器能不能行了，又得重新编辑一次