> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280754.htm)

> [原创] 绕过爱奇艺新版 libmsaoaidsec.so Frida 检测

[原创] 绕过爱奇艺新版 libmsaoaidsec.so Frida 检测

9 小时前 200

### [原创] 绕过爱奇艺新版 libmsaoaidsec.so Frida 检测

 [![](http://passport.kanxue.com/upload/avatar/681/956681.png?1659624736)](user-home-956681.htm) [yyy123](user-home-956681.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 9 小时前  200

1、环境：
=====

手机：Google Pixel 6，Aosp android13  
cpu 架构：Arm64  
Frida：16.1.10  
爱奇艺：15.2.5

2、问题
====

这段时间在研究爱奇艺的一些功能，当用 Frida 调试时发现有反 Frida 检测，现象是执行`frida -l index.js -U -f com.qiyi.video`时进程重启，下面是对 Frida 检测的一些分析，并通过 hook 绕过 Frida 检测。

3、分析
====

用 Frida hook do_dlopen 函数看加载哪个 so 时崩溃的，hook 之前先获取 do_dlopen 在 linker64 中的相对偏移  
![](https://bbs.kanxue.com/upload/attach/202403/956681_M8YT5HFFQYW8P5S.webp)

hook 脚本如下：
----------

```
function hookDlopen() {
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x3ba00 // __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
    let android_dlopen_ext = linker64_base_addr.add(offset)
    Interceptor.attach(android_dlopen_ext, {
      onEnter: function(args){
        this.name = args[0].readCString()
        Log.log(`dlopen onEnter ${this.name}`)
      }, onLeave: function(retval){
        Log.log(`dlopen onLeave name: ${this.name}`)
                if (this.name != null && this.name.indexOf('libmsaoaidsec.so') >= 0) {
          let JNI_OnLoad = Module.getExportByName(this.name, 'JNI_OnLoad')
          Log.log(`dlopen onLeave JNI_OnLoad: ${JNI_OnLoad}`)
        }
      }
    })
  }

```

执行`frida -l index.js -U -f com.qiyi.video`
------------------------------------------

![](https://bbs.kanxue.com/upload/attach/202403/956681_6FZRKEWB3SK9J5X.webp)  
从结果可以看出加载的最后一个 so 是：libmsaoaidsec.so，并且没有调用 onLeave，由此可断定崩溃点在 libmsaoaidsec.so 中，并且在 JNI_OnLoad 之前检测的，检测点应该是在 init_array 初始化函数中。  
<br/>  
既然挂上 Frida 后进程就退出，那么我们就来分析是调用哪个系统调用退出的，可以通过 strace 查看系统调用，但在执行 strace 时需要在 dlopen 加载 libmsaoaidsec.so 之前让线程 sleep 10 秒，以便留出 strace 执行时机。

```
function hookDlopen() {
    // 获取libc中的sleep函数
    let sleep: Function = new NativeFunction(Module.getExportByName('libc.so', 'sleep'), 'uint', ['uint'])
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x3ba00 // __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
    let android_dlopen_ext = linker64_base_addr.add(offset)
    Interceptor.attach(android_dlopen_ext, {
      onEnter: function(args){
        this.name = args[0].readCString()
        Log.log(`dlopen onEnter ${this.name}`)
        if (this.name != null && this.name.indexOf('libmsaoaidsec.so') >= 0) {
          sleep(10) // sleep10秒留出strace执行时机
        }
      }, onLeave: function(retval){
        Log.log(`dlopen onLeave name: ${this.name}`)
      }
    })
  }

```

下面重新执行`frida -l index.js -U -f com.qiyi.video`
----------------------------------------------

![](https://bbs.kanxue.com/upload/attach/202403/956681_327JNERN9DYECMW.webp)

然后得到 pid 立即执行 strace -e trace=process -i -f -p 15534 等待 10 秒后结果：
----------------------------------------------------------------

![](https://bbs.kanxue.com/upload/attach/202403/956681_TPDWPJV3V6B3CYY.webp)  
我们看这一行`[pid 15576] [000000751926c008] exit_group(0 <unfinished ...>`，显示 15576 线程是在 0x751926c008 地址处调用 exit_group 退出的，通过 proc/15534/maps 查看 libc.so 的地址范围是 0x7509b9c000 - 0x7509c70000，很明显 0x751926c008 不是 libc.so 的地址，由此可以断定 exit_group 的代码是动态释放的。  
<br/>  
动态释放代码一定是要操作内存的，接下来我们用前面相同的逻辑，用 strace 查看调用了哪些和内存相关的系统调用  
<br/>  
`strace -e trace=process,memory -i -f -p 25947`  
![](https://bbs.kanxue.com/upload/attach/202403/956681_AJDYG9RMXV9257X.webp)  
注意这行：[pid 25990] [0000007509c48ab8] mmap(NULL, 28, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x751926c000  
<br/>  
上面这行 mmap 申请的内存返回的地址是：0x751926c000，正好匹配后面 exit_group 前面的地址：000000751926c008  
<br/>  
由 mmap 的地址可以看出 mmap 是 libc.so 中的函数，于是我们可以 hook mmap 打印一下调用栈

```
function hook_mmap() {
  const mmap = Module.getExportByName("libc.so", "mmap");
  Interceptor.attach(mmap, {
    onEnter: function (args) {
      let length = args[1].toString(16)
      if (parseInt(length, 16) == 28) {
        Log.log('backtrace:\n' + Thread.backtrace(this.context, Backtracer.ACCURATE)
                                                  .map(DebugSymbol.fromAddress).join('\n') + '\n');
      }
    }
  })
}
 
function hookDlopen() {
    let sleep: Function = new NativeFunction(Module.getExportByName('libc.so', 'sleep'), 'uint', ['uint'])
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x3ba00 // __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
    let android_dlopen_ext = linker64_base_addr.add(offset)
    Interceptor.attach(android_dlopen_ext, {
      onEnter: function(args){
        this.name = args[0].readCString()
        Log.log(`dlopen onEnter ${this.name}`)
        if (this.name != null && this.name.indexOf('libmsaoaidsec.so') >= 0) {
          // sleep(10)
      hook_mmap()
        }
      }, onLeave: function(retval) {
        Log.log(`dlopen onLeave name: ${this.name}`)
      }
    })
  }

```

调用栈如下：
------

![](https://bbs.kanxue.com/upload/attach/202403/956681_VGC94QD3PNWEDYC.webp)

下面用 IDA 打开 libmsaoaidsec.so，看 0x2358c 地址处对应的函数
----------------------------------------------

![](https://bbs.kanxue.com/upload/attach/202403/956681_UVHAUG3SJ5VEEMJ.webp)

下面看 C 语言伪码，的确是在释放代码并且执行：
------------------------

![](https://bbs.kanxue.com/upload/attach/202403/956681_HEWBB69AM2F9M8H.webp)

经过 IDA 分析 0x1bdcc 地址对应的函数：`void sub_1B924()`是检测逻辑所在，具体检测逻辑就不在这赘述了，大家有兴趣可以自行研究
-----------------------------------------------------------------------------

![](https://bbs.kanxue.com/upload/attach/202403/956681_S4YX3ED9AVUNGA4.webp)

4、解决方案
======

解决方案就是 replace 掉这个无返回值的函数 `void sub_1B924()`  
<br/>  
由于这个检测逻辑在. init_函数中，所以得先找到 hook 时机，查看 Aosp 代码发现 linker 执行 init_array 类的函数是 call_constructors：

```
void soinfo::call_constructors() {
  if (constructors_called) {
    return;
  }
  constructors_called = true;
 
  if (!is_main_executable() && preinit_array_ != nullptr) {
    // The GNU dynamic linker silently ignores these, but we warn the developer.
    PRINT("\"%s\": ignoring %zd-entry DT_PREINIT_ARRAY in shared library!",
          get_realpath(), preinit_array_count_);
  }
 
  get_children().for_each([] (soinfo* si) {
    si->call_constructors();
  });
 
  TRACE("\"%s\": calling constructors", get_realpath());
 
  // DT_INIT should be called before DT_INIT_ARRAY if both are present.
  call_function("DT_INIT", init_func_);
  call_array("DT_INIT_ARRAY", init_array_, init_array_count_, false);
}

```

所以直接 hook call_constructors 函数，在 onEnter 中 replace 掉 sub_1b924  
<br/>  
查看 call_constructors 在 linker64 中的偏移  
![](https://bbs.kanxue.com/upload/attach/202403/956681_USE5ZNV8TMW8T5J.webp)

5、最终代码
======

```
function hookDlopen() {
  let linker64_base_addr = Module.getBaseAddress('linker64')
  let offset = 0x3ba00 // __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
  let android_dlopen_ext = linker64_base_addr.add(offset)
  if (android_dlopen_ext != null) {
    Interceptor.attach(android_dlopen_ext, {
      onEnter: function(args){
        this.name = args[0].readCString()
        if (this.name != null && this.name.indexOf('libmsaoaidsec.so') >= 0) {
          hook_linker_call_constructors()
        }
      }, onLeave: function(retval){
        Log.log(`dlopen onLeave name: ${this.name}`)
        if (this.name != null && this.name.indexOf('libmsaoaidsec.so') >= 0) {
          let JNI_OnLoad = Module.getExportByName(this.name, 'JNI_OnLoad')
          Log.log(`dlopen onLeave JNI_OnLoad: ${JNI_OnLoad}`)
        }
      }
    })
  }
}
 
function hook_linker_call_constructors() {
  let linker64_base_addr = Module.getBaseAddress('linker64')
  let offset = 0x521f0 // __dl__ZN6soinfo17call_constructorsEv
  let call_constructors = linker64_base_addr.add(offset)
  let listener = Interceptor.attach(call_constructors, {
    onEnter: function (args) {
      Log.log('hook_linker_call_constructors onEnter')
      let secmodule = Process.findModuleByName("libmsaoaidsec.so")
      if (secmodule != null) {
        hook_sub_1b924(secmodule)
        listener.detach()
      }
    }
  })
}
 
function hook_sub_1b924(secmodule) {
  Interceptor.replace(secmodule.base.add(0x1b924), new NativeCallback(function () {
    Log.log(`hook_sub_1b924 >>>>>>>>>>>>>>>>> replace`)
  }, 'void', []));
}

```

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

最后于 9 小时前 被 yyy123 编辑 ，原因： [#逆向分析](forum-161-1-118.htm)