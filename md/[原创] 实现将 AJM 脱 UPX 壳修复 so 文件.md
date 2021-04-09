> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-258604.htm)

实现将 AJM 脱 UPX 壳修复 so 文件
=======================

工具使用: IDA6.8, 010 Editor

开始前描述现象:
--------

将原 APP 解压出来的 libexec.so 放到 IDA 中加载发现打开的函数不能进行 F5 转为 C 代码

![](https://bbs.pediy.com/upload/attach/202004/732177_QWN5Q4EJ88GRU9Q.png)  

在该函数中按 F5 会弹出该提示, 使我们无法转为 C 代码进行进一步的分析

![](https://bbs.pediy.com/upload/attach/202004/732177_EUKE9VSXEJ7T683.png)  

1. 提取调试手机 linker 文件
-------------------

![](https://bbs.pediy.com/upload/attach/202004/732177_BGFJ8K43PQZKTP6.png)  

2. 分析 linker 文件获取加载 so 文件的偏移地址
------------------------------

在 String 表中直接搜索 calling, 选择第一个结果进入.

![](https://bbs.pediy.com/upload/attach/202004/732177_QCRNVWQQFSSFX8W.png)![](https://bbs.pediy.com/upload/attach/202004/732177_2H6S3DWUAAWE6DD.png)  

接着查看该信息的交叉引用, 选择第一个点击进入.

![](https://bbs.pediy.com/upload/attach/202004/732177_B4VDTMXB5MU9A8K.png)  

在调用字符串下面函数的偏移地址就是我们所需要的.

![](https://bbs.pediy.com/upload/attach/202004/732177_3ZJ5EQ8QBKUPDJP.png)  

3.dump so 操作
------------

经过 IDA 一波的附加操作, 待加载了 libexec.so 后即可开始干活了.

![](https://bbs.pediy.com/upload/attach/202004/732177_AJJTWAUVTFUUQ83.png)  

查看 linker 的基址

![](https://bbs.pediy.com/upload/attach/202004/732177_AY9MHREN6G5BJUD.png)  

计算绝对地址

0xB6F14000+0x2464= 0xB6F16464

G 键跳转进入并下断

![](https://bbs.pediy.com/upload/attach/202004/732177_EJCCZ4N2FFDYX2D.png)  

接着 F9 进入该函数, 待进来后继续 F9 一下.

![](https://bbs.pediy.com/upload/attach/202004/732177_DAMGDH92ZVASX5M.png)  

现在可以在 module list 中搜索 libexec.so

![](https://bbs.pediy.com/upload/attach/202004/732177_QMD5JYED4GR933C.png)  

使用脚本 dump libexec.so

![](https://bbs.pediy.com/upload/attach/202004/732177_FF4PZT6QURQP4G8.png)  

观望一眼 dump 出来的 so 文件

![](https://bbs.pediy.com/upload/attach/202004/732177_WDB6XYKT33JH2KN.png)  

使用 IDA 工具查看一下 dump 出来的 libexec.so, 大家可以看到导入导出表里面是空的. 其实这是 AJM 的变形壳. 接着下一步准备去修复 libexec.so

![](https://bbs.pediy.com/upload/attach/202004/732177_8GB6JDPB9TA79XN.png)![](https://bbs.pediy.com/upload/attach/202004/732177_47ZRXBYHABUFUHM.png)  

4. 根据 UPX 数据结构分析得出真实的数据大小
-------------------------

使用 010 Editoe 工具先加载 app 的原 libexec.so

![](https://bbs.pediy.com/upload/attach/202004/732177_C643RMJG5776VV2.png)  

接着搜索关键字”AJM!”, 选择第一个

![](https://bbs.pediy.com/upload/attach/202004/732177_6TPCCJBGWQZCHYB.png)  

接着使用 IDA 打开 app 里面的原 libexec.so, 待加载完毕后在导出表中搜索”.init_proc” 并进入

![](https://bbs.pediy.com/upload/attach/202004/732177_SQZDXGVJWSDVVE2.png)  

此位置就是 so 的开始位置

![](https://bbs.pediy.com/upload/attach/202004/732177_57VNXBANJNTM3PW.png)  

那么接着在 010 Editor 中可以看到刚才搜索的位置就在 0x11A20 开始

![](https://bbs.pediy.com/upload/attach/202004/732177_FG97EMKRD4FE3K3.png)  

其中结构体我这边是参考 一只流浪狗 大神写的  upx 分析原理 进行分析的

![](https://bbs.pediy.com/upload/attach/202004/732177_EWZRSDKV7M5WMBS.png)  

里面共有 3 个结构体共 36 个字节, 其实大家关注的地位就在第三个结构体中的 sz_cpr 字段的数据.

其数据为第二行后 4 字节. 即 0x91

那么计算压缩前数据的初始位置, 从魔数位置开始算起, 也就是 UPX 结构体的后面开始计算得出记录原始数据数据段大小的位置: 0x11A44+0x91= 0x11AD5

现在跟进 0x11AD5 的位置所在的 12 个字节数据就是对应压缩前, 压缩后的数据大小, 分别为 0x46A50 和 0x28B55

![](https://bbs.pediy.com/upload/attach/202004/732177_2Z5RYFKBFH978KE.png)  

5. 修复 libexec.so
----------------

现在知道了压缩前的数据大小了, 那么接着开始修复. 首先将之前在内存中 dump 出来的 libexec.so 拖到 010 Editor 里面使用 elf 结构解析脚本进行解析.

![](https://bbs.pediy.com/upload/attach/202004/732177_MRRQ4MZZUEG2DD3.png)  

修复的话只需修复 01~03 段即可.

01 段 只需要将之前得出的 0x46A50 转换为十进制进行将 01 段的数值替换即可

改前:

![](https://bbs.pediy.com/upload/attach/202004/732177_A8TYWP2MUYEAGVP.png)  

改后:

![](https://bbs.pediy.com/upload/attach/202004/732177_UMCYDW5E2M4GWBX.png)  

02 段: 将 offset 的值改为下面一行的 0x59E78 即可

改前:

![](https://bbs.pediy.com/upload/attach/202004/732177_FSS54X4TWRT4AJC.png)  

改后:

![](https://bbs.pediy.com/upload/attach/202004/732177_79EUZ6PBK58KMYA.png)  

03 段: 修改的地方与 02 段一致

改前:

![](https://bbs.pediy.com/upload/attach/202004/732177_TUJN6S7EYBSNYTM.png)  

改后:

![](https://bbs.pediy.com/upload/attach/202004/732177_Q7XK36KX8XHB6C3.png)  

Ok, 修改了这三处地方后记得保存, 目前到了这一步已经修复完了. 那么现在可以使用 IDA 来进行验证, 随便在导出表中找一个函数打开并且看能否进行 F5 转换 C 代码即可.

![](https://bbs.pediy.com/upload/attach/202004/732177_Q2KCQHUHWV7YBQX.png)![](https://bbs.pediy.com/upload/attach/202004/732177_9H7SUBNUXYA8FY5.png)  

具体样本在附件.

[[公告] 春风十里不如你，看雪团队诚邀你的加入！](https://mp.weixin.qq.com/s/bJEtd2Fu_MwEjUdkT4H5bQ)

最后于 2020-4-3 13:43 被 guanlingji 编辑 ，原因：

上传的附件：

*   [crame.apk](javascript:void(0)) （363.14kb，65 次下载）