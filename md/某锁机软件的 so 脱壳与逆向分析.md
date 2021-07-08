> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1472169-1-1.html)

> [md]## 0x01 背景高考完了，有时间水群吹牛逼了。

![](https://avatar.52pojie.cn/data/avatar/001/60/99/79_avatar_middle.jpg)fnv1c _ 本帖最后由 fnv1c 于 2021-7-8 09:15 编辑_  

0x01 背景
-------

高考完了，有时间水群吹牛逼了。碰巧某同学说自己平板被锁机软件锁了，只能通过输入动态码解锁。他提供了样本 apk，说希望能逆出来动态码算法。虽然对安卓逆向非常陌生，但总比打一天游戏更有意义，于是开干。

> **免责声明：本文旨在与分享移动安全相关知识，相关内容已作脱敏处理。请勿用于非法用途。**

0x02 前置知识和工具
------------

本文主要是 so 层的脱壳逆向，几乎没有涉及 java 层的知识

> 预热篇：  
> 前置知识：有手就行  
> 工具：可以反编译 ARM 的 IDA   
> 
> 正篇：  
> 前置知识：C++ 逆向经验，ELF 相关的一点经验，frida 基础使用  
> 工具：ADB,frida, 可以反编译 ARM 的 IDA,xAnSo

0x03 预热篇
--------

基础逆向发现 java 层通过调用 JNI 层函数`verifyDevtoken(用户输入内容,null)`来判断用户输入解锁码是否正确。  
![](https://attach.52pojie.cn/forum/202107/07/191106lvfyogumjpnpc2j3.png)

**图片. png** _(84.16 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgyMnw1NDFiNmZmMXwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:11 上传

![](https://attach.52pojie.cn/forum/202107/07/191144vz3pkqz4atfa16wt.png)

**图片. png** _(22.39 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgyM3xjYWFiNDYxOXwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:11 上传

  
找到对应的 so，ida 打开，肉眼可见验证函数。  
![](https://attach.52pojie.cn/forum/202107/07/191233zep9m9tyj9zpyuma.png)

**图片. png** _(15.9 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgyNnxhNGIwMzQ2YnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:12 上传

  
![](https://attach.52pojie.cn/forum/202107/07/191246r9mmnhxmny9ke3dy.png)

**图片. png** _(40.01 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgyOHw4NjljMzczOHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:12 上传

  
发现对 a1 进行了几次虚函数调用，然后和 creatDevToken 函数生成的动态码比对并返回结果。  
根据函数 signature 可以看出 a1 为 JNIEnv，根据 JNIEnv 虚表可知 + 676 出为`GetUTF8Char`，即获取 jstring 的内容。  
跟入 creatDevToken 逻辑比较清晰，但在最后发现一个古怪的函数调用 DDD4();  
![](https://attach.52pojie.cn/forum/202107/07/191311d8qeiqrtq4tteeqq.png)

**图片. png** _(63.17 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgyOXw3ZDQ5ZjI1ZnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:13 上传

  
跟入发现是 std::string 构造函数的 thunk 函数，显然函数参数个数不对，继续跟入 f5，然后返回再次 f5，直到 creatDevToken 逻辑正常。  
![](https://attach.52pojie.cn/forum/202107/07/191329fjt3cijs3ic3lgrg.png)

**图片. png** _(8.54 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgzMXwyZDVhOWUwMnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:13 上传

  
由逻辑判断此函数应该为 substr,v15 为计算完成的 key，根据时间对 key 做 substr 再返回。  
![](https://attach.52pojie.cn/forum/202107/07/191340z8ms88b6zrprpcb8.png)

**图片. png** _(14.93 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgzMnwyODU4MDIwYXwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:13 上传

  
容易写出 keygen  
![](https://attach.52pojie.cn/forum/202107/07/191354tb070u0eumlkeely.png)

**图片. png** _(46.67 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgzM3w4ZWQ2NGRkOHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:13 上传

0x04 正篇
-------

发完 key，他告诉我 key 解不了。他的应该是新版本软件，又来了个新版本样本。我人傻在那里，这就是买一赠一吗，爱了爱了。好在 so 名字没变，直接拖 ida 看，发现 export 炸了。。。  
![](https://attach.52pojie.cn/forum/202107/07/191409g3jqefnq3app31wv.png)

**图片. png** _(78.24 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgzNHxjNGJhN2U1ZHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:14 上传

![](https://attach.52pojie.cn/forum/202107/07/191511dlukdkdkwgqmmce1.png)

**图片. png** _(60.23 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgzNnwwN2VlYWE2YnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:15 上传

  
![](https://attach.52pojie.cn/forum/202107/07/191524tvebo122y1uvhhcl.png)

**图片. png** _(98.84 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgzN3xjZjRiZmM5OHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:15 上传

  
全部都是只有 ret 的小函数，又有不正常的段，应该是加了奇奇怪怪的壳子。搜不到 jni 调用的函数，说明可能是动态注册 (RegisterNative) 或者是动态生成 soinfo 的强壳。看了下 init_array 和 jni_onload，发现啥都看不懂。  
![](https://attach.52pojie.cn/forum/202107/07/191556ej9fjj95jq11i1nj.png)

**图片. png** _(17 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgzOHxhMDM4MjI5NnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:15 上传

  
![](https://attach.52pojie.cn/forum/202107/07/191605fthzz6u1eih6hezk.png)

**图片. png** _(73.01 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODgzOXw5MDQzODE5M3wxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:16 上传

  
![](https://attach.52pojie.cn/forum/202107/07/191617gryrrf8fjrorjff6.png)

**图片. png** _(83.64 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0MHw1ZmNhN2NlM3wxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:16 上传

  
![](https://attach.52pojie.cn/forum/202107/07/191629a3g8mgm33ibz3izm.png)

**图片. png** _(61.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0MXwyOTg5ODk5YXwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:16 上传

  
只能上 frida 真机调试了。github 上找一个可以 hook RegisterNative 的脚本 [frida_hook_libart](https://hub.fastgit.org/lasting-yang/frida_hook_libart)，载入启动。启动过程中没有见到动态码相关的函数 RegisterNative，应该是动态生成 soinfo 的强壳。考虑到 JNI 层验证动态码必须要获取 jstring 内容，所以给`GetUTF8Char`下断，根据调用者判断验证函数的地址。  
![](https://attach.52pojie.cn/forum/202107/07/191654g14lclnl6nlruuyu.png)

**图片. png** _(72.64 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0M3w0ODNhYzA0ZnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:16 上传

  
终于发现 so 库的名字和 JNI 函数名，但是由于加了动态 soinfo 强壳，只能在 frida 中获取库基址和函数地址，ida 中看不到导出项目。看看 maps 有什么特点。  
![](https://attach.52pojie.cn/forum/202107/07/191711cjok7o3i7oewz359.png)

**图片. png** _(26.86 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0NHwwMDMxYzNhYnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:17 上传

  
![](https://attach.52pojie.cn/forum/202107/07/191721eci0tk2kkld26zrk.png)

**图片. png** _(64.25 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0NXw4YjdmNDUzY3wxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:17 上传

  
验证函数居然不在 so 的映射范围内，而是在一个新的匿名段。四个匿名段有很典型的 rx,rw,ro 的权限分布，猜测是子母 ELF 壳，母 so 加载时解密真实 ELF，并且装入内存。魔改 soinfo 使得子 ELF 的导出项对外可见。由于 soinfo 中库基址指向母 ELF，直接将子母 ELF 一起 dump 到文件分析。  
![](https://attach.52pojie.cn/forum/202107/07/191741rtrqyc7cz87ym8ai.png)

**图片. png** _(17.24 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0Nnw5NDk3ZjU1NnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:17 上传

  
由于是内存 dump，ida 分析时需要选择手动装入并指定基址。  
![](https://attach.52pojie.cn/forum/202107/07/191818j4lg7igde4ktxfl1.png)

**图片. png** _(11.45 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0OHw5MGFhZjRlNXwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:18 上传

  
![](https://attach.52pojie.cn/forum/202107/07/191804l2v91zv99l41q496.png)

**图片. png** _(10.21 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0N3wxZjUyNzdlMHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:18 上传

  
装入后就可以找到真实 jni 函数的导出项，但是 ida 无法显示其中内容。  
![](https://attach.52pojie.cn/forum/202107/07/191838yfkkdwwk2lduf2np.png)

**图片. png** _(17.9 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg0OXxlZGM2ZDUxZnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:18 上传

  
![](https://attach.52pojie.cn/forum/202107/07/191848t33icu6bbrzizdu4.png)

**图片. png** _(44.84 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1MHxmOTZmZDM3ZnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:18 上传

  
经过查找资料，得知内存中 dump 出来的 so 必须修复 sections 才能被 ida 正确分析。通过 xAnSo(github 上可以找到) 进行修复，再次拖入 ida 分析。 ![](https://attach.52pojie.cn/forum/202107/07/191917rkex7eumuumuvmoh.png)

**图片. png** _(45.21 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1MXxmZmEwZTEyN3wxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:19 上传

  
ida 这次可以读出函数的内容，但是无法创建函数，提示控制流错误。  
![](https://attach.52pojie.cn/forum/202107/07/191934qo1l99bj5h0bezaz.png)

**图片. png** _(67.63 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1MnwxNzgwYTA0MnwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:19 上传

  
定位到错误点附近，发现已经有了 pop PC, 而下面有一个函数调用，随后是一串 string 的析构函数。熟悉 C++ 逆向的朋友会发现下面一串析构函数调用是在此函数尾部一个 ret 后，大概率是栈展开相关代码。BLX 的调用大概率不是正常的控制流，可能是错误处理。此时可以对照新旧 so 的汇编看，发现 BLX 其实调用的是 stack canary 检测失败时的报错退出函数 (_stk_chk_failed)。双击进入这个函数，编辑函数，添加 noreturn 属性。然后就可以成功创建 jni 导出函数了。  
![](https://attach.52pojie.cn/forum/202107/07/191946k4fkfqvjdumkju0b.png)

**图片. png** _(8.98 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1NHwzMjk4OWY5MHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:19 上传

  
找到了 key 生成函数和关键跳，大概逻辑是生成 key 并和用户输入对比，对比完返回结果。  
![](https://attach.52pojie.cn/forum/202107/07/191957jrh6tub4obez8pbl.png)

**图片. png** _(46.67 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1NXw2MjUwNjNmNXwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:19 上传

  
追入 key 生成函数，发现 ida 无法反编译。在 ida 窗口调到对应地址，发现 ida 无法将其识别为函数 (因为上文中_stk_chk_failed 的问题)。手动创建函数，重新 f5，成功。  
![](https://attach.52pojie.cn/forum/202107/07/192007eoxp9rezmurugxlm.png)

**图片. png** _(10.92 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1Nnw0ZTE3ZDZkZHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:20 上传

  
自此可以观察到完整生成逻辑。  
![](https://attach.52pojie.cn/forum/202107/07/192019krcvk7f8lclr4zek.png)

**图片. png** _(43.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1N3w5NjZmYzhjOHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:20 上传

  
然而还是遇到了一些奇怪的函数，比如下图，跳转到了一个 ELF 地址范围外的函数。  
![](https://attach.52pojie.cn/forum/202107/07/192032gew6wf99tl6f69th.png)

**图片. png** _(44.68 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1OHxlMThkYTlmMXwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:20 上传

  
![](https://attach.52pojie.cn/forum/202107/07/192046twir2lz9rmze9is6.png)

**图片. png** _(12.53 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg1OXw2MDI4MDM4NHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:20 上传

  
其实这个函数是 plt 表中的 thunk，调用的是导入的外部其他库的函数。由于 xAnSo 不带导入表修复的功能，所以 IDA 无法识别函数名，只能当作是间接跳转处理。而 GOT 表中已经存放了相应函数的地址，ida 会反编译成 JMPOUT(外部函数地址)。可以通过 frida 的`DebugSymbol.fromAddress`功能获取对应地址的函数。进行手动导入函数修复。

第二个奇怪的地方是对 string 的读取部分，会看到下图中的特征代码。其实是 stl string 的 SSO。限于篇幅，本文不介绍 SSO 技术。可以手动定义一个 string 结构体，导入非 SSO 状态下的 string 内存布局。会让 ida 的伪代码更清晰一些。  
![](https://attach.52pojie.cn/forum/202107/07/192106xgaclal71ql9z5ai.png)

**图片. png** _(19.37 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg2MXw5YjQ2ZDg2MXwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:21 上传

![](https://attach.52pojie.cn/forum/202107/07/192118osgk3kkukyof5bfg.png)

**图片. png** _(9.68 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg2MnxjYTQzODYwZHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:21 上传

  
注：frida 中读取 string.data() 的实现  
![](https://attach.52pojie.cn/forum/202107/07/192209v558ckzz75k5aekd.png)

**图片. png** _(31.03 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg2M3xiNjBmZmE0MHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:22 上传

  
看完全部逻辑就可以写 keygen 了，经过测试 key 正确。  
![](https://attach.52pojie.cn/forum/202107/07/192219t4lb4oodh8dvk4hb.png)

**图片. png** _(4.78 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMwODg2NHwxMjA0NTI5NHwxNjI1NzA3NzI0fDIxMzQzMXwxNDcyMTY5&nothumb=yes)

2021-7-7 19:22 上传

  
(为脱敏本文不附带新版本 key 生成过程伪代码)

0x05 结语
-------

脱壳逆向过程比想象中艰难恶心，但也给人一点启示。  
**要有终身学习的觉悟**：在遇到 so 强壳时，因为没有安卓逆向经验，差点放弃。幸好狠下心花半个小时熟悉了下 frida，追到输入的动态码后带来了继续下去的动力。如果因为从没做过安卓逆向就放弃学习，只停留在熟悉的领域，估计这辈子就和安卓逆向无缘了。  
**做软件不能本末倒置**：当一个软件为了防破解而做的工作远大于软件内容本身时，就要仔细考虑是否需要引入这么复杂的防破解手段。对于某些用户必须使用的软件，砸着经费，版本迭代着，软件功能的风评却不改善，反而是于用户无用的加密手段越来越先进，何尝不是一种本末倒置，更给破解再二次倒卖的行为提供了滋生的温床。![](https://avatar.52pojie.cn/data/avatar/001/60/99/79_avatar_middle.jpg)fnv1c _ 本帖最后由 fnv1c 于 2021-7-8 09:15 编辑_  

> [laos 发表于 2021-7-8 03:59](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=39183873&ptid=1472169)  
> 配套 frida 代码能不能传一下  当然那些重要信息可以处理下

  
[https://hub.fastgit.org/lasting-yang/frida_hook_libart](https://hub.fastgit.org/lasting-yang/frida_hook_libart)  
根据真机环境可能得稍微改改脚本才能用![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)laos 配套 frida 代码能不能传一下  当然那些重要信息可以处理下 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)王者归来 back 不懂的我居然看完了！![](https://static.52pojie.cn/static/image/smiley/default/biggrin.gif)![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Zhili.An 不得不说，还是很厉害的 ![](https://avatar.52pojie.cn/data/avatar/001/63/08/72_avatar_middle.jpg) djxding 楼主牛 B 啊，向你学习。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)aihaibo 虽然不懂 还是看完了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) qcl54651 不懂的我，也看完了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) goda 谢谢分享，学习了 ![](https://avatar.52pojie.cn/data/avatar/000/35/79/26_avatar_middle.jpg) lies2014 这个可以提供样本吗？![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)毁我容颜 高中生就这么厉害，牛