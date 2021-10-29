> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270033.htm)

> [原创]Unidbg-Linker 部分源码分析 (下)

Unidbg-Linker 部分源码分析 (下)
========================

*   [Unidbg-Linker 部分源码分析 (下)](#unidbg-linker部分源码分析下)
    *   [概要](#概要)
    *   [align](#align)
    *   [resolveLibrary](#resolvelibrary)
    *   [VirtualModule](#virtualmodule)
    *   [initFunctions call](#initfunctions-call)
    *   [总结](#总结)

概要
--

上篇我们分析了 AndroidElfLoader 类中的 loadInternal 方法，很长，很大，忍一忍才能读完。那么我们下篇就来分析下 Unidbg 中 Linker 部分的其他细枝末节

align
-----

上文中有很多地方调用了 ARM.align 方法进行页对齐操作，我们看一下这个方法的实现

```
public static Alignment align(long addr, long size, long alignment) {
    // addr: 起始地址
    // size: 大小
    // alignment: 页面大小
    long mask = -alignment;
    // 计算右边界
    long right = addr + size;
    right = (right + alignment - 1) & mask;
    // addr就是左边界
    addr &= mask;
    size = right - addr;
    // 这行感觉没有必要，因为左右边界对齐后，差也是对齐的，就没必要再进行对齐了
    size = (size + alignment - 1) & mask;
    // 封装对象
    return new Alignment(addr, size);
}
 
public class Alignment {
    public final long address;
    public final long size;
 
    public Alignment(long address, long size) {
        this.address = address;
        this.size = size;
    }
}

```

resolveLibrary
--------------

Unidbg 是如何加载依赖库的？我们来看下面一段代码，是我们上篇文章分析中的依赖库加载部分

```
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

```

先来分析 libraryFile.resolveLibrary 方法

```
public LibraryFile resolveLibrary(Emulator emulator, String soName) {
    // 直接找当前so路径下的对应名字的文件
    File file = new File(elfFile.getParentFile(), soName);
    // 如果没有，就返回null
    return file.canRead() ? new ElfLibraryFile(file, is64Bit) : null;
}

```

再看下面的查找方法

```
if (libraryResolver != null && neededLibraryFile == null) {
    neededLibraryFile = libraryResolver.resolveLibrary(emulator, neededLibrary);
}

```

这句代码使用 libraryResolver 来解析一个 So 文件，这个 libraryResolver 就是我们用

```
memory.setLibraryResolver(new AndroidResolver(23));

```

创建的，那它是怎么解析的呢？继续看

```
public LibraryFile resolveLibrary(Emulator emulator, String libraryName) {
    if (needed == null) {
        return null;
    }
 
    if (!needed.isEmpty() && !needed.contains(libraryName)) {
        return null;
    }
    // 调用下面的重载
    return resolveLibrary(emulator, libraryName, sdk);
}
 
static LibraryFile resolveLibrary(Emulator emulator, String libraryName, int sdk) {
    final String lib = emulator.is32Bit() ? "lib" : "lib64";
    // 很明显吧，找下面路径下的库，我们去找个路径下看看，其实就是一些系统库
    String name = "/android/sdk" + sdk + "/" + lib + "/" + libraryName.replace('+', 'p');
    URL url = AndroidResolver.class.getResource(name);
    if (url != null) {
        return new URLibraryFile(url, libraryName, sdk, emulator.is64Bit());
    }
    return null;
}

```

VirtualModule
-------------

虚拟模块，它的作用是当目标 So 依赖一个 So 的时候，而这个 So 作用不大，甚至基本用不到的时候，就会使用 VirtualModule 来注册一个虚拟模块，防止 So 依赖报错

 

Unidbg 提供了两个虚拟模块，也可以自己实现 VirtualModule 接口

*   libandroid.so
*   libjnigraphics.so  
    已经做了一些简单的处理

假如我们要分析的目标 So 中使用到了 libjnigraphics.so 这个 So，而且它并不影响我们的分析，甚至没用到，一般也没用到，那我们可以这么做

```
new JniGraphics(emulator, vm).register(memory);

```

那在目标 So 加载的时候就不会报错了，我们看下 Unidbg 是如何处理的

```
public Module register(Memory memory) {
    if (name == null || name.trim().length() < 1) {
        throw new IllegalArgumentException("name is empty");
    }
    if (symbols.isEmpty()) {
        throw new IllegalArgumentException("symbols is empty");
    }
 
    if (log.isDebugEnabled()) {
        log.debug(String.format("Register virtual module[%s]: (%s)", name, symbols));
    }
    return memory.loadVirtualModule(name, symbols);
}

```

上面调用了 memory.loadVirtualModule 方法，最终又回到了 AndroidElfLoad 的 loadVirtualModule 方法

```
public Module loadVirtualModule(String name, Map symbols) {
    LinuxModule module = LinuxModule.createVirtualModule(name, symbols, emulator);
    modules.put(name, module);
    if (maxSoName == null || name.length() > maxSoName.length()) {
        maxSoName = name;
    }
    return module;
} 
```

处理方式简单粗暴，创建一个 LinuxModule，直接加载到我们的 modules 里面，这个 modules 保存了我们已经加载过的 So，所以在目标 So 加载的时候，就能够找到这个虚拟模块，从而不会报错

initFunctions call
------------------

大概补充上篇要讲的加载部分就这么多，下面我们来分析剩余的流程，继续看我们上节课的一个方法

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

这整个方法我们就只分析了一句，就分析了整个上篇的内容，接下来我们继续向下分析

```
resolveSymbols(!forceCallInit);

```

这个方法我们就不展开讲了，它做的就还是进行了一次对所有未重定位的地方进行重定位最后的确定操作

 

还记得在加载 So 的过程中，init 函数只是添加到了一个列表里面，保存到了 LinuxModule 对象中，还未进行执行呢对吧，那接下来看下面部分代码

```
// 先做了一下判断callInitFunction(默认) || forceCallInit(这个参数是我们传进来的)
if (callInitFunction || forceCallInit) {
    // 对所有模块进行遍历
    for (LinuxModule m : modules.values().toArray(new LinuxModule[0])) {
        // 两种为真的情况
        // 1. 模块是我们自己加载的模块 且 设置 forceCallInit参数为true
        // 2. 模块本身有一个forceCallInit参数，默认为true
        boolean forceCall = (forceCallInit && m == module) || m.isForceCallInit();
 
        // 调用初始化函数
        if (callInitFunction) {
            m.callInitFunction(emulator, forceCall);
        } else if (forceCall) {
            m.callInitFunction(emulator, true);
        }
 
        // 移除该模块下的所有初始化函数
        m.initFunctionList.clear();
    }
}

```

所以分析了上面，我们知道了 forceCallInit 这个参数想仅用初始化并没有什么用对吧，因为 callInitFunction 默认为 true，要想生效需调用

```
memory.disableCallInitFunction();

```

来禁用默认初始化，继续分析 callInitFunction 方法

```
void callInitFunction(Emulator emulator, boolean mustCallInit) throws IOException {
    // 如果非必需初始化So，且存在未处理的符号，就算了，不初始化了
    if (!mustCallInit && !unresolvedSymbol.isEmpty()) {
        for (ModuleSymbol moduleSymbol : unresolvedSymbol) {
            log.info("[" + name + "]" + moduleSymbol.getSymbol().getName() + " symbol is missing before init relocationAddr=" + moduleSymbol.getRelocationAddr());
        }
        return;
    }
    // 否则，就要挨着执行初始化函数了，这个initFunctionList就是我们上篇分析过的初始化函数列表
    while (!initFunctionList.isEmpty()) {
        InitFunction initFunction = initFunctionList.remove(0);
        initFunction.call(emulator);
    }
}

```

总结
--

下篇就分析到这里吧，Unidbg 中关于加载模块的部分我们就已经分析完了，其实在我们学习过 Android 源码中的 Linker 部分之后，再来分析 Unidbg 中的'Linker'就比较简单了。相信你读懂了这两篇文章，就能驾驭 Unidbg 在加载 So 中出现的各种问题了。老规矩 V:roysue

[【公告】【iPhone 13 大奖等你拿】看雪. 众安 2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

[#基础理论](forum-161-1-117.htm) [#程序开发](forum-161-1-124.htm) [#系统相关](forum-161-1-126.htm) [#源码分析](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm)