> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278947.htm)

> [原创] 蛮犀加固免费版脱壳

[原创] 蛮犀加固免费版脱壳

13 小时前 315

### [原创] 蛮犀加固免费版脱壳

 [![](http://passport.kanxue.com/upload/avatar/805/942805.png?1695214530)](user-home-942805.htm) [Gift1a](user-home-942805.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png) 13 小时前  315

蛮犀加固免费版脱壳
=========

前言
--

样本只作为脱壳学习的一个样本，样本的内部逻辑并没有仔细分析，且样本较为简单，通用性差。本帖仅作为学习记录，欢迎大家一同交流。

加固效果展示
------

首先先来官网看一下免费版包括的内容  
![](https://bbs.kanxue.com/upload/attach/202309/942805_S5EM3YXDG78ASKB.webp)  
![](https://bbs.kanxue.com/upload/attach/202309/942805_S2DTHYVAC9UQEM5.webp)  
![](https://bbs.kanxue.com/upload/attach/202309/942805_3WVYNS5X72SZ623.webp)  
上传一个 Demo 用于加固，Demo 反编译结果如下  
![](https://bbs.kanxue.com/upload/attach/202309/942805_SRC3Q5A4ZWFTNKT.webp)  
加固只针对 Dex 进行，且对类进行抽取，其效果如下  
![](https://bbs.kanxue.com/upload/attach/202309/942805_JS4MZ8W2XRT63H6.webp)  
![](https://bbs.kanxue.com/upload/attach/202309/942805_S4EDNV4HNZRF6DT.webp)

简单分析
----

### dex 简单分析

首先 Shell 创建了 Application 获取程序的起始控制权，之后在 attachBaseContext 通过 Helper 类加载 manxi.so 文件  
![](https://bbs.kanxue.com/upload/attach/202309/942805_PTHMEGPS8M75HZD.webp)  
![](https://bbs.kanxue.com/upload/attach/202309/942805_DSSDPB7G9C6CSPA.webp)  
![](https://bbs.kanxue.com/upload/attach/202309/942805_4GNBVCSFQV2NUDS.webp)  
之后的步骤都是在 So 层进行，所以需要对 So 层进行分析

### So 简单分析

#### 字符串解密

So 通过 ollvm 进行混淆，首先我们先通过 Qiling 模拟执行去除最简单的字符串混淆  
首先我们需要提取到 init_array 所有函数起始地址和结束地址，这里我通过 IDAPython 进行提取

```
import idaapi
import lief
import struct
from pwn import elf, pack, unpack, u64
 
idaapi.msg_clear()
 
elf_raw = elf.ELF("D:/new/frida-agent-example-main/agent/node_modules/libmanxi_copy.so")
elf_patch = elf.ELF(
    "D:/new/frida-agent-example-main/agent/node_modules/libmanxi_copy.so"
)
print(elf_raw.entrypoint)
init_array = None
binary = lief.parse(
    "D:/new/frida-agent-example-main/agent/node_modules/libmanxi_copy.so"
)
for sec in binary.sections:
    if sec.name == ".init_array":
        init_array = sec
# Read Init_array
funcArray = []
funcArray_Address = []
index = 0
while index < init_array.size:
    offset = u64(elf_raw.read(init_array.virtual_address + index, 8))
    funcArray.append(offset)
    index += 8
funcArray.pop()
 
for func in funcArray:
    function = []
    function.append(func)
    function.append(idaapi.get_func(func).end_ea)
    funcArray_Address.append(function)
print(funcArray_Address)

```

提取到地址之后即可进行模拟执行，监听内存读写，由于字符串解密函数通过 OLLVM 进行混淆，所以存在对于状态变量的写入，这会导致对内存写入的 Hook 产生多余的结果，只需要通过判断偏移是否在. data 段或者. rodata 段即可判断是否对加密字符串进行写入，过滤掉不需要修改的内容后把解密内存写入文件中即可 (如果某个函数不能模拟执行只需要从列表中去除即可)

```
import lief
from qiling.os.const import POINTER, UINT, STRING
from qiling import Qiling
from qiling.const import QL_VERBOSE
from qiling.const import QL_INTERCEPT
from qiling import os
from qiling.os.mapper import QlFsMappedObject
import struct
from pwn import elf, pack, unpack, u64
 
soData = lief.parse("libmanxi.so")
 
datasec = None
rodatasec = None
for sec in soData.sections:
    if sec.name == ".data":
        datasec = sec
    elif sec.name == ".rodata":
        rodatasec = sec
 
data_str = ""
elf_patch = elf.ELF(
    "D:/new/frida-agent-example-main/agent/node_modules/libmanxi_copy.so"
)
 
def write_hook(ql: Qiling, access: int, address: int, size: int, value: int):
    Addr = address - base_addr
    if (
        Addr >= datasec.virtual_address
        and Addr <= datasec.virtual_address + datasec.size
    ) or (
        Addr >= rodatasec.virtual_address
        and Addr <= rodatasec.virtual_address + rodatasec.size
    ):
        # if value >= 32 and value <= 128:
        print(
            f"Write Address: {hex(address-base_addr)} Value:{hex(value)} Size:{hex(size)}"
        )
        if size == 1:
            data = ql.pack8(value)
        elif size == 2:
            data = ql.pack16(value)
        elif size == 4:
            data = ql.pack32(value)
        elif size == 8:
            data = ql.pack64(value)
        elf_patch.write(Addr, data)
 
# Read Init_array
funcArray = [
    [22616, 23116],
    [28832, 29460],
    [39664, 40264],
    [42856, 43124],
    [43932, 44020],
    [46660, 46664],
    [55196, 62940],
    [63672, 63756],
    [63756, 63840],
    [63840, 63924],
    [73424, 73428],
    [73516, 73600],
    [87628, 88064],
    [122504, 122588],
    [123876, 124208],
    [124208, 124292],
    [125200, 125288],
    [127652, 128176],
    [131420, 131828],
    [135164, 135876],
    [153412, 154708],
    [164120, 167816],
    [175060, 175332],
    [183120, 183124],
    [183124, 183128],
    [186856, 186940],
    [193592, 193732],
    [215616, 216696],
    [219564, 219648],
    [232264, 232352],
    [232920, 232924],
    [233080, 233352],
    [244248, 253964],
    [255484, 256504],
    [258116, 258424],
    [259408, 262080],
    [262080, 262432],
    [264236, 264696],
    [268872, 269224],
    [269688, 270484],
    [274292, 276648],
    [283864, 284264],
    [286580, 287280],
    [288384, 288656],
    [291236, 291596],
    [292296, 293676],
    [294012, 295272],
    [295272, 295520],
    [295976, 296248],
    [299480, 299788],
    [300236, 307192],
    [307500, 307504],
    [311600, 312672],
    [313304, 313308],
    [316048, 316052],
    [324380, 325624],
    [378800, 380220],
    [382004, 382008],
    [389160, 389520],
    [390088, 390092],
    [392360, 392444],
    [407960, 407964],
    [410060, 410064],
    [414476, 415160],
    [448668, 457116],
    [464888, 464972],
    [469092, 469444],
    [471132, 471388],
    [472156, 472160],
    [472180, 472264],
    [472512, 472596],
    [472596, 472680],
    [472680, 472684],
    [472684, 472772],
    [485160, 489068],
    [491672, 493608],
    [493608, 493692],
    [506756, 506988],
    [520552, 521200],
    [523132, 523532],
    [528344, 528576],
    [533168, 539904],
    [548616, 550592],
    [568596, 569860],
    [573168, 575808],
    [579536, 579792],
    [589616, 589620],
    [610480, 610484],
    [626444, 626916],
    [636500, 636504],
    [641936, 643184],
    [644460, 644464],
    [644816, 644820],
    [42852, 42856],
]
 
# # Init Binary File
ql = Qiling(
    ["D:/new/frida-agent-example-main/agent/node_modules/libmanxi.so"],
    r"D:/new/rootfs/arm64_android",
    verbose=QL_VERBOSE.DISASM,
)
 
base_addr = ql.mem.get_lib_base("libmanxi.so")
str = f"Base Address: {hex(base_addr)}"
ql.hook_mem_write(write_hook)
 
for func in funcArray:
    ql.run(func[0] + base_addr, func[1] + base_addr - 4)
elf_patch.save("aaa.so")

```

解密之后就可以看到很多敏感字符串，但实际上该程序没有检测 Root、Fart，只在程序启动时对 Frida 检测，也就是说可通过 Attach 来进行附加  
![](https://bbs.kanxue.com/upload/attach/202309/942805_YJ5DZY8EAV3UT48.webp)  
![](https://bbs.kanxue.com/upload/attach/202309/942805_HV4XPAXGSTJEQBH.webp)  
![](https://bbs.kanxue.com/upload/attach/202309/942805_NEY2DCMYHVMBHYT.webp)

#### Frida 分析

首先需要绕过启动时的 Frida 检测，奇怪的是我 Hook `dlopen`和`android_dlopen_ext`都无法获取到 libmanxi.so 的加载，但是通过`Process.enumerateModules()`可以得到 so，这也导致了后续难以定位到检测的位置和绕过。于是使用魔改 Frida [https://github.com/Ylarod/Florida/releases](https://github.com/Ylarod/Florida/releases) (可绕过检测)  
简单 Hook 了一下程序中的 SVC 指令，通过`Radare2`的`/adj svc`可导出所有 SVC 指令的地址  
![](https://bbs.kanxue.com/upload/attach/202309/942805_8WYBYPJRHQ6CFV6.webp)  
其中 addr 是导出的结果，这里只针对 openat 进行过滤

```
function hook_svc() {
    console.log("=====Hook SVC=====")
    var base_addr = Module.findBaseAddress("libmanxi.so");
 
    addr.forEach(function (svc) {
        var offset = `0x` + svc.offset.toString(16)
        Interceptor.attach(base_addr.add(offset), {
            onEnter: function (args) {
                if (this.context.x8 == 0x38) {
                    console.log("Openat:" + ptr(this.context.x1).readCString())
                }
            },
            onLeave: function (retval) {
 
            }
        })
    })
}

```

发现其创建线程不断打开 status 文件进而查询 TracerPID 字段  
![](https://bbs.kanxue.com/upload/attach/202309/942805_X6BBGY237C626SJ.webp)

开始脱壳
----

首先尝试 Hook LoadMethod 进而进行脱壳

```
function dump_memory(base, size) {
    Java.perform(function () {
        var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
        var dir = "/data/local/tmp";
        var file_path = dir + "/mydump.so";
        var file_handle = new File(file_path, "w+");
        if (file_handle && file_handle != null) {
            Memory.protect(ptr(base), size, 'rwx');
            var libso_buffer = ptr(base).readByteArray(size);
            file_handle.write(libso_buffer);
            file_handle.flush();
            file_handle.close();
            console.log("[dump]:", file_path);
        }
    });
}
 
var size = 0
var dump_addr = 0
 
function hook_libart() {
    Java.perform(function () {
        var loadMethod = Module.findExportByName("libart.so", "_ZN3art11ClassLinker10LoadMethodERKNS_7DexFileERKNS_13ClassAccessor6MethodENS_6HandleINS_6mirror5ClassEEEPNS_9ArtMethodE")
        var dumped = 0
        Interceptor.attach(loadMethod, {
            onEnter: function (args) {
                if (dumped == 0) {
                    dump_addr = ptr(args[1].add(8).readPointer())
                    size = args[1].add(16).readInt()
                    dump_memory(dump_addr, size)
                    dumped = 1
                }
            },
            onLeave: function (retval) {}
        })
    })
}
 
function main() {
    hook_libart()
 
}
setImmediate(main)

```

然而脱下来的壳依旧只有 Shell 代码，即便我们通过延迟脱壳 (即首先获取 DexFile 的 begin 和 size，之后通过命令行调用脱壳函数) 依旧如此  
![](https://bbs.kanxue.com/upload/attach/202309/942805_9C9UU48UR24HNYR.webp)  
所以尝试别的办法，首先我们知道整体加固的原理是替换 ClassLoader 来加载原 Dex，而 Shell Dex 尾部恰好存在着大量数据，可以猜测这就是被加密的 Dex。由于 Shell 加载完成之后会将程序控制权归还给原程序，所以我们可以 Hook `performLaunchActivity`，在 Activity 启动之前获取系统 ClassLoader 进而得到其加载的 Dex 文件进行 Dump。经过测试，最后一个覆盖的 Dex 文件为原 Dex

```
function dump_memory(base, size) {
    Java.perform(function () {
        var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
        var dir = "/data/local/tmp";
        var file_path = dir + "/mydump.so";
        var file_handle = new File(file_path, "w+");
        if (file_handle && file_handle != null) {
            Memory.protect(ptr(base), size, 'rwx');
            var libso_buffer = ptr(base).readByteArray(size);
            file_handle.write(libso_buffer);
            file_handle.flush();
            file_handle.close();
            console.log("[dump]:", file_path);
        }
    });
}
 
function getDexFile() {
    Hook_Invoke()
    Java.perform(function () {
        var ActivityThread = Java.use("android.app.ActivityThread")
        ActivityThread["performLaunchActivity"].implementation = function (args) {
            var env = Java.vm.getEnv()
            var classloader = this.mInitialApplication.value.getClassLoader()
            var BaseDexClassLoader = Java.use("dalvik.system.BaseDexClassLoader")
            var elementsClass = Java.use("dalvik.system.DexPathList$Element")
            classloader = Java.cast(classloader, BaseDexClassLoader)
            var pathList = classloader.pathList.value
            var elements = pathList.dexElements.value
            console.log(elements)
            //console.log(elements.value)
            for (var i in elements) {
                var element;
                try {
                    element = Java.cast(elements[i], elementsClass);
                } catch (e) {
                    console.log(e)
                }
                //之后需要获取DexFile
                var dexFile = element.dexFile.value
                //getMethod(dexFile, classloader)
                var mCookie = dexFile.mCookie.value
                //$h获取句柄
                var length = env.getArrayLength(mCookie.$h, 0)
                //console.log(length)
                var Array = env.getLongArrayElements(mCookie.$h, 0)
                //第一个不是DexFile
                for (var i = 1; i < length; ++i) {
                    var DexFilePtr = Memory.readPointer(ptr(Array).add(i * 8))
                    var DexFileBegin = ptr(DexFilePtr).add(Process.pointerSize).readPointer()
                    var DexFileSize = ptr(DexFilePtr).add(Process.pointerSize * 2).readU32()
                    console.log(hexdump(DexFileBegin))
                    console.log("Size => " + DexFileSize.toString(16))
                    dump_memory(DexFileBegin,DexFileSize)
                    //dex_begin = DexFileBegin
                    //dex_size = DexFileSize
                }
            }
            return this.performLaunchActivity(arguments[0], arguments[1])
        }
    })
}

```

反编译结果如下，可以发现 onCreate 的方法内容未被填充  
![](https://bbs.kanxue.com/upload/attach/202309/942805_3RMDT6AGNUAJNS8.webp)  
![](https://bbs.kanxue.com/upload/attach/202309/942805_VJUV48GEYTY87MA.webp)  
所以尝试延迟 Dump 的时机，通过 Hook `ArtMethod::Invoke`来 Dump，最终代码如下

```
function Hook_Invoke() {
    var InvokeFunc = Module.findExportByName("libart.so", "_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc")
    Interceptor.attach(InvokeFunc, {
        onEnter: function (args) {
            console.log(args[1])
            // args[0].add(8).readInt().toString(16) == 0
            console.log("Method Idx -> " + args[0].add(8).readInt().toString(16) + " " + args[0].add(12).readInt().toString(16))
            dump_memory(dex_begin, dex_size)
            //dump_memory(dex_begin, dex_size)
        },
        onLeave: function (retval) {}
    })
}
var dex_begin = 0
var dex_size = 0
 
function dump_memory(base, size) {
    Java.perform(function () {
        var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
        var dir = "/data/local/tmp";
        var file_path = dir + "/mydump.so";
        var file_handle = new File(file_path, "w+");
        if (file_handle && file_handle != null) {
            Memory.protect(ptr(base), size, 'rwx');
            var libso_buffer = ptr(base).readByteArray(size);
            file_handle.write(libso_buffer);
            file_handle.flush();
            file_handle.close();
            console.log("[dump]:", file_path);
        }
    });
}
 
function getDexFile() {
 
    Hook_Invoke()
    Java.perform(function () {
 
        var ActivityThread = Java.use("android.app.ActivityThread")
        ActivityThread["performLaunchActivity"].implementation = function (args) {
            var env = Java.vm.getEnv()
            var classloader = this.mInitialApplication.value.getClassLoader()
            var BaseDexClassLoader = Java.use("dalvik.system.BaseDexClassLoader")
            var elementsClass = Java.use("dalvik.system.DexPathList$Element")
            classloader = Java.cast(classloader, BaseDexClassLoader)
            var pathList = classloader.pathList.value
            var elements = pathList.dexElements.value
            console.log(elements)
            //console.log(elements.value)
            for (var i in elements) {
                var element;
                try {
                    element = Java.cast(elements[i], elementsClass);
                } catch (e) {
                    console.log(e)
                }
                //之后需要获取DexFile
                var dexFile = element.dexFile.value
                //getMethod(dexFile, classloader)
                var mCookie = dexFile.mCookie.value
                //$h获取句柄
                var length = env.getArrayLength(mCookie.$h, 0)
                //console.log(length)
                var Array = env.getLongArrayElements(mCookie.$h, 0)
                //第一个不是DexFile
                for (var i = 1; i < length; ++i) {
                    var DexFilePtr = Memory.readPointer(ptr(Array).add(i * 8))
                    var DexFileBegin = ptr(DexFilePtr).add(Process.pointerSize).readPointer()
                    var DexFileSize = ptr(DexFilePtr).add(Process.pointerSize * 2).readU32()
                    console.log(hexdump(DexFileBegin))
                    console.log("Size => " + DexFileSize.toString(16))
                    dex_begin = DexFileBegin
                    dex_size = DexFileSize
                }
            }
            return this.performLaunchActivity(arguments[0], arguments[1])
        }
    })
}

```

反编译效果如下  
![](https://bbs.kanxue.com/upload/attach/202309/942805_FNRMTV63D8XXME5.webp)

  

[[培训] 科锐逆向工程师培训 48 期预科班将于 2023 年 10 月 13 日 正式开班](https://bbs.kanxue.com/thread-51839.htm)

最后于 13 小时前 被 Gift1a 编辑 ，原因：

上传的附件：

*   [app-release.apk](javascript:void(0)) （4.38MB，3 次下载）
*   [dump.dex](javascript:void(0)) （7.64MB，3 次下载）
*   [manxi.apk](javascript:void(0)) （5.55MB，3 次下载）