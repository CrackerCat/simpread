> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1839317-1-1.html)

> 有很多人在咨询工具的使用方案，这里通过我写的一个 CrackMe 来演示几种异常补丁方案的使用。

![](https://avatar.52pojie.cn/data/avatar/000/14/47/47_avatar_middle.jpg)Nisy _ 本帖最后由 Nisy 于 2023-9-28 16:52 编辑_  
有很多人在咨询工具的使用方案，这里通过我写的一个 CrackMe 来演示几种异常补丁方案的使用。  
源码中的判断函数：  

1.    
    
2.    
    
3.  DWORD __stdcall NsCheckInputThread(LPVOID lpThreadParameter)  
    
4.  {  
    
5.          HWND hWnd = (HWND)lpThreadParameter;  
    
6.          CHAR szBuffer[MAX_PATH] = {0};  
    
7.          int nStrLen = GetDlgItemTextA(hWnd, IDC_EDIT, szBuffer, MAX_PATH);  
    
8.          if (nStrLen){  
    
9.                  unsigned char szSrc[0x10];  
    
10.                  unsigned char szDest[0x10];  
    
11.                  CalcMD5((unsigned char*)szBuffer, nStrLen, szSrc );  
    
12.    
    
13.                  *(PDWORD)((PDWORD)szDest + 0) = 0x7de2f924;  
    
14.                  *(PDWORD)((PDWORD)szDest + 1) = 0x058698b1;  
    
15.                  *(PDWORD)((PDWORD)szDest + 2) = 0xe2c6cdc6;  
    
16.                  *(PDWORD)((PDWORD)szDest + 3) = 0x02a1bd7b;  
    
17.    
    
18.                  if(memcmp( szSrc, szDest, 0x10) == 0 ){  
    
19.                          MessageBox(hWnd, L"Success!",L"Info", 0);  
    
20.                          return 1;  
    
21.                  }  
    
22.          }  
    
23.          MessageBox(hWnd, L"Error!",L"Info", 0);  
    
24.          return 0;  
    
25.  }  
    
26.    
    

_复制代码_我们编译源码后，对 x64 产物 使用 VMP3.8 加壳保护，比较函数未调用 VMP 保护，调试器载入，搜索字符串来到比较关键点：  
![](https://attach.52pojie.cn/forum/202309/28/163138bat2uut2etccettk.png)

**400.png** _(112.72 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjY0NjEyNHxlY2YxYzAxNnwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3&nothumb=yes)

2023-9-28 16:31 上传

  
用 Baymax 对付加壳的程序，第一步要先找到壳的解码时机，通俗来说，就是要通过 HOOK API 等方式等待被补丁的地址数据解码，如果使用 HOOK API 方案，可以调用壳自身使用的 API，或者在执行到模块被补丁地址前调用过的 API 均可。由于这个 CrackMe 获取窗口输入文本调用的 API 为 GetWindowTextA，所以我们就直接 HOOK 这个 API 了，这样加任何壳补丁方案都可以通杀了。  
![](https://attach.52pojie.cn/forum/202309/28/162039bev13a4094cxfyjb.png)

**001.png** _(24.57 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjY0NjEyNXxjNzJjN2QxMXwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3&nothumb=yes)

2023-9-28 16:26 上传

  
由于代码中调用了 VMP 壳内存校验的函数，所以不要直接 Patch 汇编指令，而要使用异常机制来绕过内存校验实现补丁。  
**方案一、修改标志寄存器实现比较结果为真**  
特征码 1：85 C0 75 23 45 33 C9 ，偏移为 2  
注意：我们在使用特征码搜索地址时，一定要确保其唯一性，比如可使用插件来查找结果  
![](https://attach.52pojie.cn/forum/202309/28/163145esl3b3q3tykfbby3.png)

**401.png** _(74.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NjEzMnwxYjk0NzIyMnwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3&nothumb=yes)

2023-9-28 16:31 上传

  
设置 ZF 标志位为 1 即可。  
![](https://attach.52pojie.cn/forum/202309/28/162207l9trp9oy6z65ky9v.png)

**002.png** _(38.44 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjY0NjEyN3w2NDJhNjc1ZXwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3&nothumb=yes)

2023-9-28 16:26 上传

  
最后，异常设置这里使用对当前线程设置异常即可，不要勾选对所有线程设置异常，会被 VMP 检测到被调试。  
![](https://attach.52pojie.cn/forum/202309/28/162256nj3jau8ajtiuujhh.png)

**003.png** _(59.31 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NjEyOHwxYWEzMjUxNXwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3&nothumb=yes)

2023-9-28 16:26 上传

  
**方案二、修改 memcpy 比较函数的参数**  
00007FF74A7017DC | 41:B8 10000000          | mov r8d,10                                      |  
00007FF74A7017E2 | 48:8D9424 50010000      | lea rdx,qword ptr ss:[rsp+150]                  | rdx:EntryPoint  
00007FF74A7017EA | 48:8D8C24 60010000      | lea rcx,qword ptr ss:[rsp+160]                  |  
00007FF74A7017F2 | E8 99940000             | call testcheckmemory64.7FF74A70AC90             | memcmp 比较函数  
00007FF74A7017F7 | 85C0                    | test eax,eax                                    |  
00007FF74A7017F9 | 75 23                   | jne testcheckmemory64.7FF74A70181E              | 关键跳转  
特征码地址： E8 ?? ?? ?? ?? 85 C0 75 23  
该方案比较简单，直接将 RDX 寄存器数值修改为 RCX 即可。  
![](https://attach.52pojie.cn/forum/202309/28/162350r4uqto4u6qa6qo4j.png)

**101.png** _(35.45 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjY0NjEyOXwxODNiZGM0NXwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3&nothumb=yes)

2023-9-28 16:26 上传

  
**方案三、修改参与比较的 MD5 内存数据**  
特征码地址： E8 ?? ?? ?? ?? 85 C0 75 23  
直接将 RDX 寄存器指向的内存的数据 修改为 RCX 指向内存的数据即可, 长度为 0x10。// 注意 baymax 输入的数值均为 HEX 数值  
![](https://attach.52pojie.cn/forum/202309/28/162510d5xmifyf55yyzama.png)

**201.png** _(49.6 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjY0NjEzMHxlMjg0OWE0ZnwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3&nothumb=yes)

2023-9-28 16:26 上传

  
**方案四、修改 memcpy 函数返回值**  
我们进入 memcmp 函数：  
00007FF74A70AC90 | 48:2BD1                 | sub rdx,rcx                                     | rdx:EntryPoint  
00007FF74A70AC93 | 49:83F8 08              | cmp r8,8                                        |  
00007FF74A70AC97 | 72 22                   | jb testcheckmemory64.7FF74A70ACBB               |  
00007FF74A70AC99 | F6C1 07                 | test cl,7                                       |  
00007FF74A70AC9C | 74 14                   | je testcheckmemory64.7FF74A70ACB2               |  
00007FF74A70AC9E | 66:90                   | nop       
提取函数入口特征码：48 2B D1 49 83 F8 08 72 22 F6 C1 07 74 14  
1、勾选修改函数返回值，设置函数返回时修改类型，栈偏移为 0。  
2、 设置 RAX 寄存器为 0，即 memcmp 函数返回 0，比较内存数据相同。  
3、条件断点 参数 r8d == 0x10  
![](https://attach.52pojie.cn/forum/202309/28/162649i8tgnzqql6fbtign.png)

**301.png** _(47.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NjEzMXwwYmU5OGFhZXwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3&nothumb=yes)

2023-9-28 16:26 上传

  
**目标程序 + 源码 + 补丁方案下载：**  
链接：[https://pan.baidu.com/s/1afZOKMIdKUFI5vLbt4jmBg](https://pan.baidu.com/s/1afZOKMIdKUFI5vLbt4jmBg)  
提取码：cgr5  
解压密码：Baymax64  

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[baymax64 Patch(bpt files).zip](forum.php?mod=attachment&aid=MjY0NjEzM3xjYjM0MGFlYnwxNjk2NjU1NTMzfDIxMzQzMXwxODM5MzE3)

2023-9-28 16:09 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

2.75 KB, 下载次数: 41, 下载积分: 吾爱币 -1 CB![](https://avatar.52pojie.cn/images/noavatar_middle.gif)秋名山

> [fu920821 发表于 2023-9-28 16:33](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=48120932&ptid=1839317)  
> 本帖最后由 Nisy 于 2023-9-28 16:31 编辑

用心评论！账号来之不易！请珍惜。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)yyb1813 _ 本帖最后由 yyb1813 于 2023-9-28 17:53 编辑_  
好久不见大佬发帖了，过来看看，虽然现在用不上，友情支持一下大作也是必须的，VMP 的壳误杀太多，不太好过，但不知为什么，我用这个在 11 平台总卡，10 系统却不会。![](https://avatar.52pojie.cn/data/avatar/000/17/00/81_avatar_middle.jpg)565266718 支持一个。。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)qqycra 大佬发帖，前来学习 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) wolaikaoyan 高手，多谢楼主的分享，用心之作 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) LuckyClover 真的是优秀啊 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) wasm2023 楼主 yyds ![](https://avatar.52pojie.cn/data/avatar/001/58/56/25_avatar_middle.jpg) 无知灰灰 Nisy 大大这些年好像低调很多。。。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)ddwwtom 学习一下