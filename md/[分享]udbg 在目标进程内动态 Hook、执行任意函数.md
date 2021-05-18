> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267630.htm)

测试环境
====

1.  通过 spy 调试引擎附加到 notepad 进程
    
    PS D:\dist> notepad.exe  
    PS D:\dist> .\udbg.exe -A spy -a notepad.exe
    
2.  在 udbg 内执行`.edit temp`命令，会自动在`script/autorun`目录下打开 (创建)`temp.lua`文件，并监控其写入，然后自动执行
    
3.  执行下面的示例脚本：在`temp.lua`里编辑脚本并保存，udbg 自动执行

udbg 版本：[https://gitee.com/udbg/udbg/releases/v0.1.0-nightly](https://gitee.com/udbg/udbg/releases/v0.1.0-nightly)

工作原理
====

udbg 的 spy 调试引擎，会向目标进程内注入一个模块，然后使用封装好的`udbg.uspy`模块可以在目标进程内执行 lua 脚本，然后通过 uspy 提供的 lua 接口完成 Hook 任意函数、调用任意函数的任务

 

在目标进程内执行 lua 代码的基本用法如下

```
local uspy = require "udbg.uspy"
local s = 'hello'
uspy(function()
    log(s, 'world')
end)

```

*   uspy 会把传入的 function（闭包）转换为字节码，然后通过 RPC 发送到目标进程进行执行
*   假如闭包捕获了 upvalue，也会将这些 upvalue 序列化并传过去，前提是被捕获的 upvalue 都可进行序列化

后面的示例脚本没有特殊说明，默认都是通过`uspy(function() ... end)`在目标进程内进行调用的

Hook 任意函数
=========

基本用法

```
local uspy = require "udbg.uspy"
local CreateFileW = PA 'kernelbase!CreateFileW'
 
uspy(function()
    local libffi = require 'libffi'
    local api = require 'win.api'
    -- Hook CreateFileW 函数，并打印第一个参数
    inline_hook(CreateFileW, function(args)
        local path = libffi.read_pack(args[1], 'w')
        log('CreateFileW', path)
    end)
end)

```

Hook 并进行参数替换

```
inline_hook(CreateFileW, function(args)
    -- args[1] 在x64下相当于 args.rcx
    local path = libffi.read_pack(args[1], 'w')
    log('CreateFileW', path)
    if path:find 'a.txt$' then
        path = path:gsub('a.txt$', 'b.txt')
        log('[redirect]', 'a.txt', '->', path)
        args[1] = topointer(path:to_utf16())
    end
end)

```

Hook 并调用原函数、阻止原函数继续执行

```
local MessageBoxA = api.GetProcAddress(api.LoadLibraryA('user32'), 'MessageBoxA')
inline_hook(MessageBoxA, function(args)
    -- 手动调用原函数
    libffi.fn(args.trampoline)(0, 'LALALA', 'AAAAAA', 0)
    -- 返回后不再调用原函数
    args 'reject'
end)
local msgbox = libffi.fn(MessageBoxA)
msgbox(0, 'ABC', 'DEF', 0)

```

libffi 的使用
==========

无类型调用
-----

上述 Hook 示例中，对 MessageBoxA 函数的调用就是一个无类型调用的例子

```
local msgbox = libffi.fn(MessageBoxA)
msgbox(0, 'ABC', 'DEF', 0)

```

`libffi.fn`函数直接传入一个整数（C 函数地址），返回一个无类型的函数对象，无类型的函数对象会根据传入的 lua 参数类型自动为每个 C 参数分配类型，映射规则如下

*   `string|userdata` => pointer
*   `integer|boolean` => size_t
*   `nil|none` => NULL
*   `number` => double

无类型的调用在于写起来非常简单，能够覆盖大部分应用场景，但有些时候还是需要知道明确的函数参数才能成功调用一些函数，比如涉及到浮点数的函数、非标准的调用约定等情况

声明函数类型
------

声明函数类型需要指定函数的返回值类型和参数类型 `libffi.fn(returnType, {argsType...})`，支持的类型如下

*   `void`
*   `char` `int8`
*   `byte` `uchar`
*   `short` `int16`
*   `ushort` `uint16`
*   `int` `int32`
*   `uint` `uint32`
*   `int64` `long long`
*   `uint64`
*   `float`
*   `double`
*   `pointer`

```
local pow = api.GetProcAddress(api.LoadLibraryA'msvcrt', 'pow')
-- 通过fn声明函数类型
pow = libffi.fn('double', {'double', 'double'})(pow)
log('powf(2, 2)', pow(2, 2))
 
local powf = api.GetProcAddress(api.LoadLibraryA'msvcrt', 'powf')
powf = libffi.fn('float', {'float', 'float'})(powf)
log('powf(2, 3)', powf(2, 3))

```

指定 x86 调用约定： TODO

生成回调函数
------

示例如下

```
local WNDENUMPROC = libffi.fn('int', {'pointer', 'pointer'})
api.EnumChildWindows(0, WNDENUMPROC(function(hwnd, param)
    log('HWND:', hwnd)
end), 0)

```

主线程执行
=====

有些函数只能在某些特定线程中调用，一般是 UI 主线程，可以 Hook GetMessageW PeekMessageW 之类的函数，Hook 触发时，脚本则是在 UI 线程中调用的，然后通过 libffi 去调用想测试的函数

```
-- inline_once是对inline_hook函数的封装，Hook触发一次后立即消除Hook
inline_once(api.GetProcAddress(api.LoadLibraryA'user32', 'GetMessageW'), function()
    local msgbox = libffi.fn(MessageBoxA)
    msgbox(0, 'ABC', 'DEF', 0)
end)

```

更多示例
====

udbg 本身功能也使用了很多 libffi 调用，比如 `script/win/api.lua` `script/win/win.lua`

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)