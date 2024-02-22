> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280609.htm)

> [原创]360 加固 dex 解密流程分析

[原创]360 加固 dex 解密流程分析

4 小时前 122

### [原创]360 加固 dex 解密流程分析

 [![](http://passport.kanxue.com/upload/avatar/320/963320.png?1667284712)](user-home-963320.htm) [oacia](user-home-963320.htm) ![](https://bbs.kanxue.com/view/img/rank/9.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 4 小时前  122

前言
==

> 在去年 9 月的时候, 我就想研究一下 apk 的加固, 在网上逛了一圈, 感觉 360 加固不错, 所以先选它啦, 写了一个测试 apk 丢到 360 加固里面加固了一下
> 
> 这里我用的是 360 加固的免费版 (因为付费版太贵啦)
> 
> 本来计划着去年分析完 360 加固的, 但是总是抽不出一段完整的时间来所以就拖到了现在, 终于最近因为过年赋闲在家, 就花了几天分析了一下 360 加固, 感觉这几天探索 360 加固的过程真是充满了惊喜和乐趣呢~

java 层初步分析
==========

包名:`com.oacia.apk_protect`

入口:`com.stub.StubApp`

我们从`AndroidManifest.xml`中可以得知, 360 加固的入口是`com.stub.StubApp`, 所以我们就先进到 apk 的入口进行分析

![](https://bbs.kanxue.com/upload/attach/202402/963320_9G8MNFNAYXD6NBZ.png)

在这个入口类中, 不仅有常规的`onCreate()`函数, 还有一个函数值得注意, 他就是`attachBaseContext(Context context)`

![](https://bbs.kanxue.com/upload/attach/202402/963320_QVUGRKF3BFVAUYU.png)

> Application 的`onCreate`和`attachBaseContext`是 Application 的两个回调方法，通常我们会在其中做一些初始化操作,`attachBaseContext` 在`onCreate`**之前执行**

其中出现的字符串经过了加密混淆操作, 加密函数如下, 算法是将所有的字符与`16`进行异或

![](https://bbs.kanxue.com/upload/attach/202402/963320_2YXWJT4A6NRWJVF.png)

我们写个 jeb 脚本把加密字符串解密来方便后续的静态分析

```
# coding=utf-8
from com.pnfsoftware.jeb.client.api import IScript, IconType, ButtonGroupType
from com.pnfsoftware.jeb.core import RuntimeProjectUtil
from com.pnfsoftware.jeb.core.units.code.java import IJavaSourceUnit
from com.pnfsoftware.jeb.core.units.code import ICodeUnit, ICodeItem
from com.pnfsoftware.jeb.core.output.text import ITextDocument
from com.pnfsoftware.jeb.core.units.code.java import IJavaSourceUnit, IJavaStaticField, IJavaNewArray, IJavaConstant, IJavaCall, IJavaField, IJavaMethod, IJavaClass
from com.pnfsoftware.jeb.core.events import JebEvent, J
from com.pnfsoftware.jeb.core.util import DecompilerHelper
 
# 解密字符串函数的类名以及方法名
methodName = ['Lcom/qihoo/util/a;', 'a']
 
 
class dec_str_360jiagu(IScript):
    def run(self, ctx):
        print('start deal with strings')
        self.ctx = ctx
        engctx = ctx.getEnginesContext()
        if not engctx:
            print('Back-end engines not initialized')
            return
 
        projects = engctx.getProjects()
        if not projects:
            print('There is no opened project')
            return
 
        units = RuntimeProjectUtil.findUnitsByType(projects[0], IJavaSourceUnit, False)
        for unit in units:
            javaClass = unit.getClassElement()
            print('[+] decrypt:' + javaClass.getName())
            self.cstbuilder = unit.getFactories().getConstantFactory()
            self.processClass(javaClass)
            unit.notifyListeners(JebEvent(J.UnitChange))
        print('Done.')
 
    def processClass(self, javaClass):
        if javaClass.getName() == methodName[0]:
            return
        for method in javaClass.getMethods():
            block = method.getBody()
            i = 0
            while i < block.size():
                stm = block.get(i)
                self.checkElement(block, stm)
                i += 1
 
    def checkElement(self, parent, e):
        try:
            if isinstance(e, IJavaCall):
                mmethod = e.getMethod()
                mname = mmethod.getName()
                msig = mmethod.getSignature()
                if mname == methodName[1] and methodName[0] in msig:
                    v = []
                    for arg in e.getArguments():
                        if isinstance(arg, IJavaConstant):
                            v.append(arg.getString())
                    if len(v) == 1:
                        decstr = self.decryptstring(v[0])
                        parent.replaceSubElement(e, self.cstbuilder.createString(decstr))
 
            for subelt in e.getSubElements():
                if isinstance(subelt, IJavaClass) or isinstance(subelt, IJavaField) or isinstance(subelt, IJavaMethod):
                    continue
                self.checkElement(e, subelt)
        except:
            print('error')
 
    def decryptstring(self, string):
        src = []
        for index, char in enumerate(string):
            src.append(chr(ord(char) ^ 16))
 
        return ''.join(src).decode('unicode_escape')

```

解密后的效果如下

![](https://bbs.kanxue.com/upload/attach/202402/963320_X3DH4R48XPZDPPG.png)

我们往下进行分析, 可以知道`attachBaseContext`的第一个作用是根据目标手机的架构加载`libjiagu_xxx.so`如图

![](https://bbs.kanxue.com/upload/attach/202402/963320_8QUXQNGNJJ58VB6.png)

这些 so 在`assets`目录下

![](https://bbs.kanxue.com/upload/attach/202402/963320_PESXDAA4KC5RN22.png)

在加载完`libjiagu_xxx.so`之后, 还调用了`DtcLoader`类进行初始化, 这里使用的 jadx 的反编译结果, 因为 jeb 没有反编译出`DtcLoader.init();`方法来

![](https://bbs.kanxue.com/upload/attach/202402/963320_YQ744VNB2ESKB4V.png)

`DtcLoader`类如图所示

![](https://bbs.kanxue.com/upload/attach/202402/963320_WTRH627XU3JGVFG.png)

当`DtcLoader`类被加载到 JVM 中时, 会去加载`libjgdtc.so`, 如果加载失败, 则会尝试从`/data/app/com.oacia.apk_protect/lib/arm64/libjgdtc.so`或者`/data/data/com.oacia.apk_protect/lib/libjgdtc.so`中去加载这个 so

但是当我们去进入到这两个目录进行寻找时, 却发现没有这个`libjgdtc.so`存在

![](https://bbs.kanxue.com/upload/attach/202402/963320_GGTSXDHBUHENUU2.png)

所以我们的分析重点是在`libjiagu.so`中, 这里我选取了 arm64 架构的 so 文件`libjiagu_a64.so`进行分析

壳 ELF 导入导出表修复
=============

我们使用 ida 分析`libjiagu_a64.so`, 发现导入表和导出表都没有内容, 既然是这种情况, 那么应该是在 so 装载进内存时导入导出表才会去进行相应的链接操作

![](https://bbs.kanxue.com/upload/attach/202402/963320_9WEU7EJGZGP35QG.png)

所以我们可以先用 frida 把这个 so 给 dump 下来

首先我们在手机上运行一下 frida server

```
PS D:\frida> adb shell
blueline:/ $ su
blueline:/ # cd /data/local/tmp
blueline:/data/local/tmp # ./fs -l 0.0.0.0:1234

```

随后做一下端口转发

```
adb forward tcp:1234 tcp:1234

```

frida 命令行语句如下

```
frida -H 127.0.0.1:1234 -l .\hook.js -f "com.oacia.apk_protect"

```

注入如下脚本

```
function my_hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    dump_so("libjiagu_64.so");
                }
            }
        }
    );
}
 
function dump_so(so_name) {
    var libso = Process.getModuleByName(so_name);
    console.log("[name]:", libso.name);
    console.log("[base]:", libso.base);
    console.log("[size]:", ptr(libso.size));
    console.log("[path]:", libso.path);
    var file_path = "/data/data/com.oacia.apk_protect/" + libso.name + "_" + libso.base + "_" + ptr(libso.size) + ".so";
    var file_handle = new File(file_path, "wb");
    if (file_handle && file_handle != null) {
        Memory.protect(ptr(libso.base), libso.size, 'rwx');
        var libso_buffer = ptr(libso.base).readByteArray(libso.size);
        file_handle.write(libso_buffer);
        file_handle.flush();
        file_handle.close();
        console.log("[dump]:", file_path);
    }
}
 
setImmediate(my_hook_dlopen("libjiagu_64.so"));

```

![](https://bbs.kanxue.com/upload/attach/202402/963320_2GPCZ7RW6V45ZN4.png)

随后我们使用 [SoFixer](https://github.com/F8LEFT/SoFixer) 修复一下这个 so, 这里的`-m`参数即这个 so 在内存中的`base`基地址

```
.\SoFixer-Windows-64.exe -s .\libjiagu_64.so_0x74a2845000_0x274000.so -o .\libjiagu_64_0x74a2845000_0x274000_fix.so -m 0x74a2845000 -d

```

修复完成后, 导入表和导出表就恢复了

![](https://bbs.kanxue.com/upload/attach/202402/963320_SUNRBRTETWNMDUQ.png)

加固壳反调试初步分析
==========

首先我们去 hook 一下打开 so 的函数

```
function hook_dlopen() {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    console.log("load " + path);
                }
            }
        }
    );
}
 
setImmediate(hook_dlopen)

```

日志如下, 所以反调试是在`libjiagu_64.so`中

```
load libstats_jni.so
load /data/app/~~P6meiEqXSQZrP2ChUgVgOg==/com.oacia.apk_protect-ezyVSLdtBZmLTZejgPlSoQ==/oat/arm64/base.odex
load /data/data/com.oacia.apk_protect/.jiagu/libjiagu_64.so

```

然后去 hook 打开文件的函数`open`

```
function my_hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_open();
                }
            }
        }
    );
}
function hook_open(){
    var pth = Module.findExportByName(null,"open");
  Interceptor.attach(ptr(pth),{
      onEnter:function(args){
          this.filename = args[0];
          console.log("",this.filename.readCString())
      },onLeave:function(retval){
      }
  })
}
setImmediate(my_hook_dlopen,"libjiagu");

```

日志如下

![](https://bbs.kanxue.com/upload/attach/202402/963320_HTX6TECNG6JDK8V.png)

这里我们发现了`/proc/self/maps`, 这是常见的反调试, 要绕过这个检测, 我们可以备份一个正常的`maps`文件, 然后用 frida 去 hook `open`函数, 如果匹配到字符串`maps`, 就将字符串重定向到我们备份的`maps`文件

首先我们正常打开一次加壳的 apk, 然后使用下列命令备份 maps

```
cp /proc/self/maps /data/data/com.oacia.apk_protect/maps

```

然后我们注入如下 frida 脚本

```
function my_hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_proc_self_maps();
                }
            }
        }
    );
}
 
function hook_proc_self_maps() {
    const openPtr = Module.getExportByName(null, 'open');
    const open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
    var fakePath = "/data/data/com.oacia.apk_protect/maps";
    Interceptor.replace(openPtr, new NativeCallback(function (pathnameptr, flag) {
        var pathname = Memory.readUtf8String(pathnameptr);
        console.log("open",pathname);
        if (pathname.indexOf("maps") >= 0) {
            console.log("find",pathname,",redirect to",fakePath);
            var filename = Memory.allocUtf8String(fakePath);
            return open(filename, flag);
        }
        var fd = open(pathnameptr, flag);
        return fd;
    }, 'int', ['pointer', 'int']));
}
 
 
setImmediate(my_hook_dlopen,"libjiagu");

```

但是当注入这段脚本后, 进程由于非法内存访问而退出了, 这说明 360 加固不仅读取 maps 文件, 并且会尝试访问 maps 文件中所记录的文件或内存映射. 这里由于 frida 注入后重启 apk, 但是备份的 maps 文件中记录的是先前的映射起始地址 (这块内存在关闭 apk 后就被抹去了), 所以当壳尝试访问其中的映射时产生了非法内存访问从而让进程崩溃

![](https://bbs.kanxue.com/upload/attach/202402/963320_QUXS278UQ3GEZRY.png)

这里我的解决方式是将上述 frida 代码中的`fakePath`赋值为一个不存在的文件例如`/data/data/com.oacia.apk_protect/maps_nonexistent`, 来让壳没有内容可以访问

修改完`fakePath`后重新注入代码, 这个打印出来的日志可以说非常有意思, 我们来看一下, 相比 hook `open`之前的日志, 我们成功的让 360 加固的壳释放出了 dex

![](https://bbs.kanxue.com/upload/attach/202402/963320_GV29GNEHSPXDE96.png)

然而这个壳似乎是又发现了些什么异常, 随后赶紧让 app 退出了, 但是由于退出的太过仓促, 它甚至还没有来得及把 dex 从文件夹中删除

![](https://bbs.kanxue.com/upload/attach/202402/963320_BQ6KAW9HDX3DSN9.png)

用 010editor 打开`classes.dex`, 发现前几位并不是 dex 的魔术头, 说明这个 dex 还没有被解密, 不过现在我们只需要分析 dex 如何被壳从内存中释放出来的过程就可以了~

![](https://bbs.kanxue.com/upload/attach/202402/963320_V2EJSE4MKG66KBS.png)

如何可以定位到是什么位置调用了`open`函数来打开`classes.dex`呢?

很简单, 打印一下堆栈就可以了

假如我们使用常规的 frida 打印堆栈代码, 即使用`DebugSymbol.fromAddress`函数来判断地址所在的`so`的位置, 那么进程是会报错退出的

```
console.log('RegisterNatives called from:\\n' + Thread.backtrace(this.context, Backtracer.FUZZY).map(DebugSymbol.fromAddress).join('\\n') + '\\n');

```

所以这里`DebugSymbol.fromAddress`所实现的逻辑需要自己编写, 即下方的`addr_in_so`函数

```
function addr_in_so(addr){
    var process_Obj_Module_Arr = Process.enumerateModules();
    for(var i = 0; i < process_Obj_Module_Arr.length; i++) {
        if(addr>process_Obj_Module_Arr[i].base && addr= 0) {
            console.log("find",pathname+", redirect to",fakePath);
            var filename = Memory.allocUtf8String(fakePath);
            return open(filename, flag);
        }
        if (pathname.indexOf("dex") >= 0) {
            Thread.backtrace(this.context, Backtracer.FUZZY).map(addr_in_so);
        }
        var fd = open(pathnameptr, flag);
        return fd;
    }, 'int', ['pointer', 'int']));
}
 
function my_hook_dlopen(soName='') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    //console.log(path);
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_proc_self_maps();
                }
            }
        }
    );
}
setImmediate(my_hook_dlopen,'libjiagu'); 
```

于是我们得到了释放三个 dex 文件的堆栈回溯

*   `classes.dex`  
    ![](https://bbs.kanxue.com/upload/attach/202402/963320_CSBWY9SXQSUETM9.png)
*   `classes2.dex`  
    ![](https://bbs.kanxue.com/upload/attach/202402/963320_6E2MPVBVAF9Z9DT.png)
*   `classes3.dex`  
    ![](https://bbs.kanxue.com/upload/attach/202402/963320_3YFJH6JPEVBKC5K.png)

这里我们发现`classes.dex`与`classes2.dex`的堆栈回溯完全相同, 并且`classes3.dex`的前半部分和前两个 dex 的堆栈一样, 随后进程便又退出了

通过对堆栈的分析, 我们可以发现三个 dex 应该是在一个循环中被依次加载的

接下来我们便跳转到堆栈所打印的偏移来进一步分析下

然而当我们跳转到堆栈回溯中的`libjiagu_64.so`的偏移`0x19b780`或者`0x134598`时, 却发现这些地址的值都是 0

![](https://bbs.kanxue.com/upload/attach/202402/963320_7JX5AGYBSTZEBU8.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_JND9XHKKK8GVVPA.png)

我们很快就能想到这里用到的技术应该是先将一块内存标记为可写可执行, 随后将字节码填充进去, 所以说, 我们只需要在壳打开 dex 时, 将此时的`libjiagu_64.so`从内存中 dump 下来就可以了

```
function dump_so(so_name) {
    var libso = Process.getModuleByName(so_name);
    console.log("[name]:", libso.name);
    console.log("[base]:", libso.base);
    console.log("[size]:", ptr(libso.size));
    console.log("[path]:", libso.path);
    var file_path = "/data/data/com.oacia.apk_protect/" + libso.name + "_" + libso.base + "_" + ptr(libso.size) + ".so";
    var file_handle = new File(file_path, "wb");
    if (file_handle && file_handle != null) {
        Memory.protect(ptr(libso.base), libso.size, 'rwx');
        var libso_buffer = ptr(libso.base).readByteArray(libso.size);
        file_handle.write(libso_buffer);
        file_handle.flush();
        file_handle.close();
        console.log("[dump]:", file_path);
    }
}
 
var dump_once = false;//因为会打开三次dex,所以这里我们仅dump打开第一次dex时的libjiagu_64.so
function hook_proc_self_maps() {
    const openPtr = Module.getExportByName(null, 'open');
    const open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
    var fakePath = "/data/data/com.oacia.apk_protect/maps_nonexistent";
    Interceptor.replace(openPtr, new NativeCallback(function (pathnameptr, flag) {
        var pathname = Memory.readUtf8String(pathnameptr);
        console.log("open",pathname);//,Process.getCurrentThreadId()
        if (pathname.indexOf("maps") >= 0) {
            console.log("find",pathname+", redirect to",fakePath);
            var filename = Memory.allocUtf8String(fakePath);
            return open(filename, flag);
        }
        if (pathname.indexOf("dex") >= 0) {
            if(!dump_once){
                dump_once = true;
                dump_so("libjiagu_64.so");
            }
        }
        var fd = open(pathnameptr, flag);
        return fd;
    }, 'int', ['pointer', 'int']));
}

```

然后再去用 [SoFixer](https://github.com/F8LEFT/SoFixer) 修复这个 dump 下来的 so

```
.\SoFixer-Windows-64.exe -s .\libjiagu_64.so_0x7a69829000_0x274000_open_classes.dex.so -o .\libjiagu_64.so_0x7a69829000_0x274000_open_classes.dex_fix.so -m 0x7a69829000 -d

```

再次来到偏移`0x19B780`处, 可以发现这块空内存已经被填充了数据

![](https://bbs.kanxue.com/upload/attach/202402/963320_8KUDETUFDP2WBGE.png)

接下来我们想知道的是究竟是从什么地方开始被填充了新的数据, 所以我们可以用 WinMerge 来让填充和未填充数据的 so 进行比较看看, 结果却有了惊人的发现, 被填充的数据是从`0xe7000`开始的, 它的开头竟然是 ELF 文件的魔数头!? 这有意思了, 那么就是一个 so 里面藏了另外一个 so 咯~

![](https://bbs.kanxue.com/upload/attach/202402/963320_JHC4VWFPFHVGKM2.png)

我们写个 python 脚本, 把这个 ELF 从`0x0e7000`开始后面的所有字节都复制到新的文件里面

```
with open('libjiagu_64.so_0x7a69829000_0x274000_open_classes.dex.so','rb') as f:
    s=f.read()
with open('libjiagu_0xe7000.so','wb') as f:
    f.write(s[0xe7000::])

```

但是当把这个 elf 提取出来之后拿`010editor`看却发现`program header table`被加密了

![](https://bbs.kanxue.com/upload/attach/202402/963320_6WFEBTZ6389NCR6.png)

这就导致 ida 根本就无法进行正常的分析

![](https://bbs.kanxue.com/upload/attach/202402/963320_VEHWN8XG2GTN7RA.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_GCR3CEHD29H5XFT.png)

主 ELF 解密流程分析
============

壳 elf 加载主 elf, 并且 program header 还被加密了, 感觉这种形式很像是 **自实现 linker 加固 so**

对于这种加固方式, 壳 elf 在代码中自己实现了解析 ELF 文件的函数, 并将解析结果赋值到`soinfo`结构体中, 随后调用`dlopen`进行手动加载

来到 ida 里面在导入表对`dlopen`进行交叉引用, 我们看到 dlopen 有 5 个交叉引用

![](https://bbs.kanxue.com/upload/attach/202402/963320_G5D886AQ4PWUE6R.png)

看到第二个交叉引用, 来到`sub_3C94`函数, 这个 for 循环看起来像是在用符号表通过 dlopen 加载依赖项

![](https://bbs.kanxue.com/upload/attach/202402/963320_WMXWCWVJ4G9PEVH.png)

向上面翻翻代码, 看到这个 switch 就知道找对地方了, 这里应该就是自实现 linker 来加载 so 的

![](https://bbs.kanxue.com/upload/attach/202402/963320_NQ66CZQ8ZSYFKK5.png)

因为这和 AOSP 源码 (`android-platform\bionic\linker\linker.cpp`) 中的预链接 (`soinfo::prelink_image`) 这部分的操作极为的相似

![](https://bbs.kanxue.com/upload/attach/202402/963320_MZHDQFWNU7BREU2.png)

那接下来就在 ida 中导入 soinfo 相关的符号就可以啦

在 ida 中依次点击`View->Open subviews->Local Types`, 然后按下键盘上的`Insert`将下面的结构体添加到对话框中

```
//IMPORTANT
//ELF64启用该宏
#define __LP64__  1
//ELF32启用该宏
//#define __work_around_b_24465209__  1
 
/*
//https://android.googlesource.com/platform/bionic/+/master/linker/Android.bp
架构为32位 定义__work_around_b_24465209__宏
arch: {
        arm: {cflags: ["-D__work_around_b_24465209__"],},
        x86: {cflags: ["-D__work_around_b_24465209__"],},
    }
*/
 
//android-platform\bionic\libc\include\link.h
#if defined(__LP64__)
#define ElfW(type) Elf64_ ## type
#else
#define ElfW(type) Elf32_ ## type
#endif
 
//android-platform\bionic\linker\linker_common_types.h
// Android uses RELA for LP64.
#if defined(__LP64__)
#define USE_RELA 1
#endif
 
//android-platform\bionic\libc\kernel\uapi\asm-generic\int-ll64.h
//__signed__-->signed
typedef signed char __s8;
typedef unsigned char __u8;
typedef signed short __s16;
typedef unsigned short __u16;
typedef signed int __s32;
typedef unsigned int __u32;
typedef signed long long __s64;
typedef unsigned long long __u64;
 
//A12-src\msm-google\include\uapi\linux\elf.h
/* 32-bit ELF base types. */
typedef __u32   Elf32_Addr;
typedef __u16   Elf32_Half;
typedef __u32   Elf32_Off;
typedef __s32   Elf32_Sword;
typedef __u32   Elf32_Word;
 
/* 64-bit ELF base types. */
typedef __u64   Elf64_Addr;
typedef __u16   Elf64_Half;
typedef __s16   Elf64_SHalf;
typedef __u64   Elf64_Off;
typedef __s32   Elf64_Sword;
typedef __u32   Elf64_Word;
typedef __u64   Elf64_Xword;
typedef __s64   Elf64_Sxword;
 
typedef struct dynamic{
  Elf32_Sword d_tag;
  union{
    Elf32_Sword d_val;
    Elf32_Addr  d_ptr;
  } d_un;
} Elf32_Dyn;
 
typedef struct {
  Elf64_Sxword d_tag;       /* entry tag value */
  union {
    Elf64_Xword d_val;
    Elf64_Addr d_ptr;
  } d_un;
} Elf64_Dyn;
 
typedef struct elf32_rel {
  Elf32_Addr    r_offset;
  Elf32_Word    r_info;
} Elf32_Rel;
 
typedef struct elf64_rel {
  Elf64_Addr r_offset;  /* Location at which to apply the action */
  Elf64_Xword r_info;   /* index and type of relocation */
} Elf64_Rel;
 
typedef struct elf32_rela{
  Elf32_Addr    r_offset;
  Elf32_Word    r_info;
  Elf32_Sword   r_addend;
} Elf32_Rela;
 
typedef struct elf64_rela {
  Elf64_Addr r_offset;  /* Location at which to apply the action */
  Elf64_Xword r_info;   /* index and type of relocation */
  Elf64_Sxword r_addend;    /* Constant addend used to compute value */
} Elf64_Rela;
 
typedef struct elf32_sym{
  Elf32_Word    st_name;
  Elf32_Addr    st_value;
  Elf32_Word    st_size;
  unsigned char st_info;
  unsigned char st_other;
  Elf32_Half    st_shndx;
} Elf32_Sym;
 
typedef struct elf64_sym {
  Elf64_Word st_name;       /* Symbol name, index in string tbl */
  unsigned char st_info;    /* Type and binding attributes */
  unsigned char st_other;   /* No defined meaning, 0 */
  Elf64_Half st_shndx;      /* Associated section index */
  Elf64_Addr st_value;      /* Value of the symbol */
  Elf64_Xword st_size;      /* Associated symbol size */
} Elf64_Sym;
 
 
#define EI_NIDENT   16
 
typedef struct elf32_hdr{
  unsigned char e_ident[EI_NIDENT];
  Elf32_Half    e_type;
  Elf32_Half    e_machine;
  Elf32_Word    e_version;
  Elf32_Addr    e_entry;  /* Entry point */
  Elf32_Off e_phoff;
  Elf32_Off e_shoff;
  Elf32_Word    e_flags;
  Elf32_Half    e_ehsize;
  Elf32_Half    e_phentsize;
  Elf32_Half    e_phnum;
  Elf32_Half    e_shentsize;
  Elf32_Half    e_shnum;
  Elf32_Half    e_shstrndx;
} Elf32_Ehdr;
 
typedef struct elf64_hdr {
  unsigned char e_ident[EI_NIDENT]; /* ELF "magic number" */
  Elf64_Half e_type;
  Elf64_Half e_machine;
  Elf64_Word e_version;
  Elf64_Addr e_entry;       /* Entry point virtual address */
  Elf64_Off e_phoff;        /* Program header table file offset */
  Elf64_Off e_shoff;        /* Section header table file offset */
  Elf64_Word e_flags;
  Elf64_Half e_ehsize;
  Elf64_Half e_phentsize;
  Elf64_Half e_phnum;
  Elf64_Half e_shentsize;
  Elf64_Half e_shnum;
  Elf64_Half e_shstrndx;
} Elf64_Ehdr;
 
/* These constants define the permissions on sections in the program
   header, p_flags. */
#define PF_R        0x4
#define PF_W        0x2
#define PF_X        0x1
 
typedef struct elf32_phdr{
  Elf32_Word    p_type;
  Elf32_Off p_offset;
  Elf32_Addr    p_vaddr;
  Elf32_Addr    p_paddr;
  Elf32_Word    p_filesz;
  Elf32_Word    p_memsz;
  Elf32_Word    p_flags;
  Elf32_Word    p_align;
} Elf32_Phdr;
 
typedef struct elf64_phdr {
  Elf64_Word p_type;
  Elf64_Word p_flags;
  Elf64_Off p_offset;       /* Segment file offset */
  Elf64_Addr p_vaddr;       /* Segment virtual address */
  Elf64_Addr p_paddr;       /* Segment physical address */
  Elf64_Xword p_filesz;     /* Segment size in file */
  Elf64_Xword p_memsz;      /* Segment size in memory */
  Elf64_Xword p_align;      /* Segment alignment, file & memory */
} Elf64_Phdr;
 
typedef struct elf32_shdr {
  Elf32_Word    sh_name;
  Elf32_Word    sh_type;
  Elf32_Word    sh_flags;
  Elf32_Addr    sh_addr;
  Elf32_Off sh_offset;
  Elf32_Word    sh_size;
  Elf32_Word    sh_link;
  Elf32_Word    sh_info;
  Elf32_Word    sh_addralign;
  Elf32_Word    sh_entsize;
} Elf32_Shdr;
 
typedef struct elf64_shdr {
  Elf64_Word sh_name;       /* Section name, index in string tbl */
  Elf64_Word sh_type;       /* Type of section */
  Elf64_Xword sh_flags;     /* Miscellaneous section attributes */
  Elf64_Addr sh_addr;       /* Section virtual addr at execution */
  Elf64_Off sh_offset;      /* Section file offset */
  Elf64_Xword sh_size;      /* Size of section in bytes */
  Elf64_Word sh_link;       /* Index of another section */
  Elf64_Word sh_info;       /* Additional section information */
  Elf64_Xword sh_addralign; /* Section alignment */
  Elf64_Xword sh_entsize;   /* Entry size if section holds table */
} Elf64_Shdr;
 
 
//android-platform\bionic\linker\linker_soinfo.h
typedef void (*linker_dtor_function_t)();
typedef void (*linker_ctor_function_t)(int, char**, char**);
 
#if defined(__work_around_b_24465209__)
#define SOINFO_NAME_LEN 128
#endif
 
struct soinfo {
#if defined(__work_around_b_24465209__)
  char old_name_[SOINFO_NAME_LEN];
#endif
  const ElfW(Phdr)* phdr;
  size_t phnum;
#if defined(__work_around_b_24465209__)
  ElfW(Addr) unused0; // DO NOT USE, maintained for compatibility.
#endif
  ElfW(Addr) base;
  size_t size;
 
#if defined(__work_around_b_24465209__)
  uint32_t unused1;  // DO NOT USE, maintained for compatibility.
#endif
 
  ElfW(Dyn)* dynamic;
 
#if defined(__work_around_b_24465209__)
  uint32_t unused2; // DO NOT USE, maintained for compatibility
  uint32_t unused3; // DO NOT USE, maintained for compatibility
#endif
 
  soinfo* next;
  uint32_t flags_;
 
  const char* strtab_;
  ElfW(Sym)* symtab_;
 
  size_t nbucket_;
  size_t nchain_;
  uint32_t* bucket_;
  uint32_t* chain_;
 
#if !defined(__LP64__)
  ElfW(Addr)** unused4; // DO NOT USE, maintained for compatibility
#endif
 
#if defined(USE_RELA)
  ElfW(Rela)* plt_rela_;
  size_t plt_rela_count_;
 
  ElfW(Rela)* rela_;
  size_t rela_count_;
#else
  ElfW(Rel)* plt_rel_;
  size_t plt_rel_count_;
 
  ElfW(Rel)* rel_;
  size_t rel_count_;
#endif
 
  linker_ctor_function_t* preinit_array_;
  size_t preinit_array_count_;
 
  linker_ctor_function_t* init_array_;
  size_t init_array_count_;
  linker_dtor_function_t* fini_array_;
  size_t fini_array_count_;
 
  linker_ctor_function_t init_func_;
  linker_dtor_function_t fini_func_;
 
/*
#if defined(__arm__)
  // ARM EABI section used for stack unwinding.
  uint32_t* ARM_exidx;
  size_t ARM_exidx_count;
#endif
  size_t ref_count_;
//怎么找不到link_map这个类型的声明...
  link_map link_map_head;
 
  bool constructors_called;
 
  // When you read a virtual address from the ELF file, add this
  // value to get the corresponding address in the process' address space.
  ElfW(Addr) load_bias;
 
#if !defined(__LP64__)
  bool has_text_relocations;
#endif
  bool has_DT_SYMBOLIC;
*/
};

```

导入完成后按下`Y`键, 将`a1`定义为`soinfo*`

![](https://bbs.kanxue.com/upload/attach/202402/963320_2JB9E93A6TPTMMN.png)

然后就可以看到这些符号了, 但是看这些符号总感觉有些不太对劲, 这里不应该出现`a1[1]`或者`a1[2]`, 所以我猜测这个 soinfo 有被魔改的痕迹

![](https://bbs.kanxue.com/upload/attach/202402/963320_TTB39SUSMXHQKGB.png)

虽然这个 soinfo 可能有被魔改了, 我们还是从`sub_3C94`这个预链接相关函数入手好了, 交叉引用发现`sub_3C94`是被`sub_49F0`调用

随后我们来到`sub_49F0`内调用`sub_3C94`函数的位置, 向下看, 进入`sub_4918`函数中

![](https://bbs.kanxue.com/upload/attach/202402/963320_SVUYVK4PH3MAMPK.png)

`sub_4918`中调用了`sub_5E6C`, 我们进入`sub_5E6C`

![](https://bbs.kanxue.com/upload/attach/202402/963320_62M6YZTBSFQK8UV.png)

这个函数中出现了`0x38`这个数字,`0x38`是这个循环的步长

![](https://bbs.kanxue.com/upload/attach/202402/963320_NH5ZAE6B6KVKB63.png)

0x38 这个数字有什么特殊的含义吗? 当然有了!!

我们把刚刚提取出来的 elf 用`010editor`打开, 看到`elf_header`的`phentsize`这个字段, 这个字段的含义是一个`Program header table`的长度, 它正正好好也是`0x38`

![](https://bbs.kanxue.com/upload/attach/202402/963320_KX7EW9RXYDDURXA.png)

所以说在`sub_5E6C`中变量`v5`的类型应该是`Elf64_Phdr *`, 我们直接重定义类型

![](https://bbs.kanxue.com/upload/attach/202402/963320_UE2JDDD3GFWP5QV.png)

既然知道了真正的`program header table`就是在这个位置的, 那我们直接在这个地方把`program header table`整个给 dump 下来不就行了

所以我们直接去 hook`sub_5E6C`的三个传入的值

```
function hook_5E6C(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x5E6C), {
        // fd, buff, len
        onEnter: function (args) {
            console.log(hexdump(args[0], {
              offset: 0,// 相对偏移
              length: 0x38*0x6+0x20,//dump 的大小
              header: true,
              ansi: true
            }));
            console.log(args[1])
            console.log(args[2])
            console.log(`base = ${module.base}`)
        },
        onLeave: function (ret) {
        }
    });
}

```

![](https://bbs.kanxue.com/upload/attach/202402/963320_YPZBWW68B5SX6DS.png)

上面的第一个 hexdump 就是`program header table`, 我们可以用`cyberchef`将`hexdump`转成数组的形式

![](https://bbs.kanxue.com/upload/attach/202402/963320_YSMNQETKM3FQTT4.png)

`0x6`则对应着`phnum`, 这表示共有 6 个`program header table`

`0x793ca38000`表示这个主 ELF 的基址, 因为这个主 ELF 的位置在壳 ELF 基址的偏移`0xe7000`处, 而最下面这行也已经打印出了壳 ELF 的基址为`0x793c951000`,`0x793ca38000==0x793c951000+0xe7000`等式成立

至此为止, 我们拿到了解密之后的`program header table`, 同时我们也知道了`sub_5E6C`传入的三个参数分别是`phdr`,`phnum`以及`base`

但是`phdr`成员命名是在 soinfo 偏移的`0x0`的位置

![](https://bbs.kanxue.com/upload/attach/202402/963320_NTMK74J4N35WFRZ.png)

那假如`a1`的类型就是`soinfo*`, 为什么在`sub_4918`里面调用`sub_5E6C`传入的是偏移是`232`呢?

![](https://bbs.kanxue.com/upload/attach/202402/963320_P6G6Z2A3THKJ8TW.png)

所以`soinfo*`必定有被魔改, 同时我们也可以在 soinfo 前填充一个大小为 232 的 char 类型数组看看是什么情况

![](https://bbs.kanxue.com/upload/attach/202402/963320_Q5V285KKW97X42A.png)

很好, 这验证了我们对于`soinfo*`被魔改的猜测, 因为在一切正常的情况之下, 函数的调用应该是`sub_5E6C(a1->phdr, a1->phnum, a1->base)`才对

![](https://bbs.kanxue.com/upload/attach/202402/963320_3V72AXUNJD3UDHH.png)

但是我很想知道这个壳 ELF 究竟是如何被解密出来的, 那么首先来看看主 ELF 的函数调用链是什么样子的吧~

我写了一个 ida 插件来实现这个过程 [stalker_trace_so](https://github.com/oacia/stalker_trace_so)

在 IDA 中使用`Edit->Plugins->stalker_trace_so`后, 在 so 所在的目录下会生成一个 js 脚本, 我们用 frida 注入到 apk 中即可, 需要注意的是`so_name`需要改成`libjiagu_64.so`

![](https://bbs.kanxue.com/upload/attach/202402/963320_5RSEFHPK7T3H9P9.png)

打印出来的完整日志如下

```
call1:JNI_OnLoad
call2:j_interpreter_wrap_int64_t
call3:interpreter_wrap_int64_t 
call4:getenv
call5:sub_13908
call6:inotify_add_watch
call7:sub_11220
call8:fopen
call9:sub_9DD8
call10:sub_E3E0
call11:strtol
call12:feof
call13:raise
call14:memset
call15:sub_C918
call16:sub_9988
call17:sub_9964
call18:sub_9AC4
call19:j_ffi_prep_cif
call20:ffi_prep_cif
call21:j_ffi_prep_cif_machdep
call22:ffi_prep_cif_machdep
call23:j_ffi_call
call24:ffi_call
call25:sub_1674C
call26:j_ffi_call_SYSV
call27:ffi_call_SYSV
call28:sub_167BC
call29:sub_1647C
call30:sub_163DC
call31:sub_9900
call32:sub_94BC
call33:inotify_init
call34:fmod
call35:strncpy
call36:_Z9__arm_a_1P7_JavaVMP7_JNIEnvPvRi
call37:sub_9E58
call38:sub_999C
call39:sub_10964
call40:j_lseek_1
call41:lseek
call42:sub_96E0
call43:sub_8000
call44:dlopen
call45:sub_60E0
call46:sub_6544
call47:sub_4B54
call48:sub_6128
call49:_ZN9__arm_c_19__arm_c_0Ev
call50:sub_A3EC
call51:sub_99CC
call52:sub_9944
call53:sub_6484
call54:sub_6590
call55:prctl
call56:sub_6698
call57:sub_9FFC
call58:j_lseek_3
call59:j_lseek_2
call60:j_lseek_0
call61:sub_9A90
call62:sub_5F20
call63:sub_6044
call64:sub_3574
call65:uncompress
call66:sub_49F0
call67:sub_5400
call68:sub_5478
call69:sub_5B08
call70:sub_5650
call71:sub_580C
call72:open
call73:atoi
call74:sub_3C94
call75:strncmp
call76:sub_4918
call77:sub_4000
call78:sub_41B4
call79:sub_35AC
call80:sigaction
call81:sub_5E6C
call82:sub_5444
call83:sub_633C
call84:sub_8130
call85:sub_4C70
call86:sub_825C
call87:sub_8B50
call88:sub_8ED4
call89:sub_8430
call90:interpreter_wrap_int64_t_bridge
call91:sub_9D60
call92:sub_166C4
call93:memcpy
call94:_Z9__arm_a_2PcmS_Rii
call95:j_ffi_prep_cif_var
call96:ffi_prep_cif_var

```

我们以`sub_3C94`为起点开始分析, 因为这是我们通过`dlopen`交叉引用找到的**自实现 linker 加固 so** 的一个功能函数

对`sub_3C94`不断按下`X`查看交叉引用, 得到如下的调用关系`sub_4B54->sub_49F0->sub_3C94`

`sub_4B54`可能被`sub_8000`或`sub_8C74`调用

![](https://bbs.kanxue.com/upload/attach/202402/963320_QKT7M8V5CR7XWBJ.png)

我们将`stalker_trace_so`打印出来的内容中, 提取关键的部分拿过来看看, 说明`sub_3B54`是被`sub_8000`调用的

```
call43:sub_8000 <--
call44:dlopen
call45:sub_60E0
call46:sub_6544
call47:sub_4B54 <--
call48:sub_6128
call49:_ZN9__arm_c_19__arm_c_0Ev
call50:sub_A3EC
call51:sub_99CC
call52:sub_9944
call53:sub_6484
call54:sub_6590
call55:prctl
call56:sub_6698
call57:sub_9FFC
call58:j_lseek_3
call59:j_lseek_2
call60:j_lseek_0
call61:sub_9A90
call62:sub_5F20
call63:sub_6044
call64:sub_3574
call65:uncompress
call66:sub_49F0 <--
call67:sub_5400
call68:sub_5478
call69:sub_5B08
call70:sub_5650
call71:sub_580C
call72:open
call73:atoi
call74:sub_3C94 <--

```

`sub_8000`的函数长这个样子, 请记住第 25 行`0xB8010`这个数字, 后面会派上用场的

![](https://bbs.kanxue.com/upload/attach/202402/963320_6SX6YBJM6DQWCKF.png)

跟着函数调用链一处一处的在 IDA 中跳转到相应的地址进行查看, 在`call62:sub_5F20`我们发现了有意思的代码

这个函数, 一眼 RC4 呀

![](https://bbs.kanxue.com/upload/attach/202402/963320_NDCVYQHNFT89YQR.png)

用 frida 去 hook 一下这个函数看看 RC4 的密钥是什么

```
function hook_5f20_guess_rc4(){//像是RC4的样子,hook看看
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x5f20), {
        // fd, buff, len
        onEnter: function (args) {
            console.log(hexdump(args[0], {
              offset: 0,// 相对偏移
              length: 0x10,//dump 的大小
              header: true,
              ansi: true
            }));
            console.log(args[1])
            console.log(hexdump(args[2], {
              offset: 0,// 相对偏移
              length: 256,//dump 的大小
              header: true,
              ansi: true
            }));
        },
        onLeave: function (ret) {
 
        }
    });
}

```

![](https://bbs.kanxue.com/upload/attach/202402/963320_Z9PFFE26C99Q8XG.png)

所以密钥就是这个咯

```
key = b"vUV4#\x91#SVt"

```

继续跟着函数调用链走, 在`call63:sub_6044`我们发现了 RC4 的解密函数

![](https://bbs.kanxue.com/upload/attach/202402/963320_54VHCKBQXZ8HYY8.png)

hook 一下`call63:sub_6044`看看到底给什么数据解密了

```
var rc4_enc_text_addr,rc4_enc_size;
function hook_rc4_enc(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x6044), {
        // fd, buff, len
        onEnter: function (args) {
            rc4_enc_text_addr = args[0];
            rc4_enc_size = args[1];
            console.log(hexdump(args[0], {
              offset: 0,// 相对偏移
              length: 0x30,//dump 的大小
              header: true,
              ansi: true
            }));
            console.log(args[1])
 
        },
        onLeave: function (ret) {
            console.log(hexdump(rc4_enc_text_addr, {
              offset: 0,// 相对偏移
              length: 0x30,//dump 的大小
              header: true,
              ansi: true
            }));
        }
    });
}

```

![](https://bbs.kanxue.com/upload/attach/202402/963320_KMTRP3X878UJJTK.png)

这个函数的第二个参数是`0xb8010`, 感觉是解密的数据的长度的样子, 而且这个数字, 有没有感觉在哪里见过呢?

没错, 这个数字刚刚就出现在`sub_8000`中

![](https://bbs.kanxue.com/upload/attach/202402/963320_BUEQPTKH6HHZ7SU.png)

而`v5[0]`的值是`qword_2E270`, 这个数组也是`01 18 25 e7`开头的

![](https://bbs.kanxue.com/upload/attach/202402/963320_3EUHYVSZ4NGW9SS.png)

继续跟着调用链走, 接下来是调用`call65:uncompress`, 进行解压缩操作

```
function hook_uncompress_res(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(Module.findExportByName(null, "uncompress"), {
        onEnter: function (args) {
            console.log("hook uncompress")
            console.log(hexdump(args[2], {
              offset: 0,// 相对偏移
              length: 0x30,//dump 的大小
              header: true,
              ansi: true
            }));
            console.log(args[3])
            dump_memory(args[2],args[3],`uncompress_${args[2]}_${args[3]}`)
        },
        onLeave: function (ret) {
 
        }
    });
}

```

我们发现解压缩的数据, 前面四个字节`b9 0e 1a 00`没有包含在解压缩的字节之内

![](https://bbs.kanxue.com/upload/attach/202402/963320_46P6XGB8EVT9KCX.png)

现在既然我们已经知道了主 ELF 在壳 ELF 中的位置, 以及解密的算法, 那我们直接从解压 apk, 找到里面的`assets/libjiagu_a64.so`, 不就能直接把壳 ELF 解密出来咯

```
import zlib
import struct
def RC4(data, key):
    S = list(range(256))
    j = 0
    out = []
 
    # KSA Phase
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]
 
    # PRGA Phase
    i = j = 0
    for ch in data:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        out.append(ch ^ S[(S[i] + S[j]) % 256])
 
    return out
 
def RC4decrypt(ciphertext, key):
    return RC4(ciphertext, key)
 
 
wrap_elf_start = 0x1e270
wrap_elf_size = 0xb8010
key = b"vUV4#\x91#SVt"
with open('com.oacia.apk_protect/assets/libjiagu_a64.so','rb') as f:
    wrap_elf = f.read()
 
 
# 对密文进行解密
dec_compress_elf = RC4decrypt(wrap_elf[wrap_elf_start:wrap_elf_start+wrap_elf_size], key)
dec_elf = zlib.decompress(bytes(dec_compress_elf[4::]))
with open('wrap_elf','wb') as f:
    f.write(dec_elf)

```

解密完成后, 我们发现`0x1a0eb9`应该表示解压缩之后数据的大小

![](https://bbs.kanxue.com/upload/attach/202402/963320_G7DGM3PCAGJ2AYT.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_CP78MFKPAE9PAFT.png)

`wrap_elf`的前半部分是一大堆莫名其妙有很多`D3`的东西, 但是看到中间还是发现了壳 ELF 的身影

![](https://bbs.kanxue.com/upload/attach/202402/963320_UCEQQDKJTTXHZKJ.png)

我们以`.ELF`为标志将这两部分分离一下

```
with open('wrap_elf', 'rb') as f:
    wrap_elf = f.read()
ELF_magic = bytes([0x7F, 0x45, 0x4C, 0x46])
for i in range(len(wrap_elf) - len(ELF_magic) + 1):
    if wrap_elf[i:i + len(ELF_magic)] == ELF_magic:
        print(hex(i))
        with open('wrap_elf_part1', 'wb') as f:
            f.write(wrap_elf[0:i])
        with open('wrap_elf_part2', 'wb') as f:
            f.write(wrap_elf[i::])
        break

```

跟着函数调用链来到`call69:sub_5B08`, 这里又出现了`0x38`, 并且`word_38`跳转过去的值为 6

![](https://bbs.kanxue.com/upload/attach/202402/963320_7GFNFDD8W3T8KMJ.png)

这正好和`phentsize`和`phnum`的值相对应

![](https://bbs.kanxue.com/upload/attach/202402/963320_QWVTNTWH3SE5M2Q.png)

所以可想而知, 这又是一个关键点了, 往下看一下代码, 发现了循环异或, 那我们不妨用 frida 把`v4`的值 hook 下来看看是什么

![](https://bbs.kanxue.com/upload/attach/202402/963320_VAJCC78K8WXXV62.png)

v4 的值出现了那么多的`d3`

![](https://bbs.kanxue.com/upload/attach/202402/963320_23N4A89WUZ77KNZ.png)

而这就是`wrap_elf`的前半部分那一大堆我们看不懂的字节

![](https://bbs.kanxue.com/upload/attach/202402/963320_Q2Q79X9FTSNGHJ9.png)

接下来用来解密的循环就是一个 arm64 的 neon 运算

![](https://bbs.kanxue.com/upload/attach/202402/963320_QAKQX4M2ZWYTDKR.png)

官网可以找到 [vdupq_n_s8](https://developer.arm.com/architectures/instruction-sets/intrinsics/#q=vdupq_n_s8) 和 [veorq_s8](https://developer.arm.com/architectures/instruction-sets/intrinsics/#q=veorq_s8), 根据函数描述可以知道这里用向量运算, 把向量中的每一个元素都异或了`0xd3`

![](https://bbs.kanxue.com/upload/attach/202402/963320_SHVPGU5YYTTFVVW.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_4A3Y46FRSRBZB7P.png)

对`sub_5B08`进行分析之后, 我们便可以知道`wrap_elf_part1`的读取方式是第一个字节表示被异或的数字, 这里是`0xD3`, 后面的四个字节表示一个段的长度, 随后读取指定长度的字节并异或, 之后再读取四个字节获取到下一个段的长度, 以此类推, 直到读取到文件末尾

![](https://bbs.kanxue.com/upload/attach/202402/963320_WXJ6XACGDK2S3CY.png)

在`sub_5B08`的最后, 因为`v31`,`v19`,`v43`,`v7`代表对应的数据组的长度, 所以这里共有四个数据组, 而为了表示每一个数据组的长度共需占用`4*4=16`字节, 并且文件开头还有`1`位的异或值, 于是这些长度加起来,`*(a1 + 0x98)`的偏移就来到了主 ELF 的魔术头`.ELF`的位置了

![](https://bbs.kanxue.com/upload/attach/202402/963320_GVNTTZT9EXGHAXV.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_Q5A7C85RJX4MYH5.png)

我们可以在`sub_5B08`中为变量`a1`定义一个结构体, 成员分别表示数据组的 1,2,3,4 这四个部分, 这样我们就知道这四个部分分别被用到什么地方了

```
struct deal_extra
{
  char blank[72];
  int phnum;
  int *extra_part1;
  int phdr_size;
  char blank2[36];
  int *extra_part2;
  int *extra_part3;
  int *extra_part4;
  int *main_elf;
};

```

![](https://bbs.kanxue.com/upload/attach/202402/963320_3H38Q9AY2XCBQKU.png)

接下来再捋一下函数的调用链`sub_49F0->sub_5478(&v16, a1, v4)->sub_5B08(a1, a2, a3)`, 在`sub_5B08`中, 我们把`a1`的类型定义成了`deal_extra`, 所以理所应当的, 我们也把`sub_49F0`中的变量`v16`的类型定义为`deal_extra`

在`sub_49F0`中我们发现成员`extra_part`赋值给了变量`v7`, 所以我们也为`v7`建立一个结构体让 v7 的偏移可以对应这些变量

![](https://bbs.kanxue.com/upload/attach/202402/963320_W74FU4E6HQNQ9VK.png)

```
struct deal_extra_B
{
  char blank[232];
  int *extra_part1;
  char blank1[8];
  int phnum;
  int *extra_part4;
  char blank2[24];
  int *extra_part2;
  char blank3[8];
  int *extra_part3;
};

```

![](https://bbs.kanxue.com/upload/attach/202402/963320_7F74SSG5B5QET7B.png)

这样做有什么意义呢?

我们发现变量`v7`分别被传入到了`sub_3C94`和`sub_4918`中, 我们分别进去看看

![](https://bbs.kanxue.com/upload/attach/202402/963320_5TD6NVR2MGJMD2Q.png)

在`sub_3C94`中解析了`extra_part4`, 显而易见, 这个`switch`是用来处理动态链接库的, 即`extra_part4`对应`.dynamic`段

![](https://bbs.kanxue.com/upload/attach/202402/963320_YYSTVUGSAHTBJ55.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_AY8747TPUH8T87Y.png)

在`sub_4918`中,`extra_part2`和`extra_part3`被传入到`sub_4000`中

![](https://bbs.kanxue.com/upload/attach/202402/963320_6J73FGUTG5A2FNQ.png)

而这个函数中的`switch`是用来处理重定位的, 因为重定位主要有基址重定位和符号重定位, 这两个的值分别是`0x403`和`0x402`

所以`extra_part2`和`extra_part3`分别对应着`.rela.dyn`(403 重定位) 和`.rela.plt`(402 重定位)

![](https://bbs.kanxue.com/upload/attach/202402/963320_C8HY3BX7UVE4RQP.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_N84RVE27PD8AHVQ.png)

而之后`extra_part1`被传入到了`sub_5E6C`中

![](https://bbs.kanxue.com/upload/attach/202402/963320_6JQUN253DSMRU78.png)

而来到`sub_5E6C`也来到了我们最开始分析的起点 (兜兜转转又回来了), 所以`extra_part1`表示`program header table`

![](https://bbs.kanxue.com/upload/attach/202402/963320_D9AVRZXDSQZ79HM.png)

至此为止, 四个数据组所对应的段都分析完成

*   `数据组1`表示`program header table`
*   `数据组2`表示`.rela.plt`
*   `数据组3`表示`.rela.dyn`
*   `数据组4`表示`.dynamic`

所以接下来, 写个脚本把这四个数据组给分离成单独的文件咯

```
import copy
import zlib
 
def RC4(data, key):
    S = list(range(256))
    j = 0
    out = []
 
    # KSA Phase
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]
 
    # PRGA Phase
    i = j = 0
    for ch in data:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        out.append(ch ^ S[(S[i] + S[j]) % 256])
 
    return out
 
 
def RC4decrypt(ciphertext, key):
    return RC4(ciphertext, key)
 
 
wrap_elf_start = 0x1e270
wrap_elf_size = 0xb8010
key = b"vUV4#\x91#SVt"
with open('com.oacia.apk_protect/assets/libjiagu_a64.so', 'rb') as f:
    wrap_elf = f.read()
 
# 对密文进行解密
dec_compress_elf = RC4decrypt(wrap_elf[wrap_elf_start:wrap_elf_start + wrap_elf_size], key)
dec_elf = zlib.decompress(bytes(dec_compress_elf[4::]))
with open('wrap_elf', 'wb') as f:
    f.write(dec_elf)
 
 
class part:
    def __init__(self):
        self.name = ""
        self.value = b''
        self.offset = 0
        self.size = 0
 
 
index = 1
extra_part = [part() for _ in range(7)]
 
seg = ["phdr", ".rela.plt", ".rela.dyn", ".dynamic"]
v_xor = dec_elf[0]
 
for i in range(4):
    size = int.from_bytes(dec_elf[index:index + 4], 'little')
    index += 4
    extra_part[i + 1].name = seg[i]
    extra_part[i + 1].value = bytes(map(lambda x: x ^ v_xor, dec_elf[index:index + size]))
    extra_part[i + 1].size = size
    index += size
 
for p in extra_part:
    if p.value!=b'':
        filename = f"libjiagu.so_{hex(p.size)}_{p.name}"
        print(f"[{p.name}] get {filename}, size: {hex(p.size)}")
        with open(filename,'wb') as f:
            f.write(p.value)

```

于是我们得到了这四个文件

![](https://bbs.kanxue.com/upload/attach/202402/963320_DFAMDEAEMR9H77D.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_J6CVKAGN2K6MN68.png)

主 ELF 导入导出表修复
=============

需要被修复的主 ELF 是我们在从`assets/libjiagu_a64.so`利用`RC4`和`decompress`解密出来的文件的后半部分那个 ELF

可以写个 python 脚本分离出后面的 ELF

```
with open('wrap_elf', 'rb') as f:
    wrap_elf = f.read()
ELF_magic = bytes([0x7F, 0x45, 0x4C, 0x46])
for i in range(len(wrap_elf) - len(ELF_magic) + 1):
    if wrap_elf[i:i + len(ELF_magic)] == ELF_magic:
        with open('libjiagu_0xe7000.so', 'wb') as f:
            f.write(wrap_elf[i::])
        break

```

现在我们拿到了主 ELF 的四个重要的数据段, 分别是`phdr`,`.rela.plt`,`.rela.dyn`,`.dynamic`, 那么接下来需要做的工作就是修复主 ELF 的导入导出表了, 不然导入导出函数都看不见怎么逆嘞~

在使用自实现 linker 加固 so 时,`phdr`,`.rela.plt`,`.rela.dyn`,`.dynamic`这四个段是从待加固的 so 中提取出来, 然后加密存储到其他位置, **原来的位置会使用无关字节直接覆盖**

等到需要为加固的 so 进行预链接和重定位的工作时, 才将这些段解密并通过自己实现的预链接和重定位代码, 让待加固的 so 可以正确的被壳 so 加载出来

我们进行修复的方法其实就藏在这句话中`原来的位置会使用无关字节直接覆盖`, 我们可以将分离出来的这四个段再塞回到原来的位置

`自实现linker加固so`的加固方案既然都把那四个段加密存到其他地方了, 那怎么不直接把原来的四个段直接删除而是用无关字节覆盖呢?

因为直接把段删除掉的话, 会影响了一整个 ELF 文件的布局, 偏移就会变得和原先不一样, 然后产生各种奇奇怪怪的问题

> 在 010editor 中, 按下`ctrl+shift+C`可以复制整块内存, 按下`ctrl+shift+V`可以粘贴整块内存

1.  修复`program header table`
    
    复制`libjiagu.so_0x150_phdr`的所有字节, 然后来到`libjiagu_0xe7000.so`中选中`struct program_header_table`粘贴
    
    ![](https://bbs.kanxue.com/upload/attach/202402/963320_RFKKMSDZ5CB84UU.png)
    
    随后按下`F5`刷新模板
    
2.  修复`.dynamic`
    
    `program header table`的`(RW_) Dynamic Segment`的`p_offset`指向`.dynamic`段的位置  
    ![](https://bbs.kanxue.com/upload/attach/202402/963320_4Z2XMUHDACQVJP6.png)  
    跳转到该位置, 复制`libjiagu.so_0x1b0_.dynamic`的内容并粘贴到这个位置
    
3.  修复重定位表  
    我们需要通过`.dynamic`段的`d_tag`字段来直到重定位表的位置, 下面是 AOSP 中`d_tag`的宏定义  
    ![](https://bbs.kanxue.com/upload/attach/202402/963320_NG24GWRKJYN55RU.png)
    

所有的`d_tag`标志对应的含义可以在 [ORACLE 链接程序和库指南](https://docs.oracle.com/cd/E24847_01/html/E22196/toc.html) 中找到

对于我们修复主 ELF 比较重要的`tag`有

<table><thead><tr><th>d_tag</th><th>值</th><th>含义</th></tr></thead><tbody><tr><td>DT_JMPREL</td><td>0x17</td><td><code>.rela.plt</code>在文件中的偏移</td></tr><tr><td>DT_PLTRELSZ</td><td>0x2</td><td><code>.rela.plt</code>的大小</td></tr><tr><td>DT_RELA</td><td>0x7</td><td><code>.rela.dyn</code>在文件中的偏移</td></tr><tr><td>DT_RELASZ</td><td>0x8</td><td><code>.rela.dyn</code>的大小</td></tr></tbody></table>

我们可以在`.dynamic`中发现这些`tag`以及对应的值

![](https://bbs.kanxue.com/upload/attach/202402/963320_T8VW6KQAXA8S5E8.png)

看看这两个大小分别是`0x1650`和`0x25188`, 这不就和我们刚刚分离出来的文件大小一模一样嘛, 说明我们离修复完成不远了

![](https://bbs.kanxue.com/upload/attach/202402/963320_A2JXGJ3AED2E6HN.png)

然后就是和之前一样, 跳转到`.rela.plt`和`.rela.dyn`的对应地址, 然后把这些段本来的数据粘贴进去

现在我们就修复好啦, 拿 ida 打开主 ELF 看看, 满满的都是符号呢~

![](https://bbs.kanxue.com/upload/attach/202402/963320_DS65D39ETZ4PBSP.png)

随便找个导入函数交叉引用看看, 一切正常`(●'◡'●)`

![](https://bbs.kanxue.com/upload/attach/202402/963320_7HCT6N2J7M8Z4XT.png)

为了方便起见, 我们可以将主 ELF 的基址定义成在其在壳 ELF 的偏移`0xe7000`方便后续的分析

主 DEX 解密流程初步分析
==============

还记得在**加固壳反调试初步分析**中, 我们拿到了未解密的 dex 嘛

![](https://bbs.kanxue.com/upload/attach/202402/963320_NCB62GESABYUM84.png)

那么接下来有个问题就是, 这个未解密的 dex 究竟藏在了 apk 的什么地方呢?

我将未加固的 apk 解压出来, 然后用 7zip 压缩其中的 dex, 发现大小依然有 2.8MB

![](https://bbs.kanxue.com/upload/attach/202402/963320_4JP9N7U6EMCSPPY.png)

随后我将经过 360 加固之后的 apk 解压出来, 按大小对文件进行排序之后发现, 最大的文件就只有这个壳`classes.dex`, 而别的文件甚至连 1MB 都没到, 总不可能压缩率可以高到这种地步吧

![](https://bbs.kanxue.com/upload/attach/202402/963320_28ZMF67TE9RKV2G.png)

所以我们打开`classes.dex`看看, 在这个`classes.dex`的末尾, 果然藏着一大堆的数据

![](https://bbs.kanxue.com/upload/attach/202402/963320_GJP6VZB46NP5BKV.png)

而末尾的数据是由`71 68 00 01`和我们之前看到的加密的 dex 一模一样

接下来我们继续用 [stalker_trace_so](https://github.com/oacia/stalker_trace_so) 去看看补充上主 ELF 的函数地址以及名称之后的函数调用链是什么样子的, 首先在主 ELF 中运行插件`Edit->Plugins->stalker_trace_so`

之后同样的, 我们需要将`so_name`改成`libjiagu_64.so`, 特别注意的是, 这里我们需要把壳 ELF 的`func_addr`和`func_name`给复制过来, 同时使用`concat`方法将主 ELF 和壳 ELF 的函数地址和函数名拼接成一个新的数组

![](https://bbs.kanxue.com/upload/attach/202402/963320_87WG45Q78QESP5V.png)

之前替换`/proc/self/maps`来实现初步反调试的 js 函数`hook_proc_self_maps`也需要同时执行

输出结果如下,`KEkeELF`标志表示壳 ELF,`mainELF`表示主 ELF,(为什么是`KEke`, 只是为了对齐看着舒服:))

要判断调用的函数在哪个 ELF 里面, 在`trace_so()`里面稍作修改判断一下范围可以了

![](https://bbs.kanxue.com/upload/attach/202402/963320_3X7Q4XCFARPJFZF.png)

打印出来的结果如下

```
(KEkeELF)call1:JNI_OnLoad
(KEkeELF)call2:j_interpreter_wrap_int64_t
(KEkeELF)call3:interpreter_wrap_int64_t 
(KEkeELF)call4:getenv                   
(KEkeELF)call5:sub_13908                
(KEkeELF)call6:inotify_add_watch        
(KEkeELF)call7:sub_11220
(KEkeELF)call8:fopen
(KEkeELF)call9:sub_9DD8
(KEkeELF)call10:sub_E3E0
(KEkeELF)call11:strtol
(KEkeELF)call12:feof
(KEkeELF)call13:raise
(KEkeELF)call14:memset
(KEkeELF)call15:sub_C918
(KEkeELF)call16:sub_9988
(KEkeELF)call17:sub_9964
(KEkeELF)call18:sub_9AC4
(KEkeELF)call19:j_ffi_prep_cif
(KEkeELF)call20:ffi_prep_cif
(KEkeELF)call21:j_ffi_prep_cif_machdep
(KEkeELF)call22:ffi_prep_cif_machdep
(KEkeELF)call23:j_ffi_call
(KEkeELF)call24:ffi_call
(KEkeELF)call25:sub_1674C
(KEkeELF)call26:j_ffi_call_SYSV
(KEkeELF)call27:ffi_call_SYSV
(KEkeELF)call28:sub_167BC
(KEkeELF)call29:sub_1647C
(KEkeELF)call30:sub_163DC
(KEkeELF)call31:sub_9900
(KEkeELF)call32:sub_94BC
(KEkeELF)call33:inotify_init
(KEkeELF)call34:fmod
(KEkeELF)call35:strncpy
(KEkeELF)call36:_Z9__arm_a_1P7_JavaVMP7_JNIEnvPvRi
(KEkeELF)call37:sub_9E58
(KEkeELF)call38:sub_999C
(KEkeELF)call39:sub_10964
(KEkeELF)call40:j_lseek_1
(KEkeELF)call41:lseek
(KEkeELF)call42:sub_96E0
(KEkeELF)call43:sub_8000
(KEkeELF)call44:dlopen
(KEkeELF)call45:sub_60E0
(KEkeELF)call46:sub_6544
(KEkeELF)call47:sub_4B54
(KEkeELF)call48:sub_6128
(KEkeELF)call49:_ZN9__arm_c_19__arm_c_0Ev
(KEkeELF)call50:sub_A3EC
(KEkeELF)call51:sub_99CC
(KEkeELF)call52:sub_9944
(KEkeELF)call53:sub_6484
(KEkeELF)call54:sub_6590
(KEkeELF)call55:prctl
(KEkeELF)call56:sub_6698
(KEkeELF)call57:sub_9FFC
(KEkeELF)call58:j_lseek_3
(KEkeELF)call59:j_lseek_2
(KEkeELF)call60:j_lseek_0
(KEkeELF)call61:sub_9A90
(KEkeELF)call62:sub_5F20
(KEkeELF)call63:sub_6044
(KEkeELF)call64:sub_3574
(KEkeELF)call65:uncompress
(KEkeELF)call66:sub_49F0
(KEkeELF)call67:sub_5400
(KEkeELF)call68:sub_5478
(KEkeELF)call69:sub_5B08
(KEkeELF)call70:sub_5650
(KEkeELF)call71:sub_580C
(KEkeELF)call72:open
(KEkeELF)call73:atoi
(KEkeELF)call74:sub_3C94
(KEkeELF)call75:strncmp
(KEkeELF)call76:sub_4918
(KEkeELF)call77:sub_4000
(KEkeELF)call78:sub_41B4
(KEkeELF)call79:sub_35AC
(KEkeELF)call80:sigaction
(KEkeELF)call81:sub_5E6C
(KEkeELF)call82:sub_5444
(mainELF)call83:sub_11603C
(mainELF)call84:j__Znwm
(mainELF)call85:_Znwm
(mainELF)call86:malloc
(mainELF)call87:__cxa_atexit
(mainELF)call88:sub_1160B4
(mainELF)call89:sub_1160C4
(mainELF)call90:strlen
(mainELF)call91:memcpy
(mainELF)call92:sub_1161FC
(mainELF)call93:sub_1164AC
(mainELF)call94:sub_1164D8
(mainELF)call95:sub_116528
(mainELF)call96:sub_1165C8
(mainELF)call97:sub_1A32C0
(mainELF)call98:sub_1A3150
(mainELF)call99:sub_1A3204
(mainELF)call100:sub_1166FC
(mainELF)call101:sub_116728
(mainELF)call102:sub_116750
(mainELF)call103:sub_116830
(mainELF)call104:sub_116BA0
(KEkeELF)call105:sub_633C
(KEkeELF)call106:sub_8130
(KEkeELF)call107:sub_4C70
(KEkeELF)call108:sub_825C
(KEkeELF)call109:sub_8B50
(KEkeELF)call110:sub_8ED4
(KEkeELF)call111:sub_8430
(mainELF)call112:JNI_OnLoad
(mainELF)call113:j_interpreter_wrap_int64_t
(mainELF)call114:interpreter_wrap_int64_t
(KEkeELF)call115:interpreter_wrap_int64_t_bridge
(KEkeELF)call116:sub_9D60
(mainELF)call117:sub_1B3F0C
(mainELF)call118:gettimeofday
(mainELF)call119:sub_11BD9C
(mainELF)call120:sub_1182D8
(mainELF)call121:sub_123970
(mainELF)call122:sub_1B6448
(mainELF)call123:getenv
(mainELF)call124:sub_11F130
(mainELF)call125:sub_12047C
(mainELF)call126:j__ZdlPv
(mainELF)call127:_ZdlPv
(mainELF)call128:free
(mainELF)call129:sub_1427E8
(mainELF)call130:dlopen
(mainELF)call131:sub_11BDA8
(mainELF)call132:sub_11BE58
(mainELF)call133:sub_11F69C
(mainELF)call134:sub_117BE0
(mainELF)call135:sub_117CA0
(mainELF)call136:fopen
(mainELF)call137:sub_117E90
(mainELF)call138:sub_14285C
(mainELF)call139:sub_1429CC
(mainELF)call140:sub_11C1AC
(mainELF)call141:sub_11C1B4
(mainELF)call142:sub_11C210
(KEkeELF)call143:sub_166C4
(KEkeELF)call144:memcpy
(mainELF)call145:sub_123324
(mainELF)call146:sub_1205A0
(mainELF)call147:sub_11F768
(mainELF)call148:memcmp
(mainELF)call149:opendir
(mainELF)call150:closedir
(mainELF)call151:sub_11859C
(mainELF)call152:sub_11C268
(mainELF)call153:sub_11C300
(mainELF)call154:sub_117B68
(mainELF)call155:sub_1186B8
(mainELF)call156:sub_143964
(mainELF)call157:sub_1B66A8
(mainELF)call158:pthread_mutex_lock
(mainELF)call159:sub_142EA0
(mainELF)call160:sub_143A38
(mainELF)call161:sub_11CF8C
(mainELF)call162:sub_131D58
(mainELF)call163:sub_1B66D0
(mainELF)call164:pthread_mutex_unlock
(mainELF)call165:sub_1178E8
(mainELF)call166:sub_13D70C
(mainELF)call167:sub_19F984
(mainELF)call168:sub_11F1C8
(mainELF)call169:atoi
(mainELF)call170:sub_12D2F8
(mainELF)call171:sub_17ABE8
(mainELF)call172:sub_172660
(mainELF)call173:sub_13BFF0
(mainELF)call174:sub_172AA4
(mainELF)call175:sub_13BD80
(mainELF)call176:sub_13BE2C
(mainELF)call177:sub_13BE4C
(mainELF)call178:memmove
(mainELF)call179:sub_13BE64
(mainELF)call180:sub_172D78
(mainELF)call181:sub_13E510
(mainELF)call182:sub_1926F0
(mainELF)call183:sub_13DB7C
(mainELF)call184:sub_1B7A08
(mainELF)call185:sub_1B7ABC
(mainELF)call186:pthread_cond_broadcast
(mainELF)call187:sub_12FA34
(mainELF)call188:sub_120664
(mainELF)call189:sub_1332B8
(mainELF)call190:sub_13E0F8
(mainELF)call191:sub_12743C
(mainELF)call192:sub_124C68
(mainELF)call193:sub_125DC4
(mainELF)call194:sub_124510
(mainELF)call195:sub_126888
(mainELF)call196:strdup
(mainELF)call197:sub_126920
(mainELF)call198:sub_122180
(mainELF)call199:sub_11BC1C
(mainELF)call200:sub_13DF34
(mainELF)call201:getpid
(mainELF)call202:memset
(mainELF)call203:snprintf
(mainELF)call204:sub_124FA0
(mainELF)call205:sub_1B6498
(mainELF)call206:sub_1A0C88
(mainELF)call207:sub_217444
(mainELF)call208:sub_2175E0
(mainELF)call209:read
(mainELF)call210:strncmp
(mainELF)call211:close
(mainELF)call212:sub_1B578C
(mainELF)call213:j___self_lseek
(mainELF)call214:__self_lseek
(mainELF)call215:sub_1B586C
(mainELF)call216:j_j___read_self
(mainELF)call217:j___read_self
(mainELF)call218:__read_self
(mainELF)call219:sub_1B6528
(mainELF)call220:sub_1B6578
(mainELF)call221:mmap
(mainELF)call222:sub_1B5B50
(mainELF)call223:calloc
(mainELF)call224:memchr
(mainELF)call225:sub_1B5D04
(mainELF)call226:sub_1B5EC4
(mainELF)call227:sub_1B6270
(mainELF)call228:sub_1B6180
(mainELF)call229:sub_1B6678
(mainELF)call230:inflateInit2_
(mainELF)call231:inflate
(mainELF)call232:inflateEnd
(mainELF)call233:sub_1B6540
(mainELF)call234:munmap
(mainELF)call235:sub_1B56F8
(mainELF)call236:sub_19BC9C
(mainELF)call237:sub_19CCD4
(mainELF)call238:sub_12D470
(mainELF)call239:sub_142FE0
(mainELF)call240:sub_143008
(mainELF)call241:sub_142ABC
(mainELF)call242:sub_143848
(mainELF)call243:sub_143B48
(mainELF)call244:sub_143088
(mainELF)call245:sub_1222D0
(mainELF)call246:sub_14316C
(mainELF)call247:sub_142954
(KEkeELF)call248:_Z9__arm_a_2PcmS_Rii
(mainELF)call249:sub_142894
(mainELF)call250:sub_1428BC
(mainELF)call251:sub_127DCC
(mainELF)call252:sub_14292C
(mainELF)call253:sub_121B78
(mainELF)call254:sub_121BE0
(mainELF)call255:sub_123CE8
(mainELF)call256:sub_123BC0
(mainELF)call257:sub_11959C
(mainELF)call258:sub_1AC170
(mainELF)call259:pthread_create
(mainELF)call260:sub_1AC210
(mainELF)call261:sub_1B5DE4
(mainELF)call262:sub_1B60E8
(mainELF)call263:sub_19F7C4
(mainELF)call264:sub_1B2DC8
(mainELF)call265:sub_1B1CE8
(mainELF)call266:sub_1B0974
(mainELF)call267:sub_1AFE6C
(mainELF)call268:sub_126ED8
(mainELF)call269:sub_1AFE8C
(mainELF)call270:sub_1AFE90
(mainELF)call271:sub_1AB87C
(mainELF)call272:sub_1B26D4
(mainELF)call273:sub_1B26F4
(mainELF)call274:sub_1B27C8
(KEkeELF)call275:j_ffi_prep_cif_var
(KEkeELF)call276:ffi_prep_cif_var
(mainELF)call277:sub_1AAF48
(mainELF)call278:sub_1AAF54
(mainELF)call279:sub_2162D4
(mainELF)call280:sub_1B2898
(mainELF)call281:sub_1B2918
(mainELF)call282:sub_1ABE90
(mainELF)call283:sub_13E0EC
(mainELF)call284:sub_124900
(mainELF)call285:sub_1A0C34
(mainELF)call286:sub_217188
(mainELF)call287:j_strcmp
(mainELF)call288:strcmp
(mainELF)call289:sub_194514
(mainELF)call290:sub_1A2380
(mainELF)call291:sub_1A23CC
(mainELF)call292:sub_1A2718
(mainELF)call293:sub_1A2A94
(mainELF)call294:sub_1A25E0
(mainELF)call295:sub_1A2984

```

分析输出的结果, 我们发现了三个有趣的函数`inflateInit2_`,`inflate`,`inflateEnd`, 这不是 zlib 用来解压缩的函数嘛~

```
(mainELF)call230:inflateInit2_
(mainELF)call231:inflate
(mainELF)call232:inflateEnd

```

对`inflateInit2_`交叉引用, 发现有两个函数调用了它

![](https://bbs.kanxue.com/upload/attach/202402/963320_RGNGYZUZP2CZMKW.png)

那么要怎么知道是哪一个函数先调用的`inflateInit2_`呢? 向上看看函数调用链就行了

于是我们发现是`sub_1B6270`调用了`inflateInit2_`

```
(mainELF)call227:sub_1B6270 <--
(mainELF)call228:sub_1B6180
(mainELF)call229:sub_1B6678
(mainELF)call230:inflateInit2_
(mainELF)call231:inflate
(mainELF)call232:inflateEnd

```

我们来到`sub_1B6270`, 先到 [https://github.com/madler/zlib](https://github.com/madler/zlib) 把`zlib.h`中的`z_stream_s`, 导入的方法和之前一样

```
#  define z_const const
typedef unsigned char  Byte;  /* 8 bits */
typedef unsigned int   uInt;  /* 16 bits or more */
typedef unsigned long  uLong; /* 32 bits or more */
typedef struct z_stream_s {
    z_const Bytef *next_in;     /* next input byte */
    uInt     avail_in;  /* number of bytes available at next_in */
    uLong    total_in;  /* total number of input bytes read so far */
 
    Bytef    *next_out; /* next output byte will go here */
    uInt     avail_out; /* remaining free space at next_out */
    uLong    total_out; /* total number of bytes output so far */
} z_stream;

```

重定义`s`的类型为`z_stream`, 这四个字段的含义如下

*   `s.next_in`: 压缩数据
*   `s.avail_in`: 压缩数据的长度
*   `s.next_out`: 解压后的数据
*   `s.avail_out`: 解压后数据的长度

![](https://bbs.kanxue.com/upload/attach/202402/963320_UJFHCSNBQGTCTBZ.png)

各个成员的偏移如图所示

![](https://bbs.kanxue.com/upload/attach/202402/963320_QRWQQNVVH4U3S45.png)

随后我们用 frida 去 hook 一下`inflate`函数看看解压缩之后的数据是什么

这里有个技巧, 就是如何可以 hook 到主 ELF 中的函数,, 因为在壳 ELF 加载进内存时, 主 ELF 还没有被加载, 所以假如在壳 ELF 通过`android_dlopen_ext`打开时我们进行 hook, 是会 hook 失败的

那么如何才能获取到主 ELF 的 hook 时机呢? 我们可以通过统计外部函数的调用次数来判断是否已经加载了主 ELF, 例如我这里, 我通过`zlib_count`统计外部函数`inflate`调用次数, 因为在壳 ELF 会使用`uncompress`调用一次`inflate`, 所以当第二次调用`inflate`, 我们就知道这肯定是主 ELF 调用的, 所以我们也可以在这个位置放心大胆的 hook 了

```
function dump_memory(start,size,filename) {
    var file_path = "/data/data/com.oacia.apk_protect/" + filename;
    var file_handle = new File(file_path, "wb");
    if (file_handle && file_handle != null) {
        var libso_buffer = start.readByteArray(size.toUInt32());
        file_handle.write(libso_buffer);
        file_handle.flush();
        file_handle.close();
        console.log("[dump]:", file_path);
    }
}
function hook_zlib_result(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x1B63F0), {
        // fd, buff, len
        onEnter: function (args) {
            console.log("inflate result")
            console.log(hexdump(next_in, {
              offset: 0,// 相对偏移
              length: 0x50,//dump 的大小
              header: true,
              ansi: true
            }));
            console.log(hexdump(next_out, {
              offset: 0,// 相对偏移
              length: 0x50,//dump 的大小
              header: true,
              ansi: true
            }));
            dump_memory(next_out,avail_out,"dex001")
        },
        onLeave: function (ret) {
        }
    });
}
var zlib_count=0;
var next_in,avail_in,next_out,avail_out;
function hook_zlib(){
    Interceptor.attach(Module.findExportByName(null, "inflate"), {
        // fd, buff, len
        onEnter: function (args) {
            zlib_count+=1
            if(zlib_count>1){
                hook_zlib_result();
            }
            next_in=ptr(args[0].add(0x0).readS64());
            avail_in=ptr(args[0].add(0x8).readS64());
            next_out=ptr(args[0].add(0x18).readS64());
            avail_out=ptr(args[0].add(0x20).readS64());
            console.log(hexdump(next_in, {
              offset: 0,// 相对偏移
              length: 0x50,//dump 的大小
              header: true,
              ansi: true
            }));
            console.log(args[1]);
        },
        onLeave: function (ret) {
        }
    });
}

```

解压缩之后的输出如下, 在输出的文件头, 我们发现了`dex035`, 所以我们把这块内存 dump 下来看看, 使用上方的`dump_memory(start,size,filename)`函数即可

![](https://bbs.kanxue.com/upload/attach/202402/963320_8CNUHM7B49FTUJ9.png)

把这个解压缩之后的 dex 拖入到 jadx 里面, 却发现这个类名怎么和壳 DEX 的类名一模一样, 通过校验哈希发现 dump 下来的 dex 和壳 dex 其实是同一个文件

![](https://bbs.kanxue.com/upload/attach/202402/963320_W9JT38BKG6AWJBN.png)

我们在之前的分析中知道壳 dex 的末尾附带了一大串的加密数据, 所以通过将这个解压缩得到了这个 dex, 就说明马上要进行加密主 DEX 的解密操作了

解压缩的函数是`sub_1B6270`, 接下来我们继续通过`stalker_trace_so`打印出来的内容, 并利用交叉引用来追踪该函数的调用链

就比如说对于函数`sub_1B6270`, 它有两个交叉引用

![](https://bbs.kanxue.com/upload/attach/202402/963320_ASNFDQWWDEBHYQ7.png)

通过`stalker_trace_so`打印出来的函数调用链, 我们发现是`sub_1A0C88`在`sub_1B6270`之前调用, 所以函数的调用关系就是`sub_1A0C88->sub_1B6270`, 以此类推

```
(mainELF)call206:sub_1A0C88
(mainELF)call227:sub_1B6270

```

所以一路跟过来之后, 函数的调用链为`sub_1332B8->sub_124FA0->sub_1A0C88->sub_1B6270->inflate`,`sub_1332B8`函数之后就没有交叉引用了

```
(mainELF)call189:sub_1332B8
(mainELF)call204:sub_124FA0
(mainELF)call206:sub_1A0C88
(mainELF)call227:sub_1B6270
(mainELF)call230:inflateInit2_
(mainELF)call231:inflate
(mainELF)call232:inflateEnd

```

在这个函数中, 我们发现了`apk@classes.dex`, 而它的作用, 正是为了找到已加载到内存且优化后的壳 dex

![](https://bbs.kanxue.com/upload/attach/202402/963320_DUDUTNJBVYQ55N2.png)

* * *

在**加固壳反调试初步分析**的后半部分, 我们打印出了加固壳打开 dex 的堆栈回溯, 现在我们直接跳转到相对应的地方看看

![](https://bbs.kanxue.com/upload/attach/202402/963320_6VM9JZGEAK2E9SM.png)

我们到`0x19b780`看看, 看起来是一个标准的打开并写入文件的函数

![](https://bbs.kanxue.com/upload/attach/202402/963320_VER9JRE4F7Q5RSF.png)

随后对该函数进行交叉引用, 我们发现`sub_1332B8`竟然调用了它, 就在刚刚我们就分析出这个函数中可是同时也执行了从内存中获取壳 dex 的操作的呢

![](https://bbs.kanxue.com/upload/attach/202402/963320_8D8XVGV7UVTT543.png)

我们对这两处调用都 hook 一下看看是什么情况, 打印的结果如下, 说明这两处调用都打开了 dex,`sub_1332B8`中的前一个调用打开了`classes.dex`, 后一个调用打开了`classes2.dex`和`classes3.dex`, 而`classes.dex`文件中的内容就是加密的主 dex

![](https://bbs.kanxue.com/upload/attach/202402/963320_PK8AJDC6ABSZS62.png)

在创建完`classes2.dex`和`classes3.dex`, 通过 hook 发现调用在调用`sub_128D44`之后进程就退出了

![](https://bbs.kanxue.com/upload/attach/202402/963320_CYU4ZJ2DQY9VHZE.png)

我们去 hook 一下`sub_128D44`这个函数, 发现传入的参数`v8`正是加密的主 DEX

![](https://bbs.kanxue.com/upload/attach/202402/963320_SRR2H9VKYHB5ND2.png)

而`sub_128D44`函数是这个样子的, 并且在壳 ELF 加载时启动`stalker_trace_so`的`trace_so()`函数所打印出的结果中, 并没有这个函数的调用被打印出来

![](https://bbs.kanxue.com/upload/attach/202402/963320_Y88WWABDV38GPPU.png)

这该怎么办呢?

很简单, 在调用`sub_128D44`的位置再去调用一次`trace_so()`函数从现在的位置开始打印函数的调用链不就行咯:)

![](https://bbs.kanxue.com/upload/attach/202402/963320_D7435DMAUU8AHRP.png)

函数调用关系如下, 我们发现再`mainELF`调用完`sub_128D44`之后, 通过一系列操作又回到了壳 ELF 中, 最终调用`raise`导致进程退出

![](https://bbs.kanxue.com/upload/attach/202402/963320_DA9QE2QPSS4WQZS.png)

然而, 当我跳转到最后调用的几个函数时, 可以说函数复杂到让人咋舌

![](https://bbs.kanxue.com/upload/attach/202402/963320_9MDJ2JK77G3KZR8.png)

![](https://bbs.kanxue.com/upload/attach/202402/963320_C6D6VFURBVYT9GP.png)

这么复杂, 是给人分析的吗!? 所以我便卡在这里了很久

我想了想现在摆在面前的有两条路, 是和一眼望不到尽头的这俩函数死磕到底, 还是选择把 360 加固的反调试搞定?

我选择后者, 因为明显搞定反调试要比把这两个函数分析明白要稍微简单一点

加固壳反调试深入分析
==========

在**加固壳反调试初步分析**中, 我曾尝试过`dbus`,`TracerPid`,`readlink`,`strstr`都没有明显的效果, 只有 hook `open`函数让我看到了些许的曙光, 那么现在应该还有一种非常重要的反调试手段没有用到, 那就是`pthread_create`反调试

```
function check_pthread_create() {
    var pthread_create_addr = Module.findExportByName(null, 'pthread_create');
 
    var pthread_create = new NativeFunction(pthread_create_addr, "int", ["pointer", "pointer", "pointer", "pointer"]);
    Interceptor.replace(pthread_create_addr, new NativeCallback(function (parg0, parg1, parg2, parg3) {
        var so_name = Process.findModuleByAddress(parg2).name;
        var so_path = Process.findModuleByAddress(parg2).path;
        var so_base = Module.getBaseAddress(so_name);
        var offset = parg2 - so_base;
        var PC = 0;
        if ((so_name.indexOf("jiagu") > -1)) {
            console.log("======")
            console.log("find thread func offset", so_name, offset.toString(16));
            Thread.backtrace(this.context, Backtracer.ACCURATE).map(addr_in_so);
 
            var check_list = []//1769036,1771844
            if (check_list.indexOf(offset)!==-1) {
                console.log("check bypass")
            } else {
                PC = pthread_create(parg0, parg1, parg2, parg3);
            }
        } else {
            PC = pthread_create(parg0, parg1, parg2, parg3);
        }
        return PC;
    }, "int", ["pointer", "pointer", "pointer", "pointer"]))
}

```

注入代码之后,`pthread_create`的调用都指向了同一个地址`0x17710`

![](https://bbs.kanxue.com/upload/attach/202402/963320_GJZ7NCADJUEMXZJ.png)

我们跳转到这个地址之后却发现为什么会没有`pthread_create`呢??

![](https://bbs.kanxue.com/upload/attach/202402/963320_YBHVUTWTWH44J45.png)

看了一眼这个代码所在的函数的名称`ffi_call_SYSV`

![](https://bbs.kanxue.com/upload/attach/202402/963320_WSE92UKRZN6ZZC8.png)

hmmm, 看来是用 [libffi 动态调用函数](https://alexcode2.gitbook.io/ios-development-guidelines/ios-za-tan/libffi-dong-tai-tiao-yong-he-ding-yi-c-han-shu)呀

![](https://bbs.kanxue.com/upload/attach/202402/963320_KVJSJFV6YZ634K9.png)

直接到 [libffi 的 github 仓库](https://github.com/libffi)看一眼 [ffi_call_SYSV 的源码](https://github.com/libffi/libffi/blob/master/src/aarch64/sysv.S)

一进去注释都写得清清楚楚了

![](https://bbs.kanxue.com/upload/attach/202402/963320_76DXCG7MKHJPUKU.png)

利用注释就可以知道每行汇编都代表什么了, 所以`BLR X24`表示去动态调用函数, 而前面的`X0`,`X2`,`X4`,`X6`是用来传参的

![](https://bbs.kanxue.com/upload/attach/202402/963320_9ADKPD9J4ZDPW2U.png)

我们 hook 一下`x0`看看有没有什么敏感的字符串

```
function anti_frida_check(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x1770C), {
        onEnter: function (args) {
            try{
                console.log(this.context.x0.readCString())
            }
            catch (e){
 
            }
        },
        onLeave: function (ret) {
        }
    });
 
}

```

然而神奇的是, 我仅仅去 hook 并打印 x0 字符串, 其他什么事情都不干, apk 竟然神奇的进去了, 只不过会没有响应, 感觉距离成功不远了呢

![](https://bbs.kanxue.com/upload/attach/202402/963320_HA989D2DG2S7YC2.png)

有点意思, 筛选一下看看有没有什么敏感的字符串好咯

```
function anti_frida_check(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x1770C), {
        onEnter: function (args) {
            try{
                var s = this.context.x0.readCString();
                if (s.indexOf('frida')!==-1 ||
                    s.indexOf('gum-js-loop')!==-1 ||
                    s.indexOf('gmain')!==-1 ||
                    s.indexOf('linjector')!==-1 ||
                    s.indexOf('/proc/')!==-1){
                    console.log(s)
                }
            }
            catch (e){
 
            }
        },
        onLeave: function (ret) {
        }
    });
 
}

```

竟然还真有, 那就把这些字符串全部替换成无意义的字符串看看咯

![](https://bbs.kanxue.com/upload/attach/202402/963320_36S54VA5UK5PG4C.png)

```
function anti_frida_check(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x1770C), {
        onEnter: function (args) {
            try{
                var s = this.context.x6.readCString();
                if (s.indexOf('frida')!==-1 ||
                    s.indexOf('gum-js-loop')!==-1 ||
                    s.indexOf('gmain')!==-1 ||
                    s.indexOf('linjector')!==-1 ||
                    s.indexOf('/proc/')!==-1){
                    console.log(s)
                    Memory.protect(this.context.x0, Process.pointerSize, 'rwx');
                    var replace_str=""
                    for(var i=0;i
```

然而这样做进程却一个劲的崩溃!!

没事, 寄存器`x0`用不了, 还有`x2`,`x4`,`x6`没替换过呢! 我一个一个的试过去, 终于, 当我将寄存器改成 x6 时, 进程终于不再崩溃成功的进入了 apk!

```
function anti_frida_check(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x1770C), {
        onEnter: function (args) {
            try{
                var s = this.context.x6.readCString();
                if (s.indexOf('frida')!==-1 ||
                    s.indexOf('gum-js-loop')!==-1 ||
                    s.indexOf('gmain')!==-1 ||
                    s.indexOf('linjector')!==-1 ||
                    s.indexOf('/proc/')!==-1){
                    //console.log(s)
                    Memory.protect(this.context.x0, Process.pointerSize, 'rwx');
                    var replace_str=""
                    for(var i=0;i
```

![](https://bbs.kanxue.com/upload/attach/202402/963320_C5ZEGKHVC87D8GM.png)

看一眼检测的字符串, 怎么全是`/memfd:frida-agent-64.so`

![](https://bbs.kanxue.com/upload/attach/202402/963320_W73B8CK2J7A3X52.png)

主 DEX 加载流程分析
============

回到这个卡了我们很久位置, 现在过了反调试之后这里的代码终于可以继续执行下去了

![](https://bbs.kanxue.com/upload/attach/202402/963320_XXMJ5WYZBS3M3EZ.png)

向下看找到这一个函数

![](https://bbs.kanxue.com/upload/attach/202402/963320_WHNAKZVSTPTFTUF.png)

在这个函数中的字符串全部都是加密的, 颇有种此地无银三百两的感觉, 我们把字符串解密后发现了`DexFileLoader`相关的字符串, 说明这个函数肯定和加载 dex 有某种关联

![](https://bbs.kanxue.com/upload/attach/202402/963320_XPRKFPJ9RZW867N.png)

我们 hook 一下这个函数, 发现这个函数共调用了三次, 而且传入的值都是已经解密了的 dex,`classes.dex`,`classes2.dex`,`classes3.dex`分别通过这个函数加载

`classes.dex`

![](https://bbs.kanxue.com/upload/attach/202402/963320_A6782WHCURMW6D7.png)

`classes2.dex`

![](https://bbs.kanxue.com/upload/attach/202402/963320_BEA4FYU2UY5TMVN.png)

`classes3.dex`

![](https://bbs.kanxue.com/upload/attach/202402/963320_N9JWPRWCAHZJDN8.png)

把这三个 dex 给 dump 下来看看, 于是我们得到了这三个文件

![](https://bbs.kanxue.com/upload/attach/202402/963320_AXQR4KCRE68TFZN.png)

把最大的那个 dex 拖到 jadx 里面

![](https://bbs.kanxue.com/upload/attach/202402/963320_67TEBTWQQWM5N6P.png)

看了看这些类和直接反编译未加固的 apk 的类一模一样

![](https://bbs.kanxue.com/upload/attach/202402/963320_C2CQRCY9WNUV7CF.png)

至此, 360 加固分析完毕

总结
==

360 加固和常规的加固方案类似, 都是`壳DEX->壳ELF->主ELF->主DEX`这样的过程, 其中壳 ELF 解密主 ELF 所用到的算法是`RC4`和`uncompress`, 壳 ELF 加载主 ELF 所用到的技术是自实现 linker 加固 so

以前我就听说 360 加固是有 VMP 的, 但是我这一路下来都没遇见, 我想 VMP 应该是在 DEX 的解密中所用到的, 但是以我现在的水平或许还不足以对抗 VMP, 所以等未来学会了`Static Analysis`之后再回过头来研究这个样本中的 VMP 吧!

参考资料
====

*   [360 加固保关键技术浅析](https://www.77169.net/html/170724.html)
*   [Decrypting strings with a JEB script](https://cryptax.medium.com/decrypting-strings-with-a-jeb-script-1af522fa4979)
*   [自实现 linker 加固 so](https://www.cnblogs.com/revercc/p/17100308.html)
*   [360 加固保分析](https://bbs.kanxue.com/thread-260049.htm)
*   [误入虎穴，喜得虎子——记一次手游加固的脱壳与修复](https://bbs.kanxue.com/thread-275811.htm)

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#HOOK 注入](forum-161-1-125.htm) [#脱壳反混淆](forum-161-1-122.htm) [#NDK 分析](forum-161-1-119.htm) [#逆向分析](forum-161-1-118.htm)

上传的附件：

*   [360jiagu.apk](javascript:void(0)) （6.77MB，5 次下载）