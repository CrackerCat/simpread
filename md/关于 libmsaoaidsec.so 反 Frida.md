> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2008459-1-1.html)

> [md]# 前言最近在分析绕过 frida 检测的姿势，看到 libmsaoaidsec.so 的使用还是挺多的，比如某站、某书和某艺。既然有这么多样本，那就来分析一下检测逻辑，顺便感受一 ...

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)BubbleC

前言
--

最近在分析绕过 frida 检测的姿势，看到 libmsaoaidsec.so 的使用还是挺多的，比如某站、某书和某艺。

既然有这么多样本，那就来分析一下检测逻辑，顺便感受一下不同版本之间该. so 有哪些变化。  
**注意！**

**样本一是比较早的经典版本，想看较新的绕过思路可直接看样本二。**  
【环境】：Redmi(android 8)、v8a、Frida16

### 样本一

【某 app v7.26.1 版本】

先定位 so，`hook_dlopen`可见加载`libmsaoaidsec.so`后，frida 进程就被杀掉了，因此监测点在`libmsaoaidsec.so`中。

```
objdump -D [linker64] | grep dlopen
```

接下来要定位函数

ida 打开`libmsaoaidsec.so`找到 JNI_Onload 函数的偏移地址，尝试`hook JNI_Onload`函数，如果能成功 hook 将打印日志 “call JNI_Onload”，没有就说明 frida 检测在`hook JNI_Onload`前。

运行后没有打印日志，检测在 init_xxx。

> 需要 hook .init_xxx 的函数，但这里有一个问题，dlopen 函数调用完成之后. init_xxx 函数已经执行完成了，这个时候不容易使用 frida 进行 hook，这个问题其实很麻烦的，因为想要 hook linker 的 call_function 并不容易，这里面涉及到 linker 的自举。

这里学到的思路是在`.init_proc`函数中找一个调用了外部函数的位置，时机越早越好。

导出表搜索只搜索到`.init_proc`

f5 看到. init_proc 被混淆了，很明显的虚假控制流。不过这里在最开始调用了一个外部函数 sub_B1B4，这个时候也就是. init_proc 函数刚刚调用的时候，在这个时机点是个注入的好时机。  
[img=570,525][https://s21.ax1x.com/2025/02/22/pElACFK.png[/img](https://s21.ax1x.com/2025/02/22/pElACFK.png[/img)]

```
objdump -D [linker64] | grep call_constructors
```

在获取了一个非常早的注入时机之后，就可以定位具体的 frida 检测点了。

这里对`pthread_create`函数进行 hook，可以看到这里有两个线程地址和其他的不一样，说明这两个线程是`libmsaoaidsec.so`创建的。

[https://s21.ax1x.com/2025/02/22/pElkqWF.png](https://s21.ax1x.com/2025/02/22/pElkqWF.png)

这里需要计算一下偏移地址：

```
0xb54d4129 - 0xb54c3000 = 0x11129
0xb54d3975 - 0xb54c3000 = 0x10975
```

0x11129 和 0x10975 是线程需要执行的函数的地址，也就是 pthread_create 函数的第三个参数。

而我们需要通过交叉引用找到 pthread_create 函数被调用的地址。

G 键搜索`11129`找到`sub_11128`，然后通过 x 键交叉引用找到目标函数`sub_113E0`，同理，搜索`10975`找到`sub_10974`, 然后交叉引用找到目标函数`sub_109A8`。

然后直接把`pthread_create`函数`nop`掉即可。

### 样本二

#### v8.31.0

【另一款也使用 libmsaoaidsec.so 的 app】

这里用上面定位 frida 检测的 hook 脚本直接定位到`libmsaoaidsec.so`，

这里为了避免遇到字符串加密，而是将注意力放在分析 frida 检测逻辑上，就直接从内存中 dump 出`libmsaoaidsec.so`然后修复得到`libmsaoaidesc_fixed.so`

ida 打开，由于前面的分析可以直接定位到`.init_proc`，反编译看到有很多 while 一看便知是虚假控制流，然后发现代码与样本一的代码有些细微的变化，比如此处就没有之前较明显的外部函数可以 hook 了。那我们就来分析下具体逻辑。

这里直接使用 [D810](https://gitlab.com/eshard/d810) 去除混淆，然后来分析代码逻辑看看做了哪些检测的操作。

```
void init_proc()
{
  const char *v0; // x23
  __int64 v1; // x0
  __int64 v2; // x0
  unsigned __int64 StatusReg; // [xsp+0h] [xbp-870h] BYREF
  __int64 *v4; // [xsp+8h] [xbp-868h]
  int v6; // [xsp+18h] [xbp-858h]
  int v7; // [xsp+1Ch] [xbp-854h]
  __int64 *v8; // [xsp+20h] [xbp-850h]
  char *v10; // [xsp+30h] [xbp-840h]
  FILE *v11; // [xsp+38h] [xbp-838h]
  char v12[2000]; // [xsp+40h] [xbp-830h] BYREF
  __int64 v13; // [xsp+810h] [xbp-60h]

  StatusReg = _ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2));// 获取寄存器状态
  v13 = *(_QWORD *)(StatusReg + 40);
  v4 = (__int64 *)(&StatusReg - 250);
  *(_DWORD *)off_47FB8 = sub_123F0();           // 获取SDK版本
  sub_12550();                                  // 判断art/devm
  sub_12440();                                  // 获取版本信息
  if ( *(_DWORD *)off_47FB8 > 23 )
    *(_BYTE *)off_47ED8 = 1;
  if ( (sub_25A48() & 1) == 0 )
  {
    v10 = v12;
    memset(v10, 0, 0x7D0u);
    getpid();
    _cxa_finalize(v12);
    v11 = fopen(v12, "r");
    if ( v11 )
    {
      v8 = v4;
      memset(v4, 0, 0x7D0u);
      v0 = (const char *)v4;
      strdup((const char *)v11);
      fclose(v11);
      if ( !strchr(v0, 58) )
        sub_1BEC4();                            // anti-frida
    }
    v1 = sub_13728();
    sub_23AD4(v1);
    v6 = sub_C830();
    if ( v6 != 1 || (v7 = sub_95C8()) != 0 )
      sub_9150();
  }
  if ( *(_QWORD *)(StatusReg + 40) != v13 )
  {
    v2 = _strlcpy_chk();                        // check标志位
    sub_148A0(v2);
  }
}
```

可以定位到关键函数 sub_1BEC4，来分析一下：

```
__int64 sub_1BEC4()
{
  unsigned __int64 StatusReg; // x19
  __int64 v1; // x0
  __int64 v3; // x0
  __int64 v4; // [xsp+8h] [xbp-48h]

  StatusReg = _ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2));
  v4 = *(_QWORD *)(StatusReg + 40);
  HIDWORD(qword_49010) = getpid();
  v1 = sub_1B924();                             // check2
  if ( *(_QWORD *)(StatusReg + 40) == v4 )
    return 0LL;
  v3 = _strlcpy_chk(v1);
  return sub_1BFAC(v3);                         // check1 检测/proc/pid/tast下线程：gum-js-loop和gmain
}
```

**这里介绍下 task 和 fd 目录下的具体检测特征。**

**检测 / proc/pid/tast 下线程**  
/proc/pid/task 目录下可以查看不同线程的子目录，获取每个线程运行时信息，这些信息包括线程的状态、线程的寄存器内容、线程占用的 CPU 时间、线程的堆栈信息等。

开启 frida 调试后这个 task 目录下会多出几个线程，检测点在查看这些多出来的线程是否和 frida 调试相关。        

`gmain`：Frida 使用 Glib 库，其中的主事件循环被称为 GMainLoop。在 Frida 中，gmain 表示 GMainLoop 的线程。

`gdbus`：GDBus 是 Glib 提供的一个用于 D-Bus 通信的库。在 Frida 中，gdbus 表示 GDBus 相关的线程。

`gum-js-loop`：Gum 是 Frida 的运行时引擎，用于执行注入的 JavaScript 代码。gum-js-loop 表示 Gum 引擎执行 JavaScript 代码的线程。

`pool-frida`：Frida 中的某些功能可能会使用线程池来处理任务，pool-frida 表示 Frida 中的线程池。

**/proc/pid/fd**

该目录的作用在于提供了一种方便的方式来查看进程的文件描述符信息，通过查看文件描述符信息，可以了解进程打开了哪些文件、网络连接等。

`linjector`是一种用于 Android 设备的开源工具，它允许用户在运行时向 Android 应用程序注入动态链接库（DLL）文件。通过注入 DLL 文件，用户可以修改应用程序的行为、调试应用程序、监视函数调用等，这在逆向工程、安全研究和动态分析中是非常有用的。

##### **chek2：sub_1B924->sub_1CEF8->sub_1C544**

chek1 比较简单一眼就看到，下面详细介绍 check2。

sub_1B924 中一眼看到定位到`_strlcpy_chk()`：

```
result = (__int64)dlopen(v19, 2);
  v21 = (char *)result;
  if ( result )
  {
    v24 = atoi(v21);
    v25 = (void (__fastcall *)(__int64 *, _QWORD, void *, _QWORD))v24;
    if ( (unsigned int)sub_CAA8() == 248 && (sub_12D9C(0LL) & 1) == 0 )
      sub_1CEF8(v25);
    if ( (unsigned int)sub_CAE8() == 249 )
      v25(&qword_49650, 0LL, &loc_1B8D4, 0LL);
    else
      sub_1B380(v21, v25);
    v23 = sub_CA28();
    if ( v23 == 167 )
      v25(&qword_49658, 0LL, &loc_19E0C, 0LL);
    result = dlclose(v21);
  }
  if ( *(_QWORD *)(StatusReg + 40) != v28 )
  {
    _strlcpy_chk(result);
    return sub_1BEC4();
  }
```

交叉引用定位到关键变量 v25，进而定位到 sub_1CEF8。

sub_1CEF8 一开头就看到加载了 “libart.so”，然后一堆异或和取余操作，应该是在加解密（这里就不还原算法了），根据经验合理猜测这是通过 libc.so 的 pthread_create 创建线程实例来 anti-frida。

继续往下分析发现关键代码块为 LABEL__35：

```
LABEL_35:
  if ( (sub_25B30(v15) & 1) != 0 )
    sub_234E0(0LL);
  result = a1(&v42, 0LL, sub_1C544, v15);
  if ( *(_QWORD *)(StatusReg + 40) != v44 )
  {
    v41 = _strlcpy_chk(result);
    return std::_Rb_tree<std::string,std::string,std::_Identity<std::string>,std::less<std::string>,std::allocator<std::string>>::_M_erase(v41);
  }
```

看到老朋友`_strlcpy_chk`同理，定位到 sub_1C544，一路分析定位到 LABEL_81，最终的检测写在这个 while 循环里：

```
                                      while ( 1 )
                                      {
LABEL_81:
                                        v103 = sub_1BFAC();// check1,gum-js-loop和gmain
                                        v104 = sub_1C158(v103);// chek3,检测linjector
                                        sub_1C26C(v104);// check4,检测/data/local/tmp和frida-agent
                                        sub_26334(a1);// check5,mmap
                                        sleep(4u);
                                      }
                                    }
                                  }
```

**分析结束，下面是 hook**  
这里插入关于 linker 的讨论  
我们已经知道`dlopen(android_dlopen_ext)`将动态库加载到内存中，加载库后，系统会处理库中的依赖关系，并在库的地址空间中映射所有符号。

然后执行`.init_array` 和 `.init_proc` 段。

ELF 文件中 `.init_proc` 段的地址存储在 `.init_array` 表中。

在加载库后，系统会调用 `call_constructors` 来执行 `.init_array` 中的所有初始化函数，`.init_proc` 中的代码一般由开发者定义，它可能会调用一些重要的逻辑函数，比如初始化全局变量，设置反调试、密钥初始化等。

由此可知， 如果库中有重要的保护逻辑或初始化算法，它们通常会出现在 `.init_proc` 或由 `.init_proc` 调用的函数中。  

这里有个关键的地方在于， `call_constructors`的执行过程，即：linker64 中`dlopen通过do_dlopen->call_constructors->call_array`。

整个流程借用正己大佬给的流程图，更清晰：  
[https://s21.ax1x.com/2025/02/22/pElkze1.png](https://s21.ax1x.com/2025/02/22/pElkze1.png)  
所以我们从`call_constructors`入手。`sub_1BEC4` 是一个位于 `libmsaoaidsec.so` 中的关键函数，但在动态库完成加载前，它的地址并未映射到内存中。而`call_constructors` 是库完成加载后，首次执行初始化逻辑的地方，此时目标库的基地址已经确定，所有符号地址也已经可以解析。

使用 objdump 查看`android_dlopen_ext`和`call_constructors`相对 linker64 的偏移地址，然后 hook：

```
function hook_android_dlopen_ext() {
    var linker64_base_addr = Module.getBaseAddress("linker64")
    var android_dlopen_ext_func_off = 0x8f74
    var android_dlopen_ext_func_addr = linker64_base_addr.add(android_dlopen_ext_func_off)
    Interceptor.attach(android_dlopen_ext_func_addr, {
        onEnter: function (args) {
            if (args[0].readCString() != null && args[0].readCString().indexOf("libmsaoaidsec.so") >= 0) {
                hook_call_constructors()
            }
        },
        onLeave: function (ret) {

        }
    })
}

function hook_call_constructors() {
    var linker64_base_addr = Module.getBaseAddress("linker64")
    var call_constructors_func_off = 0x20b78
    var call_constructors_func_addr = linker64_base_addr.add(call_constructors_func_off)
    var listener = Interceptor.attach(call_constructors_func_addr, {
        onEnter: function (args) {
            var module = Process.findModuleByName("libmsaoaidsec.so")
            if (module != null) {
                Interceptor.replace(module.base.add(0x1BEC4), new NativeCallback(function () {
                    console.log("replace sub_1BEC4")
                }, "void", []))
                listener.detach()
            }
        },
    })
}

hook_android_dlopen_ext()
```

[https://s21.ax1x.com/2025/02/22/pElApo6.png](https://s21.ax1x.com/2025/02/22/pElApo6.png)  
成功绕过！

* * *

#### v8.63.0

关于样本二的最新版（24 年 11 月）我也分析了下，关键函数没变，具体检测的函数调用有点变化，不过一样可以 bypass＜（＾－＾）＞

```
__int64 sub_1BEC4()
{
  unsigned __int64 StatusReg; // x19
  __int64 v1; // x0
  __int64 v3; // x0
  __int64 v4; // [xsp+8h] [xbp-48h]

  StatusReg = _ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2));
  v4 = *(_QWORD *)(StatusReg + 40);
  HIDWORD(qword_49010) = getpid();
  v1 = sub_1B924();
  if ( *(_QWORD *)(StatusReg + 40) == v4 )
    return 0LL;
  v3 = _strlcpy_chk(v1);
  return sub_1BFAC(v3);
}
```

[img=545,286][https://s21.ax1x.com/2025/02/22/pElASdx.png[/img](https://s21.ax1x.com/2025/02/22/pElASdx.png[/img)]  
成功！

【参考】：

[[原创] 绕过 bilibili frida 反调试 - Android 安全 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-277034.htm)

[某书 Frida 检测绕过记录_libmsaoaidsec.so-CSDN 博客](https://blog.csdn.net/weixin_45582916/article/details/137973006)

后记
--

第一次在吾爱发文有些小忐忑，笨人此前备受完美主义折磨，羞愧于自己没有写出有价值的文章，认为不值得分享。  
最近笨人的想法有些转变，决定大胆迈出第一步。欢迎大家留言交流！

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)BubbleC

> [快乐的小跳蛙 发表于 2025-2-25 13:38](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52466539&ptid=2008459)  
> 请问下，我在使用 frida hook 单例模式类构造内调用的 native 方法时，使用你文中提到的在 call_constructors 中 h ...

你是指 call_constructors 没被成功 hook 吗？需要注意的是这个代码中有两处偏移地址（①和②）需要根据你的实际环境做修改：  
![](https://s21.ax1x.com/2025/02/25/pE149GF.jpg)  
![](https://s21.ax1x.com/2025/02/25/pE14eIK.jpg)  
要获得①和②的地址需要你先找到自己手机上 linker64 的位置，然后  
[Shell] _纯文本查看_ _复制代码_

```
objdump -D [linker64] | grep dlopen
```

[Shell] _纯文本查看_ _复制代码_

```
objdump -D [linker64] | grep call_constructors
```

![](https://avatar.52pojie.cn/data/avatar/001/26/93/39_avatar_middle.jpg)快乐的小跳蛙 请问下，我在使用 frida hook 单例模式类构造内调用的 native 方法时，使用你文中提到的在 call_constructors 中 hook 目标函数偏移，确认函数调用过，onEnter 的日志没有输出是什么原因，如果需要 hook 代码和 apk 信息回复一下，谢谢![](https://avatar.52pojie.cn/data/avatar/001/26/93/39_avatar_middle.jpg)快乐的小跳蛙

> [BubbleC 发表于 2025-2-25 19:26](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52469026&ptid=2008459)  
> 你是指 call_constructors 没被成功 hook 吗？需要注意的是这个代码中有两处偏移地址（①和②）需要根据你的 ...

这些我都是对照我自己的手机更改过偏移的，我打包了一份用到 apk，so，js 文件。麻烦闲暇之余看下 hook_gen 函数的 console.log("enter hook_gen"); 能不能输出，在您的设备上。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)BubbleC

> [快乐的小跳蛙 发表于 2025-2-25 19:54](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52469175&ptid=2008459)  
> 这些我都是对照我自己的手机更改过偏移的，我打包了一份用到 apk，so，js 文件。麻烦闲暇之余看下 hook_gen ...

好的我看一下![](https://avatar.52pojie.cn/data/avatar/001/26/93/39_avatar_middle.jpg)快乐的小跳蛙

> [BubbleC 发表于 2025-2-25 20:56](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52469474&ptid=2008459)  
> 好的我看一下

[https://wwxe.lanzoub.com/iIwJh2otxz5e](https://wwxe.lanzoub.com/iIwJh2otxz5e) 压缩包在这里 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) smzll 楼主, 最新版的样本二, 在打印完 replace sub_1BEC4 之后, 手机就一直卡死在加载页面了进不去, 能麻烦看下嘛 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) smzll _ 本帖最后由 smzll 于 2025-2-26 13:37 编辑_  

> smzll 发表于 2025-2-26 12:13  
> 楼主, 最新版的样本二, 在打印完 replace sub_1BEC4 之后, 手机就一直卡死在加载页面了进不去, 能麻烦看下嘛

我的是 8.67 版本, 新版本也是一样的卡死进不去 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) BubbleC

> [smzll 发表于 2025-2-26 12:30](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52473431&ptid=2008459)  
> 我的是 8.67 版本, 新版本也是一样的卡死进不去

你好，样本二 v8.67.0 我试了下，是可以绕过的![](https://s21.ax1x.com/2025/02/26/pE3mzTJ.png) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) BubbleC

> [快乐的小跳蛙 发表于 2025-2-25 21:28](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52469611&ptid=2008459)  
> https://wwxe.lanzoub.com/iIwJh2otxz5e 压缩包在这里

代码还没来得及看，不过使用本贴的代码是可以成功绕过的  
![](https://s21.ax1x.com/2025/02/26/pE3MCfs.jpg)