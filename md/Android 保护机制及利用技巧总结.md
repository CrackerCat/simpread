> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/jwoNSgcyQM0maozoYjPtwA)

> 本文总结了 Android 漏洞利用时遇到的 SELinux 以及软硬件层面的 BTI、PAC、CFI、MTE 保护机制及绕过方法。

**目**

**录**

一、前  言

二、真机内核利用适配

三、SELinux

四、CFI 保护

五、BTI 保护

六、PAC 保护

七、MTE 保护

八、AARCH64 JOP

九、总  结

  

**一**

  

**前  言**

本文总结了在 Android 上利用漏洞时遇到的一些新的保护机制以及在真机上的内核漏洞利用和调试技巧。虽然 Android 底层为 Linux 内核，但是相比较下 Android 内核更加难利用，主要体现在真机不能实时调试，可能开启了 BTI 保护、PAC 保护和 CFI 保护，同时在近年新出的一些手机，如 Pixel 10 开启了内存标记访问保护 Memory Tagging Extension(MTE)。本文还将介绍 MTE 保护在用户态时的一个特殊的绕过方法，通过探讨这些新的保护机制及其应对策略，我们希望能够帮助读者更好地理解当前 Android 安全环境，并为未来的漏洞研究提供新的思路和技术手段。

**二**

  

**真机内核利用适配**

对于一个真机内核，在编写漏洞利用程序期间可以编译一个版本一样的 Linux 内核用 qemu 模拟运行，便于掌握数据的处理过程。还可以使用 Android 模拟器，目前高版本的 Android 模拟器无法在 x86/x64 架构下模拟 AARCH64 的镜像，可以在 AARCH64 架构下的主机，如树莓派等下面运行模拟器。在模拟的内核中利用成功后，就是如何将其移植到真机上的问题。虽然真机不能实时调试，但是可以通过查看`/sys/fs/pstore/`目录下的日志文件以及`dmesg`来获取内核最后崩溃时的寄存器值。根据寄存器信息来定位漏洞利用程序中需要适配的位置。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MibA29zVuyPbffxa6CIibGmxoYczzl0ZMSia08e2Msl5pyhOuSFMicRan8Q/640?wx_fmt=png&from=appmsg)

**三**

  

**SELinux**

SELinux 是一个强制访问控制安全机制，它提供了一种灵活的、细粒度的访问控制策略，用于提高 Linux 系统的安全性。Android 上默认开启了 SeLinux，因此某些漏洞利用方法在编译的 Linux 内核中能够使用但是在 Android 上测试却失效了。

**01**

**SELinux 原理**

  

SELinux 实际上是对系统中所有的关键函数注册了`HOOK`。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MI2aWibAqRl6LoWlzKkNq4OibqFrhHVyTu6fMJHupxuFN86mah3YRDvXA/640?wx_fmt=png&from=appmsg)

这些`HOOK`函数会在函数中被调用，它们一般以`security_`开头。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MQOiaHpfUo338sOL0eINFOwTuS5MibWIpnPxbMic0MrAz7eCt4UJO2AXQw/640?wx_fmt=png&from=appmsg)

如果 SELinux 没有开启，这些`security_`函数默认返回 0 让程序继续程序，如果开启了则跳转到`HOOK`函数执行。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7Mz47h8xjuyQgP1zibrlL4dPLtB7JSmRLag9CayTPaQiaoS10EAhzJKsoA/640?wx_fmt=png&from=appmsg)

这些 HOOK 函数根据 SELinux 配置的规则对参数进行审计，以此来让一个函数执行或者拒绝。

**02**

**SELinux 绕过**

  

当开启 SELinux 时，改写`modprobe_path`或者`core_pattern`后不能触发提权脚本的执行，这是因为我们指向的脚本不在 SELinux 规则中规定的可执行路径。为了绕过 SELinux 的检查，我们查看审计函数的代码，`avc_has_perm`函数的子调用链为`avc_has_perm->avc_has_perm_noaudit`。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MB6xJk1EDIulOdlUxicovukErsFwQ2jT1Rcm40gQbDGLwHO19bhsPciag/640?wx_fmt=png&from=appmsg)

如果`avc_has_perm_noaudit`函数审计出当前的操作是被禁止的，那么调用`avc_denied`函数。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7M8aaqXaIqaGGOTUXUiarciavlmac1tibg0WSgL0VugCNWmDQMN6bOQNpXA/640?wx_fmt=png&from=appmsg)

从`avc_denied`函数来看，如果`selinux_enforcing`全局变量为 0，则仍然可以使得`avc_denied`返回 0，进而让`selinux_`函数放行，因此可以利用漏洞改写`selinux_enforcing`这个全局变量来绕过 SELinux。

在高版本 Linux 中，判断方式采用了函数，实际上判断的是`state->enforce`。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7M0yVVicLIaVgw3Gxne6p6KFWgPu6s94NiasvVTsY7S8GouhcHaX7Uvhbg/640?wx_fmt=png&from=appmsg)

而`state`指针指向的仍然是一个全局变量结构体。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7Mf6yEhIasEadibskrhTe0Duv95N9Z0h4Z0DYTsRQOoEFsB7XRRkpJMMw/640?wx_fmt=png&from=appmsg)

因此可以修改`selinux_state.enforce`变量。

**四**

  

**CFI 保护**

CFI 保护是 Android 内核中引入的，目的是保护函数指针，如果函数指针被篡改为任意地址，会被检测出来然后终止执行。开启了 CFI 保护的内核如下所示，会有很多以`.cfi`结尾的函数。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MHG6luwQriaI06icbcVcoE9kXicer87mib96oB1FnKMploQ3IX3xMppCVVQ/640?wx_fmt=png&from=appmsg)

还存在着不带`.cfi`结尾的同名函数。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7Mrc1hS387ujm7pQUhZmwicxkZKRv9Q8vjb2iatiaKWl71gQlLDRGzo0ZCg/640?wx_fmt=png&from=appmsg)

不带`.cfi`的函数中只会有一条 B 跳转指令，不会再有其他任何人指令。实际上这些函数是一张类似于 PLT 跳转表的东西，我们可以把它命名为 CFI 表。

**01**

**函数指针检查**

  

CFI 的检测实际上就是对每一个函数指针调用的位置进行了插桩，判断函数指针是否在 CFI 表中，如下是 CFI 的桩代码：

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7M7oT5ibia73pMFAqW45micmSmMGNsgHVBsmK1IIDzXPupsGxrkEg6Zgw3w/640?wx_fmt=png&from=appmsg)

如果函数指针发生了篡改，则将进入`_cfi_slowpath`函数，`_cfi_slowpath`函数调用`_cfi_check`进行检查。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7M2bJPPrpkf6icJmhpVYNeplxsjP1yFeTAx2gtTdrnmv19PYoAE8S6l1g/640?wx_fmt=png&from=appmsg)

`_cfi_check`根据`_cfi_slowpath`函数的第一个参数传入的`MAGIC`值，会再一次的判断函数指针是否能够通过检查。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MwCicgBuN2GdfxmFLEwn3T7mkbFbicz29nxwUhDibuaSNh4nxibw4T2GUhw/640?wx_fmt=png&from=appmsg)

如果函数指针与预期值不等，则调用`__cfi_check_fail`函数让内核崩溃。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7Mn7XoNfqEeNze6uonQO327OxuxLiaoPJN0XXiacRX6cbN1NnqAZLnEPDg/640?wx_fmt=png&from=appmsg)

**02**

**函数指针多值的处理**

  

某些函数指针可能有多个指向的目标，因此不能对函数指针进行固定值比较，CFI 采用了运算的方式将指针值限定在一个范围内。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MRexXGufrqU1IcZB57TxTxsll3MRvRUnEKWicSKcj1EBJZLic0ibySnWDA/640?wx_fmt=png&from=appmsg)

即只能在 CFI 表中的`single_step_handler`附近。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MsodrBLcibq5uTjVImj4utMXDd0Zd7E6u7dTcgbnChULRU9iaEtOfEALA/640?wx_fmt=png&from=appmsg)

显然，在编译时，生成的这张`CFI表`中函数的排列顺序是精心计算安排的，把一个函数指针所有可能的指向地址排列成相邻的。

**03**

**CFI 绕过的可能思路**

  

对于 ARM 架构，目前无法绕过 CFI，因为 ARM 架构的指令是对齐且定长的，不能在`CFI表`中跳转到错位的地址进而构造出`ROP gadget`。如果是在`x86`架构下，对于函数指针多值的 CFI 检查，由于指针值限定在 CFI 表的一个范围区间，可以在区间内寻找是否有合适的`gadget`能够控制执行流。

**04**

**CFI 例题**

  

在 GeekCon 的`ksocket pixel3`题目中，实现了一个自定义的`socket`，我们可以通过 UAF 控制这个`socket`对象的结构。由于开启了 CFI，我们不能去控制函数指针。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MibWd8eDVLHKSVXPT7G5XSqbNoEDibjFud02fPnSWRUuJiccMvkR1WjBrA/640?wx_fmt=png&from=appmsg)

我们观察到，在`close`时触发的`avss_release`函数中有以下的链表`unlink`操作：

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MA5Y4bu1osjLXHauc3GeK0nokialyIyqnV5ial9LMmvK9Zoiaqpvl4OdNw/640?wx_fmt=png&from=appmsg)

我们可以把`unlink`用来做任意地址写，由于两个数据都必须为合法的内存指针，因此不能直接写数据。但是可以用错位的思路，CPU 为小端，因此指针的最低一个字节存放在最前面，我们每次只需要保证指针的最低一个字节被写入到目标地址即可。令`*(v3 + 112) = addr, *(v3 + 104) = bss | byte`，则可以在`addr`处写上一个字节 byte。其中 bss 为 bss 的地址，用于保证两个数据都为合法的内存指针不会崩溃。在实现了任意地址写以后，改写`selinux_enforcing`为 0 关闭 selinux，改写`modprobe_path`为提权脚本。然后触发`modprobe_path`的执行。

**五**

  

**BTI 保护**

在 AArch64（ARMv8-A 架构的 64 位模式）中，BTI 指令用于验证间接跳转的目标是否有效。它的主要作用是确保程序控制流只能跳转到预期的代码位置（即合法的分支目标）。即 BLR/BR Rn 寄存器跳转指令跳转的目标位置的第一条指令必须为 BTI 否则函数无法继续向下执行。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MMeqp1POtpIeEtwPEz1kVM8PBbPRqRq0ictTEfLHZQTLAhVrdNMPKLsQ/640?wx_fmt=png&from=appmsg)

**六**

  

**PAC 保护**

**01**

**PAC 原理**

  

PAC（Pointer Authentication） 技术，用于验证和保护返回地址及其他指针数据的完整性。ARMv8.3-A 新引入了三类指令：

*   `PAC*` 类指令可以向指针中生成和插入 PAC。比如，PACIA X8，X9 可以在寄存器 X8 中以 X9 为上下文，APIAKey 为密钥，为指针计算 PAC，并且将结果写回到 X8 中。
    
*   `AUT*` 类指令可以验证一个指针的 PAC。如果 PAC 是合法的，将会还原原始的指针。否则，将会在指针的扩展位中将会被写入错误码，在指针被间接引用时，会触发错误。比如，AUTIA X8,X9 可以以 X9 为上下文，验证 X8 寄存器中的指针。当验证成功时会将指针写回 X8，失败时则写回一个错误码。
    
*   `XPAC*` 类指令可以移除一个指针的 PAC 并且在不验证指针有效性的前提下恢复指针的原始值。PAC 的加密生成算法不同的硬件有不同的实现。
    

在 Android 中，开启了 PAC 保护的函数如图所示，PACIASP 指令会基于当前的栈指针（SP）、私有密钥（APIAKey）以及返回地址生成认证码, 认证码被嵌入到给定的函数返回地址中, 在函数返回时，使用对应的 AUTIASP 指令对返回地址进行验证。如果地址合法且未被篡改，验证成功；否则，程序会触发异常（SIGILL 或其他非法指令异常）。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7Mv88odfbJEQgJlFdXjhqR6iaKcyY0f9VuChcHAUYEKyzsoR90EqyOtdw/640?wx_fmt=png&from=appmsg)

**02**

**PAC 绕过**

  

PAC 绕过是困难的，PAC 的密钥通过特定的系统寄存器存储和操作。内核态使用的密钥是`APIXKey_EL1`，用户态使用的密钥是`APIXKey_EL0`，因此在用户态计算出的 PAC 值不能给内核态使用。内核态下可以操作访问`APIXKey_EL1`、`APIXKey_EL0`等寄存器修改或者读取密钥。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MIB2rJhamb6bEnwgf12HE8BV0YCzvYJb8kz6Rtgp9DfsostgSYQXTTg/640?wx_fmt=png&from=appmsg)

因此有一种可能的情形就是在内核态中某个`gadget`可以将用户态的`APIXKey_EL0`修改成与内核态一样的数值，那么就可以在用户态执行 PAC 指令计算 PAC 值然后填入 ROP 链。

**七**

  

**MTE 保护**

**01**

**MTE 原理**

  

MTE (Memory Tagging Extension) 是 ARMv8.5-A 架构引入的一项硬件支持的内存安全技术，旨在检测和防止内存相关的错误和漏洞，例如越界访问和使用已释放内存（Use-After-Free, UAF）。MTE 的基本原理：

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MCzjiaic8poJ4n0ZUoTxo39eicXSva8Q1Y897PicqUKuMCbQ9mgnorSYaKA/640?wx_fmt=png&from=appmsg)

*   IRG (Insert Random Tag) 指令为指针 Xn 生成一个随机 tag，使用 Xm 作为种子，将结果保存至 Xd 中。
    
*   STG (Store Allocation Tag) 指令将 tag 应用至内存中，生效的长度取决于颗粒度，一般为 16 字节。
    
*   LDR (Load Register) 使用带有 tag 的指针读取内存。
    

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7M4MRy0Ar8fBrmg1vmO6rvUGfc9faangJ3BnCkvSOE6GY23w73mbQSvQ/640?wx_fmt=png&from=appmsg)

如图，IRG 指令执行后，X0 比 X8 在高位多了一个 TAG 值。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MbaNLqiaj5vzvnnUf3ibu6oy4uj51Ls09BBygbyvicCH5wRiaDJ2libdsPKg/640?wx_fmt=png&from=appmsg)

STG 指令执行后，以后访问这段内存需要带上正确的 TAG 值的指针才能访问，否则指令会执行错误。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MYMHWFO3YLia5lPIMK4v23sahACwrrhfIuYAf9koNVn2aXiapwOwINnwQ/640?wx_fmt=png&from=appmsg)

**02**

**MTE 应用**

  

在堆分配器中，`malloc`后，通过对申请的堆地址打上标签返回，`free`后对堆地址重新打标签。这样就能阻止 UAF 这类的漏洞，因为`free`后指针重新打了标签，导致 UAF 残留的指针无效，通过 UAF 的指针访问内存时就会崩溃。不同的堆分配器在`malloc`和`free`时有着不同的处理内存标签的方式。有关内存分配器处理 MTE 标签的分析可以参考文章 GeekCon 的文章填补盾牌的裂缝：堆分配器中的 MTE。

**03**

**MTE 爆破**

  

如果给系统调用直接传一个带有错误 TAG 的指针，会发生什么？如图，假设`buf`指向的内存已经被`free`导致重新打标签，现在传给`Sys_write`的是一个无效的指针。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MX2YYmC0xt6iaZHZGIZn4s1e0o0tiaKPMkGjGX3GMfyC6ZWX75c4WZ6fw/640?wx_fmt=png&from=appmsg)

单步进入会触发内核的`Error EL1h`。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7Mjn6VkFVJDtIRp7k7l1BtM2MmSCSq1UsvMytmn8ByuY1OLA6EfXGyzw/640?wx_fmt=png&from=appmsg)

错误会被`el0t_64_sync`函数捕捉处理。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7Maibchsfy3DFm0ibG2NTPXrhaCv9ib2z4ufJicm0pzUrn5lIzPsHuibZFXEw/640?wx_fmt=png&from=appmsg)

异常处理会调用`el0_svc`函数，并不会退出程序。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7Mcia9OqY2icsaQfyfojylniaBnfPL965tTriaxPwvicaGz9FDhS5wkfSv2BA/640?wx_fmt=png&from=appmsg)

异常处理完成后，调用`ret_to_user`返回到了用户态。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MnFUzEWJciaGiadrRbrpDHJWylRcRoqo6l43o7p1gVLGjsIQW5p7I2LXA/640?wx_fmt=png&from=appmsg)

可见，当一个不正确的 MTE 指针进入系统调用，系统调用执行不成功，同时进程不会崩溃；我们可以利用这种特性来对 TAG 值进行爆破。一般的，我们在用户态利用 UAF 漏洞时，在已知指针值但是不知道 TAG，我们可以用这样的方法爆破。

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MWialAyauhvyKrC7fXjqvrQhAR0WYDMQaFgwGPLSDnY03sRia8domZtnQ/640?wx_fmt=png&from=appmsg)

上述代码来源于我在 GeekCon Shanghai 2024 上解出的 MTE 题的 EXP。

**八**

  

**AARCH64 JOP**

在 AARCH64 中，RET 指令不会从栈里弹出返回地址进行返回，RET 指令直接跳转到`X30`寄存器指向的地址；而 BLR 指令在跳入新函数时，会将返回地址赋值给`X30`寄存器。由于这个特性，我们在搜索一些`gadgets`指令时，无需考虑 BLR 后面的代码。

在做 GeekCon 的`kSysRace`赛题时，我们控制了一个地方的函数指针，能够调用任意一个函数，以及`X0`执行的内容可控：

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MEROf1OztaA7ccJicpNBr3rmPm5WCGcG97rjh27iaZqARicxSwnFTnb7Pw/640?wx_fmt=png&from=appmsg)

让其先跳入下面的代码：

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MPTAtvWJFShDyTeF4tbG9vPlnm4TeiaPMdbWcvia5eBAXfu9IaiaIArx1w/640?wx_fmt=png&from=appmsg)

在这段代码中，我们的目的是控制`X19`指向`X0`，因为`X0`是我们可控的，我们不用担心`BLR X8`返回执行后面，因为我们可以再调用一次 BLR 来将`X30`覆盖。我们控制`X8`，让其先跳入下面的代码：

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7MLlfhMTfGa9vS4SkIHRHjReKCIPfgFHQ7SXBLBx5ojqT5dnK4jEAvRg/640?wx_fmt=png&from=appmsg)

在这段代码中，由于`X19`可控，我们可以调用 3 个参数的任意函数了，自始至终，我们的栈没有发生过调整，由于漏洞发生的位置栈尾部是这样的：

![](https://mmbiz.qpic.cn/mmbiz_png/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7M2wHjv5u7DzRRPgDqxUh7YfBJQLxfsGChD48eSpNClKaAGXHibpzKCBA/640?wx_fmt=png&from=appmsg)

栈尾部跟我们的`gadgets`一模一样，这意味着我们的`gadgets`在执行到 RET 时可以直接返回到漏洞发生的函数的上层，栈平衡了。也就是我们能够执行任意的一个函数，控制 3 个参数，同时栈能够恢复，可以让程序继续保持正常的运行状态。这样我们就可以进行多次的任意函数调用。

**九**

  

**总  结**

本文我们介绍了众多在 Android AARCH64 上所使用的保护机制以及特性，劫持程序流程变得越来越困难，在没有开启程序流保护的情况下，使用 JOP 去实现任意代码执行；当程序流保护机制开启时，可以转变思路，通过劫持一些数据结构体，利用程序中自带的`link`、`unlink`等操作去实现一个地址写或者读，本文还介绍了 MTE 保护机制的一种特殊情况下的爆破。

**十**

  

**参考链接**

1. AVSS 2024 Final Writeup

2. 填补盾牌的裂缝：堆分配器中的 MTE

  

  

  

  

【版权说明】

本作品著作权归 **ha1vk** 所有

未经作者同意，不得转载

![](https://mmbiz.qpic.cn/mmbiz_jpg/9EP6QFMcTmR75I6PS5ziaq9TvIuGrfw7M30yo76uRAwV8jZxzhelAr61REJcvAGK0YfX32XYEEVia66kQVwA3JrA/640?wx_fmt=jpeg&from=appmsg)

**ha1vk**

  

二进制安全研究员，BlackHat USA 2022 WASM 演讲者，专注 IOT、硬件、内核等方面的研究。

**往期回顾**

  

**0****1**

[CVE-2024-38054 Windows ksthunk.sys 驱动提权漏洞分析](https://mp.weixin.qq.com/s?__biz=Mzk0OTU2ODQ4Mw==&mid=2247486662&idx=1&sn=f7cb77059e031ea7368c2a1346543bf3&scene=21#wechat_redirect)

**0****2**

[WSGI 中的请求走私问题研究](https://mp.weixin.qq.com/s?__biz=Mzk0OTU2ODQ4Mw==&mid=2247486622&idx=1&sn=31ab4ff4ceabfd9ebea2b03448d74007&scene=21#wechat_redirect)

**0****3**

[硬件辅助虚拟化及 Fuzzing 工作研究](https://mp.weixin.qq.com/s?__biz=Mzk0OTU2ODQ4Mw==&mid=2247486569&idx=1&sn=db648a22c8a4d50126b01e872b092f08&scene=21#wechat_redirect)

**0****4**

[vCenter 漏洞分析：CVE-2024-37079 & CVE-2024-37080](https://mp.weixin.qq.com/s?__biz=Mzk0OTU2ODQ4Mw==&mid=2247486533&idx=1&sn=4b217f75761d56f55ddb21551727d214&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/oJZWTpJpiae90UpibicIKeZgQTNjebiaOwStfe6MJ5J6RC7F9JDFdX2kaEwibFz7GewNtNyDek6SdENJrXjf0KXA2kg/640?wx_fmt=png)

**每周三更新一篇技术文章  点击关注我们吧！**