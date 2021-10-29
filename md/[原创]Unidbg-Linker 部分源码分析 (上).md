> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270032.htm)

> [原创]Unidbg-Linker 部分源码分析 (上)

Unidbg-Linker 部分源码分析 (上)
========================

*   [Unidbg-Linker 部分源码分析 (上)](#unidbg-linker部分源码分析上)
    *   [概要](#概要)
    *   [init](#init)
    *   [load](#load)
    *   [总结](#总结)

概要
--

我们有了 Android Linker 源码部分的解析，还学习了 Unicorn 的详细使用。如果没有看过之前文章的同学可以看下之前的文章哦。今天我们就来看下 Unidbg 是如何将一个 So 加载且跑起来的

 

在 Unidbg 中，我们想要加载一个 So 一般都是通过

```
DalvikModule dalvikModule = vm.loadLibrary(new File("so_path"), true);

```

来加载一个 So 文件，通过这个方法，我们最终会发现 Unidbg 调用了 AndroidElfLoader 类的 loadInternal 方法。AndroidElfLoader 类也就充当了 Linker 的角色，所以我们今天就来分析一下这个类

init
----

我们先来分析一下 AndroidElfLoader 的构造方法

```
public AndroidElfLoader(Emulator emulator, UnixSyscallHandler syscallHandler) {
     // 调用父类构造方法，初始化emulator和syscallHandler字段
    super(emulator, syscallHandler);
 
    // 初始化SP
    stackSize = STACK_SIZE_OF_PAGE * emulator.getPageAlign();
 
    // 将栈空间mem_map映射，因为在Backend(Unicorn)中，所有需要用的内存都需要先进行映射才能够进行使用，大小就是STACK_SIZE_OF_PAGE * emulator.getPageAlign()，可读可写权限
    backend.mem_map(STACK_BASE - stackSize, stackSize, UnicornConst.UC_PROT_READ | UnicornConst.UC_PROT_WRITE);
 
    // 设置SP寄存器
    setStackPoint(STACK_BASE);
    // 初始化TLS(线程局部存储相关)，在libc一些系统库中是有线程局部变量的，如errno等。这里就做了相关的协处理器的初始化操作
    this.environ = initializeTLS(new String[] {
            "ANDROID_DATA=/data",
            "ANDROID_ROOT=/system"
    });
    this.setErrno(0);
} 
```

上面的注释也写的很清楚了，再总结一下 AndroidElfLoader 初始化做了什么

*   在类中保存了 emulator 和 syscallHandler 字段，其中 emulator 就是一个模拟器对象。syscallHandler 是 linux 系统调用相关的处理对象
*   初始化了栈空间和设置 SP 寄存器
*   初始化 TLS 操作

对 TLS 感兴趣可以阅读以下源码

> http://androidxref.com/4.4.4_r1/xref/bionic/libc/bionic/libc_init_common.cpp#111  
> http://androidxref.com/4.4.4_r1/xref/bionic/libc/bionic/pthread_internals.cpp#66  
> http://androidxref.com/4.4.4_r1/xref/bionic/libc/private/bionic_tls.h#90

load
----

那么当我们调用了 loadLibrary 方法后，在 Android 中最终调用了 AndroidElfLoader 类下的 loadInternal 方法，我们来分析这个方法，该类的初始化我们已经分析过了

```
protected final LinuxModule loadInternal(LibraryFile libraryFile, boolean forceCallInit) {
    // File对象被封装为了LibraryFile对象
    try {
        // 接着调用了loadInternal(重载)方法，继续加载流程
        LinuxModule module = loadInternal(libraryFile);
        // 处理符号(关于重定位)
        resolveSymbols(!forceCallInit);
        // callInitFunction默认true
        if (callInitFunction || forceCallInit) {
            // 调用初始化函数
            for (LinuxModule m : modules.values().toArray(new LinuxModule[0])) {
                boolean forceCall = (forceCallInit && m == module) || m.isForceCallInit();
                if (callInitFunction) {
                    m.callInitFunction(emulator, forceCall);
                } else if (forceCall) {
                    m.callInitFunction(emulator, true);
                }
                m.initFunctionList.clear();
            }
        }
        // 添加引用计数
        module.addReferenceCount();
        return module;
    } catch (IOException e) {
        throw new IllegalStateException(e);
    }
}

```

上面的函数接着调用了重载 loadInternal 方法继续加载，加载完成会返回一个 LinuxModule 对象，该对象就保存了该 So 文件加载后的信息，接着处理了符号和调用初始化函数

 

我们接着来看重载 loadInternal 方法，代码比较长

```
private LinuxModule loadInternal(LibraryFile libraryFile) throws IOException {
    // 将我们的So文件让ElfFile类去解析，这个ElfFile是jelf库经过凯神改装过的，可以帮助解析Elf文件
    final ElfFile elfFile = ElfFile.fromBytes(libraryFile.mapBuffer());
 
    // ... 判断文件是否合法，32位模拟器是否加载了64位So等待
 
    long start = System.currentTimeMillis();
 
    // 获取当前So的最大虚拟地址和页对齐align参数
    long bound_high = 0;
    long align = 0;
    for (int i = 0; i < elfFile.num_ph; i++) {
        ElfSegment ph = elfFile.getProgramHeader(i);
        if (ph.type == ElfSegment.PT_LOAD && ph.mem_size > 0) {
            // 遍历所有mem_size>0的PT_LOAD段
            long high = ph.virtual_address + ph.mem_size;
            // 寻找bound_high最大值
            if (bound_high < high) {
                bound_high = high;
            }
            // 寻找alignment最大值
            if (ph.alignment > align) {
                align = ph.alignment;
            }
        }
    }
 
    ElfDynamicStructure dynamicStructure = null;
 
    // 从获取到的So指定的alignment和默认的PageAlign取一个最大值，一般拿到的就是4K大小
    final long baseAlign = Math.max(emulator.getPageAlign(), align);
 
    // 根据baseAlign来计算该So的加载地址。初始地址0x40000000L
    final long load_base = ((mmapBaseAddress - 1) / baseAlign + 1) * baseAlign;
 
    // 这个就相当于Linker在计算load_size，但Unidbg中将所有So的最小虚拟地址默认为0
    // 这里有改进空间对吧，因为Linker中为了防止内存浪费，出现了一个load_bias_字段
    // 但是出于目的不用，Unidbg的目的是让二进制文件跑起来
    long size = ARM.align(0, bound_high, baseAlign).size;
 
    // 设置加载下个So的mmapBaseAddress
    setMMapBaseAddress(load_base + size);
 
    // MemRegion存储了哪块内存对应了哪个So文件
    final List regions = new ArrayList<>(5);
    MemoizedObject armExIdx = null;
    MemoizedObject ehFrameHeader = null;
    Alignment lastAlignment = null;
 
    // 再次遍历所有段
    for (int i = 0; i < elfFile.num_ph; i++) {
        ElfSegment ph = elfFile.getProgramHeader(i);
        switch (ph.type) {
            case ElfSegment.PT_LOAD:
                // PT_LOAD段
                // 获取该段在内存中对应的操作权限，如该段未指定，设置满权限(一般不会出现这种情况)
                int prot = get_segment_protection(ph.flags);
                if (prot == UnicornConst.UC_PROT_NONE) {
                    prot = UnicornConst.UC_PROT_ALL;
                }
                // 该段在内存中的起始地址
                final long begin = load_base + ph.virtual_address;
 
                // 计算该段在内存中的位置和大小
                Alignment check = ARM.align(begin, ph.mem_size, Math.max(emulator.getPageAlign(), ph.alignment));
 
                // 获取上一个内存快
                final int regionSize = regions.size();
                MemRegion last = regionSize <= 0 ? null : regions.get(regionSize - 1);
 
                // 处理两个段之间重叠部分？未找到案例
                MemRegion overall = null;
                if (last != null && check.address >= last.begin && check.address < last.end) {
                    overall = last;
                }
 
                if (overall != null) {
                    // 处理重叠段，应该为特殊情况，正常都会走下面else分支
                    long overallSize = overall.end - check.address;
                    backend.mem_protect(check.address, overallSize, overall.perms | prot);
                    if (ph.mem_size > overallSize) {
                        Alignment alignment = this.mem_map(begin + overallSize, ph.mem_size - overallSize, prot, libraryFile.getName(), Math.max(emulator.getPageAlign(), ph.alignment));
                        regions.add(new MemRegion(alignment.address, alignment.address + alignment.size, prot, libraryFile, ph.virtual_address));
                        if (lastAlignment != null) {
                            throw new UnsupportedOperationException();
                        }
                        lastAlignment = alignment;
                    }
                } else {
                    // 将该PT_LOAD段指示的内存大小进行映射
                    Alignment alignment = this.mem_map(begin, ph.mem_size, prot, libraryFile.getName(), Math.max(emulator.getPageAlign(), ph.alignment));
 
                    // 添加一块MemRegion
                    regions.add(new MemRegion(alignment.address, alignment.address + alignment.size, prot, libraryFile, ph.virtual_address));
                    if (lastAlignment != null) {
                        // 处理该段与上一个段之间的空隙，置0
                        long base = lastAlignment.address + lastAlignment.size;
                        long off = alignment.address - base;
                        if (off < 0) {
                            throw new IllegalStateException();
                        }
                        if (off > 0) {
                            backend.mem_map(base, off, UnicornConst.UC_PROT_NONE);
                            if (memoryMap.put(base, new MemoryMap(base, (int) off, UnicornConst.UC_PROT_NONE)) != null) {
                                log.warn("mem_map replace exists memory map base=" + Long.toHexString(base));
                            }
                        }
                    }
                    lastAlignment = alignment;
                }
 
                // 将该段对应的数据写入进已经映射好的内存
                ph.getPtLoadData().writeTo(pointer(begin));
                break;
            case ElfSegment.PT_DYNAMIC:
                // DYNAMIC段
                dynamicStructure = ph.getDynamicStructure();
                break;
            case ElfSegment.PT_INTERP:
                // INTERP段指定了解释器位置，在So中没用
                if (log.isDebugEnabled()) {
                    log.debug("[" + libraryFile.getName() + "]interp=" + ph.getInterpreter());
                }
                break;
            case ElfSegment.PT_GNU_EH_FRAME:
                // 没分析过，未知TODO
                ehFrameHeader = ph.getEhFrameHeader();
                break;
            case ElfSegment.PT_ARM_EXIDX:
                // 异常相关的段
                armExIdx = ph.getARMExIdxData();
                break;
            default:
                if (log.isDebugEnabled()) {
                    log.debug("[" + libraryFile.getName() + "]segment type=0x" + Integer.toHexString(ph.type) + ", offset=0x" + Long.toHexString(ph.offset));
                }
                break;
        }
    }
 
 
    // 此时，该So中的有用的段信息已经处理完毕
    // 该加载到内存的已经加载到内存
    // 该置空的内存也已置空
    // 动态段、异常段、PT_GNU_EH_FRAME段的信息已经保存下来，继续看接下来的处理
 
    // 动态段是必须有的
    if (dynamicStructure == null) {
        throw new IllegalStateException("dynamicStructure is empty.");
    }
    // 此SoName是动态段中的tag为SO_NAME指定的内容，而且Unidbg中的Log也是基于这个SoName打印的
    // 如果该内容为空，才会使用文件名。这也就是有的同学会问为什么我加载的是libxxxxx.so，而日志输出libyyyyyy.so呢
    final String soName = dynamicStructure.getSOName(libraryFile.getName());
 
    // 下面处理So中的依赖库
    Map neededLibraries = new HashMap<>();
 
    // dynamicStructure.getNeededLibraries()，这个方法是Unidbg改写jelf库加上的方法，会获取到所有的依赖库的名字
    for (String neededLibrary : dynamicStructure.getNeededLibraries()) {
        if (log.isDebugEnabled()) {
            log.debug(soName + " need dependency " + neededLibrary);
        }
 
        // modules字段保存了所有已经加载过的库，这里就是在寻找是否该So已经被加载过
        LinuxModule loaded = modules.get(neededLibrary);
        if (loaded != null) {
            // 如果加载过了，添加引用计数、放到neededLibraries变量
            loaded.addReferenceCount();
            neededLibraries.put(FilenameUtils.getBaseName(loaded.name), loaded);
            continue;
        }
        // 如果依赖还没有被加载过，就开始寻找这个依赖文件在哪，先在当前So的路径下找
        LibraryFile neededLibraryFile = libraryFile.resolveLibrary(emulator, neededLibrary);
 
        // 如果当前路径下没有找到，就去找library解析器去找
        if (libraryResolver != null && neededLibraryFile == null) {
            neededLibraryFile = libraryResolver.resolveLibrary(emulator, neededLibrary);
        }
 
        if (neededLibraryFile != null) {
            // 大吉大利，So找到啦，就会在这里加载
            LinuxModule needed = loadInternal(neededLibraryFile);
            needed.addReferenceCount();
            neededLibraries.put(FilenameUtils.getBaseName(needed.name), needed);
        } else {
            log.info(soName + " load dependency " + neededLibrary + " failed");
        }
    }
 
    // 到这里，该So所依赖的So也被加载进来了
 
    // 下面这个循环会处理未解决(符号为0特殊情况)的重定位，进行二次重定位，极少数能成功，如果确定没用可以注释掉
    for (LinuxModule module : modules.values()) {
        for (Iterator iterator = module.getUnresolvedSymbol().iterator(); iterator.hasNext(); ) {
            ModuleSymbol moduleSymbol = iterator.next();
            ModuleSymbol resolved = moduleSymbol.resolve(module.getNeededLibraries(), false, hookListeners, emulator.getSvcMemory());
            if (resolved != null) {
                if (log.isDebugEnabled()) {
                    log.debug("[" + moduleSymbol.soName + "]" + moduleSymbol.symbol.getName() + " symbol resolved to " + resolved.toSoName);
                }
                resolved.relocation(emulator);
                iterator.remove();
            }
        }
    }
 
 
    // 下面开始处理重定位
    List list = new ArrayList<>();
    for (MemoizedObject object : dynamicStructure.getRelocations()) {
        // 遍历So中所有的重定位信息
        ElfRelocation relocation = object.getValue();
 
        // 拿到重定位类型
        final int type = relocation.type();
        if (type == 0) {
            log.warn("Unhandled relocation type " + type);
            continue;
        }
 
        // 拿到重定位项指定的符号信息
        ElfSymbol symbol = relocation.sym() == 0 ? null : relocation.symbol();
        long sym_value = symbol != null ? symbol.value : 0;
 
        // 计算需要重定位的位置
        Pointer relocationAddr = UnidbgPointer.pointer(emulator, load_base + relocation.offset());
        assert relocationAddr != null;
 
        Log log = LogFactory.getLog("com.github.unidbg.linux." + soName);
        if (log.isDebugEnabled()) {
            log.debug("symbol=" + symbol + ", type=" + type + ", relocationAddr=" + relocationAddr + ", offset=0x" + Long.toHexString(relocation.offset()) + ", addend=" + relocation.addend() + ", sym=" + relocation.sym() + ", android=" + relocation.isAndroid());
        }
 
        ModuleSymbol moduleSymbol;
        // 根据重定位类型进行不同的处理，下面包含了32位/64位下的重定位处理
        switch (type) {
            case ARMEmulator.R_ARM_ABS32: {
                int offset = relocationAddr.getInt(0);
                moduleSymbol = resolveSymbol(load_base, symbol, relocationAddr, soName, neededLibraries.values(), offset);
                if (moduleSymbol == null) {
                    // 不能当即处理的，添加到list，后面再处理
                    list.add(new ModuleSymbol(soName, load_base, symbol, relocationAddr, null, offset));
                } else {
                    moduleSymbol.relocation(emulator);
                }
                break;
            }
            case ARMEmulator.R_AARCH64_ABS64: {
                long offset = relocationAddr.getLong(0) + relocation.addend();
                moduleSymbol = resolveSymbol(load_base, symbol, relocationAddr, soName, neededLibraries.values(), offset);
                if (moduleSymbol == null) {
                    list.add(new ModuleSymbol(soName, load_base, symbol, relocationAddr, null, offset));
                } else {
                    moduleSymbol.relocation(emulator);
                }
                break;
            }
            case ARMEmulator.R_ARM_RELATIVE: {
                int offset = relocationAddr.getInt(0);
                if (sym_value == 0) {
                    relocationAddr.setInt(0, (int) load_base + offset);
                } else {
                    throw new IllegalStateException("sym_value=0x" + Long.toHexString(sym_value));
                }
                break;
            }
            case ARMEmulator.R_AARCH64_RELATIVE:
                if (sym_value == 0) {
                    relocationAddr.setLong(0, load_base + relocation.addend());
                } else {
                    throw new IllegalStateException("sym_value=0x" + Long.toHexString(sym_value));
                }
                break;
            case ARMEmulator.R_ARM_GLOB_DAT:
            case ARMEmulator.R_ARM_JUMP_SLOT:
                moduleSymbol = resolveSymbol(load_base, symbol, relocationAddr, soName, neededLibraries.values(), 0);
                if (moduleSymbol == null) {
                    list.add(new ModuleSymbol(soName, load_base, symbol, relocationAddr, null, 0));
                } else {
                    moduleSymbol.relocation(emulator);
                }
                break;
            case ARMEmulator.R_AARCH64_GLOB_DAT:
            case ARMEmulator.R_AARCH64_JUMP_SLOT:
                moduleSymbol = resolveSymbol(load_base, symbol, relocationAddr, soName, neededLibraries.values(), relocation.addend());
                if (moduleSymbol == null) {
                    list.add(new ModuleSymbol(soName, load_base, symbol, relocationAddr, null, relocation.addend()));
                } else {
                    moduleSymbol.relocation(emulator);
                }
                break;
            case ARMEmulator.R_ARM_COPY:
                throw new IllegalStateException("R_ARM_COPY relocations are not supported");
            case ARMEmulator.R_AARCH64_COPY:
                throw new IllegalStateException("R_AARCH64_COPY relocations are not supported");
            case ARMEmulator.R_AARCH64_ABS32:
            case ARMEmulator.R_AARCH64_ABS16:
            case ARMEmulator.R_AARCH64_PREL64:
            case ARMEmulator.R_AARCH64_PREL32:
            case ARMEmulator.R_AARCH64_PREL16:
            case ARMEmulator.R_AARCH64_IRELATIVE:
            case ARMEmulator.R_AARCH64_TLS_TPREL64:
            case ARMEmulator.R_AARCH64_TLS_DTPREL32:
            case ARMEmulator.R_ARM_IRELATIVE:
            case ARMEmulator.R_ARM_REL32:
            default:
                log.warn("[" + soName + "]Unhandled relocation type " + type + ", symbol=" + symbol + ", relocationAddr=" + relocationAddr + ", offset=0x" + Long.toHexString(relocation.offset()) + ", addend=" + relocation.addend() + ", android=" + relocation.isAndroid());
                break;
        }
    }
 
    // 重定位完成后，开始执行初始化函数
    List initFunctionList = new ArrayList<>();
    if (elfFile.file_type == ElfFile.FT_EXEC) {
        // 处理可执行文件相关，我们分析So的，忽略就可以
        int preInitArraySize = dynamicStructure.getPreInitArraySize();
        int count = preInitArraySize / emulator.getPointerSize();
        if (count > 0) {
            Pointer pointer = UnidbgPointer.pointer(emulator, load_base + dynamicStructure.getPreInitArrayOffset());
            if (pointer == null) {
                throw new IllegalStateException("DT_PREINIT_ARRAY is null");
            }
            for (int i = 0; i < count; i++) {
                Pointer func = pointer.getPointer((long) i * emulator.getPointerSize());
                if (func != null) {
                    initFunctionList.add(new AbsoluteInitFunction(load_base, soName, ((UnidbgPointer) func).peer));
                }
            }
        }
    }
    if (elfFile.file_type == ElfFile.FT_DYN) {
        // 处理So的初始化函数
        //下面的处理内容在新版有修复，我们之前Linker的文章也讲过，他们的顺序不应该是平级的，需要Init函数先执行
        int init = dynamicStructure.getInit();
        if (init != 0) {
            initFunctionList.add(new LinuxInitFunction(load_base, soName, init));
        }
 
        // 处理 init.array
        int initArraySize = dynamicStructure.getInitArraySize();
        int count = initArraySize / emulator.getPointerSize();
        if (count > 0) {
            Pointer pointer = UnidbgPointer.pointer(emulator, load_base + dynamicStructure.getInitArrayOffset());
            if (pointer == null) {
                throw new IllegalStateException("DT_INIT_ARRAY is null");
            }
            for (int i = 0; i < count; i++) {
                // 当作数组来处理每一个init函数
                Pointer func = pointer.getPointer((long) i * emulator.getPointerSize());
                if (func != null) {
                    // 将他们添加到initFunction列表中
                    initFunctionList.add(new AbsoluteInitFunction(load_base, soName, ((UnidbgPointer) func).peer));
                }
            }
        }
    }
 
    // 至此，依赖So加载了，重定位可以处理的也处理了(不能处理的还会有二次处理)
    // 初始化函数也被添加到列表中了，但是还没有调用(注意)
 
    SymbolLocator dynsym = dynamicStructure.getSymbolStructure();
    if (dynsym == null) {
        throw new IllegalStateException("dynsym is null");
    }
    ElfSection symbolTableSection = null;
    try {
        symbolTableSection = elfFile.getSymbolTableSection();
    } catch(Throwable ignored) {}
 
    // 将加载好的So封装位LinuxModule对象
    LinuxModule module = new LinuxModule(load_base, size, soName, dynsym, list, initFunctionList, neededLibraries, regions,
            armExIdx, ehFrameHeader, symbolTableSection, elfFile, dynamicStructure);
    // ...
 
    // 放入已加载的So列表中
    modules.put(soName, module);
    if (maxSoName == null || soName.length() > maxSoName.length()) {
        maxSoName = soName;
    }
    if (bound_high > maxSizeOfSo) {
        maxSizeOfSo = bound_high;
    }
    // 设置可执行Elf的入口点
    module.setEntryPoint(elfFile.entry_point);
    log.debug("Load library " + soName + " offset=" + (System.currentTimeMillis() - start) + "ms" + ", entry_point=0x" + Long.toHexString(elfFile.entry_point));
    // 通知监听器，So已加载完毕
    notifyModuleLoaded(module);
    return module;
} 
```

总结
--

我们分为上下两篇文章。我们看到上面这个 loadInternal 方法就是加载一个 So 主要的内容，大致内容看到这里其实已经够用了，如果想了解更多的细节就看下篇。其中的很多方法我们没有展开来讲解，接下来我们就来处理剩余的细枝末节。如果文章有错误的地方还请指正，可以加个 VX 一起交流：roy5ue 或直接在文章下方进行评论哦

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 3 小时前 被 r0ysue 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#程序开发](forum-161-1-124.htm) [#源码分析](forum-161-1-127.htm)