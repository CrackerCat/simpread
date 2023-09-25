> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-274175.htm)

> [原创]一种通过 “快照” 的方式绕过 So 初始化时的检测

[原创]一种通过 “快照” 的方式绕过 So 初始化时的检测

2022-8-26 17:51 21953

### [原创]一种通过 “快照” 的方式绕过 So 初始化时的检测

 [![](http://passport.kanxue.com/upload/avatar/439/855439.png?1693397386)](user-home-855439.htm) [FANGG3](user-home-855439.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 2022-8-26 17:51  21953

说明
==

本文的思路是很久以前就有的，但是找不到一个合适的工具 “拍摄快照” 一直都没有成功实现，[直到看到这篇文章](https://blog.seeflower.dev/archives/171/)才成功实现，感谢该文作者 seeflower。如果原作者认为本文章不合适，请联系我删除，谢谢啦～

背景
==

每次在虚拟机中打开 ida，逆向一堆 JniOnLoad 或者. iniarray 段被混淆的代码时。我都想着有没有一种方法，免去这些初始化流程（当然这些初始化操作是厂家的重点防护对象）。

 

有一天，工作完之后，手头上的 jnionload 还没有分析完，将 VMware 中运行的机器设置暂停, 明天再分析。突然我想到，既然 VMware 能通过快照这种方式保存虚拟机的运行时状态，即使关机也可以恢复。那我能不能通过对 Android APP”拍摄 “一份” 快照“（SnapShot），正好停在我想逆向的函数前，修改入参，然后反复运行这份快照，绕过初始化过程、绕过各种检测呢？

 

思路有了，接下来就是寻找工具，怎么去实现了。

工具
==

1.  需要虚拟机载体：Unidbg 似乎是一个好的选择
2.  需要快照：Android APP 运行时的内存 dump
3.  需要相机：lldb 注：其实没看到 seelflower 的文章时，相机一开始选择的是 frida 去 dump 内存，但是研究了一段时间之后，发现 frida 有太多不足了，首先无法获取特殊状态寄存器的值，没有断点机制（frida 运行时似乎是单线程），通过 mprotct 修改内存 Readable 容易发生 crash。（也有可能是我能力不够，frida 脚本放在附件中，可以尝试用 frida 实现，最大的难点是无法实现断点，中断 app 状态）

怎么拍
===

为了能让 Unidbg 成功载入快照，拍摄内容：

1.  App 运行内存
2.  内存段的地址、权限信息
3.  断点中断时的寄存器信息
4.  SO 符号表 (包括中断时地址)

有了目标，现在开始实现。

 

​ 第一步：装载 lldb（相机）

```
#获取lldb android 服务端
$NDK_PATH/toolchains/llvm/prebuilt/linux-x86_64/lib64/clang/9.0.8/lib/linux/aarch64/lldb-server
 
#ADB导入设备、赋予权限
adb push lldb-server /data/local/tmp/
adb shell
cd /data/local/tmp
chmod 755 lldb-server
 
#开启服务
./lldb-server p --server --listen unix-abstract:///data/local/tmp/debug.sock
 
#pc端连接android
$ lldb
 platform select remote-android
 platform connect unix-abstract-connect:///data/local/tmp/debug.sock

```

第二步：编写 dump 脚本

 

​ 内存段（Segment）、寄存器（Register）的 lldb Dump 脚本

```
from asyncio.log import logger
from datetime import datetime
import hashlib
import json
from pathlib import Path
from typing import TYPE_CHECKING, Dict, List
import sys
import zlib
import os
 
if TYPE_CHECKING:
    from lldb import *
 
import lldb
black_list = {
    'startswith': ['/dev', '/system/fonts', '/dmabuf'],
    'endswith': ['(deleted)', '.apk', '.odex', '.vdex', '.dex', '.jar', '.art', '.oat', '.art]'],
    'includes': [],
}
dump_path = Path("./dump/")
 
 
def __lldb_init_module(debugger: 'SBDebugger', internal_dict: dict):
    debugger.HandleCommand('command script add -f lldb_dumper.dumpMem dumpmem')
 
def dumpMem(debugger: 'SBDebugger', command: str, exe_ctx: 'SBExecutionContext', result: 'SBCommandReturnObject', internal_dict: dict):
    target = exe_ctx.GetTarget() # type: SBTarget
    arch_long = dump_arch_info(target)
    frame = exe_ctx.GetFrame() # type: SBFrame
    regs = dump_regs(frame)
 
    target = exe_ctx.GetTarget() # type: SBTarget
    sections = dump_memory_info(target)
 
    max_seg_size = 64 * 1024 * 1024
# dump内存
    process = exe_ctx.GetProcess() # type: SBProcess
    segments = dump_memory(process, dump_path, black_list, max_seg_size)
    (dump_path / "regs.json").write_text(json.dumps(regs))
    (dump_path / "sections.json").write_text(json.dumps(sections))
    (dump_path / "segments.json").write_text(json.dumps(segments))
 
 
def dump_arch_info(target: 'SBTarget'):
    triple = target.GetTriple()
    print(f'[dump_arch_info] triple => {triple}')
    # 'aarch64', 'unknown', 'linux', 'android'
    arch, vendor, sys, abi = triple.split('-')
    if arch == 'aarch64' or arch == 'arm64':
        return 'arm64le'
    elif arch == 'aarch64_be':
        return 'arm64be'
    elif arch == 'armeb':
        return 'armbe'
    elif arch == 'arm':
        return 'armle'
    else:
        return ''
 
def dump_regs(frame: 'SBFrame'):
    regs = {} # type: Dict[str, int]
    registers = None # type: List[SBValue]
    for registers in frame.GetRegisters():
        # - General Purpose Registers
        # - Floating Point Registers
        print(f'registers name => {registers.GetName()}')
        for register in registers:
            register_name = register.GetName()
            register.SetFormat(lldb.eFormatHex)
            register_value = register.GetValue()
            regs[register_name] = register_value
    print(f'regs => {json.dumps(regs, ensure_ascii=False, indent=4)}')
    return regs
 
 
def dump_memory_info(target: 'SBTarget'):
    logger.debug('start dump_memory_info')
    sections = []
    # 先查找全部分段信息
    for module in target.module_iter():
        module: SBModule
        for section in module.section_iter():
            section: SBSection
            module_name = module.file.GetFilename()
            start, end, size, name = get_section_info(target, section)
            section_info = {
                'module': module_name,
                'start': start,
                'end': end,
                'size': size,
                'name': name,
            }
            # size 好像有负数的情况 不知道是什么情况
            print(f'Appending: {name}')
            sections.append(section_info)
    return sections
 
def get_section_info(tar, sec):
    name = sec.name if sec.name is not None else ""
    if sec.GetParent().name is not None:
        name = sec.GetParent().name + "." + sec.name
    module_name = sec.addr.module.file.GetFilename()
    module_name = module_name if module_name is not None else ""
    long_name = module_name + "." + name
    return sec.GetLoadAddress(tar), (sec.GetLoadAddress(tar) + sec.size), sec.size, long_name
 
def dump_memory(process: 'SBProcess', dump_path: Path, black_list: Dict[str, List[str]], max_seg_size: int):
    logger.debug('start dump memory')
    memory_list = []
    mem_info = lldb.SBMemoryRegionInfo()
    start_addr = -1
    next_region_addr = 0
    while next_region_addr > start_addr:
        # 从内存起始位置开始获取内存信息
        err = process.GetMemoryRegionInfo(next_region_addr, mem_info) # type: SBError
        if not err.success:
            logger.warning(f'GetMemoryRegionInfo failed, {err}, break')
            break
        # 获取当前位置的结尾地址
        next_region_addr = mem_info.GetRegionEnd()
        # 如果超出上限 结束遍历
        if next_region_addr >= sys.maxsize:
            logger.info(f'next_region_addr:0x{next_region_addr:x} >= sys.maxsize, break')
            break
        # 获取当前这块内存的起始地址和结尾地址
        start = mem_info.GetRegionBase()
        end = mem_info.GetRegionEnd()
        # 很多内存块没有名字 预设一个
        region_name = 'UNKNOWN'
        # 记录分配了的内存
        if mem_info.IsMapped():
            name = mem_info.GetName()
            if name is None:
                name = ''
            mem_info_obj = {
                'start': start,
                'end': end,
                'name': name,
                'permissions': {
                    'r': mem_info.IsReadable(),
                    'w': mem_info.IsWritable(),
                    'x': mem_info.IsExecutable(),
                },
                'content_file': '',
            }
            memory_list.append(mem_info_obj)
    # 开始正式dump
    for seg_info in memory_list:
        try:
            start_addr = seg_info['start'] # type: int
            end_addr = seg_info['end'] # type: int
            region_name = seg_info['name'] # type: str
            permissions = seg_info['permissions'] # type: Dict[str, bool]
 
            # 跳过不可读 之后考虑下是不是能修改权限再读
            if seg_info['permissions']['r'] is False:
                logger.warning(f'Skip dump {region_name} permissions => {permissions}')
                continue
 
            # 超过预设大小的 跳过dump
            predicted_size = end_addr - start_addr
            if predicted_size > max_seg_size:
                logger.warning(f'Skip dump {region_name} size:0x{predicted_size:x}')
                continue
 
            skip_dump = False
 
            for rule in black_list['startswith']:
                if region_name.startswith(rule):
                    skip_dump = True
                    logger.warning(f'Skip dump {region_name} hit startswith rule:{rule}')
            if skip_dump: continue
 
            for rule in black_list['endswith']:
                if region_name.endswith(rule):
                    skip_dump = True
                    logger.warning(f'Skip dump {region_name} hit endswith rule:{rule}')
            if skip_dump: continue
 
            for rule in black_list['includes']:
                if rule in region_name:
                    skip_dump = True
                    logger.warning(f'Skip dump {region_name} hit includes rule:{rule}')
            if skip_dump: continue
 
            # 开始读取内存
            ts = datetime.now()
            err = lldb.SBError()
            seg_content = process.ReadMemory(start_addr, predicted_size, err)
            tm = (datetime.now() - ts).total_seconds()
            # 读取成功的才写入本地文件 并计算md5
            # 内存里面可能很多地方是0 所以压缩写入文件 减少占用
            if seg_content is None:
                logger.debug(f'Segment empty: @0x{start_addr:016x} {region_name} => {err}')
            else:
                logger.info(f'Dumping @0x{start_addr:016x} {tm:.2f}s size:0x{len(seg_content):x}: {region_name} {permissions}')
                compressed_seg_content = zlib.compress(seg_content)
                md5_sum = hashlib.md5(compressed_seg_content).hexdigest() + '.bin'
                seg_info['content_file'] = md5_sum
                (dump_path / md5_sum).write_bytes(compressed_seg_content)
        except Exception as e:
            # 这里好像不会出现异常 因为前面有 SBError 处理了 不过还是保留
            logger.error(f'Exception reading segment {region_name}', exc_info=e)
 
    return memory_list

```

​ 符号表（SymTab）的 lldb Python 脚本：

```
from audioop import add
import json
from pathlib import Path
from typing import TYPE_CHECKING
 
 
if TYPE_CHECKING:
    from lldb import *
 
import lldb
symbolList = []
 
def __lldb_init_module(debugger: 'SBDebugger', internal_dict: dict):
    debugger.HandleCommand('command script add -f lldb_symb.dumpsym dumpsym')
 
def dumpsym(debugger: 'SBDebugger', command: str, exe_ctx: 'SBExecutionContext', result: 'SBCommandReturnObject', internal_dict: dict):
    target = exe_ctx.GetTarget()
    m = target.modules
 
    for module in m:
        module:SBModule
        module_name = module.file.GetFilename()
        if command in module_name:
            symbols_array = module.get_symbols_array()
 
            for symbol in symbols_array:
 
                symbolList.append(symInfo(symbol,target))
            print(json.dumps(symbolList))
            dump_path = Path("./")
            f_name = command+"_sym.json"
            (dump_path / f_name).write_text(json.dumps(symbolList))
 
def symInfo(symbol,target):
    return {"name":symbol.name,"addr":symbol.addr.GetLoadAddress(target)}

```

​ 第三步：按下快门

 

​ 将脚本导入到 lldb：

 

​ command script import lldb_dumper.py

 

​ command script import lldb_symb.py

 

​ 附加到目标 so：

```
#env: lldb
#附加so
attach ${pid}
br set -s ${so_name} -n ${函数名} # 或者使用 -a ${相对偏移}

```

​ 触发断点以后，通过以下命令得到快照文件：

```
dumpsym  ${so_name}#dump符号表
dumpmem  #dump内存信息

```

生成的文件有：

 

​ 符号表 json 文件：libxxxxxxx.so_sym.json

 

​ dump 文件夹：./dump/*

 

​ Register json 文件：./dump/regs.json

 

​ Segment json 文件：./dump/segments.json

 

​ Section json 文件：./dump/sections.json

怎么加载快照
======

Unidbg 是基于 Unicorn 增加 JNI 支持的 Unicorn Plus Pro Ultra（就是牛皮哦～），所以 Unidbg 也通过了一系列 Api 对寄存器和内存操作：

 

**恢复寄存器状态：**

```
String js = FileUtil.readString("/Users/fangg3/WorkSpace/tools/androidReverse/unidbg-0.9.6/unidbg-android/src/test/java/com/lldb/dumper/dump/regs.json", StandardCharsets.UTF_8);
JSONObject json = new JSONObject(js);
Backend backend = emulator.getBackend();
Memory memory = emulator.getMemory();
backend.reg_write(Arm64Const.UC_ARM64_REG_X0, Long.decode(json.getStr("x0")));
backend.reg_write(Arm64Const.UC_ARM64_REG_X1, Long.decode(json.getStr("x1")));
backend.reg_write(Arm64Const.UC_ARM64_REG_X2, Long.decode(json.getStr("x2")));
        ...
        backend.reg_write(Arm64Const.UC_ARM64_REG_X28, Long.decode(json.getStr("x28")));
 
backend.reg_write(Arm64Const.UC_ARM64_REG_LR, Long.decode(json.getStr("lr")));
backend.reg_write(Arm64Const.UC_ARM64_REG_FP, Long.decode(json.getStr("fp")));
backend.reg_write(Arm64Const.UC_ARM64_REG_PC, Long.decode(json.getStr("pc")));
backend.reg_write(Arm64Const.UC_ARM64_REG_SP, Long.decode(json.getStr("sp")));
 
backend.reg_write(ArmConst.UC_ARM_REG_CPSR, Long.decode(json.getStr("cpsr")));
 
backend.reg_write(Arm64Const.UC_ARM64_REG_CPACR_EL1, 0x300000L);
//backend.reg_write(Arm64Const.UC_ARM64_REG_TPIDR_EL0, 0x000000797d778588L);
        //TPIDR_EL0 这个寄存器很特殊，存储了当前线程信息，需要lldb下断点获取

```

**恢复内存状态：**

 

因为执行快照时，并不需要一些内存段，这里过滤一下：

```
Boolean isWhite(String name) {
    String[] list = {
            "libxxxxxx.so", //目标so
            "libc.so",  // 一般都要加载的
            "[stack:14414]",// 堆栈相关 这里14414不固定 可以看看sp指向哪里，在segments.json里找
            "[anon:.bss]",
            "[anon:bionic TLS]" //TPIDR_EL0 寄存器会指向这里
                                                  //还有其他根据实际情况添加
    };
    for (String a : list) {
        if (name.contains(a))
            return true;
    }
 
    return false;
}

```

这里内存文件已经 zlib 压缩，需要解压后再写入内存中。

```
    int UNICORN_PAGE_SIZE = 0x1000;
 
    private long align_page_down(long x) {
        return x & ~(UNICORN_PAGE_SIZE - 1);
    }
 
    private long align_page_up(long x) {
        return (x + UNICORN_PAGE_SIZE - 1) & ~(UNICORN_PAGE_SIZE - 1);
    }
 
    private void map_segment(long address, long size, int perms) {
 
        long mem_start = address;
        long mem_end = address + size;
        long mem_start_aligned = align_page_down(mem_start);
        long mem_end_aligned = align_page_up(mem_end);
 
        if (mem_start_aligned < mem_end_aligned) {
            emulator.getBackend().mem_map(mem_start_aligned, mem_end_aligned - mem_start_aligned, perms);
        }
    }   
 
void loadMemory() {
        String js = FileUtil.readString("/Users/fangg3/WorkSpace/tools/androidReverse/unidbg-0.9.6/unidbg-android/src/test/java/com/lldb/dumper/dump/segments.json", StandardCharsets.UTF_8);
        JSONArray json = new JSONArray(js);
        Memory memory = emulator.getMemory();
        System.out.println(json.size());
        try {
 
 
            for (int i = 0; i < json.size(); i++) {
 
                JSONObject seg_json = new JSONObject(json.get(i));
                String name = seg_json.getStr("name");
                Long start = seg_json.getLong("start");
                Long end = seg_json.getLong("end");
                String filename = seg_json.getStr("content_file");
                JSONObject permissions = seg_json.getJSONObject("permissions");
                Boolean r = permissions.getBool("r");
                Boolean w = permissions.getBool("w");
                Boolean x = permissions.getBool("x");
 
                if (Objects.equals(filename, "") || filename == null) {
                    continue;
                }
                if (!isWhite(name)) {
 
                    continue;
                }
                int per = 0;
                if (r) {
                    per |= UnicornConst.UC_PROT_READ;
                }
                if (w) {
                    per |= UnicornConst.UC_PROT_WRITE;
                }
                if (x) {
                    per |= UnicornConst.UC_PROT_EXEC;
                }
                map_segment(start, end - start, per);
                System.out.println(filename + " " + name);
                byte[] content_file = FileUtil.readBytes("/Users/fangg3/WorkSpace/tools/androidReverse/unidbg-0.9.6/unidbg-android/src/test/java/com/lldb/dumper/dump/" + filename);
                content_file = ZipUtil.unZlib(content_file);
                emulator.getBackend().mem_write(start, content_file);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("load memory over");
    }

```

**运行快照**：

```
List list = new ArrayList();
       list.add(vm.getJNIEnv());
   DvmClass NativeApiobj = vm.resolveClass("com/mihoyo/hyperion/net/aaaaa");
       list.add(NativeApiobj.hashCode());
       list.add(vm.addLocalObject(new
 
   StringObject(vm,"123123123")));
       list.add(vm.addLocalObject(new
 
   StringObject(vm,"123123123")));
 
   Number result = Module.emulateFunction(emulator, ctx_addr, list.toArray());
 
   //        获取返回结果
   String sign_str = (String) vm.getObject(result.intValue()).getValue();
       System.out.println("sign_str="+sign_str); 
```

一些优化
====

有时候，会出现 Memory unmap 的情况，这个时候，在 Segments.json，找到对应地址，将 name 添加到 whileList 里面即可。

 

用 unidbg 的 libc 替换内存段中的 libc，便于调试：

```
//载入libc.so
...
vm.loadLibrary(new File("/Users/fangg3/WorkSpace/tools/androidReverse/unidbg-0.9.6/unidbg-android/src/main/resources/android/sdk23/lib64/libc.so"), false);
Module libc = emulator.getMemory().findModule("libc.so");
...
//过滤符号
      public static String getSymName(long addr) {
        JSONArray jarr = (JSONArray) parse;
        for (int i = 0; i < jarr.size(); i++) {
            JSONObject jsonObject = jarr.getJSONObject(i);
            if (jsonObject.getLong("addr") == addr) {
                return jsonObject.getStr("name");
            }
        }
        return "";
    }
 
//添加hook 筛选运行地址 pc寄存器指向下一条运行的地址
...
        emulator.getBackend().hook_add_new(new CodeHook() {
            @Override
            public void hook(Backend backend, long address, int size, Object user) {
                long pc_value = emulator.getBackend().reg_read(Arm64Const.UC_ARM64_REG_PC).longValue();
                   if (sym_json.contains(String.valueOf(pc_value))) {
                    String symName = getSymName(address);
                    Symbol symbolByName = libc.findSymbolByName(symName);
                    if (symbolByName != null) {
                        emulator.getBackend().reg_write(Arm64Const.UC_ARM64_REG_PC, symbolByName.getAddress());
                        System.out.println("call " + symName);
                    }
 
                 }
 
            }

```

一些大问题
=====

有大佬能帮忙看看这些问题是什么原因么：

1.  arm32 类型 So 运行快照时，有时会出现 arm/thumb 指令切换问题。
2.  某些 So 运行的时候，函数已经结束了，Unidbg 还继续往下跑，不返回结果。
3.  反正 Bug 还蛮多的....

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#程序开发](forum-161-1-124.htm)

上传的附件：

*   [dumpmem.js](javascript:void(0)) （5.14kb，35 次下载）