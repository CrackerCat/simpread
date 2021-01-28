> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265692.htm)

本实例可以通过 app.any.run 获取：https://app.any.run/tasks/0feaad68-539d-4ef8-9090-f4e59e4d0c61/  
样本家族名称：Ransomeware/Paradise  
MD5：3e94ad15587dc71173bbd10bda5d56e4  
在 IDA 第一次解析 PE 文件时，往往会首先会咨询我们使用什么配置。它会默认忽略 Load resources 选项，该选项可以为我们提供更多信息，应当手动勾选。  
![](https://bbs.pediy.com/upload/attach/202101/784955_UBMZV5ADM386VXM.png)  
进入 OEP 按住空格键即可删除图形化界面从而切换到反汇编界面。从 OEP 处反汇编处可以看到病毒连续调用两个 call 指令。  
![](https://bbs.pediy.com/upload/attach/202101/784955_43J6KWAARD3UZM5.png)  
进入第一个 call 指令，看到熟悉的 fs:[0x30] 和相关的模块名，可以判断该函数和动态获取 API 有关。  
![](https://bbs.pediy.com/upload/attach/202101/784955_9NUM6CUTD95VTAR.png)  
仔细观察这段函数，可以发现病毒将一段 4 字节 16 进制数据和模块基址作为参数传入了 sub_401ca0。该函数返回的值存入一个全局变量地址 dword_406004。接下来每个加载 dll 相关的操作都和 dword_406004 函数相关，所以基本可以确定 sub_401ca0 函数用于根据模块基址和函数 hash 动态获取 API 地址。  
![](https://bbs.pediy.com/upload/attach/202101/784955_DHBWUS7CBF7JBJ5.png)  
查看 dword_406004 地址处数据发现为未知内存，则证实了 dword_406004 存储动态获取 API 地址。  
![](https://bbs.pediy.com/upload/attach/202101/784955_G6WNKV9Y6TRAHNP.png)  
修改参数名和函数名，使用交叉引用 X 快捷键产看 hash 参数被引用位置。  
![](https://bbs.pediy.com/upload/attach/202101/784955_PVUEMEP2M6SSV7W.png)  
找到 hash 引用上方调用了一个函数，而且紧接着会将返回值 eax 与 hash 参数进行比较，所以猜测 sub_4016f0 为 hash 运算相关的函数。  
![](https://bbs.pediy.com/upload/attach/202101/784955_QBS59SSCJDVWB9G.png)  
进入函数查看基本可以确定在进行 hash 运算。  
![](https://bbs.pediy.com/upload/attach/202101/784955_YWKE97NXBMTP8NR.png)  
接下来返回第一个函数起始位置向下浏览代码。发现有大量将数据存入临时变量的行为。其中有一些 ida 无法解析的数据，以 offset 开头。  
![](https://bbs.pediy.com/upload/attach/202101/784955_W7NCT63H5GX7CAP.png)  
双击进入可以看到一些全局数据。  
![](https://bbs.pediy.com/upload/attach/202101/784955_U4T4UER8D3ZFUDA.png)  
继续向下观察，可以看到病毒又在根据 hash 动态获取 API 的操作，而且每次传入的 hash 都是通过 esi 中取出，由此我们很容易联想到，病毒将输入按照一定规律连续存储在栈内，并通过循环获取不同的参数，最终将获取的 API 存储在 ecx 指向的内存。所以可以确定上面图中数据为要获取的 hash 值。  
![](https://bbs.pediy.com/upload/attach/202101/784955_S3U8DK589EXX99B.png)  
由此我们也理解了第一个函数是用来动态解析 API 的。  
![](https://bbs.pediy.com/upload/attach/202101/784955_RPG6EX66NDYNCX3.png)  
返回入口点代码变得更加直观。  
![](https://bbs.pediy.com/upload/attach/202101/784955_RVYG7TKBKT58R2D.png)  
我们继续向下观察代码，我们可以发现有很多 call 指向一个全局变量地址。这种代表间接调用 IAT 处函数。  
![](https://bbs.pediy.com/upload/attach/202101/784955_4WDKVEMCXWJH8D5.png)  
但是当我们点击该函数会发现 IDA 无法解析 API。因为病毒自己构造了自己的 IAT，需要我们动态对 IAT 表进行修复。  
![](https://bbs.pediy.com/upload/attach/202101/784955_GEWWHFD8BYQ8B6E.png)  
首先在动态解析 API 函数地点设置断点。  
![](https://bbs.pediy.com/upload/attach/202101/784955_X9ZG534CJG9ZW89.png)  
启动 IDA 的动态调试功能，我是用的被接管调试器为 Windbg。(也可以选择远程调试，全凭大家喜好)  
![](https://bbs.pediy.com/upload/attach/202101/784955_DEA3FU2GQKTWFRX.png)  
在病毒填充 IAT 位置设置断点。并让程序跑起来直到断住。  
![](https://bbs.pediy.com/upload/attach/202101/784955_B8RFB4M9RBMH2YB.png)  
断住后继续向下执行一条，让导入表被 EAX 填充。然后打开一个新的反汇编窗口。  
![](https://bbs.pediy.com/upload/attach/202101/784955_4V7CV6W6EY6AD42.png)  
通过新的反汇编窗口查看 IAT 表。  
![](https://bbs.pediy.com/upload/attach/202101/784955_KR43TEH8MA9KTMF.png)  
可以看到 API 数据已经动态填充到了 IAT 表中.  
![](https://bbs.pediy.com/upload/attach/202101/784955_62MRCBCG94CRQEZ.png)  
接下来选择 API 的地址，按下 O 快捷键，让 IDA 解析出符号名。  
![](https://bbs.pediy.com/upload/attach/202101/784955_PYGRN4BZE29YUKA.png)  
了解重新解析 IAT 的方法后，我们只需要在动态获取 API 函数的 ret 指令下断点，让其自动解析。  
![](https://bbs.pediy.com/upload/attach/202101/784955_Q28XBYGHM83RREH.png)  
执行到 ret 指令查看 IAT 表内存，已经填充完毕。  
![](https://bbs.pediy.com/upload/attach/202101/784955_FMQ9FJNJC6XTY8D.png)  
我们可以通过手动解析来验证一下是否填充完整。  
![](https://bbs.pediy.com/upload/attach/202101/784955_M5ZGCYZRFE9ARPY.png)  
接下来我们需要记录 IAT 表的起始地址和结束地址。

```
01175FA8 01176014

```

然后选择 Edit->Universal Unpacker Manual Recounstruct 通用脱壳插件。  
![](https://bbs.pediy.com/upload/attach/202101/784955_RCGYYKWSHNVXW5Q.png)  
分别填写 OEP、代码节起始位置和结束位置、IAT 表起始位置和结束位置。  
![](https://bbs.pediy.com/upload/attach/202101/784955_WYMWUZQY5PVN5FN.png)  
修复 IAT 完成后可以看到 IDA 已经解析了所有 IAT 表中的 API 符号。接下来我们就可以终止调试了。  
![](https://bbs.pediy.com/upload/attach/202101/784955_NY8FUNNKDPTMJS8.png)  
结束调试后使用 Rebase program。  
![](https://bbs.pediy.com/upload/attach/202101/784955_HYEW9V8XZ982W8Q.png)  
将 IDB 的基址重新定向回 0x00400000。  
![](https://bbs.pediy.com/upload/attach/202101/784955_5KSAJ36PX5WR4MY.png)  
![](https://bbs.pediy.com/upload/attach/202101/784955_8E9UY57JFH3VN45.png)  
可以看到 IDA 已经可以解析病毒构造的 IAT 表中 API 地址。  
![](https://bbs.pediy.com/upload/attach/202101/784955_VF5798JM6HCKA84.png)

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)

最后于 2 小时前 被独钓者 OW 编辑 ，原因：