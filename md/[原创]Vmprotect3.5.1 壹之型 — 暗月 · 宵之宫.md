> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271546.htm)

> [原创]Vmprotect3.5.1 壹之型 — 暗月 · 宵之宫

在很久很久以前，，神奇的 nooby 有了个天才的想法，通过修改 themida 的引擎，使其强制输出了没有混淆和加密的程序，很轻松的就分析完了外壳逻辑和 vm 的逻辑，盖亚。  
![](https://bbs.pediy.com/upload/attach/202202/841912_74ZPPWPY7UA9UF5.png)  
通过 nooby 的想法，我们也将 vmp3.5.1 的引擎进行了修改，使其强行输出了没有混淆和 vm 后的代码，便于我们分析外壳和 handler 的逻辑, 今天先来看看 vmp 的反调试原理，先将 vmp 配置成如下，避免其他功能的干扰。  
![](https://bbs.pediy.com/upload/attach/202202/841912_Y5URQ4RU5WKUVSF.png)  
保护后得到了非常干净的程序。  
![](https://bbs.pediy.com/upload/attach/202202/841912_2JQ57XYMV6FZHQD.png)

 

![](https://bbs.pediy.com/upload/attach/202202/841912_KBK729MQFZU39VH.png)  
如图，这时的入口不再是骇人的 push 0xXXXXXX / call xxxxxx  
而是

```
int start()
{
  if ( (unsigned int)sub_4F4664() == 1 )
    return mainCRTStartup();
  sub_4F44EC();
  return 0xDEADC0DE;
}
```

sub_4F4664() 是一个非常大的函数，vmp 整个外壳的逻辑所在，内存保护，导入表保护，资源保护，压缩等等就是在这个函数中处理的。  
第一步，先获取 ntdll 的版本信息  
![](https://bbs.pediy.com/upload/attach/202202/841912_CYGYJH9PWWT3KA7.png)  
根据 ntdll 的版本初始化一些 HardCode，后面会用到  
![](https://bbs.pediy.com/upload/attach/202202/841912_CFDWHE8YM2SZ288.png)  
接着我们直奔诸葛亮三轮车, vmp 自己封装了一个从模块的导出表直接得到地址的函数，这种方式在 shellcode 中比较常见。  
![](https://bbs.pediy.com/upload/attach/202202/841912_QW6QSTUCK5WC7JC.png)  
比较完善，还还考虑了转发的情况  
![](https://bbs.pediy.com/upload/attach/202202/841912_TZXW7JQYXJSSK2Y.png)  
接着通过 GetExportAddress 和上面通过 ntdll 定位的 syscall 序号进行反调试检测。  
![](https://bbs.pediy.com/upload/attach/202202/841912_67JDFGFMFZGU7W3.png)  
vmp 分别通过了  
IsDebuggerPresent,CheckRemoteDebuggerPresent, NtQueryInformationProcess, 以及 ZwSetInformationThread 进行用户态反调试，vmp 在调用函数之前，会检测头部是不是 0xCC, 然后直接报错。  
还有就是直接 syscall 直接调用比较有效。

 

![](https://bbs.pediy.com/upload/attach/202202/841912_4J58X62ACM9NRCS.png)  
那么，剩下的贰之型 · 珠华弄月再说..

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 2022-2-19 02:50 被冰鸡编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#VM 保护](forum-4-1-4.htm)