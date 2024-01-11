> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1879292-1-1.html)

> 我看了现在大一部分帖子，在修复 VMPIAT 的方法上都是通过扫描代码段的 pushreg call，call ret 这种特征来进行的对应地址修复。

![](https://avatar.52pojie.cn/data/avatar/000/88/51/96_avatar_middle.jpg)R-R， <0> 我看了现在大一部分帖子，在修复 VMPIAT 的方法上都是通过扫描代码段的 pushreg call，call ret 这种特征来进行的对应地址修复。我想也许应该有种更好的方法来修复 VMP 的 IAT。  
<1> 我以 VMP3.85 的版本示例来演示另外一种修复的方法，这里因为我们只需要修复 IAT，所以在资源加密上我选择了否。  
![](https://attach.52pojie.cn/forum/202401/11/130629xamezi05ip2i2775.png)

**1.png** _(59.98 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY2OTIyN3xlNWZlMzlhZHwxNzA0OTU0NTg3fDIxMzQzMXwxODc5Mjky&nothumb=yes)

2024-1-11 13:06 上传

  
<2>VMP3.8 以下的版本，在 IAT 调用的时候一般是以 lea reg,dword ptr ds:[reg+ 0xx] 这种形式调用，但在 3.85 的版本上可以看到在 iat 的调用上，  
lea reg,dword ptr ds:[reg+reg+0xxxx], 显然，在 3.8 的版本上 IAT 也带来了一些变化。  
但这并不影响，在图下 iat 的调用上，会有 reg+reg 调用，这里我回退看了下两个寄存器的值         
mov edi,0x8438AD1A  
mov eax,dword ptr ds:[eax+edi+0x7BC752E6]  
lea edi,dword ptr ds:[edi*2+0x97BCCB3C]  
Eax 是以内存的形式存储，edi 的值则是以一种固定的形式计算。  
![](https://attach.52pojie.cn/forum/202401/11/130713lq7yt4tlkfbz2fbl.png)

**2.png** _(160.67 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY2OTIyOHw4N2FlY2VkNXwxNzA0OTU0NTg3fDIxMzQzMXwxODc5Mjky&nothumb=yes)

2024-1-11 13:07 上传

  
<3> 思路打开：那么现在可以推断出，VMP 首先会调用自建的 GetProcAddress 来获取表里的 IAT 地址然后经过计算得出 Reg 的存储在内存的值，  
在调用时就会取出这个内存中的值经过在 vm 代码里面的计算得出 iat 地址，那么现在我们抛开一切不必要的因素找到重点: 找到它计算后的内存值，和对应的 iat 地址，然后我们构建一段汇编代码模拟 vm 的计算方式来计算出内存值填充进去。  
<4> 上面图中的 eax 值是从 vmp 段里获取的，对地址下硬件写入断点，运行几次在出现到内存值时如下图，Eax 为计算后的内存值，ecx 为写入值的内存地址，esp+10 为对应的 iat 地址。  
![](https://attach.52pojie.cn/forum/202401/11/130904bptetnak9dv4itkv.png)

**3.png** _(411.95 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY2OTIyOXwxNDNkNzNlNnwxNzA0OTU0NTg3fDIxMzQzMXwxODc5Mjky&nothumb=yes)

2024-1-11 13:09 上传

  
<5> 脚本构建思路，现在得到了计算后的值和对应的 iat 地址，那么我们要获取到那个被减值，也就是用来计算内存值的那个固定值，  
用脚本构建的话，获取到被减值，内存写入地址，iat 名，和对应的模块名，如下图，为了方便快捷，我选择了友好的易语言，当然脚本应该会更快，获取方法如下图。  
![](https://attach.52pojie.cn/forum/202401/11/131018tpmfzmflxmprmfea.png)

**4.png** _(73.22 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY2OTIzMXw1MjZhMDQ3NHwxNzA0OTU0NTg3fDIxMzQzMXwxODc5Mjky&nothumb=yes)

2024-1-11 13:10 上传

  
<6> 在就是 iat 修复了，有了这些信息，完全可以构建汇编来模拟 vmp 的计算方式。  
为了快捷方便，我选择了友好的易语言来构建 dll。  
首先我们得到了计算值，计算值大部分会产生借位问题，这里只需要保留低八位即可，因为在 lea 的时候越位也会舍去高位，这也是 vmp 的一个精妙设计之一。  
有了被计算值，有了模块名和对应的函数名，那么在写 dll 时就可以这样：通过函数名和模块名获取到对应的函数地址，然后通过函数地址减去固定的被减值，  
得到新的内存值，然后写入对应的内存地址，最后绑定 dll 导出函数，在运行 OEP 之前写入内存值，这样无论在任意机器上我们就仍然可以保证 vmp 在 vm 里调用 iat 地址时一直都是正确的 iat 了。  
dll 代码如下图  
![](https://attach.52pojie.cn/forum/202401/11/131115tv1qsbdv8ufv8l22.png)

**5.png** _(21.41 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY2OTIzMnwyOTA1ODIzMXwxNzA0OTU0NTg3fDIxMzQzMXwxODc5Mjky&nothumb=yes)

2024-1-11 13:11 上传

  
<7> 还有一点，vmp 在对 Getversion 和 GetversionEx 特殊照顾，他是直接保留在内存里，在[脱壳](https://www.52pojie.cn/forum-5-1.html)时可以在 vm 段里扫到这两个地址然后重建出来。  
<8> 我经过了几个版本的脱壳测试，这种修复方式还算是比较稳定高效的。  
<9> 题外话: 在 vmp 脱壳时 iat 修复其实算的上是比较小的一个问题。  
资源的修复，vmp 的 vm 校验，这些比较难，当然，其实深入一点也不是很难。  
在资源修复上补全 PEB.heap，HeapCreate 两段内存，找到 vmphook 的函数，然后照着挂钩上即可。  
至于 vm 函数里的校验，补版本的低四位和 cpuid 即可，当然，我选择更好的方式就是找到 cpuid 的算法，然后用算法计算出 hash 写到校验内存里，这样所有的 vm 函数都能在脱壳时过掉，而不需要补所有的 cpuid。  
最后就是 vmp 的授权，vmp 的授权在没加 sdk 时可以直接至 0 过掉，这个地方是比较简单的，最难的一点是 vmp 在 vm 函数时有个锁定到序列号，这个校验点除了 Patch 机器码和 keygen 之外，我发现了另外一直方法，只需要补全它的 ProductCode 即可。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)cm19890204 感谢分享