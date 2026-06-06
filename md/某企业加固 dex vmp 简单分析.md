> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2111432-1-1.html)

> 前言本人小白一枚，想尝试学习一些加固，在身边朋友的推荐下选择了这个壳子. 恰巧最近论坛上也有很多大佬发布了自己的分析贴（比如乐子人大佬的帖子），只是复现下来后发 ...

![](https://avatar.52pojie.cn/data/avatar/002/00/19/38_avatar_middle.jpg)linglian _ 本帖最后由 linglian 于 2026-6-6 13:25 编辑_  
**前言**  
本人小白一枚，想尝试学习一些加固，在身边朋友的推荐下选择了这个壳子. 恰巧最近论坛上也有很多大佬发布了自己的分析贴（比如[乐子人大佬的帖子](https://bbs.kanxue.com/thread-291069.htm)），只是复现下来后发现后续的 dex vmp 相关帖子比较少（很不幸只找到了爱吃[菠菜大佬的这篇帖子](https://bbs.kanxue.com/thread-270799-1.htm)，不过因为年代久远甚至指令结构都不同了），于是写篇帖子记录一下.  
新人发帖，如有错误还望大佬指出.  
帖子使用的样本不方便说出来，不过是今年的.（因为据说这个样本保护项开得多）  
**Java 层特征**  
![](https://attach.52pojie.cn/forum/202606/06/103032jxguluk2ul22wxgw.png)

**图片 29.png** _(65.9 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjE4OHxjNDgzYjlmM3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:30 上传

  
被 vmp 保护的方法调用了 Lcom/fort/andjni/JniLib; 类的一个 native 方法  
![](https://attach.52pojie.cn/forum/202606/06/103255nzzr36vwrq3uumtt.png)

**图片 30.png** _(703.87 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjE4OXw5ODFlNTQ1N3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:32 上传

  
JniLib 类中有很多种不同的 native 函数，显然是通过不同的函数签名来处理不同返回值的跳板函数. 既然这个类加载了 libdexjni.so，那么这些 native 函数大概率就是被这个 so 注册的.  
**Native 层初探**  
通过 BinaryNinja 加载 so（这里把 so 基址设为了 0x12000000），尝试在函数表中搜索 Java_来定位，发现没有结果.  
扫了一眼导出表  
![](https://attach.52pojie.cn/forum/202606/06/103344xnwd8165iel6n1ed.png)

**图片 31.png** _(167.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIwOHxkNzhmYzIwOHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:33 上传

  
虽然有些明显的名称混淆，但 JNI_OnLoad 这些关键函数还在，导出表应该是没有被保护的.  
既然静态注册搜不到，那么这些 native 函数必然是动态注册的了.  
直接来到 JNI_OnLoad：  
![](https://attach.52pojie.cn/forum/202606/06/103417poc81nv516188y8m.png)

**图片 32.png** _(42.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIwOXxhZmU3NTU4YXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:34 上传

  
发现函数不可解析，应该是被保护了.  
往上查找. init.array：  
![](https://attach.52pojie.cn/forum/202606/06/103437r5iau6116eni11uh.png)

**图片 33.png** _(20.96 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIxMHxmMmU0ZWNkZnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:34 上传

  
![](https://attach.52pojie.cn/forum/202606/06/103503vp640me04vpy4g3h.png)

**图片 34.png** _(17.14 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIxMXxlMWYwM2JjYnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:35 上传

  
对应的地址完全是空的，那么只可能是在更早执行的. init 进行解密了.  
先从 dynamic 里找到 init 地址  
![](https://attach.52pojie.cn/forum/202606/06/103523xz5o0tpg0nt5to2o.png)

**图片 35.png** _(13.81 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIxMnwwN2QwMGJhNnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:35 上传

  
![](https://attach.52pojie.cn/forum/202606/06/103526b2540mjtivnejk0w.png)

**图片 36.png** _(41.08 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIxM3xhNDJiOTAzOXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:35 上传

  
sub_125617dc 分配了一块内存并复制了 arg2 的数据，数据大小被存储在了 arg1 + 0x28.  
而 sub_1256150c 主要是进行了数据解压（疑似还有其他操作，但我这个样本并没有执行到，就没去分析）：  
![](https://attach.52pojie.cn/forum/202606/06/103605p6ala3wgezgnl4lg.png)

**图片 37.png** _(71.37 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIxNXw3NmEyN2ExNnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:36 上传

  
解压算法是 nrv2d，以上数据均来自 init 函数最开始的参数 0x12561b48：  
![](https://attach.52pojie.cn/forum/202606/06/103625zo1m55yy3751zxmc.png)

**图片 38.png** _(39.79 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIxNnwyZGI0MThiM3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:36 上传

  
具体逻辑搞清楚了，修复就简单了. 可选择：  
1. 直接使用支持 nrv2d 的库对 0x120096d8 处大小 0x33dfd 的数据进行解压；  
2. 在 init 函数运行完后直接 dump 下 0x120096d8 处大小 0x7e500 的数据.  
得到解压数据后直接塞回 so 文件对应位置即可.  
这里需要补充一段，完成这步之后 BinaryNinja 已经可以正常解析函数了，但是后面在使用 ida 分析 so 时发现虽然导入表和导出表正确，但 ida 未能将符号对应的函数解析完整：  
![](https://attach.52pojie.cn/forum/202606/06/103706d4oxsx9cc49azjdj.png)

**图片 39.png** _(86.79 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIxN3wzOTA3YTcyMXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:37 上传

  
简单分析发现这是因为这个 so 的 section 被动了手脚：  
![](https://attach.52pojie.cn/forum/202606/06/103722v3ggoyvcogv1z9wl.png)

**图片 40.png** _(49.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIxOHwxMGE3ZjMxNXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:37 上传

  
节中将一个很大的区域标记成 SHT_NOBITS 扰乱了 ida 的分析，只需把 type 改成 SHT_NULL(0) 或者 SHT_PROGBITS(1) 即可使 ida 正常解析.  
**[虚拟机](https://www.52pojie.cn/thread-661779-1-1.html)入口分析**  
解压数据回填后，JNI_OnLoad 已经可以正常反编译了，映入眼帘的便是三个函数调用.  
![](https://attach.52pojie.cn/forum/202606/06/103757p6e6gubl3s8ps9j8.png)

**图片 41.png** _(62.26 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIyMHwzNGNhM2JkZHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:37 上传

  
挨个翻了一下，发现前两个函数都有很明显的控制流：  
![](https://attach.52pojie.cn/forum/202606/06/103823ztttfftzyty5tc6b.png)

**图片 42.png** _(170.49 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIyMXxhNDFjZWQyOHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:38 上传

  
而第三个函数看上去非常清晰，并且一眼就能看到很多关键数据（为了方便直观这里稍微处理了一下）：  
![](https://attach.52pojie.cn/forum/202606/06/103825pfu5052w425ww4je.png)

**图片 43.png** _(171.6 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIyMnw0MTJlMWY1OXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:38 上传

  
既然前两个函数不太好看，那么就先分析第三个函数好了.  
这个函数很简单，首先 sub_12010028 初始化了一些全局对象：  
![](https://attach.52pojie.cn/forum/202606/06/103855k9vyhp49p4zmvuhv.png)

**图片 44.png** _(133.63 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIyM3xkZDJhODBkZHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:38 上传

  
初始化完成后就注册了 JniLib 类的 native 函数：  
![](https://attach.52pojie.cn/forum/202606/06/103917rappnup4ubb8si2n.png)

**图片 45.png** _(25.97 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIyNHxhN2Y4ZTllOHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:39 上传

  
既然要找的函数已经在这了，不妨直接进去看看吧：  
![](https://attach.52pojie.cn/forum/202606/06/103949xj0ypdf1z1hqmjjm.png)

**图片 46.png** _(113.51 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjIyOXw1MmU4ZTUzOXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:39 上传

  
sub_12017adc 就是虚拟机的入口了，这里命名为 vm_entrance.  
由上图不难看出，这个函数的签名大概是 (JNIEnv* env, jobjectArray args, void* result)  
到了这里，先回顾一下函数的入参：  
jobjectArray 第 0 位是 jclass，若为实例方法则第 1 位是实例的 jobject，最后一位 jint 是 vm 函数的 idx.  
![](https://attach.52pojie.cn/forum/202606/06/104322tymw0p22o1e0wo38.png)

**图片 47.png** _(116.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI0OXwwMGEzODQyYnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:43 上传

  
入口函数先获取了 jobjectArray 最后一位的 Integer，然后调用了 intValue 函数读到 jint 传给 j__lSSO05lI$I5IllOl_0lS$_lllIII__l$IOlI55$O550IS__lS5$：  
![](https://attach.52pojie.cn/forum/202606/06/104345hzp7b1btss28csiz.png)

**图片 48.png** _(26.24 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1MHw0NWI0MDc3YnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:43 上传

  
而这个函数通过 vm_idx 去查了表 data_1255ff48，因此这个表里一定存储着所有 vm 方法的数据.  
交叉引用可以看到另一个函数对这个表进行了很多操作，应该就是初始化了：  
![](https://attach.52pojie.cn/forum/202606/06/104407qowgffcfp9ddvwcc.png)

**图片 49.png** _(67.83 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1MXxjNjM3OTZiMHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:44 上传

  
至于这个函数，我们在上面的流程里其实已经见过了，正是 JNI_OnLoad 调用的第一个函数.  
既然终究是躲不过的话，就只好顶着控制流简单看一下喽：  
![](https://attach.52pojie.cn/forum/202606/06/104433g769he086df0tahw.png)

**图片 50.png** _(139.32 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1MnxkNmMyM2ZhZnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:44 上传

  
这个表里的结构体大小是 0x28，通过一些函数读取数据，读取格式是 MessagePack：  
![](https://attach.52pojie.cn/forum/202606/06/104458ebx3zkwz05snkkzy.png)

**图片 51.png** _(115.38 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1M3w2N2I0NDQ3ZHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:44 上传

  
这里偷懒用 frida 在这个函数初始化完后去遍历表 data_1255ff48（这里只处理了 libDexHelper.so 的检测，libdexjni.so 应该是没有检测的），根据打印出的数据，大概猜测结构体结构：  
[C] _纯文本查看_ _复制代码_

```
struct method_info __packed
{
    uint32_t idx;
    uint32_t code_item_size;
    void* code_item;
    uint32_t reg_count;
    uint32_t ins;
    uint32_t outs;
    uint32_t is_static;//这个没去验证，纯经验猜测
    char* shorty;
};

```

* 注：这里的 code_item 是我先入为主瞎取的名字，并不是真正的标准 code_item，因为这个 vmp 的字节码很像 code_item 的结构，具体长这样：  
![](https://attach.52pojie.cn/forum/202606/06/104756djf0lz2ke2a25e50.png)

**图片 52.png** _(108.3 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1NHxlYzJkZTc4M3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:47 上传

  
而 code_item_size 则是这段字节码数据的 size.  
后面会对其进行详细分析，这个命名我感觉还是比较贴合的，就是有点容易混淆，因此提一嘴.  
重新回到 vm_entrance 函数，可以看到在栈上重新排了一个结构体：  
![](https://attach.52pojie.cn/forum/202606/06/104838f2ftv5v7wiwovzsz.png)

**图片 53.png** _(95.66 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1NXxhYTlhMmVmZHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:48 上传

  
![](https://attach.52pojie.cn/forum/202606/06/104840jgbmmnsn9sozzxyo.png)

**图片 54.png** _(28.26 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1Nnw3OWZkZTk4MHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:48 上传

  
最终传入了函数 sub_12018544.  
sub_12018544 大概扫了眼，应该是整理 vm 方法的参数，给包装类拆箱：  
![](https://attach.52pojie.cn/forum/202606/06/104913tx88zvsnc1hbt1n7.png)

**图片 55.png** _(81.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1N3w1Y2RiNzZjZnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:49 上传

  
这里创建了一个大小 0x48 的结构体，与寄存器有关，偏移 0x8 处的列表用于存储寄存器  
![](https://attach.52pojie.cn/forum/202606/06/104951tvuyzfyi5m1yy5f1.png)

**图片 56.png** _(65.76 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1OHwxZjgzZDM1ZHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:49 上传

  
![](https://attach.52pojie.cn/forum/202606/06/104955jxs3p8gx33h3sk8s.png)

**图片 57.png** _(25.57 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI1OXwzYzhjMWNkZXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:49 上传

  
最终来到函数 j__lIl_0ll5OIlIlISl00lI_lI5lIll$I$IIlIl00I$lIIIl5S0S5$  
![](https://attach.52pojie.cn/forum/202606/06/105023ddovzz09tj6ys6tv.png)

**图片 58.png** _(46.16 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI2MHxiMzlkNzVjN3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:50 上传

  
跳到这个函数后 binaryninja 就有点不好使了，加载了半天也没加载好，控制流混淆得很严重：  
![](https://attach.52pojie.cn/forum/202606/06/105043q0x05c576w0q074s.png)

**图片 59.png** _(98.05 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI2MXxlODlkYmYwMHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:50 上传

  
并且函数体量奇大，根据以上种种，不难猜到这个就是解释器函数了.  
面对这一望无际的工作量，死磕明显不太现实.  
尝试过写反混淆插件，但只能处理些简单的，效果不尽人意，若要兼容更多情况，插件工程量也相当大.  
反混淆无果后，决定依靠 trace. 试了下 frida trace 感觉不太稳定，经常会闪退，最后选择了 unidbg trace.  
**虚拟机解释器分析**  
简单看了一下，这个 so 的依赖相当少，甚至没有对 libDexHepler.so 的依赖：  
![](https://attach.52pojie.cn/forum/202606/06/105210apm56i0cyllkryg5.png)

**图片 60.png** _(11.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI2M3xiNTQxYTE2YXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:52 上传

  
因此 unidbg 应该不难运行起来，事实也的确如此：  
[Plain Text] _纯文本查看_ _复制代码_

```
JNIEnv->FindClass([Z) was called from RWX@0x120170e4[libdexjni.so]0x170e4
JNIEnv->NewGlobalRef(class [Z) was called from RWX@0x120170f8[libdexjni.so]0x170f8
JNIEnv->FindClass([B) was called from RWX@0x12014e44[libdexjni.so]0x14e44
JNIEnv->NewGlobalRef(class [B) was called from RWX@0x12014e58[libdexjni.so]0x14e58
JNIEnv->FindClass([S) was called from RWX@0x1201503c[libdexjni.so]0x1503c
JNIEnv->NewGlobalRef(class [S) was called from RWX@0x12015050[libdexjni.so]0x15050
JNIEnv->FindClass([C) was called from RWX@0x12014000[libdexjni.so]0x14000
JNIEnv->NewGlobalRef(class [C) was called from RWX@0x12014014[libdexjni.so]0x14014
JNIEnv->FindClass([I) was called from RWX@0x12010320[libdexjni.so]0x10320
JNIEnv->NewGlobalRef(class [I) was called from RWX@0x12010334[libdexjni.so]0x10334
JNIEnv->FindClass([J) was called from RWX@0x12010628[libdexjni.so]0x10628
JNIEnv->NewGlobalRef(class [J) was called from RWX@0x1201063c[libdexjni.so]0x1063c
JNIEnv->FindClass([F) was called from RWX@0x12010494[libdexjni.so]0x10494
JNIEnv->NewGlobalRef(class [F) was called from RWX@0x120104a8[libdexjni.so]0x104a8
JNIEnv->FindClass([D) was called from RWX@0x120112b4[libdexjni.so]0x112b4
JNIEnv->NewGlobalRef(class [D) was called from RWX@0x120112c8[libdexjni.so]0x112c8
JNIEnv->FindClass(java/lang/Class) was called from RWX@0x120112f8[libdexjni.so]0x112f8
JNIEnv->NewGlobalRef(class java/lang/Class) was called from RWX@0x1201130c[libdexjni.so]0x1130c
JNIEnv->FindClass(java/lang/Object) was called from RWX@0x12010574[libdexjni.so]0x10574
JNIEnv->NewGlobalRef(class java/lang/Object) was called from RWX@0x12010588[libdexjni.so]0x10588
JNIEnv->FindClass(java/lang/reflect/Field) was called from RWX@0x120105e4[libdexjni.so]0x105e4
JNIEnv->NewGlobalRef(class java/lang/reflect/Field) was called from RWX@0x120105f8[libdexjni.so]0x105f8
JNIEnv->FindClass(java/lang/String) was called from RWX@0x12015fb8[libdexjni.so]0x15fb8
JNIEnv->NewGlobalRef(class java/lang/String) was called from RWX@0x12015fcc[libdexjni.so]0x15fcc
JNIEnv->FindClass(java/lang/reflect/Proxy) was called from RWX@0x12015d14[libdexjni.so]0x15d14
JNIEnv->NewGlobalRef(class java/lang/reflect/Proxy) was called from RWX@0x12015d28[libdexjni.so]0x15d28
JNIEnv->FindClass(java/lang/reflect/AccessibleObject) was called from RWX@0x120100d4[libdexjni.so]0x100d4
JNIEnv->NewGlobalRef(class java/lang/reflect/AccessibleObject) was called from RWX@0x120100e8[libdexjni.so]0x100e8
JNIEnv->FindClass(com/fort/andjni/JniLib) was called from RWX@0x12010734[libdexjni.so]0x10734
JNIEnv->NewGlobalRef(class com/fort/andjni/JniLib) was called from RWX@0x12010748[libdexjni.so]0x10748
JNIEnv->FindClass(java/lang/Boolean) was called from RWX@0x12014f24[libdexjni.so]0x14f24
JNIEnv->NewGlobalRef(class java/lang/Boolean) was called from RWX@0x12014f38[libdexjni.so]0x14f38
JNIEnv->FindClass(java/lang/Byte) was called from RWX@0x12014ea0[libdexjni.so]0x14ea0
JNIEnv->NewGlobalRef(class java/lang/Byte) was called from RWX@0x12014eb4[libdexjni.so]0x14eb4
JNIEnv->FindClass(java/lang/Character) was called from RWX@0x12014dd0[libdexjni.so]0x14dd0
JNIEnv->NewGlobalRef(class java/lang/Character) was called from RWX@0x12014de4[libdexjni.so]0x14de4
JNIEnv->FindClass(java/lang/Short) was called from RWX@0x1201428c[libdexjni.so]0x1428c
JNIEnv->NewGlobalRef(class java/lang/Short) was called from RWX@0x120142a0[libdexjni.so]0x142a0
JNIEnv->FindClass(java/lang/Integer) was called from RWX@0x120106a8[libdexjni.so]0x106a8
JNIEnv->NewGlobalRef(class java/lang/Integer) was called from RWX@0x120106bc[libdexjni.so]0x106bc
JNIEnv->FindClass(java/lang/Long) was called from RWX@0x120107c8[libdexjni.so]0x107c8
JNIEnv->NewGlobalRef(class java/lang/Long) was called from RWX@0x120107dc[libdexjni.so]0x107dc
JNIEnv->FindClass(java/lang/Float) was called from RWX@0x12010364[libdexjni.so]0x10364
JNIEnv->NewGlobalRef(class java/lang/Float) was called from RWX@0x12010378[libdexjni.so]0x10378
JNIEnv->FindClass(java/lang/Double) was called from RWX@0x12015f4c[libdexjni.so]0x15f4c
JNIEnv->NewGlobalRef(class java/lang/Double) was called from RWX@0x12015f60[libdexjni.so]0x15f60
JNIEnv->FindClass(java/lang/AbstractMethodError) was called from RWX@0x1201357c[libdexjni.so]0x1357c
JNIEnv->NewGlobalRef(class java/lang/AbstractMethodError) was called from RWX@0x12013590[libdexjni.so]0x13590
JNIEnv->FindClass(java/lang/ArithmeticException) was called from RWX@0x120170a0[libdexjni.so]0x170a0
JNIEnv->NewGlobalRef(class java/lang/ArithmeticException) was called from RWX@0x120170b4[libdexjni.so]0x170b4
JNIEnv->FindClass(java/lang/ArrayIndexOutOfBoundsException) was called from RWX@0x120171c8[libdexjni.so]0x171c8
JNIEnv->NewGlobalRef(class java/lang/ArrayIndexOutOfBoundsException) was called from RWX@0x120171dc[libdexjni.so]0x171dc
JNIEnv->FindClass(java/lang/ArrayStoreException) was called from RWX@0x12017128[libdexjni.so]0x17128
JNIEnv->NewGlobalRef(class java/lang/ArrayStoreException) was called from RWX@0x1201713c[libdexjni.so]0x1713c
JNIEnv->FindClass(java/lang/ClassCastException) was called from RWX@0x12015d88[libdexjni.so]0x15d88
JNIEnv->NewGlobalRef(class java/lang/ClassCastException) was called from RWX@0x12015d9c[libdexjni.so]0x15d9c
JNIEnv->FindClass(java/lang/ClassCircularityError) was called from RWX@0x120101e0[libdexjni.so]0x101e0
JNIEnv->NewGlobalRef(class java/lang/ClassCircularityError) was called from RWX@0x120101f4[libdexjni.so]0x101f4
JNIEnv->FindClass(java/lang/ClassNotFoundException) was called from RWX@0x12011360[libdexjni.so]0x11360
JNIEnv->NewGlobalRef(class java/lang/ClassNotFoundException) was called from RWX@0x12011374[libdexjni.so]0x11374
JNIEnv->FindClass(java/lang/ClassFormatError) was called from RWX@0x1201126c[libdexjni.so]0x1126c
JNIEnv->NewGlobalRef(class java/lang/ClassFormatError) was called from RWX@0x12011280[libdexjni.so]0x11280
JNIEnv->FindClass(java/lang/Error) was called from RWX@0x12015ddc[libdexjni.so]0x15ddc
JNIEnv->NewGlobalRef(class java/lang/Error) was called from RWX@0x12015df0[libdexjni.so]0x15df0
JNIEnv->FindClass(java/lang/ExceptionInInitializerError) was called from RWX@0x12014d0c[libdexjni.so]0x14d0c
JNIEnv->NewGlobalRef(class java/lang/ExceptionInInitializerError) was called from RWX@0x12014d20[libdexjni.so]0x14d20
JNIEnv->FindClass(java/io/FileNotFoundException) was called from RWX@0x12017184[libdexjni.so]0x17184
JNIEnv->NewGlobalRef(class java/io/FileNotFoundException) was called from RWX@0x12017198[libdexjni.so]0x17198
JNIEnv->FindClass(java/io/IOException) was called from RWX@0x120108d8[libdexjni.so]0x108d8
JNIEnv->NewGlobalRef(class java/io/IOException) was called from RWX@0x120108ec[libdexjni.so]0x108ec
JNIEnv->FindClass(java/lang/IllegalAccessError) was called from RWX@0x12015e78[libdexjni.so]0x15e78
JNIEnv->NewGlobalRef(class java/lang/IllegalAccessError) was called from RWX@0x12015e8c[libdexjni.so]0x15e8c
JNIEnv->FindClass(java/lang/IllegalAccessException) was called from RWX@0x12013520[libdexjni.so]0x13520
JNIEnv->NewGlobalRef(class java/lang/IllegalAccessException) was called from RWX@0x12013534[libdexjni.so]0x13534
JNIEnv->FindClass(java/lang/IllegalArgumentException) was called from RWX@0x12013fa4[libdexjni.so]0x13fa4
JNIEnv->NewGlobalRef(class java/lang/IllegalArgumentException) was called from RWX@0x12013fb8[libdexjni.so]0x13fb8
JNIEnv->FindClass(java/lang/IllegalMonitorStateException) was called from RWX@0x120102bc[libdexjni.so]0x102bc
JNIEnv->NewGlobalRef(class java/lang/IllegalMonitorStateException) was called from RWX@0x120102d0[libdexjni.so]0x102d0
JNIEnv->FindClass(java/lang/IllegalStateException) was called from RWX@0x1201414c[libdexjni.so]0x1414c
JNIEnv->NewGlobalRef(class java/lang/IllegalStateException) was called from RWX@0x12014160[libdexjni.so]0x14160
JNIEnv->FindClass(java/lang/IllegalThreadStateException) was called from RWX@0x12014f68[libdexjni.so]0x14f68
JNIEnv->NewGlobalRef(class java/lang/IllegalThreadStateException) was called from RWX@0x12014f7c[libdexjni.so]0x14f7c
JNIEnv->FindClass(java/lang/IncompatibleClassChangeError) was called from RWX@0x120140c4[libdexjni.so]0x140c4
JNIEnv->NewGlobalRef(class java/lang/IncompatibleClassChangeError) was called from RWX@0x120140d8[libdexjni.so]0x140d8
JNIEnv->FindClass(java/lang/InstantiationError) was called from RWX@0x12010230[libdexjni.so]0x10230
JNIEnv->NewGlobalRef(class java/lang/InstantiationError) was called from RWX@0x12010244[libdexjni.so]0x10244
JNIEnv->FindClass(java/lang/InstantiationException) was called from RWX@0x12014044[libdexjni.so]0x14044
JNIEnv->NewGlobalRef(class java/lang/InstantiationException) was called from RWX@0x12014058[libdexjni.so]0x14058
JNIEnv->FindClass(java/lang/InternalError) was called from RWX@0x1201433c[libdexjni.so]0x1433c
JNIEnv->NewGlobalRef(class java/lang/InternalError) was called from RWX@0x12014350[libdexjni.so]0x14350
JNIEnv->FindClass(java/lang/InterruptedException) was called from RWX@0x120104e8[libdexjni.so]0x104e8
JNIEnv->NewGlobalRef(class java/lang/InterruptedException) was called from RWX@0x120104fc[libdexjni.so]0x104fc
JNIEnv->FindClass(java/lang/LinkageError) was called from RWX@0x12010868[libdexjni.so]0x10868
JNIEnv->NewGlobalRef(class java/lang/LinkageError) was called from RWX@0x1201087c[libdexjni.so]0x1087c
JNIEnv->FindClass(java/lang/NegativeArraySizeException) was called from RWX@0x12010920[libdexjni.so]0x10920
JNIEnv->NewGlobalRef(class java/lang/NegativeArraySizeException) was called from RWX@0x12010934[libdexjni.so]0x10934
JNIEnv->FindClass(java/lang/NoClassDefFoundError) was called from RWX@0x12010530[libdexjni.so]0x10530
JNIEnv->NewGlobalRef(class java/lang/NoClassDefFoundError) was called from RWX@0x12010544[libdexjni.so]0x10544
JNIEnv->FindClass(java/lang/NoSuchFieldError) was called from RWX@0x12014d74[libdexjni.so]0x14d74
JNIEnv->NewGlobalRef(class java/lang/NoSuchFieldError) was called from RWX@0x12014d88[libdexjni.so]0x14d88
JNIEnv->FindClass(java/lang/NoSuchFieldException) was called from RWX@0x120103e4[libdexjni.so]0x103e4
JNIEnv->NewGlobalRef(class java/lang/NoSuchFieldException) was called from RWX@0x120103f8[libdexjni.so]0x103f8
JNIEnv->FindClass(java/lang/NoSuchMethodError) was called from RWX@0x12014fac[libdexjni.so]0x14fac
JNIEnv->NewGlobalRef(class java/lang/NoSuchMethodError) was called from RWX@0x12014fc0[libdexjni.so]0x14fc0
JNIEnv->FindClass(java/lang/NullPointerException) was called from RWX@0x1201080c[libdexjni.so]0x1080c
JNIEnv->NewGlobalRef(class java/lang/NullPointerException) was called from RWX@0x12010820[libdexjni.so]0x10820
JNIEnv->FindClass(java/lang/OutOfMemoryError) was called from RWX@0x12013604[libdexjni.so]0x13604
JNIEnv->NewGlobalRef(class java/lang/OutOfMemoryError) was called from RWX@0x12013618[libdexjni.so]0x13618
JNIEnv->FindClass(java/lang/RuntimeException) was called from RWX@0x120142d0[libdexjni.so]0x142d0
JNIEnv->NewGlobalRef(class java/lang/RuntimeException) was called from RWX@0x120142e4[libdexjni.so]0x142e4
JNIEnv->FindClass(java/lang/StackOverflowError) was called from RWX@0x1201419c[libdexjni.so]0x1419c
JNIEnv->NewGlobalRef(class java/lang/StackOverflowError) was called from RWX@0x120141b0[libdexjni.so]0x141b0
JNIEnv->GetMethodID(java/lang/Class.getField(Ljava/lang/String;)Ljava/lang/reflect/Field;) => 0xd585d5a3 was called from RWX@0x12015164[libdexjni.so]0x15164
JNIEnv->GetMethodID(java/lang/Class.getDeclaredField(Ljava/lang/String;)Ljava/lang/reflect/Field;) => 0x987c467d was called from RWX@0x120152c4[libdexjni.so]0x152c4
JNIEnv->GetMethodID(java/lang/reflect/Field.get(Ljava/lang/Object;)Ljava/lang/Object;) => 0x7abc2552 was called from RWX@0x120155ac[libdexjni.so]0x155ac
JNIEnv->GetMethodID(java/lang/reflect/Field.getBoolean(Ljava/lang/Object;)Z) => 0xfae67c5e was called from RWX@0x12015828[libdexjni.so]0x15828
JNIEnv->GetMethodID(java/lang/reflect/Field.getByte(Ljava/lang/Object;)B) => 0x35776224 was called from RWX@0x12015a80[libdexjni.so]0x15a80
JNIEnv->GetMethodID(java/lang/reflect/Field.getShort(Ljava/lang/Object;)S) => 0xaf19c8c3 was called from RWX@0x12015ce0[libdexjni.so]0x15ce0
JNIEnv->GetMethodID(java/lang/reflect/Field.getChar(Ljava/lang/Object;)C) => 0x50a94c97 was called from RWX@0x12012668[libdexjni.so]0x12668
JNIEnv->GetMethodID(java/lang/reflect/Field.getInt(Ljava/lang/Object;)I) => 0x82f4a366 was called from RWX@0x12012890[libdexjni.so]0x12890
JNIEnv->GetMethodID(java/lang/reflect/Field.getLong(Ljava/lang/Object;)J) => 0xd72cad38 was called from RWX@0x12012ae4[libdexjni.so]0x12ae4
JNIEnv->GetMethodID(java/lang/reflect/Field.getFloat(Ljava/lang/Object;)F) => 0xba8696d6 was called from RWX@0x12012d44[libdexjni.so]0x12d44
JNIEnv->GetMethodID(java/lang/reflect/Field.getDouble(Ljava/lang/Object;)D) => 0xfe53f3d was called from RWX@0x12012fb0[libdexjni.so]0x12fb0
JNIEnv->GetMethodID(java/lang/reflect/Field.set(Ljava/lang/Object;Ljava/lang/Object;)V) => 0xf25f2330 was called from RWX@0x12013270[libdexjni.so]0x13270
JNIEnv->GetMethodID(java/lang/reflect/Field.setBoolean(Ljava/lang/Object;Z)V) => 0xc42cc02e was called from RWX@0x120134f8[libdexjni.so]0x134f8
JNIEnv->GetMethodID(java/lang/reflect/Field.setChar(Ljava/lang/Object;C)V) => 0xb43f5b1f was called from RWX@0x12016270[libdexjni.so]0x16270
JNIEnv->GetMethodID(java/lang/reflect/Field.setByte(Ljava/lang/Object;B)V) => 0x6933f390 was called from RWX@0x120164b8[libdexjni.so]0x164b8
JNIEnv->GetMethodID(java/lang/reflect/Field.setShort(Ljava/lang/Object;S)V) => 0x3c37f3fb was called from RWX@0x12016710[libdexjni.so]0x16710
JNIEnv->GetMethodID(java/lang/reflect/Field.setInt(Ljava/lang/Object;I)V) => 0x256c9664 was called from RWX@0x12016958[libdexjni.so]0x16958
JNIEnv->GetMethodID(java/lang/reflect/Field.setFloat(Ljava/lang/Object;F)V) => 0x9e64b90e was called from RWX@0x12016b9c[libdexjni.so]0x16b9c
JNIEnv->GetMethodID(java/lang/reflect/Field.setLong(Ljava/lang/Object;J)V) => 0xfe28280c was called from RWX@0x12016e10[libdexjni.so]0x16e10
JNIEnv->GetMethodID(java/lang/reflect/Field.setDouble(Ljava/lang/Object;D)V) => 0xe7ac6ddb was called from RWX@0x12017064[libdexjni.so]0x17064
JNIEnv->GetMethodID(java/lang/reflect/Field.setAccessible(Z)V) => 0xcb19eff2 was called from RWX@0x12014440[libdexjni.so]0x14440
JNIEnv->GetMethodID(java/lang/reflect/AccessibleObject.setAccessible(Z)V) => 0xc15280ab was called from RWX@0x120144f0[libdexjni.so]0x144f0
JNIEnv->GetMethodID(java/lang/String.([B)V) => 0xd5642694 was called from RWX@0x1201464c[libdexjni.so]0x1464c
JNIEnv->GetMethodID(java/lang/String.intern()Ljava/lang/String;) => 0x38d73d3 was called from RWX@0x12014714[libdexjni.so]0x14714
JNIEnv->GetMethodID(java/lang/Object.getClass()Ljava/lang/Class;) => 0xb529717c was called from RWX@0x1201495c[libdexjni.so]0x1495c
JNIEnv->GetMethodID(java/lang/Class.getName()Ljava/lang/String;) => 0x4a974877 was called from RWX@0x12014a40[libdexjni.so]0x14a40
JNIEnv->GetStaticMethodID(java/lang/reflect/Proxy.isProxyClass(Ljava/lang/Class;)Z) => 0x7698fdb9 was called from RWX@0x12014cc0[libdexjni.so]0x14cc0
JNIEnv->GetStaticMethodID(com/fort/andjni/JniLib.InvokeObject([Ljava/lang/Object;)Ljava/lang/Object;) => 0xac6fba43 was called from RWX@0x12011700[libdexjni.so]0x11700
JNIEnv->GetStaticMethodID(com/fort/andjni/JniLib.getAllSuperClassesAndInterfaces(Ljava/lang/Class;)[Ljava/lang/Class;) => 0x2c346997 was called from RWX@0x12011b04[libdexjni.so]0x11b04
JNIEnv->GetStaticMethodID(java/lang/Boolean.valueOf(Z)Ljava/lang/Boolean;) => 0x1d8c249f was called from RWX@0x12011cc0[libdexjni.so]0x11cc0
JNIEnv->GetStaticMethodID(java/lang/Byte.valueOf(B)Ljava/lang/Byte;) => 0x25bc8287 was called from RWX@0x12011e40[libdexjni.so]0x11e40
JNIEnv->GetStaticMethodID(java/lang/Character.valueOf(C)Ljava/lang/Character;) => 0xba1ae606 was called from RWX@0x12011fec[libdexjni.so]0x11fec
JNIEnv->GetStaticMethodID(java/lang/Short.valueOf(S)Ljava/lang/Short;) => 0xe91382b0 was called from RWX@0x12012184[libdexjni.so]0x12184
JNIEnv->GetStaticMethodID(java/lang/Integer.valueOf(I)Ljava/lang/Integer;) => 0x8f152ce2 was called from RWX@0x12012338[libdexjni.so]0x12338
JNIEnv->GetStaticMethodID(java/lang/Float.valueOf(F)Ljava/lang/Float;) => 0xdac25823 was called from RWX@0x120137fc[libdexjni.so]0x137fc
JNIEnv->GetStaticMethodID(java/lang/Double.valueOf(D)Ljava/lang/Double;) => 0xca7b90a5 was called from RWX@0x12013984[libdexjni.so]0x13984
JNIEnv->GetStaticMethodID(java/lang/Long.valueOf(J)Ljava/lang/Long;) => 0x1a324bff was called from RWX@0x12013b0c[libdexjni.so]0x13b0c
INFO [com.github.unidbg.spi.Dlfcn] (Dlfcn:31) - Find symbol "dvmDecodeIndirectRef" failed: handle=0x0, LR=RWX@0x12013c8c[libdexjni.so]0x13c8c
INFO [com.github.unidbg.spi.Dlfcn] (Dlfcn:31) - Find symbol "dvmFindStaticFieldHier" failed: handle=0x0, LR=RWX@0x12013e34[libdexjni.so]0x13e34
JNIEnv->GetMethodID(java/lang/Integer.intValue()I) => 0x5d9f068b was called from RWX@0x12013ed8[libdexjni.so]0x13ed8
JNIEnv->GetMethodID(java/lang/Short.intValue()I) => 0xa3b9250d was called from RWX@0x12013f70[libdexjni.so]0x13f70
JNIEnv->GetMethodID(java/lang/Character.charValue()C) => 0x71f71ced was called from RWX@0x12010ae4[libdexjni.so]0x10ae4
JNIEnv->GetMethodID(java/lang/Byte.intValue()I) => 0x4d014b99 was called from RWX@0x12010b80[libdexjni.so]0x10b80
JNIEnv->GetMethodID(java/lang/Boolean.booleanValue()Z) => 0x31f67dab was called from RWX@0x12010d20[libdexjni.so]0x10d20
JNIEnv->GetMethodID(java/lang/Long.longValue()J) => 0x44606195 was called from RWX@0x12010e84[libdexjni.so]0x10e84
JNIEnv->GetMethodID(java/lang/Float.floatValue()F) => 0x6ff98ad7 was called from RWX@0x12010ff8[libdexjni.so]0x10ff8
JNIEnv->GetMethodID(java/lang/Double.doubleValue()D) => 0x8acb0bf9 was called from RWX@0x12011178[libdexjni.so]0x11178
JNIEnv->FindClass(android/os/Build$VERSION) was called from RWX@0x1201732c[libdexjni.so]0x1732c
JNIEnv->GetStaticFieldID(android/os/Build$VERSION.SDK_INTI) => 0x1e4ff4f1 was called from RWX@0x12017354[libdexjni.so]0x17354
JNIEnv->GetStaticIntField(class android/os/Build$VERSION, SDK_INT => 0x17) was called from RWX@0x1201736c[libdexjni.so]0x1736c
JNIEnv->FindClass(com/fort/andjni/JniLib) was called from RWX@0x1200fea4[libdexjni.so]0xfea4
JNIEnv->RegisterNatives(com/fort/andjni/JniLib, RW@0x1255fe50[libdexjni.so]0x55fe50, 10) was called from RWX@0x1200ff58[libdexjni.so]0xff58
RegisterNative(com/fort/andjni/JniLib, cV([Ljava/lang/Object;)V, RWX@0x12017770[libdexjni.so]0x17770)
RegisterNative(com/fort/andjni/JniLib, cI([Ljava/lang/Object;)I, RWX@0x120177c4[libdexjni.so]0x177c4)
RegisterNative(com/fort/andjni/JniLib, cL([Ljava/lang/Object;)Ljava/lang/Object;, RWX@0x1201781c[libdexjni.so]0x1781c)
RegisterNative(com/fort/andjni/JniLib, cS([Ljava/lang/Object;)S, RWX@0x12017874[libdexjni.so]0x17874)
RegisterNative(com/fort/andjni/JniLib, cC([Ljava/lang/Object;)C, RWX@0x120178cc[libdexjni.so]0x178cc)
RegisterNative(com/fort/andjni/JniLib, cB([Ljava/lang/Object;)B, RWX@0x12017924[libdexjni.so]0x17924)
RegisterNative(com/fort/andjni/JniLib, cJ([Ljava/lang/Object;)J, RWX@0x1201797c[libdexjni.so]0x1797c)
RegisterNative(com/fort/andjni/JniLib, cZ([Ljava/lang/Object;)Z, RWX@0x120179d4[libdexjni.so]0x179d4)
RegisterNative(com/fort/andjni/JniLib, cF([Ljava/lang/Object;)F, RWX@0x12017a2c[libdexjni.so]0x17a2c)
RegisterNative(com/fort/andjni/JniLib, cD([Ljava/lang/Object;)D, RWX@0x12017a84[libdexjni.so]0x17a84) 
```

整个运行流程非常清晰，可以依据这个来补 sub_12010028 中的结构体.  
这里我为了方便选择调用 0 号函数（这张图好像是第三次出现了 233）  
[Java] _纯文本查看_ _复制代码_

```
vm.resolveClass("com/fort/andjni/JniLib").callStaticJniMethod(emulator, "cV([Ljava/lang/Object;)V", new ArrayObject(vm.resolveClass("android/app/Activity"), vm.resolveClass("android/app/Activity").newObject(null), vm.resolveClass("android/os/Bundle").newObject(null), DvmInteger.valueOf(vm, 0)));

```

因为函数简单（毕竟类都叫 BlankActivity 了），没有复杂逻辑，只补了几个简单函数就跑完了，整个流程被顺利 trace 下来.  
毕竟是 vmp，接下来的首要任务是先找到 insns.  
这里通过定位我命名的那个 code_item 结构然后偏移 0x10 即为 insns，而 code_item 又在 j__lSSO05lI$I5IllOl_0lS$_lllIII__l$IOlI55$O550IS__lS5$ 函数返回的结构体中：  
![](https://attach.52pojie.cn/forum/202606/06/105413x9aee9esbazbkpea.png)

**图片 61.png** _(25.08 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI2NHxhMTRiYTEwNnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:54 上传

  
![](https://attach.52pojie.cn/forum/202606/06/105415wdjjujhfd1h73y1d.png)

**图片 62.png** _(34.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI2NXw0OTMwYzIzMHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:54 上传

  
在 trace 中搜索 0x12018194：  
[Plain Text] _纯文本查看_ _复制代码_

```
[libdexjni.so 0x018194] [f80300aa] 0x12018194: "mov x24, x0" x0=0x12893030 => x24=0x12893030

```

得到结构体内存地址 0x12893030，再查找其偏移 0x8 处的 code_item 读取：  
![](https://attach.52pojie.cn/forum/202606/06/105500wu13wh2v1qv9ve1w.png)

**图片 63.png** _(155.25 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI2NnxkYjk5MDNhNHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:55 上传

  
[Plain Text] _纯文本查看_ _复制代码_

```
[libdexjni.so 0x018228] [080740f9] 0x12018228: "ldr x8, [x24, #8]" ; mem[READ] abs=0x12893038 x24=0x12893030 => x8=0x128ac000

```

得到 code_item 地址 0x128ac000，偏移 0x10 得到 0x128ac010 即为要找的 insns 地址.  
同时也在这个 0x12018228 处下断点，先把 code_item 的具体数据获取到，看看长什么样子  
[Plain Text] _纯文本查看_ _复制代码_

```
m0x128ac000 0x70
>-----------------------------------------------------------------------------<
size: 112
0000: 04 00 02 00 02 00 00 00 CC 68 1C 00 30 00 00 00    .........h..0...
0010: 2E 9C 00 00 F9 04 08 8A 6C 8D 01 00 90 50 4D 6E    ........l....PMn
0020: 2A 4A 20 00 C3 06 9A E6 10 00 68 C5 54 B3 20 00    *J .......h.T. .
0030: 40 00 82 5E 32 00 D7 FA 4D 5B 10 00 3A 00 55 9B    @..^2...M[..:.U.
0040: 02 00 73 66 AF FC 03 00 FE 25 00 00 00 04 5B CE    ..sf.....%....[.
0050: 28 00 FD 58 4D 5B 20 00 01 03 F5 4D 03 00 E5 23    (..XM[ ....M...#
0060: 4D 5B 10 00 2D 00 48 1E 02 00 1C 66 A8 47 F4 62    M[..-.H....f.G.b
^-----------------------------------------------------------------------------^

```

对比一下 [code_item 的标准结构](https://source.android.com/docs/core/runtime/dex-format?hl=zh-cn#code-item)（这里不去关注后面的 try_item）  
[C] _纯文本查看_ _复制代码_

```
struct code_item {
    uint16_t registers_size;
    uint16_t ins_size;
    uint16_t outs_size;
    uint16_t tries_size;
    uint32_t debug_info_off;
    uint32_t insns_size;
    uint16_t insns[insns_size];
}

```

前 0x10 字节的确非常符合 code_item 的结构，并且可以验证偏移 0xc 处的 0x30 正是 insns_size（这段数据没有 try_item，长度共 0x70，符合 0x10+0x30*2），因此后面 0x60 字节即为 insns.  
查找对 0x128ac010 处的读取：  
[Plain Text] _纯文本查看_ _复制代码_

```
[libdexjni.so 0x04e5cc] [090140b9] 0x1204e5cc: "ldr w9, [x8]" ; mem[READ] abs=0x128ac010 x8=0x128ac010 => w9=0x9c2e

```

![](https://attach.52pojie.cn/forum/202606/06/105641h265fk7lxwf755ff.png)

**图片 64.png** _(111.13 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI2OHwyMTYzYWNkMXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:56 上传

  
读出了 0x00009c2e 四个字节，低 16 位被拿去查表 0x12098a30，结果是 0x12098a30 + (0x8d6c << 3)) = 0x120e6ba0  
然后从这个位置读出了一个地址，最后 br 跳过去了  
![](https://attach.52pojie.cn/forum/202606/06/105707ycwiww2a928fwu93.png)

**图片 65.png** _(13.6 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI2OXw3MzAxZmY4ZHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:57 上传

  
至于这个目标地址... 依然在解释器函数之内：  
![](https://attach.52pojie.cn/forum/202606/06/105717j73vh4occww2bb6v.png)

**图片 66.png** _(16.77 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3MHw1MGQyZTYxYnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:57 上传

  
（0x12029578-0x1206e068，这函数实在太大了）  
而高 16 位则是  
[Plain Text] _纯文本查看_ _复制代码_

```
[libdexjni.so 0x04e5d0] [e9ab0af9] 0x1204e5d0: "str x9, [sp, #0x1550]" ; mem[WRITE] abs=0xe4ffe500 x9=0x9c2e sp=0xe4ffcfb0 => x9=0x9c2e
 
->
 
[libdexjni.so 0x038d24] [e8ab4af9] 0x12038d24: "ldr x8, [sp, #0x1550]" ; mem[READ] abs=0xe4ffe500 sp=0xe4ffcfb0 => x8=0x9c2e
[libdexjni.so 0x038d28] [085d1053] 0x12038d28: "ubfx w8, w8, #0x10, #8" w8=0x9c2e => w8=0x0
[libdexjni.so 0x038d2c] [e8a70af9] 0x12038d2c: "str x8, [sp, #0x1548]" ; mem[WRITE] abs=0xe4ffe4f8 x8=0x0 sp=0xe4ffcfb0 => x8=0x0
 
->
 
[libdexjni.so 0x056770] [e8a74af9] 0x12056770: "ldr x8, [sp, #0x1548]" ; mem[READ] abs=0xe4ffe4f8 sp=0xe4ffcfb0 => x8=0x0
[libdexjni.so 0x056774] [083d4092] 0x12056774: "and x8, x8, #0xffff" x8=0x0 => x8=0x0
[libdexjni.so 0x056778] [e11742f9] 0x12056778: "ldr x1, [sp, #0x428]" ; mem[READ] abs=0xe4ffd3d8 sp=0xe4ffcfb0 => x1=0x35ef1869
[libdexjni.so 0x05677c] [e9b34af9] 0x1205677c: "ldr x9, [sp, #0x1560]" ; mem[READ] abs=0xe4ffe510 sp=0xe4ffcfb0 => x9=0x134d1080
[libdexjni.so 0x056780] [217928f8] 0x12056780: "str x1, [x9, x8, lsl #3]" ; mem[WRITE] abs=0x134d1080 x1=0x35ef1869 x9=0x134d1080 x8=0x0 => x1=0x35ef1869

```

通过高 16 位查询了一个表并写入了一个值.  
低 16 位的查表跳转，很像是在解释指令；而高 16 位的查表写入，则像是对寄存器的操作.  
但是标准的 opcode 明明是 uint8_t，这里却是 uint16_t，难不成魔改了？  
简单翻了一下表 0x12098a30，发现其中元素大多为 0，但非 0 元素全部指向解释器函数内部，加大了 uint16_t 的 vmcode 的可能性.  
为了验证，直接查一下第 0xffff 位的元素地址，看看这个表是否真的有这么大：  
0x12098a30 + (0xffff << 3) = 0x12118a28  
![](https://attach.52pojie.cn/forum/202606/06/105823b6q2edi60j65qt6j.png)

**图片 67.png** _(30.5 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3MXxjY2M4M2E0Y3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:58 上传

  
结果非常神奇，恰好达到了一个数据分界处，0x12118a28 之上的部分即为跳转表，之下的部分则是其他数据，所以这个表的确有 0xffff 个元素，涵盖了整个 uint16_t 范围.  
种种迹象表明，vmcode 就是 uint16_t 类型的，因此这个 vmp 必然存在魔改.  
跳转表得到后，使用 unidbg 对整个表范围的读取进行监听：  
[Java] _纯文本查看_ _复制代码_

```
emulator.traceRead(0x12098a30, 0x12118a28);

```

得到结果：  
[Plain Text] _纯文本查看_ _复制代码_

```
Memory READ at 0x120e6ba0, data size = 8, data value = 0x000000001206588c, PC=RWX@0x1204e5e0[libdexjni.so]0x4e5e0, LR=RWX@0x12050708[libdexjni.so]0x50708
Memory READ at 0x120df590, data size = 8, data value = 0x00000000120449fc, PC=RWX@0x1204b05c[libdexjni.so]0x4b05c, LR=RWX@0x1205081c[libdexjni.so]0x5081c
Memory READ at 0x120bdb80, data size = 8, data value = 0x0000000012065158, PC=RWX@0x120553a4[libdexjni.so]0x553a4, LR=RWX@0x12031234[libdexjni.so]0x31234
Memory READ at 0x120f24d0, data size = 8, data value = 0x0000000012044d90, PC=RWX@0x1202e93c[libdexjni.so]0x2e93c, LR=RWX@0x12051cfc[libdexjni.so]0x51cfc
Memory READ at 0x120c6498, data size = 8, data value = 0x00000000120651c0, PC=RWX@0x120570c8[libdexjni.so]0x570c8, LR=RWX@0x12056044[libdexjni.so]0x56044
Memory READ at 0x12116fa8, data size = 8, data value = 0x000000001205cba4, PC=RWX@0x12052584[libdexjni.so]0x52584, LR=RWX@0x1205f1cc[libdexjni.so]0x5f1cc
Memory READ at 0x120aba20, data size = 8, data value = 0x0000000012064fd8, PC=RWX@0x12064ae8[libdexjni.so]0x64ae8, LR=RWX@0x12049f78[libdexjni.so]0x49f78
Memory READ at 0x120c6498, data size = 8, data value = 0x00000000120651c0, PC=RWX@0x12053570[libdexjni.so]0x53570, LR=RWX@0x12049f78[libdexjni.so]0x49f78
Memory READ at 0x120c6498, data size = 8, data value = 0x00000000120651c0, PC=RWX@0x12052584[libdexjni.so]0x52584, LR=RWX@0x1205f1cc[libdexjni.so]0x5f1cc
Memory READ at 0x120bc770, data size = 8, data value = 0x0000000012065708, PC=RWX@0x12052584[libdexjni.so]0x52584, LR=RWX@0x1205f1cc[libdexjni.so]0x5f1cc

```

对读取的地址进行逆向操作，还原出 vmcode：  
vmcode = (address - 0x12098a30) >>> 3  
逐个翻译得：0x9c2e、0x8d6c、0x4a2a、0xb354、0x5b4d、0xfcaf、0x25fe、0x5b4d、0x5b4d、0x47a8.  
并以此为特征将指令分割：  
![](https://attach.52pojie.cn/forum/202606/06/105929no3v37k3kiso233j.png)

**图片 68.png** _(43.38 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3MnxhYzM1NGQyZnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:59 上传

  
现在 vmcode 已知了，接下来想要分析指令操作，还有必不可少的一步：寄存器监听.  
在上文有一个地方我们看到了对寄存器内存的分配：  
![](https://attach.52pojie.cn/forum/202606/06/105950v8ikn82hqbzb8qvb.png)

**图片 69.png** _(44.31 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3M3wxOGIzMDRlNnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 10:59 上传

  
unidbg 断点可得分配的内存地址为 0x134d1080，而这个方法的 registers_size 为 4，因此监听：  
[Java] _纯文本查看_ _复制代码_

```
emulator.traceRead(0x134d1080, 0x134d10a0);
emulator.traceWrite(0x134d1080, 0x134d10a0);

```

其中 0x134d1080 为 v0 寄存器、0x134d1088 v1、0x134d1090 p0、0x134d1098 p1  
![](https://attach.52pojie.cn/forum/202606/06/110028u80a1o0h9h0jz9c0.png)

**图片 70.png** _(1002.35 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3NHxiNjZlYzdmNHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:00 上传

  
可以清楚看到先对寄存器进行了一些初始化，写入了 p0(this) 和 p1(android/os/Bundle) 的值.  
Memory READ at 0x120e6ba0 后开始运行第一条指令，在 v0 寄存器上写入了值 0x000000004bb4de6a；第二条指令在 v1 寄存器写入值 0x000000002f8f5f62，并且两条指令执行的作用看上去出奇一致，都是在初始化一个 jstring，不过 vmcode 不同.  
从 log 中可以猜到，这两个值正是来自 Memory WRITE 上方的 JNIEnv->NewGlobalRef.  
这里凭空出现了一个字符串，自然要搞清来由.  
直接跳转到上方 jni 调用的地址处，发现是从 j__l$I0lOll00SOlSI_ISl50IO0lOI_Ol$_IIIlIIlIl5IIll5IS5$ 读取的  
![](https://attach.52pojie.cn/forum/202606/06/110111byu9p7qqx2rzgxgq.png)

**图片 71.png** _(36.15 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3NXxlMWU2YjMyNXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:01 上传

  
跟过去发现又是在查表：  
![](https://attach.52pojie.cn/forum/202606/06/110127urc1kusxjg7qmuuk.png)

**图片 72.png** _(26.28 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3Nnw4ZDQzZjhmNXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:01 上传

  
先不急着去追数据，先打印一下用于查表的 idx：  
[Java] _纯文本查看_ _复制代码_

```
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        System.out.printf("idx：%#x\n", backend.reg_read(Unicorn.UC_ARM64_REG_X0).intValue());
    }
 
    @Override
    public void onAttach(UnHook unHook) {
    }
 
    @Override
    public void detach() {
    }
}, 0x1200cea0, 0x1200cea0, null);

```

![](https://attach.52pojie.cn/forum/202606/06/110210a08jy180vs28r40p.png)

**图片 73.png** _(162.83 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3N3w5MjE0ODdhMXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:02 上传

  
再回头看一眼 insns，这不就在这嘛：  
![](https://attach.52pojie.cn/forum/202606/06/110232hee6hsr668c5ib0i.png)

**图片 74.png** _(42.44 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3OHwyNDVmMDdiMnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:02 上传

  
所以这两字节就是用于查字符串表的索引喽.  
交叉引用一下字符串表，依旧是熟悉的感觉：  
![](https://attach.52pojie.cn/forum/202606/06/110247ial9g7qj9chhc67i.png)

**图片 75.png** _(43.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI3OXxmMTFiMzFmY3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:02 上传

  
看到这里，你可能已经猜到了，没错，这就是 JNI_OnLoad 调用的第二个函数.（之前跳过的函数最后都没逃过 qwq）  
![](https://attach.52pojie.cn/forum/202606/06/110315guqffegugzpkizye.png)

**图片 76.png** _(71.91 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4MHw3MTFkZGZhYnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:03 上传

  
依然是通过 MessagePack 解析二进制数据，并且这个函数初始化的数据不只有这一个表，函数中一共对五个表进行了初始化：  
![](https://attach.52pojie.cn/forum/202606/06/110341cugdfukpzkvza0pl.png)

**图片 77.png** _(23.07 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4MXxiMDhkMmM1OHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:03 上传

  
依旧是等这个函数运行完后，使用 unidbg 直接打印出表，然后猜测含义，这里直接给出分析结果：  
![](https://attach.52pojie.cn/forum/202606/06/110343oeqfy0kjnc248280.png)

**图片 78.png** _(26.89 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4Mnw1NjhiNWY4NnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:03 上传

  
![](https://attach.52pojie.cn/forum/202606/06/110420ru6gtgzs6cmgg3m5.png)

**图片 79.png** _(71.38 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4M3w4NTMzZTFkN3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:04 上传

  
![](https://attach.52pojie.cn/forum/202606/06/110422bq2o2q6g769q26qf.png)

**图片 80.png** _(38.33 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4NHw3MjQzNzNjMHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:04 上传

  
![](https://attach.52pojie.cn/forum/202606/06/110424t6j6mccmmsow880v.png)

**图片 81.png** _(63.97 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4Nnw5NmQ3YjRmMXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:04 上传

  
![](https://attach.52pojie.cn/forum/202606/06/110426aggg3gkgk5j4xvpx.png)

**图片 82.png** _(69.51 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4N3w0NjE3MzllMnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:04 上传

  
![](https://attach.52pojie.cn/forum/202606/06/110428x9zezwywqy0lmtwh.png)

**图片 83.png** _(77.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4OHw2NDdmMzI5ZHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:04 上传

  
结构体：  
[C] _纯文本查看_ _复制代码_

```
struct dvm_string __packed
{
char* string;
uint64_t len;
};
 
struct dvm_method __packed
{
char* class_name;
char* method_args;
char* method_name;
};
 
struct dvm_method2 __packed
{
char* class_name;
char* method_sig;
char* method_name;
char* method_shorty;
uint64_t zero;
};

```

看了这么多，至于被解析的数据在哪，在函数开头有这么一个调用：  
![](https://attach.52pojie.cn/forum/202606/06/110537lekgi7ze97e9nw7z.png)

**图片 84.png** _(64.77 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI4OXw5YWUxNzk2OXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:05 上传

  
这个函数就是对 so 中一段数据进行 lz4 解压，所以来源就很明显了.  
![](https://attach.52pojie.cn/forum/202606/06/110603epigd66d8occn888.png)

**图片 85.png** _(38.98 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI5MHwzNDE3MmNkZXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:06 上传

  
现在数据来源都清楚了，那么写入 v0 还是 v1 是怎么决定的呢，其实就是 vmcode 后面的两字节，也就是之前追踪的 “0x00009c2e 的高 16 位”：  
![](https://attach.52pojie.cn/forum/202606/06/110605nr077neneq0kz0e7.png)

**图片 86.png** _(42.17 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI5MXw1NTMzYzMwNnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:06 上传

  
第一条指令是 0x00，所以写入 v0 寄存器；第二条指令是 0x01，写入 v1 寄存器.  
至于最后 0x8a08 和 0x6e4d 那两字节数据，在 trace 中也没有读取的记录，只读了个半字  
[Plain Text] _纯文本查看_ _复制代码_

```
[libdexjni.so 0x038d1c] [01094079] 0x12038d1c: "ldrh w1, [x8, #4]" ; mem[READ] abs=0x128ac014 x8=0x128ac010 => w1=0x4f9

```

应该是用于对齐的无用数据了（后面同样有大量这种数据）.  
因此这两条指令翻译过来就是  
[Plain Text] _纯文本查看_ _复制代码_

```
const-string v0, "BlankActivity"
const-string v1, "onCreate"

```

然后再来看下一条，这条指令显然是在调用 com/albert/xchatkit/d.b(Ljava/lang/String;Ljava/lang/String;)V 方法  
![](https://attach.52pojie.cn/forum/202606/06/110744eaa0wzt8qaja2jal.png)

**图片 87.png** _(157.57 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI5MnxiNWE0NWMzZXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:07 上传

  
不出意外要用到刚才解析出来的 method 表.  
为了省事，直接把四个表（有一个表不存在这样的查表函数，因此是四个）的读取全部 hook 一下：  
![](https://attach.52pojie.cn/forum/202606/06/110801hbdaugaa0azd9b90.png)

**图片 88.png** _(85.5 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI5M3xhZmY3NzA1Y3wxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:08 上传

  
[Java] _纯文本查看_ _复制代码_

```
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        System.out.printf("g_string_list2：%#x\n", backend.reg_read(Unicorn.UC_ARM64_REG_X0).intValue());
    }
 
    @Override
    public void onAttach(UnHook unHook) {
    }
 
    @Override
    public void detach() {
    }
}, 0x1200cea0, 0x1200cea0, null);
 
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        System.out.printf("g_int_list：%#x\n", backend.reg_read(Unicorn.UC_ARM64_REG_X0).intValue());
    }
 
    @Override
    public void onAttach(UnHook unHook) {
    }
 
    @Override
    public void detach() {
    }
}, 0x1200ceec, 0x1200ceec, null);
 
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        System.out.printf("g_method_list：%#x\n", backend.reg_read(Unicorn.UC_ARM64_REG_X0).intValue());
    }
 
    @Override
    public void onAttach(UnHook unHook) {
    }
 
    @Override
    public void detach() {
    }
}, 0x1200cf38, 0x1200cf38, null);
 
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        System.out.printf("g_method_list2：%#x\n", backend.reg_read(Unicorn.UC_ARM64_REG_X0).intValue());
    }
 
    @Override
    public void onAttach(UnHook unHook) {
    }
 
    @Override
    public void detach() {
    }
}, 0x1200cf84, 0x1200cf84, null);

```

重新运行一下得到 method idx 是 0x6c3  
![](https://attach.52pojie.cn/forum/202606/06/110850jjjaav0ev3javhve.png)

**图片 89.png** _(108.53 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI5NHwzNWQ3OWE4ZHwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:08 上传

  
![](https://attach.52pojie.cn/forum/202606/06/110852cv0vzdqde6v72iph.png)

**图片 90.png** _(28.53 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI5NXw5OTE4ZDE2NXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:08 上传

  
由于调用的 jni 函数是 CallStaticVoidMethodA，因此这条指令应该是 invoke-static，并且从寄存器读取或者结合前两条指令可以得出两个入参 jstring 是 v0 和 v1.  
现在这条指令被魔改成了这样，尝试找一下其中是否存在规律.  
invoke-static 原是 35c 格式的，如果要写出原指令的字节码，应是（我的图做得好烂啊 hhhhh）：  
![](https://attach.52pojie.cn/forum/202606/06/110912o63n1c9ceyscc5ez.png)

**图片 91.png** _(43.02 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI5NnxkNjA4NzAyOXwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:09 上传

  
有没有发现有点眼熟？忽视掉用于对齐的垃圾数据：  
![](https://attach.52pojie.cn/forum/202606/06/110932gjt44ve35u5wx3t5.png)

**图片 92.png** _(26.35 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NjI5N3wwMDExZmMxYnwxNzgwNzM4MzMxfDB8MjExMTQzMg%3D%3D&nothumb=yes)

2026-6-6 11:09 上传

  
除了 opcode 变成了二字节的 vmcode，其他数据简直一模一样！  
所以这条 invoke 指令也就被还原了：  
[Plain Text] _纯文本查看_ _复制代码_

```
invoke-static {v0, v1}, Lcom/albert/xchatkit/d;->b(Ljava/lang/String;Ljava/lang/String;)V

```

后面的指令根据 trace 大概看了下，和上述逻辑均相同，都是由标准结构塞入一些对齐数据来魔改的，本质和标准结构类似.  
这里给出 trace、vm 字节码以及还原对照：  
第四条指令：  
trace：  
[Plain Text] _纯文本查看_ _复制代码_

```
Memory READ at 0x120f24d0, data size = 8, data value = 0x0000000012044d90, PC=RWX@0x1202e93c[libdexjni.so]0x2e93c, LR=RWX@0x12051cfc[libdexjni.so]0x51cfc
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x1204f910[libdexjni.so]0x4f910, LR=RWX@0x12051cfc[libdexjni.so]0x51cfc
Memory READ at 0x134d1080, data size = 8, data value = 0x000000003bd94634, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1205f96c[libdexjni.so]0x5f96c
Memory READ at 0x134d1088, data size = 8, data value = 0x0000000074294adb, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
Memory READ at 0x134d1098, data size = 8, data value = 0x0000000017baae6e, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
g_method_list2：0x40
JNIEnv->FindClass(android/app/Activity) was called from RWX@0x1205f990[libdexjni.so]0x5f990
JNIEnv->FindClass(java/lang/Class) was called from RWX@0x1205f9ac[libdexjni.so]0x5f9ac
JNIEnv->GetMethodID(java/lang/Class.isInterface()Z) => 0x789061df was called from RWX@0x1205f9d0[libdexjni.so]0x5f9d0
JNIEnv->CallBooleanMethodV(class android/app/Activity, isInterface() => true) was called from RWX@0x12018df0[libdexjni.so]0x18df0
JNIEnv->GetMethodID(android/app/Activity.onCreate(Landroid/os/Bundle;)V) => 0x2f2ecda8 was called from RWX@0x1204adc4[libdexjni.so]0x4adc4
Memory READ at 0x134d1098, data size = 8, data value = 0x0000000017baae6e, PC=RWX@0x1202b0b8[libdexjni.so]0x2b0b8, LR=RWX@0x1204adc4[libdexjni.so]0x4adc4
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x1204e57c[libdexjni.so]0x4e57c, LR=RWX@0x1204adc4[libdexjni.so]0x4adc4
JNIEnv->CallNonVirtualVoidMethodA(android.app.Activity@43d7741f, android/app/Activity, onCreate(android.os.Bundle@17baae6e)) was called from RWX@0x12055d44[libdexjni.so]0x55d44

```

[Plain Text] _纯文本查看_ _复制代码_

```
字节码：54 B3 20 00 40 00 82 5E 32 00 D7 FA
0xb354 -> opcode：invoke-super
0x(00)20 -> registerCount=2, regG=0
0x0040 -> g_method_list2 idx[android/app/Activity.onCreate(Landroid/os/Bundle;)V]
0x5e82 -> padding
0x0032 -> regC=2(p0), regD=3(p1), regE=0, regF=0
0xfad7 -> padding
Smali：invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V;

```

第五条指令：  
trace：  
[Plain Text] _纯文本查看_ _复制代码_

```
Memory READ at 0x120c6498, data size = 8, data value = 0x00000000120651c0, PC=RWX@0x120570c8[libdexjni.so]0x570c8, LR=RWX@0x12056044[libdexjni.so]0x56044
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x1205da04[libdexjni.so]0x5da04, LR=RWX@0x12056044[libdexjni.so]0x56044
Memory READ at 0x134d1080, data size = 8, data value = 0x000000003bd94634, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x12062f18[libdexjni.so]0x62f18
Memory READ at 0x134d1088, data size = 8, data value = 0x0000000074294adb, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
Memory READ at 0x134d1098, data size = 8, data value = 0x0000000017baae6e, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
g_method_list2：0x3a
JNIEnv->FindClass(android/app/Activity) was called from RWX@0x12062f3c[libdexjni.so]0x62f3c
JNIEnv->GetMethodID(android/app/Activity.getWindow()Landroid/view/Window;) => 0x4333f814 was called from RWX@0x120327b4[libdexjni.so]0x327b4
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x12055a5c[libdexjni.so]0x55a5c, LR=RWX@0x1204bc14[libdexjni.so]0x4bc14
JNIEnv->CallObjectMethodA(android.app.Activity@43d7741f, getWindow() => android.view.Window@130f889) was called from RWX@0x1205ee14[libdexjni.so]0x5ee14

```

[Plain Text] _纯文本查看_ _复制代码_

```
字节码：4D 5B 10 00 3A 00 55 9B 02 00 73 66
0x5b4d -> opcode：invoke-virtual
0x(00)10 -> registerCount=1, regG=0
0x003a -> g_method_list2 idx[android/app/Activity.getWindow()Landroid/view/Window;]
0x9b55 -> padding
0x0002 -> regC=2(p0), regD=0, regE=0, regF=0
0x6673 -> padding
Smali：invoke-virtual {p0}, Landroid/app/Activity;->getWindow()Landroid/view/Window;

```

第六条指令：  
trace：  
[Plain Text] _纯文本查看_ _复制代码_

```
Memory READ at 0x12116fa8, data size = 8, data value = 0x000000001205cba4, PC=RWX@0x12052584[libdexjni.so]0x52584, LR=RWX@0x1205f1cc[libdexjni.so]0x5f1cc
Memory WRITE at 0x134d1098, data size = 8, data value = 0x000000000130f889, PC=RWX@0x12049f68[libdexjni.so]0x49f68, LR=RWX@0x1205f1cc[libdexjni.so]0x5f1cc

```

[Plain Text] _纯文本查看_ _复制代码_

```
字节码：AF FC 03 00
0xfcaf -> opcode：move-result-object
0x0003 -> reg=3(p1)
Smali：move-result-object p1

```

第七条指令：  
trace：  
[Plain Text] _纯文本查看_ _复制代码_

```
Memory READ at 0x120aba20, data size = 8, data value = 0x0000000012064fd8, PC=RWX@0x12064ae8[libdexjni.so]0x64ae8, LR=RWX@0x12049f78[libdexjni.so]0x49f78
Memory WRITE at 0x134d1080, data size = 8, data value = 0x0000000000280400, PC=RWX@0x12035028[libdexjni.so]0x35028, LR=RWX@0x12049f78[libdexjni.so]0x49f78

```

[Plain Text] _纯文本查看_ _复制代码_

```
字节码：FE 25 00 00 00 04 5B CE 28 00 FD 58
0x25fe -> opcode：const
0x(00)00 -> reg=0(v0)
0x0400 -> low 16 bits
0xce5b -> padding
0x0028 -> high 16 bits
0x58fd -> padding
Smali：const v0, 0x00280400

```

第八条指令：  
trace：  
[Plain Text] _纯文本查看_ _复制代码_

```
Memory READ at 0x120c6498, data size = 8, data value = 0x00000000120651c0, PC=RWX@0x12053570[libdexjni.so]0x53570, LR=RWX@0x12049f78[libdexjni.so]0x49f78
Memory READ at 0x134d1098, data size = 8, data value = 0x000000000130f889, PC=RWX@0x1205da04[libdexjni.so]0x5da04, LR=RWX@0x12049f78[libdexjni.so]0x49f78
Memory READ at 0x134d1080, data size = 8, data value = 0x0000000000280400, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x12062f18[libdexjni.so]0x62f18
Memory READ at 0x134d1088, data size = 8, data value = 0x0000000074294adb, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
Memory READ at 0x134d1098, data size = 8, data value = 0x000000000130f889, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
g_method_list2：0x301
JNIEnv->FindClass(android/view/Window) was called from RWX@0x12062f3c[libdexjni.so]0x62f3c
JNIEnv->GetMethodID(android/view/Window.addFlags(I)V) => 0x3448735f was called from RWX@0x120327b4[libdexjni.so]0x327b4
Memory READ at 0x134d1080, data size = 8, data value = 0x0000000000280400, PC=RWX@0x1202ed4c[libdexjni.so]0x2ed4c, LR=RWX@0x1204bc14[libdexjni.so]0x4bc14
Memory READ at 0x134d1098, data size = 8, data value = 0x000000000130f889, PC=RWX@0x12055a5c[libdexjni.so]0x55a5c, LR=RWX@0x1204bc14[libdexjni.so]0x4bc14
JNIEnv->CallVoidMethodA(android.view.Window@130f889, addFlags(0x280400)) was called from RWX@0x1205eec8[libdexjni.so]0x5eec8

```

[Plain Text] _纯文本查看_ _复制代码_

```
字节码：4D 5B 20 00 01 03 F5 4D 03 00 E5 23
0x5b4d分析前面已经给出
Smali：invoke-virtual {p1, v0}, Landroid/view/Window;->addFlags(I)V

```

第九条指令：  
trace：  
[Plain Text] _纯文本查看_ _复制代码_

```
Memory READ at 0x120c6498, data size = 8, data value = 0x00000000120651c0, PC=RWX@0x12052584[libdexjni.so]0x52584, LR=RWX@0x1205f1cc[libdexjni.so]0x5f1cc
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x1205da04[libdexjni.so]0x5da04, LR=RWX@0x1205f1cc[libdexjni.so]0x5f1cc
Memory READ at 0x134d1080, data size = 8, data value = 0x0000000000280400, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x12062f18[libdexjni.so]0x62f18
Memory READ at 0x134d1088, data size = 8, data value = 0x0000000074294adb, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
Memory READ at 0x134d1098, data size = 8, data value = 0x000000000130f889, PC=RWX@0x1206e2fc[libdexjni.so]0x6e2fc, LR=RWX@0x1206e308[libdexjni.so]0x6e308
g_method_list2：0x2d
JNIEnv->FindClass(android/app/Activity) was called from RWX@0x12062f3c[libdexjni.so]0x62f3c
JNIEnv->GetMethodID(android/app/Activity.finish()V) => 0xa8352854 was called from RWX@0x120327b4[libdexjni.so]0x327b4
Memory READ at 0x134d1090, data size = 8, data value = 0x0000000043d7741f, PC=RWX@0x12055a5c[libdexjni.so]0x55a5c, LR=RWX@0x1204bc14[libdexjni.so]0x4bc14
JNIEnv->CallVoidMethodA(android.app.Activity@43d7741f, finish()) was called from RWX@0x1205eec8[libdexjni.so]0x5eec8

```

[Plain Text] _纯文本查看_ _复制代码_

```
字节码：4D 5B 10 00 2D 00 48 1E 02 00 1C 66
0x5b4d分析前面已经给出
Smali：invoke-virtual {p0}, Landroid/app/Activity;->finish()V

```

第十条指令：  
trace：  
[Plain Text] _纯文本查看_ _复制代码_

```
Memory READ at 0x120bc770, data size = 8, data value = 0x0000000012065708, PC=RWX@0x12052584[libdexjni.so]0x52584, LR=RWX@0x1205f1cc[libdexjni.so]0x5f1cc

```

[Plain Text] _纯文本查看_ _复制代码_

```
字节码：A8 47 F4 62
0x47a8 -> opcode：return-void
0x62f4 -> padding
Smali：return-void

```

所以这个函数还原下来就是：  
[Plain Text] _纯文本查看_ _复制代码_

```
const-string v0, "BlankActivity"
const-string v1, "onCreate"
invoke-static {v0, v1}, Lcom/albert/xchatkit/d;->b(Ljava/lang/String;Ljava/lang/String;)V
invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V;
invoke-virtual {p0}, Landroid/app/Activity;->getWindow()Landroid/view/Window;
move-result-object p1
const v0, 0x00280400
invoke-virtual {p1, v0}, Landroid/view/Window;->addFlags(I)V
invoke-virtual {p0}, Landroid/app/Activity;->finish()V
return-void

```

因为所选函数本身比较简单，所以还原起来并不复杂，如果包含一些跳转指令的话也许会复杂一些，不过大概原理应该就是这些啦.  
**总结**  
整个运行流程还是比较常规的，解压释放 text 代码段，JNI_OnLoad 初始化全部 vm 数据并注册跳板函数，跳板函数通过 idx 查数据进行一些初始化进入解释器解释执行.  
比较有意思的就是 vm 的字节码进行了一些魔改，好像每个指令长度都变成了之前的两倍（不确定，但帖子中见到的都符合），添加了很多用于对齐的随机数据，是我见到的第一个魔改了指令结构的 dex vmp.  
大概是对这个 vmp 的整体进行了一通简单的分析，不过因为控制流以及自身能力的缘故，可能有很多东西我没能看到，就交给其他大佬来作补充吧~  
最后感谢一下 @WZTong 教我用 unidbg，以及 @正己 的《安卓逆向这档事》带我入门，伟大无需多言  
（小白第一帖，多有不足，望多多包涵，写成后还是有点小激动的说）![](https://avatar.52pojie.cn/data/avatar/001/10/94/58_avatar_middle.jpg)正己 加精华，期待师傅后续更多佳作