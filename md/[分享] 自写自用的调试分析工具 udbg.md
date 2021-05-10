> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267410.htm)

自己利用业余时间写了很长时间了，核心功能使用 rust 开发，上层逻辑使用 lua 开发，不敢说比现有的调试器更好用，主要是方便我自己用来做一些自动化调试分析的事情

*   项目地址： https://gitee.com/udbg/udbg
*   文档地址： https://udbg.github.io/udbg/index.html

从 [release 页面](https://gitee.com/udbg/udbg/releases/)下载 dist.zip 解压到本地即可使用，**路径中不要包含 Unicode 字符**

 

要介绍的内容有点多，文档还在完善中...

设计理念
----

*   尽可能支持更多的调试 / 分析场景
    *   Windows：标准调试器
    *   Windows：VEH 调试器
    *   非侵入式调试、进程信息查看
    *   WinDbg 调试引擎：可用于分析 dmp、内核调试
    *   [ ] ARK 工具、内核信息查看
    *   [ ] 跨平台：支持 linux、android
*   方便地编写扩展
    *   udbg 本身使用 lua 编写了大部分上层逻辑
    *   提供了大量 lua 接口，支持 **libffi**
    *   提供原生的 Rust 接口，编写高性能的扩展
*   界面 / 核心分离，可进行远程分析 / 调试
*   CUI && GUI 双界面: CUI 便于和其他第三方编辑器集成、GUI 便于展示更多数据
*   功能抽象、分层，跨平台：既保留各平台的共性，降低学习成本，也允许差异化定制，实现平台特定的功能

启动调试器
-----

主要是命令行启动

*   `udbg` 启动 udbg，并显示主窗口，相当于在资源管理器中双击 udbg.exe 进行启动
*   `udbg -W` 无窗口启动 udbg，在命令行中调试
*   `udbg -e test.lua` 启动 udbg 并执行 test.lua 脚本
*   `udbg -e test.lua --watch` 监控 test.lua 脚本改动并自动执行
*   `udbg notepad.exe` (默认的调试引擎) 创建并调试 notepad.exe
*   `udbg -a notepad.exe` (默认的调试引擎) 附加到 notepad.exe
*   `udbg -A spy notepad.exe` 通过 spy 调试引擎 创建并调试 notepad.exe
*   `udbg -A spy -a notepad.exe` 通过 spy 调试引擎 附加到 notepad.exe
*   `udbg -r localhost:2333` 连接到 udbg-server，并启动，后面可跟上面所有的参数，参考 [远程调试](./scene/remote-debug.md)

完整的命令行语法如下

```
udbg 0.1.0
metaworm
 
USAGE:
    udbg.exe [FLAGS] [OPTIONS] [--] [ARGS]
 
FLAGS:
    -a, --attach       Attach target
    -h, --help         Prints help information
    -W, --no-window    Dont show the main window
    -o, --open         Open the target, not attach
    -p, --pid          Target as pid
        --version      Prints version information
    -V, --verbose      Show the verbose info
    -w, --watch        Watch the executed lua script
 
OPTIONS:
    -A, --adaptor Specify the adaptor [default: ]
    -r, --remote Connect to udbg server, with cui [env: UDBG_SERVER=]
        --cwd Set CWD for target
    -e, --execute Execute lua script
    -c, --config ...    Set the __config
 
ARGS:
     Create debug target
    ...        Shell Arguments to target 
```

软件截图  
![](https://bbs.pediy.com/upload/attach/202105/865242_KES868YGFFSQ3SE.png)

脚本自动执行
------

1.  最直接的方法是通过命令行启动`udbg -e test.lua --watch`指定运行一个脚本并监控其改动，然后自动执行
2.  可以在 udbg.exe 所在目录下创建`config/autorun/`目录，autorun 目录下的 lua 脚本发生改动时都会自动执行

断点
--

设置断点最简单的方法是在反汇编视图中，选择对应汇编语句，按`F2`键

 

但如果想设置更复杂的断点需要使用`bp`命令，列几个常用的断点设置示例

*   常规断点 `bp CreateFileW` 在 CreateFileW 函数处下断点
*   日志断点 `bp CreateFileW wstr(reg[1])` 在 CreateFileW 函数处下断点，并显示第一个参数
    *   其中`wstr(reg[1])`是一个 lua 表达式，表示需要记录的内容，`reg[1]`表示当前架构下标准调用约定的第一个参数 (x64 下相当于`reg.rcx`)，可跟多个表达式比如 `bp CreateFileW wstr(reg[1]) reg.rdx`
    *   如果想过滤需要记录的日志，可加`-f`参数 + lua 表达式，比如 `bp CreateFileW wstr(reg[1]) -f v1:find'log.txt'`可以过滤带有 log.txt 的路径，其中 v1 表示 CreateFileW 后面的表达式列表里的第一个表达式的值
*   条件断点 `bp CreateFileW wstr(reg[1]) -c v1:find'log.txt'` 当 CreateFileW 的第一个参数路径包含 log.txt 时断下
*   硬件断点 `bp CreateFileW -t e1` 临时 (一次性) 断点 `bp CreateFileW --temp`

bp 命令的完整帮助 `bp -h`

```
bp                                        设置断点
    

       (string)              断点地址
     (optional string)     变量列表
 
    -n, --name        (optional string)   断点名称
    -c, --cond        (optional string)   中断条件
    -l, --log         (optional string)   日志表达式
    -f, --filter      (optional string)   日志条件
    -s, --statistics  (optional string)   统计表达式
    -t, --type        (optional bp_type)  断点类型(e|w|a)(1|2|4|8)
    --tid             (optional number)   命中线程
    --temp                                临时断点
    --symbol                              转换为符号
    --hex                                 十六进制显示
    --caller                              显示调用者
    -m, --module                          模块加载断点 


```

如果需要了解 bp 命令的详细实现方式，参考`script/udbg/command/bp.lua`

### 自定义断点回调

通过脚本定制更复杂的断点处理逻辑

```
add_bp('kernel32!CreateFileW', function()
    local path = read_wstring(reg.rcx)
    log('[CreateFileW]', path)
    if path:find 'xxx' then
        -- 返回true表示这个断点会中断给用户处理
        return true
    end
    -- 在调用 CreateFileW 的返回点处设置断点
    add_bp(reg.rsp, function()
        log('[CreateFileW]', 'return', reg.rax)
    end, {temp = true, type = 'table'})
end)

```

堆栈回溯
----

除了堆栈视图，还可以使用`dp -r rsp`来查看堆栈；udbg 目前未实现基于符号的堆栈回溯，`dp -r`本质上是暴力搜索堆栈上的返回地址

模块符号
----

udbg 自带的调试引擎支持加载 pdb 符号，可在配置文件中通过`__config.symbol_cache = 'D:/Symbols`来指定 pdb 符号所在的目录，目录结构和 windbg 的符号缓存目录一致，可以在 windbg 里下载系统所需要的符号

 

udbg 首先会根据 dll/exe 里的 pdb 路径去加载符号，如果没有的话再寻找 dll 所在目录下的同名 pdb，如果也没有就会去符号缓存目录中加载

 

如果想给一个模块手动指定 pdb 路径，可以使用命令 `load-symbol xx.dll D:\xx.pdb`

 

也可以在配置文件中写脚本来自动加载

```
function uevent.on.module_load(m)
    if m.base == PA 'xx.dll' then
        m:load_symbol [[D:\xx.pdb]]
    end
end

```

内存读写
----

*   直接在内存视图中查看
    *   `Ctrl+1` `Ctrl+2` `Ctrl+3` `Ctrl+4` 分别切换 BYTE WORD DWORD QWORD 显示
    *   `Ctrl+F`切换 float 显示
    *   `Ctrl+D`切换 double 显示
    *   `Ctrl+G`可以输入地址表达式并跳转到对应地址
*   `mem <address>` 命令在内存视图中 跳转到对应的地址表达式 `mem ntdll.dll` `mem [[0x403000]+0x10]`
*   `e` 命令编辑内存
*   Lua 函数 `{read|write}_{u8|u16|u32|u64|ptr|float|double}`

异常处理
----

### 相关配置

*   `__config.ignore_all_exception = false` 是否忽略所有异常

### 处理异常事件

可以通过脚本配置更复杂的异常处理

```
function uevent.on.exception(tid, code, first)
    if code == STATUS_CPP_EH_EXCEPTION then
        return 'run'
    end
    if reg.rip == 0 then
        return 'run'
    end
end

```

单步跟踪
----

在目标断下时，可以通过如下脚本来启动一个单步跟踪过程

```
-- 单步跟踪1000步并输出每一步的汇编语句
local count = 1000
local disasm = disasm
ui.continue('step', function()
    count = count - 1
    local pc = reg._pc
    log(hex(pc), disasm(pc).string)
    return count == 0
end)

```

配置脚本
----

*   udbg 所在目录下的`config`目录为配置目录，可以通过同步盘同步配置，软连接到 config
*   config 目录本身也会被加到 lua 的模块搜索路径里
*   config 下的`client.lua`为第一个被执行的配置脚本，主要用于定制一些跟客户端 UI 相关的配置，配置脚本示例如下
    
    ```
    -- 指定其他插件/配置路径
    plugins = {
        {path = [[D:\Plugin1]]},
        {path = [[D:\Plugin2]]},
    }
     
    -- 指定启动编辑器的命令
    --   默认是 notepad
    edit_cmd = 'notepad.exe %1'
    --   如果安装了vscode，建议改为 vscode
    edit_cmd = 'code %1'
     
    -- 远程地址映射
    --   配置了此项可在命令行中用键名代替对应的地址端口
    --   比如: udbg -r vmware C:\Windows\notepad.exe
    remote_map = {
        ['local'] = '127.0.0.1:2333',
        local1 = '127.0.0.1:2334',
        local2 = '127.0.0.1:2335',
        vmware = '192.168.1.239:2333',
    }
    
    ```
    
*   config 下的`udbg/plugin/init.lua`会在调试器初始化时被执行，可以做一些跟调试器相关的配置，比如
    
    ```
    -- 忽略初始断点(不中断给用户)
    __config.ignore_initbp = true
    -- 初始断点事件触发时自动下断点
    function uevent.on.init_bp()
        if udbg.target.path:find 'notepad' then
            ucmd 'bp kernel32!CreateFileW'
        end
    end
    
    ```
    
    **注意**，执行`client.lua`和`udbg/plugin/init.lua`的是两个不同的 lua 虚拟机，前者是给客户端 UI 使用的，后者是给调试器核心使用的
    
*   `autorun/`目录下的 lua 脚本发生改动时，会被（调试器的 lua 虚拟机）自动执行

远程调试
----

1.  启动 udbg 服务：将`udbg-server.exe` `lua54.dll` `uspy.dll` 三个文件拷到目标机器上，然后管理员权限运行 udbg-server.exe
2.  通过`udbg -r 目标机器地址:端口`连接到 udbg 服务进行调试，比如`udbg -r 192.168.1.239:2333 C:\Windows\notepad.exe`

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)