> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [vul.360.net](https://vul.360.net/archives/83)

> 2021 年 4 月 14 日至 15 日，卡巴斯基技术检测到一波针对多家公司的高度针对性攻击。

### **前言：**

_2021 年 4 月 14 日至 15 日，卡巴斯基技术检测到一波针对多家公司的高度针对性攻击。更仔细的分析表明，所有这些攻击都利用了一系列 Google Chrome 和 Microsoft Windows 零日漏洞 (CVE-2021-31956）。虽然我们无法在 Chrome 网络浏览器中检索用于远程代码执行 (RCE) 的漏洞，但我们能够找到并分析用于逃离沙箱并获取系统权限的特权提升 (EoP) 漏洞，并且它利用了两个不同的 Microsoft Windows 操作系统内核。_

——_引用自卡巴斯基 [1]_

该漏洞使用了 WNF 模块来完成任意地址读写以及越界读写操作，所以笔者最近对 Windows Kernel 的 WNF 模块进行了一系列研究，发现 WNF 模块对于漏洞利用有非常大的帮助（Paged Pool）。

本文对 Windows 中的 WNF 在实用功能上不做讨论，仅对我们关心的漏洞利用进行讨论。

### **漏洞利用：**

在逆向 NtosKrnl.exe 后笔者发现，WNF_Object 结构是保存在 Paged Pool 中的一个固定大小 (0xC0|0xD0) 的内核池，该结构由函数 Nt!ExpWnfCreateNameInstance 创建并初始化，该函数在 R3 中也可以通过 Ntdll!NtCreateWnfStateName 调用。

![](https://vul.360.net/wp-content/uploads/2021/07/image-1.png)

![](https://vul.360.net/wp-content/uploads/2021/07/image-2.png)

在 WNF_Object 结构中最重要的就是 WNF_Object+0x58 处的 StateData 结构指针，该指针在 WNF_Object 结构初始化时并未定义，而是在后续使用 nt!ExpNtUpdateWnfStateData 函数更新 StateData 时使用了 nt! ExpWnfWriteStateData 函数更新 StateData 数据时经过一些判断，来确定是否 创建 Or 修改 StateData 的内容。

![](https://vul.360.net/wp-content/uploads/2021/07/image-3.png)

StateData 结构的大小为我们传入的数据内容 + 0x10，这里的 0x10 为方便管理 Data 所创建的，结构如下。（其中 StateData 为动态长度）

Struct StateDataObject {

   +0x00 UINT Flag;

   +0x04 UINT MaxLength;

   +0x08 UINT UseLength;

   +0x0c UINT NON;

   +0x10 CHAR StateData[1];

};

#### 有限制的任意地址读写以及相对应的越界读写：

也就是说只要修改了 WNF_Object+0x58 处的 StateData 指针，即可获取一个有限制的任意地址读写。

#### 有限制的任意地址写：

       修改 StateData 指针之后获取一个有限制的任意地址读写的原因是因为 WNF_Object+0x58 处保存的是 StateData 的结构指针，而不是单纯的数据，其结构在前几节已经介绍过了，具体修改 StateData 的代码如下。（nt! ExpWnfWriteStateData）

![](https://vul.360.net/wp-content/uploads/2021/07/image-4.png)

   可以看到复制时不单单复制了 StateData 的内容，并且会相对应的修改目前使用的长度以及其他信息，所以笔者认为这是获取到了一个有限制的任意地址写。

#### 有限制的任意地址读：

既然任意地址写都有限制了，那任意地址读是否也是如同任意地址写一样，是有限制的呢？ 答案是，确实是这样的，任意地址读也有一些相应的限制，请看代码。（nt!ExpWnfReadStateData）

![](https://vul.360.net/wp-content/uploads/2021/07/image-5.png)

   在读取时，虽然不会修改当前 StateData 结构的任何属性，但是该函数会完整的读取 StateData 数据段的全部内容。

也就是说如果我们将 StateData 的指针修改后用其进行任意地址读则需要注意 StateData 的 UseLength, 他的长度则代表了复制的总长度，如果我们 Copy 的长度低于 UseLength，则会返回错误代码。

#### StateData Object 的越界读写：

   与此同时笔者也发现既然任意地址读写的限制都由 StateData 结构中 MaxLength 、UseLength 的引发，但是换一种思路思考的话，也可以推断出如果修改了 MaxLength 以及 UseLength，可以获取一个无限制的越界读写（OOBRW）。

如果布局得当则可以在 Ntoskrnl 模块中复现 Win32k 经典利用手法 BitMap 以及 Palette 在 Windows10 1709 之前的骄傲成绩。

#### WNF_Object 信息泄露造成的危害：

       在 WNF_Object 创建时，会在 WNF_Object + 0x98 处 存放当前进程 EPROCESS 的地址，只要布局得当，则可以使用前几节的内容 StateData 的越界读并联合其任意地址读写，读出当前进程 EPROCESS 的地址，并使用任意地址读来读取 EPROCESS->Token 的指针来获取当前进程 Token 的地址，对其使用任意地址写来覆盖 Token 中的 SEP_TOKEN_PRIVILEGES 结构位来开启当前进程所有的权限，来完成一次完整的漏洞利用，甚至不需要任何一个信息泄露的漏洞！！！！

![](https://vul.360.net/wp-content/uploads/2021/07/image-6.png)

![](https://vul.360.net/wp-content/uploads/2021/07/image-7.png)

![](https://vul.360.net/wp-content/uploads/2021/07/image-9-1024x555.png)

[1] 卡巴斯基对该漏洞的介绍：[https://securelist.com/puzzlemaker-chrome-zero-day-exploit-chain/102771/](https://securelist.com/puzzlemaker-chrome-zero-day-exploit-chain/102771/)