> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2121326-1-1.html)

> [md]# Binary Ninja 5.3.9434 Personal 授权逻辑分析 ## 前言前两天在吾爱上看到有大佬分享了这个软件的 Windows 版以及破解补丁（），就想着研究一下到底是怎么破 ......

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)andy_wang425

Binary Ninja 5.3.9434 Personal 授权逻辑分析
-------------------------------------

### 前言

前两天在吾爱上看到有大佬分享了这个软件的 Windows 版以及破解补丁（[https://www.52pojie.cn/thread-2105125-1-1.html](https://www.52pojie.cn/thread-2105125-1-1.html)），就想着研究一下到底是怎么破解的。本来想原汤化原食，用 Binary Ninja 自己分析自己，但 Binary Ninja 分析大文件耗时特别长（也可能是我设置调的不对），等半天终于反编译完了保存数据库时还崩溃了，所以最后还是转向 IDA Pro。这是我第一次逆向一个正式的 PE 软件，以静态分析为主，欢迎大佬批评指正。

### 过程

#### 找到关键函数 BNIsLicenseValidated

安装完 Binary Ninja，运行 `binaryninja.exe` 时会提示我们需要许可证：

![](https://attach.52pojie.cn/forum/202608/05/070348qk576kk7156g1414.png)

**license-required.png** _(43.4 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU4Mnw3YjQxNmQxM3wxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:03 上传

点击 `Locate license file...` 可以选择许可证文件，随便选个文件看看：

![](https://attach.52pojie.cn/forum/202608/05/070442oz9o7dkcq58ndogq.png)

**error-invalid-license.png** _(27.1 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU4NnxmMTVkZThhOHwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

点击 OK 后又会显示 License Required 弹窗。

在 IDA Pro 中打开 `binaryninja.exe`，打开一个 Strings 视图，搜索标题字符串 `License Required`：

![](https://attach.52pojie.cn/forum/202608/05/070428h3i46leiejr443le.png)

**License-Required-String.png** _(13.62 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU4M3wzYWI3NjJjN3wxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

![](https://attach.52pojie.cn/forum/202608/05/070434jlzurnlpknsolncl.png)

**License-Required-String-Jump.png** _(144.94 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU4NHw4ZTM0ZDY5OHwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

跳转到交叉引用该字符串的位置 `sub_140111A60+2F33`：

![](https://attach.52pojie.cn/forum/202608/05/070440vms4dctutsmtdoto.png)

**sub_140111A60 2F33.png** _(130.03 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU4NXxkNjA5YjIyOHwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

往上翻，看到底是什么决定是否走 License Required 弹窗的逻辑，这里用 Graph View 看会比较清晰。即使不分析具体逻辑，通过看字符串也能看出这几个分支是干什么的：

![](https://attach.52pojie.cn/forum/202608/05/070445sfv9gnf8sqcsu283.png)

**exe-lic-entrance.png** _(307.31 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU4N3w1MDQzZWM2YnwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

右侧还有个没截图到的分支，推测这是曾经已经选过正确的许可证时走的分支（毕竟没必要每次打开软件都提示 `Thank you for supporting Binary Ninja.`，所以肯定有一个对应这种情况的分支）。再往下就是一些初始化相关的逻辑。

![](https://attach.52pojie.cn/forum/202608/05/070447zbhmuo6bf2shxhsj.png)

**exe-lic-entrance-next.png** _(154.34 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU4OHw5Y2E5MzExZXwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

那么这些分支顶部的入口就很关键了，再看看：

![](https://attach.52pojie.cn/forum/202608/05/070450c1h7kdawarazk5r4.png)

**exe-entrance-big.png** _(39.08 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU4OXxiYWU4YWY0NnwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

可以看到 `BNIsLicenseValidated` 函数决定了之后的走向。通过刚刚的分析可以得出，该函数在检测到合法许可证时返回 1，此时就能顺利进入软件，否则返回 0，表示没有检测到合法许可证。

该函数的定义在另一个模块 `binaryninjacore`，所以 `binaryninja.exe` 可以暂时放在一边了，后续需要进一步分析 `binaryninjacore.dll` 中该函数的逻辑。

![](https://attach.52pojie.cn/forum/202608/05/070453lj2h377a2l7nclct.png)

**__imp_BNIsLicenseValidated.png** _(19.62 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5MHw2NGUxYTBlMnwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

#### 深入 binaryninjacore

深入之前我想先说明：肯定有人我一样，觉得根本没必要分析 `binaryninjacore` 的具体逻辑，直接让 `BNIsLicenseValidated` 函数返回 1，或者强制让程序走许可证正确的分支就行了。这个我也尝试过，但打开软件发现功能上有缺失：只能以 Raw 模式查看文件，PE 模式没法用，这样 Binary Ninja 就退化成 Hex 编辑器了。猜测是 `binaryninjacore` 内部的某些逻辑记录了一些数据（或者说归档），后续软件会依据这些数据决定是否启用某些功能。因此深入分析一下 `binaryninjacore` 还是有必要的。

在 IDA Pro 中打开 `binaryninjacore.dll`，在 Exports 视图中搜索 `BNIsLicenseValidated`：

![](https://attach.52pojie.cn/forum/202608/05/070455hsdisbj888bxjsb1.png)

**BNIsLicenseValidated.png** _(14.53 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5MXxiNjBlMGNmYnwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:04 上传

![](https://attach.52pojie.cn/forum/202608/05/070500iwr01bcyibob9xrr.png)

**BNIsLicenseValidated-next.png** _(31.35 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5Mnw2ZjRiMThjM3wxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:05 上传

不难看出 `byte_18AA4A089` 直接决定函数的返回值。搜一下：

![](https://attach.52pojie.cn/forum/202608/05/070503voxgiyiyss9ydd6x.png)

**byte_18AA4A089.png** _(18.21 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5M3w1ODMzMGVmYnwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:05 上传

看一下它的交叉引用：

![](https://attach.52pojie.cn/forum/202608/05/070505xzb2ehmva778fzel.png)

**byte_18AA4A089-xref.png** _(35.39 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5NHwxZDg3ZTMyMXwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:05 上传

可以看到它是在 `sub_1814A3790` 中被赋值。同时注意到 `sub_1814A3790` 内部有将 `byte_18AA4A089` 置 1 的动作，说明许可证校验的具体逻辑极有可能在该函数内部。`BNIsLicenseValidated` 仅负责返回校验结果。

此时我意识到刚刚对 `binaryninja.exe` 的分析不彻底，一定是在调用 `BNIsLicenseValidated` 前先有过对 `sub_1814A3790` 的调用才对。

看下 `sub_1814A3790` 的交叉引用：

![](https://attach.52pojie.cn/forum/202608/05/070508k8tph8dhabpxtfdb.png)

**sub_1814A3790-xref.png** _(55.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5NXw2YzQ5ZjcxN3wxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:05 上传

这几个 BN 开头的函数很可疑，都是当前模块导出的函数。猜测 `binaryninja.exe` 在早期调用过其中一个或多个，之后走到 `BNIsLicenseValidated` 时 `byte_18AA4A089` 才可能为 1。

#### 回看 binaryninja.exe

再次在 IDA 中打开 `binaryninja.exe`。之前分析的代码都位于 `sub_140111A60`。一点点往上排查，在这个函数的开头部分发现了两个 BN 函数：

![](https://attach.52pojie.cn/forum/202608/05/070510sddvdarflltxjyzd.png)

**sub_140111A60-start.png** _(65.59 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5Nnw2ODNlZDFkYnwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:05 上传

`BNInitUI` 是该函数调用的第一个 BN 函数，嫌疑很大，之后回到 `binaryninjacore` 重点看下函数。注意这里给函数传了个参数 `0x74CA8C062D314459`。

#### 继续探索 binaryninjacore

`BNInitUI` 函数内容如下：

![](https://attach.52pojie.cn/forum/202608/05/070513dvy099tkluqp2y9q.png)

**BNInitUI.png** _(86.66 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5N3wyNGJlZmI3MXwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:05 上传

`rcx` 是第一个函数参数，而 `binaryninja.exe` 传的就是 `0x74CA8C062D314459`，`rax = rcx`，因此不会触发 `jnz` 的跳转。这里还给 `byte_18AA4A08A` 赋值为 1 了，之后会用到。最后跳去了 `sub_1814A3790`。结合之前的分析，许可证校验的具体逻辑应该就在这个函数里。

#### 真正关键的函数 sub_1814A3790

这个函数分析起来挺费劲的，是本次逆向的难点。函数很长，我选择先反编译看伪代码。

##### 寻找许可证

开头先尝试读环境变量 `BN_LICENSE`，如果不存在则去找许可证文件：

```
    if ( getenv(VarName: "BN_LICENSE") != nullptr ) // getenv 读环境变量
    {
      v2 = getenv(VarName: "BN_LICENSE");
      v3 = (void (__fastcall ***)(void *, __int64))operator new(Size: 0x38u);
      __wind
      {
        v97 = v3;
        if ( v3 != nullptr )
        {
          v4 = -1;
          do
            ++v4;
          while ( v2[v4] != 0 );
          v5 = (void *)sub_180FB79E0(a1: v3, a2: v2, a3: v4);
        }
        else
        {
          v5 = nullptr;
        }
        qword_18AA4A020 = v5;
      }
      __unwind
      {
        j_BNFreeDataReferences_0_0(Block: v97);
      }
      goto LABEL_299;
    }
    v6 = (_QWORD *)sub_1814A5420(a1: v147); // 计算许可证文件路径
    // ...

```

我们肯定选择走许可证文件，没必要为了这个软件去建环境变量。看看 `sub_1814A5420` 是怎么找许可证文件的：

```
__int64 __fastcall sub_1814A5420(__int64 a1)
{
  __int64 result; // rax
  _QWORD v3[4]; // [rsp+30h] [rbp-58h] BYREF
  __int64 v4; // [rsp+50h] [rbp-38h] BYREF
  __m128i si128; // [rsp+60h] [rbp-28h]

  v3[2] = 11;
  v3[3] = 15;
  v3[0] = 0x2E65736E6563696CLL; // little endian 的字符串 license.
  v3[1] = 7627108;
  __wind
  {
    sub_180CCBF50(); // 获取许可证路径
    // 省略，看上去是在对输入做清洗防注入...
  return result;
}

```

继续跟进去看 `sub_180CCBF50`：

```
__int64 __fastcall sub_180CCBF50(__int64 a1)
{
  char *v2; // rax
  __int64 v3; // rdi
  size_t v4; // rdi
  __int64 v5; // r14
  void *v7[2]; // [rsp+40h] [rbp-C0h] BYREF
  size_t v8; // [rsp+50h] [rbp-B0h]
  unsigned __int64 v9; // [rsp+58h] [rbp-A8h]
  __int128 v10; // [rsp+60h] [rbp-A0h] BYREF
  __int64 v11; // [rsp+70h] [rbp-90h]
  unsigned __int64 v12; // [rsp+78h] [rbp-88h]
  CHAR Src[272]; // [rsp+80h] [rbp-80h] BYREF

  v2 = getenv(VarName: "BN_USER_DIRECTORY"); // 先尝试读环境变量 BN_USER_DIRECTORY 作为路径
  if ( v2 != nullptr )
  {
    *(_OWORD *)a1 = 0;
    *(_QWORD *)(a1 + 16) = 0;
    *(_QWORD *)(a1 + 24) = 0;
    v3 = -1;
    do
      ++v3;
    while ( v2[v3] != 0 );
    sub_180965640(a1, a2: v2, a3: v3);
  }
  else if ( SHGetFolderPathA(hwnd: nullptr, csidl: 26, hToken: nullptr, dwFlags: 0, pszPath: Src) >= 0 )
  { // SHGetFolderPathA 获取特殊文件夹 %AppData% 的路径
    HIDWORD(v10) = 0;
    v11 = 12;
    v12 = 15;
    strcpy((char *)&v10, "Binary Ninja"); // 拼路径，%AppData% 后面接 Binary Ninja 字符串
    *(_OWORD *)v7 = 0;
    // 最终拼出 license.dat 的完整路径，省略...
  return a1;
}

```

这部分逻辑展示了许可证文件的存储位置（位于 `%AppData%\Binary Ninja\license.dat`）。但其实不知道也没事，因为当我们通过 `Locate license file...` 选择了一个合法的许可证文件后，程序会把文件复制到指定位置。

##### 读取并解析许可证

读取可证文件内容：

```
    sub_180FDBA40(a1: v98, a2: v6, a3: 0); // 打开文件
    // ...
    if ( sub_180FDBBC0(a1: v98, a2: v165, a3: 0, a4: v8) != v8 ) // 读文件内容
    {
      if ( __eh34_unwind(5) )
        goto unwind_state_5;
      __eh34_exit_wind_state(5, 4);
      sub_180FB7AF0(a1: v164);
      goto LABEL_21;
    }

```

```
_QWORD *__fastcall sub_180FDBA40(_QWORD *a1, const char *a2, char a3)
{
  const char *v7; // rdx

  *a1 = &BinaryNinjaCore::DefaultFileAccessor::`vftable';
  if ( a3 != 0 )
  {
    if ( (unsigned __int8)BNIsPathDirectory(a1: a2) != 0 )
    {
LABEL_3:
      a1[1] = 0;
      return a1;
    }
  }
  else if ( (unsigned __int8)BNIsPathRegularFile(a1: a2) == 0 )
  {
    goto LABEL_3;
  }
  v7 = "rb";
  if ( a3 != 0 )
    v7 = "wb";
  a1[1] = fopen(FileName: a2, Mode: v7); // fopen 打开文件，拿到文件描述符
  return a1;
}

```

```
size_t __fastcall sub_180FDBBC0(__int64 a1, void *a2, __int64 a3, size_t a4)
{
  FILE *v6; // rcx

  v6 = *(FILE **)(a1 + 8); // 使用刚刚拿到的文件句柄
  if ( v6 == nullptr )
    return 0;
  _fseeki64(Stream: v6, Offset: a3, Origin: 0);
  return fread(Buffer: a2, ElementSize: 1u, ElementCount: a4, Stream: *(FILE **)(a1 + 8));
}

```

JSON 解析：

```
   v19 = sub_1809D0380(a1: v112); // 解析许可证文件内容
  __wind
  {
    v46 = (void *)(*(__int64 (__fastcall **)(__int64))(*(_QWORD *)v19 + 8LL))(a1: v19);
    v101 = v46;
    sub_1809D2210(a1: v112);
  }
  __unwind
  {
    sub_1809D2210(a1: v112);
  }

```

```
_QWORD *__fastcall sub_1809D0380(_QWORD *a1)
{
  _QWORD *v2; // rcx
  __int64 v3; // rdx
  __int64 v4; // r8
  __int64 v5; // r9
  _QWORD *result; // rax

  *a1 = &Json::CharReaderBuilder::`vftable'; // 构造一个 Json::CharReaderBuilder 对象，完成虚表绑定等
  v2 = a1 + 1;
  *((_BYTE *)v2 + 8) = 0;
  *((_DWORD *)v2 + 2) &= ~0x100u;
  v2[2] = 0;
  v2[3] = 0;
  v2[4] = 0;
  __wind
  {
    __wind
    {
      sub_1809E8AD0();
      result = a1;
    }
    __unwind
    {
      sub_1809D2660(a1: a1 + 1);
    }
  }
  __unwind
  {
    ((void (__fastcall *)(_QWORD *, __int64, __int64, __int64))sub_1809D2330)(a1, a2: v3, a3: v4, a4: v5);
  }
  return result;
}

```

遍历 JSON 对象：

```
  while ( (unsigned __int8)sub_1809DF860(a1: v102, a2: v106) == 0 )
  {
    v20 = sub_1809DC730(a1: v102);
    v21 = sub_1809D3440(a1: v20, a2: "product");
    sub_1809D8D30(a1: v21, a2: Buf1);
    __eh34_enter_wind_state(13, 14);
    v22 = sub_1809D3440(a1: v20, a2: "email");
    sub_1809D8D30(a1: v22, a2: v121);
    __eh34_enter_wind_state(14, 15);
    v23 = sub_1809D3440(a1: v20, a2: "serial");
    sub_1809D8D30(a1: v23, a2: Src);
    __eh34_enter_wind_state(15, 16);
    v24 = sub_1809D3440(a1: v20, a2: "created");
    sub_1809D8D30(a1: v24, a2: v130);
    __eh34_enter_wind_state(16, 17);
    v25 = sub_1809D3440(a1: v20, a2: "type");
    sub_1809D8D30(a1: v25, a2: v124);
    __eh34_enter_wind_state(17, 18);
    v26 = sub_1809D3440(a1: v20, a2: "count");
    v27 = sub_1809D8840(a1: v26);
    v28 = sub_1809D3440(a1: v20, a2: "data");
    sub_1809D8D30(a1: v28, a2: v127);
    __eh34_enter_wind_state(18, 19);
    sub_180FB8120(a1: v167, a2: v127);
    __eh34_enter_wind_state(19, 20);
    v29 = sub_1809D3440(a1: v20, a2: "signature");
    v30 = sub_1809D8D30(a1: v29, a2: v162);
    // ...

```

从以上这些代码不难看出许可证文件的内容是一段 JSON，有 `product`，`email` 等字段。比较难看出来的是许可证是 JSON 数组格式的（由 `[]` 包裹），而不是单个对象。关于这点从上面这段循环中可以略窥一二，如果仅支持单个 JSON 对象就没必要循环了。许可证示例（由文章开头提到的破解补丁生成）：

```
[
  {
    "count": 10000,
    "created": "2026-08-04T20:58:43.903+00:00",
    "data": "/2/Mi5JTF6NypoNJ28rNmtFodKeou+n2esQhTOMprbGxyYWWYGwxC/KI4XpKyFKOdXxDe5Cmwa6t8XjGbiv0ISKmhiYvYNJgPhKLrv9HO/RCl4FLWKf5Qz94KfQW6iM8ukgqAGVnae5eg2ZC3IdbQSAFoJK5+M3z3nXApu8lPMmxicx2WlrJ5wO57xTVqf+5lb8G6cCWRCpk7RbbGKxjLvNOot2ZPe3pKSlTBFsYkny6cMA6kiYgDtxA5/YXC59obLCrG1kTOtSoNcZWIbR6dq3H3a2N+qQUg52US9qP27v+mCH0uiwapWgfWxwdziFKngWR1WfzTwiJiTthhe7PsSjtUh3pntF4f3oVkaVe4iMDEgFsbStO7Q==",
    "email": "binaryninja@vector35.com",
    "product": "Binary Ninja Personal",
    "serial": "ea3855525cd46551ed10e401739a2ec6",
    "signature": "LYVpVUY3BTllXMxtYLDshpntvK+onc7CSw3NEFZydyE=",
    "type": "User"
  }
]

```

##### 许可证过期时间解析

后续有一段关于 `expiresEpoch` 字段的逻辑：

```
    if ( (unsigned __int8)sub_1809DFA50(a1: v20, a2: "expiresEpoch") != 0 ) // sub_1809DFA50 内部和 sub_1809D3440 有点类似，但功能是判断字段是否存在
    {
      v31 = sub_1809D3440(a1: v20, a2: "expiresEpoch"); // 读字段
      v32 = sub_1809D8EB0(a1: v31); // 解析字段值
    }
    else
    {
      v32 = 0;
    }

```

```
__int64 __fastcall sub_1809D3440(__int64 a1, __int64 a2, __int64 a3, __int64 a4, int a5)
{
  __int64 v5; // r8

  v5 = -1;
  do
    ++v5;
  while ( *(_BYTE *)(a2 + v5) != 0 );
  return sub_1809E8050(a1, a2, a3: a2 + v5, a4, a5);
}

__int64 *__fastcall sub_1809E8050(__int64 a1, void *a2, int a3)
{
  __int64 ***v5; // r14
  char v6; // al
  _QWORD *v7; // rax
  _QWORD *v8; // rbx
  _QWORD *v9; // rax
  unsigned int v10; // esi
  __int64 *v11; // r13
  __int64 *v12; // rbx
  __int64 *v13; // r12
  const void *v14; // rcx
  unsigned int v15; // edi
  bool v16; // cf
  unsigned int v17; // edi
  unsigned int v18; // eax
  int v19; // eax
  __int64 v21; // rbx
  int v22; // edi
  unsigned __int8 v23; // al
  __int64 v24; // r14
  char *v25; // rsi
  __int64 v26; // rax
  void *v28; // [rsp+28h] [rbp-D8h] BYREF
  void *Block; // [rsp+30h] [rbp-D0h] BYREF
  int v30; // [rsp+38h] [rbp-C8h]
  _QWORD v31[2]; // [rsp+40h] [rbp-C0h] BYREF
  void *v32; // [rsp+50h] [rbp-B0h]
  void *v33; // [rsp+68h] [rbp-98h] BYREF
  unsigned int v34; // [rsp+70h] [rbp-90h]
  _BYTE v35[240]; // [rsp+80h] [rbp-80h] BYREF
  _QWORD *v36; // [rsp+170h] [rbp+70h] BYREF
  int v37; // [rsp+178h] [rbp+78h]
  __int128 v38; // [rsp+180h] [rbp+80h]
  __int64 v39; // [rsp+190h] [rbp+90h]

  v5 = (__int64 ***)a1;
  v6 = *(_BYTE *)(a1 + 8);
  if ( v6 != 0 )
  {
    if ( v6 != 7 )
    {
      sub_1809CFAD0(a1: v35, a2: 1);
      __wind
      {
        sub_1809C9E70(a1: v35, a2: "in Json::Value::resolveReference(key, end): requires objectValue");
        v26 = sub_1809E9250(a1: v35, a2: &v36);
        // 从报错信息可以看出这是个读字段的函数，后面省略...

```

```
__int64 __fastcall sub_1809D8EB0(char *a1)
{
  int v2; // ecx
  int v3; // ecx
  int v4; // ecx
  int v5; // ecx
  double v7; // xmm1_8
  unsigned __int64 v8; // rcx
  __int64 v9; // rax
  __int64 v10; // rax
  __int64 v11; // rax
  _BYTE v12[240]; // [rsp+30h] [rbp-128h] BYREF
  _BYTE v13[32]; // [rsp+120h] [rbp-38h] BYREF

  v2 = a1[8];
  if ( v2 == 0 )
    return 0;
  v3 = v2 - 1;
  if ( v3 != 0 )
  {
    v4 = v3 - 1;
    if ( v4 != 0 )
    {
      v5 = v4 - 1;
      if ( v5 != 0 )
      {
        if ( v5 != 2 )
        {
          sub_1809CFAD0(a1: v12, a2: 1);
          __eh34_enter_wind_state(-1, 0);
          sub_1809C9E70(a1: v12, a2: "Value is not convertible to UInt64.");
          // 同样看报错信息，这是个解析字段值的函数，后面省略...

```

如果 `expiresEpoch` 存在就解析，没有就给个默认值 0。猜测 0 其实就意味着永远不过期，所以构造许可证时可以直接省略这个字段。

##### data 字段长度校验

data 字段校验：

```
    if ( v167[0] != 280 ) // v167[0] 是数据长度
    {
      sub_181043690(a1: v108, a2: &byte_186D50093);
      throw (std::runtime_error *)v108;
    }

```

回顾一下之前关于 data 字段的部分：

```
    v28 = sub_1809D3440(a1: v20, a2: "data");
    sub_1809D8D30(a1: v28, a2: v127); // 读取为 string
    __eh34_enter_wind_state(18, 19);
    sub_180FB8120(a1: v167, a2: v127); // Base64 解码为二进制数组

```

```
_QWORD *__fastcall sub_1809D8D30(_QWORD *a1, _QWORD *a2)
{
  _DWORD *v4; // rdx
  __int64 v5; // rax
  const char *v6; // rdx
  __int64 v7; // rax
  _BYTE v8[240]; // [rsp+40h] [rbp-128h] BYREF
  _QWORD v9[4]; // [rsp+130h] [rbp-38h] BYREF

  switch ( *((_BYTE *)a1 + 8) )
  {
    case 0:
      goto LABEL_2;
    case 1:
      sub_1809EAFF0(a1: a2, a2: *a1, a3: 0);
      return a2;
    case 2:
      sub_1809EB140(a1: a2, a2: *a1, a3: 0);
      return a2;
    case 3:
      sub_1809EABF0(a1: (_DWORD)a2, a2: (unsigned int)&_ImageBase, a3: 0, a4: 17, a5: 0);
      return a2;
    case 4:
      v4 = (_DWORD *)*a1;
      if ( *a1 != 0 )
      {
        if ( (a1[1] & 0x100) != 0 )
        {
          LODWORD(v5) = *v4++;
        }
        else
        {
          v5 = -1;
          do
            ++v5;
          while ( *((_BYTE *)v4 + v5) != 0 );
        }
        *(_OWORD *)a2 = 0;
        a2[2] = 0;
        a2[3] = 0;
        sub_180965640(a1: a2, a2: v4, a3: (unsigned int)v5);
      }
      else
      {
LABEL_2:
        *(_OWORD *)a2 = 0;
        a2[2] = 0;
        a2[3] = 15;
        *(_BYTE *)a2 = 0;
      }
      break;
    case 5:
      v6 = "false";
      if ( *(_BYTE *)a1 != 0 )
        v6 = "true";
      sub_18096C170(a1: a2, a2: v6);
      break;
    default:
      sub_1809CFAD0(a1: v8, a2: 1);
      __wind
      { // 依旧看报错信息，说明这是个读字段值的函数
        sub_1809C9E70(a1: v8, a2: "Type is not convertible to string");
        v7 = sub_1809E9250(a1: v8, a2: v9);
        __wind
        {
          sub_1809E9760(a1: v7);
        }
        __unwind
        {
          sub_18096DD70(a1: v9);
        }
      }
      __unwind
      {
        sub_1809D3F40(a1: v8);
      }
  }
  return a2;
}

```

Base64 解码分析：

```
unsigned __int64 *__fastcall sub_180FB8120(unsigned __int64 *a1, char *a2)
{ // a2 是 Base64 字符串，a1 指向一个用于存放解码结果的动态缓冲区结构
  __int128 *v2; // r12
  unsigned __int64 v3; // rax
  char *v5; // r8
  _QWORD *v6; // r9
  char *v7; // r14
  char *v8; // r13
  unsigned __int64 v9; // rdi
  char v10; // dl
  unsigned __int64 v11; // r15
  unsigned __int64 v12; // rcx
  unsigned __int64 v13; // rsi
  __int16 v14; // kr00_2
  char v15; // bp
  unsigned __int64 i; // rax
  _OWORD *v17; // rax
  __int128 v18; // xmm0
  unsigned __int64 v19; // rcx
  unsigned __int64 v20; // r14
  char v21; // bp
  unsigned __int64 v22; // rax
  unsigned __int64 v23; // rsi
  _OWORD *v24; // rax
  __int128 v25; // xmm0
  unsigned __int64 v26; // rax
  unsigned __int64 v27; // rax
  unsigned __int64 v28; // r14
  unsigned __int64 v29; // rsi
  unsigned __int64 v30; // rax
  _OWORD *v31; // rax
  __int128 v32; // xmm0
  unsigned __int64 v33; // rcx
  __int16 v35; // [rsp+70h] [rbp+8h]
  __int16 v36; // [rsp+70h] [rbp+8h]
  char v37; // [rsp+78h] [rbp+10h]
  unsigned __int8 v38; // [rsp+79h] [rbp+11h]
  unsigned __int8 v39; // [rsp+7Ah] [rbp+12h]
  char v40; // [rsp+7Bh] [rbp+13h]

  v2 = (__int128 *)(a1 + 3);
  a1[1] = 32; // 总容量
  v3 = 0;
  *a1 = 0; // 当前已写入大小
  *(_OWORD *)(a1 + 3) = 0; // 缓冲区，先创建了 16 字节
  v5 = a2;
  *(_OWORD *)(a1 + 5) = 0; // 又创建了 16 字节，合计 32 字节
  a1[2] = (unsigned __int64)(a1 + 3); // 数据指针（初始指向结构体内联的 32 字节小缓冲区，容量不足时会自动扩容并切换到堆内存）
  if ( *((_QWORD *)a2 + 3) < 0x10u )
  {
    v6 = a2;
  }
  else
  {
    v5 = *(char **)a2;
    v6 = *(_QWORD **)a2;
  }
  v7 = v5;
  v8 = (char *)v6 + *((_QWORD *)a2 + 2);
  if ( v5 == v8 )
    return a1;
  v9 = 64;
  do
  {
    v10 = *v7; // 下面这段有明显的 Base64 解码算法特征
    if ( (unsigned __int8)(*v7 - 65) <= 0x19u ) //  'A'-'Z' 映射到 0-25
    {
      *(&v37 + v3) = v10 - 65;
LABEL_17:
      ++v3;
      goto LABEL_18;
    }
    if ( (unsigned __int8)(v10 - 97) <= 0x19u ) // 'a'-'z' => 26-51
    {
      *(&v37 + v3) = v10 - 71; // 'a' - 71 = 97-71=26, 所以 'a' 映射到 26
      goto LABEL_17;
    }
    if ( (unsigned __int8)(v10 - 48) <= 9u ) // '0'-'9' => 52-61
    {
      *(&v37 + v3) = v10 + 4; // '0'+4 = 52
      goto LABEL_17;
    }
    if ( ((v10 - 43) & 0xFD) == 0 ) // 43='+', 45='-', & 0xFD 过滤 bit 1, 所以是 '+' 或 '-'
    {   // 猜测是为了兼容 Base64url 变体（使用 '-' 替代 '+'）
      *(&v37 + v3) = 62; // 映射为 62
      goto LABEL_17;
    }
    if ( v10 == 47 || v10 == 95 ) // '/' 和 '_'
    { // 应该还是兼容 Base64url 变体（使用 '_' 替代 '/'）
      *(&v37 + v3) = 63; // 映射为 63
      goto LABEL_17;
    }
LABEL_18:
    if ( v3 >= 4 )
    {
      v11 = *a1;
      LOBYTE(v35) = (v38 >> 4) | (4 * v37);
      v12 = a1[1];
      v13 = *a1 + 3;
      v14 = v39 << 6;
      v15 = v40 | v14;
      HIBYTE(v35) = (16 * v38) | HIBYTE(v14);
      if ( v13 > v12 )
      {
        for ( i = 64; v13 > i; i *= 2LL )
        {
          if ( v12 == 0 )
            break;
        }
        if ( i == 0 )
          i = v11 + 3;
        a1[1] = i;
        if ( v12 == 32 )
        {
          v17 = sub_181F82810(a1: i, a2: 1);
          v18 = *v2;
          a1[2] = (unsigned __int64)v17;
          *v17 = v18;
          v17[1] = v2[1];
        }
        else
        {
          a1[2] = sub_181F866E0(a1: a1[2], a2: i);
        }
      }
      v19 = a1[2];
      *a1 = v13;
      *(_WORD *)(v19 + v11) = v35;
      v3 = 0;
      *(_BYTE *)(v19 + v11 + 2) = v15;
    }
    ++v7;
  }
  while ( v7 != v8 );
  if ( v3 == 2 )
  {
    v20 = *a1;
    v21 = (v38 >> 4) | (4 * v37);
    v22 = a1[1];
    v23 = *a1 + 1;
    if ( v23 > v22 )
    {
      if ( v23 > 0x40 )
      {
        do
        {
          if ( v22 == 0 )
            break;
          v9 *= 2LL;
        }
        while ( v23 > v9 );
      }
      if ( v9 == 0 )
        v9 = *a1 + 1;
      a1[1] = v9;
      if ( v22 == 32 )
      {
        v24 = sub_181F82810(a1: v9, a2: 1);
        v25 = *v2;
        a1[2] = (unsigned __int64)v24;
        *v24 = v25;
        v24[1] = v2[1];
        v26 = a1[2];
        *a1 = v23;
        *(_BYTE *)(v20 + v26) = v21;
        return a1;
      }
      a1[2] = sub_181F866E0(a1: a1[2], a2: v9);
    }
    v27 = a1[2];
    *a1 = v23;
    *(_BYTE *)(v20 + v27) = v21;
    return a1;
  }
  if ( v3 == 3 )
  {
    v28 = *a1;
    v29 = *a1 + 2;
    LOBYTE(v36) = (v38 >> 4) | (4 * v37);
    HIBYTE(v36) = (16 * v38) | (v39 >> 2);
    v30 = a1[1];
    if ( v29 > v30 )
    {
      if ( v29 > 0x40 )
      {
        do
        {
          if ( v30 == 0 )
            break;
          v9 *= 2LL;
        }
        while ( v29 > v9 );
      }
      if ( v9 == 0 )
        v9 = *a1 + 2;
      a1[1] = v9;
      if ( v30 == 32 )
      {
        v31 = sub_181F82810(a1: v9, a2: 1);
        v32 = *v2;
        a1[2] = (unsigned __int64)v31;
        *v31 = v32;
        v31[1] = v2[1];
      }
      else
      {
        a1[2] = sub_181F866E0(a1: a1[2], a2: v9);
      }
    }
    v33 = a1[2];
    *a1 = v29;
    *(_WORD *)(v28 + v33) = v36;
  }
  return a1;
}

```

这一段的作用是把 Base64 字符串 a2 解码，并将结果存在 a1（一个类似于 `std::vector<unsigned char>` 的动态数组）。

再回看 data 字段校验的部分，如果 `v167[0] != 280` ，即 data 解码后的字节数不等于 280，就会抛异常。因此我们构造 data 字段时要用 280 字节的数据进行 Base64 编码。

##### 许可证验签

接下来这段是重头戏，许可证签名校验：

```
v43 = operator new(Size: 0x128u);
    *v43 = 2137282818;
    v43[1] = 1415828994;
    v43[2] = -617784040;
    v43[3] = 1550112453;
    v43[4] = 1583734323;
    v43[5] = 1567283888;
    v43[6] = 1466194178;
    v43[7] = 1550080304;
    v43[8] = -1022816974;
    v43[9] = -1360852998;
    v43[10] = -438848535;
    v43[11] = 207666573;
    v43[12] = -1455235503;
    v43[13] = 1719462622;
    v43[14] = -1134842021;
    v43[15] = -322008108;
    v43[16] = 1007090832;
    v43[17] = -1343825790;
    v43[18] = 1460518159;
    v43[19] = -438909606;
    v43[20] = -870673938;
    v43[21] = -1420346269;
    v43[22] = 1041001702;
    v43[23] = -766522909;
    v43[24] = 907428458;
    v43[25] = 1106551020;
    v43[26] = -1187974739;
    v43[27] = 1004564767;
    v43[28] = -484018439;
    v43[29] = 1699397722;
    v43[30] = 531010087;
    v43[31] = -1466645810;
    v43[32] = 153160097;
    v43[33] = -260913594;
    v43[34] = 1855191744;
    v43[35] = -1255639145;
    v43[36] = 1801923948;
    v43[37] = 1805394714;
    v43[38] = 1141815998;
    v43[39] = -1721299845;
    v43[40] = 1076135467;
    v43[41] = 1179304423;
    v43[42] = 2145813174;
    v43[43] = -1304700757;
    v43[44] = -1099209382;
    v43[45] = -2101262001;
    v43[46] = -818064313;
    v43[47] = 1754493434;
    v43[48] = -255664746;
    v43[49] = 1591362941;
    v43[50] = -2027825703;
    v43[51] = -1377927046;
    v43[52] = 1688069802;
    v43[53] = -1250977518;
    v43[54] = 1639919382;
    v43[55] = -938734853;
    v43[56] = 742045703;
    v43[57] = -1893963421;
    v43[58] = -2252989;
    v43[59] = -1076642136;
    v43[60] = -1597370178;
    v43[61] = -2091224855;
    v43[62] = -1131081546;
    v43[63] = -162687539;
    v43[64] = -1810120105;
    v43[65] = -1877419569;
    v43[66] = -582144440;
    v43[67] = 676843405;
    v43[68] = -547163062;
    v43[69] = 442425329;
    v43[70] = 1641275184;
    v43[71] = -232904846;
    v43[72] = 1550244097;
    v43[73] = 1566956082; // 74 字节的加密公钥数据
    *(_OWORD *)v88 = 0;
    v89 = nullptr;
    __eh34_enter_wind_state(24, 25);
    for ( k = 0; k < 0x126; ++k )
    {
      v45 = (v43[k >> 2] ^ 0x5D65DB32u) >> (8 * (k & 3)); // 逐个异或 0x5D65DB32u 解出 296 字节公钥
      v86[0] = v45;
      if ( v88[1] == v89 )
        sub_1814A3340(a1: v88, a2: v88[1], a3: v86);
      else
        *(_BYTE *)v88[1]++ = v45;
      v46 = v101;
    }
    BNFreeDataReferences_0(Block: v43);
    v47 = (void (__fastcall ***)(_QWORD, __int64))sub_181D151A0(a1: v88);
    v105 = v47;
    __eh34_enter_wind_state(25, 26);
    *(_OWORD *)v150 = 0;
    v151 = 0;
    v152 = 15;
    LOBYTE(v150[0]) = 0;
    v134 = 14;
    v135 = 15;
    strcpy((char *)Block, "EMSA3(SHA-256)");
    HIBYTE(Block[1]) = 0;
    __wind
    {
      __wind
      {
        sub_181C4F740(a1: (unsigned int)v103, a2: (_DWORD)v47, a3: (unsigned int)Block, a4: 0, a5: (__int64)v150);
        if ( v135 >= 0x10 )
        {
          v48 = Block[0];
          if ( v135 + 1 >= 0x1000 )
          {
            v48 = *((void **)Block[0] - 1);
            if ( (unsigned __int64)((char *)Block[0] - (char *)v48 - 8) > 0x1F )
              _invalid_parameter_noinfo_noreturn();
          }
          j_BNFreeDataReferences_0_0(Block: v48);
        }
        v134 = 0;
        v135 = 15;
        LOBYTE(Block[0]) = 0;
        if ( v152 >= 0x10 )
        {
          v49 = v150[0];
          if ( v152 + 1 >= 0x1000 )
          {
            v49 = *((void **)v150[0] - 1);
            if ( (unsigned __int64)((char *)v150[0] - (char *)v49 - 8) > 0x1F )
              _invalid_parameter_noinfo_noreturn();
          }
          j_BNFreeDataReferences_0_0(Block: v49);
        }
      }
      __unwind
      {
        sub_18096DDB0(a1: Block);
      }
    }
    __unwind
    {
      sub_18096DDB0(a1: v150);
    }
    __eh34_enter_wind_state(26, 31);
    v151 = 0;
    v152 = 15;
    LOBYTE(v150[0]) = 0;
    sub_181D544D0(a1: v103, a2: v149[2], a3: v149[0]);
    if ( (unsigned __int8)sub_181CBEBD0(a1: v103, a2: v166[2], a3: v166[0]) == 0 ) // 签名校验：PK_Verifier::check_signature RSA+EMSA3(SHA-256)
    {
      sub_181043690(a1: v109, a2: &byte_186D50093);
      throw (std::runtime_error *)v109;
    }

```

看看签名校验是怎么做的：

```
char __fastcall sub_181CBEBD0(__int64 **a1, __int64 a2, __int64 a3)
{
  int v6; // eax
  char result; // al
  void *v8; // rax
  __int64 v9; // rax
  __int64 *v10; // rsi
  __int64 *v11; // r8
  _BYTE *v12; // rdi
  signed __int64 v13; // rdx
  __int64 v14; // r14
  __int64 *v15; // rcx
  __int64 v16; // rax
  char *v17; // rbx
  char v18; // si
  void *v19; // rcx
  void *v20; // rdx
  void *v21; // rdx
  char *v22; // rax
  _QWORD *v23; // rax
  void *v24; // rdx
  void *v25; // rdx
  char v26; // [rsp+30h] [rbp-1D8h]
  void *v27[2]; // [rsp+38h] [rbp-1D0h] BYREF
  __int64 v28; // [rsp+48h] [rbp-1C0h]
  void *v29[2]; // [rsp+50h] [rbp-1B8h] BYREF
  __int64 v30; // [rsp+60h] [rbp-1A8h]
  __int64 v31; // [rsp+68h] [rbp-1A0h] BYREF
  __int64 v32; // [rsp+70h] [rbp-198h] BYREF
  void *v33[2]; // [rsp+78h] [rbp-190h]
  __int64 v34; // [rsp+88h] [rbp-180h]
  __int64 v35; // [rsp+90h] [rbp-178h]
  __int64 v36; // [rsp+98h] [rbp-170h] BYREF
  void *v37[2]; // [rsp+A0h] [rbp-168h] BYREF
  __int64 v38; // [rsp+B0h] [rbp-158h]
  __int64 v39; // [rsp+B8h] [rbp-150h]
  int v40; // [rsp+C0h] [rbp-148h]
  char v41[8]; // [rsp+C8h] [rbp-140h] BYREF
  int v42; // [rsp+D0h] [rbp-138h]
  void *Block; // [rsp+D8h] [rbp-130h]
  __int128 v44; // [rsp+E0h] [rbp-128h]
  __int64 v45; // [rsp+F0h] [rbp-118h]
  __int64 v46; // [rsp+F8h] [rbp-110h]
  void *v47; // [rsp+100h] [rbp-108h]
  void *v48[2]; // [rsp+108h] [rbp-100h] BYREF
  __int64 v49; // [rsp+118h] [rbp-F0h]
  _BYTE pExceptionObject[56]; // [rsp+128h] [rbp-E0h] BYREF
  _BYTE v51[56]; // [rsp+160h] [rbp-A8h] BYREF
  _BYTE v52[56]; // [rsp+198h] [rbp-70h] BYREF

  try
  {
    v6 = *((_DWORD *)a1 + 2);
    if ( v6 != 0 )
    {
      if ( v6 != 1 )
      {
        sub_18096C480(a1: v48, a2: "PK_Verifier: Invalid signature format enum");
        __wind
        {
          sub_181C4A710(a1: v52, a2: v48);
          throw (Botan::Internal_Error *)v52;
        }
        __unwind
        {
          sub_18096DDB0(a1: v48);
        }
      }
      __wind
      {
        __wind
        {
          __wind
          {
            *(_OWORD *)v27 = 0;
            v28 = 0;
            v31 = 0;
            v32 = 65280;
            *(_OWORD *)v33 = 0;
            v34 = 0;
            v36 = 0;
            v8 = operator new(Size: 0x28u);
            __wind
            {
              v47 = v8;
              if ( v8 != nullptr )
                v9 = sub_181C46150(a1: v8, a2, a3);
              else
                v9 = 0;
              v36 = v9;
            }
            __unwind
            {
              j_BNFreeDataReferences_0_0(Block: v47);
            }
          }
          __unwind
          {
            sub_181C5B920(a1: &v36);
          }
        }
        __unwind
        {
          sub_181C5CC40(a1: &v32);
        }
        __wind
        {
          v35 = v9;
          sub_181D4B4F0(a1: &v31, a2: v41, a3: 16);
          __wind
          { // 使用了 Botan 库：https://botan.randombit.net/doxygen/classBotan_1_1PK__Verifier.html
            if ( a1[2] == nullptr || a1[3] == nullptr )
              sub_181C85280(
                a1: (unsigned int)"m_parts != 0 && m_part_size != 0",
                a2: (unsigned int)&byte_186D50093,
                a3: (unsigned int)"check_signature",
                a4: (unsigned int)"C:\\jenkins\\workspace\\ja-stable-multibranch_stable_5.3\\build\\botan\\botan\\botan_all.cpp",
                a5: 31243);
            v10 = nullptr;
            while ( (*(unsigned __int8 (__fastcall **)(__int64))(*(_QWORD *)v45 + 24LL))(a1: v45) == 0 || v42 != 65280 )
            {
              __wind
              {
                *(_OWORD *)v37 = 0;
                v38 = 0;
                v39 = -1;
                v40 = 1;
                sub_181CDA400(a1: v41, a2: v37, a3: 2);
                v23 = (_QWORD *)sub_181CECD30(a1: v48, a2: v37, a3: a1[3]);
                __wind
                {
                  sub_181C24A60(a1: v27, a2: v27[1], a3: *v23, a4: v23[1] - *v23);
                }
                __unwind
                {
                  sub_180A382D0(a1: v48);
                }
                v24 = v48[0];
                if ( v48[0] != nullptr )
                {
                  memset(v48[0], 0, v49 - (unsigned __int64)v48[0]);
                  free(Block: v24);
                  v48[0] = nullptr;
                  v48[1] = nullptr;
                  v49 = 0;
                }
                v10 = (__int64 *)((char *)v10 + 1);
                v25 = v37[0];
                if ( v37[0] != nullptr )
                {
                  memset(v37[0], 0, (v38 - (unsigned __int64)v37[0]) & 0xFFFFFFFFFFFFFFF8uLL);
                  free(Block: v25);
                }
              }
              __unwind
              {
                sub_181C5CCA0(a1: v37);
              }
            }
            v11 = a1[2];
            if ( v10 != v11 )
            {
              sub_18096C480(a1: v48, a2: "PK_Verifier: signature size invalid");
              __wind
              {
                sub_181C463F0(a1: pExceptionObject, a2: v48);
                throw (Botan::Decoding_Error *)pExceptionObject;
              }
              __unwind
              {
                sub_18096DDB0(a1: v48);
              }
            }
            sub_181CE8480(a1: v29, a2: v27, a3: v11, a4: a1[3]);
            __wind
            {
              v12 = v29[0];
              v13 = (char *)v29[1] - (char *)v29[0];
              if ( (char *)v29[1] - (char *)v29[0] != a3 )
                goto LABEL_47;
              while ( 1 )
              {
                v26 = 0;
                if ( v13 != 0 )
                {
                  v14 = a2 - (_QWORD)v12;
                  do
                  {
                    v26 |= *v12 ^ v12[v14];
                    ++v12;
                    --v13;
                  }
                  while ( v13 != 0 );
                }
                if ( v26 == 0 )
                  break;
LABEL_47:
                sub_18096C480(a1: v48, a2: "PK_Verifier: signature is not the canonical DER encoding");
                __wind
                {
                  sub_181C463F0(a1: v51, a2: v48);
                  throw (Botan::Decoding_Error *)v51;
                }
                __unwind
                {
                  sub_18096DDB0(a1: v48);
                }
              }
              v15 = *a1;
              v16 = **a1;
              v17 = (char *)v27[0];
              v18 = (*(__int64 (__fastcall **)(__int64 *, void *, signed __int64))(v16 + 8))(
                      a1: v15,
                      a2: v27[0],
                      a3: (char *)v27[1] - (char *)v27[0]);
            }
            __unwind
            {
              sub_180A45B60(a1: v29);
            }
            v19 = v29[0];
            if ( v29[0] != nullptr )
            {
              if ( v30 - (unsigned __int64)v29[0] >= 0x1000 )
              {
                v19 = *((void **)v29[0] - 1);
                if ( (unsigned __int64)((char *)v29[0] - (char *)v19 - 8) > 0x1F )
                  _invalid_parameter_noinfo_noreturn();
              }
              j_BNFreeDataReferences_0_0(Block: v19);
              *(_OWORD *)v29 = 0;
              v30 = 0;
            }
            if ( v46 != 0 )
              (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v46 + 48LL))(a1: v46, a2: 1);
            v20 = Block;
            if ( Block != nullptr )
            {
              memset(Block, 0, *((_QWORD *)&v44 + 1) - (_QWORD)Block);
              free(Block: v20);
              Block = nullptr;
              v44 = 0;
            }
            if ( v36 != 0 )
              (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v36 + 48LL))(a1: v36, a2: 1);
            v21 = v33[0];
            if ( v33[0] != nullptr )
            {
              memset(v33[0], 0, v34 - (unsigned __int64)v33[0]);
              free(Block: v21);
              *(_OWORD *)v33 = 0;
              v34 = 0;
            }
            if ( v17 != nullptr )
            {
              v22 = v17;
              if ( (unsigned __int64)(v28 - (_QWORD)v17) >= 0x1000 )
              {
                v17 = *((char **)v17 - 1);
                if ( (unsigned __int64)(v22 - v17 - 8) > 0x1F )
                  _invalid_parameter_noinfo_noreturn();
              }
              j_BNFreeDataReferences_0_0(Block: v17);
            }
            result = v18;
          }
          __unwind
          {
            sub_181C5CBD0(a1: v41);
          }
        }
        __unwind
        {
          sub_181C5CBD0(a1: &v31);
        }
      }
      __unwind
      {
        sub_180A45B60(a1: v27);
      }
    }
    else
    {
      result = (*(__int64 (__fastcall **)(__int64 *))(**a1 + 8))(a1: *a1);
    }
  }
  catch ( Botan::Invalid_Argument )
  {
    return 0;
  }
  return result;
}

```

这是唯一一段不能通过构造许可证来通过的校验，因为我们不可能拿到 Binary Ninja 的私钥来签名。绕过方法很简单，直接 patch 一下跳转指令。找到签名校验的分支处：

```
    if ( (unsigned __int8)sub_181CBEBD0(a1: v103, a2: v166[2], a3: v166[0]) == 0 ) // 签名校验：PK_Verifier::check_signature RSA+EMSA3(SHA-256)
    {
      sub_181043690(a1: v109, a2: &byte_186D50093);
      throw (std::runtime_error *)v109;
    }

```

对应的汇编：

![](https://attach.52pojie.cn/forum/202608/05/070515byd9bb6jy665g56g.png)

**sign-bypass.png** _(30.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg2OTU5OHxlNTc4MTViYXwxNzg1OTcxMTQwfDB8MjEyMTMyNg%3D%3D&nothumb=yes)

2026-8-5 07:05 上传

只需要把 `jz loc_1814A539A` 修改为 `jnz loc_1814A539A`，即可实现在验签失败的情况下，走成功的路径。这个单字节 patch 用代码实现很容易，打开 `binaryninjacore.dll`，在文件偏移 `0x14A386D` 处写入 1 字节 `0x85`（原来是 `0x84`），即可把 `je` 反转成 `jne`。

##### data 归档

data 字段归档：

```
{
      *(_OWORD *)v139 = 0;
      v140 = 3;
      v141 = 15;
      LODWORD(v139[0]) = 3490893; // 小端序的字符串 MD5
      __wind
      {
        sub_181CD0730();
        if ( v141 >= 0x10 )
        {
          v50 = v139[0];
          if ( v141 + 1 >= 0x1000 )
          {
            v50 = *((void **)v139[0] - 1);
            if ( (unsigned __int64)((char *)v139[0] - (char *)v50 - 8) > 0x1F )
              _invalid_parameter_noinfo_noreturn();
          }
          j_BNFreeDataReferences_0_0(Block: v50);
        }
        v140 = 0;
        v141 = 15;
        LOBYTE(v139[0]) = 0;
        if ( v155 >= 0x10 )
        {
          v51 = v153[0];
          if ( v155 + 1 >= 0x1000 )
          {
            v51 = *((void **)v153[0] - 1);
            if ( (unsigned __int64)((char *)v153[0] - (char *)v51 - 8) > 0x1F )
              _invalid_parameter_noinfo_noreturn();
          }
          j_BNFreeDataReferences_0_0(Block: v51);
        }
      }
      __unwind
      {
        sub_18096DDB0(a1: v139);
      }
    }
    __unwind
    {
      sub_18096DDB0(a1: v153);
    }
    __eh34_enter_wind_state(31, 36);
    v154 = 0;
    v155 = 15;
    LOBYTE(v153[0]) = 0;
    (*(void (__fastcall **)(__int64, _BYTE *, __int64))(*(_QWORD *)v91 + 16LL))(a1: v91, a2: v168, a3: 256); // 取 data 前 256 字节（动态调试容易看懂）

    // 省略...

    v65 = &unk_18AA4A090; // 再从中取 16 字节摘要，存在 unk_18AA4A090
    v66 = v168;
    for ( n = 2; n != 0; --n )
    {
      *v65 = *v66;
      v65[1] = v66[1];
      v65[2] = v66[2];
      v65[3] = v66[3];
      v65[4] = v66[4];
      v65[5] = v66[5];
      v65[6] = v66[6];
      v65 += 8;
      *(v65 - 1) = v66[7];
      v66 += 8;
    }

```

这部分有个隐含条件是 data Base64 解码后至少要有 256 字节，但由于前面要求 data 解码后必须是 280 字节，所以这里肯定是能通过的。

除 data 字段以外别的很多字段也有归档逻辑（email/type/count/expiresEpoch），但由于没校验（字段缺失也没事），就不赘述了。这些归档逻辑是解锁完整功能的关键。

##### 许可证吊销

serial 字段黑名单匹配：

```
 v54 = dword_18AA4A198; // dword_18AA4A198 = 0x1313（13107）数组长度
    v55 = qword_18AA4A190; // qword_18AA4A190 指向一个位于 .rdata 0x186DCC0A0（约 419 KB）数组
    *(_OWORD *)v159 = 0;
    v160 = 0;
    v161 = 15;
    LOBYTE(v159[0]) = 0;
    __wind
    {
      v136[1] = nullptr;
      v137 = 7;
      v138 = 15;
      v136[0] = (void *)0x3635322D414853LL; // 这个奇怪的数是 little endian 的 7 字节字符串 SHA-256
      __wind
      {
        sub_181CD0730();
        if ( v138 >= 0x10 )
        {
          v56 = v136[0];
          if ( v138 + 1 >= 0x1000 )
          {
            v56 = *((void **)v136[0] - 1);
            if ( (unsigned __int64)((char *)v136[0] - (char *)v56 - 8) > 0x1F )
              _invalid_parameter_noinfo_noreturn();
          }
          j_BNFreeDataReferences_0_0(Block: v56);
        }
        v137 = 0;
        v138 = 15;
        LOBYTE(v136[0]) = 0;
        if ( v161 >= 0x10 )
        {
          v57 = v159[0];
          if ( v161 + 1 >= 0x1000 )
          {
            v57 = *((void **)v159[0] - 1);
            if ( (unsigned __int64)((char *)v159[0] - (char *)v57 - 8) > 0x1F )
              _invalid_parameter_noinfo_noreturn();
          }
          j_BNFreeDataReferences_0_0(Block: v57);
        }
        v160 = 0;
        v161 = 15;
        LOBYTE(v159[0]) = 0;
        v58 = Src;
        if ( v117 >= 0x10 )
          v58 = (void **)Src[0];
      }
      __unwind
      {
        sub_18096DDB0(a1: v136);
      }
    }
    __unwind
    {
      sub_18096DDB0(a1: v159);
    }
    __eh34_enter_wind_state(42, 47);
    (*(void (__fastcall **)(__int64, void **, size_t))(*(_QWORD *)v90 + 16LL))(a1: v90, a2: v58, a3: v116); // calc serial sha-256
    sub_180A42140(a1: v90, a2: &v99);
    __eh34_enter_wind_state(47, 48);
    for ( m = 0; m < v54; ++m )
    {
      v59 = 32 * m; // v99 是 serial字段的 SHA-256 32字节哈希值
      if ( *(_QWORD *)v99 == *(_QWORD *)(v59 + v55)
        && *(_QWORD *)(v99 + 8) == *(_QWORD *)(v59 + v55 + 8)
        && *(_QWORD *)(v99 + 16) == *(_QWORD *)(v59 + v55 + 16) )
      {
        sub_181043690(a1: v110, a2: &byte_186D50093);
        throw (std::runtime_error *)v110;
      }
    }

```

`qword_18AA4A190` 指向一个很大的数组，里面的每一项都是一个 32 字节的哈希。`dword_18AA4A198` 是数组长度 13107。这两项通过动态调试看会比较容易。

这一段的逻辑是计算 serial 字段的 SHA-256 哈希值，得到一个 32 字节的摘要。再将这个摘要与一份黑名单中的每一项比较（但看起来只比较了前 24 字节，不知道为什么），如果相同就抛异常。猜测这部分的目的是吊销一些旧的许可证。

##### Personal 分支校验

最后会走进 Personal 分支，激活 Binary Ninja 的个人版：

```
if ( v71 == 21 && memcmp(Buf1: v74, Buf2: "Binary Ninja Personal", Size: 0x15u) == 0 ) // v74 是 product 字段的值，Size 是 product 字符串长度
    {
      if ( byte_18AA4A08A == 0 ) // byte_18AA4A08A 在 BNInitUI 中被设置为 1，所以不会走进异常分支
      {
        sub_181043690(a1: v111, a2: &byte_186D50093);
        throw (std::runtime_error *)v111;
      }
      off_18A80E040 = (__int64 (__fastcall *)())sub_180CCFFE0;
      off_18A80E038[0] = sub_180C51010;
      // 省略...
      byte_18AA4A089 = 1;
      // ...

```

只要 product 正常填 `Binary Ninja Personal` 就能通过校验。最后 `byte_18AA4A089` 被设置为 1，从而使得 `BNIsLicenseValidated` 返回 1。至此授权逻辑全部走通。

#### 破解思路

根据之前的分析，我们只需要：

1.  给 `binaryninjacore.dll` 打一个 1 字节的补丁（在 `0x14A386D` 处写入 1 字节 `0x85` 反转跳转逻辑），绕过签名校验
2.  随便生成一份符合格式要求的许可证

许可证里的很多字段都是可选的（可通过先前的分析以及实际测试得知），必填的字段仅有 data 和 product。product 必须是 `Binary Ninja Personal`，而 data 的唯一要求就是 Base64 解码后长度为 280 字节，内容无要求。

一种简单的破解思路：打开 Binary Ninja 所在目录，运行下面这条命令

```
python -c 'import os, base64, json; f = open("binaryninjacore.dll", "r+b"); f.seek(0x14A386D); f.write(b"\x85"); f.close(); f = open("license.dat", "w", encoding="utf-8"); json.dump([{"data": base64.b64encode(os.urandom(280)).decode(), "product": "Binary Ninja Personal"}], f, indent=4); f.close()'

```

这行命令包含了 dll 补丁和许可证生成两部分。也可以把其它字段补上，但必要的字段只有这两个。然后运行 Binary Ninja，点击 `Locate license file...`，选择程序目录下的 `license.dat` 即可完成破解。

### 结语

第一次逆向一款正式的 Windows 软件，即便提前知晓了许可证文件的格式，而且 Binary Ninja 没有混淆没有壳，许可证校验还特别容易过，依旧花了不少时间。希望这篇文章能帮到一些和我一样刚开始学 Windows 逆向的新手。

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)YukiQ 感谢分享，思路梳理得很清晰。分析的过程挺有参考价值，顺便后面试试能不能原汤化原食![](https://static.52pojie.cn/static/image/smiley/default/48.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 782878952 _ 本帖最后由 782878952 于 2026-8-5 18:35 编辑_  
感觉好复杂呀，目前通过软件想 update 到最新版本，许可校验还是失败的无法下载到最新版 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) xzchina 非常好, 其实我也跟过这个工具, 这个 DLL 非常大, 反汇编花了好久 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) daviddot 非常好，逻辑清晰，学习了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) andy_wang425

> [782878952 发表于 2026-8-5 18:27](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=55681336&ptid=2121326)  
> 感觉好复杂呀，目前通过软件想 update 到最新版本，许可校验还是失败的无法下载到最新版

这个真没办法，必须有一份真的许可证才能通过服务端的校验。我直接在 hosts 文件里加了一条  
0.0.0.0   master.binary.ninja  
让更新检查快速失败，以后想用新版本了再找资源或者自己破解吧。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)chewy 很值得学习，，，对我这种新人兼职太友好了 ![](https://avatar.52pojie.cn/data/avatar/000/17/62/35_avatar_middle.jpg) Coxxs _ 本帖最后由 Coxxs 于 2026-8-6 04:01 编辑_  
感谢分析交流！  
新版不确定，但是旧版的 data 里我记得是有一段密钥的，如果密钥错误的话，主程序里的某段常量会静默的解密失败，造成实际运行时出错。可以试一下保存和加载数据库功能是否还正常。  
也有可能现在免费之后，去掉了这些验证。