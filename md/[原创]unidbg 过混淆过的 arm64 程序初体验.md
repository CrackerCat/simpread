> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268376.htm)

> [原创]unidbg 过混淆过的 arm64 程序初体验

一、程序初分析  
1、先 ida 反编译，结果如下 ![](https://bbs.pediy.com/upload/attach/202107/926377_2GSNZU9HS5ZBFCC.png) ![](https://bbs.pediy.com/upload/attach/202107/926377_VZ6B724ENPJ24XU.png)  
可见有明显的混淆，无法看清程序逻辑。  
查看 String，如动图所示，只发现程序中有用到 socket、gettimeofday 导入函数（怀疑是网络验证），其中还发现 123456789 等简单明文，怀疑是解密其他字符串用的秘钥。  
![](https://bbs.pediy.com/upload/attach/202107/926377_QXJEW2FN83CKSUU.gif)。  
2、尝试分析  
![](https://bbs.pediy.com/upload/attach/202107/926377_VDRWYNG9JUJDQEZ.png)  
![](https://bbs.pediy.com/upload/attach/202107/926377_95HQQFZV52TAGAX.png)  
![](https://bbs.pediy.com/upload/attach/202107/926377_QFJSGZUY22GTUZU.png)  
这几个函数跟入，感觉混淆强度很大未发现有靠谱的线索。  
二、unidbg 调试解难局  
1、模拟执行获得新思路。  
由于这是一个未知程序，不敢贸然使用真机 + fridahook+ida 调试（其实我感觉 Ida 跑混淆或者有壳的很容易跑丢了，万一程序执行了问题代码，可能会被刷机掉），故拿出了 unidbg 进行调试：  
①选定要调试的程序路径并建立 arm64 模拟器  
![](https://bbs.pediy.com/upload/attach/202107/926377_Q2JTSYGWQC233ST.png)  
②对 socket 导出表进行 hook 与 wireshark 抓包联合获得程序执行逻辑，证实进行网络校验，后由于校验不通过退出程序。  
1、ida 静态再分析。  
找到 connect 导入表，按 x 进行交叉引用查看。 ![](https://bbs.pediy.com/upload/attach/202107/926377_KZBEFKS33QYNZAZ.png)  
发现有两条分支：  
①第一条分支 sub_3E108  
![](https://bbs.pediy.com/upload/attach/202107/926377_E5X8FUD9HDGAHRZ.png)  
我们只需要关注第二个参数，得知与 a1 有关，向上追溯，按 x 交叉引用得到 sub_3e01c.  
![](https://bbs.pediy.com/upload/attach/202107/926377_R9HUWXU7ZZP7U6T.png)  
继续追溯  
![](https://bbs.pediy.com/upload/attach/202107/926377_QNR92TAVD7RQ7WA.png)

 

继续追溯得到该函数 ![](https://bbs.pediy.com/upload/attach/202107/926377_RKH5EEYHHFV7XHQ.png)  
虽然这里是伪 C，但是有过 C 开发可以立刻看出这里的突破点，这个就是 sockaddr_in 的 sin_addr 进行赋值。关注 a1[2]。得知 a1[2] = v2;  
观察 v2 = operator new(0x200078uLL);  
sub_3AAD0(v2, a1); 有理由猜想 sub_3AAD0 执行后对 v2 进行赋值，毕竟这个是伪 c 呀，而且还混淆了。跟进 sub_3AAD0 。  
sub_3aad0 关键代码如图所示： ![](https://bbs.pediy.com/upload/attach/202107/926377_B53EYGF4FRCHRKF.png)  
不过被加密了。  
![](https://bbs.pediy.com/upload/attach/202107/926377_HXX5BA8QKYMZZCR.png)  
这里我突然想到了 init_array 这里是可以进行加入 decode 函数的，在程序一开始对 bss 段解密，也就是说静态看是密文，运行时是明文。  
于是拿出 unidbg，在 main 开头加断点

```
debugger.addBreakPoint(module, 0x22F5C);
```进行dump内存    ![图片描述](upload/attach/202107/926377_PE3QUSH26PYES35.png)

```

System.err.printf(String.valueOf(emulator.getMemory().pointer(module.base+0).dump(0xfc470,100)));  
```  
![](https://bbs.pediy.com/upload/attach/202107/926377_3RZFY6VZ5DYHW4K.png)  
很不幸这个就是个 127.0.0.1 的，于是断定网络验证在分支 2.  
②第二条分支 sub_424E4  
ida 刚一点击，就报错了 ![](https://bbs.pediy.com/upload/attach/202107/926377_VS5YRUV6FT2ZSFZ.png)  
原因未知。（希望大佬留言解决我用的是 ida7.5）  
于是到直接不看伪 C 直接 X 交叉引用追溯。  
![](https://bbs.pediy.com/upload/attach/202107/926377_SYGHJBV4DRA7HRD.png)  
F5 查看 ![](https://bbs.pediy.com/upload/attach/202107/926377_T6SK4GU6CMKBQCS.png)

 

这里直接解数据得了 sub_424E4 这块绝对网络验证的返回值，返回值校验应该就在该函数不远处了。  
直接追溯该函数， ![](https://bbs.pediy.com/upload/attach/202107/926377_V4FFYZWCHBJRJRJ.png)

 

**  
到这里终于看到了曙光**  
ida 给我做了个标识：nptr  
![](https://bbs.pediy.com/upload/attach/202107/926377_3AS7JEPQCNQ84NH.png)  
![](https://bbs.pediy.com/upload/attach/202107/926377_QD9E23RF2W92KQP.png)  
这里我们有理由相信标注处就是校验代码  
![](https://bbs.pediy.com/upload/attach/202107/926377_7S6SHZVSZJMU8HP.png)。

 

分析到了这里，程序全 pj 也就很是简单了，可以 crack 内存截取（mmap、process_vm_writev 这类函数）修改或者网包修改或者其他一堆方法，我就不举例说明了。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 15 小时前 被 wx_Fruits Basket 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm)