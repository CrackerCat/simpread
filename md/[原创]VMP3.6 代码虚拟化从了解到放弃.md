> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274112.htm)

> [原创]VMP3.6 代码虚拟化从了解到放弃

**VMP3.6 代码虚拟化从了解到放弃**
----------------------

QQ 群:`935394656`欢迎各位逆向手子们加入 (钓鱼佬别来)

 

**VMP3.6 从了解到放弃 (其他版本与其他基于堆栈的 VM 其实也都差不多)**

### 采用 VMP3.6 超级 (变异 + 虚拟) 生成

先人肉跟踪一波整个执行流程

 

文中的`handle switch`是我个人的理解. 为切换到下一个`"执行块"`的意思

 

使用 x96dbg 来人肉跟踪`handle switch`很快 (但也是根据当前 VM 代码的自身体积决定的)

 

如果大家感兴趣这个过程的话, 可以留言我后面在编辑详细说明一下.

 

![](https://bbs.pediy.com/upload/attach/202208/687333_7BGH3UU9KTHMN8H.png)

 

![](https://bbs.pediy.com/upload/attach/202208/687333_D7H4WJRXKF952Z8.png)

 

![](https://bbs.pediy.com/upload/attach/202208/687333_8FCYEDVUTH7SSQ5.png)

### 怎么开始理解与初步分析 VM 代码虚拟化

基本都在加 VMP,TMD 这类强壳了. 不学一点 VM 的分析技巧真的很难混下去了啊...

 

前期不要过度纠结单行汇编指令与 VM_Entry 这么做对你前期了解没有任何意义. 优先分析 VM 的执行流程也就是`handle switch`跟返回正常代码段的流程.

 

由于用了一下 [xeroxz](https://git.back.engineering/_xeroxz/bestrings) 现有的代码 (VMP3.X 他的也不完善已经亲自问过了), 跑了一下 VM handle 的模拟执行感觉还挺好玩的, 自己又想深入了解一下.

 

笔者这边直接就拿最新版本 3.6 测试了... 其他版本以后有机会再看吧

 

由于 VMP3.6 的 VM_Entry 是每次编译都有一些小变化. 笔者这里发现了三种形式 (可能更多). 现在不存在 push call 这个经典入口了. 所以无法精准指出 VM_Entry 进入方式大家就直接单步进去就行了, 一直到`JMP reg`或者`PUSH reg Ret`在开始下断点跟打印寄存器信息就好了.

 

想要打印所有执行 Handle 纯纯的体力活没啥难度 (会玩的也可以用自动化).... 然后再去分析对应 Handle, 通过看命中次数来逐步分析`handle switch`.

 

如果想写出模拟执行 VM Handle 甚至 VM 的自动化分析还原 非常非常有必要手动去跟踪到每个`handle switch`. 现阶段所有开源的代码都不是特别完善, 想要分析稍微复杂一点的代码就要自己去完善模拟执行过程.

 

图上对应的源代码如下 (msvcx64 没办法编译裸函数吧? 也不清楚用的很少)

```
#include "VMProtectSDK.h"
 
uint64_t g_Number = 0;
 
void novmp()
{
    printf("[%d] novmp!!!!\r\n", g_Number);
}
 
 
void __declspec(naked) init()
{
    VMProtectBeginUltra("hello::init");
    g_Number = g_Number + 1;
    //if (g_Number % 5) {
    //    printf("[%d] hello!!!!\r\n", g_Number);
    //}
    //else {
    //    printf("[%d] world!!!!\r\n", g_Number);
    //}
    VMProtectEnd();
}
 
 
 
int main() {
    while (1)
    {
        VMProtectBeginUltra("main");
        init();
        Sleep(1);
        VMProtectEnd();
        getchar();
    }
}
```

笔者本来是想测试 VM 虚拟化代码内嵌的过程. 也就是 VM 函数中再调用 VM 的函数. 可惜 msvc 的 x64 编译器不支持这种写法. 不管你有没有返回值和参数. 它都会给你增加`sub rsp,xxx`和`add rsp,xxx`

 

这边有群友帮忙编译了一个 clang 的裸函数. 大概看了一下如果是纯裸函数, 整个虚拟机流程不会有 VM_CALL 跟 VM_RET 存在只有代码执行完成以后会执行 VM_Exit

 

想要分析更复杂的 VM 的虚拟机执行逻辑就去修改被 VM 代码复杂度. 先以最简单为例开始了解 VM 的`Handle`.

### VMP3.6 的 handle 执行

这里引用一句百度百科的解释

 

**寄存器轮转**

 

VMP 将所有寄存器都存放在了[堆栈](https://baike.baidu.com/item/堆栈/1682032)的结构中（VM_CONTEXT），结构中的每一项代表一个寄存器或者临时变量。但在运行过程中，其中的项所映射的真实寄存器都是不固定的，可以把它比作一个齿轮，每做完一个动作，部分项的映射就互换了一下位置，或者执行完一段指令，齿轮就按不固定的方向和度数转动一下，然后全部的项映射就改变了。VMP 在生成[字节码](https://baike.baidu.com/item/字节码/9953683)的过程中，维护了一份结构中每一项所映射的真实寄存器，但这只存在于编译过程，而在运行时是没有明确的信息的。这直接导致了分析和识别的难度。

 

为了防止调用过程的返回地址出现在堆栈中都是采用`JMP reg PUSH reg RET 执行下一个Handle`.

 

`JMP reg(当前`执行块`未完全结束时寄存器不会发生轮转)`从上面的截图也可以观察得出轮转 JMP 与 PUSH RET. 不定某一种方式会成为下一个`执行块`主要切换方式.

 

`PUSH reg RET(当前`执行块`未完全结束时寄存器不会发生轮转)`跟上面 JMP 是一样的

 

`RET 被VMP用来作为返回正常代码段的方式,也就是VM_CALL VM_Ret VM_ImpCALL VM_Exit...(待定不知道还有没有其他的)`

 

这里的`"执行块"`并不是指单一`handle`而是多个`handle`组成的一个大的伪函数. 共用一个`handle switch`. 由于笔者并没有分析过老版本的 (只是看过一些大佬的文章)VMP2.0X 版本或者 1.0X 版本, 所以这里就自行理解命名了.

 

在每个 Handle 的代码过程中又采用了大量的`JMP`指令进行 Handle 的代码分割. 此做法应该是为了防止 IDA F5.(TMD 壳的 Handle 可以 F5 如果有兴趣的也可以去分析 TMD 的... 但是 TMD 的 Handle 数量有点爆炸... 手动去分析也是有点人以麻)

 

想要肉眼看汇编分析打印出来的 Handle 的过程也有点麻木... 又是纯纯的体力活... 对汇编功底要求还是要扎实...

```
//笔者这里打印出来的大概有273个左右的Handle(不包括VM_Entry) 简简单单的几行代码...换算成执行指令的数量的话大概是273*30(粗略估计其实有的handle远远不止30行...)
//为什么会产生这么多呢(其中有一部分是重复调用) 有一些是VMP必须要执行的 即是只有一行代码 生成出来的执行的handle数量也不会少于200+吧(待定)
7FF7F588EDE5
7FF7F58A655E
......
7FF7F5866D04
VM_CALL"7FF7F5821210"               
7FF7F5924AFC
......
7FF7F5867E91                       
VM_RET"7FF7F5821238"               
7FF7F592E9F7
7FF7F58D6580
......
VM_ImpCALL"7FFAE2F9ADA0"            //没有加IAT保护如果加了的话这里调用方式应该还会一些变化
......
7FF7F58D9B16
7FF7F5917599
7FF7F5935279
7FF7F590F342
7FF7F58EA639
7FF7F591DC1B
7FF7F58C0A21
VM_Exit"7FF7F582126E"
```

总结就是珍爱生命远离分析 VM 代码.
-------------------

取巧的话就是分析到`VM_CALL VM_RET VM_ImpCALL VM_Exit`就好了 通过上面的几个 handle 来取巧... 至于能不能取巧就要看运气了...

 

真想要分析常规被 VM 保护的应用还是得上自动化 (但是自动化也需要手动去分析过`handle`才能更加完善自动化 所以是一个大大的莫比乌斯环). 初步学习参考[周壑](https://space.bilibili.com/37877654/channel/seriesdetail?sid=1467286) 稍微完善点的参考`NOVMP`与上面提到的 [xeroxz](https://git.back.engineering/vmp3) 但都有一定缺陷与问题 需要填的坑也不算少. 路漫漫其修远兮, 吾将上下而求索. 头发掉多少看大家的功力了

### 文中如有错误请大家及时指出.

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

最后于 15 小时前 被如梦而醉编辑 ，原因： 1111

[#调试逆向](forum-4-1-1.htm) [#VM 保护](forum-4-1-4.htm)